# 遺產處理：當 Task 帶著 Lock 離開

**作者**: Danny Jiang
**日期**: 2025-12-26

---

## 前言：死人不會放手

想像這個場景：

```
Task A: mutex_lock(&shared_mutex)
        ... 正在處理 ...
        *invalid_ptr = 42;  ← Access Fault!
        ← Task A 被終止，但 mutex 還被它持有！

Task B: mutex_lock(&shared_mutex)
        ← 永遠等待...死鎖！
```

這是一個經典的問題：**當 Task 持有資源時異常終止，其他 Task 會死鎖**。

在 v1.x 的單核世界裡，這個問題相對簡單。但在 v3.x 的 SMP + User Mode 環境中，問題變得更加複雜：

- 多個核心可能同時存取同一個 mutex
- User Task 可能因為各種原因被終止（fault、syscall exit、被 kill）
- 資源可能分散在不同的同步物件中

這一章，我們要探討 danieRTOS v3.x 如何處理這個「遺產」問題。

---

## 一、問題分析

### 1.1 可能的終止原因

| 終止原因 | 觸發方式 | 可預期性 |
|----------|----------|----------|
| 正常退出 | `sys_exit()` | 可預期 |
| Access Fault | 存取無效記憶體 | 不可預期 |
| Illegal Instruction | 執行無效指令 | 不可預期 |
| 被其他 Task Kill | `task_kill(task)` | 不可預期 |
| Watchdog Timeout | 超時被終止 | 不可預期 |

### 1.2 可能持有的資源

| 資源類型 | 持有方式 | 影響 |
|----------|----------|------|
| Mutex | `mutex_lock()` | 其他 Task 死鎖 |
| Semaphore | `sem_wait()` | 資源計數錯誤 |
| Spinlock | `spinlock_lock()` | 系統死鎖（嚴重！） |
| 動態記憶體 | `malloc()` | 記憶體洩漏 |

---

## 二、設計策略

### 2.1 Resource Ownership Tracking

核心思想：**追蹤每個 Task 持有的資源，在 Task 終止時自動釋放**。

```c
typedef struct tcb {
    /* ... 基本欄位 ... */
    
    /* Resource Tracking */
    mutex_t *held_mutexes;    /* 持有的 mutex 鏈表 */
    /* 未來可擴展：
    sem_t *held_sems;
    void *held_memory;
    */
} tcb_t;
```

### 2.2 Mutex 加入 Owner 指針

```c
typedef struct mutex {
    spinlock_t lock;
    tcb_t *owner;           /* 持有者 */
    int32_t lock_count;     /* 遞迴計數 */
    tcb_t *wait_list;       /* 等待列表 */
    mutex_t *next_held;     /* 同一 Task 持有的下一個 mutex */
} mutex_t;
```

---

## 三、Mutex 實作

### 3.1 mutex_lock

```c
int mutex_lock(mutex_t *mutex)
{
    cpu_t *cpu = smp_get_cpu();
    tcb_t *task = cpu->current_task;
    
    reg_t state = spinlock_lock_irqsave(&mutex->lock);
    
    if (mutex->owner == NULL) {
        /* 無人持有，直接獲取 */
        mutex->owner = task;
        mutex->lock_count = 1;
        
        /* 加入 Task 的持有列表 */
        mutex->next_held = task->held_mutexes;
        task->held_mutexes = mutex;
        
        spinlock_unlock_irqrestore(&mutex->lock, state);
        return 0;
    }
    
    if (mutex->owner == task) {
        /* 遞迴鎖定 */
        mutex->lock_count++;
        spinlock_unlock_irqrestore(&mutex->lock, state);
        return 0;
    }
    
    /* 需要等待 */
    task->state = TASK_STATE_BLOCKED;
    task->blocked_on = mutex;
    task->next = mutex->wait_list;
    mutex->wait_list = task;
    sched_remove_ready(task);
    
    spinlock_unlock_irqrestore(&mutex->lock, state);
    sched_schedule();
    
    /* 被喚醒後，檢查是否因為 owner 死亡 */
    if (task->wake_reason == WAKE_REASON_OWNER_DEAD) {
        return -E_OWNER_DEAD;
    }
    
    return 0;
}
```

### 3.2 mutex_unlock

```c
int mutex_unlock(mutex_t *mutex)
{
    cpu_t *cpu = smp_get_cpu();
    tcb_t *task = cpu->current_task;
    
    reg_t state = spinlock_lock_irqsave(&mutex->lock);
    
    if (mutex->owner != task) {
        /* 不是持有者，錯誤 */
        spinlock_unlock_irqrestore(&mutex->lock, state);
        return -EPERM;
    }
    
    mutex->lock_count--;
    if (mutex->lock_count > 0) {
        /* 遞迴鎖定，還沒完全釋放 */
        spinlock_unlock_irqrestore(&mutex->lock, state);
        return 0;
    }
    
    /* 從 Task 的持有列表移除 */
    mutex_remove_from_held_list(task, mutex);
    
    /* 完全釋放 */
    if (mutex->wait_list != NULL) {
        /* 喚醒第一個等待者 */
        tcb_t *waiter = mutex->wait_list;
        mutex->wait_list = waiter->next;
        waiter->next = NULL;
        
        /* 轉移所有權 */
        mutex->owner = waiter;
        mutex->lock_count = 1;
        mutex->next_held = waiter->held_mutexes;
        waiter->held_mutexes = mutex;
        
        spinlock_unlock_irqrestore(&mutex->lock, state);
        task_make_ready(waiter);
    } else {
        mutex->owner = NULL;
        spinlock_unlock_irqrestore(&mutex->lock, state);
    }

    return 0;
}
```

---

## 四、Task 終止時的清理

### 4.1 task_cleanup_resources

```c
void task_cleanup_resources(tcb_t *task)
{
    /* 釋放所有持有的 mutex */
    while (task->held_mutexes != NULL) {
        mutex_t *mutex = task->held_mutexes;
        task->held_mutexes = mutex->next_held;

        reg_t state = spinlock_lock_irqsave(&mutex->lock);

        /* 標記 owner 已死亡 */
        mutex->owner = NULL;
        mutex->lock_count = 0;

        /* 喚醒所有等待者，通知 owner 已死亡 */
        while (mutex->wait_list != NULL) {
            tcb_t *waiter = mutex->wait_list;
            mutex->wait_list = waiter->next;
            waiter->next = NULL;
            waiter->wake_reason = WAKE_REASON_OWNER_DEAD;

            spinlock_unlock_irqrestore(&mutex->lock, state);
            task_make_ready(waiter);
            state = spinlock_lock_irqsave(&mutex->lock);
        }

        spinlock_unlock_irqrestore(&mutex->lock, state);
    }

    /* 未來：釋放其他資源 */
    /* task_cleanup_semaphores(task); */
    /* task_cleanup_memory(task); */
}
```

### 4.2 整合到 task_exit

```c
void task_exit(void)
{
    cpu_t *cpu = smp_get_cpu();
    tcb_t *task = cpu->current_task;

    reg_t state = critical_enter();

    /* 1. 清理持有的資源 */
    task_cleanup_resources(task);

    /* 2. 從 ready queue 移除 */
    sched_remove_ready(task);

    /* 3. 標記為已刪除 */
    task->state = TASK_STATE_DELETED;

    /* 4. 釋放 Stack（如果是動態分配的） */
    task_free_stacks(task);

    critical_exit(state);

    /* Scheduler 會切換到其他 Task */
}
```

### 4.3 Fault Handler 整合

```c
void handle_fault(context_t *ctx, uint64_t mcause, uint64_t mtval)
{
    cpu_t *cpu = smp_get_cpu();
    tcb_t *task = cpu->current_task;

    uart_printf("[Fault] Task '%s' faulted: mcause=%d, mtval=0x%lx\n",
                task->name, mcause, mtval);

    /* 檢查是否是 User Task */
    if (is_user_task(task)) {
        /* User Task fault：清理資源並終止 */
        task_cleanup_resources(task);
        task->state = TASK_STATE_DELETED;
        sched_remove_ready(task);

        /* 切換到其他 Task */
        sched_schedule();
    } else {
        /* Kernel Task fault：這是嚴重錯誤 */
        uart_puts("[PANIC] Kernel task faulted!\n");
        while (1) { asm volatile("wfi"); }
    }
}
```

---

## 五、E_OWNER_DEAD 機制

### 5.1 設計理念

當 mutex 的 owner 死亡時，等待者需要知道這個情況：

```c
/* 錯誤碼 */
#define E_OWNER_DEAD    100  /* 資源持有者已死亡 */

/* 喚醒原因 */
typedef enum {
    WAKE_REASON_NONE = 0,
    WAKE_REASON_TIMEOUT,
    WAKE_REASON_SIGNAL,
    WAKE_REASON_OWNER_DEAD,  /* 新增 */
} wake_reason_t;
```

### 5.2 使用範例

```c
void user_task(void *arg)
{
    int ret = sys_mutex_lock(&shared_mutex);

    if (ret == -E_OWNER_DEAD) {
        /* 前一個持有者死亡，mutex 狀態可能不一致 */
        sys_puts("[Warning] Previous owner died, reinitializing...\n");
        reinitialize_shared_data();
    }

    /* 正常處理 */
    process_shared_data();

    sys_mutex_unlock(&shared_mutex);
}
```

---

## 六、Spinlock 的特殊處理

### 6.1 為什麼 Spinlock 不能用同樣的方式？

Spinlock 是**不可阻塞**的：
- 持有 spinlock 時，中斷通常是禁用的
- 如果 Task 在持有 spinlock 時 fault，系統可能已經處於不一致狀態

### 6.2 設計決策

**danieRTOS v3.x 的策略**：
- User Task 不能直接使用 spinlock
- Spinlock 只在 Kernel 內部使用
- Kernel 程式碼經過審查，不應該 fault

```c
/* User Task 只能使用 mutex（透過 syscall） */
int sys_mutex_lock(mutex_t *mutex);
int sys_mutex_unlock(mutex_t *mutex);

/* Spinlock 只在 Kernel 內部使用 */
/* 不暴露給 User Task */
```

---

## 七、完整流程範例

### 7.1 場景

```
Task A: 持有 mutex，存取無效記憶體
Task B: 等待 mutex
Task C: 等待 mutex
```

### 7.2 流程

```
Time T1: Task A 持有 mutex
         Task B, C 在 mutex->wait_list

Time T2: Task A 存取無效記憶體
         ↓
         Access Fault (mcause = 13 or 15)
         ↓
         handle_fault()

Time T3: task_cleanup_resources(Task A)
         ↓
         mutex->owner = NULL
         ↓
         喚醒 Task B (wake_reason = OWNER_DEAD)
         喚醒 Task C (wake_reason = OWNER_DEAD)

Time T4: Task B 被排程
         mutex_lock() 返回 -E_OWNER_DEAD
         Task B 決定如何處理

Time T5: Task C 被排程
         mutex_lock() 返回 -E_OWNER_DEAD
         Task C 決定如何處理
```

---

## 八、進階議題

### 8.1 Priority Inheritance

當高優先級 Task 等待低優先級 Task 持有的 mutex 時，可能發生 Priority Inversion。

**解決方案**：Priority Inheritance

```c
int mutex_lock_with_pi(mutex_t *mutex)
{
    tcb_t *task = task_get_current();

    if (mutex->owner != NULL &&
        mutex->owner->priority < task->priority) {
        /* 提升 owner 的優先級 */
        mutex->owner->priority = task->priority;
        /* 如果 owner 在 ready queue，需要重新排序 */
        sched_update_priority(mutex->owner);
    }

    /* ... 正常的 lock 邏輯 ... */
}
```

### 8.2 Deadlock Detection

```c
/* 簡單的死鎖檢測：檢查等待環 */
bool detect_deadlock(tcb_t *task, mutex_t *mutex)
{
    tcb_t *owner = mutex->owner;

    while (owner != NULL) {
        if (owner == task) {
            /* 發現環！死鎖！ */
            return true;
        }

        /* 檢查 owner 是否在等待其他 mutex */
        if (owner->state == TASK_STATE_BLOCKED &&
            owner->blocked_on != NULL) {
            mutex_t *blocked_mutex = (mutex_t *)owner->blocked_on;
            owner = blocked_mutex->owner;
        } else {
            break;
        }
    }

    return false;
}
```

---

## 九、本章總結

遺產處理的核心要點：

| 設計決策 | 理由 |
|----------|------|
| **Resource Ownership Tracking** | 追蹤每個 Task 持有的資源 |
| **Mutex 加入 owner 指針** | 知道誰持有 mutex |
| **task_cleanup_resources** | Task 終止時自動釋放資源 |
| **E_OWNER_DEAD 機制** | 通知等待者 owner 已死亡 |
| **Spinlock 不暴露給 User** | 避免不可恢復的死鎖 |

**經驗法則**：
- 任何可能被 User Task 持有的資源，都需要追蹤
- Task 終止時，必須清理所有持有的資源
- 等待者需要知道 owner 死亡的情況

下一章，我們將進行 **整合測試和 Demo**，驗證 v3.x 的完整功能。

---

## 參考資料

**作業系統設計**

- Linux Kernel: Robust Futex
- POSIX: pthread_mutex_consistent()

**danieRTOS 系列**

- Ch 10: Mutex 基礎 (v0.x)
- Ch 35: 多核 Syscall (v3.x)

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)
```

