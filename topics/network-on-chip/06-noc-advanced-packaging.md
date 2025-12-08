# NoC 與先進封裝：突破物理邊界

**作者**: Danny Jiang  
**日期**: 2025-12-07  

---

## 前言：摩爾定律的終結與重生

2019 年，NVIDIA 發布了 A100 GPU，採用 7nm 製程，擁有 542 億個電晶體。當時我在想：下一代會是什麼樣子？5nm？3nm？電晶體數量能翻倍嗎？

2022 年，NVIDIA 發布了 H100 GPU，採用 4nm 製程，擁有 800 億個電晶體。但更驚人的是它的封裝技術：**CoWoS-S**，整合了 5 個 HBM3 記憶體堆疊，總頻寬達到 **3 TB/s**。

這讓我意識到：**NoC 的未來不在於把單晶片做得更大，而在於如何突破晶片的物理邊界。**

在這篇文章中，我們將探討：
- NoC 面臨的物理瓶頸
- Chiplet 如何延伸 NoC 的邊界
- UCIe 標準的作用
- CoWoS 如何實現高頻寬互連
- CPO 光互連的未來
- 真實案例：NVIDIA H100, AMD MI300X, Intel Ponte Vecchio

---

## 一、NoC 的物理瓶頸

### 1.1 Wire Delay 的惡化

隨著製程節點縮小，電晶體變快了，但**導線延遲（Wire Delay）卻變慢了**。

**為什麼？**

```
製程縮小：
  電晶體尺寸：↓ 0.7x
  導線寬度：↓ 0.7x
  導線電阻：↑ 2x (更細 → 電阻更大)
  導線電容：→ 1x (間距也縮小)
  
  RC Delay = R × C
           ↑ 2x
  
  → Wire Delay 增加 2x！
```

**實際數據**：

```
180nm 製程：
  Gate Delay: 100 ps
  Wire Delay: 50 ps
  → Wire Delay 佔 33%

7nm 製程：
  Gate Delay: 10 ps
  Wire Delay: 100 ps
  → Wire Delay 佔 91%！
```

**對 NoC 的影響**：

```
在 7nm 製程，20mm 的 NoC Link：
  延遲：~50 cycles @ 2 GHz
  功耗：~10 mW

  → 長距離通訊變得非常昂貴
```

> 註：以上 Wire Delay 與 Gate Delay 數字為基於業界公開文獻的代表性估計值，
> 用來說明製程縮小對導線延遲的影響，
> 實際數字會依製程節點、金屬層與佈線密度而異。

---

### 1.2 Reticle Limit：晶片面積的天花板

**光刻機的限制**：

```
ASML EUV 光刻機：
  最大曝光面積 (Reticle Limit)：
    26mm × 33mm = 858 mm²
  
  → 單晶片不能超過這個尺寸
```

**實際案例**：

```
NVIDIA A100 (7nm):
  Die Size: 826 mm²
  → 已經接近極限

NVIDIA H100 (4nm):
  Die Size: 814 mm²
  → 仍然接近極限
  
  → 無法再做更大的單晶片
```

---

### 1.3 良率問題

**良率與晶片面積的關係**：

```
良率公式 (簡化版):
  Yield = (1 - Defect_Density × Area)^N
  
  假設：
    Defect_Density = 0.1 defects/cm²
    
  100 mm² 晶片：
    Yield ≈ 90%
  
  800 mm² 晶片：
    Yield ≈ 45%
  
  → 大晶片良率低，成本高
```

> 註：以上良率公式為簡化版本，用來說明晶片面積與良率的反比關係，
> 實際良率會依製程成熟度、缺陷密度與設計複雜度而有所不同。

---

## 二、Chiplet：NoC 的邊界延伸

### 2.1 什麼是 Chiplet？

**Chiplet** 是一種設計方法論：**將大晶片拆成多個小晶片，然後用先進封裝技術連接起來。**

**比喻**：

```
傳統單晶片 (Monolithic)：
  像是一棟超大的獨棟別墅
  - 優點：內部通訊快
  - 缺點：建造困難、成本高、良率低

Chiplet：
  像是一個社區，由多棟小房子組成
  - 優點：建造容易、成本低、良率高
  - 缺點：房子之間通訊較慢
```

---

### 2.2 Chiplet 對 NoC 的影響

**NoC 的階層化**：

```
單晶片時代：
  NoC 的邊界 = 晶片的邊緣
  所有 Router 都在同一個 Die 上

Chiplet 時代：
  On-Die NoC：Chiplet 內部的 NoC (例如 Mesh)
  On-Package NoC：Chiplet 之間的 NoC (例如 Ring, Star)

  → NoC 變成階層化 (Hierarchical)
```

**延遲的增加**：

```
晶片內部 (On-Die):
  Link 延遲：1-2 cycles
  Router 延遲：4 cycles

  總延遲：5-6 cycles = 2-3 ns @ 2 GHz

跨 Chiplet (Die-to-Die):
  Link 延遲：10-20 cycles (經過 D2D 介面)
  Router 延遲：4 cycles
  SerDes 延遲：5-10 cycles

  總延遲：19-34 cycles = 10-17 ns @ 2 GHz

  → 延遲增加 3-5x
```

**頻寬的限制**：

```
晶片內部：
  Link Width：128-512 bits
  Frequency：2 GHz
  Bandwidth：256-1024 GB/s (per link)

跨 Chiplet (D2D 介面):
  Link Width：16-32 bits (受限於 Bump 數量)
  Frequency：2 GHz
  Bandwidth：32-64 GB/s (per link)

  → 頻寬降低 8-16x
```

---

### 2.3 案例：AMD EPYC (Zen 2)

**架構**：

```
AMD EPYC 7742 (64 核心):
  8 個 Chiplet (CCD - Core Complex Die)
  每個 Chiplet：8 核心 + L3 Cache
  1 個 I/O Die (cIOD - Central I/O Die)

拓撲：
  Chiplet 內部：Mesh (2×4)
  Chiplet 之間：透過 I/O Die (Star 拓撲)
```

**Infinity Fabric 的設計**：

```
Chiplet 內部 (On-Die NoC):
  拓撲：2×4 Mesh
  Link Width：256 bits
  Bandwidth：~500 GB/s (per link)
  延遲：~5 ns

Chiplet 到 I/O Die (D2D 介面):
  Link Width：32 bits
  Bandwidth：~50 GB/s (per link)
  延遲：~40 ns

Chiplet 之間 (經過 I/O Die):
  延遲：~80 ns (40 + 40)

  → 跨 Chiplet 延遲是內部的 16x
```

> 註：以上 Chiplet 內部與跨 Chiplet 延遲數字為基於 AMD 公開文件與代表性測試的估計值，
> 用來說明 Chiplet 架構的延遲特性，
> 實際數字會依 EPYC 型號與工作負載而異。

**為什麼這樣設計？**

```
優點：
  1. 良率高：
     單個 Chiplet：~80 mm²
     良率：~90%

     vs 單晶片 640 mm²：
     良率：~20%

  2. 成本低：
     8 個小晶片 + 1 個 I/O Die
     vs 1 個超大晶片

     成本降低 ~40%

  3. 靈活性：
     可以混合不同製程
     CCD：7nm (高效能)
     I/O Die：12nm (成本低)

缺點：
  跨 Chiplet 延遲高
  → 需要軟體優化 (NUMA-aware)
```

---

## 三、UCIe：Chiplet 的統一語言

### 3.1 為什麼需要 UCIe？

**問題**：

```
每個廠商都有自己的 D2D 介面：
  Intel：EMIB, Foveros
  AMD：Infinity Fabric
  NVIDIA：NVLink

  → 不同廠商的 Chiplet 無法互連
  → 生態系統封閉
```

**UCIe 的目標**：

```
Universal Chiplet Interconnect Express (UCIe)
  → Chiplet 之間的「USB 標準」

  讓不同廠商的 Chiplet 可以互連：
    Intel CPU Chiplet
    + NVIDIA GPU Chiplet
    + AMD AI Accelerator Chiplet

    → 在同一個封裝裡無縫對話
```

---

### 3.2 UCIe 與 NoC 的關係

**UCIe 的定位**：

```
UCIe 是 D2D (Die-to-Die) 介面標準
  → 連接不同 Chiplet 的 NoC

類比：
  On-Die NoC = 城市內部的道路
  UCIe = 城市之間的高速公路

  → UCIe 是 NoC 的延伸
```

**UCIe 的規格**：

```
Physical Layer:
  Standard Package (Organic):
    Pitch: 25 μm
    Bandwidth: 2-4 GB/s per pin

  Advanced Package (CoWoS):
    Pitch: 10 μm
    Bandwidth: 8-16 GB/s per pin

Protocol Layer:
  支援多種協議：
    PCIe
    CXL (Compute Express Link)
    Streaming

  → 靈活性高
```

---

### 3.3 業界採用

**UCIe 聯盟成員**：

```
創始成員 (2022):
  Intel, AMD, ARM, TSMC, Samsung
  Google, Meta, Microsoft
  Qualcomm, ASE, Amkor

  → 幾乎所有主要廠商都加入
```

**實際產品**：

```
Intel Meteor Lake (2023):
  4 個 Tiles (Chiplet)
  使用 Foveros 3D + UCIe

  → Intel 第一個採用 UCIe 的產品
```

---

## 四、CoWoS：高頻寬的橋樑

### 4.1 什麼是 CoWoS？

**CoWoS (Chip-on-Wafer-on-Substrate)** 是台積電的 **2.5D 封裝技術**。

**結構**：

```
傳統封裝：
  Chip → Substrate (PCB) → Package

  連接方式：Wire Bonding 或 Flip-Chip
  Pitch：~100 μm
  Bandwidth：~10 GB/s per mm

CoWoS 封裝：
  Chip → Silicon Interposer → Substrate → Package

  連接方式：Micro-Bump
  Pitch：~40 μm (CoWoS-S) 或 ~10 μm (CoWoS-L)
  Bandwidth：~100 GB/s per mm

  → 頻寬提升 10x
```

**Silicon Interposer**：

```
什麼是 Interposer？
  一層薄的矽晶片，上面有密集的金屬導線

  作用：
    連接多個 Chiplet
    提供超高密度的互連

  優點：
    Pitch 小 (10-40 μm)
    導線密度高 (10,000+ 條/mm)
    延遲低 (矽的介電常數低)
```

---

### 4.2 CoWoS 如何改善 NoC 性能

**頻寬提升**：

```
HBM (High Bandwidth Memory) + CoWoS:

  HBM 規格：
    Interface Width: 1024 bits (per stack)
    Frequency: 3.2 GHz (HBM3)
    Bandwidth: 410 GB/s (per stack)

  NVIDIA H100 (5 個 HBM3 stacks):
    Total Bandwidth: 3 TB/s

  如果用傳統 PCB：
    無法實現 1024 bits 的連接
    → 物理上不可能

  CoWoS 的 Interposer：
    可以提供 10,000+ 條導線
    → 輕鬆實現 1024 bits × 5 = 5,120 bits
```

**延遲降低**：

```
傳統 PCB (Organic Substrate):
  介電常數：~4.0
  訊號速度：~15 cm/ns

  10 cm 的距離：
    延遲：~6.7 ns

CoWoS Interposer (Silicon):
  介電常數：~11.9
  訊號速度：~8.7 cm/ns

  但距離更短：~1 cm
    延遲：~1.1 ns

  → 延遲降低 6x
```

**功耗降低**：

```
傳統 I/O (SerDes):
  需要序列化/解序列化
  功耗：~10 pJ/bit

CoWoS (Parallel I/O):
  不需要 SerDes
  功耗：~1 pJ/bit

  → 功耗降低 10x
```

---

### 4.3 案例：NVIDIA H100

**架構**：

```
NVIDIA H100 (2022):
  製程：TSMC 4nm
  Die Size：814 mm²
  電晶體：800 億

  封裝：CoWoS-S

  記憶體：
    5 個 HBM3 stacks
    每個 stack：16 GB
    總容量：80 GB
    總頻寬：3 TB/s

  NoC 架構：
    內部：Mesh (推測)
    連接 HBM：專用 NoC
```

**HBM 與 GPU 的連接**：

```
每個 HBM stack：
  Interface Width：1024 bits
  Micro-Bump 數量：~10,000 個
  Pitch：~40 μm

  透過 CoWoS Interposer 連接到 GPU

  延遲：~10 ns
  頻寬：410 GB/s

  vs 傳統 DDR5：
    延遲：~50 ns
    頻寬：~50 GB/s

  → HBM 延遲降低 5x，頻寬提升 8x
```

**為什麼 AI 需要 HBM？**

```
AI 訓練的瓶頸：
  不是算得不夠快
  而是資料來不及餵給 GPU

  GPT-3 (175B 參數):
    模型大小：~350 GB (FP16)
    每次訓練迭代需要讀取整個模型

    如果用 DDR5 (50 GB/s):
      讀取時間：7 秒

    如果用 HBM3 (3 TB/s):
      讀取時間：0.12 秒

    → HBM 讓訓練速度提升 58x
```

---

## 五、CPO：光的速度

### 5.1 為什麼需要光互連？

**電互連的極限**：

```
高速 SerDes (112G PAM4):
  功耗：~10 pJ/bit
  傳輸距離：~10 cm (on PCB)

  如果要傳輸 1 米：
    需要 Retimer (中繼器)
    功耗增加到 ~20 pJ/bit

  如果要傳輸 10 米：
    需要主動式光纜 (AOC)
    成本：~$500
    功耗：~30 pJ/bit
```

**光互連的優勢**：

```
光纖傳輸：
  功耗：~5 pJ/bit (不隨距離增加)
  傳輸距離：~10 km (單模光纖)

  → 長距離傳輸的最佳選擇
```

---

### 5.2 什麼是 CPO？

**CPO (Co-Packaged Optics)** 是將**光引擎直接封裝到晶片旁邊**的技術。

**傳統光模組**：

```
Switch ASIC → SerDes → Optical Module (外部)

  問題：
    SerDes 功耗高 (~10 pJ/bit)
    需要外部光模組 (成本高)
    訊號經過多次轉換 (延遲高)
```

**CPO**：

```
Switch ASIC → Optical Engine (封裝內部) → 光纖

  優點：
    省去 SerDes (功耗降低 50%)
    光引擎在封裝內 (成本降低)
    訊號直接轉換 (延遲降低)
```

---

### 5.3 CPO 的應用場景

**資料中心交換機**：

```
傳統 51.2T 交換機：
  64 個 800G 光模組
  功耗：~3,000 W
  成本：~$100,000

  CPO 交換機：
  64 個 800G CPO 介面
  功耗：~1,500 W
  成本：~$60,000

  → 功耗降低 50%，成本降低 40%
```

**AI 集群 (GPU-to-GPU)**：

```
NVIDIA DGX H100 (8 個 GPU):
  每個 GPU：8 個 NVLink 4.0 (900 GB/s)

  傳統方案：
    使用銅纜 (限制在 1-2 米)
    或主動式光纜 (成本高)

  CPO 方案 (未來):
    GPU 直接輸出光訊號
    可以傳輸 10+ 米
    功耗降低 50%
```

---

### 5.4 CPO 與 NoC 的關係

**目前**：

```
CPO 主要用於晶片對外部的連接：
  Switch ASIC → 外部網路
  GPU → GPU (跨機櫃)

  → 不是晶片內部的 NoC
```

**未來**：

```
Optical NoC (晶片內部光互連):

  概念：
    在晶片內部使用光波導 (Waveguide)
    取代傳統的金屬導線

  優點：
    頻寬極高 (THz 級別)
    功耗極低 (~0.1 pJ/bit)
    延遲極低 (光速)

  挑戰：
    製程整合困難 (光學 + CMOS)
    成本高
    熱管理複雜

  → 目前仍在研究階段
```

---

## 六、製程節點的影響

### 6.1 Wire Delay 的惡化

**實際數據**：

```
28nm 製程：
  Gate Delay：20 ps
  Wire Delay (1mm)：30 ps
  Wire Delay 佔比：60%

7nm 製程：
  Gate Delay：10 ps
  Wire Delay (1mm)：100 ps
  Wire Delay 佔比：91%

3nm 製程：
  Gate Delay：7 ps
  Wire Delay (1mm)：150 ps
  Wire Delay 佔比：96%

  → Wire Delay 主導延遲
```

---

### 6.2 NoC 設計的權衡

**Link Width vs Frequency**：

```
選項 1：寬而慢
  Link Width：512 bits
  Frequency：1 GHz
  Bandwidth：512 GB/s
  功耗：低 (低頻)

選項 2：窄而快
  Link Width：128 bits
  Frequency：4 GHz
  Bandwidth：512 GB/s
  功耗：高 (高頻)

  在先進製程 (7nm, 5nm)：
    選項 1 更好 (Wire Delay 高，低頻更省電)
```

**Pipeline Depth vs Latency**：

```
淺 Pipeline (2-stage):
  延遲：低 (2 cycles)
  時脈頻率：低 (~1 GHz)

  適合：28nm, 16nm

深 Pipeline (4-stage):
  延遲：高 (4 cycles)
  時脈頻率：高 (~2 GHz)

  適合：7nm, 5nm

  → 先進製程需要更深的 Pipeline
```

---

## 七、真實案例深度分析

### 7.1 NVIDIA H100

**完整架構**：

```
NVIDIA H100 Tensor Core GPU (2022):

  製程：TSMC 4nm (N4)
  Die Size：814 mm²
  電晶體：800 億

  運算單元：
    132 個 SM (Streaming Multiprocessor)
    16,896 個 CUDA Cores
    528 個 Tensor Cores (第 4 代)

  記憶體：
    L2 Cache：60 MB
    HBM3：80 GB (5 stacks)
    HBM3 頻寬：3 TB/s

  封裝：CoWoS-S

  互連：
    NVLink 4.0：18 links × 50 GB/s = 900 GB/s
    PCIe 5.0：128 GB/s
```

**NoC 架構（推測）**：

```
內部 NoC：
  拓撲：Mesh (推測 11×12)
  連接：132 個 SM + L2 Cache Slices

  Link Width：256 bits (推測)
  Frequency：2 GHz (推測)
  Bandwidth：512 GB/s per link (推測)

HBM NoC：
  專用的 NoC 連接 5 個 HBM Controller

  每個 HBM Controller：
    Interface Width：1024 bits
    Frequency：3.2 GHz
    Bandwidth：410 GB/s

  透過 CoWoS Interposer 連接到 HBM
```

**性能**：

```
FP64 (雙精度)：
  60 TFLOPS

FP32 (單精度)：
  60 TFLOPS (with FP64 Tensor Cores)

FP16 (半精度)：
  1,979 TFLOPS (with Tensor Cores)

INT8：
  3,958 TOPS (with Tensor Cores)

功耗：700 W
```

> 註：以上 NVIDIA H100 性能數據（TFLOPS、TOPS、頻寬等）為基於 NVIDIA 公開 whitepaper 的規格值，
> 用來說明 CoWoS 封裝與 NoC 設計的性能特性，
> 實際性能會依應用、軟體優化與系統配置而異。

---

### 7.2 AMD MI300X

**完整架構**：

```
AMD Instinct MI300X (2023):

  製程：
    Compute Die (XCD)：TSMC 5nm
    I/O Die (IOD)：TSMC 6nm

  Chiplet 架構：
    8 個 XCD (Compute Chiplet)
    4 個 IOD (I/O Chiplet)

  運算單元：
    304 個 CU (Compute Unit)
    19,456 個 Stream Processors

  記憶體：
    L3 Cache：256 MB (Infinity Cache)
    HBM3：192 GB (8 stacks, 24 GB each)
    HBM3 頻寬：5.3 TB/s

  封裝：CoWoS-S
```

**Infinity Fabric 3.0**：

```
XCD 內部 (On-Die NoC):
  拓撲：Mesh
  連接：38 個 CU per XCD

XCD 到 IOD (D2D 介面):
  使用 Infinity Fabric Links
  Bandwidth：~100 GB/s per link (推測)

IOD 到 HBM：
  每個 IOD 連接 2 個 HBM stacks
  每個 HBM：
    Interface Width：1024 bits
    Frequency：5.2 GHz
    Bandwidth：665 GB/s

  總頻寬：8 × 665 = 5,320 GB/s
```

**性能**：

```
FP64 (雙精度)：
  163 TFLOPS

FP32 (單精度)：
  163 TFLOPS

FP16 (半精度)：
  1,307 TFLOPS (with Matrix Cores)

INT8：
  2,614 TOPS (with Matrix Cores)

功耗：750 W
```

**vs NVIDIA H100**：

```
記憶體容量：
  MI300X：192 GB
  H100：80 GB
  → MI300X 多 2.4x

記憶體頻寬：
  MI300X：5.3 TB/s
  H100：3 TB/s
  → MI300X 多 1.77x

FP16 性能：
  MI300X：1,307 TFLOPS
  H100：1,979 TFLOPS
  → H100 快 1.51x
```

> 註：以上 AMD MI300X 性能數據（TFLOPS、TOPS、頻寬等）為基於 AMD 公開 whitepaper 的規格值，
> 用來說明 3D 堆疊與 Infinity Fabric 的性能特性，
> 實際性能會依應用、軟體優化與系統配置而異。

---

### 7.3 Intel Ponte Vecchio

**完整架構**：

```
Intel Ponte Vecchio (2022):

  製程：
    Compute Tile：TSMC 5nm
    Base Tile：Intel 7nm
    HBM Base Die：TSMC 7nm

  Chiplet 架構：
    47 個 Tiles (Chiplet)
    - 16 個 Compute Tiles (Xe Cores)
    - 8 個 Rambo Cache Tiles (L2)
    - 2 個 Base Tiles (I/O)
    - 8 個 HBM Base Dies
    - 11 個 EMIB (Embedded Multi-die Interconnect Bridge)
    - 2 個 Xe Link I/O Tiles

  記憶體：
    HBM2e：128 GB (8 stacks, 16 GB each)
    HBM2e 頻寬：3.2 TB/s

  封裝：EMIB + Foveros 3D
```

**EMIB 與 Foveros**：

```
EMIB (2.5D 封裝):
  類似 CoWoS 的 Interposer
  但只在需要連接的地方使用
  → 成本較低

Foveros (3D 封裝):
  垂直堆疊 Chiplet
  使用 TSV (Through-Silicon Via)
  → 密度更高
```

**Xe Link**：

```
Xe Link (類似 NVLink):
  16 links × 50 GB/s = 800 GB/s

  連接多個 GPU：
    2 個 GPU：1,600 GB/s
    4 個 GPU：3,200 GB/s
```

**挑戰**：

```
47 個 Tiles 的整合：
  複雜度極高
  良率問題
  成本高

  → Intel 在 2023 年停止了 Ponte Vecchio 的後續開發
```

---

## 八、未來趨勢

### 8.1 3D 堆疊

**垂直方向的 NoC**：

```
傳統 2D NoC：
  所有 Router 在同一平面

3D NoC：
  Router 在多個層次
  使用 TSV 連接不同層

  優點：
    距離更短 (垂直距離 < 水平距離)
    延遲更低
    頻寬更高

  挑戰：
    散熱困難 (熱量集中)
    TSV 成本高
```

**案例**：

```
AMD 3D V-Cache (Zen 3):
  在 CPU 上方堆疊 L3 Cache
  使用 TSV 連接

  L3 Cache：64 MB → 96 MB
  遊戲性能提升：~15%
```

---

### 8.2 Optical NoC（進階）

**晶片內部光互連**：

```
概念：
  在晶片內部使用光波導
  取代金屬導線

優點：
  頻寬：THz 級別 (vs GHz)
  功耗：~0.1 pJ/bit (vs 1-10 pJ/bit)
  延遲：光速 (vs 電子速度)

挑戰：
  製程整合：光學 + CMOS
  成本：非常高
  熱管理：雷射需要穩定溫度

  → 目前仍在研究階段
```

**研究進展**：

```
MIT (2015):
  展示了晶片內部光互連原型
  功耗：0.5 pJ/bit
  頻寬：300 GB/s

  → 但距離商用化還很遠
```

---

### 8.3 異構整合

**未來的 SoC**：

```
單一封裝內整合：
  CPU Chiplet (Intel, AMD, ARM)
  GPU Chiplet (NVIDIA, AMD)
  NPU Chiplet (Google TPU, Apple Neural Engine)
  Memory Chiplet (HBM, GDDR)
  I/O Chiplet (PCIe, CXL, Ethernet)

  透過 UCIe 標準連接
  使用 CoWoS 封裝

  → 真正的「樂高式」晶片設計
```

**挑戰**：

```
1. 標準化：
   UCIe 需要更廣泛的採用

2. 軟體生態：
   需要支援異構計算的軟體框架

3. 成本：
   先進封裝成本高

4. 散熱：
   功耗密度極高
```

---

## 九、總結

NoC 的未來在於**突破物理邊界**：

1. **Chiplet**：
   - 將 NoC 從晶片內延伸到晶片間
   - 提高良率、降低成本、增加靈活性
   - 挑戰：延遲增加、頻寬受限

2. **UCIe**：
   - Chiplet 之間的統一標準
   - 實現開放的生態系統
   - 讓不同廠商的 Chiplet 可以互連

3. **CoWoS**：
   - 2.5D 封裝技術
   - 實現超高頻寬互連（HBM）
   - 降低延遲、降低功耗
   - 案例：NVIDIA H100, AMD MI300X

4. **CPO**：
   - 光互連技術
   - 主要用於晶片對外連接
   - 未來可能用於晶片內部（Optical NoC）

5. **製程影響**：
   - Wire Delay 隨製程縮小而惡化
   - NoC 設計需要適應先進製程
   - 寬而慢 > 窄而快

6. **未來趨勢**：
   - 3D 堆疊（垂直 NoC）
   - Optical NoC（光互連）
   - 異構整合（樂高式設計）

**關鍵洞察**：

NoC 不再只是晶片內部的互連網路，它正在演變成：
- **On-Die NoC**：晶片內部（Mesh, Torus）
- **On-Package NoC**：封裝內部（Chiplet, UCIe, CoWoS）
- **On-Board NoC**：板級互連（CPO, Optical）

這些技術讓 NoC 從**微米級**（晶片內）擴展到**米級**（資料中心），成為現代 AI 和 HPC 系統的核心骨幹。

---

## 參考資料

**公開文檔**：
- *Principles and Practices of Interconnection Networks* - Dally & Towles
- NVIDIA H100 Tensor Core GPU Architecture Whitepaper (2022)
- AMD Instinct MI300X Architecture Whitepaper (2023)
- Intel Ponte Vecchio Architecture Overview (2022)
- UCIe Consortium Specification v1.0 (2022)
- TSMC CoWoS Technology Overview
- *A Survey of Chiplet Interconnect Technologies* - IEEE (2021)

**個人筆記**：
- Network-on-Chip Technical Notes (基於公開領域知識整理)
- Advanced Packaging Technologies Notes (基於公開領域知識整理)


---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: https://github.com/djiangtw/tech-column-public
