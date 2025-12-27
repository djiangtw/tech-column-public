# 多核心排程器：SMP Scheduler

**作者**: Danny Jiang
**日期**: 2025-12-14

---

## 前言：從單核心到多核心

在 v0.x 的單核心版本中，我們的 Scheduler 很簡單：

```c
tcb_t *schedule(void) {
    // 找到最高優先權的 Ready Task
    for (int prio = MAX_PRIORITY - 1; prio >= 0; prio--) {
        if (ready_queue[prio] != NULL) {
            return ready_queue[prio];
        }
    }
    return idle_task;
}
```

但在多核心環境中，這個設計有幾個問題：

1. **Ready Queue 是共享資源**：多個核心同時存取，需要保護
2. **每個核心需要自己的 Current Task**：不能用全域變數
3. **排程決策更複雜**：要考慮 CPU Affinity、負載平衡

這一章，我們將設計 danieRTOS v2.x 的 SMP Scheduler。

---

## 一、設計選擇：Global Queue vs Per-Core Queue

### 1.1 兩種設計

**Global Queue（全域佇列）**：

```
┌─────────────────────────────────────────────────────────────┐
│                    Global Ready Queue                        │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                    │
│  │ T1  │→│ T2  │→│ T3  │→│ T4  │→│ T5  │                    │
│  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘                    │
└─────────────────────────────────────────────────────────────┘
      ↑                                   ↑
   Core 0                              Core 1
   取 Task                             取 Task
```

**Per-Core Queue（每核心佇列）**：

```
┌─────────────────────────┐  ┌─────────────────────────┐
│   Core 0 Ready Queue    │  │   Core 1 Ready Queue    │
│  ┌─────┐ ┌─────┐        │  │  ┌─────┐ ┌─────┐        │
│  │ T1  │→│ T2  │        │  │  │ T3  │→│ T4  │        │
│  └─────┘ └─────┘        │  │  └─────┘ └─────┘        │
└─────────────────────────┘  └─────────────────────────┘
      ↑                            ↑
   Core 0                       Core 1
```

### 1.2 優缺點比較

| 特性 | Global Queue | Per-Core Queue |
|------|--------------|----------------|
| **實作複雜度** | 簡單 | 複雜（需要 Work Stealing）|
| **負載平衡** | 自動 | 需要額外機制 |
| **鎖競爭** | 高（所有核心競爭同一個鎖）| 低（各自的鎖）|
| **Cache 效率** | 差（Task 可能在不同核心間跳動）| 好（Task 傾向留在同一核心）|
| **適用場景** | 2-4 核心 | 8+ 核心 |

### 1.3 danieRTOS 的選擇

對於 2-4 核心的嵌入式系統，我們選擇 **Global Queue**：

1. **簡單**：不需要實作 Work Stealing
2. **自動負載平衡**：Task 自然分散到各核心
3. **足夠好**：對於小型 SMP 系統，鎖競爭不是瓶頸

---

## 二、資料結構

### 2.1 擴展 TCB

```c
typedef struct tcb {
    // 原有欄位
    reg_t *sp;
    task_state_t state;
    priority_t priority;
    char name[16];

    // SMP 新增欄位
    uint32_t affinity_mask;    // CPU Affinity（位元遮罩）
    int32_t running_on;        // 正在哪個核心執行（-1 = 不在執行）
    struct tcb *next;          // Ready Queue 鏈結
} tcb_t;
```

### 2.2 CPU Affinity

`affinity_mask` 是一個位元遮罩，表示 Task 可以在哪些核心執行：

```c
#define AFFINITY_CORE_0     (1 << 0)  // 0b0001
#define AFFINITY_CORE_1     (1 << 1)  // 0b0010
#define AFFINITY_CORE_2     (1 << 2)  // 0b0100
#define AFFINITY_CORE_3     (1 << 3)  // 0b1000
#define AFFINITY_ALL        0xFFFFFFFF  // 可在任何核心執行

// 檢查 Task 是否可以在指定核心執行
static inline bool task_can_run_on(tcb_t *task, int hartid) {
    return (task->affinity_mask & (1 << hartid)) != 0;
}
```

### 2.3 Global Ready Queue

```c
// 每個優先權一個佇列
static tcb_t *g_ready_queue[MAX_PRIORITY];

// 保護 Ready Queue 的 Spinlock
static spinlock_t g_sched_lock;
```

---

## 三、SMP Scheduler 核心

### 3.1 初始化

```c
void sched_init(void) {
    spinlock_init(&g_sched_lock);

    for (int i = 0; i < MAX_PRIORITY; i++) {
        g_ready_queue[i] = NULL;
    }

### 3.3 選擇下一個 Task

這是 SMP Scheduler 的核心邏輯：

```c
static tcb_t *select_next_task(int hartid) {
    // 從最高優先權開始搜尋
    for (int prio = MAX_PRIORITY - 1; prio >= 0; prio--) {
        tcb_t **pp = &g_ready_queue[prio];

        while (*pp != NULL) {
            tcb_t *task = *pp;

            // 檢查 CPU Affinity
            if (task_can_run_on(task, hartid)) {
                // 從佇列中移除
                *pp = task->next;
                task->next = NULL;
                return task;
            }

            pp = &(*pp)->next;
        }
    }

    // 沒有可執行的 Task，返回 Idle Task
    return get_cpu()->idle_task;
}
```

**注意**：這個函式必須在持有 `g_sched_lock` 的情況下呼叫。

### 3.4 排程函式

```c
reg_t *sched_schedule(reg_t *current_sp) {
    spinlock_acquire(&g_sched_lock);

    cpu_t *cpu = get_cpu();
    tcb_t *current = cpu->current_task;
    tcb_t *next;

    // 1. 保存當前 Task 的 SP
    if (current != NULL) {
        current->sp = current_sp;
        current->running_on = -1;

        // 如果還是 Ready 狀態，放回佇列
        if (current->state == TASK_STATE_RUNNING) {
            current->state = TASK_STATE_READY;

            // 加到佇列尾端（Round-Robin）
            priority_t prio = current->priority;
            tcb_t **pp = &g_ready_queue[prio];
            while (*pp != NULL) {
                pp = &(*pp)->next;
            }
            *pp = current;
            current->next = NULL;
        }
    }

    // 2. 選擇下一個 Task
    next = select_next_task(cpu->hartid);

    // 3. 更新狀態
    next->state = TASK_STATE_RUNNING;
    next->running_on = cpu->hartid;
    cpu->current_task = next;

    spinlock_release(&g_sched_lock);

    return next->sp;
}
```

### 3.5 Yield

```c
void sched_yield(void) {
    // 觸發 Software Interrupt 來進行排程
    // 這樣可以確保在 Trap Handler 中進行 Context Switch
    asm volatile("csrs mip, %0" : : "r"(1 << 3));  // 設定 MSIP
}
```

或者直接呼叫排程：

```c
reg_t *sched_yield_from_trap(reg_t *current_sp) {
    return sched_schedule(current_sp);
}
```

---

## 四、跨核心排程

### 4.1 問題：喚醒其他核心的 Task

考慮這個場景：

1. Core 0 正在執行 Task A（優先權 5）
2. Core 1 正在執行 Task B（優先權 5）
3. Core 0 的 Timer 中斷喚醒了 Task C（優先權 10，綁定到 Core 1）

Task C 應該立即在 Core 1 執行，但 Core 1 不知道有新的高優先權 Task！

### 4.2 解決方案：IPI 通知

```c
void task_wake(tcb_t *task) {
    spinlock_acquire(&g_sched_lock);

    // 加入 Ready Queue
    task->state = TASK_STATE_READY;
    priority_t prio = task->priority;
    tcb_t **pp = &g_ready_queue[prio];
    while (*pp != NULL) {
        pp = &(*pp)->next;
    }
    *pp = task;
    task->next = NULL;

    // 檢查是否需要通知其他核心
    for (int i = 0; i < MAX_CORES; i++) {
        cpu_t *cpu = &g_cpus[i];

        // 跳過自己
        if (i == get_hartid()) continue;

        // 檢查 Affinity
        if (!task_can_run_on(task, i)) continue;

        // 如果這個核心的當前 Task 優先權較低
        if (cpu->current_task->priority < task->priority) {
            // 發送 IPI
            smp_request_reschedule(i);
            break;  // 只需要通知一個核心
        }
    }

    spinlock_release(&g_sched_lock);

    // 也檢查自己是否需要讓出 CPU
    if (get_current_task()->priority < task->priority) {
        sched_yield();
    }
}
```

### 4.3 IPI Handler 中的排程

```c
void ipi_handler(void) {
    ipi_clear();

    cpu_t *cpu = get_cpu();
    uint32_t flags = __atomic_exchange_n(&cpu->ipi_flags, 0, __ATOMIC_ACQUIRE);

    if (flags & IPI_RESCHEDULE) {
        // 標記需要排程，在 Trap 返回時處理
        sched_request_switch();
    }

    // ... 其他 IPI 類型
}
```

---

## 五、Timer 中斷處理

### 5.1 每個核心獨立的 Timer

在 SMP 系統中，每個核心有自己的 Timer Compare 暫存器：

```c
#define CLINT_MTIMECMP(hart) (*(volatile uint64_t *)(CLINT_BASE + 0x4000 + (hart) * 8))

void timer_handler(void) {
    uint64_t hartid = get_hartid();

    // 設定下一次中斷
    CLINT_MTIMECMP(hartid) += TICK_INTERVAL;

    // 只有 Hart 0 更新全域 Tick
    if (hartid == 0) {
        g_tick_count++;

        // 檢查 Sleep Queue
        tick_check_sleeping_tasks();
    }

    // 每個核心都檢查是否需要排程
    cpu_t *cpu = get_cpu();
    tcb_t *current = cpu->current_task;

    if (current->time_slice > 0) {
        current->time_slice--;
    }

    if (current->time_slice == 0) {
        sched_request_switch();
    }
}
```

### 5.2 Time Slice

每個 Task 有一個時間片：

```c
typedef struct tcb {
    // ...
    uint32_t time_slice;       // 剩餘時間片
    uint32_t time_slice_max;   // 時間片最大值
} tcb_t;

// 在排程時重置時間片
void sched_schedule(...) {
    // ...
    next->time_slice = next->time_slice_max;
    // ...
}
```

---

## 六、完整的 Trap Handler

### 6.1 trap.S

```asm
.global trap_handler
.align 4

trap_handler:
    # 保存 Context
    SAVE_CONTEXT

    # 呼叫 C Handler
    csrr a0, mcause
    mv   a1, sp
    call trap_handler_c

    # a0 = 新的 SP（可能是不同的 Task）
    mv   sp, a0

    # 恢復 Context
    RESTORE_CONTEXT

    mret
```

### 6.2 trap.c

```c
reg_t *trap_handler_c(reg_t mcause, reg_t *sp) {
    cpu_t *cpu = get_cpu();
    cpu->irq_nest_depth++;

    if (mcause & MCAUSE_INTERRUPT) {
        reg_t cause = mcause & 0xFF;

        switch (cause) {
        case 7:  // Timer
            timer_handler();
            break;
        case 3:  // Software (IPI)
            ipi_handler();
            break;
        case 11: // External
            plic_handler();
            break;
        }
    } else {
        exception_handler(mcause, sp);
    }

    cpu->irq_nest_depth--;

    // 只在最外層中斷返回時進行排程
    if (cpu->irq_nest_depth == 0 && sched_switch_pending()) {
        sp = sched_schedule(sp);
    }

    return sp;
}
```

---

## 七、啟動流程

### 7.1 Primary Core

```c
void main(void) {
    // 1. 初始化硬體
    uart_init();

    // 2. 初始化 Scheduler
    sched_init();

    // 3. 建立 Idle Task（每個核心一個）
    for (int i = 0; i < MAX_CORES; i++) {
        g_cpus[i].idle_task = task_create_idle(i);
    }

    // 4. 建立應用程式 Task
    task_create("Task1", task1_func, 5, AFFINITY_ALL);
    task_create("Task2", task2_func, 5, AFFINITY_ALL);

    // 5. 初始化 Per-Core 資料
    cpu_init(0);

    // 6. 通知 Secondary Cores 啟動
    smp_boot_flag = 1;
    fence_w_r();

    // 7. 啟動 Scheduler
    sched_start();
}
```

### 7.2 Secondary Core

```c
void secondary_main(void) {
    uint64_t hartid = get_hartid();

    // 1. 初始化 Per-Core 資料
    cpu_init(hartid);

    // 2. 初始化 Timer
    timer_init();

    // 3. 初始化 IPI
    ipi_init();

    // 4. 啟動 Scheduler
    sched_start();
}
```

---

## 八、本章回顧

在這一章，我們設計了 SMP Scheduler：

1. **Global Queue**：簡單且自動負載平衡
2. **CPU Affinity**：控制 Task 可以在哪些核心執行
3. **Spinlock 保護**：確保 Ready Queue 的一致性
4. **IPI 通知**：跨核心排程通知
5. **Per-Core Timer**：每個核心獨立的時間片管理

關鍵設計決策：

| 決策 | 選擇 | 原因 |
|------|------|------|
| Queue 類型 | Global | 簡單，適合 2-4 核心 |
| 鎖類型 | Spinlock | 臨界區短，不能睡眠 |
| 排程觸發 | IPI | 即時通知其他核心 |

下一章，我們將實作 **SMP Mutex**——支援多核心的互斥鎖。

---

## 參考資料

- **FreeRTOS SMP**, Symmetric Multiprocessing Support
- **Linux Kernel**, Completely Fair Scheduler (CFS)
- **Tech Column**, RTOS SMP 系列

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。
