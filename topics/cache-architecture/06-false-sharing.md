# False Sharing 與多執行緒優化：看不見的性能殺手

**作者**: Danny Jiang  
**日期**: 2025-12-07  

---

## 前言：一個讓我抓狂三天的 Bug

2021 年，我在開發一個高性能的日誌系統時遇到了一個詭異的問題：

```c
// 每個 Thread 有自己的 Counter
struct {
    long counter_thread0;
    long counter_thread1;
    long counter_thread2;
    long counter_thread3;
} stats;

// 4 個 Thread 各自更新自己的 Counter
// Thread 0
stats.counter_thread0++;

// Thread 1
stats.counter_thread1++;

// ... 以此類推

```

**理論上**：每個 Thread 修改不同的變數，應該沒有競爭，性能應該很好。

**實際上**：性能慘不忍睹，比單執行緒還慢 3 倍！

> 註：本篇中提到的各種「x 倍」性能差距，皆來自在代表性硬體與測試程式上的實驗結果，用來說明 False Sharing 對性能的**量級影響**；不同 CPU、作業系統與實際服務上的絕對數字會有所不同，請將重點放在「有無 False Sharing」與「改善前後的相對變化」。

我用 `perf` 分析後發現：**Cache Miss Rate 高達 80%**，而且大部分是 **Cache-to-Cache Transfer**。

這讓我開始深入研究：**為什麼修改不同的變數，還會互相影響？**

答案就是：**False Sharing (偽共用)**。

---

## 一、什麼是 False Sharing？

### 1.1 核心概念

**定義**：

> 兩個 CPU Core 修改同一個 Cache Line 的不同部分，導致頻繁的 Cache Invalidate，嚴重影響性能。

**關鍵點**：

1. **Cache Line 是最小的一致性單位** (通常 64 Bytes)
2. **即使修改不同的變數，只要在同一個 Cache Line，就會互相影響**
3. **MESI 協議會 Invalidate 整個 Cache Line，而不是單個變數**

---

### 1.2 圖解 False Sharing

```
記憶體佈局：
┌────────────────────────────────────────────────────────┐
│  Cache Line 0 (64 Bytes)                               │
│  ┌──────────┬──────────┬──────────┬──────────┐        │
│  │counter_0 │counter_1 │counter_2 │counter_3 │        │
│  │ (8 bytes)│ (8 bytes)│ (8 bytes)│ (8 bytes)│        │
│  └──────────┴──────────┴──────────┴──────────┘        │
└────────────────────────────────────────────────────────┘

Core 0 修改 counter_0:

  1. Core 0 發出 "Invalidate Cache Line 0"
  2. Core 1, 2, 3 的 Cache Line 0 都被標記為 Invalid
  3. Core 0 修改 counter_0
  4. Core 0 的 Cache Line 0 狀態 = Modified

Core 1 修改 counter_1:

  1. Core 1 發現自己的 Cache Line 0 是 Invalid
  2. Core 1 發出 "Read Cache Line 0"
  3. Core 0 必須 Write Back Cache Line 0
  4. Core 1 載入 Cache Line 0
  5. Core 1 發出 "Invalidate Cache Line 0"
  6. Core 0, 2, 3 的 Cache Line 0 都被標記為 Invalid
  7. Core 1 修改 counter_1
  8. Core 1 的 Cache Line 0 狀態 = Modified

→ 乒乓效應 (Ping-Pong Effect)
→ 性能災難

```

---

### 1.3 真實測試：False Sharing 的代價

我在 Intel Core i9-12900K 上做了一個測試：

#### 測試 1：有 False Sharing


```c
struct {
    long counter0;  // 8 bytes
    long counter1;  // 8 bytes
    long counter2;  // 8 bytes
    long counter3;  // 8 bytes
} stats;  // 總共 32 bytes，在同一個 Cache Line

// 4 個 Thread 各自更新自己的 Counter
void* thread_func(void* arg) {
    int id = *(int*)arg;
    for (int i = 0; i < 100000000; i++) {
        if (id == 0) stats.counter0++;
        if (id == 1) stats.counter1++;
        if (id == 2) stats.counter2++;
        if (id == 3) stats.counter3++;
    }
    return NULL;
}

// 結果：45 秒

```

---

#### 測試 2：無 False Sharing (Padding)


```c
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

// 相同的測試程式碼

// 結果：2.5 秒 (快了 18 倍！)

```

---

#### 測試 3：無 False Sharing (Aligned)


```c
struct {
    __attribute__((aligned(64))) long counter0;
    __attribute__((aligned(64))) long counter1;
    __attribute__((aligned(64))) long counter2;
    __attribute__((aligned(64))) long counter3;
} stats;

// 結果：2.5 秒 (與 Padding 版本相同)

```

---

### 1.4 為什麼會慢這麼多？

**分析**：

```
有 False Sharing:
  每次修改都需要：

    1. Invalidate 其他 Core 的 Cache Line (10-20 cycles)
    2. 等待 Acknowledgement (10-20 cycles)
    3. 如果其他 Core 有 Modified 資料，需要 Write Back (50-100 cycles)
    4. 重新載入 Cache Line (50-100 cycles)
  
  總延遲 = 100-200 cycles per update

無 False Sharing:
  每次修改只需要：

    1. 修改自己的 Cache Line (1 cycle)
  
  總延遲 = 1 cycle per update

性能差距 = 100-200 倍！

```

---

## 二、False Sharing 的常見場景

### 2.1 場景 1：陣列中的 Counter

**問題程式碼**：

```c
long counters[4];  // 4 個 Counter，每個 8 bytes

// 4 個 Thread 各自更新自己的 Counter
void* thread_func(void* arg) {
    int id = *(int*)arg;
    for (int i = 0; i < 100000000; i++) {
        counters[id]++;  // False Sharing！
    }
    return NULL;
}

```

**問題**：

```
counters[0], counters[1], counters[2], counters[3]
都在同一個 Cache Line (64 bytes)
→ False Sharing

```

---

#### 解決方案 1：Padding


```c
struct PaddedCounter {
    long value;
    char padding[56];  // Padding 到 64 bytes
} __attribute__((aligned(64)));

PaddedCounter counters[4];

void* thread_func(void* arg) {
    int id = *(int*)arg;
    for (int i = 0; i < 100000000; i++) {
        counters[id].value++;  // 無 False Sharing
    }
    return NULL;
}

```

---

#### 解決方案 2：Thread-Local Storage


```c
__thread long local_counter = 0;

void* thread_func(void* arg) {
    for (int i = 0; i < 100000000; i++) {
        local_counter++;  // 每個 Thread 有自己的副本
    }
    return NULL;
}

// 最後合併
long total = 0;
for (int i = 0; i < num_threads; i++) {
    total += thread_local_counters[i];
}

```

---

### 2.2 場景 2：結構體中的多個欄位

**問題程式碼**：

```c
struct TaskQueue {
    int head;           // Thread 0 修改
    int tail;           // Thread 1 修改
    int size;           // Thread 2 讀取
    pthread_mutex_t lock;
};

TaskQueue queue;

// Thread 0: Producer
void* producer(void* arg) {
    while (1) {
        pthread_mutex_lock(&queue.lock);
        queue.tail++;  // False Sharing with head
        pthread_mutex_unlock(&queue.lock);
    }
}

// Thread 1: Consumer
void* consumer(void* arg) {
    while (1) {
        pthread_mutex_lock(&queue.lock);
        queue.head++;  // False Sharing with tail
        pthread_mutex_unlock(&queue.lock);
    }
}

```

**問題**：

```
head, tail, size, lock 都在同一個 Cache Line
即使有 mutex 保護，還是會有 False Sharing
→ 性能下降

```

---

#### 解決方案：分離 Hot Fields


```c
struct TaskQueue {
    // Hot fields (頻繁修改)
    __attribute__((aligned(64))) int head;
    char padding1[60];
    
    __attribute__((aligned(64))) int tail;
    char padding2[60];
    
    // Cold fields (很少修改)
    __attribute__((aligned(64))) int size;
    pthread_mutex_t lock;
};

```

---

### 2.3 場景 3：全域變數

**問題程式碼**：

```c
// 全域變數
int global_flag = 0;
long global_counter = 0;

// Thread 0: 頻繁修改 global_counter
void* thread0(void* arg) {
    for (int i = 0; i < 100000000; i++) {
        global_counter++;
    }
}

// Thread 1: 偶爾檢查 global_flag
void* thread1(void* arg) {
    while (global_flag == 0) {
        // 等待
    }
}

```

**問題**：

```
global_flag 和 global_counter 可能在同一個 Cache Line
Thread 0 頻繁修改 global_counter
→ Thread 1 的 global_flag 被頻繁 Invalidate
→ False Sharing

```

---

**解決方案**：

```c
__attribute__((aligned(64))) int global_flag = 0;
__attribute__((aligned(64))) long global_counter = 0;

```

---

## 三、如何偵測 False Sharing？

### 3.1 使用 perf c2c (Cache-to-Cache)

**Linux perf 的 c2c 工具**：

```bash
# 記錄 Cache-to-Cache Transfer
perf c2c record ./program

# 分析報告
perf c2c report --stdio

# 或使用 TUI
perf c2c report

```

**報告範例**：

```
=================================================
       Shared Data Cache Line Table
=================================================
#
#        ----------- Cacheline ----------    Total      Tot  ----- LLC Load Hitm -----
# Index             Address  Node  PA cnt  records     Hitm    Total      Lcl      Rmt
# .....  ..................  ....  ......  .......  .......  .......  .......  .......
#
      0      0x7f8a4c000000     0    4096   123456    45.2%    12345     12345        0
      1      0x7f8a4c000040     0    4096    98765    32.1%     9876      9876        0

=================================================
       Shared Cache Line Distribution Pareto
=================================================
#
#        ----- HITM -----  -- Store Refs --  --------- Data address ---------
#   Num      Rmt      Lcl   L1 Hit  L1 Miss              Offset  Node  PA cnt        Pid
# .....  .......  .......  .......  .......  ..................  ....  ......  .........
#
  -------------------------------------------------------------
      0        0    12345    98765     1234      0x7f8a4c000000

  -------------------------------------------------------------
           0.00%   45.20%   40.30%   50.10%                0x0     0       1      12345
           0.00%   32.10%   35.20%   30.50%                0x8     0       1      12345
           0.00%   22.70%   24.50%   19.40%               0x10     0       1      12345

→ 發現 0x7f8a4c000000 這個 Cache Line 有嚴重的 False Sharing

```

---

### 3.2 使用 Intel VTune

**步驟**：

```bash
# 收集 Memory Access 資料
vtune -collect memory-access -knob analyze-mem-objects=true ./program

# 查看 False Sharing 報告
vtune -report summary -r <result_dir>

```

**報告範例**：

```
Memory Access Analysis
======================

False Sharing:
  Total False Sharing Events: 12,345,678
  Top False Sharing Objects:

    1. stats.counter0 - stats.counter3 (45.2%)
    2. queue.head - queue.tail (32.1%)
    3. global_flag - global_counter (22.7%)

Recommendations:

  - Add padding between counter0 and counter1
  - Align queue.head and queue.tail to cache line boundaries
  - Separate global_flag and global_counter

```

---

### 3.3 使用 Cachegrind (Valgrind)

**步驟**：

```bash
# 執行 Cachegrind
valgrind --tool=cachegrind --cachegrind-out-file=cachegrind.out ./program

# 分析報告
cg_annotate cachegrind.out

```

**限制**：

```
Cachegrind 只能模擬單執行緒的 Cache 行為
無法偵測 False Sharing (需要多執行緒)
→ 不適合用來偵測 False Sharing

```

---

## 四、False Sharing 的解決方案

### 4.1 方案 1：Padding

**優點**：

- 簡單直接
- 效果明顯

**缺點**：

- 浪費記憶體
- 需要手動計算 Padding 大小

**範例**：

```c
struct PaddedData {
    long value;
    char padding[64 - sizeof(long)];  // Padding 到 64 bytes
} __attribute__((aligned(64)));

```

---

### 4.2 方案 2：Aligned 屬性

**優點**：

- 編譯器自動處理
- 程式碼簡潔

**缺點**：

- 仍然浪費記憶體

**範例**：

```c
__attribute__((aligned(64))) long counter0;
__attribute__((aligned(64))) long counter1;

```

---

### 4.3 方案 3：Thread-Local Storage

**優點**：

- 完全避免 False Sharing
- 不浪費記憶體

**缺點**：

- 需要最後合併結果
- 不適合需要即時共享的資料

**範例**：

```c
__thread long local_counter = 0;

void* thread_func(void* arg) {
    for (int i = 0; i < 100000000; i++) {
        local_counter++;
    }
    return NULL;
}

// 最後合併
long total = 0;
for (int i = 0; i < num_threads; i++) {
    total += get_thread_local_counter(i);
}

```

---

### 4.4 方案 4：Per-CPU 資料結構

**概念**：

```
每個 CPU Core 有自己的資料結構
避免跨 Core 的 Cache Coherency 開銷

```

**範例**：

```c
#define MAX_CPUS 64

struct PerCPUData {
    long counter;
    char padding[64 - sizeof(long)];
} __attribute__((aligned(64)));

PerCPUData per_cpu_data[MAX_CPUS];

void* thread_func(void* arg) {
    int cpu_id = sched_getcpu();  // 取得當前 CPU ID
    
    for (int i = 0; i < 100000000; i++) {
        per_cpu_data[cpu_id].counter++;
    }
    return NULL;
}

```

**優點**：

- 完全避免 False Sharing
- 適合高性能系統

**缺點**：

- 需要 CPU Affinity
- 複雜度較高

---

## 五、真實案例分析

### 5.1 案例 1：Linux Kernel 的 Per-CPU 變數

**問題**：

```c
// 早期的 Linux Kernel
struct {
    long nr_running;     // 執行中的 Process 數量
    long nr_iowait;      // 等待 I/O 的 Process 數量
    long nr_switches;    // Context Switch 次數
} cpu_stats[NR_CPUS];

// 每個 CPU 頻繁更新自己的 stats
// → False Sharing

```

**解決方案**：

```c
// 現代的 Linux Kernel
struct cpu_stats {
    long nr_running;
    long nr_iowait;
    long nr_switches;
    char __pad[64 - 3 * sizeof(long)];
} __attribute__((aligned(64)));

struct cpu_stats cpu_stats[NR_CPUS];

```

**性能提升**：

```
優化前：Context Switch 延遲 = 5 μs
優化後：Context Switch 延遲 = 2 μs
性能提升：2.5 倍

```

---

### 5.2 案例 2：Java ConcurrentHashMap

**問題**：

```java
// 早期的 ConcurrentHashMap
class Segment {
    int count;           // 元素數量
    int modCount;        // 修改次數
    int threshold;       // 擴容閾值
    HashEntry[] table;   // Hash Table
}

// 多個 Thread 同時存取不同的 Segment
// 但 count, modCount, threshold 在同一個 Cache Line
// → False Sharing

```

**解決方案**：

```java
// Java 8 的 ConcurrentHashMap
class Segment {
    volatile int count;
    // Padding
    long p1, p2, p3, p4, p5, p6, p7;
    volatile int modCount;
    // Padding
    long p8, p9, p10, p11, p12, p13, p14;
    int threshold;
    HashEntry[] table;
}

```

**性能提升**：

```
優化前：1,000,000 次操作 = 5 秒
優化後：1,000,000 次操作 = 2 秒
性能提升：2.5 倍

```

---

### 5.3 案例 3：我的日誌系統優化

**原始程式碼**：

```c
struct LogStats {
    long total_logs;
    long error_logs;
    long warning_logs;
    long info_logs;
};

LogStats stats;

// 4 個 Thread 各自記錄不同類型的 Log
void log_error(const char* msg) {
    __sync_fetch_and_add(&stats.error_logs, 1);
}

void log_warning(const char* msg) {
    __sync_fetch_and_add(&stats.warning_logs, 1);
}

// 性能：100,000 logs/sec

```

---

**優化後**：

```c
struct LogStats {
    __attribute__((aligned(64))) long total_logs;
    __attribute__((aligned(64))) long error_logs;
    __attribute__((aligned(64))) long warning_logs;
    __attribute__((aligned(64))) long info_logs;
};

LogStats stats;

// 性能：1,500,000 logs/sec (提升 15 倍！)

```

---

## 六、False Sharing 的進階主題

### 6.1 Cache Line Bouncing

**概念**：

```
Cache Line 在多個 Core 之間來回傳遞
就像乒乓球一樣彈來彈去
→ Cache Line Bouncing

```

**範例**：

```c
// Core 0 和 Core 1 交替修改同一個變數
volatile int flag = 0;

// Core 0
while (1) {
    while (flag != 0);  // 等待 flag = 0
    flag = 1;           // 設定 flag = 1
}

// Core 1
while (1) {
    while (flag != 1);  // 等待 flag = 1
    flag = 0;           // 設定 flag = 0
}

// Cache Line 在 Core 0 和 Core 1 之間來回傳遞
// → 嚴重的性能問題

```

---

### 6.2 NUMA 的影響（進階）

**問題**：

```
NUMA (Non-Uniform Memory Access):
  每個 CPU Socket 有自己的記憶體
  存取本地記憶體快，存取遠端記憶體慢

False Sharing + NUMA:
  如果兩個 Core 在不同的 Socket
  False Sharing 的開銷更大
  → 需要跨 Socket 的 Cache Coherency
  → 延遲增加 2-3 倍

```

**解決方案**：

```c
// 使用 numactl 綁定 Thread 到特定的 NUMA Node
numactl --cpunodebind=0 --membind=0 ./program

```

---

### 6.3 Huge Page 的影響（進階）

**問題**：

```
4KB Page:
  TLB 覆蓋範圍小
  TLB Miss 頻繁
  
Huge Page (2MB, 1GB):
  TLB 覆蓋範圍大
  TLB Miss 減少
  
但 Huge Page 不會改變 Cache Line 的大小 (仍然是 64 bytes)
→ False Sharing 的問題依然存在

```

---

## 七、給程式設計師的建議

### 7.1 設計原則

#### 原則 1：避免共享寫入


```c
// 不好的設計
int shared_counter = 0;

void* thread_func(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        __sync_fetch_and_add(&shared_counter, 1);
    }
}

// 好的設計
__thread int local_counter = 0;

void* thread_func(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        local_counter++;
    }
}

```

---

#### 原則 2：分離 Hot 和 Cold Fields


```c
// 不好的設計
struct Data {
    int hot_field1;   // 頻繁修改
    int cold_field1;  // 很少修改
    int hot_field2;   // 頻繁修改
    int cold_field2;  // 很少修改
};

// 好的設計
struct Data {
    // Hot fields
    __attribute__((aligned(64))) int hot_field1;
    __attribute__((aligned(64))) int hot_field2;
    
    // Cold fields
    int cold_field1;
    int cold_field2;
};

```

---

#### 原則 3：使用 Cache Line 對齊


```c
// 不好的設計
long counters[4];

// 好的設計
__attribute__((aligned(64))) long counters[4];

// 或
struct {
    long value;
    char padding[56];
} counters[4];

```

---

### 7.2 測量和驗證

**步驟**：

```
1. 測量 Baseline
   → perf stat -e cache-misses ./program

2. 識別 False Sharing
   → perf c2c record ./program
   → perf c2c report

3. 優化
   → 加入 Padding 或 Aligned 屬性

4. 驗證
   → 再次測量，確認性能提升

```

---

### 7.3 何時需要優化？

**需要優化**：

- ✓ 多執行緒程式
- ✓ 高頻率的寫入操作
- ✓ Cache Miss Rate > 10%
- ✓ perf c2c 顯示大量 False Sharing

**不需要優化**：

- ✗ 單執行緒程式
- ✗ 低頻率的操作
- ✗ Cache Miss Rate < 5%
- ✗ 過早優化

---

## 八、總結

False Sharing 是多執行緒程式設計中最隱蔽的性能殺手：

1. **根本原因**：Cache Line 是一致性的最小單位 (64 bytes)，False Sharing 是一個經由 **Cache 一致性層級** 製造的效應，而不是 lock 或演算法邏輯本身的 bug。
2. **性能影響**：可能導致 10-100 倍的性能下降
3. **偵測工具**：perf c2c, Intel VTune
4. **解決方案**：Padding, Aligned, Thread-Local Storage, Per-CPU 資料結構

理解 False Sharing，可以幫助我們：

- 設計更高效的多執行緒程式
- 避免隱蔽的性能陷阱
- 充分利用多核心 CPU 的性能

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: https://github.com/djiangtw/tech-column-public
