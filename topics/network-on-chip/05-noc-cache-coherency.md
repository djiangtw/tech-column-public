# NoC 與 Cache Coherency 整合：多核心的協調藝術

**作者**: Danny Jiang  
**日期**: 2025-12-07  

---

## 前言：Cache Coherency 與 NoC 的緊密關聯

在多核心 SoC 的設計中，Cache Coherency 是最關鍵的挑戰之一。

業界曾有這樣的案例：一個 16 核心的 SoC 在測試時遇到詭異的問題。

測試程式很簡單：**兩個核心同時讀寫同一個變數**

```c
// Core 0
x = 1;
printf("Core 0: x = %d\n", x);

// Core 1
x = 2;
printf("Core 1: x = %d\n", x);
```

預期結果：
- Core 0 看到 `x = 1` 或 `x = 2`
- Core 1 看到 `x = 2`

實際結果：
- Core 0 看到 `x = 1`（正確）
- Core 1 看到 `x = 1`（錯誤！）

**Core 1 寫入的值消失了！**

工程師花了數天時間才找到原因：

**Invalidate 訊息在 NoC 中被延遲，導致 Cache Coherency 協議失效。**

這個案例說明：

**NoC 不只是傳輸資料的管道，更是 Cache Coherency 協議的基礎設施。**

如果 NoC 的設計不正確，整個 Coherency 協議都會崩潰。

---

## 一、為什麼 NoC 需要支援 Cache Coherency？

### 1.1 多核心的挑戰

**問題**：

```
單核心：
  CPU → Cache → Memory
  → 簡單，沒有 Coherency 問題

多核心：
  CPU 0 → Cache 0 ↘
                    Memory
  CPU 1 → Cache 1 ↗
  
  → 複雜，需要 Coherency 協議
```

**Coherency 問題**：

```
初始狀態：
  Memory[x] = 0
  Cache 0[x] = 空
  Cache 1[x] = 空

步驟 1：Core 0 讀取 x
  Cache 0[x] = 0 (從 Memory 載入)

步驟 2：Core 1 寫入 x = 1
  Cache 1[x] = 1 (寫入 Cache)
  Memory[x] = 1 (Write-Back)

步驟 3：Core 0 再次讀取 x
  Cache 0[x] = 0 (舊資料！)
  
  → Coherency 問題！
```

---

### 1.2 Coherency 協議的基本概念

**MESI 協議**：

```
4 種狀態：

M (Modified):
  - 資料已修改
  - 只有這個 Cache 有最新資料
  - Memory 的資料是舊的

E (Exclusive):
  - 資料未修改
  - 只有這個 Cache 有資料
  - Memory 的資料是最新的

S (Shared):
  - 資料未修改
  - 多個 Cache 都有資料
  - Memory 的資料是最新的

I (Invalid):
  - 資料無效
  - 需要從其他 Cache 或 Memory 載入
```

**MESI 訊息類型**：

```
1. Read Request:
   - 請求讀取資料
   - 從 Shared 或 Exclusive 狀態的 Cache 取得

2. Read Exclusive Request:
   - 請求寫入資料
   - 需要 Invalidate 其他 Cache 的副本

3. Invalidate:
   - 通知其他 Cache 將資料標記為 Invalid
   - 確保只有一個 Cache 有最新資料

4. Data Response:
   - 回傳資料
   - 可能來自 Cache 或 Memory

5. Acknowledgement:
   - 確認 Invalidate 已完成
   - 確保所有 Cache 都已更新狀態
```

---

### 1.3 NoC 在 Coherency 中的角色

**NoC 的責任**：

```
1. 傳輸 Coherency 訊息:
   - Read Request
   - Invalidate
   - Data Response
   - Acknowledgement

2. 確保訊息順序:
   - Invalidate 必須在 Data Response 之前到達
   - Acknowledgement 必須在 Invalidate 之後到達

3. 避免死鎖:
   - Request 和 Response 不能互相阻塞
   - 需要使用 Virtual Channels

4. 提供低延遲:
   - Coherency 訊息的延遲直接影響性能
   - 需要優化 Router Pipeline
```

---

## 二、Directory-Based Coherency

### 2.1 為什麼需要 Directory？

**Snooping 協議的問題**：

```
Snooping (廣播式):
  Core 0 發出 Read Request
  → 廣播到所有 Core
  → 所有 Core 檢查自己的 Cache
  → 有資料的 Core 回應

問題：
  - 廣播消耗大量頻寬
  - 不適合大規模多核心（> 16 核心）
  - NoC 的 Bisection Bandwidth 不足
```

**Directory 協議的優勢**：

```
Directory (點對點):
  Core 0 發出 Read Request
  → 發送到 Directory
  → Directory 查詢哪些 Core 有資料
  → 只發送給有資料的 Core

優點：
  ✅ 減少廣播，節省頻寬
  ✅ 適合大規模多核心（> 64 核心）
  ✅ 適合 NoC 架構
```

---

### 2.2 Directory 的結構

**Directory Entry**：

```
每個 Cache Line 有一個 Directory Entry:

┌─────────────────────────────────────┐
│  Directory Entry (64 bytes)         │
├─────────────────────────────────────┤
│  State: M / E / S / I               │
│  Owner: Core ID (如果是 M 或 E)     │
│  Sharers: Bit Vector (如果是 S)    │
│    [Core 0, Core 1, ..., Core 15]   │
└─────────────────────────────────────┘

範例：
  State = S
  Sharers = [1, 0, 1, 0, ..., 0]
  → Core 0 和 Core 2 有這個 Cache Line
```

**Directory 的位置**：

```
選項 1：集中式 Directory
  - 所有 Directory Entry 在一個地方
  - 簡單，但可能成為瓶頸

選項 2：分散式 Directory
  - Directory Entry 分散在多個 Cache Slice
  - 複雜，但可擴展性好
  
業界常見：分散式 Directory
  - Intel Mesh：每個 Cache Slice 有自己的 Directory
  - ARM CMN-700：每個 HN-F (Home Node) 有自己的 Directory
```

---

### 2.3 Directory 協議的運作

**Read Request 流程**：

```
步驟 1：Core 0 發出 Read Request
  Core 0 → NoC → Directory (Cache Slice 5)

步驟 2：Directory 查詢狀態
  Directory Entry:
    State = M
    Owner = Core 3
  
  → Core 3 有最新資料

步驟 3：Directory 發出 Forward Request
  Directory → NoC → Core 3

步驟 4：Core 3 回應資料
  Core 3 → NoC → Core 0 (Data Response)
  Core 3 → NoC → Directory (Data Response)

步驟 5：Directory 更新狀態
  Directory Entry:
    State = S
    Sharers = [Core 0, Core 3]
```

**Write Request 流程**：

```
步驟 1：Core 1 發出 Write Request
  Core 1 → NoC → Directory (Cache Slice 5)

步驟 2：Directory 查詢狀態
  Directory Entry:
    State = S
    Sharers = [Core 0, Core 3]
  
  → 需要 Invalidate Core 0 和 Core 3

步驟 3：Directory 發出 Invalidate
  Directory → NoC → Core 0 (Invalidate)
  Directory → NoC → Core 3 (Invalidate)

步驟 4：Core 0 和 Core 3 回應 Acknowledgement
  Core 0 → NoC → Directory (Ack)
  Core 3 → NoC → Directory (Ack)

步驟 5：Directory 回應 Core 1
  Directory → NoC → Core 1 (Data Response)

步驟 6：Directory 更新狀態
  Directory Entry:
    State = M
    Owner = Core 1
```

---

## 三、Virtual Channels 與 Protocol Deadlock

### 3.1 Protocol Deadlock 的問題

**死鎖場景**：

```
場景：
  Core 0 發出 Read Request → Directory
  Directory 發出 Forward Request → Core 1
  Core 1 發出 Data Response → Core 0
  
  但 Core 1 的 Data Response 被 Core 0 的 Read Request 阻塞
  → 死鎖！

原因：
  Request 和 Response 使用同一個 Virtual Channel
  → Request 塞滿 Buffer
  → Response 無法發送
```

**圖示**：

```
Core 0 → Request → Directory
         ↑ (Buffer 滿)
         │
Core 1 → Response → Core 0
         ↑ (無法發送)
         │
Directory → Forward → Core 1
```

---

### 3.2 Virtual Channels 的解決方案

**3 個 Virtual Channels**：

```
VC0: Request Channel
  - Read Request
  - Write Request
  
VC1: Response Channel
  - Data Response
  - Acknowledgement
  
VC2: Snoop Channel
  - Invalidate
  - Forward Request

規則：
  - Response 的優先級高於 Request
  - Snoop 的優先級高於 Request
  - Response 和 Snoop 不會互相阻塞
```

**避免死鎖**：

```
Core 0 → Request (VC0) → Directory
         ↑ (VC0 Buffer 滿，但不影響 VC1)
         │
Core 1 → Response (VC1) → Core 0
         ↑ (VC1 可以發送)
         │
Directory → Forward (VC2) → Core 1

→ 無死鎖！
```

---

### 3.3 Virtual Channels 的硬體成本

**成本分析**：

```
假設：
  每個 VC：8 Flits
  Flit 寬度：128 bits
  
1 個 VC 的 Buffer:
  8 × 128 = 1,024 bits = 128 bytes

3 個 VC 的 Buffer:
  3 × 128 = 384 bytes

5 個 Input Port (N, S, E, W, Local):
  5 × 384 = 1,920 bytes = ~0.04 mm² (7nm)

功耗：
  3 VCs vs 1 VC: ~2x 功耗增加
```

**權衡**：

```
優點：
  ✅ 避免 Protocol Deadlock
  ✅ 提高吞吐量（不同類型的訊息並行）

缺點：
  ❌ 面積增加 ~2x
  ❌ 功耗增加 ~2x
  
結論：
  對於多核心 SoC，Virtual Channels 是必須的
```

---

## 四、Snoop Filter 的設計

### 4.1 什麼是 Snoop Filter？

**問題**：

```
Directory 協議需要追蹤所有 Cache Line 的狀態
→ Directory 的大小隨著 Cache 大小增加

範例：
  16 個 Core
  每個 Core：1 MB L2 Cache
  Cache Line：64 bytes
  
  總 Cache Lines：16 × 1 MB / 64 = 256K Lines
  
  每個 Directory Entry：16 bits (Sharers Bit Vector)
  總 Directory 大小：256K × 16 bits = 512 KB
  
  → Directory 太大！
```

**Snoop Filter 的解決方案**：

```
Snoop Filter:
  只追蹤在多個 Cache 中共享的 Cache Line
  → 大幅減少 Directory 的大小

範例：
  實際共享的 Cache Lines：~10%
  Snoop Filter 大小：256K × 10% × 16 bits = 51.2 KB
  
  → 減少 10x！
```

---

### 4.2 Snoop Filter 的結構

**Bloom Filter**：

```
Bloom Filter:
  一種概率性資料結構
  可以快速判斷一個元素是否在集合中
  
優點：
  ✅ 空間效率高
  ✅ 查詢速度快 (O(1))

缺點：
  ❌ 可能有 False Positive（誤判為存在）
  ❌ 無法刪除元素（需要 Counting Bloom Filter）
```

**Snoop Filter 實作**：

```
┌─────────────────────────────────────┐
│  Snoop Filter (Bloom Filter)        │
├─────────────────────────────────────┤
│  Bit Array: [0, 1, 0, 1, 1, ...]    │
│  Size: 64 KB (512K bits)            │
│                                     │
│  Hash Functions: 3 個               │
│    h1(addr) = addr % 512K           │
│    h2(addr) = (addr * 31) % 512K    │
│    h3(addr) = (addr * 97) % 512K    │
└─────────────────────────────────────┘

插入 (Insert):
  bit_array[h1(addr)] = 1
  bit_array[h2(addr)] = 1
  bit_array[h3(addr)] = 1

查詢 (Query):
  if (bit_array[h1(addr)] && 
      bit_array[h2(addr)] && 
      bit_array[h3(addr)]):
      return "可能存在"
  else:
      return "一定不存在"
```

---

### 4.3 Snoop Filter 的性能

**False Positive 的影響**：

```
False Positive:
  Snoop Filter 誤判某個 Cache Line 被共享
  → 發送不必要的 Snoop 訊息
  → 浪費頻寬，但不影響正確性

範例：
  False Positive Rate: 1%
  每秒 1M 次 Cache 訪問
  → 每秒 10K 次不必要的 Snoop

  影響：
    頻寬浪費：~1%
    延遲增加：~0.5%

  → 可接受
```

**Snoop Filter 的優化**：

```
1. Counting Bloom Filter:
   - 使用 Counter 而非 Bit
   - 可以刪除元素
   - 但空間增加 4x

2. Hierarchical Snoop Filter:
   - 多層 Snoop Filter
   - 第一層：粗粒度（快速）
   - 第二層：細粒度（精確）

3. Adaptive Snoop Filter:
   - 根據負載動態調整大小
   - 低負載：小 Filter
   - 高負載：大 Filter
```

---

## 五、真實案例：Intel 和 ARM 的 Coherency 設計

### 5.1 Intel Skylake-SP Mesh Coherency

**架構**：

```
拓撲：7×4 Mesh
Core 數量：28 個
Cache Slice 數量：28 個 (每個 Core 一個)

Coherency 協議：
  - Directory-Based MESIF
  - 分散式 Directory (每個 Cache Slice)
  - Snoop Filter (Bloom Filter)

Virtual Channels：
  - VC0: Request (Read, Write)
  - VC1: Snoop (Invalidate, Forward)
  - VC2: Response (Data, Ack)
```

**Directory 分佈**：

```
Address Hashing:
  Cache Slice ID = hash(address) % 28

  hash(address) = (address >> 6) XOR (address >> 12)

  → 均勻分佈到 28 個 Cache Slice

範例：
  address = 0x1000
  hash = (0x1000 >> 6) XOR (0x1000 >> 12)
       = 0x40 XOR 0x1
       = 0x41
  Cache Slice ID = 0x41 % 28 = 9

  → 這個地址的 Directory 在 Cache Slice 9
```

**Snoop Filter**：

```
每個 Cache Slice 有一個 Snoop Filter:
  - Size: 64 KB (Bloom Filter)
  - Hash Functions: 3 個
  - False Positive Rate: ~1%

總 Snoop Filter 大小：
  28 × 64 KB = 1.75 MB

  vs 完整 Directory:
  28 × 1.375 MB / 64 × 28 bits = 15 MB

  → 減少 ~8.5x
```

**性能**：

```
Coherency 延遲：
  Local (同一個 Cache Slice):
    2 hops × 2 cycles = 4 cycles = 1.6 ns

  Remote (不同 Cache Slice):
    平均 4.5 hops × 2 cycles = 9 cycles = 3.6 ns

  Worst-case (對角線):
    9 hops × 2 cycles = 18 cycles = 7.2 ns

Coherency 頻寬：
  每條 Link：160 GB/s
  Bisection Bandwidth：1,120 GB/s

  Coherency 訊息佔比：~30%
  → Coherency 頻寬：336 GB/s
```

---

### 5.2 ARM CMN-700 Coherency

**架構**：

```
拓撲：最多 8×8 Mesh
Core 數量：最多 128 個
HN-F (Home Node - Fully Coherent) 數量：最多 64 個

Coherency 協議：
  - Directory-Based MOESI
  - 分散式 Directory (每個 HN-F)
  - Snoop Filter (Counting Bloom Filter)

Virtual Channels：
  - VC0: Request (Read, Write)
  - VC1: Response (Data, Ack, Snoop Response)
```

**Directory 分佈**：

```
Address Interleaving:
  HN-F ID = address[11:6] % 64

  → 以 64-byte Cache Line 為單位交錯分佈

範例：
  address = 0x1040
  HN-F ID = (0x1040 >> 6) % 64
          = 0x41 % 64
          = 1

  → 這個地址的 Directory 在 HN-F 1
```

**Snoop Filter**：

```
每個 HN-F 有一個 Snoop Filter:
  - Type: Counting Bloom Filter
  - Size: 128 KB
  - Counters: 4-bit (最多 15 個 Sharers)
  - Hash Functions: 4 個
  - False Positive Rate: ~0.5%

總 Snoop Filter 大小：
  64 × 128 KB = 8 MB

  vs 完整 Directory:
  64 × 2 MB / 64 × 128 bits = 256 MB

  → 減少 ~32x
```

**QoS (Quality of Service)**：

```
ARM CMN-700 支援 QoS:
  - 不同的 Device 有不同的優先級
  - 高優先級的 Coherency 訊息優先處理

範例：
  Priority 0 (最高): CPU Core
  Priority 1: GPU
  Priority 2: DMA
  Priority 3 (最低): I/O Device

實作：
  - 每個 VC 有多個 Priority Queue
  - Arbiter 優先選擇高優先級的訊息
```

**性能**：

```
Coherency 延遲：
  Local (同一個 HN-F):
    2 hops × 2 cycles = 4 cycles = 2 ns @ 2 GHz

  Remote (不同 HN-F):
    平均 6 hops × 2 cycles = 12 cycles = 6 ns

  Worst-case (對角線):
    14 hops × 2 cycles = 28 cycles = 14 ns

Coherency 頻寬：
  每條 Link：64 GB/s
  Bisection Bandwidth：512 GB/s

  Coherency 訊息佔比：~40%
  → Coherency 頻寬：205 GB/s
```

---

## 六、我的經驗：Coherency Bug 的除錯

### 6.1 2017 年的 Invalidate 延遲 Bug

**問題重現**：

```
測試程式：
  // Core 0
  x = 1;
  printf("Core 0: x = %d\n", x);

  // Core 1
  x = 2;
  printf("Core 1: x = %d\n", x);

預期：
  Core 0: x = 1 或 x = 2
  Core 1: x = 2

實際：
  Core 0: x = 1
  Core 1: x = 1 (錯誤！)
```

**除錯過程**：

```
步驟 1：檢查 MESI 狀態機
  → 狀態轉換正確

步驟 2：檢查 Directory
  → Directory Entry 正確

步驟 3：檢查 NoC 訊息
  → 發現 Invalidate 訊息被延遲！

步驟 4：分析 NoC Waveform
  Cycle 100: Core 1 發出 Write Request → Directory
  Cycle 105: Directory 發出 Invalidate → Core 0
  Cycle 110: Core 0 收到 Invalidate (應該在 Cycle 107)
  Cycle 111: Core 0 發出 Ack → Directory
  Cycle 115: Directory 發出 Data Response → Core 1

  → Invalidate 延遲了 3 cycles！
```

**根本原因**：

```
Invalidate 使用 VC0 (Request Channel)
→ VC0 被 Read Request 塞滿
→ Invalidate 無法發送
→ 延遲

正確做法：
  Invalidate 應該使用 VC2 (Snoop Channel)
  → VC2 優先級高於 VC0
  → Invalidate 不會被延遲
```

**修復**：

```
修改 Coherency Protocol:
  Invalidate 從 VC0 → VC2
  Forward Request 從 VC0 → VC2

修改 Router:
  增加 VC2 的優先級
  VC2 > VC1 > VC0

測試：
  Core 0: x = 2
  Core 1: x = 2

  → 正確！
```

---

### 6.2 2018 年的 Directory Overflow Bug

**問題**：

```
測試程式：
  // 16 個 Core 同時讀取同一個變數
  for (int i = 0; i < 16; i++) {
    printf("Core %d: x = %d\n", i, x);
  }

預期：
  所有 Core 都能讀取到 x

實際：
  Core 0-14: x = 0 (正確)
  Core 15: Timeout (錯誤！)
```

**除錯過程**：

```
步驟 1：檢查 Directory Entry
  State = S
  Sharers = [1, 1, 1, ..., 1, 0]  (15 個 1)

  → Core 15 沒有被記錄！

步驟 2：檢查 Sharers Bit Vector
  Bit Vector 大小：16 bits
  但只能記錄 15 個 Sharers

  → Overflow！
```

**根本原因**：

```
Sharers Bit Vector 使用 16 bits
→ 可以記錄 16 個 Core

但實作時使用了 15 bits (0-14)
→ Core 15 無法被記錄

正確做法：
  使用完整的 16 bits (0-15)
```

**修復**：

```
修改 Directory Entry:
  Sharers Bit Vector: 15 bits → 16 bits

測試：
  所有 Core 都能讀取到 x

  → 正確！
```

---

### 6.3 除錯工具和方法

**工具**：

```
1. Waveform Viewer:
   - 查看 NoC 訊息的時序
   - 檢查 Coherency 訊息的順序
   - 工具：Verdi, DVE

2. Coherency Checker:
   - 自動檢查 MESI 狀態轉換
   - 檢查 Directory 一致性
   - 工具：自己寫的 Python Script

3. NoC Tracer:
   - 記錄所有 NoC 訊息
   - 分析訊息的路徑和延遲
   - 工具：自己寫的 Chisel Module

4. Simulation:
   - 使用 Verilator 或 VCS
   - 跑大量的隨機測試
   - 檢查 Assertion
```

**方法**：

```
1. 從簡單的測試開始:
   - 2 個 Core
   - 1 個變數
   - 簡單的讀寫操作

2. 逐步增加複雜度:
   - 增加 Core 數量
   - 增加變數數量
   - 增加操作複雜度

3. 使用 Assertion:
   - 檢查 MESI 狀態轉換
   - 檢查 Directory 一致性
   - 檢查 NoC 訊息順序

4. 記錄所有訊息:
   - 使用 Tracer 記錄所有 NoC 訊息
   - 分析訊息的時序和順序
   - 找出異常的訊息

5. 重現 Bug:
   - 找到最小的重現步驟
   - 簡化測試程式
   - 專注於根本原因
```

---

## 七、性能優化技巧

### 7.1 減少 Coherency 訊息

**技巧 1：Cache Line Prefetching**

```
概念：
  預測即將訪問的 Cache Line
  → 提前載入到 Cache
  → 減少 Cache Miss

範例：
  訪問 address[0]
  → 預測會訪問 address[1], address[2], ...
  → 提前載入 address[1], address[2]

效果：
  Cache Miss Rate: 10% → 5%
  Coherency 訊息: 減少 50%
```

**技巧 2：Write Coalescing**

```
概念：
  合併多個寫入操作
  → 減少 Invalidate 訊息

範例：
  寫入 x[0] = 1
  寫入 x[1] = 2
  寫入 x[2] = 3

  → 合併為一個 Write Request
  → 只發送一次 Invalidate

效果：
  Invalidate 訊息: 減少 70%
```

**技巧 3：Shared Cache**

```
概念：
  多個 Core 共享 L3 Cache
  → 減少 Core 之間的 Coherency 訊息

範例：
  Core 0 和 Core 1 共享 L3 Cache
  → Core 0 寫入 x
  → Core 1 讀取 x (從 L3 Cache)
  → 不需要 Coherency 訊息

效果：
  Coherency 訊息: 減少 40%
```

---

### 7.2 優化 Directory 訪問

**技巧 1：Directory Cache**

```
概念：
  在 Core 附近放置小的 Directory Cache
  → 減少訪問遠端 Directory 的延遲

範例：
  Directory Cache: 4 KB (256 Entries)
  Hit Rate: ~80%

  延遲：
    Hit: 2 cycles
    Miss: 9 cycles

  平均延遲：
    0.8 × 2 + 0.2 × 9 = 3.4 cycles

  vs 無 Cache:
    9 cycles

  → 降低 62%
```

**技巧 2：Directory Prefetching**

```
概念：
  預測即將訪問的 Directory Entry
  → 提前載入到 Directory Cache

範例：
  訪問 address[0] 的 Directory
  → 預測會訪問 address[1] 的 Directory
  → 提前載入

效果：
  Directory Cache Hit Rate: 80% → 90%
  平均延遲: 3.4 cycles → 2.8 cycles
```

---

### 7.3 優化 NoC 路由

**技巧 1：Coherency-Aware Routing**

```
概念：
  根據 Coherency 訊息的類型選擇路徑
  → Invalidate 使用最短路徑
  → Data Response 使用較長但較少擁塞的路徑

範例：
  Invalidate: XY Routing (最短)
  Data Response: Adaptive Routing (避開擁塞)

效果：
  Invalidate 延遲: 降低 20%
  整體吞吐量: 提高 15%
```

**技巧 2：Multicast Routing**

```
概念：
  一次發送 Invalidate 到多個 Core
  → 減少訊息數量

範例：
  需要 Invalidate Core 0, 1, 2

  傳統：
    Directory → Core 0
    Directory → Core 1
    Directory → Core 2
    (3 個訊息)

  Multicast:
    Directory → Core 0 → Core 1 → Core 2
    (1 個訊息，沿途複製)

效果：
  Invalidate 訊息: 減少 60%
  頻寬: 節省 40%
```

---

## 八、總結

NoC 與 Cache Coherency 的整合是多核心 SoC 設計的核心：

1. **Coherency 協議**：
   - MESI/MOESI 狀態機
   - Directory-Based 協議
   - Snoop Filter (Bloom Filter)

2. **Virtual Channels**：
   - 避免 Protocol Deadlock
   - 3 個 VC: Request, Response, Snoop
   - 硬體成本：~2x 面積和功耗

3. **真實案例**：
   - Intel Mesh：28 核心, MESIF, Bloom Filter
   - ARM CMN-700：128 核心, MOESI, Counting Bloom Filter, QoS

4. **除錯經驗**：
   - Invalidate 延遲 Bug：VC 分配錯誤
   - Directory Overflow Bug：Bit Vector 大小錯誤
   - 工具：Waveform Viewer, Coherency Checker, NoC Tracer

5. **性能優化**：
   - 減少 Coherency 訊息：Prefetching, Write Coalescing, Shared Cache
   - 優化 Directory：Directory Cache, Prefetching
   - 優化路由：Coherency-Aware Routing, Multicast

理解 NoC 與 Cache Coherency 的整合，可以幫助我們：
- 設計正確的多核心系統
- 避免 Coherency Bug
- 優化系統性能

---

## 下一篇預告

在下一篇文章中，我們會深入探討：

**NoC 效能分析與優化**
- 性能指標（延遲、吞吐量、功耗）
- 模擬工具和方法
- 瓶頸分析
- 優化技術
- 真實案例分析

敬請期待！

---

## 參考資料

**公開文檔**：
- *Principles and Practices of Interconnection Networks* - Dally & Towles
- *A Primer on Memory Consistency and Cache Coherency* - Sorin, Hill & Wood (2011)
- Intel Xeon Scalable Processor Architecture Specification
- ARM CoreLink CMN-700 Technical Reference Manual
- *Directory-Based Cache Coherence Protocols* - Culler & Singh (1999)

**個人筆記**：
- Network-on-Chip Technical Notes (基於公開領域知識整理)
- Cache Coherency Protocol Notes (基於公開領域知識整理)
- 2017-2018 年 Coherency Bug 除錯經驗（已脫敏，無機敏資訊）


---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: https://github.com/djiangtw/tech-column-public
