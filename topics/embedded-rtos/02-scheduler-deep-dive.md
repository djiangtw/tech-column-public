# 深入 RTOS Scheduler：從 Ready List 到 Context Switch

**作者**: Danny Jiang
**日期**: 2025-12-09

---

## 前言：一個讓人抓狂的性能問題

2017 年，我在一家物聯網設備公司擔任系統軟體工程師。有一天，測試團隊報告了一個奇怪的問題：

**系統在高負載時，Task 切換時間從 5 微秒暴增到 50 微秒，導致即時性需求無法滿足。**

我們的產品是一個工業感測器網關，需要同時處理：

- 10 個感測器的資料採集（每個 100 ms 一次）
- 1 個網路通訊 Task（處理 MQTT）
- 1 個資料處理 Task（計算統計值）
- 1 個 UI 更新 Task（LCD 顯示）

總共 13 個 Task，優先權從 1 到 5 不等。

經過一週的除錯，我發現問題出在 **Scheduler 的實作細節**：

**我們自己寫的 Scheduler 使用單一 Linked List 管理所有 Ready Tasks，每次選擇下一個 Task 時，需要遍歷整個 List 找到最高優先權的 Task。**

```c
// 我們的錯誤實作（簡化版）
task_t *find_highest_priority_task(void) {
    task_t *highest = NULL;
    task_t *curr = ready_list;
    
    while (curr != NULL) {  // O(n) 遍歷！
        if (highest == NULL || curr->priority > highest->priority) {
            highest = curr;
        }
        curr = curr->next;
    }
    
    return highest;
}
```

**問題**：

- 1 個 Task：遍歷 1 次，~5 微秒 ✅
- 13 個 Task：遍歷 13 次，~50 微秒 ❌

最後，我們改用 **FreeRTOS 的 Ready List 設計**：Array of Linked Lists，每個優先權一個 List。Task 切換時間回到 5 微秒，問題完全解決。

這個慘痛的教訓讓我深刻體會到：**Scheduler 的資料結構設計，直接影響系統的即時性。**

本文將深入探討 FreeRTOS Scheduler 的實作細節。讀完這篇文章，你將能夠：

- 理解 Ready List 的資料結構設計（Array of Linked Lists）
- 知道如何用 Bitmap 實現 O(1) 的優先權查找
- 掌握 Tick Interrupt 觸發 Context Switch 的完整流程
- 了解 vTaskSwitchContext() 的運作機制
- 理解 Round Robin 與 Time Slicing 的實作

---

## 一、Task 的狀態機

在深入 Scheduler 之前，我們需要先理解 Task 的狀態。

### 1.1 四種 Task 狀態

FreeRTOS 的 Task 有四種狀態：

**1. Running（執行中）**

- Task 正在 CPU 上執行
- 單核心環境下，同一時間只有一個 Task 處於 Running 狀態

**2. Ready（就緒）**

- Task 準備好執行，但還沒輪到
- 等待 Scheduler 分配 CPU 時間

**3. Blocked（阻塞）**

- Task 在等待某個事件（例如：Queue, Semaphore, Delay）
- 不會被 Scheduler 選中執行

**4. Suspended（暫停）**

- Task 被明確暫停（呼叫 vTaskSuspend()）
- 需要明確恢復（呼叫 vTaskResume()）

### 1.2 狀態轉換圖

```
                    vTaskSuspend()
         ┌──────────────────────────────────┐
         │                                  │
         ▼                                  │
    ┌─────────┐                        ┌────────────┐
    │Suspended│◄───────────────────────┤  Running   │
    └─────────┘    vTaskSuspend()      └────────────┘
         │                                  ▲    │
         │ vTaskResume()                    │    │
         ▼                                  │    │
    ┌─────────┐    Scheduler選中           │    │ Preempted或
    │  Ready  │───────────────────────────►│    │ vTaskDelay()
    └─────────┘                                  │
         ▲                                       │
         │                                       ▼
         │         Event發生              ┌────────────┐
         └───────────────────────────────┤  Blocked   │
                                         └────────────┘
```

### 1.3 狀態轉換的觸發條件

**Running → Ready**：

- 高優先權 Task 變成 Ready（被搶佔）
- Time Slice 用完（Round Robin）

**Running → Blocked**：

- 呼叫 vTaskDelay()
- 等待 Queue（xQueueReceive()）
- 等待 Semaphore（xSemaphoreTake()）

**Blocked → Ready**：

- Delay 時間到
- Queue 有資料可讀
- Semaphore 被釋放

**Ready → Running**：

- Scheduler 選中這個 Task

**Running/Ready → Suspended**：

- 呼叫 vTaskSuspend()

**Suspended → Ready**：

- 呼叫 vTaskResume()

---

## 二、Ready List 的資料結構設計

現在讓我們看看 FreeRTOS 如何管理 Ready Tasks。

### 2.1 為什麼不用單一 Linked List？

回到前言的問題：單一 Linked List 的問題是什麼？

**問題 #1：O(n) 查找**

- 每次選擇下一個 Task，需要遍歷整個 List
- Task 數量越多，時間越長
- 不符合 RTOS 的「確定性」要求

**問題 #2：不可預測的性能**

- 1 個 Task：5 微秒
- 10 個 Task：50 微秒
- 100 個 Task：500 微秒

**RTOS 的要求**：

- Context Switch 時間必須是**固定的**
- 不能隨著 Task 數量增加而增加

### 2.2 FreeRTOS 的解決方案：Array of Linked Lists

FreeRTOS 使用一個巧妙的設計：**Array of Linked Lists**。

**核心概念**：

- 每個優先權有自己的 Linked List
- 所有 Lists 放在一個 Array 中
- Array 的 Index 就是優先權

**資料結構**：

```c
#define configMAX_PRIORITIES 32  // 最多 32 個優先權

// 每個優先權一個 List
List_t pxReadyTasksLists[configMAX_PRIORITIES];

// Bitmap：記錄哪些優先權有 Ready Tasks
UBaseType_t uxTopReadyPriority;
```

**比喻**：想像一個停車場，有 32 個停車區（優先權 0-31），每個停車區有一排停車位（Linked List）。

**優勢**：

1. **O(1) 插入**：知道優先權，直接插入對應的 List
2. **O(1) 查找**：用 Bitmap 快速找到最高優先權
3. **O(1) 刪除**：從對應的 List 中刪除

### 2.3 Bitmap 的魔法：O(1) 查找最高優先權

**問題**：如何快速找到哪個優先權有 Ready Tasks？

**FreeRTOS 的解決方案**：使用 Bitmap。

**Bitmap 的設計**：

```c
// 32-bit Bitmap（假設 configMAX_PRIORITIES = 32）
// 每個 bit 代表一個優先權
// bit = 1：該優先權有 Ready Tasks
// bit = 0：該優先權沒有 Ready Tasks

UBaseType_t uxTopReadyPriority = 0;

// 範例：
// 優先權 0, 2, 5 有 Ready Tasks
// uxTopReadyPriority = 0b00000000000000000000000000100101
//                                                    ↑ ↑ ↑
//                                                    5 2 0
```

**設定 Bit（Task 變成 Ready）**：

```c
// Task 的優先權是 5
uxTopReadyPriority |= (1 << 5);  // 設定 bit 5
```

**清除 Bit（該優先權沒有 Ready Tasks）**：

```c
// 優先權 5 的 List 變空了
uxTopReadyPriority &= ~(1 << 5);  // 清除 bit 5
```

**查找最高優先權（關鍵！）**：

```c
// 使用 CLZ (Count Leading Zeros) 指令
// CLZ 計算最高位的 1 之前有多少個 0

// 範例：uxTopReadyPriority = 0b00000000000000000000000000100101
// CLZ(0b00000000000000000000000000100101) = 26
// 最高優先權 = 31 - 26 = 5

UBaseType_t uxTopPriority = 31 - __builtin_clz(uxTopReadyPriority);
```

**為什麼這麼快？**

- CLZ 是 CPU 的硬體指令（RISC-V: `clz`, ARM: `clz`）
- 執行時間：1 個 CPU Cycle
- **O(1) 時間複雜度**

**如果 CPU 沒有 CLZ 指令怎麼辦？**

```c
// 軟體實作（查表法）
static const uint8_t clz_table[16] = {
    4, 3, 2, 2, 1, 1, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0
};

UBaseType_t software_clz(UBaseType_t x) {
    UBaseType_t n = 0;
    if (x == 0) return 32;

    if ((x & 0xFFFF0000) == 0) { n += 16; x <<= 16; }
    if ((x & 0xFF000000) == 0) { n += 8;  x <<= 8;  }
    if ((x & 0xF0000000) == 0) { n += 4;  x <<= 4;  }

    return n + clz_table[x >> 28];
}
```

### 2.4 完整的 Ready List 操作

**插入 Task 到 Ready List**：

```c
void vListInsertEnd(List_t *pxList, ListItem_t *pxNewListItem) {
    // 插入到 List 的尾端（O(1)）
    ListItem_t *pxIndex = pxList->pxIndex;

    pxNewListItem->pxNext = pxIndex;
    pxNewListItem->pxPrevious = pxIndex->pxPrevious;
    pxIndex->pxPrevious->pxNext = pxNewListItem;
    pxIndex->pxPrevious = pxNewListItem;

    pxList->uxNumberOfItems++;
}

void prvAddTaskToReadyList(TCB_t *pxTCB) {
    UBaseType_t uxPriority = pxTCB->uxPriority;

    // 1. 插入到對應優先權的 List
    vListInsertEnd(&pxReadyTasksLists[uxPriority], &pxTCB->xStateListItem);

    // 2. 設定 Bitmap
    uxTopReadyPriority |= (1 << uxPriority);
}
```

**從 Ready List 移除 Task**：

```c
void uxListRemove(ListItem_t *pxItemToRemove) {
    // 從 List 中移除（O(1)）
    pxItemToRemove->pxNext->pxPrevious = pxItemToRemove->pxPrevious;
    pxItemToRemove->pxPrevious->pxNext = pxItemToRemove->pxNext;

    List_t *pxList = pxItemToRemove->pxContainer;
    pxList->uxNumberOfItems--;
}

void prvRemoveTaskFromReadyList(TCB_t *pxTCB) {
    UBaseType_t uxPriority = pxTCB->uxPriority;

    // 1. 從 List 中移除
    uxListRemove(&pxTCB->xStateListItem);

    // 2. 如果該優先權的 List 變空了，清除 Bitmap
    if (listLIST_IS_EMPTY(&pxReadyTasksLists[uxPriority])) {
        uxTopReadyPriority &= ~(1 << uxPriority);
    }
}
```

**選擇下一個 Task（關鍵！）**：

```c
TCB_t *prvSelectNextTask(void) {
    // 1. 找到最高優先權（O(1)）
    UBaseType_t uxTopPriority = 31 - __builtin_clz(uxTopReadyPriority);

    // 2. 從該優先權的 List 中取出第一個 Task（O(1)）
    List_t *pxList = &pxReadyTasksLists[uxTopPriority];
    ListItem_t *pxListItem = listGET_HEAD_ENTRY(pxList);
    TCB_t *pxTCB = listGET_LIST_ITEM_OWNER(pxListItem);

    return pxTCB;
}
```

**時間複雜度總結**：

- 插入 Task：O(1)
- 移除 Task：O(1)
- 選擇下一個 Task：O(1)

**與單一 Linked List 的比較**：

| 操作 | 單一 Linked List | Array of Linked Lists |
|------|------------------|------------------------|
| 插入 Task | O(1) | O(1) |
| 移除 Task | O(1) | O(1) |
| 選擇下一個 Task | **O(n)** ❌ | **O(1)** ✅ |
| 記憶體使用 | 較少 | 較多（Array） |

---

## 三、Priority-Based Preemptive Scheduling

現在讓我們看看 Scheduler 如何選擇下一個 Task。

### 3.1 Scheduling 的核心原則

FreeRTOS 的 Scheduler 遵循一個簡單的原則：

**最高優先權的 Ready Task 一定會執行。**

**規則**：

1. 如果有多個 Task 處於 Ready 狀態，選擇優先權最高的
2. 如果多個 Task 有相同的最高優先權，使用 Round Robin
3. 高優先權 Task 可以搶佔低優先權 Task

### 3.2 Preemption（搶佔）的觸發時機

**什麼時候會發生 Preemption？**

**時機 #1：Tick Interrupt**

- 每個 Tick（例如：1 ms），檢查是否需要切換 Task
- 如果有更高優先權的 Task 變成 Ready，立即切換

**時機 #2：Task 主動讓出 CPU**

- 呼叫 vTaskDelay()
- 呼叫 xQueueReceive()（Blocked）
- 呼叫 xSemaphoreTake()（Blocked）

**時機 #3：ISR 中的 Task 喚醒**

- ISR 中呼叫 xQueueSendFromISR()
- ISR 中呼叫 xSemaphoreGiveFromISR()
- 如果喚醒的 Task 優先權更高，ISR 返回後立即切換

### 3.3 vTaskSwitchContext() 的運作機制

**vTaskSwitchContext()** 是 Scheduler 的核心函式，負責選擇下一個 Task。

**完整流程**：

```c
void vTaskSwitchContext(void) {
    // 1. 檢查是否有 Task 需要 Yield
    if (uxSchedulerSuspended == 0) {
        // 2. 找到最高優先權（O(1)）
        UBaseType_t uxTopPriority = 31 - __builtin_clz(uxTopReadyPriority);

        // 3. 從該優先權的 List 中取出下一個 Task
        List_t *pxList = &pxReadyTasksLists[uxTopPriority];

        // 4. Round Robin：移動 List Index 到下一個
        listGET_OWNER_OF_NEXT_ENTRY(pxCurrentTCB, pxList);

        // 5. pxCurrentTCB 現在指向下一個要執行的 Task
    }
}
```

**關鍵步驟解析**：

**步驟 1：檢查 Scheduler 是否被暫停**

```c
if (uxSchedulerSuspended == 0) {
    // Scheduler 正常運行
}
```

**步驟 2：找到最高優先權（O(1)）**

```c
UBaseType_t uxTopPriority = 31 - __builtin_clz(uxTopReadyPriority);
```

**步驟 3-4：Round Robin 選擇下一個 Task**

```c
// listGET_OWNER_OF_NEXT_ENTRY 的實作
#define listGET_OWNER_OF_NEXT_ENTRY(pxTCB, pxList) \
{ \
    List_t *pxConstList = (pxList); \
    /* 移動 Index 到下一個 */ \
    (pxConstList)->pxIndex = (pxConstList)->pxIndex->pxNext; \
    /* 如果到了 List 的尾端，回到開頭 */ \
    if ((pxConstList)->pxIndex == &((pxConstList)->xListEnd)) { \
        (pxConstList)->pxIndex = (pxConstList)->pxIndex->pxNext; \
    } \
    /* 取得 Task 的 TCB */ \
    (pxTCB) = (pxConstList)->pxIndex->pvOwner; \
}
```

**步驟 5：更新 pxCurrentTCB**

```c
// pxCurrentTCB 是全域變數，指向當前執行的 Task
TCB_t *pxCurrentTCB;
```

---

## 四、Tick Interrupt 與 Context Switch

現在讓我們看看 Tick Interrupt 如何觸發 Context Switch。

### 4.1 什麼是 Tick Interrupt？

**Tick Interrupt** 是 RTOS 的心跳，定期觸發（例如：每 1 ms）。

**作用**：

1. 更新系統時間（Tick Count）
2. 檢查 Delayed Tasks 是否該喚醒
3. 檢查是否需要 Context Switch
4. 實現 Round Robin（Time Slicing）

**RISC-V 的 Timer Interrupt**：

```c
// RISC-V 使用 mtime 和 mtimecmp 實現 Timer
// mtime：當前時間（64-bit counter）
// mtimecmp：比較值（64-bit）
// 當 mtime >= mtimecmp 時，觸發 Timer Interrupt

#define MTIME_ADDR    0x0200BFF8  // QEMU virt machine
#define MTIMECMP_ADDR 0x02004000

void vPortSetupTimerInterrupt(void) {
    uint64_t *mtime = (uint64_t *)MTIME_ADDR;
    uint64_t *mtimecmp = (uint64_t *)MTIMECMP_ADDR;

    // 設定下一次 Interrupt 的時間
    // configTICK_RATE_HZ = 1000（1 ms）
    // CPU_FREQ = 10 MHz
    uint64_t next_tick = *mtime + (CPU_FREQ / configTICK_RATE_HZ);
    *mtimecmp = next_tick;

    // 啟用 Timer Interrupt
    set_csr(mie, MIE_MTIE);  // Machine Timer Interrupt Enable
}
```

### 4.2 Tick Interrupt Handler 的完整流程

**xPortSysTickHandler()** 是 Tick Interrupt 的處理函式。

**完整流程**：

```c
void xPortSysTickHandler(void) {
    // 1. 暫停 Scheduler（避免重入問題）
    vTaskSuspendAll();

    // 2. 更新 Tick Count，檢查 Delayed Tasks
    BaseType_t xSwitchRequired = xTaskIncrementTick();

    // 3. 恢復 Scheduler
    xTaskResumeAll();

    // 4. 如果需要 Context Switch，觸發 PendSV
    if (xSwitchRequired != pdFALSE) {
        portYIELD();  // 觸發 Context Switch
    }

    // 5. 設定下一次 Tick Interrupt
    uint64_t *mtime = (uint64_t *)MTIME_ADDR;
    uint64_t *mtimecmp = (uint64_t *)MTIMECMP_ADDR;
    *mtimecmp = *mtime + (CPU_FREQ / configTICK_RATE_HZ);
}
```

**關鍵步驟解析**：

**步驟 1：暫停 Scheduler**

```c
void vTaskSuspendAll(void) {
    ++uxSchedulerSuspended;  // 計數器 +1
}
```

**步驟 2：xTaskIncrementTick() 的運作**

```c
BaseType_t xTaskIncrementTick(void) {
    BaseType_t xSwitchRequired = pdFALSE;

    // 1. Tick Count +1
    ++xTickCount;

    // 2. 檢查 Delayed Tasks List
    if (listLIST_IS_EMPTY(pxDelayedTaskList) == pdFALSE) {
        // 遍歷 Delayed Tasks，檢查是否該喚醒
        while (listGET_ITEM_VALUE_OF_HEAD_ENTRY(pxDelayedTaskList) <= xTickCount) {
            // 從 Delayed List 移除
            TCB_t *pxTCB = listGET_OWNER_OF_HEAD_ENTRY(pxDelayedTaskList);
            uxListRemove(&pxTCB->xStateListItem);

            // 加入 Ready List
            prvAddTaskToReadyList(pxTCB);

            // 如果喚醒的 Task 優先權更高，需要 Context Switch
            if (pxTCB->uxPriority >= pxCurrentTCB->uxPriority) {
                xSwitchRequired = pdTRUE;
            }
        }
    }

    // 3. Round Robin：Time Slice 用完
    #if (configUSE_TIME_SLICING == 1)
    {
        if (listCURRENT_LIST_LENGTH(&pxReadyTasksLists[pxCurrentTCB->uxPriority]) > 1) {
            xSwitchRequired = pdTRUE;  // 同優先權有多個 Task，需要切換
        }
    }
    #endif

    return xSwitchRequired;
}
```

**步驟 3：恢復 Scheduler**

```c
void xTaskResumeAll(void) {
    --uxSchedulerSuspended;  // 計數器 -1

    if (uxSchedulerSuspended == 0) {
        // 如果有 Pending 的 Context Switch，現在執行
        if (xYieldPending != pdFALSE) {
            portYIELD();
        }
    }
}
```

**步驟 4：觸發 Context Switch**

```c
#define portYIELD() \
{ \
    /* 觸發軟體中斷（RISC-V 使用 ecall） */ \
    __asm volatile ("ecall"); \
}
```

### 4.3 Context Switch 的完整流程

**Context Switch** 是 Scheduler 的核心操作，負責保存當前 Task 的狀態，並恢復下一個 Task 的狀態。

**完整流程**（RISC-V）：

**1. 保存當前 Task 的 Context**

```asm
/* portASM.S */
.global xPortStartScheduler
.global vPortYield

vPortYield:
    /* 1. 保存所有暫存器到 Stack */
    addi sp, sp, -32*4  /* 分配 Stack 空間（32 個暫存器） */

    sw x1,  0*4(sp)     /* ra (return address) */
    sw x5,  1*4(sp)     /* t0 */
    sw x6,  2*4(sp)     /* t1 */
    /* ... 保存 x7-x31 ... */

    /* 2. 保存 CSRs */
    csrr t0, mepc       /* Machine Exception PC */
    sw t0, 31*4(sp)

    /* 3. 保存 Stack Pointer 到 TCB */
    la t0, pxCurrentTCB
    lw t1, 0(t0)        /* t1 = pxCurrentTCB */
    sw sp, 0(t1)        /* pxCurrentTCB->pxTopOfStack = sp */

    /* 4. 呼叫 vTaskSwitchContext() 選擇下一個 Task */
    call vTaskSwitchContext

    /* 5. 恢復下一個 Task 的 Stack Pointer */
    la t0, pxCurrentTCB
    lw t1, 0(t0)        /* t1 = pxCurrentTCB（新的 Task） */
    lw sp, 0(t1)        /* sp = pxCurrentTCB->pxTopOfStack */

    /* 6. 恢復所有暫存器 */
    lw t0, 31*4(sp)
    csrw mepc, t0       /* 恢復 mepc */

    lw x1,  0*4(sp)     /* ra */
    lw x5,  1*4(sp)     /* t0 */
    /* ... 恢復 x6-x31 ... */

    addi sp, sp, 32*4   /* 釋放 Stack 空間 */

    /* 7. 返回（mret 會跳到 mepc） */
    mret
```

**關鍵步驟**：

1. **保存暫存器**：x1-x31（31 個通用暫存器）
2. **保存 CSRs**：mepc, mstatus
3. **保存 Stack Pointer**：pxCurrentTCB->pxTopOfStack = sp
4. **選擇下一個 Task**：呼叫 vTaskSwitchContext()
5. **恢復 Stack Pointer**：sp = pxCurrentTCB->pxTopOfStack
6. **恢復暫存器**：x1-x31, mepc, mstatus
7. **返回**：mret（跳到 mepc）

**時間成本**：

- 保存暫存器：~32 個 Store 指令
- 恢復暫存器：~32 個 Load 指令
- 總計：~64 個記憶體存取
- 在 100 MHz CPU 上：~5 微秒

---

## 五、Round Robin 與 Time Slicing

最後，讓我們看看 Round Robin 如何實現。

### 5.1 什麼是 Round Robin？

**Round Robin** 是一種公平的排程策略，用於同優先權的 Tasks。

**核心概念**：

- 同優先權的 Tasks 輪流執行
- 每個 Task 執行一個 Time Slice（時間片）
- Time Slice 用完後，切換到下一個 Task

**比喻**：想像一群小朋友輪流玩遊戲機，每個人玩 5 分鐘，時間到就換下一個。

### 5.2 Time Slicing 的實作

**configUSE_TIME_SLICING** 控制是否啟用 Time Slicing。

**啟用 Time Slicing**：

```c
#define configUSE_TIME_SLICING 1
```

**Tick Interrupt 中的檢查**：

```c
#if (configUSE_TIME_SLICING == 1)
{
    // 檢查當前優先權是否有多個 Ready Tasks
    if (listCURRENT_LIST_LENGTH(&pxReadyTasksLists[pxCurrentTCB->uxPriority]) > 1) {
        xSwitchRequired = pdTRUE;  // 需要切換到下一個 Task
    }
}
#endif
```

**vTaskSwitchContext() 中的 Round Robin**：

```c
// listGET_OWNER_OF_NEXT_ENTRY 會移動 List Index 到下一個
listGET_OWNER_OF_NEXT_ENTRY(pxCurrentTCB, pxList);
```

### 5.3 Round Robin 的範例

**場景**：3 個 Task，優先權都是 2

```
Task A (Priority 2)
Task B (Priority 2)
Task C (Priority 2)
```

**Ready List 的狀態**：

```
pxReadyTasksLists[2]: A → B → C → A → ...
                      ↑
                   pxIndex
```

**Tick 0**：

- pxIndex 指向 A
- Task A 執行

**Tick 1**：

- Time Slice 用完
- pxIndex 移動到 B
- Task B 執行

**Tick 2**：

- Time Slice 用完
- pxIndex 移動到 C
- Task C 執行

**Tick 3**：

- Time Slice 用完
- pxIndex 移動到 A（回到開頭）
- Task A 執行

**循環往復**。

### 5.4 禁用 Time Slicing 的效果

**禁用 Time Slicing**：

```c
#define configUSE_TIME_SLICING 0
```

**效果**：

- 同優先權的 Tasks 不會自動切換
- 只有當 Task 主動讓出 CPU（vTaskDelay, xQueueReceive）時才會切換
- 可能導致某個 Task 「霸佔」CPU

**適用場景**：

- 需要最小化 Context Switch 開銷
- Tasks 會主動讓出 CPU

---

## 六、重要澄清：任務搶佔 vs. 巢狀中斷

在討論 RTOS Scheduler 時，有一個常見的誤解需要澄清：

### 6.1 核心觀念

**「任務搶佔 (Task Preemption)」並不依賴「巢狀中斷 (Nested Interrupts)」。**

即便 RISC-V 硬體預設不支援巢狀中斷，FreeRTOS 依然可以完美地實現任務搶佔。

### 6.2 兩者的區別

**任務搶佔 (Task Preemption)**：

- **定義**：中斷打斷正在執行的 **Task**，並切換到另一個 **Task**
- **流程**：Task A → ISR → Task B
- **需求**：只需要「中斷」能打斷「任務」
- **RISC-V 支援**：✅ 完全支援（所有 CPU 的基本功能）

**巢狀中斷 (Nested Interrupt)**：

- **定義**：高優先級中斷打斷正在執行的 **ISR**
- **流程**：ISR A → ISR B → ISR A
- **需求**：需要「中斷」能打斷「中斷」
- **RISC-V 支援**：❌ 硬體預設不支援（需軟體實作）

### 6.3 RISC-V 的中斷行為

**當 Trap 發生時，硬體自動執行**：

```c
// 硬體自動執行（不可見的操作）
mepc = PC;                    // 保存當前 PC
mstatus.MPIE = mstatus.MIE;   // 保存中斷啟用狀態
mstatus.MIE = 0;              // ❗ 自動關閉全域中斷
mcause = interrupt_code;      // 設定中斷原因
PC = mtvec;                   // 跳到 ISR
```

**關鍵點**：`mstatus.MIE = 0` 表示進入 ISR 後，中斷被關閉，因此預設不支援巢狀中斷。

**當 mret 執行時，硬體自動執行**：

```c
// 硬體自動執行（不可見的操作）
PC = mepc;                    // 恢復 PC
mstatus.MIE = mstatus.MPIE;   // ✅ 恢復中斷啟用狀態
```

### 6.4 FreeRTOS 任務搶佔的實際流程

讓我們看看 FreeRTOS 如何在沒有巢狀中斷的情況下，把 Task A 換成 Task B：

**步驟 1：Task A 執行中**

- `mstatus.MIE = 1`（中斷開啟）
- Task A 正在執行

**步驟 2：Timer 中斷觸發**

- 硬體自動：`mstatus.MIE = 0`（關閉中斷）
- 跳轉到 Tick ISR

**步驟 3：在 ISR 中**

- 保存 Task A 的 Context
- 呼叫 `xTaskIncrementTick()`
- FreeRTOS 發現 Task B 優先權更高
- 呼叫 `vTaskSwitchContext()`
- `pxCurrentTCB` 從 Task A 移向 Task B

**步驟 4：退出 ISR**

- 從 Task B 的 Stack 恢復 Context
- 執行 `mret`
- 硬體自動：`mstatus.MIE = 1`（恢復中斷）

**步驟 5：Task B 執行中**

- Task B 開始執行

**結論**：整個過程是 **Task → ISR → Task**，沒有 **ISR → ISR**，因此不需要巢狀中斷。

### 6.5 什麼時候需要巢狀中斷？

雖然 FreeRTOS 不需要巢狀中斷，但某些**即時性要求極高**的應用會需要：

**場景**：

- 正在處理耗時的 UART 中斷（低優先級）
- 突然馬達控制中斷來了（高優先級）
- 如果沒有巢狀中斷，馬達控制必須等 UART 處理完，可能導致馬達抖動

**如何實作**（進階主題）：

```c
void UART_ISR(void) {
    // 1. 保存 CSR（因為巢狀中斷會覆蓋它們）
    uint32_t mstatus_save = read_csr(mstatus);
    uint32_t mepc_save = read_csr(mepc);

    // 2. 手動開啟中斷，允許更高優先級中斷打斷
    set_csr(mstatus, MSTATUS_MIE);

    // --- 耗時的 UART 處理 ---
    // 此時馬達中斷可以進來

    // 3. 處理完畢，關閉中斷
    clear_csr(mstatus, MSTATUS_MIE);

    // 4. 恢復 CSR
    write_csr(mstatus, mstatus_save);
    write_csr(mepc, mepc_save);
}
```

### 6.6 總結

| 概念 | 定義 | RISC-V 支援 | FreeRTOS 需要 |
|------|------|-------------|---------------|
| **任務搶佔** | 中斷打斷 Task | ✅ 硬體支援 | ✅ 必須 |
| **巢狀中斷** | 中斷打斷 ISR | ❌ 需軟體實作 | ❌ 非必須 |

**記住**：

- FreeRTOS 的任務搶佔是靠 ISR 打斷 Task，RISC-V 完全支援
- RISC-V 不支援巢狀中斷只影響中斷延遲，不影響 OS 的多工排程能力
- 本文討論的 Context Switch 都發生在 **Task → ISR → Task** 之間，不涉及 ISR 之間的切換

---

## 七、總結

本文深入探討了 FreeRTOS Scheduler 的實作細節。讓我們回顧重點：

### 核心概念

1. **Task 狀態機**：
   - Running, Ready, Blocked, Suspended
   - 狀態轉換的觸發條件

2. **Ready List 的資料結構**：
   - Array of Linked Lists（每個優先權一個 List）
   - Bitmap 實現 O(1) 優先權查找
   - CLZ 指令加速查找

3. **Priority-Based Preemptive Scheduling**：
   - 最高優先權的 Ready Task 一定會執行
   - vTaskSwitchContext() 選擇下一個 Task
   - O(1) 時間複雜度

4. **Tick Interrupt 與 Context Switch**：
   - Tick Interrupt 是 RTOS 的心跳
   - xTaskIncrementTick() 喚醒 Delayed Tasks
   - Context Switch 保存/恢復暫存器
   - 發生在 Task → ISR → Task 之間，不需要巢狀中斷

5. **Round Robin 與 Time Slicing**：
   - 同優先權 Tasks 輪流執行
   - 每個 Tick 檢查是否需要切換
   - configUSE_TIME_SLICING 控制啟用/禁用

6. **任務搶佔 vs. 巢狀中斷**：
   - 任務搶佔：中斷打斷 Task（RISC-V 完全支援）
   - 巢狀中斷：中斷打斷 ISR（RISC-V 需軟體實作）
   - FreeRTOS 只需要任務搶佔，不需要巢狀中斷

### 實務建議

1. **選擇合適的優先權數量**：
   - configMAX_PRIORITIES 不要設太大（浪費記憶體）
   - 通常 8-32 個優先權就足夠

2. **避免過多的同優先權 Tasks**：
   - Round Robin 會增加 Context Switch 次數
   - 盡量用不同優先權區分 Tasks

3. **監控 Context Switch 開銷**：
   - 使用 FreeRTOS 的 Run Time Stats
   - 確保 Context Switch 時間 < 5 微秒

4. **理解 Scheduler 的限制**：
   - Scheduler 本身也需要 CPU 時間
   - 過多的 Tasks 會增加 Scheduler 開銷

### 下一篇預告

在下一篇文章中，我們將深入探討 **RTOS 中斷處理：從硬體到軟體的完整流程**：

- RISC-V Timer Interrupt 的設定（mtime, mtimecmp, CSRs）
- ISR 中可以安全呼叫的 FreeRTOS API
- FromISR 系列 API 與一般 API 的差異
- QEMU 虛擬硬體（CLINT, PLIC, UART）
- 實際的 ISR 實作範例

敬請期待！

---

## 參考資料

**官方文檔**：

- [FreeRTOS Kernel Developer Docs](https://www.freertos.org/FreeRTOS-Kernel-Developer-Docs.html)
- [FreeRTOS Task States](https://www.freertos.org/RTOS-task-states.html)
- [RISC-V Privileged Specification](https://riscv.org/technical/specifications/)

**原始碼**：

- [FreeRTOS tasks.c](https://github.com/FreeRTOS/FreeRTOS-Kernel/blob/main/tasks.c)
- [FreeRTOS list.c](https://github.com/FreeRTOS/FreeRTOS-Kernel/blob/main/list.c)
- [FreeRTOS RISC-V Port](https://github.com/FreeRTOS/FreeRTOS-Kernel/tree/main/portable/GCC/RISC-V)

**技術文章**：

- [Understanding FreeRTOS Scheduler](https://www.embedded.com/understanding-freertos-scheduler/)
- [RISC-V Timer Interrupt](https://five-embeddev.com/riscv-isa-manual/latest/machine.html#machine-timer-registers-mtime-and-mtimecmp)

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
