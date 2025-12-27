# Mutex 與 Priority Inversion

**作者**: Danny Jiang
**日期**: 2025-12-13

---

## 前言：火星探測器的教訓

1997 年 7 月 4 日，NASA 的火星探測器「Pathfinder」成功登陸火星，全世界歡欣鼓舞。

然後，系統開始不斷重啟。

地球上的工程師們看著遙測數據，完全不知道發生了什麼事。程式沒有 crash，硬體也正常，但系統就是一直重啟。更詭異的是，這個問題在地球上的測試中從未發生過。

經過數天的分析，他們終於找到了罪魁禍首：**Priority Inversion（優先級反轉）**。

一個低優先級的 Task 持有了一個鎖，然後被中優先級的 Task 搶佔。結果呢？高優先級的 Task 需要那個鎖，卻被間接阻塞——Watchdog 認為系統死機了，於是觸發重啟。

這個 bug 差點讓數億美元的任務失敗。

幸運的是，工程師們知道解法。他們遠端上傳了一個修補程式，啟用了 **Priority Inheritance** 機制，問題解決。

這個故事告訴我們：Priority Inversion 是多任務系統中最隱蔽的問題之一。它不會讓程式 crash，也不會產生錯誤訊息，但會讓高優先級的 Task 被無限期延遲——這在即時系統中是致命的。

**Mutex** 是解決這個問題的關鍵工具。本文將解釋 Priority Inversion 的原理，並實作帶有 **Priority Inheritance** 的 Mutex。

> 💡 **真實案例**：Mars Pathfinder 的故事被廣泛記錄，成為電腦科學課程中的經典案例。如果你對細節感興趣，可以搜尋「What Really Happened on Mars」。

---

## 一、Mutex vs Semaphore

### 1.1 看起來一樣，但用途不同

Mutex 和 Binary Semaphore 的操作看起來完全一樣：

| 操作 | Semaphore | Mutex |
|------|-----------|-------|
| 取得 | `take()` | `lock()` |
| 釋放 | `give()` | `unlock()` |
| 計數器 | 0 或 1 | 0 或 1 |

但它們的**語意**不同：

**Semaphore：訊號通知**

- Task A give，Task B take
- 「生產者通知消費者」

**Mutex：互斥存取**

- Task A lock，Task A unlock（同一個 Task）
- 「保護共享資源」

### 1.2 關鍵差異

| 特性 | Semaphore | Mutex |
|------|-----------|-------|
| 誰可以 give/unlock | 任何人 | 只有擁有者 |
| 所有權概念 | ❌ 無 | ✅ 有 |
| Priority Inheritance | ❌ 無 | ✅ 可以有 |
| 遞迴鎖定 | ❌ 不行 | ✅ 可以（Recursive Mutex） |

### 1.3 錯誤使用的後果

如果用 Semaphore 來保護共享資源：

```c
semaphore_t lock;

void task_a(void) {
    semaphore_take(&lock, WAIT_FOREVER);
    // 使用共享資源...
    semaphore_give(&lock);
}

void task_b(void) {
    // 忘記 take，直接 give！
    semaphore_give(&lock);  // 破壞了保護
    // 現在 count = 2，兩個 Task 可以同時進入！
}
```

用 Mutex 就不會有這個問題，因為只有「擁有者」可以 unlock。

---

## 二、Priority Inversion 問題

### 2.1 經典場景

假設有三個 Task：

- **Task H**：高優先級（Priority 3）
- **Task M**：中優先級（Priority 2）
- **Task L**：低優先級（Priority 1）

Task H 和 Task L 需要存取同一個共享資源（用 Mutex 保護）。

### 2.2 災難發生

```
時間軸：
─────────────────────────────────────────────────────────────────

1. Task L 開始執行，取得 Mutex
   │ Task L 正在使用共享資源...
   ▼

2. Task H 變成 Ready（例如：被中斷喚醒）
   │ Scheduler 切換到 Task H（搶佔）
   ▼

3. Task H 嘗試取得 Mutex
   │ Mutex 被 Task L 持有 → Task H 進入 Blocked
   │ Scheduler 切換回 Task L
   ▼

4. Task M 變成 Ready
   │ Task M 優先級 > Task L → Scheduler 切換到 Task M
   │ Task M 開始執行（可能執行很久！）
   ▼

5. Task H 在等什麼？
   │ Task H 在等 Task L 釋放 Mutex
   │ Task L 在等 Task M 執行完
   │ Task M 可能執行無限久...
   │
   │ 結果：高優先級 Task H 被中優先級 Task M 阻塞！
   ▼

   這就是 Priority Inversion！
```

### 2.3 問題的本質

**優先級反轉**：高優先級 Task 實際上被比它低優先級的 Task 阻塞。

- Task H（優先級 3）被 Task L（優先級 1）持有的 Mutex 阻塞 → 這是正常的
- Task L 又被 Task M（優先級 2）搶佔 → 這導致 Task H 間接被 Task M 阻塞

問題：Task H 和 Task M 沒有任何資源競爭，但 Task H 卻被 Task M 延遲了！

---

## 三、Priority Inheritance Protocol

### 3.1 解法概念

當高優先級 Task 因為 Mutex 被阻塞時，**暫時提升** Mutex 擁有者的優先級。

```
修正後的時間軸：
─────────────────────────────────────────────────────────────────

1. Task L 開始執行，取得 Mutex
   ▼

2. Task H 變成 Ready，搶佔 Task L
   ▼

3. Task H 嘗試取得 Mutex，發現被 Task L 持有
   │
   │ ★ Priority Inheritance 啟動 ★
   │ Task L 的優先級被提升到 3（和 Task H 一樣）
   │
   │ Task H 進入 Blocked
   │ Scheduler 切換到 Task L（現在優先級是 3）
   ▼

4. Task M 變成 Ready
   │ Task M 優先級 = 2，Task L 現在優先級 = 3
   │ Task M 不能搶佔 Task L！
   ▼

5. Task L 繼續執行，釋放 Mutex
   │ Task L 優先級恢復為 1
   │ Task H 被喚醒，立即執行
   ▼

6. Task H 完成後，Task M 才能執行

   問題解決！高優先級 Task 不再被無關的中優先級 Task 阻塞。
```

### 3.2 Priority Inheritance 的規則

1. 當 Task A 因為 Mutex 被阻塞時，檢查 Mutex 的擁有者 Task B
2. 如果 Task A 的優先級 > Task B 的優先級，提升 Task B 的優先級為 Task A 的優先級
3. 當 Task B 釋放 Mutex 時，恢復 Task B 的原始優先級

---

## 四、Mutex 實作

### 4.1 資料結構

```c
typedef struct {
    tcb_t *owner;                // Mutex 的擁有者
    uint32_t lock_count;         // 遞迴鎖定計數
    uint32_t owner_original_priority;  // 擁有者的原始優先級
    tcb_t *waiting_list_head;    // 等待中的 Task
} mutex_t;
```

### 4.2 初始化

```c
void mutex_init(mutex_t *mutex) {
    mutex->owner = NULL;
    mutex->lock_count = 0;
    mutex->owner_original_priority = 0;
    mutex->waiting_list_head = NULL;
}
```

### 4.3 Lock 操作

```c
bool mutex_lock(mutex_t *mutex, uint32_t timeout_ticks) {
    critical_enter();

    // 1. 如果沒有擁有者，取得 Mutex
    if (mutex->owner == NULL) {
        mutex->owner = current_tcb;
        mutex->lock_count = 1;
        mutex->owner_original_priority = current_tcb->priority;
        critical_exit();
        return true;
    }

    // 2. 如果是自己（遞迴鎖定）
    if (mutex->owner == current_tcb) {
        mutex->lock_count++;
        critical_exit();
        return true;
    }

    // 3. 被別人持有，需要等待
    if (timeout_ticks == 0) {
        critical_exit();
        return false;
    }

    // 4. ★ Priority Inheritance ★
    if (current_tcb->priority > mutex->owner->priority) {
        // 提升擁有者的優先級
        ready_list_remove(mutex->owner);
        mutex->owner->priority = current_tcb->priority;
        ready_list_add(mutex->owner);
    }

    // 5. 加入等待列表並 Block
    waiting_list_add_by_priority(mutex, current_tcb);

    if (timeout_ticks != WAIT_FOREVER) {
        current_tcb->wake_time = tick_count + timeout_ticks;
        delayed_list_add(current_tcb);
    }

    ready_list_remove(current_tcb);
    current_tcb->state = TASK_BLOCKED;
    current_tcb->blocked_on = mutex;

    schedule();

    critical_exit();

    return (current_tcb->wake_reason == WAKE_REASON_SIGNALED);
}
```

### 4.4 Unlock 操作

```c
bool mutex_unlock(mutex_t *mutex) {
    critical_enter();

    // 1. 檢查是否為擁有者
    if (mutex->owner != current_tcb) {
        critical_exit();
        return false;  // 不是擁有者，不能 unlock
    }

    // 2. 遞迴鎖定計數
    mutex->lock_count--;
    if (mutex->lock_count > 0) {
        critical_exit();
        return true;  // 還有遞迴鎖定，不釋放
    }

    // 3. ★ 恢復原始優先級 ★
    if (current_tcb->priority != mutex->owner_original_priority) {
        ready_list_remove(current_tcb);
        current_tcb->priority = mutex->owner_original_priority;
        ready_list_add(current_tcb);
    }

    // 4. 釋放 Mutex
    mutex->owner = NULL;

    // 5. 喚醒等待的 Task（如果有）
    if (mutex->waiting_list_head != NULL) {
        tcb_t *next_owner = waiting_list_remove_first(mutex);

        // 移除超時設定
        if (next_owner->wake_time != UINT64_MAX) {
            delayed_list_remove(next_owner);
        }

        // 設為新擁有者
        mutex->owner = next_owner;
        mutex->lock_count = 1;
        mutex->owner_original_priority = next_owner->priority;

        next_owner->wake_reason = WAKE_REASON_SIGNALED;
        next_owner->blocked_on = NULL;
        next_owner->state = TASK_READY;
        ready_list_add(next_owner);

        // 重新排程
        schedule();
    }

    critical_exit();
    return true;
}
```

---

## 五、測試 Priority Inversion

### 5.1 測試案例

```c
mutex_t shared_mutex;
volatile int shared_resource = 0;

void task_high(void) {
    danie_delay(100);  // 讓 Task L 先取得 Mutex

    uart_puts("H: Trying to lock...\n");
    mutex_lock(&shared_mutex, WAIT_FOREVER);
    uart_puts("H: Got mutex!\n");

    shared_resource++;

    mutex_unlock(&shared_mutex);
    uart_puts("H: Released mutex\n");
}

void task_medium(void) {
    danie_delay(150);  // 在 Task H 等待時開始

    uart_puts("M: Running (no mutex needed)...\n");
    for (int i = 0; i < 1000000; i++);  // 模擬長時間計算
    uart_puts("M: Done\n");
}

void task_low(void) {
    mutex_lock(&shared_mutex, WAIT_FOREVER);
    uart_puts("L: Got mutex, working...\n");

    for (int i = 0; i < 500000; i++);  // 模擬工作

    mutex_unlock(&shared_mutex);
    uart_puts("L: Released mutex\n");
}
```

### 5.2 沒有 Priority Inheritance 的輸出

```
L: Got mutex, working...
H: Trying to lock...
M: Running (no mutex needed)...
M: Done                          ← Task H 被 Task M 阻塞！
L: Released mutex
H: Got mutex!
H: Released mutex
```

### 5.3 有 Priority Inheritance 的輸出

```
L: Got mutex, working...
H: Trying to lock...             ← Task L 優先級被提升
L: Released mutex                ← Task L 繼續執行，不被 M 搶佔
H: Got mutex!
H: Released mutex
M: Running (no mutex needed)...  ← Task M 最後才執行
M: Done
```

---

## 總結

本文實作了 danieRTOS 的 Mutex 機制：

1. **Mutex vs Semaphore**：Mutex 有所有權概念，只有擁有者能 unlock
2. **Priority Inversion**：高優先級 Task 被無關的中優先級 Task 阻塞
3. **Priority Inheritance**：暫時提升 Mutex 擁有者的優先級
4. **實作**：lock 時檢查並提升，unlock 時恢復
5. **遞迴 Mutex**：同一 Task 可以多次 lock

Mutex 和 Semaphore 解決了「同步」和「互斥」問題。但有時候 Task 之間需要傳遞資料，這就是 Queue 的用途。

---

## 參考資料

**經典論文**

- **Priority Inheritance Protocols: An Approach to Real-Time Synchronization**
  Sha, L., Rajkumar, R., and Lehoczky, J. P.
  IEEE Transactions on Computers, 1990
  Priority Inheritance Protocol 的原始論文，解決 Priority Inversion 問題的經典方案。

**案例研究**

- **What Really Happened on Mars?**
  Mike Jones, Microsoft Research
  https://www.microsoft.com/en-us/research/people/mbj/
  Mars Pathfinder 的 Priority Inversion 事件分析，說明為什麼 Mutex 設計如此重要。

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)
