# 現代 CPU Cache 架構設計：從 L1 到 L3 的設計哲學

**作者**: Danny Jiang  
**日期**: 2025-12-07  

---

## 前言：一次性能調校的頓悟

2019 年，我在優化一個高頻交易系統時遇到了一個奇怪的現象：同樣的程式碼，在 Intel Xeon 上跑得飛快，但在 AMD EPYC 上卻慢了 30%。更詭異的是，當我把資料結構從 48KB 改成 32KB 後，AMD 的性能突然超越了 Intel。

這讓我開始深入研究：**為什麼不同 CPU 的 Cache 設計會如此不同？L1、L2、L3 各自扮演什麼角色？**

經過幾週的研究和實驗，我終於理解了現代 CPU Cache 架構背後的設計哲學。今天，我想和你分享這些洞察。

---

## 一、Cache 階層的設計哲學：速度、容量、命中率的三角習題

### 1.1 辦公桌的比喻

想像你是一個研究員，需要處理大量文獻：

```
你的辦公桌 (L1 Cache):

  - 極小 (只能放 2-3 本書)
  - 極快 (伸手就拿到)
  - 但容量有限，放不下太多東西

你的書架 (L2 Cache):

  - 中等大小 (可以放 20-30 本書)
  - 快 (走幾步就到)
  - 容量比辦公桌大，但還是有限

圖書館 (L3 Cache):

  - 很大 (可以放數千本書)
  - 慢 (要走到圖書館)
  - 容量大，但存取時間長

倉庫 (Main Memory):

  - 超大 (可以放數萬本書)
  - 超慢 (要開車去倉庫)
  - 容量無限，但延遲極高

```

**核心問題**：如何在這四個層級之間取得平衡？

---

### 1.2 設計的三角習題

現代 CPU 設計師面臨一個經典的三角習題：

```
        速度
         ↑
         │
         │
容量 ←───┼───→ 命中率
         │
         │
         ↓
      硬體成本

```

**你無法同時最大化所有指標**：

- 要速度快 → 容量必須小 (SRAM 延遲與容量成正比)
- 要容量大 → 速度必然慢 (更多 Sets 需要更多解碼時間)
- 要命中率高 → 需要高 Associativity (更多比較器 = 更高成本)

**解決方案**：分層設計，每一層有不同的優化目標。

---

## 二、L1 Cache：速度至上的極致設計

### 2.1 為什麼 L1 必須極快？

**真實案例**：Intel Core i9-12900K 的 L1 延遲只有 **4 cycles**。

> 註：以下關於頻率、cycle 數與延遲的數字，採用公開文件與代表性設定做近似估計，目的是幫助理解「延遲等級差」，並非嚴格的 benchmark 結果，實際數值會依具體 CPU 與平台而異。

這意味著什麼？

```
假設 CPU 頻率 = 5 GHz
1 cycle = 0.2 ns (奈秒)
L1 延遲 = 4 cycles = 0.8 ns

如果 L1 延遲增加到 8 cycles:
  每次存取多花 4 cycles
  如果程式每 10 條指令就存取一次記憶體
  性能損失 = 4 / 10 = 40%！

```

**結論**：L1 的延遲直接影響 CPU 的 IPC (Instructions Per Cycle)，必須極快。

---

### 2.2 L1 的設計特點

#### A. VIPT (Virtually Indexed, Physically Tagged)

**為什麼用 VIPT？**

```
傳統 PIPT 流程：
  VA → TLB (4-5 cycles) → PA → Cache Index → Data
  總延遲 = TLB + Cache = 4 + 4 = 8 cycles

VIPT 流程：
  VA → Cache Index (並行)
  VA → TLB → PA Tag (並行)
  兩條路徑同時進行，最後比較 Tag
  總延遲 = max(TLB, Cache) = 4 cycles

節省 = 50% 延遲！

```

**代價**：容量受限於 Page Size。

```
4KB Page:
  Index 最多用 bit[11:6] = 6 bits
  最大 Sets = 64
  最大容量 = 64 × 16-Way × 64B = 64KB

```

---

#### B. 分離 I-Cache 和 D-Cache

**為什麼要分離？**

```
Harvard Architecture 的優勢：
  CPU 可以同時 Fetch 指令和存取資料
  
如果 I 和 D 共用同一個 Cache:
  Fetch 和 Load/Store 會互相競爭
  → 結構性衝突 (Structural Hazard)
  → 性能下降

```

**實際案例**：

```
Intel Core i9-12900K (P-Core):
  L1I: 32KB, 8-Way
  L1D: 48KB, 12-Way
  分離設計，避免衝突

ARM Cortex-X4:
  L1I: 64KB, 4-Way
  L1D: 64KB, 4-Way
  分離設計，支援高 IPC

```

---

#### C. 高 Associativity

**為什麼 L1 需要高 Associativity？**

```
L1 容量小 (32-64KB)
如果 Associativity 低 (例如 4-Way):
  衝突率高
  → Cache Miss 增加
  → 性能下降

解決方案：提高 Associativity
  Intel: 8-Way, 12-Way
  ARM: 4-Way (但用更大的容量補償)

```

**硬體成本**：

```
8-Way Set Associative:
  需要 8 個 Tag 比較器
  需要 8:1 Multiplexer
  面積和功耗都增加

但對於 L1 來說，速度比成本更重要！

```

---

### 2.3 L1 的容量限制

**為什麼 L1 通常只有 32-64KB？**

#### 原因 1：VIPT 的數學限制

```
4KB Page, 64B Line:
  Index 最多用 bit[11:6]
  最大 Sets = 64
  最大容量 = 64 × Ways × 64B

如果要 128KB:
  需要 128 Sets
  Index 需要 bit[12:6]
  → bit[12] 超出 Page Offset
  → 別名問題 (Aliasing)

```

#### 原因 2：延遲與容量的權衡

```
SRAM 延遲 ∝ √(容量)

32KB L1: 4 cycles
64KB L1: 5 cycles
128KB L1: 7 cycles

每增加 1 cycle 延遲 = 性能損失 10-20%

```

#### 原因 3：功耗

```
SRAM 功耗 ∝ 容量

64KB L1: 1W
128KB L1: 2W

對於行動裝置，功耗是關鍵限制

```

---

## 三、L2 Cache：承上啟下的中繼站

### 3.1 L2 的角色定位

**L2 的任務**：

1. **彌補 L1 的容量不足**
2. **避免頻繁存取 L3 或 DRAM**
3. **平衡速度與容量**

**延遲預算**：

```
L1 Miss → L2 Hit: 10-15 cycles
L1 Miss → L2 Miss → L3 Hit: 40-50 cycles
L1 Miss → L2 Miss → L3 Miss → DRAM: 200+ cycles

L2 的存在可以避免 80% 的 L3 存取

```

---

### 3.2 L2 的設計特點

#### A. PIPT (Physically Indexed, Physically Tagged)

**為什麼 L2 用 PIPT？**

```
L2 的延遲已經 10-15 cycles
TLB 查找 (4-5 cycles) 早就完成了
不需要 VIPT 的並行優化

PIPT 的優勢：

  1. 無別名問題
  2. 設計簡單
  3. 容量無限制

```

---

#### B. 統一 Cache (Unified Cache)

**為什麼 L2 不分離 I 和 D？**

```
L1 分離的原因：
  需要同時 Fetch 和 Load/Store
  
L2 不需要分離的原因：
  L2 的頻寬足夠
  統一 Cache 可以動態分配容量
  
範例：
  如果程式是計算密集型 (少量資料存取):
    L2 可以把更多空間給 I-Cache
  如果程式是資料密集型 (大量資料存取):
    L2 可以把更多空間給 D-Cache

```

---

#### C. 中等 Associativity

**為什麼 L2 通常是 8-Way 或 16-Way？**

```
L1: 8-12-Way (高 Associativity，提升命中率)
L2: 8-16-Way (中等 Associativity，平衡成本)
L3: 12-20-Way (低 Associativity，容量優先)

L2 的 Associativity 選擇：
  太低 (4-Way): 衝突率高
  太高 (32-Way): 硬體成本高，延遲增加
  剛好 (8-16-Way): 平衡點

```

---

### 3.3 L2 的容量設計

**為什麼 L2 通常是 256KB - 2MB？**

```
太小 (128KB):
  無法有效彌補 L1 的不足
  L3 存取頻率過高
  
剛好 (256KB - 1MB):
  可以容納大部分 Working Set
  避免頻繁存取 L3
  
太大 (4MB):
  延遲增加
  成本過高 (每個 Core 都有獨立 L2)

```

**實際案例**：

```
Intel Core i9-12900K (P-Core):
  L2: 1.25MB per core
  
AMD Ryzen 9 7950X (Zen 4):
  L2: 1MB per core
  
ARM Cortex-X4:
  L2: 512KB - 2MB (可配置)

```

---

## 四、L3 Cache：最後一道防線

### 4.1 L3 的角色定位

**L3 的任務**：

1. **避免存取 DRAM** (200+ cycles vs 40-50 cycles)
2. **所有 Core 共用** (減少重複資料)
3. **容量優先** (速度已經不是第一考量)

**真實案例**：

```
2020 年，我在優化一個視訊編碼器時發現：
  增加 L3 容量從 16MB 到 32MB
  性能提升 25%
  
原因：
  視訊編碼需要大量參考幀
  L3 可以容納更多參考幀
  避免頻繁存取 DRAM

```

---

### 4.2 L3 的設計特點

#### A. Shared Cache (共享快取)

**為什麼 L3 要所有 Core 共用？**

```
優勢 1：減少重複資料
  如果每個 Core 都有獨立 L3:
    同一份資料可能被複製 8 次
    浪費容量
    
  共享 L3:
    同一份資料只存一份
    容量利用率高

優勢 2：Core 間通訊
  Core A 寫入資料
  Core B 可以直接從 L3 讀取
  不需要經過 DRAM

```

**挑戰**：

```
挑戰 1：頻寬競爭
  8 個 Core 同時存取 L3
  → 頻寬不足
  
解決方案：Slicing (切片)
  把 L3 分成多個 Slice
  分散在不同位置
  增加總頻寬

挑戰 2：延遲不一致
  Core 0 存取 Slice 0: 快
  Core 0 存取 Slice 7: 慢
  
解決方案：Hash 函數
  用地址的某些 bits 決定存在哪個 Slice
  平均分散存取

```

---

#### B. Slicing (切片架構)

**什麼是 Slicing？**

```
傳統設計：
  L3 是一個大的整體
  所有 Core 共用一個 Port
  頻寬 = 單一 Port 的頻寬

Slicing 設計：
  L3 分成 N 個 Slice (通常 N = Core 數量)
  每個 Slice 有獨立的 Port
  總頻寬 = N × 單一 Port 頻寬

```

**實際案例**：

```
Intel Core i9-12900K:
  L3: 30MB, 12-Way
  分成 8 個 Slice (每個 Slice ~3.75MB)
  每個 P-Core 附近有一個 Slice
  
地址映射：
  用地址的 bit[8:6] 決定存在哪個 Slice
  Hash 函數確保平均分散

```

---

#### C. 低 Associativity

**為什麼 L3 的 Associativity 較低？**

```
L1: 8-12-Way (速度優先)
L2: 8-16-Way (平衡)
L3: 12-20-Way (容量優先)

L3 容量大 (數 MB 到數十 MB)
即使 Associativity 低，衝突率也不高

範例：
  32MB L3, 12-Way, 64B Line
  Sets = 32MB / (12 × 64B) = 43,690 Sets
  
  即使是 Direct Mapped (1-Way):
    Sets = 524,288
    衝突率也很低

```

---

### 4.3 L3 的容量設計

**為什麼 L3 通常是 8MB - 64MB？**

```
消費級 CPU:
  Intel Core i9: 30MB
  AMD Ryzen 9: 64MB
  
伺服器 CPU:
  Intel Xeon: 60MB - 120MB
  AMD EPYC: 256MB - 384MB

```

**容量選擇的考量**：

```
因素 1：Working Set Size
  如果程式的 Working Set < L3 容量
  → 幾乎不會存取 DRAM
  → 性能極佳
  
因素 2：成本
  SRAM 成本 ∝ 容量
  L3 佔 CPU Die 面積的 30-50%
  
因素 3：功耗
  更大的 L3 = 更高的功耗
  對於行動裝置，需要權衡

```

---

## 五、真實案例分析：Intel vs AMD vs ARM

### 5.1 Intel Core i9-12900K (Alder Lake, 2021)

```
P-Core (Performance Core):
  L1I: 32KB, 8-Way, VIPT
  L1D: 48KB, 12-Way, VIPT
  L2:  1.25MB, 10-Way, PIPT (per core)
  L3:  30MB, 12-Way, PIPT (shared, 8 slices)

E-Core (Efficiency Core):
  L1I: 64KB, 8-Way, VIPT
  L1D: 32KB, 8-Way, VIPT
  L2:  2MB, 16-Way, PIPT (per 4 E-cores)
  L3:  30MB, 12-Way, PIPT (shared)

```

**設計特點**：

1. **P-Core 的 L1D 是 48KB** - 突破傳統 32KB 限制
2. **E-Core 共享 L2** - 4 個 E-Core 共用 2MB L2
3. **L3 Slicing** - 8 個 Slice，分散在不同位置

---

### 5.2 AMD Ryzen 9 7950X (Zen 4, 2022)

```
每個 Core:
  L1I: 32KB, 8-Way, VIPT
  L1D: 32KB, 8-Way, VIPT
  L2:  1MB, 8-Way, PIPT (per core)
  L3:  32MB, 16-Way, PIPT (per CCD, 8 cores share)

```

**設計特點**：

1. **L2 容量大** - 每個 Core 有 1MB L2
2. **L3 分成兩個 CCD** - 每個 CCD 有 8 個 Core，共享 32MB L3
3. **總 L3 = 64MB** - 兩個 CCD 合計

---

### 5.3 ARM Cortex-X4 (2023)

```
L1I: 64KB, 4-Way, VIPT
L1D: 64KB, 4-Way, VIPT
L2:  512KB - 2MB, 8-Way, PIPT (per core, 可配置)
L3:  可配置, PIPT (shared)

```

**設計特點**：

1. **L1 容量大** - 64KB L1I 和 L1D
2. **L2 可配置** - 從 512KB 到 2MB
3. **靈活性高** - 適合不同應用場景

> 註：本節中列舉的 Intel / AMD / ARM Cache 參數與架構特性，均依撰文時官方公開文件整理而成，未來世代處理器可能調整具體大小與組態，請以最新官方資料為準；本文重點在於說明設計哲學與取捨方向。

---

## 六、設計權衡的總結

### 6.1 三層 Cache 的對比表

| 特性 | L1 Cache | L2 Cache | L3 Cache |
|------|----------|----------|----------|
| **目標** | 速度至上 | 平衡速度與容量 | 容量優先 |
| **延遲** | 4-5 cycles | 10-15 cycles | 40-50 cycles |
| **容量** | 32-64KB | 256KB-2MB | 8MB-64MB |
| **Indexing** | VIPT | PIPT | PIPT |
| **Associativity** | 8-12-Way | 8-16-Way | 12-20-Way |
| **分離/統一** | 分離 (I/D) | 統一 | 統一 |
| **共享** | Per Core | Per Core | Shared |
| **Slicing** | 無 | 無 | 有 |

---

### 6.2 設計決策的原因

**L1 為什麼用 VIPT？**

- 隱藏 TLB 延遲，節省 50% 時間

**L1 為什麼分離 I 和 D？**

- 避免結構性衝突，支援高 IPC

**L2 為什麼用 PIPT？**

- TLB 已完成，無需並行優化
- 無別名問題，設計簡單

**L3 為什麼共享？**

- 減少重複資料，提升容量利用率
- 支援 Core 間通訊

**L3 為什麼用 Slicing？**

- 增加總頻寬，避免頻寬競爭

---

## 七、給程式設計師的建議

### 7.1 理解 Cache 階層的影響

#### 建議 1：優化 Working Set Size


```c
// 不好的設計：Working Set 超過 L2
struct large_data {
    char buffer[2 * 1024 * 1024];  // 2MB
};

// 好的設計：Working Set 在 L2 內
struct small_data {
    char buffer[256 * 1024];  // 256KB
};

```

#### 建議 2：利用 L3 的共享特性


```c
// 不好的設計：每個 Thread 都有自己的副本
__thread char buffer[1024 * 1024];  // 每個 Thread 1MB

// 好的設計：共享資料，利用 L3
char shared_buffer[1024 * 1024];  // 所有 Thread 共用

```

#### 建議 3：避免跨 NUMA Node 存取


```c
// 不好的設計：Thread 在 Node 0，資料在 Node 1
// 需要跨 Node 存取，延遲增加

// 好的設計：Thread 和資料在同一個 Node
// 利用 numactl 綁定

```

---

### 7.2 測量和優化

**工具**：

```bash
# Linux perf 測量 Cache Miss
perf stat -e cache-references,cache-misses ./your_program

# Intel VTune 分析 Cache 行為
vtune -collect memory-access ./your_program

# AMD uProf 分析 Cache 行為
AMDuProfCLI collect --event cache ./your_program

```

**指標**：

```
L1 Miss Rate < 5%: 優秀
L1 Miss Rate 5-10%: 良好
L1 Miss Rate > 10%: 需要優化

L2 Miss Rate < 1%: 優秀
L2 Miss Rate 1-5%: 良好
L2 Miss Rate > 5%: 需要優化

L3 Miss Rate < 0.1%: 優秀
L3 Miss Rate 0.1-1%: 良好
L3 Miss Rate > 1%: 需要優化

```

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: https://github.com/djiangtw/tech-column-public
