# Delay 機制

**作者**: Danny Jiang
**日期**: 2025-12-13

---

## 前言：讓 Task 學會「睡覺」

你有沒有遇過那種「什麼都要自己盯著」的同事？

他不會設鬧鐘，不會用 Calendar 提醒，每次要等某件事完成，他就坐在那邊一直盯著螢幕看。即使等待的時間長達一小時，他也不會去做別的事。

在 RTOS 的世界裡，busy-wait 就是這樣的「盯著螢幕」：

```c
void delay_ms(int ms) {
    for (volatile int i = 0; i < ms * 1000; i++);
}
```

這種做法有兩個致命問題：

1. **浪費 CPU**：Task 在等待時霸佔 CPU，其他 Task 沒機會執行
2. **不精確**：迴圈次數和實際時間的關係很難精確控制

我第一次寫嵌入式程式時，就是用這種方法。結果呢？當我需要同時閃爍 LED 和讀取按鈕時，系統變得一團糟——LED 在 delay 時，按鈕完全沒反應。

**正確的做法是讓 Task 學會「睡覺」。**

就像設了鬧鐘的人，他可以安心去做別的事，時間到了鬧鐘會叫他。在 RTOS 中，這就是 `delay()` 的核心概念：Task 主動讓出 CPU，等時間到了再被喚醒。

> 💡 **關鍵洞察**：Delay 不是「什麼都不做」，而是「讓別人有機會做事」。這是 RTOS 多任務的精髓。

本文將實作 danieRTOS 的 Delay 機制。讀完這篇文章，你將能夠：

- 理解 Blocked 狀態的意義
- 實作 `danie_delay()` 和 `danie_delay_until()`
- 設計高效的 Delayed Task List

> 💡 **相關閱讀**：Delay 機制使用 Sorted Linked List 來管理等待的 Task。關於 Linked List 的詳細實作，可以參考《Data Structures in Practice》系列。

---

## 一、Blocked 狀態

### 1.1 Task 狀態回顧

在 Ch3 中，我們定義了 Task 的四種狀態：

```c
typedef enum {
    TASK_READY,      // 準備好執行
    TASK_RUNNING,    // 正在執行
    TASK_BLOCKED,    // 等待某個事件
    TASK_SUSPENDED   // 被暫停
} task_state_t;
```

**Blocked 狀態**是「Task 正在等待某個事件」的狀態。Task 主動說：「我現在沒事做，叫我起床再執行吧。」

### 1.2 Blocked vs Ready

| 特性 | Ready | Blocked |
|------|-------|---------|
| 是否在 Ready List | ✅ 是 | ❌ 否 |
| 會被 Scheduler 選中 | ✅ 會 | ❌ 不會 |
| 何時離開此狀態 | 被 Scheduler 選中執行 | 等待的事件發生 |

**關鍵理解**：Blocked Task 不會出現在 Ready List 中，所以 Scheduler 永遠不會選中它。它必須等到「喚醒事件」發生，被移回 Ready List，才有機會執行。

### 1.3 喚醒事件的類型

Task 可能因為各種原因進入 Blocked 狀態：

| 原因 | 喚醒事件 |
|------|----------|
| `delay(100)` | 100 個 Tick 過去 |
| `semaphore_take()` | Semaphore 被 give |
| `queue_receive()` | Queue 收到資料 |
| `event_wait()` | Event 被 set |

本章專注於**時間等待**（delay）。其他類型的等待會在後續章節討論。

---

## 二、danie_delay() 實作

### 2.1 基本邏輯

`danie_delay(ticks)` 的語意是：「從現在開始，睡 `ticks` 個 Tick。」

實作步驟：

1. 計算喚醒時間：`wake_time = tick_count + ticks`
2. 將 Task 從 Ready List 移除
3. 設定 Task 狀態為 BLOCKED
4. 將 Task 加入 Delayed List
5. 觸發排程（讓其他 Task 執行）

### 2.2 程式碼實作

```c
void danie_delay(uint32_t ticks) {
    // 不能在 ISR 中呼叫
    if (in_interrupt_context()) {
        danie_panic("delay() called from ISR!");
    }

    // 0 ticks 等於 yield
    if (ticks == 0) {
        danie_yield();
        return;
    }

    // 進入 Critical Section（暫時關閉中斷）
    critical_enter();

    // 1. 計算喚醒時間
    current_tcb->wake_time = tick_count + ticks;

    // 2. 從 Ready List 移除
    ready_list_remove(current_tcb);

    // 3. 設定狀態
    current_tcb->state = TASK_BLOCKED;

    // 4. 加入 Delayed List（依喚醒時間排序）
    delayed_list_add(current_tcb);

    // 5. 觸發排程
    schedule();

    // 離開 Critical Section
    critical_exit();

    // 返回時，表示 delay 時間已經過去
}
```

### 2.3 為什麼要 Critical Section？

想像以下場景：

```
Task A 正在執行 danie_delay():
    current_tcb->wake_time = tick_count + 100;
    ready_list_remove(current_tcb);
    ← Timer Interrupt 發生！
    ← Tick Handler 檢查 Delayed List
    ← 但 Task A 還沒加入 Delayed List！
    ← Task A 永遠不會被喚醒...
```

使用 Critical Section 可以確保整個操作是「原子性」的，不會被中斷打斷。

---

## 三、danie_delay_until() 實作

### 3.1 delay() vs delay_until()

**`danie_delay(100)`**：從「現在」開始睡 100 個 Tick。

問題：如果 Task 執行了 10 個 Tick，然後 delay 100 個 Tick，實際週期是 110 個 Tick。

```
執行時間軸：
|--執行 10--|-----delay 100-----|--執行 10--|-----delay 100-----|
            實際週期 = 110                   實際週期 = 110
```

**`danie_delay_until(&last_wake, 100)`**：確保「兩次喚醒之間」恰好是 100 個 Tick。

```
執行時間軸：
|--執行 10--|--delay 90--|--執行 10--|--delay 90--|
            精確週期 = 100           精確週期 = 100
```

### 3.2 使用場景

`delay_until()` 適用於需要**精確週期**的任務：

- 感測器取樣（每 10 ms 讀取一次）
- PID 控制（固定更新頻率）
- 通訊協定（固定心跳間隔）

### 3.3 程式碼實作

```c
void danie_delay_until(uint64_t *previous_wake_time, uint32_t period) {
    critical_enter();

    // 計算下一次喚醒時間
    uint64_t next_wake = *previous_wake_time + period;

    // 如果下一次喚醒時間已經過了（Task 執行太久），立即返回
    if (next_wake <= tick_count) {
        *previous_wake_time = tick_count;
        critical_exit();
        return;
    }

    // 計算需要睡多久
    uint32_t sleep_ticks = next_wake - tick_count;

    // 更新上次喚醒時間
    *previous_wake_time = next_wake;

    // 執行 delay
    current_tcb->wake_time = next_wake;
    ready_list_remove(current_tcb);
    current_tcb->state = TASK_BLOCKED;
    delayed_list_add(current_tcb);
    schedule();

    critical_exit();
}
```

### 3.4 使用範例

```c
void sensor_task(void) {
    uint64_t last_wake = tick_count;  // 初始化

    while (1) {
        // 每 100 個 Tick（100 ms）執行一次
        danie_delay_until(&last_wake, 100);

        // 讀取感測器
        int value = read_sensor();
        process(value);
    }
}
```

---

## 四、Delayed List 管理

### 4.1 設計目標

在 Tick Handler 中，我們需要快速檢查「有沒有 Task 該醒了」。

**樸素做法**：遍歷所有 Blocked Task

```c
void wake_delayed_tasks(void) {
    for (int i = 0; i < MAX_TASKS; i++) {
        if (task_pool[i].state == TASK_BLOCKED &&
            task_pool[i].wake_time <= tick_count) {
            wake_task(&task_pool[i]);
        }
    }
}
```

問題：時間複雜度 O(N)，每個 Tick 都要遍歷所有 Task。

**優化做法**：使用 **Sorted List**

### 4.2 Sorted Delayed List

把 Delayed Task 依「喚醒時間」排序：

```
Delayed List:
Head → [wake: 105] → [wake: 200] → [wake: 350] → NULL
        Task A        Task B        Task C
```

**插入**：O(N)，需要找到正確位置
**檢查喚醒**：O(1)，只需要檢查 Head

### 4.3 實作

**TCB 擴展**

```c
typedef struct tcb {
    volatile uint64_t *sp;
    uint32_t priority;
    uint32_t state;
    char name[16];

    struct tcb *next;
    struct tcb *prev;

    uint64_t wake_time;  // 喚醒時間
} tcb_t;
```

**Delayed List**

```c
tcb_t *delayed_list_head = NULL;

void delayed_list_add(tcb_t *tcb) {
    // 找到正確的插入位置（依 wake_time 排序）
    tcb_t *prev = NULL;
    tcb_t *curr = delayed_list_head;

    while (curr != NULL && curr->wake_time <= tcb->wake_time) {
        prev = curr;
        curr = curr->next;
    }

    // 插入
    tcb->next = curr;
    tcb->prev = prev;

    if (prev != NULL) {
        prev->next = tcb;
    } else {
        delayed_list_head = tcb;
    }

    if (curr != NULL) {
        curr->prev = tcb;
    }
}

void delayed_list_remove(tcb_t *tcb) {
    if (tcb->prev != NULL) {
        tcb->prev->next = tcb->next;
    } else {
        delayed_list_head = tcb->next;
    }

    if (tcb->next != NULL) {
        tcb->next->prev = tcb->prev;
    }

    tcb->next = NULL;
    tcb->prev = NULL;
}
```

### 4.4 高效的喚醒檢查

因為 List 是排序的，檢查變得非常快：

```c
void wake_delayed_tasks(void) {
    while (delayed_list_head != NULL &&
           delayed_list_head->wake_time <= tick_count) {

        tcb_t *task = delayed_list_head;

        // 從 Delayed List 移除
        delayed_list_remove(task);

        // 加回 Ready List
        task->state = TASK_READY;
        ready_list_add(task);
    }
}
```

**時間複雜度**：O(K)，其中 K 是「這個 Tick 需要喚醒的 Task 數量」。在大多數情況下，K 是 0 或 1。

---

## 五、完整流程圖

```
┌─────────────────────────────────────────────────────────────┐
│              Task A 呼叫 danie_delay(100)                   │
└─────────────────────────────┬───────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  1. wake_time = tick_count + 100 = 1000 + 100 = 1100        │
│  2. 從 Ready List 移除                                       │
│  3. state = BLOCKED                                          │
│  4. 加入 Delayed List                                        │
│  5. schedule() → 切換到 Task B                               │
└─────────────────────────────┬───────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              ... 時間流逝 ...                                │
│              Task B, C, D 輪流執行                           │
│              tick_count: 1000 → 1050 → 1100                  │
└─────────────────────────────┬───────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│          Tick Handler @ tick_count = 1100                   │
│  wake_delayed_tasks():                                       │
│    delayed_list_head->wake_time = 1100 <= tick_count        │
│    → 喚醒 Task A！                                           │
│    → Task A 加回 Ready List                                  │
└─────────────────────────────┬───────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  schedule():                                                 │
│    如果 Task A 優先級最高 → current_tcb = Task A             │
│  RESTORE_CONTEXT → Task A 繼續執行                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 六、API 設計考量

### 6.1 函數命名

FreeRTOS 的命名習慣是 `vTaskDelay()`、`xTaskDelayUntil()`。

danieRTOS 使用更簡潔的命名：

```c
void danie_delay(uint32_t ticks);
void danie_delay_until(uint64_t *previous_wake_time, uint32_t period);

// 便利函數
void danie_delay_ms(uint32_t ms);
```

### 6.2 便利函數

```c
#define TICKS_PER_MS (TICK_RATE_HZ / 1000)

void danie_delay_ms(uint32_t ms) {
    danie_delay(ms * TICKS_PER_MS);
}
```

### 6.3 錯誤處理

```c
void danie_delay(uint32_t ticks) {
    // 不能在 ISR 中呼叫
    if (in_interrupt_context()) {
        danie_panic("delay() called from ISR!");
    }

    // 不能在 Scheduler 啟動前呼叫
    if (current_tcb == NULL) {
        danie_panic("delay() called before scheduler started!");
    }

    // ...
}
```

---

## 總結

本文實作了 danieRTOS 的 Delay 機制：

1. **Blocked 狀態**：Task 主動讓出 CPU，等待喚醒
2. **danie_delay()**：相對延遲，從現在開始睡指定時間
3. **danie_delay_until()**：絕對週期，確保精確的執行頻率
4. **Delayed List**：使用 Sorted List 實現 O(1) 喚醒檢查

現在 Task 可以「睡覺」了！但有時候 Task 之間需要協調——例如，一個 Task 產生資料，另一個 Task 消費資料。這就需要 **Inter-Process Communication (IPC)**。

---

## 參考資料

**RTOS 參考實作**

- **FreeRTOS Kernel - tasks.c**
  https://github.com/FreeRTOS/FreeRTOS-Kernel
  vTaskDelay() 和 vTaskDelayUntil() 的參考實作。

**延伸閱讀**

- **Data Structures in Practice**
  Danny Jiang
  Sorted Linked List 的實作，用於 Delayed List 的高效管理。

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)
