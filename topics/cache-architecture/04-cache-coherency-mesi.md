# Cache Coherency 與 MESI 協議：多核心時代的一致性挑戰

**作者**: Danny Jiang  
**日期**: 2025-12-07  

---

## 前言：一個詭異的 Bug

2018 年，我在開發一個多執行緒的資料處理系統時遇到了一個詭異的 Bug：

```c
// Thread 1
shared_data.counter = 100;
printf("Thread 1: %d\n", shared_data.counter);  // 輸出 100

// Thread 2 (在另一個 CPU Core)
printf("Thread 2: %d\n", shared_data.counter);  // 輸出 0 ？！

```

> 註：本文中的 C 範例刻意簡化（未展開完整的原子操作與記憶體模型細節），目的是聚焦在 **Cache Coherency / MESI 行為** 的直覺理解；實際生產程式需搭配正確的同步原語與記憶體順序保證。

兩個 Thread 讀取同一個變數，卻看到不同的值！更詭異的是，如果我加上 `sleep(1)`，問題就消失了。

這讓我開始深入研究：**多核心 CPU 如何保證 Cache 的一致性？為什麼有時候會看到「舊」的資料？**

經過幾天的研究和實驗，我終於理解了 **Cache Coherency** 和 **MESI 協議** 的運作原理。今天，我想和你分享這個故事。

---

## 一、問題的根源：每個 Core 都有自己的 Cache

### 1.1 單核心時代：沒有一致性問題

```
單核心 CPU:
┌──────────────┐
│   CPU Core   │
│  ┌─────────┐ │
│  │ L1 Cache│ │
│  └─────────┘ │
└──────────────┘
      ↓
┌─────────────┐
│Main Memory  │
└─────────────┘

只有一個 Cache，不會有不一致的問題

```

---

### 1.2 多核心時代：災難的開始

```
多核心 CPU:
┌─────────────┐  ┌─────────────┐
│  Core 0     │  │  Core 1     │
│ ┌────────┐  │  │ ┌────────┐  │
│ │L1 Cache│  │  │ │L1 Cache│  │
│ └────────┘  │  │ └────────┘  │
└─────────────┘  └─────────────┘
      ↓                ↓
┌──────────────────────────────┐
│      Main Memory             │
└──────────────────────────────┘

問題：
  Core 0 和 Core 1 都有變數 X 的副本
  Core 0 修改了 X
  Core 1 如何知道它的副本已經過期？

```

---

### 1.3 真實案例：資料不一致的災難

**場景**：

```c
// 初始狀態：Memory 中 X = 100

// Core 0 讀取 X
int a = X;  // Core 0 的 L1 Cache: X = 100

// Core 1 讀取 X
int b = X;  // Core 1 的 L1 Cache: X = 100

// Core 0 修改 X
X = 200;    // Core 0 的 L1 Cache: X = 200
            // Core 1 的 L1 Cache: X = 100 (舊的！)
            // Memory: X = 100 (還沒寫回)

// Core 1 再次讀取 X
int c = X;  // Core 1 讀到 100 (錯誤！)

```

**結果**：Core 1 看到的是「舊」的資料，程式邏輯錯誤！

---

## 二、解決方案：Cache Coherency Protocol

### 2.1 什麼是 Cache Coherency？

**定義**：

> Cache Coherency 是一種機制，確保多個 CPU Core 的 Cache 中，同一個記憶體位置的資料保持一致。

**核心原則**：

1. **Write Propagation (寫入傳播)**：一個 Core 的寫入必須傳播到其他 Core
2. **Transaction Serialization (交易序列化)**：所有 Core 看到的寫入順序必須一致

---

### 2.2 兩種主流方案

#### A. Snooping (監聽/嗅探)

```
所有 Core 的 Cache 都連接到共享匯流排
每個 Cache 都會「監聽」匯流排上的交易

當 Core A 發出 "Write X" 時：
  Core B, C, D 的 Cache 都會聽到
  如果它們有 X 的副本，就標記為 Invalid

```

**優點**：

- 實作簡單
- 延遲低 (直接廣播)

**缺點**：

- 頻寬開銷大 (所有交易都要廣播)
- 不適合大量 Core (> 16 Core)

**適用場景**：消費級 CPU (2-16 Core)

---

#### B. Directory-based (目錄式)

```
有一個中央目錄記錄每個 Cache Line 的狀態
當 Core A 要寫入 X 時：
  查詢目錄，找出哪些 Core 有 X 的副本
  只通知這些 Core，不需要廣播

```

**優點**：

- 頻寬開銷小 (點對點通訊)
- 適合大量 Core (> 16 Core)

**缺點**：

- 實作複雜
- 延遲較高 (需要查詢目錄)

**適用場景**：伺服器 CPU (32+ Core)

---

## 三、MESI 協議：Snooping 的經典實作

### 3.1 MESI 的四種狀態

**M (Modified)**：

```
這個 Core 修改過資料
其他 Core 的副本都是 Invalid
資料與記憶體不一致 (Dirty)
最終要 Write Back 到記憶體

```

**E (Exclusive)**：

```
只有這個 Core 有副本
資料與記憶體一致 (Clean)
可以直接修改 (變成 M 狀態)
不需要通知其他 Core

```

**S (Shared)**：

```
多個 Core 都有副本
所有副本與記憶體一致 (Clean)
要修改必須先 Invalidate 其他副本

```

**I (Invalid)**：

```
這個副本無效
不能使用
需要重新從記憶體或其他 Cache 載入

```

---

### 3.2 MESI 狀態轉換：一個完整的故事

讓我用一個真實的場景來說明 MESI 的運作：

#### 場景 1：Core A 讀取資料 (E 狀態)

```
初始狀態：
  Core A Cache: (空)
  Core B Cache: (空)
  Memory: X = 100

Core A 讀取 X:

  1. Core A 發出 "Read X" 到匯流排
  2. Core B 監聽到，檢查自己的 Cache → 沒有
  3. 從 Memory 讀取 X = 100
  4. Core A Cache: X = 100 [E]  (Exclusive，只有我有)

```

**為什麼是 E 而不是 S？**

因為沒有其他 Core 有副本，所以是 Exclusive。這很重要，因為：

- 如果要修改，可以直接變成 M，不需要通知其他 Core
- 節省了一次匯流排交易

---

#### 場景 2：Core B 也讀取資料 (S 狀態)

```
當前狀態：
  Core A Cache: X = 100 [E]
  Core B Cache: (空)

Core B 讀取 X:

  1. Core B 發出 "Read X" 到匯流排
  2. Core A 監聽到，發現自己有 X [E]
  3. Core A 回應 "我有 X"，並提供資料
  4. Core A Cache: X = 100 [S]  (Shared，變成共享)
  5. Core B Cache: X = 100 [S]  (Shared，我也有了)

```

**關鍵點**：

- Core A 的狀態從 E 變成 S
- 資料可以從 Core A 直接傳給 Core B (Cache-to-Cache Transfer)
- 不需要存取 Memory，速度更快

---

#### 場景 3：Core A 寫入資料 (M 狀態 - 關鍵！)

```
當前狀態：
  Core A Cache: X = 100 [S]
  Core B Cache: X = 100 [S]

Core A 寫入 X = 200:

  1. Core A 發出 "Invalidate X" 到匯流排
  2. Core B 監聽到，標記自己的 X 為 [I] (Invalid)
  3. Core A 修改 X = 200
  4. Core A Cache: X = 200 [M]  (Modified，只有我有最新版)
  5. Core B Cache: X = 100 [I]  (Invalid，不能用了)
  6. Memory: X = 100 (舊的，還沒更新)

```

**關鍵點**：

- Core A 必須先 Invalidate 其他 Core 的副本
- 這需要一次匯流排交易 (Invalidate Request)
- 如果有很多 Core 有副本，需要等所有 Core 都回應 (Invalidate Acknowledgement)
- 這就是為什麼 **False Sharing** 會嚴重影響性能！

---

#### 場景 4：Core B 試圖讀取被修改過的資料 (Write Back)

```
當前狀態：
  Core A Cache: X = 200 [M]
  Core B Cache: X = 100 [I]
  Memory: X = 100 (舊的)

Core B 讀取 X:

  1. Core B 發出 "Read X" 到匯流排
  2. Core A 監聽到，發現自己有 X [M] (Modified)
  3. Core A 必須先 Write Back:
     a. 把 X = 200 寫回 Memory
     b. 狀態變成 [S]

  4. Core B 從 Memory 讀取 X = 200
  5. Core A Cache: X = 200 [S]
  6. Core B Cache: X = 200 [S]
  7. Memory: X = 200 (已更新)

```

**關鍵點**：

- Core A 的 Modified 資料必須先 Write Back
- 這增加了 Core B 的讀取延遲
- 這就是為什麼「寫入後立即讀取」會比較慢

---

### 3.3 MESI 狀態轉換圖

```
                    ┌─────────────┐
                    │  Invalid    │
                    │     (I)     │
                    └──────┬──────┘
                           │
                    Read (from Memory)
                           │
                           ↓
        ┌──────────────────┴──────────────────┐
        │                                      │
   No other     ┌─────────────┐         Other cores
   cores have   │  Exclusive  │         have copy
   copy         │     (E)     │              │
        │       └──────┬──────┘              │
        │              │                     │
        │         Local Write                │
        │              │                     │
        │              ↓                     ↓
        │       ┌─────────────┐       ┌─────────────┐
        └──────→│  Modified   │       │   Shared    │
                │     (M)     │       │     (S)     │
                └──────┬──────┘       └──────┬──────┘
                       │                     │
                  Write Back            Local Write
                       │              (需要 Invalidate)
                       │                     │
                       ↓                     ↓
                ┌─────────────┐       ┌─────────────┐
                │   Shared    │       │  Modified   │
                │     (S)     │       │     (M)     │
                └─────────────┘       └─────────────┘
                       │                     │
                  Invalidate            Invalidate
                   from other           from other
                    cores                 cores
                       │                     │
                       ↓                     ↓
                ┌─────────────┐       ┌─────────────┐
                │  Invalid    │       │  Invalid    │
                │     (I)     │       │     (I)     │
                └─────────────┘       └─────────────┘

```

---

## 四、MESI 的性能影響

### 4.1 Invalidate 的開銷

**問題**：

```c
// Core 0
for (int i = 0; i < 1000000; i++) {
    shared_data.counter++;  // 每次都要 Invalidate Core 1 的副本
}

// Core 1
for (int i = 0; i < 1000000; i++) {
    shared_data.counter++;  // 每次都要 Invalidate Core 0 的副本
}

```

**性能災難**：

```
每次寫入都需要：

  1. 發出 Invalidate Request
  2. 等待其他 Core 的 Acknowledgement
  3. 才能修改資料

如果兩個 Core 頻繁修改同一個變數：
  → 乒乓效應 (Ping-Pong Effect)
  → 性能下降 10-100 倍！

```

---

### 4.2 真實測試：Invalidate 的代價

我在 Intel Core i9-12900K 上做了一個測試：

```c
// 測試 1：無競爭
__attribute__((aligned(64))) int counter0;  // 獨立 Cache Line
__attribute__((aligned(64))) int counter1;  // 獨立 Cache Line

// Thread 0
for (int i = 0; i < 100000000; i++) {
    counter0++;
}

// Thread 1
for (int i = 0; i < 100000000; i++) {
    counter1++;
}

// 結果：2.5 秒

```

```c
// 測試 2：有競爭
int shared_counter;  // 共享變數

// Thread 0
for (int i = 0; i < 100000000; i++) {
    shared_counter++;
}

// Thread 1
for (int i = 0; i < 100000000; i++) {
    shared_counter++;
}

// 結果：45 秒 (慢了 18 倍！)

```

> 註：以上測試數字為在代表性開發環境上的實驗結果，主要用來說明 **Invalidate / Cache-to-Cache 傳輸 對延遲的量級影響**，不同 CPU 與 workload 下的實際秒數與倍數會有所差異。

**原因**：

- 測試 1：兩個 Counter 在不同的 Cache Line，無 Invalidate 開銷
- 測試 2：共享同一個變數，頻繁 Invalidate，性能災難

---

### 4.3 Write Back 的開銷

**問題**：

```c
// Core 0 寫入大量資料
for (int i = 0; i < 1000; i++) {
    data[i] = compute(i);  // 資料在 Core 0 的 Cache 中，狀態 = M
}

// Core 1 讀取資料
for (int i = 0; i < 1000; i++) {
    process(data[i]);  // 每次讀取都要等 Core 0 Write Back
}

```

**性能影響**：

```
正常讀取延遲：4 cycles (L1 Hit)
需要 Write Back 的讀取延遲：

  1. Core 1 發出 Read Request
  2. Core 0 監聽到，發現是 Modified
  3. Core 0 Write Back (10-20 cycles)
  4. Core 1 從 Memory 讀取 (200+ cycles)
  
總延遲 = 200+ cycles (慢了 50 倍！)

```

---

## 五、MESI 的變種：MOESI 和 MESIF

### 5.1 MOESI (AMD 的選擇)

#### 新增狀態：O (Owned)


```
O (Owned):
  這個 Core 有最新的資料 (Dirty)
  其他 Core 也可以有副本 (Shared)
  但只有這個 Core 負責 Write Back

```

**優勢**：

```
MESI 的問題：
  Core A 有 Modified 資料
  Core B 要讀取
  → Core A 必須 Write Back 到 Memory
  → Core B 從 Memory 讀取
  → 浪費頻寬

MOESI 的解決方案：
  Core A 有 Modified 資料
  Core B 要讀取
  → Core A 變成 Owned，直接傳給 Core B
  → Core B 變成 Shared
  → 不需要 Write Back 到 Memory
  → 節省頻寬

```

**使用者**：AMD Ryzen, AMD EPYC

---

### 5.2 MESIF (Intel 的選擇)

#### 新增狀態：F (Forward)


```
F (Forward):
  多個 Core 都有副本 (Shared)
  但只有這個 Core 負責回應 Read Request

```

**優勢**：

```
MESI 的問題：
  Core A, B, C 都有 Shared 副本
  Core D 要讀取
  → 誰應該回應？
  → 需要仲裁機制

MESIF 的解決方案：
  Core A 是 Forward，Core B, C 是 Shared
  Core D 要讀取
  → 只有 Core A 回應
  → 不需要仲裁

```

**使用者**：Intel Core, Intel Xeon

---

## 六、給程式設計師的建議

### 6.1 避免 False Sharing

**問題**：

```c
struct {
    int counter_core0;  // Core 0 使用
    int counter_core1;  // Core 1 使用
} shared_data;

// Core 0
shared_data.counter_core0++;  // 修改 counter_core0

// Core 1
shared_data.counter_core1++;  // 修改 counter_core1

// 問題：兩個變數在同一個 Cache Line (64 Bytes)
// → 頻繁 Invalidate
// → 性能災難

```

**解決方案**：

```c
struct {
    int counter_core0;
    char padding0[60];  // Padding 到 64 Bytes
    int counter_core1;
    char padding1[60];  // Padding 到 64 Bytes
} shared_data;

// 或使用 aligned 屬性
struct {
    __attribute__((aligned(64))) int counter_core0;
    __attribute__((aligned(64))) int counter_core1;
} shared_data;

```

---

### 6.2 減少共享資料的寫入

**不好的設計**：

```c
// 所有 Thread 都寫入同一個 Counter
int global_counter = 0;

void worker() {
    for (int i = 0; i < 1000000; i++) {
        __sync_fetch_and_add(&global_counter, 1);  // 頻繁 Invalidate
    }
}

```

**好的設計**：

```c
// 每個 Thread 有自己的 Counter
__thread int local_counter = 0;

void worker() {
    for (int i = 0; i < 1000000; i++) {
        local_counter++;  // 無 Invalidate 開銷
    }
}

// 最後合併
int total = 0;
for (int i = 0; i < num_threads; i++) {
    total += thread_counters[i];
}

```

---

### 6.3 使用 Read-Mostly 資料結構

**概念**：

```
如果資料大部分時間是讀取，很少寫入：
  → 大部分時間處於 Shared 狀態
  → 無 Invalidate 開銷
  → 性能良好

```

**範例**：RCU (Read-Copy-Update)

```c
// 讀取：無鎖，無 Invalidate
void* read_data() {
    rcu_read_lock();
    void* data = rcu_dereference(global_ptr);
    // 使用 data
    rcu_read_unlock();
    return data;
}

// 寫入：Copy-on-Write
void update_data(void* new_data) {
    void* old_data = global_ptr;
    rcu_assign_pointer(global_ptr, new_data);  // 原子更新
    synchronize_rcu();  // 等待所有讀取者完成
    free(old_data);
}

```

---

## 七、測量和除錯

### 7.1 使用 perf 測量 Cache Coherency 開銷

```bash
# 測量 Invalidate 次數
perf stat -e cache-misses,cache-references,LLC-load-misses ./your_program

# 測量 False Sharing
perf c2c record ./your_program
perf c2c report

```

---

### 7.2 使用 Intel VTune 分析

```bash
# 收集 Cache Coherency 資料
vtune -collect memory-access -knob analyze-mem-objects=true ./your_program

# 查看報告
vtune -report summary

```

**關鍵指標**：

```
Remote Cache Hit Rate: 應該 < 5%
  (從其他 Core 的 Cache 讀取的比例)

False Sharing: 應該 < 1%
  (因為 False Sharing 導致的 Invalidate)

```

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: https://github.com/djiangtw/tech-column-public
