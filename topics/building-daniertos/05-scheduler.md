# Scheduler 實作

**作者**: Danny Jiang
**日期**: 2025-12-13

---

## 前言：RTOS 的大腦

你有沒有想過，當你的電腦同時運行 10 個程式時，作業系統是怎麼決定「現在該讓誰跑」的？

這個問題看起來很簡單，但當我第一次實作 Scheduler 時，才發現裡面的學問比我想像的深得多。

我最初的版本是這樣的：每次 Timer Interrupt 來，就掃描所有 Task，找出優先級最高的那個。聽起來合理吧？

結果呢？當我有 8 個 Task 時，每次排程都要掃描 8 次。如果 Timer 每 1ms 觸發一次，一秒就是 8000 次比較。**在真實的嵌入式系統中，這種浪費是不能接受的。**

後來我看了 FreeRTOS 的做法，恍然大悟：用一個 **Priority Array of Lists**，就可以把複雜度降到 O(1)。這個「找到最高優先級 Task」的操作，從「掃描」變成「查表」，快了不知道多少倍。

如果說 Context Switch 是 RTOS 的「心臟」，那麼 Scheduler 就是 RTOS 的「大腦」。

心臟負責機械性地「保存」和「恢復」Context，它不需要思考。但大腦要做決策：

- **什麼時候**該切換 Task？
- **切換給誰**？
- 如果有多個 Task 都想執行，**誰先執行**？

這些問題的答案，就是 **Scheduling Algorithm（排程演算法）**。

對於 danieRTOS，我們選擇最經典的組合：

**Priority-based Preemptive Scheduling + Round-Robin Time Slicing**

只有兩條規則：

1. **永遠執行優先級最高的 Task**
2. **同優先級的 Task 輪流執行**

就這樣。沒有複雜的數學，沒有花俏的演算法，只有兩條簡單的規則。

本文將實作 danieRTOS 的 Scheduler。讀完這篇文章，你將能夠：

- 理解 Priority-based Preemptive Scheduling 的原理
- 設計高效的 Ready List 資料結構
- 實作 `schedule()` 函數
- 理解 Idle Task 的必要性

> 💡 **相關閱讀**：如果你對 Linked List 的實作不熟悉，可以參考《Data Structures in Practice》系列。

---

## 一、排程演算法

### 1.1 Priority-based Preemptive Scheduling

這個名字有三個關鍵字：

**1. Priority-based（基於優先級）**

每個 Task 都有一個優先級數字。在 danieRTOS 中，**數字越大，優先級越高**。

```c
#define PRIORITY_IDLE    0   // 最低，只有 Idle Task 使用
#define PRIORITY_LOW     1
#define PRIORITY_NORMAL  2
#define PRIORITY_HIGH    3
#define PRIORITY_URGENT  4   // 最高
```

Scheduler 永遠會選擇「Ready List 中優先級最高的 Task」來執行。

**2. Preemptive（搶佔式）**

如果一個高優先級的 Task 變成 Ready（例如：被中斷喚醒），Scheduler 會**立刻**打斷當前執行的低優先級 Task，切換到高優先級 Task。

這就是「搶佔」——高優先級 Task 可以「搶走」CPU。

**與 Cooperative Scheduling 的對比**：

| 特性 | Preemptive | Cooperative |
|------|------------|-------------|
| 切換時機 | 任何時刻（中斷觸發） | Task 主動讓出 CPU |
| 即時性 | 高 | 低 |
| 複雜度 | 較高（需要完整 Context Save） | 較低 |
| 風險 | Race condition | 一個 Task 可能霸佔 CPU |

**3. Scheduling（排程）**

決定「誰來執行」的過程就叫排程。

### 1.2 Round-Robin Time Slicing

當有多個 Task 擁有相同的優先級時，該怎麼辦？

答案是 **Round-Robin（輪流執行）**。

想像一個旋轉木馬：每個人坐一圈，然後換下一個人。Time Slicing 的概念類似：每個 Task 執行一個「時間片」，然後切換到下一個同優先級的 Task。

**時間片（Time Slice）** = 一定數量的 Tick

在 danieRTOS 中，我們用最簡單的方式實作：每次 Tick Interrupt，如果有同優先級的其他 Task 在等待，就切換。

```
┌───────────────────────────────────────────────────────┐
│  Priority 3: [Task A] → [Task B] → [Task C] → ...    │
│                  ↑                                    │
│                  └──── Round-Robin 輪轉 ────┘         │
└───────────────────────────────────────────────────────┘
│  Priority 2: [Task D]                                 │
│  Priority 1: [Task E] → [Task F]                      │
│  Priority 0: [Idle Task]                              │
└───────────────────────────────────────────────────────┘
```

**行為**：

- 如果 Priority 3 有 Task，只執行 Priority 3 的 Task（A、B、C 輪流）
- 只有當 Priority 3 的所有 Task 都 Blocked 時，才會執行 Priority 2
- 以此類推

---

## 二、Ready List 資料結構

Scheduler 最頻繁的操作是：「找到優先級最高的 Ready Task」。資料結構的選擇直接影響效率。

### 2.1 方案比較

**方案 1：單一陣列**

```c
tcb_t task_pool[MAX_TASKS];

tcb_t *find_highest_priority(void) {
    tcb_t *best = NULL;
    for (int i = 0; i < MAX_TASKS; i++) {
        if (task_pool[i].state == TASK_READY) {
            if (best == NULL || task_pool[i].priority > best->priority) {
                best = &task_pool[i];
            }
        }
    }
    return best;
}
```

- **優點**：實作簡單
- **缺點**：每次都要遍歷所有 Task，時間複雜度 O(N)

**方案 2：Priority Array of Lists（FreeRTOS 方式）**

```c
#define MAX_PRIORITY 8
tcb_t *ready_lists[MAX_PRIORITY];  // 每個優先級一個 List

tcb_t *find_highest_priority(void) {
    for (int prio = MAX_PRIORITY - 1; prio >= 0; prio--) {
        if (ready_lists[prio] != NULL) {
            return ready_lists[prio];
        }
    }
    return NULL;
}
```

- **優點**：時間複雜度 O(P)，其中 P 是優先級數量（通常很小）
- **缺點**：需要維護多個 List

**方案 3：Priority Array + Bitmap（進階優化）**

```c
uint8_t ready_bitmap;  // 每個 bit 代表一個優先級是否有 Task

tcb_t *find_highest_priority(void) {
    if (ready_bitmap == 0) return NULL;
    int prio = 31 - __builtin_clz(ready_bitmap);  // Count Leading Zeros
    return ready_lists[prio];
}
```

- **優點**：時間複雜度 O(1)
- **缺點**：需要硬體 CLZ 指令支援

**danieRTOS 選擇方案 2**：簡單且效率足夠。對於 8 個優先級，最多只需要 8 次比較。

### 2.2 Linked List 實作

每個優先級的 Ready List 是一個**雙向鏈結串列**：

```c
// 雙向鏈結串列節點（內嵌在 TCB 中）
typedef struct tcb {
    volatile uint64_t *sp;
    uint32_t priority;
    uint32_t state;
    char name[16];

    struct tcb *next;  // 同優先級的下一個 Task
    struct tcb *prev;  // 同優先級的上一個 Task
} tcb_t;

// Ready List 陣列
#define MAX_PRIORITY 8
tcb_t *ready_lists[MAX_PRIORITY] = {NULL};
```

### 2.3 List 操作

**加入 List（插入到尾端，支援 Round-Robin）**

```c
void ready_list_add(tcb_t *tcb) {
    uint32_t prio = tcb->priority;

    if (ready_lists[prio] == NULL) {
        // List 為空，直接設為 head
        ready_lists[prio] = tcb;
        tcb->next = tcb;  // 環狀：指向自己
        tcb->prev = tcb;
    } else {
        // 插入到 head 之前（等於尾端）
        tcb_t *head = ready_lists[prio];
        tcb_t *tail = head->prev;

        tail->next = tcb;
        tcb->prev = tail;
        tcb->next = head;
        head->prev = tcb;
    }
}
```

**從 List 移除**

```c
void ready_list_remove(tcb_t *tcb) {
    uint32_t prio = tcb->priority;

    if (tcb->next == tcb) {
        // List 只有一個元素
        ready_lists[prio] = NULL;
    } else {
        // 從鏈中移除
        tcb->prev->next = tcb->next;
        tcb->next->prev = tcb->prev;

        // 如果移除的是 head，更新 head
        if (ready_lists[prio] == tcb) {
            ready_lists[prio] = tcb->next;
        }
    }

    tcb->next = NULL;
    tcb->prev = NULL;
}
```

**Round-Robin 輪轉**

```c
void ready_list_rotate(uint32_t prio) {
    if (ready_lists[prio] != NULL && ready_lists[prio]->next != ready_lists[prio]) {
        // 把 head 往後移一個
        ready_lists[prio] = ready_lists[prio]->next;
    }
}
```

**為什麼用環狀串列？**

1. **簡化 Round-Robin**：輪轉只需要移動 head 指標
2. **無 NULL 檢查**：遍歷時不需要檢查是否到達尾端
3. **FreeRTOS 也這樣做**：這是經過驗證的設計

---

## 三、schedule() 函數

這是 Scheduler 的核心函數，在每次 Tick Interrupt 中被呼叫。

### 3.1 基本邏輯

```c
tcb_t *current_tcb = NULL;

void schedule(void) {
    tcb_t *old_tcb = current_tcb;
    tcb_t *new_tcb = NULL;

    // 1. 如果當前 Task 還是 Ready，進行 Round-Robin 輪轉
    if (old_tcb != NULL && old_tcb->state == TASK_READY) {
        ready_list_rotate(old_tcb->priority);
    }

    // 2. 找到最高優先級的 Ready Task
    for (int prio = MAX_PRIORITY - 1; prio >= 0; prio--) {
        if (ready_lists[prio] != NULL) {
            new_tcb = ready_lists[prio];
            break;
        }
    }

    // 3. 如果找不到（不應該發生，因為有 Idle Task）
    if (new_tcb == NULL) {
        danie_panic("No runnable task!");
    }

    // 4. 更新 current_tcb
    current_tcb = new_tcb;
}
```

### 3.2 與 Tick Handler 整合

```c
void handle_timer_interrupt(void) {
    // 1. 設定下一次 Timer 中斷
    CLINT_MTIMECMP += TICK_INTERVAL;

    // 2. 更新系統 Tick
    tick_count++;

    // 3. 處理 Blocked Task 的喚醒
    wake_blocked_tasks();

    // 4. 執行排程
    schedule();

    // 5. Context Switch 在 trap_handler 的 RESTORE_CONTEXT 中自動發生
}

void wake_blocked_tasks(void) {
    for (int i = 0; i < MAX_TASKS; i++) {
        tcb_t *t = &task_pool[i];
        if (t->state == TASK_BLOCKED && tick_count >= t->ticks_to_wake) {
            t->state = TASK_READY;
            ready_list_add(t);
        }
    }
}
```

### 3.3 Task 狀態轉換與 List 操作

**Task 創建**

```c
void task_create(tcb_t *tcb, void (*entry)(void), ...) {
    // 初始化 TCB...
    tcb->state = TASK_READY;
    ready_list_add(tcb);
}
```

**Task 進入 Blocked**

```c
void danie_delay(uint32_t ticks) {
    // 從 Ready List 移除
    ready_list_remove(current_tcb);

    // 設定喚醒時間
    current_tcb->state = TASK_BLOCKED;
    current_tcb->ticks_to_wake = tick_count + ticks;

    // 主動觸發排程
    schedule();
    trigger_context_switch();  // 觸發 Trap，強制 Context Switch
}
```

**Task 被喚醒**

```c
void wake_blocked_tasks(void) {
    // ... 如上所示 ...
    t->state = TASK_READY;
    ready_list_add(t);  // 加回 Ready List
}
```

---

## 四、Idle Task

### 4.1 為什麼需要 Idle Task？

想像一個場景：所有 Task 都在 Blocked 狀態（等待 delay 或事件）。這時候 Scheduler 找不到任何 Ready Task。

**CPU 應該做什麼？**

如果沒有 Idle Task，`find_highest_priority()` 會返回 NULL，系統會 Panic。

**Idle Task 是一個「永遠 Ready」的 Task**，它的優先級最低（0），只在沒有其他 Task 可以執行時才會運行。

### 4.2 Idle Task 的實作

```c
// Idle Task 的 Stack
static uint64_t idle_stack[256];  // 2KB
static tcb_t idle_tcb;

void idle_task(void) {
    while (1) {
        // 省電模式：等待下一個中斷
        asm volatile("wfi");
    }
}

void create_idle_task(void) {
    task_create(&idle_tcb, idle_task, idle_stack, sizeof(idle_stack), PRIORITY_IDLE);
    strncpy(idle_tcb.name, "idle", 16);
}
```

### 4.3 WFI 指令

`wfi`（Wait For Interrupt）是 RISC-V 的省電指令。執行後，CPU 進入低功耗狀態，直到下一個中斷發生。

這樣做的好處：

- **省電**：CPU 不會空轉
- **簡單**：不需要複雜的電源管理

### 4.4 Idle Task 的其他用途

在商業 RTOS 中，Idle Task 通常還會做一些「背景工作」：

- **Stack 使用量檢查**：檢測是否有 Task Stack Overflow
- **統計 CPU 使用率**：計算系統有多少時間在 Idle
- **背景清理**：釋放已刪除 Task 的資源

在 danieRTOS Phase 1，我們只做最基本的功能：`wfi` 等待中斷。

---

## 五、Preemption 的觸發點

Scheduler 只是「決定」下一個 Task 是誰，真正的 Context Switch 發生在 Trap Handler 中。

### 5.1 觸發點總結

| 觸發點 | 說明 |
|--------|------|
| **Timer Interrupt** | 每個 Tick 都會觸發 Scheduler |
| **Task 進入 Blocked** | 呼叫 `delay()` 或等待資源 |
| **Task 被喚醒** | 高優先級 Task 變成 Ready |
| **Task 主動讓出** | 呼叫 `yield()` |

### 5.2 yield() 實作

有時候 Task 想主動讓出 CPU，即使它的時間片還沒用完：

```c
void danie_yield(void) {
    // 觸發一個 Software Interrupt 或直接呼叫 schedule
    schedule();
    trigger_context_switch();
}
```

在 RISC-V 上，可以用 `ecall`（Environment Call）觸發一個同步 Trap，讓 Trap Handler 處理 Context Switch。

---

## 六、完整的 Scheduler 流程圖

```
┌─────────────────────────────────────────────────────────────┐
│                    Timer Interrupt                          │
└─────────────────────────────┬───────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    SAVE_CONTEXT                             │
│  (保存當前 Task 的所有暫存器到 Stack)                         │
└─────────────────────────────┬───────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 handle_timer_interrupt()                    │
│  1. 設定下一次 Timer                                         │
│  2. tick_count++                                            │
│  3. wake_blocked_tasks()                                    │
│  4. schedule()  ←── 決定 current_tcb                        │
└─────────────────────────────┬───────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   RESTORE_CONTEXT                           │
│  (從 current_tcb->sp 恢復暫存器)                             │
│  (可能是不同的 Task！)                                       │
└─────────────────────────────┬───────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                         mret                                │
│  (跳到 mepc 指向的地址，繼續執行 Task)                        │
└─────────────────────────────────────────────────────────────┘
```

---

## 總結

本文實作了 danieRTOS 的 Scheduler：

1. **排程演算法**：Priority-based Preemptive + Round-Robin
2. **資料結構**：Priority Array of Circular Doubly Linked Lists
3. **schedule() 函數**：輪轉同優先級 Task，選擇最高優先級
4. **Idle Task**：確保系統永遠有 Task 可以執行
5. **Preemption 觸發點**：Timer Interrupt、Blocked、Yield

現在我們有了完整的 Task 管理和排程機制。下一步是把所有東西整合起來，寫一個可以運行的 Demo。

---

## 參考資料

**RTOS 參考實作**

- **FreeRTOS Kernel - tasks.c**
  https://github.com/FreeRTOS/FreeRTOS-Kernel
  Priority Array of Lists 的參考實作。

**延伸閱讀**

- **Data Structures in Practice**
  Danny Jiang
  Linked List 的實作細節，用於 Ready List 管理。

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)
