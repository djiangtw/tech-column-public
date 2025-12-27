# 看不見的同步：Cache Coherency

**作者**: Danny Jiang
**日期**: 2025-12-14

---

## 前言：一個詭異的 Bug

有一天，你的多核心 RTOS 出現了這個問題：

```c
// Core 0
ready_flag = 1;

// Core 1
while (ready_flag == 0) {}  // 無限迴圈！
```

Core 0 明明設定了 `ready_flag = 1`，但 Core 1 卻永遠看不到。

這不是程式邏輯錯誤，而是 **Cache Coherency** 問題。

---

## 一、多核心的 Cache 挑戰

### 1.1 每個核心都有自己的 Cache

在多核心系統中，每個核心都有自己的 L1 Cache（有時 L2 也是獨立的）：

```
         ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
         │ Core 0  │     │ Core 1  │     │ Core 2  │     │ Core 3  │
         └────┬────┘     └────┬────┘     └────┬────┘     └────┬────┘
              │               │               │               │
         ┌────┴────┐     ┌────┴────┐     ┌────┴────┐     ┌────┴────┐
         │L1 Cache │     │L1 Cache │     │L1 Cache │     │L1 Cache │
         │ X = 0   │     │ X = ?   │     │ X = ?   │     │ X = ?   │
         └────┬────┘     └────┬────┘     └────┬────┘     └────┬────┘
              │               │               │               │
              └───────────────┴───────┬───────┴───────────────┘
                                      │
                              ┌───────┴───────┐
                              │  Shared L3    │
                              │    Cache      │
                              └───────┬───────┘
                                      │
                              ┌───────┴───────┐
                              │   Memory      │
                              │   X = 0       │
                              └───────────────┘
```

### 1.2 問題：資料不一致

當多個核心快取同一個記憶體位置時，問題就出現了：

```c
// 初始狀態：X = 0 (在 Memory 中)

// Core 0 讀取 X
int a = X;  // Core 0 的 Cache: X = 0

// Core 1 讀取 X
int b = X;  // Core 1 的 Cache: X = 0

// Core 0 寫入 X
X = 100;    // Core 0 的 Cache: X = 100 (Memory 還是 0！)

// Core 1 讀取 X
int c = X;  // Core 1 的 Cache 還是 X = 0 ← 錯誤！
```

Core 1 看到的是「舊」的資料，因為 Core 0 的寫入還沒有傳播到 Core 1 的 Cache。

### 1.3 什麼是 Cache Coherency？

**定義**：

> Cache Coherency 是一種機制，確保多個核心的 Cache 中，同一個記憶體位置的資料保持一致。

**核心原則**：

1. **Write Propagation**：一個核心的寫入必須傳播到其他核心
2. **Transaction Serialization**：所有核心看到的寫入順序必須一致

---

## 二、硬體如何解決：MESI 協議

### 2.1 MESI 的四種狀態

MESI 是最常見的 Cache Coherency 協議，每個 Cache Line 都有一個狀態標記：

| 狀態 | 名稱 | 意義 |
|------|------|------|
| **M** | Modified | 只有這個核心有，且已被修改（Dirty） |
| **E** | Exclusive | 只有這個核心有，且與 Memory 一致（Clean） |
| **S** | Shared | 多個核心都有，且與 Memory 一致 |
| **I** | Invalid | 這個副本無效，不能使用 |

### 2.2 狀態轉換範例

**場景：Core 0 讀取 → Core 1 讀取 → Core 0 寫入**

```
Step 1: Core 0 讀取 X
┌────────────────────────────────────────────────────────────┐
│ Core 0 Cache: X = 0 [E]  ← Exclusive (只有我有)            │
│ Core 1 Cache: (空)                                         │
│ Memory: X = 0                                              │
└────────────────────────────────────────────────────────────┘

Step 2: Core 1 讀取 X
┌────────────────────────────────────────────────────────────┐
│ Core 0 Cache: X = 0 [S]  ← 變成 Shared                     │
│ Core 1 Cache: X = 0 [S]  ← Shared                          │
│ Memory: X = 0                                              │
└────────────────────────────────────────────────────────────┘

Step 3: Core 0 寫入 X = 100
┌────────────────────────────────────────────────────────────┐
│ Core 0 發出 "Invalidate X" 訊號                            │
│ Core 1 收到訊號，標記 X 為 Invalid                         │
├────────────────────────────────────────────────────────────┤
│ Core 0 Cache: X = 100 [M]  ← Modified (只有我有最新版)     │
│ Core 1 Cache: X = 0 [I]    ← Invalid (不能用)              │
│ Memory: X = 0 (還沒更新)                                   │
└────────────────────────────────────────────────────────────┘

---

## 三、False Sharing：隱藏的效能殺手

### 3.1 什麼是 False Sharing？

當兩個核心修改「不同」的變數，但這些變數位於「同一個 Cache Line」時，就會發生 False Sharing：

```c
struct {
    int counter_core0;  // Core 0 使用
    int counter_core1;  // Core 1 使用
} shared_data;

// Core 0
shared_data.counter_core0++;  // 修改 counter_core0

// Core 1
shared_data.counter_core1++;  // 修改 counter_core1
```

問題：`counter_core0` 和 `counter_core1` 在同一個 Cache Line！

```
┌────────────────────────────────────────────────────────────────┐
│ Cache Line (64 bytes)                                          │
├────────────────────────────┬───────────────────────────────────┤
│ counter_core0 (4 bytes)    │ counter_core1 (4 bytes)           │
│ Core 0 修改                │ Core 1 修改                       │
└────────────────────────────┴───────────────────────────────────┘
```

每次 Core 0 修改 `counter_core0`，都會 Invalidate Core 1 的 Cache Line。
每次 Core 1 修改 `counter_core1`，都會 Invalidate Core 0 的 Cache Line。

**結果**：兩個核心不斷互相 Invalidate，效能大幅下降。

### 3.2 實測：False Sharing 的影響

```c
// 有 False Sharing
struct {
    long counter0;
    long counter1;
    long counter2;
    long counter3;
} stats;  // 全部在同一個 Cache Line

// 4 個 Thread 各自更新自己的 Counter
// 結果：45 秒
```

```c
// 無 False Sharing
struct {
    long counter0;
    char padding0[56];  // Padding 到 64 bytes
    long counter1;
    char padding1[56];
    long counter2;
    char padding2[56];
    long counter3;
    char padding3[56];
} stats;  // 每個 Counter 在不同的 Cache Line

// 相同的測試
// 結果：2.5 秒 (快了 18 倍！)
```

### 3.3 解決方案：Cache Line 對齊

在 danieRTOS 中，我們已經在 `cpu_t` 結構使用了這個技巧：

```c
typedef struct {
    uint64_t hartid;
    tcb_t *current_task;
    tcb_t *idle_task;
    int32_t irq_nest_depth;
    uint64_t scratch[8];
    uint8_t padding[CACHE_LINE_SIZE - 24];
} __attribute__((aligned(CACHE_LINE_SIZE))) cpu_t;
```

每個核心的 `cpu_t` 都在獨立的 Cache Line，不會互相干擾。

---

## 四、Memory Ordering：另一個隱藏問題

### 4.1 為什麼 Cache Coherency 還不夠？

即使有 MESI 協議保證 Cache 一致性，記憶體操作的「順序」仍可能被重排。

考慮這個經典問題：

```c
// Thread 1 (Core 0)
data = 42;           // (A)
ready_flag = 1;      // (B)

// Thread 2 (Core 1)
while (ready_flag == 0) {}  // (C)
use(data);                   // (D)
```

直覺上，我們期望：

1. Core 0 先寫 `data`，再寫 `ready_flag`
2. Core 1 看到 `ready_flag == 1` 後，`data` 一定是 42

但實際上，CPU 和編譯器可能重排這些操作！

### 4.2 重排的來源

**1. 編譯器優化**：

```c
// 編譯器可能把 (B) 排在 (A) 之前
// 因為它們沒有資料依賴
ready_flag = 1;  // 先設 flag
data = 42;       // 後寫 data
```

**2. CPU Out-of-Order Execution**：

```c
// CPU 可能先執行簡單的操作
// data = 42 可能還在 Store Buffer 中
// ready_flag = 1 已經寫入 Cache
```

**3. Store Buffer**：

```c
// Core 0 的 Store Buffer：
// data = 42       ← 還沒寫入 Cache
// ready_flag = 1  ← 還沒寫入 Cache

// Core 1 可能先看到 ready_flag = 1，但 data 還是舊值
```

### 4.3 RISC-V 的 Weak Memory Model

RISC-V 使用 **RVWMO (RISC-V Weak Memory Ordering)**，允許大量重排：

```
Load → Load   : 可能重排
Load → Store  : 可能重排
Store → Load  : 可能重排
Store → Store : 可能重排
```

這意味著幾乎所有記憶體操作都可能被重排！

### 4.4 解決方案：FENCE 指令

`fence` 指令強制記憶體操作按順序執行：

```asm
fence pred, succ
```

- `pred`：fence 之前的操作類型
- `succ`：fence 之後的操作類型
- 類型：`r` (read), `w` (write), `rw` (both)

**常用 Fence**：

| Fence | 效果 | 用途 |
|-------|------|------|
| `fence rw, rw` | Full barrier | 最強，禁止所有重排 |
| `fence w, w` | Store-store | 確保 store 順序 |
| `fence r, r` | Load-load | 確保 load 順序 |
| `fence r, rw` | Acquire | 獲取鎖之後 |
| `fence rw, w` | Release | 釋放鎖之前 |

### 4.5 修正前言的 Bug

回到最開始的問題：

```c
// 修正後
// Core 0
data = 42;
asm volatile("fence w, w");  // 確保 data 先寫入
ready_flag = 1;
asm volatile("fence w, r");  // 確保 ready_flag 對其他核心可見

// Core 1
while (ready_flag == 0) {
    asm volatile("fence r, r");  // 每次迴圈都重新讀取
}
asm volatile("fence r, rw");  // 確保看到最新的 data
use(data);
```

更簡潔的寫法，使用 `volatile` 和原子操作：

```c
volatile int ready_flag = 0;
int data = 0;

// Core 0
data = 42;
__atomic_store_n(&ready_flag, 1, __ATOMIC_RELEASE);

// Core 1
while (__atomic_load_n(&ready_flag, __ATOMIC_ACQUIRE) == 0) {}
use(data);
```

---

## 五、在 danieRTOS 中的應用

### 5.1 Boot Flag

在 Ch 21 的多核心啟動中：

```asm
# Primary Core
fence w, w              # 確保初始化完成
sw t0, 0(t1)            # smp_boot_flag = 1
fence w, r              # 確保 flag 對其他核心可見

# Secondary Core
wait_loop:
    lw t2, 0(t1)        # 讀取 flag
    bnez t2, setup
    j wait_loop
setup:
    fence r, rw         # 確保看到所有初始化資料
```

### 5.2 Spinlock

在 Ch 23 的 Spinlock 中：

```asm
spinlock_acquire:
    li t0, 1
1:
    amoswap.w.aq t1, t0, (a0)  # .aq = Acquire 語意
    bnez t1, 1b
    ret

spinlock_release:
    amoswap.w.rl zero, zero, (a0)  # .rl = Release 語意
    ret
```

`.aq` 和 `.rl` 後綴自動包含必要的 fence。

### 5.3 Task 狀態更新

```c
void task_set_ready(tcb_t *task) {
    spinlock_acquire(&ready_queue_lock);

    task->state = TASK_STATE_READY;
    list_add_tail(&task->node, &ready_queue);

    spinlock_release(&ready_queue_lock);
    // Release 語意確保 task->state 的寫入對其他核心可見
}
```

---

## 六、效能考量

### 6.1 Fence 的成本

| 操作 | 預估 Cycles |
|------|-------------|
| Normal Load | 1-4 |
| Normal Store | 1-4 |
| `fence rw, rw` | 10-50 |
| `amoswap` | 10-30 |
| Cache Miss | 100-300 |
| Cross-Core Invalidate | 50-200 |

Fence 有成本，但比 Cache Miss 便宜。不要過度使用，但也不要遺漏必要的同步。

### 6.2 設計原則

1. **最小化共享資料**：減少需要同步的資料
2. **使用 Per-Core 資料**：每個核心有自己的副本
3. **Cache Line 對齊**：避免 False Sharing
4. **選擇適當的 Fence**：不要總是用 `fence rw, rw`

---

## 七、本章回顧

在這一章，我們探討了 Cache Coherency：

1. **MESI 協議**：硬體自動維護 Cache 一致性
2. **False Sharing**：隱藏的效能殺手，使用 Cache Line 對齊解決
3. **Memory Ordering**：即使 Cache 一致，操作順序仍可能被重排
4. **FENCE 指令**：強制記憶體操作順序

關鍵概念：

```
Cache Coherency ≠ Memory Ordering

MESI 保證：如果 Core 1 讀取 X，它會拿到最新值
MESI 不保證：Core 1 何時「看到」Core 0 的寫入
```

下一章，我們將實作 **IPI (Inter-Processor Interrupt)**——讓一個核心通知另一個核心。

---

## 參考資料

- **Data Structures in Practice**, Ch 2 - CPU Architecture
- **Tech Column**, Cache Architecture Series
- [RISC-V Memory Model](https://riscv.org/specifications/isa-spec-pdf/)

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。
