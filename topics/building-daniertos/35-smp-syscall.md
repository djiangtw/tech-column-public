# 多核 System Call：當兩個核心同時敲門

**作者**: Danny Jiang
**日期**: 2025-12-26

---

## 前言：同時敲門的問題

在單核世界裡，Syscall 處理很簡單：一次只有一個 Task 在執行，一次只有一個 Syscall 在處理。

但在 SMP 世界裡，事情變得有趣了。

想像這個場景：

```
Core 0: Task A 呼叫 sys_sem_post(&sem)
Core 1: Task B 呼叫 sys_sem_wait(&sem)
        （同時發生！）
```

兩個核心同時存取同一個 semaphore。如果沒有適當的同步，結果可能是：
- 資料損壞
- 死鎖
- 遺失喚醒

這一章，我們要探討 danieRTOS v3.x 如何處理多核 Syscall。

---

## 一、Syscall 的基本流程

### 1.1 從 User Mode 到 Kernel

```
User Mode                                    Kernel Mode
    │                                            │
    │ sys_sem_post(&sem)                         │
    │   ↓                                        │
    │ ecall                                      │
    ▼                                            │
┌─────────────────────────────────────────────────────────────┐
│ trap_entry:                                                 │
│   - 保存 context                                            │
│   - 切換到 Kernel Stack                                     │
│   - 呼叫 trap_handler()                                     │
└─────────────────────────────────────────────────────────────┘
    │                                            │
    ▼                                            │
┌─────────────────────────────────────────────────────────────┐
│ trap_handler:                                               │
│   if (mcause == ECALL_FROM_U) {                             │
│       syscall_dispatch(ctx);                                │
│   }                                                         │
└─────────────────────────────────────────────────────────────┘
    │                                            │
    ▼                                            │
┌─────────────────────────────────────────────────────────────┐
│ syscall_dispatch:                                           │
│   switch (syscall_num) {                                    │
│       case SYS_SEM_POST: sys_sem_post_handler(ctx); break;  │
│       case SYS_SEM_WAIT: sys_sem_wait_handler(ctx); break;  │
│       ...                                                   │
│   }                                                         │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Syscall 編號

```c
/* syscall.h */
#define SYS_EXIT        0
#define SYS_PUTCHAR     1
#define SYS_PUTS        2
#define SYS_DELAY       3
#define SYS_SEM_WAIT    4
#define SYS_SEM_POST    5
#define SYS_MUTEX_LOCK  6
#define SYS_MUTEX_UNLOCK 7
#define SYS_QUEUE_SEND  8
#define SYS_QUEUE_RECV  9
```

---

## 二、無鎖 Dispatcher

### 2.1 Dispatcher 設計

Syscall dispatcher 本身是**無鎖**的：

```c
void syscall_dispatch(context_t *ctx)
{
    uint64_t syscall_num = ctx->a7;  /* a7 = syscall number */
    uint64_t arg0 = ctx->a0;
    uint64_t arg1 = ctx->a1;
    uint64_t arg2 = ctx->a2;
    
    int64_t ret = 0;
    
    switch (syscall_num) {
    case SYS_EXIT:
        task_exit();
        /* 不返回 */
        break;
        
    case SYS_PUTCHAR:
        uart_putchar((char)arg0);
        break;
        
    case SYS_PUTS:
        /* 驗證 User 指標 */
        if (is_valid_user_string((const char *)arg0)) {
            uart_puts((const char *)arg0);
        } else {
            ret = -EFAULT;
        }
        break;
        
    case SYS_DELAY:
        task_delay_internal((tick_t)arg0);
        break;
        
    case SYS_SEM_WAIT:
        ret = sem_wait_internal((sem_t *)arg0);
        break;
        
    case SYS_SEM_POST:
        ret = sem_post_internal((sem_t *)arg0);
        break;
        
    /* ... 其他 syscall ... */
        
    default:
        ret = -ENOSYS;
        break;
    }
    
    ctx->a0 = ret;  /* 返回值放在 a0 */
}
```

**為什麼 dispatcher 是無鎖的？**

- Dispatcher 只是一個 switch-case，不存取共享資料
- 每個核心有自己的 `ctx`，不會衝突
- 鎖定發生在具體的 syscall handler 內部

---

## 三、Fine-Grained Locking

### 3.1 Per-Object Locking

每個同步物件有自己的 spinlock：

```c
typedef struct semaphore {
    spinlock_t lock;        /* 保護這個 semaphore */
    int32_t count;
    tcb_t *wait_list;
} sem_t;

typedef struct mutex {
    spinlock_t lock;        /* 保護這個 mutex */
    tcb_t *owner;
    int32_t lock_count;
    tcb_t *wait_list;
} mutex_t;

typedef struct queue {
    spinlock_t lock;        /* 保護這個 queue */
    uint8_t *buffer;
    size_t item_size;
    size_t capacity;
    size_t head, tail, count;
    tcb_t *send_wait_list;
    tcb_t *recv_wait_list;
} queue_t;
```

### 3.2 Semaphore 實作

```c
int sem_wait_internal(sem_t *sem)
{
    cpu_t *cpu = smp_get_cpu();
    tcb_t *task = cpu->current_task;
    
    reg_t state = spinlock_lock_irqsave(&sem->lock);
    
    if (sem->count > 0) {
        /* 有資源，直接獲取 */
        sem->count--;
        spinlock_unlock_irqrestore(&sem->lock, state);
        return 0;
    }
    
    /* 沒有資源，需要阻塞 */
    task->state = TASK_STATE_BLOCKED;
    task->blocked_on = sem;
    
    /* 加入等待列表 */
    task->next = sem->wait_list;
    sem->wait_list = task;
    
    /* 從 ready queue 移除 */
    sched_remove_ready(task);
    
    spinlock_unlock_irqrestore(&sem->lock, state);

    /* 觸發重新排程 */
    sched_schedule();

    return 0;
}

int sem_post_internal(sem_t *sem)
{
    reg_t state = spinlock_lock_irqsave(&sem->lock);

    if (sem->wait_list != NULL) {
        /* 有 Task 在等待，喚醒第一個 */
        tcb_t *task = sem->wait_list;
        sem->wait_list = task->next;
        task->next = NULL;

        spinlock_unlock_irqrestore(&sem->lock, state);

        /* 喚醒 Task（可能在其他核心） */
        task_make_ready(task);

        /* 如果喚醒的 Task 優先級更高，請求重新排程 */
        cpu_t *cpu = smp_get_cpu();
        if (task->priority > cpu->current_task->priority) {
            sched_request_switch();
        }

        /* 如果 Task 綁定到其他核心，發送 IPI */
        if ((task->affinity_mask & (1U << cpu->hartid)) == 0) {
            /* 找到 Task 可以執行的核心，發送 IPI */
            for (uint32_t i = 0; i < CONFIG_NUM_CORES; i++) {
                if (task->affinity_mask & (1U << i)) {
                    smp_request_reschedule(i);
                    break;
                }
            }
        }
    } else {
        /* 沒有 Task 等待，增加計數 */
        sem->count++;
        spinlock_unlock_irqrestore(&sem->lock, state);
    }

    return 0;
}
```

---

## 四、IPI 發送策略

### 4.1 為什麼需要 IPI？

考慮這個場景：

```
Core 0: Task A (priority 3) 執行中
        呼叫 sem_post(&sem)
        喚醒 Task B (priority 5, 綁定 Core 1)

Core 1: Task C (priority 2) 執行中
        不知道 Task B 已經 ready
```

如果 Core 0 不通知 Core 1，Task B 會一直等到 Core 1 的下一個 Timer 中斷才能執行。

**IPI (Inter-Processor Interrupt)** 解決了這個問題：Core 0 發送 IPI 給 Core 1，強制 Core 1 立即重新排程。

### 4.2 IPI 發送邏輯

```c
void smp_request_reschedule(uint64_t target_hartid)
{
    if (target_hartid >= CONFIG_NUM_CORES) {
        return;
    }

    /* 設置目標核心的 reschedule 旗標 */
    g_cpus[target_hartid].need_reschedule = 1;

    /* 如果目標不是自己，發送 IPI */
    if (target_hartid != smp_get_hartid()) {
        smp_send_ipi(target_hartid);
    }
}

void smp_send_ipi(uint64_t target_hartid)
{
    /* 寫入 CLINT 的 MSIP 暫存器 */
    volatile uint32_t *msip = CLINT_MSIP(target_hartid);
    *msip = 1;
}
```

### 4.3 IPI 處理

```c
void smp_handle_ipi(void)
{
    cpu_t *cpu = smp_get_cpu();

    /* 清除 IPI */
    smp_clear_ipi();

    /* 檢查是否需要重新排程 */
    if (cpu->need_reschedule) {
        cpu->need_reschedule = 0;
        sched_request_switch();
    }
}
```

---

## 五、完整的 sem_post 流程

讓我們追蹤一個完整的 `sem_post` 流程：

### 5.1 場景設定

```
Core 0: Task A 執行中，呼叫 sys_sem_post(&sem)
Core 1: Task B 在 sem 上等待
```

### 5.2 流程圖

```
Core 0                                  Core 1
  │                                       │
  │ Task A: sys_sem_post(&sem)            │ Task B: blocked on sem
  │   ↓                                   │
  │ ecall                                 │
  ▼                                       │
┌─────────────────────────────────────────────────────────────┐
│ trap_entry → trap_handler → syscall_dispatch                │
│   → sem_post_internal(&sem)                                 │
└─────────────────────────────────────────────────────────────┘
  │                                       │
  ▼                                       │
┌─────────────────────────────────────────────────────────────┐
│ spinlock_lock(&sem->lock)                                   │
│ task = sem->wait_list  (= Task B)                           │
│ sem->wait_list = task->next                                 │
│ spinlock_unlock(&sem->lock)                                 │
└─────────────────────────────────────────────────────────────┘
  │                                       │
  ▼                                       │
┌─────────────────────────────────────────────────────────────┐
│ task_make_ready(Task B)                                     │
│   - Task B 加入 ready queue                                 │
│   - Task B 可能在任何核心執行                               │
└─────────────────────────────────────────────────────────────┘
  │                                       │
  ▼                                       │
┌─────────────────────────────────────────────────────────────┐
│ 檢查 Task B 的 affinity                                     │
│ if (Task B 綁定 Core 1) {                                   │
│     smp_request_reschedule(1);  ─────────────────────────►  │
│ }                                       │                   │
└─────────────────────────────────────────────────────────────┘
  │                                       │
  │                                       ▼
  │                             ┌─────────────────────────────┐
  │                             │ IPI 中斷                    │
  │                             │ smp_handle_ipi()            │
  │                             │ sched_request_switch()      │
  │                             └─────────────────────────────┘
  │                                       │
  │                                       ▼
  │                             ┌─────────────────────────────┐
  │                             │ Scheduler 選擇 Task B       │
  │                             │ switch_to(current, Task B)  │
  │                             └─────────────────────────────┘
  │                                       │
  │                                       ▼
  │                                       │ Task B 執行中
  ▼                                       │
  │ Task A 繼續執行                        │
```

---

## 六、競爭條件分析

### 6.1 同時 wait 和 post

```
Core 0: sem_wait(&sem)          Core 1: sem_post(&sem)
        ↓                               ↓
   spinlock_lock(&sem->lock)       spinlock_lock(&sem->lock)
        ↓                               ↓
   (等待 Core 1 釋放)              (獲得鎖)
                                        ↓
                                   sem->count++ 或喚醒
                                        ↓
                                   spinlock_unlock()
        ↓
   (獲得鎖)
        ↓
   檢查 count 或阻塞
        ↓
   spinlock_unlock()
```

**結論**：spinlock 確保了操作的原子性，不會有競爭條件。

### 6.2 遺失喚醒問題

這是一個經典的 bug：

```c
/* 錯誤的實作 */
int sem_wait_bad(sem_t *sem)
{
    if (sem->count > 0) {
        sem->count--;
        return 0;
    }

    /* 問題：在這裡，另一個核心可能呼叫 sem_post */
    /* 但我們已經決定要阻塞了！ */

    task_block(current_task, sem);
    sched_schedule();
    return 0;
}
```

**正確的實作**：在持有 spinlock 的情況下做決定和阻塞。

---

## 七、效能考量

### 7.1 Spinlock 的 Overhead

| 操作 | 週期數（估計） |
|------|----------------|
| spinlock_lock (無競爭) | 10-20 cycles |
| spinlock_lock (有競爭) | 100-1000+ cycles |
| spinlock_unlock | 5-10 cycles |

### 7.2 減少競爭的策略

1. **Fine-Grained Locking**：每個物件有自己的鎖
2. **短臨界區**：持有鎖的時間越短越好
3. **避免巢狀鎖**：減少死鎖風險

### 7.3 IPI 的 Overhead

| 操作 | 週期數（估計） |
|------|----------------|
| 發送 IPI | 50-100 cycles |
| 接收 IPI | 100-200 cycles |
| 處理 IPI | 50-100 cycles |

**IPI 是昂貴的**，但對於即時性要求高的系統，這是必要的代價。

---

## 八、本章總結

多核 Syscall 的核心要點：

| 設計決策 | 理由 |
|----------|------|
| **無鎖 Dispatcher** | Dispatcher 不存取共享資料 |
| **Per-Object Locking** | 減少鎖競爭 |
| **IPI 通知** | 確保即時喚醒 |
| **在持有鎖時做決定** | 避免遺失喚醒 |

**經驗法則**：
- Syscall handler 內部需要適當的同步
- 使用 fine-grained locking 減少競爭
- 喚醒其他核心的 Task 時，考慮發送 IPI

下一章，我們將探討 **遺產處理**：當 Task 持有 Lock 時 fault，如何避免死鎖。

---

## 參考資料

**作業系統設計**

- Linux Kernel: Futex Implementation
- FreeRTOS SMP: Inter-Core Communication

**danieRTOS 系列**

- Ch 24-25: Spinlock 設計 (v2.x)
- Ch 26-27: IPI 機制 (v2.x)

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)
```

