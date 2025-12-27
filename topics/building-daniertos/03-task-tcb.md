# Task 結構與 TCB 設計

**作者**: Danny Jiang
**日期**: 2025-12-13

---

## 前言：RTOS 的靈魂——讓 CPU「記住」每個 Task

我還記得第一次嘗試設計 TCB 時的窘境。

我打開 FreeRTOS 的 `tasks.c`，看到那個龐大的 `tskTCB` 結構——超過 20 個欄位，一堆 `#if` 巨集，光是看懂每個欄位的用途就花了我一整天。

「這也太複雜了吧，」我心想，「一個 Task 需要記住這麼多東西嗎？」

後來我才明白，那是二十年來累積的功能疊加。對於一個教育用途的 RTOS，我們需要的是：**找到最小但完整的核心。**

這就像設計一棟房子。你可以參考豪華別墅的設計，但如果你只是要建一個「能住的地方」，你需要的是：牆壁、屋頂、門窗——不是游泳池和健身房。

在上一篇文章中，我們讓程式從 Reset 一路跑到 `main()`，還設定好了 Timer Interrupt。現在 CPU 每隔 1ms 就會跳進我們的 Trap Handler。

但這有什麼用？**現在只有一個「主程式」在跑，根本沒有「任務」可以切換。**

要讓 RTOS 真正發揮作用，我們需要回答一個根本問題：

**「Task」在程式碼中到底是什麼？CPU 要如何「記住」一個被暫停的 Task，並在稍後「恢復」它？**

答案就是 **TCB（Task Control Block）**——每個 Task 的身分證。

> 💡 **設計原則**：我們會從「最小可行」出發，只加入真正需要的欄位。這樣設計錯誤的代價最低，也最容易理解。

本文將帶你設計 danieRTOS 的 TCB，從「需要記住什麼」出發，一步步推導出最終的資料結構。讀完這篇文章，你將能夠：

- 理解 TCB 的核心欄位及其設計理由
- 學會如何「偽造」Stack 讓新 Task「看起來像」被中斷過
- 掌握 Task 狀態機的設計
- 了解 Task List 的管理策略

---

## 一、思考問題：CPU 需要「記住」什麼？

### 1.1 一個簡單的思想實驗

假設你正在用計算機算一道複雜的數學題：

```
(17 × 23 + 45) ÷ 8 - 13
```

你算到一半：`17 × 23 = 391`，`391 + 45 = 436`，正準備除以 8...

這時候老闆叫你去開會。

開完會回來，你想繼續算。但你需要記住什麼才能「恢復」計算？

1. **中間結果**：`436`（到目前為止的計算結果）
2. **進度**：「已經做到除法」
3. **接下來的步驟**：除以 8，再減 13

如果這些資訊遺失了，你只能從頭算起。

**CPU 切換 Task 也是同樣的道理。**

### 1.2 CPU 的「中間狀態」是什麼？

對 CPU 來說，「中間狀態」就是 **暫存器的值**：

- **通用暫存器（x1-x31）**：存放計算的中間結果、變數的值、函數參數
- **Program Counter（mepc）**：執行到哪一行了？
- **狀態暫存器（mstatus）**：中斷是開還是關？特權模式是什麼？

如果我們在 Task A 執行到一半時，把這些暫存器的值「拍照」保存下來，然後載入 Task B 的暫存器值，CPU 就會以為自己一直在執行 Task B。

**這就是 Context Switch 的本質：偷天換日，讓 CPU 產生「錯覺」。**

---

## 二、TCB 設計：Task 的身分證

### 2.1 核心欄位

基於上面的思考，TCB 至少需要記錄：

1. **Stack Pointer**：指向這個 Task 保存的 Context（暫存器值）
2. **Priority**：這個 Task 有多重要？
3. **State**：這個 Task 現在在幹嘛？（執行中？等待中？）
4. **Name**：除錯時用來辨識

```c
typedef struct tcb {
    // === 核心欄位 ===
    volatile uint64_t *sp;      // Stack Pointer：指向保存的 Context
    uint32_t priority;          // 優先級
    uint32_t state;             // 狀態（Running, Ready, Blocked）

    // === 輔助欄位 ===
    uint32_t ticks_to_wake;     // 延遲功能：還要睡多久？
    char name[16];              // Task 名稱（除錯救星）

    // === 鏈結串列 ===
    struct tcb *next;           // Ready List 或 Blocked List 的鏈結
    struct tcb *prev;
} tcb_t;
```

**為什麼 `sp` 是最重要的欄位？**

因為 Context Switch 的核心操作就是：

1. 把當前暫存器存到當前 Task 的 Stack
2. 把 `current_tcb->sp` 更新為當前 Stack Pointer
3. 把 `current_tcb` 換成下一個 Task
4. 從 `current_tcb->sp` 恢復暫存器

**只要 `sp` 正確，整個 Task 的狀態就能完美恢復。**

### 2.2 與 FreeRTOS 的比較

如果你打開 FreeRTOS 的 `tasks.c`，會看到一個巨大的 TCB 結構體，充滿了 `#if` 巨集：

```c
// FreeRTOS TCB（簡化版）
typedef struct tskTaskControlBlock
{
    volatile StackType_t *pxTopOfStack;    // 注意：FreeRTOS 用匈牙利命名法
    ListItem_t xStateListItem;             // 複雜的 List 結構
    ListItem_t xEventListItem;
    UBaseType_t uxPriority;
    StackType_t *pxStack;
    char pcTaskName[ configMAX_TASK_NAME_LEN ];

    #if ( portSTACK_GROWTH > 0 )
        StackType_t *pxEndOfStack;
    #endif

    #if ( portCRITICAL_NESTING_IN_TCB == 1 )
        UBaseType_t uxCriticalNesting;
    #endif

    // ... 還有更多條件編譯 ...
} tskTCB;
```

**差異分析**：

| 面向 | FreeRTOS | danieRTOS |
|------|----------|-----------|
| **命名風格** | `pxTopOfStack`（匈牙利） | `sp`（簡潔） |
| **List 實作** | 獨立的 `ListItem_t` 結構 | 直接內嵌 `next/prev` 指標 |
| **條件編譯** | 大量 `#if` 巨集 | 無（固定功能集） |
| **可讀性** | 需要跳躍閱讀 | 一目瞭然 |

FreeRTOS 這樣設計是有原因的——它需要支援數十種 MCU 和各種配置選項。但對於教育用途，這些複雜性只會妨礙理解。

**danieRTOS 的策略：Keep It Simple。**

### 2.3 記憶體配置：靜態 vs 動態

**靜態配置**：

```c
#define MAX_TASKS 8
static tcb_t task_pool[MAX_TASKS];
```

**動態配置**：

```c
tcb_t *tcb = malloc(sizeof(tcb_t));
```

**Phase 1 強烈建議使用靜態配置**，原因如下：

1. **確定性**：記憶體用量在編譯期就確定，不會有 `malloc` 失敗的問題
2. **除錯友善**：在 GDB 中可以直接查看 `task_pool` 陣列
3. **無碎片化**：不需要擔心 Heap 碎片
4. **專注核心**：我們要專注在 Scheduler 邏輯，而不是 `malloc` 的 bug

---

## 三、Stack 初始化：偽造一個「被中斷過」的 Task

這是 Context Switch 能夠運作的關鍵魔法。

### 3.1 問題：新 Task 從來沒執行過

當我們創建一個新 Task 時，它根本沒有「被中斷過」，Stack 上沒有任何保存的 Context。

但 Context Switch 的邏輯是：從 Stack 恢復暫存器 → 執行 `mret` → 跳到 `mepc` 指向的地址。

**如果 Stack 是空的，CPU 會恢復到什麼狀態？答案是：垃圾值，然後 Crash。**

### 3.2 解法：偽造 Context

既然 Context Switch 期待 Stack 上有保存的暫存器，我們就**手動偽造一個**！

我們要讓新 Task 的 Stack「看起來像」：

1. 這個 Task 正在執行它的入口函數
2. 突然被中斷打斷
3. 所有暫存器都保存到 Stack 上

這樣當 Scheduler 選中這個 Task 並執行 Context Restore 時，CPU 就會「返回」到 Task 的入口函數開始執行。

### 3.3 Stack Frame 設計

在 RISC-V RV64 上，每個暫存器 8 bytes，我們需要保存：

- 31 個通用暫存器（x1-x31，x0 永遠是 0 不用存）
- 2 個 CSR（mepc、mstatus）

總共 33 × 8 = 264 bytes。為了對齊，我們分配 272 bytes（34 slots）。

```
Stack Top (High Address)
        ┌──────────────────────┐
        │ mstatus              │ ← offset 264
        │ mepc                 │ ← offset 256
        │ x31 (t6)             │ ← offset 248
        │ x30 (t5)             │
        │ ...                  │
        │ x2 (sp) [保留]        │ ← offset 16
        │ x1 (ra)              │ ← offset 8
        │ [reserved]           │ ← offset 0, SP 指向這裡
        └──────────────────────┘
Stack Bottom (Low Address)
```

### 3.4 初始化程式碼

```c
// 偽造 Stack，讓新 Task 看起來像被中斷過
void task_stack_init(tcb_t *tcb, void (*entry)(void), uint64_t *stack_top) {
    // Stack 從高地址往低地址成長
    uint64_t *sp = stack_top;

    // 分配 Context Frame
    sp -= 34;  // 34 slots × 8 bytes = 272 bytes

    // 填入 CSR
    sp[33] = 0x1880;            // mstatus: MPIE=1, MPP=11 (M-mode)
    sp[32] = (uint64_t)entry;   // mepc: Task 入口函數地址

    // 填入通用暫存器（大部分設為 0 或除錯用的魔術數字）
    sp[31] = 0;                 // x31 (t6)
    sp[30] = 0;                 // x30 (t5)
    // ... x3-x29 都設為 0 ...
    sp[2] = 0;                  // x2 (sp) - 會被 Context Restore 忽略
    sp[1] = (uint64_t)task_exit_handler;  // x1 (ra) - 如果 Task return，跳到這裡
    sp[0] = 0;                  // reserved

    // 保存 Stack Pointer 到 TCB
    tcb->sp = sp;
}

// Task 意外 return 時的處理
void task_exit_handler(void) {
    uart_puts("Task unexpectedly returned!\n");
    while (1);  // 或者刪除這個 Task
}
```

**關鍵欄位解釋**：

| 欄位 | 值 | 為什麼？ |
|------|-----|---------|
| `mepc` | `entry` | Context Restore 後，`mret` 會跳到這裡 |
| `mstatus` | `0x1880` | MPIE=1 確保 `mret` 後中斷開啟；MPP=11 確保在 M-mode |
| `ra` | `task_exit_handler` | 如果 Task 函數執行 `return`，會跳到這裡處理 |

### 3.5 Stack Overflow 檢測

Stack Overflow 是 RTOS 開發中最常見的 bug 之一。一個 Task 用光了自己的 Stack，會覆蓋到其他記憶體區域，導致神秘的 Crash。

**Canary 檢測法**：

```c
#define STACK_CANARY 0xA5A5A5A5A5A5A5A5

void task_create(tcb_t *tcb, void (*entry)(void),
                 uint64_t *stack_bottom, size_t stack_size) {
    // 填滿整個 Stack 區域
    for (size_t i = 0; i < stack_size / 8; i++) {
        stack_bottom[i] = STACK_CANARY;
    }

    // 初始化 Stack（從頂端開始）
    uint64_t *stack_top = stack_bottom + (stack_size / 8);
    task_stack_init(tcb, entry, stack_top);
}

// 在 Tick Interrupt 中檢查
void check_stack_overflow(tcb_t *tcb, uint64_t *stack_bottom) {
    if (*stack_bottom != STACK_CANARY) {
        danie_panic("Stack overflow detected!");
    }
}
```

**為什麼是 0xA5？**

- `0x00` 太常見（未初始化的記憶體可能就是 0）
- `0xFF` 也太常見
- `0xA5` 的二進位是 `10100101`，在 Hex Dump 中非常顯眼
- 而且它是奇數，容易抓出對齊錯誤

---

## 四、Task 狀態機

### 4.1 三個基本狀態

對於 Phase 1，我們只需要三個狀態：

```c
typedef enum {
    TASK_RUNNING,   // 正在執行
    TASK_READY,     // 可以執行，但在排隊
    TASK_BLOCKED    // 被阻塞，等待某個事件
} task_state_t;
```

**重要觀察：在單核心系統中，同一時間只有一個 Task 處於 RUNNING 狀態。**

### 4.2 狀態轉換圖

```
                    ┌─────────────┐
                    │   BLOCKED   │
                    └──────┬──────┘
                           │
                     delay 時間到
                     (Tick Interrupt)
                           │
                           ▼
┌──────────┐  Scheduler   ┌─────────────┐
│ RUNNING  │◄────────────►│    READY    │
└──────────┘  選中/時間片  └─────────────┘
     │         用完             ▲
     │                          │
     └──────────────────────────┘
              呼叫 delay()
```

**狀態轉換觸發條件**：

| 轉換 | 觸發條件 |
|------|----------|
| READY → RUNNING | Scheduler 選中這個 Task |
| RUNNING → READY | 時間片用完（Tick Interrupt）或被高優先級 Task 搶佔 |
| RUNNING → BLOCKED | Task 主動呼叫 `danie_delay()` |
| BLOCKED → READY | 延遲時間到期（在 Tick Interrupt 中檢查） |

### 4.3 實作範例

```c
// 全域變數：當前執行的 Task
tcb_t *current_tcb = NULL;

void scheduler(void) {
    // 如果當前 Task 是 RUNNING，變成 READY
    if (current_tcb && current_tcb->state == TASK_RUNNING) {
        current_tcb->state = TASK_READY;
    }

    // 找到最高優先級的 READY Task
    tcb_t *next = find_highest_priority_ready_task();

    if (next) {
        next->state = TASK_RUNNING;
        current_tcb = next;
    }
}

void danie_delay(uint32_t ticks) {
    // 把當前 Task 設為 BLOCKED
    current_tcb->state = TASK_BLOCKED;
    current_tcb->ticks_to_wake = tick_count + ticks;

    // 觸發重新排程
    scheduler();
    context_switch();
}

// 在 Tick Interrupt 中呼叫
void tick_handler(void) {
    tick_count++;

    // 檢查是否有 Task 需要喚醒
    for (int i = 0; i < MAX_TASKS; i++) {
        tcb_t *t = &task_pool[i];
        if (t->state == TASK_BLOCKED &&
            tick_count >= t->ticks_to_wake) {
            t->state = TASK_READY;
        }
    }

    // 觸發排程
    scheduler();
}
```

---

## 五、Task List 管理

### 5.1 為什麼需要 List？

Scheduler 需要快速找到「下一個要執行的 Task」。如果每次都遍歷整個 `task_pool` 陣列，效率很低。

更好的做法是維護一個 **Ready List**：只包含狀態為 READY 的 Task。

### 5.2 資料結構選擇

**Option 1：單一 Linked List**

```c
tcb_t *ready_list_head;
```

找最高優先級需要遍歷整個 List，時間複雜度 O(N)。

**Option 2：Priority Array of Lists（FreeRTOS 方式）**

```c
#define MAX_PRIORITY 8
tcb_t *ready_lists[MAX_PRIORITY];
```

每個優先級一個 List。找最高優先級只需要從陣列頂端開始，找到第一個非空的 List，時間複雜度 O(1)。

**danieRTOS 採用 Option 2**，因為：

1. 效率高：找最高優先級是 O(1)
2. 直觀：優先級直接對應陣列索引
3. 簡單：每個 List 內的 Task 優先級相同，可以用 Round-Robin 輪轉

### 5.3 實作範例

```c
#define MAX_PRIORITY 8

// 每個優先級一個 Ready List
tcb_t *ready_lists[MAX_PRIORITY] = {NULL};

// 加入 Ready List
void ready_list_add(tcb_t *tcb) {
    uint32_t prio = tcb->priority;

    // 插入到 List 頭部
    tcb->next = ready_lists[prio];
    tcb->prev = NULL;
    if (ready_lists[prio]) {
        ready_lists[prio]->prev = tcb;
    }
    ready_lists[prio] = tcb;
}

// 從 Ready List 移除
void ready_list_remove(tcb_t *tcb) {
    uint32_t prio = tcb->priority;

    if (tcb->prev) {
        tcb->prev->next = tcb->next;
    } else {
        ready_lists[prio] = tcb->next;
    }
    if (tcb->next) {
        tcb->next->prev = tcb->prev;
    }

    tcb->next = tcb->prev = NULL;
}

// 找最高優先級的 Ready Task
tcb_t *find_highest_priority_ready_task(void) {
    // 從最高優先級開始找
    for (int prio = MAX_PRIORITY - 1; prio >= 0; prio--) {
        if (ready_lists[prio]) {
            return ready_lists[prio];
        }
    }
    return NULL;  // 沒有 Ready Task（應該不會發生）
}
```

---

## 總結

本文設計了 danieRTOS 的 TCB 和 Task 管理機制：

1. **TCB 核心欄位**：`sp`（Stack Pointer）是最重要的欄位，決定了 Task 的 Context
2. **Stack 初始化**：偽造一個「被中斷過」的 Stack Frame，讓新 Task 可以被 Context Switch
3. **Task 狀態機**：RUNNING、READY、BLOCKED 三個狀態，配合 Scheduler 和 Tick Interrupt 轉換
4. **Task List**：Priority Array of Lists，O(1) 找到最高優先級 Task

現在我們有了 Task 的「身分證」（TCB），也知道如何「偽造」一個新 Task 的 Context。

下一步，就是真正實作 Context Switch——讓 CPU 在不同 Task 之間「跳來跳去」。

---

## 參考資料

**RTOS 參考實作**

- **FreeRTOS Kernel - tasks.c**
  https://github.com/FreeRTOS/FreeRTOS-Kernel
  FreeRTOS 的 TCB 結構設計，本文的主要比較對象。

**延伸閱讀**

- **Data Structures in Practice**
  Danny Jiang
  Linked List 和 Priority Queue 的實作細節，用於 Task List 管理。

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)
