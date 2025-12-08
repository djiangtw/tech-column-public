# Network-on-Chip 入門：從 Bus 到 Network 的演進

**作者**: Danny Jiang  
**日期**: 2025-12-07  

---

## 前言：晶片內部的微型網際網路

2015 年，我第一次看到 Intel Xeon Phi 的架構圖時，被震撼了。

那是一顆擁有 **72 個核心**的處理器，每個核心都有自己的 L1 和 L2 Cache。我當時的第一個反應是：

**「這些核心要怎麼互相溝通？」**

如果用傳統的 Bus（匯流排），72 個核心搶一條線，不就塞爆了嗎？

後來我才知道，Intel 在這顆晶片裡面塞了一個 **2D Mesh Network**，就像一個微型的網際網路，每個核心都是一個「節點」，資料被打包成「封包」，在網路中一站一站地傳遞。

這就是 **Network-on-Chip (NoC)**。

---

## 一、為什麼需要 NoC？

### 1.1 Bus 的黃金年代

在多核心處理器出現之前，CPU 內部的通訊非常簡單：

```
┌─────────┐     ┌─────────┐     ┌─────────┐
│  CPU 0  │     │  Cache  │     │  Memory │
│         │     │         │     │ Ctrl    │
└────┬────┘     └────┬────┘     └────┬────┘
     │               │               │
     └───────────────┴───────────────┘
              Shared Bus
```

**Bus 的特性**：
- **共享媒介**：所有人共用一條線
- **廣播通訊**：一個人說話，所有人都聽到
- **仲裁機制**：同一時間只有一個人可以使用

**類比**：就像一個走廊，同一時間只有一個人可以大喊大叫傳遞訊息。

---

### 1.2 Bus 的瓶頸

當核心數量增加時，Bus 開始出現問題：

**問題 1：頻寬瓶頸**

```
4 核心：每個核心平均分到 1/4 的頻寬
8 核心：每個核心平均分到 1/8 的頻寬
16 核心：每個核心平均分到 1/16 的頻寬
...
```

**問題 2：延遲增加**

```
Bus 的延遲 = 仲裁延遲 + 傳輸延遲 + 等待延遲

當核心數增加：
  仲裁延遲 ↑（更多人競爭）
  等待延遲 ↑（排隊時間變長）
```

**問題 3：功耗爆炸**

```
Bus 是廣播通訊：
  一個人說話，所有人都要聽
  → 所有連接到 Bus 的電路都要充放電
  → 功耗 = O(N)，N 是連接的節點數
```

---

### 1.3 真實案例：Intel 的困境

**Intel Pentium 4 時代（2000-2006）**：

```
Front-Side Bus (FSB):
  頻寬：6.4 GB/s (800 MHz × 64-bit)
  核心數：1-2 個
  
問題：當 Intel 想做 4 核心時，FSB 成為瓶頸
  4 個核心搶 6.4 GB/s
  → 每個核心只有 1.6 GB/s
  → 性能無法線性擴展
```

**Intel 的解決方案**：

```
2006 年：Intel Core 2 Quad
  → 使用兩個 Dual-Core Die
  → 每個 Die 有自己的 FSB
  → 跨 Die 通訊需要經過 Memory Controller
  → 延遲高，效率低

2008 年：Intel Nehalem (Core i7)
  → 引入 QuickPath Interconnect (QPI)
  → 點對點連接，不再是共享 Bus
  → 這是 NoC 的雛形
```

> 註：以上 FSB 頻寬與核心數為 Intel Pentium 4 / Core 2 時代的代表性配置，
> 用來說明 Bus 架構在多核心擴展時的瓶頸，
> 實際數字會依具體 CPU 型號而異。

---

## 二、NoC 的核心概念

### 2.1 從 Bus 到 Network

**Bus 的思維**：
- 共享媒介
- 廣播通訊
- 集中式仲裁

**Network 的思維**：
- 點對點連接
- 封包交換
- 分散式路由

**類比**：

```
Bus = 走廊
  → 同一時間只有一個人可以大喊
  → 所有人都聽到

Network = 郵局
  → 每個人寫信（封包）
  → 郵差（Router）負責轉送
  → 多個人可以同時寄信
```

---

### 2.2 NoC 的基本組成

**1. Node (節點)**：

```
Node 可以是：
  - CPU Core
  - Cache Slice
  - Memory Controller
  - GPU
  - Accelerator (AI, Video, etc.)
```

**2. Router (路由器)**：

```
Router 的功能：
  - 接收封包
  - 查看目的地
  - 決定往哪個方向轉送
  - 管理 Buffer（避免封包遺失）
```

**3. Link (連結)**：

```
Link 是兩個 Router 之間的實體連線：
  - 寬度：通常 64-512 bits
  - 頻率：通常與 CPU 頻率相同或稍低
  - 延遲：1-2 cycles per hop
```

**4. Packet (封包)**：

```
封包結構：
  ┌──────────┬──────────┬──────────┐
  │  Header  │  Payload │  Tail    │
  └──────────┴──────────┴──────────┘

Header:
  - Source Address (來源)
  - Destination Address (目的地)
  - Packet Type (Request/Response/Data)
  - Virtual Channel ID

Payload:
  - 實際資料（例如 Cache Line, 64 bytes）

Tail:
  - CRC (錯誤檢查)
```

---

### 2.3 NoC 的優勢

**優勢 1：可擴展性 (Scalability)**

```
Bus:
  頻寬 = 固定
  延遲 = O(N)（N 是節點數）

NoC:
  頻寬 = O(N)（多條路徑可以同時傳輸）
  延遲 = O(√N)（Mesh）或 O(log N)（Tree）
```

**優勢 2：並行性 (Parallelism)**

```
Bus:
  同一時間只有一個傳輸

NoC:
  多個傳輸可以同時進行
  例如：Core 0 → Cache 0 的同時，Core 63 → Cache 63
```

**優勢 3：功耗效率**

```
Bus:
  廣播通訊，所有節點都要充放電
  功耗 = O(N)

NoC:
  點對點通訊，只有路徑上的 Router 需要充放電
  功耗 = O(Hops)
```

---

## 三、NoC 的拓撲結構

### 3.1 Ring (環狀拓撲)

**結構**：

```
[N0] -- [N1] -- [N2] -- [N3]
 |                       |
 └───────────────────────┘
```

**特性**：

```
Degree (度數)：2（每個節點連接 2 個鄰居）
Diameter (直徑)：N/2（最遠距離）
Bisection Bandwidth (對半頻寬)：2 條 Link
```

**優點**：
- ✅ 實作簡單
- ✅ 佈線容易
- ✅ 成本低

**缺點**：
- ❌ 延遲高（O(N)）
- ❌ 頻寬低（對半頻寬只有 2 條 Link）

**業界案例**：

```
Intel Sandy Bridge (2011):
  4 核心 + iGPU + System Agent
  使用 Bidirectional Ring
  
為什麼選 Ring？
  - 核心數少（4-8 個）
  - 延遲可接受（最多 4 跳）
  - 成本低，適合消費級產品
```

---

### 3.2 Mesh (網狀拓撲)

**結構**：

```
[N0] -- [N1] -- [N2] -- [N3]
 |       |       |       |
[N4] -- [N5] -- [N6] -- [N7]
 |       |       |       |
[N8] -- [N9] -- [N10]-- [N11]
 |       |       |       |
[N12]-- [N13]-- [N14]-- [N15]
```

**特性**：

```
Degree (度數)：4（內部節點）, 2-3（邊緣節點）
Diameter (直徑)：2 × (√N - 1)
Bisection Bandwidth (對半頻寬)：√N 條 Link
```

**優點**：
- ✅ 延遲適中（O(√N)）
- ✅ 頻寬高（對半頻寬 = √N）
- ✅ 佈線可行（2D 平面）

**缺點**：
- ❌ 邊緣節點的度數不一致
- ❌ Router 複雜度較高（4-5 個 Port）

**業界案例**：

```
Intel Xeon Scalable (Skylake-SP, 2017):
  28 核心，使用 2D Mesh
  
為什麼選 Mesh？
  - 核心數多（28-56 個）
  - 需要高頻寬（多個核心同時存取不同 Cache Slice）
  - 延遲可接受（最多 10 跳）
```

---

### 3.3 Torus (環面拓撲)

**結構**：

```
Torus = Mesh + Wrap-around Links

[N0] -- [N1] -- [N2] -- [N3]
 |       |       |       |  |
[N4] -- [N5] -- [N6] -- [N7]
 |       |       |       |  |
[N8] -- [N9] -- [N10]-- [N11]
 |       |       |       |  |
[N12]-- [N13]-- [N14]-- [N15]
 |       |       |       |  |
 └───────┴───────┴───────┴──┘
```

**特性**：

```
Degree (度數)：4（所有節點都一致）
Diameter (直徑)：√N（比 Mesh 少一半）
Bisection Bandwidth (對半頻寬)：2√N 條 Link（比 Mesh 多一倍）
```

**優點**：
- ✅ 延遲低（直徑減半）
- ✅ 頻寬高（對半頻寬加倍）
- ✅ 所有節點的度數一致（對稱性好）

**缺點**：
- ❌ **Wrap-around Links 在 2D 晶片上很難實現**
- ❌ 長導線 (Long Wires) 導致 Timing Violation
- ❌ 佈線擁塞 (Routing Congestion)

**為什麼 Torus 在晶片內很少見？**

```
問題：Wrap-around Links

在 Torus 中：
  最左邊的節點 [N0] 要連到最右邊的節點 [N3]
  → 需要一條橫跨整個晶片的導線
  → 長度可能是 10-20 mm（在大晶片上）

在現代製程（7nm, 5nm）：
  導線延遲 = R × C（電阻 × 電容）
  長導線的延遲可能是 10-20 cycles
  → 完全抵消了 Torus 的延遲優勢

解決方案：
  1. 使用 Repeater（中繼器）插入 Buffer
     → 增加功耗和面積
  2. 降低頻率
     → 犧牲性能
  3. 放棄 Torus，改用 Mesh
     → 業界的選擇
```

**業界案例**：

```
超級電腦（機櫃互連）：
  - IBM Blue Gene/Q：5D Torus
  - Cray XC40：Dragonfly (變種 Torus)

為什麼超級電腦可以用 Torus？
  - 節點之間是實體電纜，不是晶片內導線
  - 可以使用光纖，延遲可控
  - 對稱性好，適合科學計算的通訊模式
```

---

## 四、NoC 與 Cache Coherency 的關聯

### 4.1 MESI 訊息在 NoC 上的傳輸

如果你讀過我的 Cache Coherency 文章，你會知道 MESI 協議需要在多個核心之間傳遞訊息：

**MESI 的訊息類型**：

```
1. Read Request (讀取請求)
   Core 0 → Cache Slice 5: "我要讀取地址 0x1000"

2. Invalidate (失效通知)
   Core 0 → All Cores: "我要寫入 0x1000，你們的副本都失效了"

3. Data Response (資料回應)
   Cache Slice 5 → Core 0: "這是 0x1000 的資料"

4. Acknowledgement (確認)
   Core 1, 2, 3 → Core 0: "我已經失效了我的副本"
```

**這些訊息都是透過 NoC 傳輸的封包！**

---

### 4.2 NoC 的設計必須配合 Coherency Protocol

**問題**：如果 NoC 設計不當，會導致 **Protocol Deadlock**。

**範例**：

```
場景：
  Core 0 發出 Read Request → Cache Slice 5
  Cache Slice 5 需要發出 Invalidate → All Cores
  但 NoC 的 Buffer 已滿，Invalidate 無法送出
  → Core 0 一直等待 Data Response
  → Cache Slice 5 一直等待 Buffer 空位
  → 死鎖！
```

**解決方案：Virtual Channels (虛擬通道)**

```
將 NoC 的 Buffer 分成多個獨立的 Virtual Channels：

VC0: Request Channel (Read, Write)
VC1: Response Channel (Data, Ack)
VC2: Snoop Channel (Invalidate, Forward)

即使 VC0 (Request) 塞滿了，VC1 (Response) 依然暢通
→ 避免死鎖
```

**這是 NoC 設計中最精彩的部分，我們會在後續文章深入探討！**

---

## 五、真實案例分析

### 5.1 Intel Xeon Scalable (Skylake-SP)

**架構**：

```
28 核心，2D Mesh NoC

每個 Tile 包含：
  - 1 個 CPU Core
  - 1 個 L2 Cache (1 MB)
  - 1 個 L3 Cache Slice (1.375 MB)
  - 1 個 Router (5 個 Port)

Mesh 配置：
  7 × 4 = 28 Tiles

Router 的 5 個 Port：
  - North, South, East, West (連接鄰居)
  - Local (連接本地 Core)
```

**性能數據**：

```
Zero-load Latency (空載延遲):
  最近鄰居：~10 cycles
  對角線：~40 cycles

Bandwidth (頻寬):
  每條 Link：32 bytes/cycle @ 2.5 GHz = 80 GB/s
  總頻寬：28 × 4 × 80 GB/s = 8,960 GB/s (理論值)
```

> 註：以上 Zero-load Latency 與 Bandwidth 數字為基於 Intel 公開文件的代表性估計，
> 用來說明 Mesh 拓撲的性能特性，
> 實際數字會依 Skylake-SP 具體型號與工作負載而異。

**設計權衡**：

```
為什麼選 Mesh 而不是 Ring？
  - 28 核心用 Ring，直徑 = 14，延遲太高
  - Mesh 的直徑 = 6 + 3 = 9，延遲可接受
  - Mesh 的對半頻寬 = 7 條 Link，遠高於 Ring 的 2 條
```

---

### 5.2 ARM CoreLink CMN-700

**架構**：

```
ARM 的 Coherent Mesh Network (CMN)

Cross Point (XP):
  - 類似 Router
  - 可以連接 CPU Cluster, GPU, Accelerator
  - 支援 AMBA CHI (Coherent Hub Interface)

配置範例：
  8 × 8 Mesh = 64 個 XP
  每個 XP 可以連接 1-4 個 Device
  → 最多支援 256 個 Device
```

**特色**：

```
1. 靈活性：
   - 可以連接異質運算單元（CPU, GPU, NPU）
   - 支援大小核設計（Cortex-A78 + Cortex-A55）

2. QoS (Quality of Service):
   - 支援優先級調度
   - 保證關鍵任務的延遲

3. 功耗優化：
   - 支援 Clock Gating（閒置的 XP 可以關閉）
   - 支援 Power Domain（不同區域可以獨立供電）
```

---

### 5.3 AMD Infinity Fabric

**架構**：

```
AMD EPYC (Rome, 7nm):
  64 核心，分成 8 個 Chiplet
  每個 Chiplet：8 核心 + L3 Cache

Infinity Fabric:
  - 連接 8 個 Chiplet 和 1 個 I/O Die
  - 使用 2D Mesh 拓撲
  - 支援 Coherent 和 Non-Coherent 傳輸
```

**創新點**：

```
Chiplet 設計：
  - 每個 Chiplet 是獨立的小晶片
  - 透過 Infinity Fabric 連接
  - 優點：良率高，成本低
  - 缺點：跨 Chiplet 延遲較高

性能數據：
  Chiplet 內部延遲：~40 ns
  跨 Chiplet 延遲：~80 ns
  → 2 倍差距
```

---

## 六、NoC 的未來趨勢

### 6.1 3D NoC

**概念**：

```
傳統 NoC：2D 平面
3D NoC：垂直堆疊多層晶片

優勢：
  - 縮短平均距離（增加一個 Z 軸）
  - 提高頻寬（垂直連接可以非常密集）

挑戰：
  - 散熱問題（中間層的熱量難以散出）
  - 製程成本（TSV - Through-Silicon Via）
```

**業界案例**：

```
AMD 3D V-Cache (Zen 3):
  - 在 CPU Die 上方堆疊 64 MB L3 Cache
  - 使用 TSV 連接
  - 性能提升：遊戲性能 +15%
```

---

### 6.2 光學 NoC (Optical NoC)

**概念**：

```
傳統 NoC：電訊號
光學 NoC：光訊號

優勢：
  - 頻寬極高（光的頻率遠高於電）
  - 功耗低（光不需要充放電）
  - 延遲低（光速 > 電訊號速度）

挑戰：
  - 光電轉換的功耗和延遲
  - 製程整合（光學元件 + CMOS）
```

**研究現狀**：

```
MIT, UC Berkeley, IBM 都有研究
目前還在實驗室階段
預計 2030 年後才可能商用化
```

---

### 6.3 AI 加速器的 NoC

**趨勢**：

```
AI 晶片（如 Google TPU, NVIDIA A100）：
  - 數千個小核心（Tensor Core, Matrix Engine）
  - 需要極高的頻寬
  - 通訊模式與 CPU 不同（更多的 All-to-All）

NoC 設計的挑戰：
  - 如何支援 All-to-All 通訊？
  - 如何避免 Hot Spot（某些 Router 過載）？
  - 如何動態調整路由以平衡負載？
```

---

## 七、總結

Network-on-Chip 是現代多核心處理器的核心技術：

1. **為什麼需要 NoC**：Bus 無法擴展到多核心
2. **NoC 的核心概念**：封包交換、分散式路由
3. **拓撲結構**：Ring（簡單）、Mesh（平衡）、Torus（理想但難實現）
4. **與 Cache Coherency 的關聯**：MESI 訊息透過 NoC 傳輸
5. **真實案例**：Intel Mesh、ARM CMN、AMD Infinity Fabric
6. **未來趨勢**：3D NoC、光學 NoC、AI 加速器

理解 NoC，可以幫助我們：
- 理解為什麼跨核心通訊有延遲
- 設計更高效的多執行緒程式
- 理解 NUMA 的本質
- 為未來的異質運算做準備

---

## 下一篇預告

在下一篇文章中，我們會深入探討：

**NoC 拓撲結構的圖論分析**
- 如何用圖論指標（Degree, Diameter, Bisection Bandwidth）量化分析拓撲
- 為什麼 Intel 和 ARM 選擇 Mesh？
- Hypercube, Fat-Tree, Dragonfly 等進階拓撲
- 如何在 2D 晶片上映射 3D 拓撲？

敬請期待！

---

## 參考資料

**公開文檔**：
- *Principles and Practices of Interconnection Networks* - Dally & Towles
- Intel Xeon Scalable Processor Architecture Specification
- ARM CoreLink CMN-700 Technical Reference Manual
- *Computer Architecture: A Quantitative Approach* (6th Edition) - Hennessy & Patterson

**個人筆記**：
- Network-on-Chip Technical Notes (基於公開領域知識整理)


---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: https://github.com/djiangtw/tech-column-public
