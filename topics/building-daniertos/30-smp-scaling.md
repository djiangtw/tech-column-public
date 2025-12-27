# SMP 擴展：從 2 核心到 4 核心的實戰經驗

**作者**: Danny Jiang
**日期**: 2025-12-15

---

## 前言：當系統「安靜地」停止

在完成 Ch 29 的 SMP 整合後，danieRTOS v2.x 在雙核心環境下運行得相當穩定。2-core SMP demo 可以持續運行超過 28000 ticks，Spinlock、IPI、Mutex、Queue 都正常工作。

然後我決定挑戰一個更接近真實世界的場景：**4-core SSD Controller Demo**。

這個 demo 模擬一個典型的 SSD 控制器架構：

| Core | Task | 優先權 | Affinity | 職責 |
|------|------|--------|----------|------|
| Core 0 | Front-End | High | 固定 | NVMe 命令處理 |
| Core 1 | Back-End | High | 固定 | Flash R/W, FTL |
| Core 2 | GC | Medium | 固定 | Garbage Collection |
| Core 3 | Misc | Low | 可遷移 | Power/Thermal/SMART |

看起來很簡單，對吧？把 `SMP=2` 改成 `SMP=4`，加幾個 task，應該就能跑了。

**現實給了我一記重拳。**

```
[Core 0] Starting scheduler
[Core 1] Starting scheduler
[Core 2] Starting scheduler
[Core 3] Starting scheduler
[6] [FrontEnd] Started on Core 0
[6] [BackEnd] Started on Core 1
[14] [GC] Started on Core 2
[19] [Misc] Started on Core 3
[21] [Misc] Health report #0 - Temp=46C, Power=OK
... (一些正常的輸出)
[~200] (系統停止，沒有任何錯誤訊息)
```

系統在約 200 ticks 後「安靜地」停止了。沒有 Exception，沒有 FATAL 錯誤，就這樣停住。

接下來的兩天，我經歷了一場深刻的調試之旅。這一章將記錄我發現的兩個關鍵 bug，以及從中學到的 SMP 編程教訓。

---

## 一、問題 1：Delay List 損壞（雙重等待問題）

### 1.1 症狀觀察

第一個問題的症狀非常詭異：
- SMP=2 可以運行 28000+ ticks
- SMP=4 只能運行 150-300 ticks
- Timer IRQ 似乎停止觸發
- 沒有任何錯誤訊息

### 1.2 調試過程

我首先懷疑是 Timer 配置問題，但檢查後發現 CLINT 設置正確。接著我加入 debug 輸出：

```c
void delay_check_wakeups(tick_t current_tick)
{
    uart_printf("[DELAY] Checking wakeups at tick %lu\n", current_tick);
    uart_printf("[DELAY] List head: %s (wake=%lu)\n",
        g_delay_list ? g_delay_list->name : "NULL",
        g_delay_list ? g_delay_list->wake_tick : 0);
    // ...
}
```

然後我看到了驚人的輸出：

```
[DELAY] Checking wakeups at tick 150
[DELAY] List head: BackEnd (wake=102)
[DELAY] Waking BackEnd, but state is RUNNING!?
```

一個 **正在運行的 task** 出現在 delay list 中！

### 1.3 根因分析

經過仔細追蹤，我發現問題出在「帶 timeout 的阻塞操作」：

```
┌─────────────────────────────────────────────────────────────┐
│              帶 Timeout 的 Queue Receive                     │
├─────────────────────────────────────────────────────────────┤
│  queue_receive(&q, &item, 1000);  // 等待最多 1000 ticks    │
│                                                              │
│  1. Task 加入 wait queue（等待數據）                         │
│  2. Task 加入 delay list（timeout 保護）                     │
│  3. Task 進入 BLOCKED 狀態                                   │
│                                                              │
│  ─────── 等待中 ───────                                      │
│                                                              │
│  情況 A：數據到達（被 signal 喚醒）                          │
│    → 從 wait queue 移除 ✓                                   │
│    → 從 delay list 移除 ✗ ← BUG！                           │
│                                                              │
│  情況 B：timeout 到期                                        │
│    → 從 delay list 移除 ✓                                   │
│    → 從 wait queue 移除 ✓                                   │
└─────────────────────────────────────────────────────────────┘
```

問題就在「情況 A」：當 task 被 signal 喚醒時，代碼只設置了 `wake_reason`，但**沒有從 delay list 移除 task**！

```c
// 錯誤的代碼
bool received = (current->wake_reason == WAKE_REASON_SIGNALED);
if (!received) {
    waitqueue_remove(&q->recv_wait, current);  // 只處理了 timeout 情況
}
// 缺少：被 signal 喚醒時，需要從 delay list 移除！
```

這導致：
1. Task 被 signal 喚醒，恢復運行
2. Delay list 中仍然保留這個 task 的 entry
3. 當原本的 timeout 到期時，`delay_check_wakeups()` 會嘗試喚醒一個已經在運行的 task
4. Ready queue 被損壞，系統最終停止

### 1.4 解決方案

修復很直接——被 signal 喚醒時，需要從 delay list 移除：

```c
bool received = (current->wake_reason == WAKE_REASON_SIGNALED);
if (received) {
    /* Woken by signal - remove from delay list if we had a timeout */
    if (timeout_ticks != WAIT_FOREVER) {
        delay_remove(current);  // ← 這行是關鍵！
    }
} else {
    /* Woken by timeout - remove from wait queue */
    waitqueue_remove(&q->recv_wait, current);
}
```

這個修改需要應用到所有帶 timeout 的阻塞 API：
- `queue_send()` / `queue_receive()`
- `sem_wait()`
- `mutex_lock()`

---

## 二、問題 2：共用 Next 指針（隱藏的耦合）

修復第一個問題後，系統可以運行更久了，但新的問題浮現：

### 2.1 症狀觀察

```
[21] [Misc] Health report #0 - Temp=46C, Power=OK
... (其他 3 個 tasks 正常運行)
... (Misc 再也沒有出現！)
```

Misc task 只執行了一次 health report，然後就「消失」了。沒有錯誤，沒有 Exception，就是不再執行。

### 2.2 調試過程

我加入更多 debug 輸出來追蹤 delay list：

```c
void debug_delay_list(void)
{
    uart_printf("[DELAY] List: ");
    tcb_t *p = g_delay_list;
    while (p) {
        uart_printf("%s(wake=%lu) -> ", p->name, p->wake_tick);
        p = p->next;
    }
    uart_printf("NULL\n");
}
```

輸出讓我震驚：

```
Tick 8:  [DELAY] List: BackEnd(wake=1002) -> Misc(wake=1008) -> NULL
Tick 10: [BackEnd 被 queue signal 喚醒]
Tick 22: [DELAY] List: FrontEnd(wake=100) -> GC(wake=700) -> BackEnd(wake=1022) -> NULL
         Misc 消失了！？
```

Misc 明明在 delay list 中，為什麼會「消失」？

### 2.3 根因分析

經過仔細檢查 TCB 結構，我發現了問題：

```c
typedef struct tcb {
    reg_t *sp;
    task_state_t state;
    priority_t priority;
    char name[16];
    uint32_t affinity_mask;
    int32_t running_on;
    tick_t wake_tick;
    tcb_t *next;        // ← 這個指針被兩個地方使用！
    // ...
} tcb_t;
```

**`next` 指針同時被 ready queue 和 delay list 使用！**

這在單核心或簡單情況下可能沒問題，但在 4-core SMP 中，timing 變得更緊湊：

```
┌─────────────────────────────────────────────────────────────┐
│                    Race Condition 時序                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Tick 8:  Misc 調用 delay_ticks(1000)                        │
│           delay list: BackEnd -> Misc -> NULL                │
│           Misc->next = NULL                                  │
│                                                              │
│  Tick 10: BackEnd 收到 queue signal，被喚醒                  │
│           sched_add_ready(BackEnd)                           │
│           BackEnd->next = NULL  ← 這會影響 delay list！      │
│                                                              │
│  結果：delay list 的鏈結被破壞                               │
│        Misc 從 delay list 中「斷開」了                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

問題的本質是：**一個 task 可能同時存在於 delay list 和 ready queue**（在短暫的過渡期間），而它們共用同一個 `next` 指針會導致鏈表損壞。

### 2.4 解決方案

解決方案是為 delay list 使用獨立的指針：

```c
typedef struct tcb {
    // ... 其他欄位 ...
    tcb_t *next;        // 用於 ready queue 和 wait queues
    tcb_t *delay_next;  // 專用於 delay list  ← 新增！
} tcb_t;
```

修改 `delay.c` 中所有鏈表操作：

```c
// delay_add_locked()
static void delay_add_locked(tcb_t *task, tick_t wake_tick)
{
    task->wake_tick = wake_tick;

    tcb_t **pp = &g_delay_list;
    while (*pp != NULL && (*pp)->wake_tick <= wake_tick) {
        pp = &(*pp)->delay_next;  // 使用 delay_next
    }
    task->delay_next = *pp;       // 使用 delay_next
    *pp = task;
}

// delay_check_wakeups()
void delay_check_wakeups(tick_t current_tick)
{
    // ...
    while (g_delay_list != NULL &&
           g_delay_list->wake_tick <= current_tick) {
        tcb_t *task = g_delay_list;
        g_delay_list = task->delay_next;  // 使用 delay_next
        task->delay_next = NULL;
        // ...
    }
}

// delay_remove()
void delay_remove(tcb_t *task)
{
    // ...
    tcb_t **pp = &g_delay_list;
    while (*pp != NULL) {
        if (*pp == task) {
            *pp = task->delay_next;       // 使用 delay_next
            task->delay_next = NULL;
            break;
        }
        pp = &(*pp)->delay_next;          // 使用 delay_next
    }
}
```

---

## 三、修復後的驗證

修復這兩個問題後，4-core SSD Controller demo 終於正常運行：

```
╔══════════════════════════════════════════════════════════════╗
║           SSD Controller Demo (4-Core SMP)                   ║
╠══════════════════════════════════════════════════════════════╣
║  Core 0: Front-End  - NVMe command processing                ║
║  Core 1: Back-End   - Flash R/W, FTL operations              ║
║  Core 2: GC         - Garbage Collection                     ║
║  Core 3: Misc       - Power/Thermal/SMART                    ║
╚══════════════════════════════════════════════════════════════╝

[6] [FrontEnd] Started on Core 0 - NVMe command processing
[6] [BackEnd] Started on Core 1 - Flash R/W, FTL ops
[14] [GC] Started on Core 2 - Garbage Collection
[19] [Misc] Started on Core 3 - Power/Thermal/SMART
[21] [Misc] Health report #0 - Temp=46C, Power=OK
...
[1024] [Misc] Health report #1 - Temp=49C, Power=OK
...
[2026] [Misc] Health report #2 - Temp=51C, Power=OK
...
[14369] [BackEnd] Completed cmd #139
```

四個 tasks 都正常運行，Misc 的 health report 每 1000 ticks 準時出現。

---

## 四、教訓總結

這次從 2-core 到 4-core 的擴展過程，教會我幾個重要的 SMP 編程教訓：

### 4.1 教訓 1：雙重等待需要雙重清理

當一個 task 同時在兩個等待結構中（如 wait queue + delay list），被喚醒時必須從**兩個地方**都清理。

```
┌─────────────────────────────────────────────────────────────┐
│                    雙重等待模式                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Task 阻塞時：                                               │
│    ┌──────────────┐   ┌──────────────┐                      │
│    │  Wait Queue  │   │  Delay List  │                      │
│    │  (等待事件)   │   │ (timeout 保護)│                      │
│    └──────┬───────┘   └──────┬───────┘                      │
│           │                  │                               │
│           └────────┬─────────┘                               │
│                    ▼                                         │
│              ┌──────────┐                                    │
│              │   Task   │                                    │
│              │ (BLOCKED)│                                    │
│              └──────────┘                                    │
│                                                              │
│  Task 喚醒時：必須從兩個結構中移除！                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 教訓 2：共用指針是隱藏的耦合

看似無害的「共用 next 指針」設計，在多核心環境下成為 bug 來源。

**設計原則**：每個獨立的鏈表結構應該有自己專用的 link 指針。

```c
// ❌ 錯誤：共用指針
struct task {
    task_t *next;  // ready queue 和 delay list 共用
};

// ✅ 正確：獨立指針
struct task {
    task_t *next;        // ready queue 專用
    task_t *delay_next;  // delay list 專用
    task_t *wait_next;   // wait queue 專用（如果需要）
};
```

### 4.3 教訓 3：SMP=2 OK ≠ SMP=4 OK

核心數增加會改變 timing，暴露出原本隱藏的 race condition。

| 核心數 | Timing 特性 | Bug 暴露機率 |
|--------|-------------|--------------|
| 1 | 序列執行 | 低 |
| 2 | 輕微並行 | 中 |
| 4+ | 高度並行 | 高 |

**測試建議**：即使目標是 2-core，也應該在更多核心上測試。

### 4.4 教訓 4：「安靜的失敗」最難調試

系統沒有 crash、沒有 Exception，只是「停止」了。這種情況通常是資料結構損壞，導致：
- 無限迴圈（如 Ready queue 成環）
- 死鎖（如所有核心都在等待）
- Task 丟失（如 delay list 損壞）

**調試策略**：
1. 加入狀態 dump（定期輸出 ready queue、delay list 狀態）
2. 加入 sanity check（如檢查 task state 是否合法）
3. 使用 watchdog timer（如果系統停止，強制輸出診斷訊息）

---

## 五、本章回顧

在這一章，我們經歷了從 2-core 到 4-core 的實戰擴展：

1. **問題 1**：帶 timeout 的阻塞操作，被 signal 喚醒時沒有從 delay list 移除
2. **問題 2**：ready queue 和 delay list 共用 next 指針，導致鏈表損壞

關鍵教訓：

| 教訓 | 說明 |
|------|------|
| 雙重等待雙重清理 | Task 在多個等待結構中時，喚醒時都要清理 |
| 避免共用指針 | 每個鏈表用獨立的 link 指針 |
| 多核心測試 | 增加核心數會暴露隱藏的 race condition |
| 為「安靜失敗」準備 | 加入狀態 dump 和 sanity check |

至此，danieRTOS v2.x 的 4-core SMP 支援已經完成！

---

## 參考資料

- **FreeRTOS SMP**, Task State Machine
- **Linux Kernel**, Wait Queue Implementation
- **RISC-V Privileged Spec**, Multi-hart Systems

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。
