# 多核心互斥鎖：SMP Mutex

**作者**: Danny Jiang
**日期**: 2025-12-14

---

## 前言：Spinlock 不夠用

在 Ch 23，我們實作了 Spinlock。它很快，但有個致命缺點：

```c
spinlock_acquire(&lock);
// 如果這裡需要等待很久...
// 例如：等待 I/O、等待其他 Task 完成
spinlock_release(&lock);
```

**問題**：Spinlock 會讓 CPU 空轉，浪費運算資源。

對於「短暫」的臨界區（< 1 μs），Spinlock 很好。
對於「較長」的臨界區（> 1 μs），我們需要 **Mutex**——一種可以讓 Task 睡眠的鎖。

---

## 一、Spinlock vs Mutex

### 1.1 核心差異

| 特性 | Spinlock | Mutex |
|------|----------|-------|
| **等待方式** | 忙等待（Busy-Wait）| 睡眠（Sleep）|
| **Context Switch** | 否 | 是 |
| **CPU 使用** | 高（一直佔用）| 低（讓出 CPU）|
| **適用場景** | 極短臨界區 | 較長臨界區 |
| **中斷安全** | 是 | 否 |
| **可遞迴** | 通常否 | 可以 |
| **Priority Inheritance** | 否 | 可以 |

### 1.2 何時用哪個？

```
臨界區時間 < 1 μs  → Spinlock
臨界區時間 > 1 μs  → Mutex
在 ISR 中         → Spinlock（不能睡眠）
需要 Priority Inheritance → Mutex
```

### 1.3 SMP 環境的挑戰

在單核心系統，Mutex 只需要禁用中斷就能保護臨界區。
在多核心系統，禁用中斷只能保護「本核心」，其他核心仍可能同時存取。

**解決方案**：Mutex 內部使用 Spinlock 保護自己的資料結構。

---

## 二、SMP Mutex 設計

### 2.1 資料結構

```c
typedef struct mutex {
    spinlock_t lock;           // 保護 Mutex 內部狀態
    tcb_t *owner;              // 目前擁有者
    uint32_t lock_count;       // 遞迴計數
    priority_t owner_base_prio; // 擁有者的原始優先權
    waitqueue_t wait_queue;    // 等待佇列
} mutex_t;

#define MUTEX_INIT { \
    .lock = SPINLOCK_INIT, \
    .owner = NULL, \
    .lock_count = 0, \
    .owner_base_prio = 0, \
    .wait_queue = WAITQUEUE_INIT \
}
```

### 2.2 Wait Queue

等待佇列按優先權排序，最高優先權的 Task 在最前面：

```c
typedef struct waitqueue {
    tcb_t *head;
} waitqueue_t;

#define WAITQUEUE_INIT { .head = NULL }

// 按優先權插入（高優先權在前）
void waitqueue_add(waitqueue_t *wq, tcb_t *task) {
    tcb_t **pp = &wq->head;

    while (*pp != NULL && (*pp)->priority >= task->priority) {
        pp = &(*pp)->wait_next;
    }

    task->wait_next = *pp;
    *pp = task;
}

// 取出最高優先權的 Task
tcb_t *waitqueue_pop(waitqueue_t *wq) {
    tcb_t *task = wq->head;
    if (task != NULL) {
        wq->head = task->wait_next;
        task->wait_next = NULL;
    }
    return task;
}
```

---

## 三、Lock 操作

### 3.1 基本流程

```
mutex_lock():
1. 取得 Spinlock（保護 Mutex 內部）
2. 如果沒有擁有者 → 取得 Mutex，返回
3. 如果是自己（遞迴）→ 增加計數，返回
4. 如果被別人持有：
   a. Priority Inheritance（如果需要）
   b. 加入等待佇列
   c. 釋放 Spinlock
   d. 進入睡眠
   e. 醒來後重新取得 Spinlock
5. 釋放 Spinlock
```

### 3.2 實作

```c
bool mutex_lock(mutex_t *mtx, uint32_t timeout_ticks) {

### 3.3 Priority Inheritance

當高優先權 Task 等待低優先權 Task 持有的 Mutex 時，需要「提升」低優先權 Task 的優先權：

```c
void boost_priority(tcb_t *task, priority_t new_priority) {
    if (task->priority >= new_priority) {
        return;  // 已經夠高
    }

    // 如果 Task 在 Ready Queue 中，需要重新排序
    if (task->state == TASK_STATE_READY) {
        spinlock_acquire(&g_sched_lock);
        sched_remove_ready(task);
        task->priority = new_priority;
        sched_add_ready(task);
        spinlock_release(&g_sched_lock);
    } else {
        task->priority = new_priority;
    }
}
```

---

## 四、Unlock 操作

### 4.1 基本流程

```
mutex_unlock():
1. 取得 Spinlock
2. 檢查是否為擁有者
3. 減少遞迴計數
4. 如果計數歸零：
   a. 恢復原始優先權
   b. 從等待佇列取出下一個 Task
   c. 如果有等待者，轉移擁有權
   d. 喚醒等待者
5. 釋放 Spinlock
```

### 4.2 實作

```c
void mutex_unlock(mutex_t *mtx) {
    spinlock_acquire(&mtx->lock);

    tcb_t *current = get_current_task();

    // 必須是擁有者
    if (mtx->owner != current) {
        spinlock_release(&mtx->lock);
        return;  // 錯誤：不是擁有者
    }

    // 遞迴解鎖
    if (mtx->lock_count > 1) {
        mtx->lock_count--;
        spinlock_release(&mtx->lock);
        return;
    }

    // 恢復原始優先權
    if (current->priority != mtx->owner_base_prio) {
        current->priority = current->base_priority;
    }

    // 取出下一個等待者
    tcb_t *next_owner = waitqueue_pop(&mtx->wait_queue);

    if (next_owner != NULL) {
        // 轉移擁有權
        mtx->owner = next_owner;
        mtx->lock_count = 1;
        mtx->owner_base_prio = next_owner->priority;

        // 喚醒
        next_owner->wake_reason = WAKE_REASON_SIGNALED;
        next_owner->state = TASK_STATE_READY;

        spinlock_acquire(&g_sched_lock);
        sched_add_ready(next_owner);
        spinlock_release(&g_sched_lock);

        // 如果喚醒的 Task 優先權更高，觸發排程
        if (next_owner->priority > current->priority) {
            spinlock_release(&mtx->lock);
            sched_yield();
            return;
        }
    } else {
        // 沒有等待者
        mtx->owner = NULL;
        mtx->lock_count = 0;
    }

    spinlock_release(&mtx->lock);
}
```

---

## 五、跨核心喚醒

### 5.1 問題

當 Core 0 釋放 Mutex，喚醒的 Task 可能綁定到 Core 1：

```
Core 0                          Core 1
------                          ------
mutex_unlock()
  → 喚醒 Task B（綁定 Core 1）
  → Task B 加入 Ready Queue
                                (Core 1 不知道有新 Task)
```

### 5.2 解決方案：IPI

```c
void mutex_unlock(mutex_t *mtx) {
    // ... 前面的程式碼 ...

    if (next_owner != NULL) {
        // 轉移擁有權並喚醒
        mtx->owner = next_owner;
        mtx->lock_count = 1;
        next_owner->wake_reason = WAKE_REASON_SIGNALED;
        next_owner->state = TASK_STATE_READY;

        spinlock_acquire(&g_sched_lock);
        sched_add_ready(next_owner);
        spinlock_release(&g_sched_lock);

        // 檢查是否需要通知其他核心
        uint64_t my_hartid = get_hartid();

        for (int i = 0; i < MAX_CORES; i++) {
            if (i == my_hartid) continue;

            // 如果 Task 可以在這個核心執行
            if (task_can_run_on(next_owner, i)) {
                cpu_t *cpu = &g_cpus[i];

                // 如果這個核心的當前 Task 優先權較低
                if (cpu->current_task->priority < next_owner->priority) {
                    smp_request_reschedule(i);
                    break;
                }
            }
        }

        // 也檢查自己
        if (next_owner->priority > current->priority) {
            spinlock_release(&mtx->lock);
            sched_yield();
            return;
        }
    }

    // ...
}
```

---

## 六、Timeout 處理

### 6.1 Timeout 喚醒

當 Task 等待 Mutex 超時時，Timer 會喚醒它：

```c
void tick_check_sleeping_tasks(void) {
    uint64_t now = tick_get();

    // 檢查 Delay Queue
    while (delay_queue != NULL && delay_queue->wake_time <= now) {
        tcb_t *task = delay_queue;
        delay_queue = task->delay_next;

        // 設定喚醒原因
        task->wake_reason = WAKE_REASON_TIMEOUT;

        // 從 Mutex 等待佇列中移除
        if (task->blocked_on != NULL) {
            mutex_t *mtx = (mutex_t *)task->blocked_on;
            spinlock_acquire(&mtx->lock);
            waitqueue_remove(&mtx->wait_queue, task);
            spinlock_release(&mtx->lock);
            task->blocked_on = NULL;
        }

        // 加入 Ready Queue
        task->state = TASK_STATE_READY;
        spinlock_acquire(&g_sched_lock);
        sched_add_ready(task);
        spinlock_release(&g_sched_lock);
    }
}
```

### 6.2 Wait Queue 移除

```c
void waitqueue_remove(waitqueue_t *wq, tcb_t *task) {
    tcb_t **pp = &wq->head;

    while (*pp != NULL) {
        if (*pp == task) {
            *pp = task->wait_next;
            task->wait_next = NULL;
            return;
        }
        pp = &(*pp)->wait_next;
    }
}
```

---

## 七、完整實作

### 7.1 mutex.h

```c
#ifndef MUTEX_H
#define MUTEX_H

#include "spinlock.h"
#include "task.h"

typedef struct waitqueue {
    tcb_t *head;
} waitqueue_t;

typedef struct mutex {
    spinlock_t lock;
    tcb_t *owner;
    uint32_t lock_count;
    priority_t owner_base_prio;
    waitqueue_t wait_queue;
} mutex_t;

#define WAITQUEUE_INIT { .head = NULL }
#define MUTEX_INIT { \
    .lock = SPINLOCK_INIT, \
    .owner = NULL, \
    .lock_count = 0, \
    .owner_base_prio = 0, \
    .wait_queue = WAITQUEUE_INIT \
}

void mutex_init(mutex_t *mtx);
bool mutex_lock(mutex_t *mtx, uint32_t timeout_ticks);
bool mutex_trylock(mutex_t *mtx);
void mutex_unlock(mutex_t *mtx);
tcb_t *mutex_get_owner(mutex_t *mtx);

#endif // MUTEX_H
```

### 7.2 使用範例

```c
mutex_t data_mutex = MUTEX_INIT;
int shared_data = 0;

void task_writer(void *arg) {
    while (1) {
        if (mutex_lock(&data_mutex, 1000)) {  // 1 秒 Timeout
            shared_data++;
            uart_printf("Write: %d\n", shared_data);
            mutex_unlock(&data_mutex);
        } else {
            uart_printf("Timeout!\n");
        }
        task_delay(100);
    }
}

void task_reader(void *arg) {
    while (1) {
        if (mutex_lock(&data_mutex, WAIT_FOREVER)) {
            uart_printf("Read: %d\n", shared_data);
            mutex_unlock(&data_mutex);
        }
        task_delay(50);
    }
}
```

---

## 八、效能比較

### 8.1 Spinlock vs Mutex

| 場景 | Spinlock | Mutex |
|------|----------|-------|
| 無競爭 | 5 cycles | 50 cycles |
| 短暫競爭（< 1 μs）| 10-100 cycles | 500+ cycles |
| 長時間競爭（> 1 μs）| 浪費 CPU | 讓出 CPU |

### 8.2 選擇指南

```c
// 保護 Ready Queue（極短臨界區）
spinlock_t ready_queue_lock;

// 保護共享資料（可能需要等待）
mutex_t data_mutex;

// 保護 I/O 操作（一定需要等待）
mutex_t io_mutex;
```

---

## 九、本章回顧

在這一章，我們實作了 SMP Mutex：

1. **Spinlock 保護**：Mutex 內部使用 Spinlock 保護自己
2. **Wait Queue**：按優先權排序的等待佇列
3. **Priority Inheritance**：防止 Priority Inversion
4. **跨核心喚醒**：使用 IPI 通知其他核心
5. **Timeout 處理**：支援等待超時

關鍵設計：

```
Mutex = Spinlock + Wait Queue + Priority Inheritance
```

下一章，我們將實作 **SMP Semaphore**——支援多核心的信號量。

---

## 參考資料

- **FreeRTOS**, Mutex Implementation
- **Linux Kernel**, rt_mutex (Real-Time Mutex)
- **Tech Column**, RTOS Mutex 系列

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。
