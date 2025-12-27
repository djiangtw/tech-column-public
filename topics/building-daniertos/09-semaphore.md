# Semaphore 實作

**作者**: Danny Jiang
**日期**: 2025-12-13

---

## 前言：來自 1965 年的天才發明

1965 年，荷蘭電腦科學家 Edsger Dijkstra 發明了 Semaphore。

是的，就是那個發明最短路徑演算法的 Dijkstra。他不只會找最短路，還解決了多任務系統中的同步問題。

Dijkstra 的天才之處在於，他把複雜的同步問題抽象成一個極簡的模型：**一個計數器，加上兩個操作**。

他用荷蘭文命名這兩個操作：**P**（Proberen，嘗試）和 **V**（Verhogen，增加）。這個命名困擾了幾代程式設計師——「P 和 V 到底是什麼意思？」——但概念本身簡單到優雅。

六十年後的今天，Semaphore 仍然是每個作業系統和 RTOS 的核心元件。

到目前為止，我們的 Task 只能「各做各的」。如果一個 Task 需要等待另一個 Task 完成某件事，該怎麼辦？

想像一個場景：

- **Task A**：讀取感測器，把資料放到緩衝區
- **Task B**：處理緩衝區的資料

Task B 不能一直詢問「資料準備好了嗎？」（這叫 **Polling**，會浪費 CPU）。更好的方式是：Task B 先睡覺，等 Task A 準備好資料後，再把 Task B 叫醒。

這就是 **Semaphore（訊號量）** 的用途：讓 Task 可以「等待」某個事件，並在事件發生時被「喚醒」。

> 💡 **歷史小知識**：Dijkstra 後來說，如果他知道 Semaphore 會被全世界使用，他一定會用英文命名。但 P 和 V 已經成為經典，出現在無數教科書中。

---

## 一、Semaphore 的概念

### 1.1 什麼是 Semaphore？

Semaphore 是一個**計數器**，支援兩個操作：

- **Take（P 操作）**：嘗試取得資源。如果計數器 > 0，減一並繼續；否則等待
- **Give（V 操作）**：釋放資源。計數器加一，並喚醒等待的 Task

```
Semaphore 狀態：count = 1

Task A: take() → count = 0, 繼續執行
Task B: take() → count = 0, 等待...
Task A: give() → count = 1, 喚醒 Task B
Task B: 被喚醒，count = 0, 繼續執行
```

### 1.2 Binary vs Counting

**Binary Semaphore**：計數器只能是 0 或 1。

- 用途：事件通知（「資料準備好了」）
- 特點：Give 多次不會累積（count 最多 = 1）

**Counting Semaphore**：計數器可以是任意正整數。

- 用途：資源池（例如：5 個 UART 通道）
- 特點：Give 幾次就累積幾個

---

## 二、資料結構設計

### 2.1 Semaphore 結構

```c
typedef struct {
    volatile int32_t count;      // 計數器
    int32_t max_count;           // 最大值（Binary = 1）
    tcb_t *waiting_list_head;    // 等待中的 Task（FIFO）
} semaphore_t;
```

### 2.2 初始化

```c
void semaphore_init(semaphore_t *sem, int32_t initial, int32_t max) {
    sem->count = initial;
    sem->max_count = max;
    sem->waiting_list_head = NULL;
}

// 便利函數
void binary_semaphore_init(semaphore_t *sem) {
    semaphore_init(sem, 0, 1);  // 初始為 0，最多 1
}

void counting_semaphore_init(semaphore_t *sem, int32_t initial) {
    semaphore_init(sem, initial, INT32_MAX);
}
```

---

## 三、Take 操作

### 3.1 基本邏輯

```c
bool semaphore_take(semaphore_t *sem, uint32_t timeout_ticks) {
    critical_enter();

    // 1. 嘗試取得資源
    if (sem->count > 0) {
        sem->count--;
        critical_exit();
        return true;  // 成功取得
    }

    // 2. 資源不可用，需要等待
    if (timeout_ticks == 0) {
        critical_exit();
        return false;  // 不等待，立即返回失敗
    }

    // 3. 設定喚醒時間
    if (timeout_ticks != WAIT_FOREVER) {
        current_tcb->wake_time = tick_count + timeout_ticks;
    } else {
        current_tcb->wake_time = UINT64_MAX;  // 永遠等待
    }

    // 4. 加入等待列表
    waiting_list_add(sem, current_tcb);

    // 5. 從 Ready List 移除，進入 Blocked 狀態
    ready_list_remove(current_tcb);
    current_tcb->state = TASK_BLOCKED;
    current_tcb->blocked_on = sem;  // 記錄在等什麼

    // 6. 觸發排程
    schedule();

    critical_exit();

    // 7. 醒來後檢查是否成功
    return (current_tcb->wake_reason == WAKE_REASON_SIGNALED);
}
```

### 3.2 等待原因

當 Task 被喚醒時，我們需要知道是因為「資源可用」還是「超時」：

```c
typedef enum {
    WAKE_REASON_NONE,
    WAKE_REASON_SIGNALED,  // 資源可用
    WAKE_REASON_TIMEOUT    // 超時
} wake_reason_t;

typedef struct tcb {
    // ... 其他欄位 ...
    wake_reason_t wake_reason;
    void *blocked_on;  // 正在等待的 Semaphore/Queue/etc
} tcb_t;
```

---

## 四、Give 操作

### 4.1 基本邏輯

```c
bool semaphore_give(semaphore_t *sem) {
    critical_enter();

    // 1. 檢查是否有 Task 在等待
    if (sem->waiting_list_head != NULL) {
        // 喚醒第一個等待的 Task
        tcb_t *task = waiting_list_remove_first(sem);

        // 從 Delayed List 移除（如果有設定 timeout）
        if (task->wake_time != UINT64_MAX) {
            delayed_list_remove(task);
        }

        // 設定喚醒原因
        task->wake_reason = WAKE_REASON_SIGNALED;
        task->blocked_on = NULL;

        // 加回 Ready List
        task->state = TASK_READY;
        ready_list_add(task);

        // 如果喚醒的 Task 優先級更高，需要重新排程
        if (task->priority > current_tcb->priority) {
            schedule();
        }

        critical_exit();
        return true;
    }

    // 2. 沒有 Task 在等待，增加計數器
    if (sem->count < sem->max_count) {
        sem->count++;
        critical_exit();
        return true;
    }

    // 3. 計數器已滿（Binary Semaphore 的情況）
    critical_exit();
    return false;
}
```

### 4.2 從 ISR 呼叫

ISR 版本不能觸發 `schedule()`，而是返回一個 flag：

```c
bool semaphore_give_from_isr(semaphore_t *sem, bool *need_switch) {
    *need_switch = false;

    if (sem->waiting_list_head != NULL) {
        tcb_t *task = waiting_list_remove_first(sem);

        task->wake_reason = WAKE_REASON_SIGNALED;
        task->blocked_on = NULL;
        task->state = TASK_READY;
        ready_list_add(task);

        // 標記需要 Context Switch
        if (task->priority > current_tcb->priority) {
            *need_switch = true;
        }

        return true;
    }

    if (sem->count < sem->max_count) {
        sem->count++;
        return true;
    }

    return false;
}
```

---

## 五、超時處理

### 5.1 整合 Delayed List

如果 Task 在等待 Semaphore 時設定了 timeout，它同時會被加入：

1. Semaphore 的 Waiting List
2. 系統的 Delayed List

### 5.2 Tick Handler 的修改

```c
void wake_delayed_tasks(void) {
    while (delayed_list_head != NULL &&
           delayed_list_head->wake_time <= tick_count) {

        tcb_t *task = delayed_list_head;
        delayed_list_remove(task);

        // 檢查是否在等待某個資源
        if (task->blocked_on != NULL) {
            // 從該資源的等待列表中移除
            semaphore_t *sem = (semaphore_t *)task->blocked_on;
            waiting_list_remove(sem, task);
            task->wake_reason = WAKE_REASON_TIMEOUT;
            task->blocked_on = NULL;
        }

        // 加回 Ready List
        task->state = TASK_READY;
        ready_list_add(task);
    }
}
```

---

## 六、使用範例

### 6.1 生產者-消費者

```c
semaphore_t data_ready;

void producer_task(void) {
    while (1) {
        // 產生資料
        int data = read_sensor();
        buffer_write(data);

        // 通知消費者
        semaphore_give(&data_ready);

        danie_delay(100);
    }
}

void consumer_task(void) {
    while (1) {
        // 等待資料
        if (semaphore_take(&data_ready, WAIT_FOREVER)) {
            // 處理資料
            int data = buffer_read();
            process(data);
        }
    }
}

void main(void) {
    binary_semaphore_init(&data_ready);

    task_create(&producer, producer_task, ...);
    task_create(&consumer, consumer_task, ...);

    scheduler_start();
}
```

### 6.2 資源池

```c
#define UART_COUNT 3
semaphore_t uart_pool;

void init(void) {
    counting_semaphore_init(&uart_pool, UART_COUNT);
}

void task_that_needs_uart(void) {
    // 取得一個 UART
    if (semaphore_take(&uart_pool, 1000)) {  // 等待最多 1 秒
        // 使用 UART
        uart_send("Hello!");

        // 釋放 UART
        semaphore_give(&uart_pool);
    } else {
        // 超時，所有 UART 都忙碌
        handle_timeout();
    }
}
```

---

## 七、Waiting List 實作

### 7.1 FIFO vs Priority

**FIFO（先進先出）**：先等待的 Task 先被喚醒

```c
void waiting_list_add(semaphore_t *sem, tcb_t *task) {
    task->next = NULL;

    if (sem->waiting_list_head == NULL) {
        sem->waiting_list_head = task;
    } else {
        tcb_t *tail = sem->waiting_list_head;
        while (tail->next != NULL) {
            tail = tail->next;
        }
        tail->next = task;
    }
}
```

**Priority-based**：優先級最高的 Task 先被喚醒

```c
void waiting_list_add_by_priority(semaphore_t *sem, tcb_t *task) {
    // 依優先級插入到正確位置
    // ... 類似 Sorted List 的插入 ...
}
```

**danieRTOS 選擇 FIFO**：簡單且公平。

---

## 總結

本文實作了 danieRTOS 的 Semaphore 機制：

1. **概念**：計數器 + Take/Give 操作
2. **Binary vs Counting**：事件通知 vs 資源池
3. **Take**：嘗試取得資源，失敗則進入 Blocked
4. **Give**：釋放資源，喚醒等待的 Task
5. **超時**：整合 Delayed List 實現 Timeout
6. **ISR 安全**：提供 FromISR 版本

Semaphore 解決了「Task 之間的同步」問題。但還有一個重要的議題：**互斥存取共享資源**。這需要 Mutex。

---

## 參考資料

**經典論文**

- **Cooperating Sequential Processes**
  Dijkstra, E. W., 1965
  Semaphore 概念的原創論文，定義了 P() 和 V() 操作。

**作業系統教科書**

- **Operating Systems: Three Easy Pieces**
  Arpaci-Dusseau, R. H. and Arpaci-Dusseau, A. C.
  https://pages.cs.wisc.edu/~remzi/OSTEP/
  Chapter 31: Semaphores，免費線上教科書，深入淺出的 Semaphore 講解。

- **Operating System Concepts (Dinosaur Book)**
  Silberschatz, A., Galvin, P. B., and Gagne, G.
  經典 OS 教科書，Chapter 6-7 涵蓋 Synchronization 和 Semaphore。

**RTOS 參考實作**

- **FreeRTOS Kernel - semphr.h**
  https://github.com/FreeRTOS/FreeRTOS-Kernel
  Binary Semaphore 和 Counting Semaphore 的參考實作。

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)
