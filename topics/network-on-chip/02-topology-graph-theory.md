# NoC 拓撲結構的圖論分析：從數學到硬體

**作者**: Danny Jiang  
**日期**: 2025-12-07  

---

## 前言：當我第一次看到 Knuth 的圖論

2016 年，我在讀 Donald Knuth 的 *The Art of Computer Programming* 時，被第 7.2 章的圖論深深吸引。

當時我還不知道，這些看似抽象的數學概念（Vertex, Edge, Diameter, Bisection Bandwidth），會在幾年後成為我設計 NoC 時最重要的工具。

**圖論不是抽象的數學遊戲，而是硬體架構師手中的量化分析工具。**

在這篇文章中，我會帶你用圖論的視角，重新理解 NoC 的拓撲結構。

---

## 一、圖論基礎：NoC 的數學語言

### 1.1 什麼是圖 (Graph)？

**定義**：

```
圖 G = (V, E)
  V = Vertices (頂點集合)
  E = Edges (邊集合)

在 NoC 中：
  Vertex = Router
  Edge = Link (連接兩個 Router 的實體線路)
```

**範例**：

```
4 個 Router 的 Ring:

V = {R0, R1, R2, R3}
E = {(R0, R1), (R1, R2), (R2, R3), (R3, R0)}

圖示：
[R0] -- [R1]
 |       |
[R3] -- [R2]
```

---

### 1.2 圖論的關鍵指標

**1. Degree (度數)**

```
定義：一個頂點連接的邊數

在 NoC 中：
  Degree = Router 的 Port 數量
  
為什麼重要？
  Degree 越高 → Router 越複雜 → 面積越大 → 成本越高
```

**範例**：

```
Ring:
  Degree = 2（每個 Router 連接 2 個鄰居）
  
Mesh (內部節點):
  Degree = 4（North, South, East, West）
  
Mesh (邊緣節點):
  Degree = 2-3（少了一些方向）
```

---

**2. Diameter (直徑)**

```
定義：圖中任意兩點之間的最短路徑的最大值

在 NoC 中：
  Diameter = 封包在最壞情況下需要跳幾次 (Hops)
  
為什麼重要？
  Diameter 決定了 Zero-load Latency（空載延遲）
  Latency = Diameter × Hop Latency
```

**範例**：

```
Ring (N 個節點):
  Diameter = N/2
  例如：8 個節點，Diameter = 4

Mesh (√N × √N):
  Diameter = 2 × (√N - 1)
  例如：16 個節點 (4×4)，Diameter = 2 × (4-1) = 6

Torus (√N × √N):
  Diameter = √N
  例如：16 個節點 (4×4)，Diameter = 4
```

---

**3. Bisection Bandwidth (對半頻寬)**

```
定義：將圖切成兩半時，需要切斷的最少邊數

在 NoC 中：
  Bisection Bandwidth = 跨越中線的最大頻寬
  
為什麼重要？
  決定了網路的整體吞吐量（Throughput）
  如果 Bisection Bandwidth 太低，網路容易塞車
```

**範例**：

```
Ring (N 個節點):
  Bisection Bandwidth = 2 條 Link
  → 非常容易塞車

Mesh (√N × √N):
  Bisection Bandwidth = √N 條 Link
  例如：16 個節點 (4×4)，Bisection Bandwidth = 4

Torus (√N × √N):
  Bisection Bandwidth = 2√N 條 Link
  例如：16 個節點 (4×4)，Bisection Bandwidth = 8
```

---

### 1.3 圖論指標的權衡

**設計三角習題**：

```
┌─────────────┐
│   Degree    │  ← 硬體成本
│   (低越好)   │
└──────┬──────┘
       │
       │
┌──────┴──────┬──────────────┐
│  Diameter   │   Bisection  │
│  (低越好)    │   Bandwidth  │
│             │   (高越好)    │
└─────────────┴──────────────┘
```

**沒有完美的拓撲，只有適合的拓撲。**

---

## 二、經典拓撲的圖論分析

### 2.1 Ring：簡單但受限

**圖論指標**：

```
N 個節點的 Ring:

Degree:              2
Diameter:            N/2
Bisection Bandwidth: 2
```

**量化分析**：

```
假設：8 個核心，每條 Link = 64 bytes/cycle

Diameter:
  最遠距離 = 8/2 = 4 跳
  Hop Latency = 2 cycles
  Zero-load Latency = 4 × 2 = 8 cycles

Bisection Bandwidth:
  跨越中線的頻寬 = 2 × 64 bytes/cycle = 128 bytes/cycle

問題：
  如果 4 個核心同時要跨越中線存取資料
  → 每個核心只能分到 128/4 = 32 bytes/cycle
  → 頻寬不足
```

> 註：以上 Hop Latency、Bisection Bandwidth 等數字為基於代表性配置的估計值，
> 用來說明 **不同拓撲之間的量級差異**，
> 實際數值會依 CPU 世代、Link 寬度與頻率而有所不同。

**適用場景**：

```
✅ 核心數少（4-8 個）
✅ 成本敏感（消費級產品）
✅ 通訊模式簡單（大部分是本地存取）

❌ 核心數多（16+ 個）
❌ 需要高頻寬（Server, HPC）
```

---

### 2.2 Mesh：平衡的選擇

**圖論指標**：

```
√N × √N 的 Mesh:

Degree:              4 (內部), 2-3 (邊緣)
Diameter:            2 × (√N - 1)
Bisection Bandwidth: √N
```

**量化分析**：

```
假設：16 個核心 (4×4 Mesh)，每條 Link = 64 bytes/cycle

Diameter:
  最遠距離 = 2 × (4-1) = 6 跳
  Hop Latency = 2 cycles
  Zero-load Latency = 6 × 2 = 12 cycles

Bisection Bandwidth:
  跨越中線的頻寬 = 4 × 64 bytes/cycle = 256 bytes/cycle
  
比較 Ring:
  Diameter: 6 vs 8 (Mesh 更好)
  Bisection Bandwidth: 256 vs 128 (Mesh 更好)
  Degree: 4 vs 2 (Ring 更簡單)
```

**適用場景**：

```
✅ 核心數多（16-64 個）
✅ 需要高頻寬（Server, Workstation）
✅ 2D 晶片佈局（容易實現）

❌ 成本極度敏感（Mesh 的 Router 更複雜）
```

---

### 2.3 Torus：理想但難實現

**圖論指標**：

```
√N × √N 的 Torus:

Degree:              4 (所有節點)
Diameter:            √N
Bisection Bandwidth: 2√N
```

**量化分析**：

```
假設：16 個核心 (4×4 Torus)，每條 Link = 64 bytes/cycle

Diameter:
  最遠距離 = 4 跳
  Hop Latency = 2 cycles
  Zero-load Latency = 4 × 2 = 8 cycles

Bisection Bandwidth:
  跨越中線的頻寬 = 8 × 64 bytes/cycle = 512 bytes/cycle
  
比較 Mesh:
  Diameter: 4 vs 6 (Torus 更好)
  Bisection Bandwidth: 512 vs 256 (Torus 更好)
  Degree: 4 vs 4 (相同)
  
問題：
  Wrap-around Links 在 2D 晶片上很難實現
  → 長導線延遲可能抵消 Diameter 的優勢
```

---

### 2.4 拓撲比較表

**完整比較**：

| 拓撲 | Degree | Diameter | Bisection BW | 2D 可行性 | 成本 |
|------|--------|----------|--------------|-----------|------|
| Ring (N) | 2 | N/2 | 2 | ✅ 容易 | 低 |
| Mesh (√N×√N) | 4 | 2(√N-1) | √N | ✅ 容易 | 中 |
| Torus (√N×√N) | 4 | √N | 2√N | ❌ 困難 | 高 |
| Hypercube (2^k) | k | k | 2^(k-1) | ❌ 困難 | 極高 |
| Fat-Tree | 變動 | 2log₂N | N/2 | ✅ 可行 | 極高 |

> 註：上表中的 Degree、Diameter、Bisection BW 為圖論理論值，
> 2D 可行性與成本為基於業界實務的定性評估，
> 實際設計需考慮製程、佈線與功耗等因素。

**結論**：

```
Ring:
  - 最簡單，但性能受限
  - 適合小規模（4-8 核心）

Mesh:
  - 平衡的選擇
  - 適合中大規模（16-64 核心）
  - 業界主流

Torus:
  - 理論最優，但實現困難
  - 適合超級電腦（機櫃互連）

Hypercube, Fat-Tree:
  - 性能極佳，但成本極高
  - 適合特殊應用（HPC, Data Center）
```

---

## 三、進階拓撲：Hypercube 與 Fat-Tree

### 3.1 Hypercube (超立方體)（進階）

**結構**：

```
k-維 Hypercube 有 2^k 個節點

1-D Hypercube (2 個節點):
[0] -- [1]

2-D Hypercube (4 個節點):
[00] -- [01]
 |       |
[10] -- [11]

3-D Hypercube (8 個節點):
      [000] -- [001]
       |        |
      [010] -- [011]
       |        |
      [100] -- [101]
       |        |
      [110] -- [111]
```

**圖論指標**：

```
k-維 Hypercube (N = 2^k 個節點):

Degree:              k
Diameter:            k
Bisection Bandwidth: 2^(k-1)
```

**優點**：

```
✅ Diameter = log₂N（非常低）
✅ Bisection Bandwidth = N/2（非常高）
✅ 對稱性極佳（所有節點完全相同）
```

**缺點**：

```
❌ Degree = log₂N（隨著 N 增加而增加）
❌ 在 2D 平面上無法完美映射
❌ 佈線極度複雜

範例：
  64 個節點的 Hypercube (k=6)
  → 每個 Router 需要 6 個 Port
  → 在 2D 平面上，會有大量的長導線
```

**業界案例**：

```
Intel iWarp (1990s):
  - 使用 2D Mesh 模擬 Hypercube
  - 已停產

現代晶片：
  - 幾乎沒有使用 Hypercube
  - 原因：佈線成本太高
```

---

### 3.2 Fat-Tree (胖樹)（進階）

**結構**：

```
Fat-Tree 是一種分層結構：

Level 2 (Root):     [R]
                   /   \
Level 1:        [S0]   [S1]
               / | \   / | \
Level 0:    [L0][L1][L2][L3][L4][L5]

特點：
  - 越往上，頻寬越高（"胖"）
  - 每個 Switch 連接多個下層節點
```

**圖論指標**：

```
k-ary Fat-Tree (N = k^3/4 個節點):

Degree:              k
Diameter:            2log_k(N)
Bisection Bandwidth: N/2
```

**優點**：

```
✅ Bisection Bandwidth = N/2（極高）
✅ 適合 All-to-All 通訊
✅ 可以在 2D 平面上實現（分層佈局）
```

**缺點**：

```
❌ 成本極高（需要大量的 Switch）
❌ 功耗高（多層 Switch）
❌ 延遲較高（需要經過多層）

範例：
  64 個節點的 Fat-Tree
  → 需要 80 個 Switch
  → 成本是 Mesh 的 3-4 倍
```

**業界案例**：

```
Data Center 網路：
  - Google, Facebook 的 Data Center
  - 使用 Fat-Tree 或變種（Clos Network）

為什麼 Data Center 可以用 Fat-Tree？
  - 節點之間是實體網路線，不是晶片內導線
  - 可以使用高速 Ethernet (100 Gbps)
  - 成本可以分攤到數千台伺服器

晶片內 NoC：
  - 幾乎沒有使用 Fat-Tree
  - 原因：成本太高，功耗太大
```

---

## 四、真實案例：Intel 和 ARM 的選擇

### 4.1 Intel 的演進：從 Ring 到 Mesh

**2011-2016：Ring 時代**

```
Intel Sandy Bridge (2011):
  4 核心 + iGPU + System Agent
  使用 Bidirectional Ring

圖論分析：
  Degree: 2
  Diameter: 4/2 = 2
  Bisection Bandwidth: 2

為什麼選 Ring？
  - 核心數少（4-8 個）
  - Diameter = 2-4，延遲可接受
  - 成本低，適合消費級產品
```

**2017+：Mesh 時代**

```
Intel Skylake-SP (2017):
  28 核心，使用 2D Mesh (7×4)

圖論分析：
  Degree: 4-5
  Diameter: (7-1) + (4-1) = 9
  Bisection Bandwidth: 7 (垂直切) 或 4 (水平切)

為什麼改用 Mesh？
  - 核心數增加到 28-56 個
  - Ring 的 Diameter = 14-28，延遲太高
  - Mesh 的 Bisection Bandwidth = 7，遠高於 Ring 的 2
```

**性能對比**：

```
假設：28 核心，每條 Link = 64 bytes/cycle

Ring:
  Diameter = 14
  Zero-load Latency = 14 × 2 = 28 cycles
  Bisection Bandwidth = 2 × 64 = 128 bytes/cycle

Mesh (7×4):
  Diameter = 9
  Zero-load Latency = 9 × 2 = 18 cycles
  Bisection Bandwidth = 7 × 64 = 448 bytes/cycle

性能提升：
  延遲：28 → 18 cycles (36% 改善)
  頻寬：128 → 448 bytes/cycle (3.5 倍提升)
```

---

### 4.2 ARM CMN-700：靈活的 Mesh

**架構**：

```
ARM Coherent Mesh Network (CMN-700):
  最多 8×8 Mesh = 64 個 Cross Point (XP)
  每個 XP 可以連接 1-4 個 Device

圖論分析：
  Degree: 4-5 (4 個鄰居 + 1-4 個本地 Device)
  Diameter: 2 × (8-1) = 14
  Bisection Bandwidth: 8
```

**靈活性**：

```
配置 1：手機 SoC
  4×4 Mesh = 16 個 XP
  連接：4 個 Cortex-A78 + 4 個 Cortex-A55 + GPU + NPU

配置 2：Server 晶片
  8×8 Mesh = 64 個 XP
  連接：64 個 Neoverse V2 核心
```

**為什麼選 Mesh？**

```
1. 靈活性：
   - 可以連接異質運算單元（CPU, GPU, NPU）
   - 可以動態調整配置

2. 可擴展性：
   - 從 4×4 到 8×8，只需要增加 XP
   - 不需要重新設計整個網路

3. 2D 可行性：
   - Mesh 在 2D 平面上容易實現
   - 佈線簡單，Timing 容易收斂
```

---

### 4.3 AMD Infinity Fabric：Mesh + Chiplet

**架構**：

```
AMD EPYC (Rome, 7nm):
  8 個 Chiplet + 1 個 I/O Die
  每個 Chiplet：8 核心 + L3 Cache

Infinity Fabric 拓撲：
  Chiplet 內部：Mesh (2×4)
  Chiplet 之間：All-to-All (透過 I/O Die)
```

**圖論分析**：

```
Chiplet 內部 (8 核心, 2×4 Mesh):
  Degree: 4
  Diameter: (2-1) + (4-1) = 4
  Bisection Bandwidth: 2

Chiplet 之間 (8 個 Chiplet):
  Degree: 8 (每個 Chiplet 連接到 I/O Die)
  Diameter: 2 (Chiplet → I/O Die → Chiplet)
  Bisection Bandwidth: 8 (透過 I/O Die)
```

**設計權衡**：

```
優點：
  ✅ Chiplet 設計提高良率
  ✅ 可以混搭不同製程（7nm Chiplet + 14nm I/O Die）
  ✅ 成本低

缺點：
  ❌ 跨 Chiplet 延遲較高（~80 ns vs ~40 ns）
  ❌ I/O Die 可能成為瓶頸
```

---

## 五、圖論在 NoC 設計中的應用

### 5.1 如何選擇拓撲？

**決策樹**：

```
1. 核心數量？
   ├─ 4-8 個 → Ring
   ├─ 16-64 個 → Mesh
   └─ 64+ 個 → Mesh + Chiplet 或 3D NoC

2. 通訊模式？
   ├─ 大部分本地存取 → Ring 或 Mesh
   ├─ 大量跨核心通訊 → Mesh 或 Torus
   └─ All-to-All 通訊 → Fat-Tree (Data Center)

3. 成本限制？
   ├─ 低成本 → Ring
   ├─ 中等成本 → Mesh
   └─ 高成本 → Torus 或 Fat-Tree

4. 2D 佈局限制？
   ├─ 必須 2D → Ring 或 Mesh
   └─ 可以 3D → Torus 或 Hypercube
```

---

### 5.2 圖論指標的實際意義

**Degree → 硬體成本**

```
Degree = 2 (Ring):
  Router 面積：~0.1 mm² (7nm)
  功耗：~10 mW

Degree = 4 (Mesh):
  Router 面積：~0.3 mm² (7nm)
  功耗：~30 mW

Degree = 6 (Hypercube):
  Router 面積：~0.6 mm² (7nm)
  功耗：~60 mW

結論：
  Degree 每增加 1，面積和功耗約增加 50%
```

---

**Diameter → 延遲**

```
假設：Hop Latency = 2 cycles, 頻率 = 2.5 GHz

Diameter = 4:
  Zero-load Latency = 4 × 2 = 8 cycles = 3.2 ns

Diameter = 9:
  Zero-load Latency = 9 × 2 = 18 cycles = 7.2 ns

Diameter = 14:
  Zero-load Latency = 14 × 2 = 28 cycles = 11.2 ns

結論：
  Diameter 每增加 1，延遲增加 0.8 ns
  對於延遲敏感的應用（如 Cache Coherency），這很重要
```

---

**Bisection Bandwidth → 吞吐量**

```
假設：每條 Link = 64 bytes/cycle @ 2.5 GHz = 160 GB/s

Bisection Bandwidth = 2 (Ring):
  跨越中線的總頻寬 = 2 × 160 = 320 GB/s

Bisection Bandwidth = 7 (Mesh 7×4):
  跨越中線的總頻寬 = 7 × 160 = 1,120 GB/s

Bisection Bandwidth = 8 (Torus 4×4):
  跨越中線的總頻寬 = 8 × 160 = 1,280 GB/s

結論：
  Bisection Bandwidth 決定了網路的整體吞吐量
  對於多執行緒應用，這是關鍵指標
```

---

## 六、圖論的進階應用：路由與死鎖

### 6.1 圖論與路由演算法

**路由演算法本質上是圖的遍歷**：

```
問題：
  從節點 A 到節點 B，有多條路徑
  如何選擇最佳路徑？

圖論解法：
  1. 最短路徑：Dijkstra 演算法
  2. 確定性路由：XY Routing (Dimension-Order)
  3. 自適應路由：根據擁塞情況動態選擇
```

**XY Routing 的圖論解釋**：

```
XY Routing:
  1. 先沿著 X 軸移動，直到 X 座標相符
  2. 再沿著 Y 軸移動，直到 Y 座標相符

圖論視角：
  - 這是一種 Dimension-Order Routing
  - 保證路徑唯一（Deterministic）
  - 保證無環（Cycle-free）
```

---

### 6.2 圖論與死鎖避免

**死鎖的圖論定義**：

```
死鎖 = 圖中存在有向環 (Directed Cycle)

範例：
  封包 A：R0 → R1 → R2
  封包 B：R2 → R1 → R0

  如果 A 在 R1 等待 R2 的 Buffer
  同時 B 在 R1 等待 R0 的 Buffer
  → 形成環 → 死鎖
```

**避免死鎖的圖論方法**：

```
方法 1：確保路由圖無環 (Acyclic)
  - XY Routing 保證無環
  - 證明：X 軸方向單調遞增/遞減，Y 軸方向單調遞增/遞減
  - 不可能形成環

方法 2：Virtual Channels (VC)
  - 將物理網路分成多個虛擬網路
  - 每個虛擬網路無環
  - 即使物理網路有環，虛擬網路也無環
```

**我們會在下一篇文章深入探討路由與死鎖！**

---

## 七、總結

圖論是 NoC 設計的數學基礎：

1. **圖論指標**：Degree, Diameter, Bisection Bandwidth
2. **經典拓撲**：Ring（簡單）、Mesh（平衡）、Torus（理想但難實現）
3. **進階拓撲**：Hypercube（對稱）、Fat-Tree（高頻寬）
4. **業界選擇**：Intel Mesh、ARM CMN、AMD Infinity Fabric
5. **設計權衡**：成本 vs 性能 vs 可行性
6. **進階應用**：路由演算法、死鎖避免

理解圖論，可以幫助我們：
- 量化分析不同拓撲的優劣
- 理解業界的設計選擇
- 設計自己的 NoC 拓撲
- 為未來的異質運算做準備

---

## 下一篇預告

在下一篇文章中，我們會深入探討：

**NoC 路由演算法與死鎖避免**
- XY Routing 的硬體實作
- Virtual Channels 的原理
- Protocol Deadlock 的解決方案
- Turn Model (West-First, North-Last)
- Credit-Based Flow Control

敬請期待！

---

## 參考資料

**公開文檔**：
- *The Art of Computer Programming, Volume 4* - Donald Knuth
- *Principles and Practices of Interconnection Networks* - Dally & Towles
- *Computer Architecture: A Quantitative Approach* (6th Edition) - Hennessy & Patterson
- Intel Xeon Scalable Processor Architecture Specification
- ARM CoreLink CMN-700 Technical Reference Manual

**個人筆記**：
- Network-on-Chip Technical Notes (基於公開領域知識整理)


---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: https://github.com/djiangtw/tech-column-public
