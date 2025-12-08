# Cache 基礎概念入門：用圖書館理解 CPU Cache

**作者**: Danny Jiang  
**日期**: 2025-12-07  

---

## 前言：一個真實的性能災難

2018 年，我在一家晶片公司擔任系統軟體工程師。有一天，測試團隊發現一個奇怪的問題：同一個影像處理程式，在開發板上跑得飛快（每秒處理 60 幀），但在客戶的產品上卻慢得像蝸牛（每秒只有 15 幀）。

兩個系統的 CPU 頻率相同，記憶體容量也一樣，為什麼性能差了 4 倍？

經過一週的分析，我們發現罪魁禍首是 **Cache**。開發板的 L2 Cache 是 512 KB，而客戶產品為了省成本，
只配了 128 KB。程式的工作集（Working Set）大約是 300 KB，在開發板上剛好放得進 L2 Cache，
但在客戶產品上卻不斷發生 Cache Miss，導致性能崩潰。

這個慘痛的教訓讓我深刻體會到：**不理解 Cache，就無法寫出高性能的程式**。

本文將用生動的比喻——**圖書館**——來幫助你理解 CPU Cache 的基本概念。讀完這篇文章，你將能夠：

- 理解為什麼 CPU 需要多層 Cache
- 知道 Cache 如何組織和查找資料
- 學會如何寫出對 Cache 友善的程式碼

---

## 為什麼需要 Cache？速度的鴻溝

### 研究員的困境

想像一下，你是一位研究員，正在撰寫一篇關於量子物理的論文。你需要查閱大量的參考資料——從愛因斯坦的相對論到最新的量子糾纏研究。這些資料存放在哪裡呢？

- **主記憶體（RAM）**：就像一座巨大的圖書館，藏書豐富（數 GB），但距離你的辦公桌很遠。每次查資料都要走到圖書館，來回可能要花 10 分鐘。
- **CPU 暫存器（Registers）**：就像你辦公桌上的筆記本，容量很小（只能記幾個重點），但隨手可得，1 秒鐘就能翻開。

問題來了：圖書館太遠（RAM 太慢），筆記本太小（Registers 太少）。如果每次需要引用一個公式都要走到圖書館，你的論文可能永遠寫不完！

### 小書架的魔法

**解決方案**：在辦公桌旁邊放一個**小書架**，把最常用的書放在上面。這個小書架，就是 **Cache**！

當你需要查資料時：

1. **先看小書架**（Cache）——如果找到了，太好了！只需要 3 秒鐘
2. **如果小書架沒有**，才走到圖書館（RAM）——需要 10 分鐘
3. **從圖書館借回來的書**，順便放到小書架上——下次就不用再跑一趟了

這就是 Cache 的核心概念：**用小容量但高速的儲存空間，來彌補大容量但低速的主記憶體**。

### 速度與容量的鴻溝：數字會說話

讓我們看看真實的數字。以一顆現代 CPU（例如 Intel Core i9 或 AMD Ryzen 9）為例：

| 儲存層級 | 容量 | 存取時間 | 比喻 |
|---------|------|---------|------|
| CPU Registers | 極小（數十 Bytes） | 0.3 ns (1 cycle) | 辦公桌上的筆記本 |
| L1 Cache | 小（32-64 KB） | 1 ns (3-4 cycles) | 辦公桌旁的小書架 |
| L2 Cache | 中（256 KB - 1 MB） | 3-5 ns (10-20 cycles) | 辦公室的書櫃 |
| L3 Cache | 大（8-64 MB） | 10-20 ns (40-80 cycles) | 樓層的共用書庫 |
| RAM | 很大（8-128 GB） | 50-100 ns (200-300 cycles) | 遠處的圖書館 |
| SSD | 超大（256 GB - 4 TB） | 50,000-100,000 ns | 城市圖書館 |

> 註：上述容量與延遲數字為代表性估計，用來說明 **不同層級之間的量級差異**，實際數值會隨 CPU 世代、主機板與記憶體配置而有所不同。

**關鍵觀察**：

- L1 Cache 比 RAM 快 **50-100 倍**
- 如果你的程式能把 90% 的資料存取都命中 L1 Cache，性能可以提升數十倍！

這就是為什麼現代 CPU 都有多層 Cache——它們是速度與容量之間的橋樑。

---

## Cache 的基本組成：圖書館比喻

現在，讓我們深入了解 Cache 的內部結構。我會用圖書館的概念來解釋每個元件。

### 1. Cache Line（書本）

**Cache Line** 是 Cache 的基本單位，就像圖書館裡的一本書。

- **大小**：現代 CPU 的 Cache Line 通常是 **64 Bytes**
- **為什麼是 64 Bytes？**：這是空間局部性（Spatial Locality）與硬體複雜度的甜蜜點
  - 太小（如 32 Bytes）：頻繁搬運，效率低
  - 太大（如 128 Bytes）：浪費空間，可能搬來用不到的資料

**比喻**：

- 你去圖書館借書，不會只借一頁紙，而是借整本書（64 Bytes）
- 因為你讀完第 10 頁，很可能接著讀第 11 頁（空間局部性）

**真實案例**：

假設你在處理一個整數陣列：

```c
int array[1000];
for (int i = 0; i < 1000; i++) {
    sum += array[i];  // 每次讀取 4 bytes
}
```

當 CPU 讀取 `array[0]` 時，它不會只載入 4 bytes，而是載入整個 Cache Line（64 bytes）。這意味著 `array[0]` 到 `array[15]`（共 16 個整數）都被載入了！接下來讀取 `array[1]` 到 `array[15]` 時，都是 Cache Hit，速度飛快。

### 2. Cache Set（書架的層架）

**Cache Set** 是 Cache 的索引單位，就像書架上的一層層架子。

- 每個 Set 可以存放多個 Cache Line
- CPU 用地址的一部分（Index）來決定資料應該放在哪個 Set

**比喻**：

- 圖書館的書架有很多層（Set 0, Set 1, Set 2, ...）
- 你要找的書在第幾層？看書的分類號（Index）

### 3. Cache Way（層架上的格子）

**Cache Way** 代表每個 Set 裡有幾個位置（格子）可以放 Cache Line。

- **Associativity（相連度）**：每個 Set 有幾個 Way
  - 2-Way：每層有 2 個格子
  - 4-Way：每層有 4 個格子
  - 8-Way：每層有 8 個格子
  - 16-Way：每層有 16 個格子

**比喻**：

- 每層書架有多個格子（Way 0, Way 1, Way 2, ...）
- 同一層可以放多本書，增加彈性

### 4. Tag（書名標籤）

**Tag** 用來確認這個 Cache Line 是否是你要找的資料。

- 每個 Cache Line 都有一個 Tag，記錄它存的是哪個地址的資料
- CPU 會比對 Tag 來確認是否命中（Hit）

**比喻**：

- 你找到了第 5 層書架（Index），但這層有 4 本書（4-Way）
- 你要看每本書的書名（Tag），確認哪本是你要的

---

## 核心公式：Cache 容量計算

理解了基本組成後，讓我們看看如何計算 Cache 的容量：

```
總容量 = Sets × Ways × Cache Line Size
```

這個公式非常重要，讓我們用一個實際例子來理解。

### 範例：32 KB, 8-Way, 64-Byte Cache Line

假設我們有一個 32 KB 的 L1 Data Cache，配置如下：

```
總容量 = 32 KB
Ways = 8
Cache Line Size = 64 Bytes

計算 Sets：
32 KB = Sets × 8 × 64 Bytes
Sets = 32 KB / (8 × 64 Bytes)
Sets = 32,768 / 512
Sets = 64
```

所以這個 Cache 有：

- **64 個 Sets**（64 層書架）
- **每個 Set 有 8 個 Ways**（每層有 8 個格子）
- **每個 Way 存放 64 Bytes**（每本書 64 頁）

---

## Cache 查找過程：尋書之旅

現在，讓我們看看 CPU 如何在 Cache 中查找資料。這是整篇文章最重要的部分！

### 地址切分：三個關鍵部分

當 CPU 要讀取一個記憶體地址時，它會把地址切成三個部分：

```
記憶體地址（假設 32-bit）：
┌─────────────┬──────────┬────────────┐
│    Tag      │  Index   │   Offset   │
│  (20 bits)  │ (6 bits) │  (6 bits)  │
└─────────────┴──────────┴────────────┘
 bit[31:12]    bit[11:6]   bit[5:0]
```

**各部分的功能**：

1. **Offset (6 bits)**：定位 Cache Line 內的哪個 Byte
   - 6 bits 可以表示 0-63，剛好對應 64 Bytes
   - 就像書中的頁碼

2. **Index (6 bits)**：選擇哪個 Set
   - 6 bits 可以表示 0-63，剛好對應 64 個 Sets
   - 就像書架的層數編號

3. **Tag (20 bits)**：確認是否是正確的資料
   - 剩餘的高位元，用來比對身份
   - 就像書名或 ISBN

### 查找步驟：一個完整的例子

假設 CPU 要讀取地址 `0x12345678`，讓我們一步步看看 Cache 如何查找：

#### 步驟 1：切分地址

```
地址：0x12345678 = 0001 0010 0011 0100 0101 0110 0111 1000

切分：
Tag    = bits[31:12] = 0x12345 (0001 0010 0011 0100 0101)
Index  = bits[11:6]  = 0x19    (011001) = 25 (十進位)
Offset = bits[5:0]   = 0x38    (111000) = 56 (十進位)
```

#### 步驟 2：用 Index 找到對應的 Set

```
Index = 25 → 去第 25 層書架
```

#### 步驟 3：並行比較該 Set 中所有 Ways 的 Tag

```
Set 25 有 8 個 Way（8 本書）：

Way 0: Tag = 0x2468A, Valid = 1, Data = [...]
Way 1: Tag = 0x2468D, Valid = 1, Data = [...]
Way 2: Tag = 0x12345, Valid = 1, Data = [...]  ← 找到了！
Way 3: Tag = 0x00000, Valid = 0, Data = [...]
Way 4: Tag = 0xABCDE, Valid = 1, Data = [...]
Way 5: Tag = 0x11111, Valid = 1, Data = [...]
Way 6: Tag = 0x22222, Valid = 1, Data = [...]
Way 7: Tag = 0x33333, Valid = 1, Data = [...]

比對：
地址的 Tag = 0x12345
Way 2 的 Tag = 0x12345
→ 匹配！Cache Hit！
```

#### 步驟 4：讀取資料

```
Offset = 56 → 從 Way 2 的 Cache Line 中，讀取第 56 個 Byte
```

整個過程只需要 **3-4 個 CPU cycles**（約 1 ns），比從 RAM 讀取快 50-100 倍！

### Cache Hit vs Cache Miss

- **Cache Hit（命中）**：資料在 Cache 中找到了
  - 速度快（1-5 ns）
  - 就像你要的書剛好在小書架上

- **Cache Miss（未命中）**：資料不在 Cache 中
  - 需要從 RAM 讀取（50-100 ns）
  - 就像你要的書不在小書架上，要去圖書館借
  - 借回來後，會放到 Cache 裡（替換掉某本舊書）

**性能影響**：

假設你的程式執行 1 億次記憶體存取：

- 如果 Cache Hit Rate = 95%，平均延遲 = 0.95 × 1ns + 0.05 × 100ns = **5.95 ns**
- 如果 Cache Hit Rate = 50%，平均延遲 = 0.50 × 1ns + 0.50 × 100ns = **50.5 ns**

Hit Rate 從 95% 降到 50%，性能差了 **8.5 倍**！這就是為什麼理解 Cache 如此重要。

---

## 真實案例：現代 CPU 的 Cache 配置

讓我們看看幾個真實的 CPU 是如何配置 Cache 的。

### Intel Core i9-12900K (Alder Lake, 2021)

**P-Core (Performance Core)**：

- **L1 I-Cache**: 32 KB, 8-Way, 64 Bytes Line
- **L1 D-Cache**: 48 KB, 12-Way, 64 Bytes Line
- **L2 Cache**: 1.25 MB, 10-Way, 64 Bytes Line (per core)
- **L3 Cache**: 30 MB, 12-Way, 64 Bytes Line (shared)

**觀察**：

- L1 容量小但速度快（48 KB, 12-Way）
- L2 容量大幅增加（1.25 MB），但 Associativity 只有 10-Way
- L3 容量巨大（30 MB），所有核心共用

### AMD Ryzen 9 7950X (Zen 4, 2022)

- **L1 I-Cache**: 32 KB, 8-Way, 64 Bytes Line
- **L1 D-Cache**: 32 KB, 8-Way, 64 Bytes Line
- **L2 Cache**: 1 MB, 8-Way, 64 Bytes Line (per core)
- **L3 Cache**: 64 MB, 16-Way, 64 Bytes Line (shared)

**觀察**：

- AMD 的 L3 Cache 更大（64 MB vs Intel 的 30 MB）
- 所有層級都使用 64 Bytes 的 Cache Line

### ARM Cortex-X4 (2023)

- **L1 I-Cache**: 64 KB, 4-Way, 64 Bytes Line
- **L1 D-Cache**: 64 KB, 4-Way, 64 Bytes Line
- **L2 Cache**: 512 KB - 2 MB, 8-Way, 64 Bytes Line (per core)

**觀察**：

- ARM 的 L1 Cache 更大（64 KB）
- Associativity 較低（4-Way），但容量大

### 通用 RISC-V 處理器（典型配置）

- **L1 I-Cache**: 32 KB, 8-Way, 64 Bytes Line
- **L1 D-Cache**: 32 KB, 8-Way, 64 Bytes Line
- **L2 Cache**: 256 KB - 1 MB, 8-Way, 64 Bytes Line

**觀察**：

- RISC-V 的配置與 x86/ARM 類似
- 都遵循「64 Bytes Cache Line」的業界標準

---

## 為什麼所有層級都用 64 Bytes？

你可能會問：為什麼 L1、L2、L3 都用 64 Bytes 的 Cache Line？為什麼不是 L1 用 32 Bytes，L2 用 128 Bytes？

### 原因 1：簡化硬體設計

如果 L1 用 64 Bytes，L2 用 128 Bytes：

- L1 → L2 傳輸時，需要處理 1:2 的映射關係
- 硬體邏輯變複雜，延遲增加

**統一 64 Bytes**：

- L1 ↔ L2 ↔ L3 ↔ RAM 都是 1:1 對應
- 硬體設計簡單，延遲低

### 原因 2：避免 Read-Modify-Write

如果 L1 是 64 Bytes，L2 是 128 Bytes：

- L1 寫回 L2 時，只修改了 128 Bytes 中的 64 Bytes
- L2 需要先讀取整個 128 Bytes，修改一半，再寫回（Read-Modify-Write）
- 性能損失嚴重

### 原因 3：DDR 記憶體的 Burst Length

現代 DDR4/DDR5 記憶體的 Burst Length = 8：

```
每次 Burst 傳輸 = 8 × 8 bytes = 64 bytes
```

如果 Cache Line = 64 Bytes：

- 一次 DDR Burst 剛好填滿一個 Cache Line
- 頻寬利用率 = 100%

如果 Cache Line = 32 Bytes：

- 需要半個 Burst，浪費頻寬

如果 Cache Line = 128 Bytes：

- 需要兩次 Burst，增加延遲

### 原因 4：64 Bytes 是甜蜜點

- **太小（如 32 Bytes）**：無法充分利用空間局部性，頻繁的 Cache Miss
- **太大（如 128 Bytes）**：可能搬來用不到的資料，Cache 污染（Pollution）
- **64 Bytes**：平衡空間局部性與 Cache 利用率

---

## 總結：Cache 的核心概念

### 關鍵要點回顧

1. **Cache Line**：Cache 的基本單位（通常 64 Bytes）
2. **Cache Set**：用 Index 定位的層架
3. **Cache Way**：每個 Set 裡的格子數（Associativity）
4. **Tag**：確認資料身份的標籤

### 核心公式

```
總容量 = Sets × Ways × Cache Line Size

Offset bits = log₂(Cache Line Size)
Index bits  = log₂(Sets)
Tag bits    = Address bits - Offset bits - Index bits
```

### 圖書館比喻總結

| Cache 概念 | 圖書館比喻 | 實際大小 |
|-----------|-----------|---------|
| Cache Line | 一本書 | 64 Bytes |
| Cache Set | 書架的一層 | 64-256 個 |
| Cache Way | 層架上的格子 | 4-16 個 |
| Tag | 書名標籤 | 20-50 bits |
| Index | 書架層數編號 | 6-8 bits |
| Offset | 書中的頁碼 | 6 bits (0-63) |
| Cache Hit | 書在小書架上 | 1-5 ns |
| Cache Miss | 要去圖書館借 | 50-100 ns |

### 給程式設計師的建議

1. **循序存取優於隨機存取**：利用空間局部性
2. **小心陣列的 Stride**：Stride 太大會降低 Cache Hit Rate
3. **注意資料結構的大小**：盡量讓工作集放進 L2 Cache
4. **避免 False Sharing**：多執行緒程式要注意 Cache Line 對齊

---

## 下一篇預告

在下一篇文章中，我們將深入探討 **Cache Associativity（相連度）**：

- 為什麼需要 Associativity？
- 1-Way vs 4-Way vs 16-Way 有什麼差別？
- 如何平衡命中率與硬體複雜度？
- 停車場比喻：理解「選擇的自由度」

我們會用**停車場**的比喻，讓你理解為什麼現代 CPU 都選擇 8-Way 或 16-Way Set Associative Cache。

---

## 參考資料

1. *Computer Architecture: A Quantitative Approach* (6th Edition) - Hennessy & Patterson
2. *What Every Programmer Should Know About Memory* - Ulrich Drepper
3. Intel Core i9-12900K Architecture Whitepaper
4. AMD Zen 4 Architecture Whitepaper
5. ARM Cortex-X4 Technical Reference Manual
6. 個人在系統軟體工程領域 20+ 年的實務經驗

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: https://github.com/djiangtw/tech-column-public
