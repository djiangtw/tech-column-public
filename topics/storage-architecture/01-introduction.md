# Storage Architecture - 從硬體到軟體的完整視角

**作者**: Danny Jiang  
**日期**: 2025-12-08  

---

## 前言：一次讓我重新認識存儲的經歷

2018 年，我在一家雲端服務公司擔任系統工程師。有一天，我們的資料庫服務突然出現嚴重的效能衰退：

**問題現象**：

```
正常情況：
- 平均查詢延遲：5 ms
- P99 延遲：15 ms
- QPS (Queries Per Second)：50,000

異常情況（突然發生）：
- 平均查詢延遲：50 ms  ← 增加 10 倍
- P99 延遲：500 ms     ← 增加 33 倍
- QPS：5,000           ← 下降 90%
```

**緊急排查**：

我們檢查了所有可能的原因：

- ✅ CPU 使用率：正常（40%）
- ✅ 記憶體使用率：正常（60%）
- ✅ 網路頻寬：正常（< 1 Gbps）
- ✅ 資料庫查詢計畫：沒有變化
- ✅ 應用程式程式碼：沒有部署新版本

所有指標都顯示「正常」，但效能就是很差。

**轉折點**：

直到我查看了 **SSD 的 SMART 資訊**：

```bash
sudo nvme smart-log /dev/nvme0n1

# 關鍵發現：
Percentage Used: 95%  ← 接近壽命終點
Available Spare: 5%   ← 備用空間幾乎用完
Data Units Written: 487,234,567  ← 寫入量驚人

# 計算 Write Amplification Factor (WAF):
Host Writes: 120 TB
NAND Writes: 468 TB
WAF = 468 / 120 = 3.9  ← 非常高！
```

**根本原因**：

這顆 SSD 的 **Over-Provisioning (OP) 空間幾乎耗盡**，導致：

1. **Garbage Collection (GC) 頻繁觸發**
   - 每次寫入都需要先執行 GC
   - GC 會阻塞正常的讀寫操作

2. **Write Amplification 惡化**
   - WAF 從正常的 1.5 飆升到 3.9
   - 每寫入 1 GB 資料，實際寫入 3.9 GB 到 NAND Flash

3. **延遲大幅增加**
   - 正常寫入延遲：200 μs
   - GC 期間寫入延遲：10 ms（增加 50 倍）

**解決方案**：

```bash
# 1. 執行 TRIM，釋放無效資料
sudo fstrim -v /mnt/database

# 結果：釋放了 800 GB 空間
# Available Spare: 5% → 35%

# 2. 調整資料庫配置，降低寫入量
# - 增加 Write Buffer 大小
# - 調整 Compaction 策略
# - 啟用壓縮

# 3. 監控 WAF，設定告警閾值
# WAF > 2.5 → 發送警告
# WAF > 3.0 → 觸發自動優化
```

**結果**：

效能立即恢復：

- 平均延遲：50 ms → 5 ms
- P99 延遲：500 ms → 15 ms
- QPS：5,000 → 50,000

**深刻的教訓**：

這次經歷讓我意識到：

1. **硬體細節至關重要**
   - 不了解 SSD 的內部機制（FTL、GC、WAF），就無法真正優化系統效能
   - 傳統的 CPU/Memory/Network 監控是不夠的

2. **存儲不只是「讀寫資料」**
   - SSD 內部有複雜的 Flash Translation Layer (FTL)
   - Garbage Collection 會影響延遲
   - Write Amplification 會影響壽命

3. **系統軟體工程師需要理解硬體**
   - 不能把 SSD 當成「黑盒子」
   - 需要理解接口協議（SATA、PCIe、NVMe）
   - 需要理解控制器內部（FTL、ECC、RAID）

從那天起，我開始深入研究 Storage Architecture。這個系列文章，就是我這幾年學習和實踐的總結。

---

## 一、為什麼要學習 Storage Architecture？

### 1.1 對系統軟體工程師的價值

**1. 效能優化**

```
案例：資料庫寫入優化

不了解 SSD：
- 使用預設配置
- 隨機寫入，4KB 小塊
- WAF = 3.5
- 寫入吞吐量：200 MB/s

了解 SSD 後：
- 使用 Direct I/O，避免 Page Cache
- 批次寫入，128KB 大塊
- 對齊到 SSD Page (16KB)
- 使用 TRIM 釋放空間
- WAF = 1.5
- 寫入吞吐量：800 MB/s  ← 提升 4 倍
```

**2. 故障診斷**

```
案例：間歇性延遲尖峰

現象：
- P99 延遲偶爾飆升到 100 ms
- 無法穩定重現

傳統排查：
- 檢查 CPU、Memory、Network
- 找不到原因

Storage 視角：
- 檢查 SSD SMART 資訊
- 發現 Read Retry 次數異常增加
- 原因：NAND Flash 老化，BER 上升
- 解決：更換 SSD，問題消失
```

**3. 容量規劃**

```
案例：SSD 壽命預測

不了解 SSD：
- 只看 TBW (Total Bytes Written) 規格
- 1 PB TBW，每天寫入 1 TB
- 預期壽命：1,000 天

了解 SSD 後：
- 考慮 WAF = 3.0
- 實際 NAND 寫入：3 TB/天
- 實際壽命：333 天  ← 只有 1/3
- 需要提前規劃更換
```

---

### 1.2 對嵌入式系統工程師的價值

**1. 硬體選型**

```
案例：選擇合適的存儲接口

需求：
- 嵌入式 Linux 系統
- 需要存儲 100 GB 資料
- 讀取延遲 < 1 ms
- 成本敏感

選項分析：

選項 1: SATA SSD
- 頻寬：6 Gbps (600 MB/s)
- 延遲：~100 μs
- 成本：中等
- 問題：AHCI 協議開銷大，單一佇列

選項 2: NVMe SSD (PCIe Gen3 x4)
- 頻寬：32 Gbps (4 GB/s)
- 延遲：~50 μs
- 成本：高
- 優勢：多佇列，低延遲

選項 3: eMMC
- 頻寬：400 MB/s
- 延遲：~500 μs
- 成本：低
- 問題：效能較差，但足夠

決策：
- 如果預算充足：選 NVMe（最佳效能）
- 如果成本敏感：選 eMMC（夠用就好）
- SATA 處於尷尬位置（效能不如 NVMe，成本不如 eMMC）
```

**2. 電源管理**

```
案例：Power Loss Protection (PLP)

問題：
- 嵌入式系統可能突然斷電
- SSD 內部有資料在 DRAM Buffer
- 斷電會導致資料丟失

解決方案：
- 選擇有 PLP 的 SSD（內建電容）
- 或自行設計 PLP 電路
- 或使用 Sync Write，犧牲效能換取可靠性

成本分析：
- 有 PLP 的 SSD：+30% 成本
- 自行設計 PLP：+10% 成本 + 開發時間
- Sync Write：0 成本，但效能降低 50%
```

---

### 1.3 對效能優化工程師的價值

**1. 理解延遲來源**

```
案例：分解 I/O 延遲

總延遲：100 μs

分解：
1. 應用程式 → Kernel：5 μs
   - System call 開銷
   - Context switch

2. Kernel → Block Layer：10 μs
   - I/O Scheduler
   - Request merging

3. Block Layer → NVMe Driver：5 μs
   - 建立 NVMe Command
   - 寫入 Submission Queue

4. NVMe Driver → SSD Controller：10 μs
   - PCIe TLP 傳輸
   - DMA 設定

5. SSD Controller 處理：50 μs
   - FTL 查詢 L2P Mapping
   - 讀取 NAND Flash
   - ECC 解碼

6. SSD → Kernel：10 μs
   - DMA 傳輸資料
   - MSI-X 中斷

7. Kernel → 應用程式：10 μs
   - 喚醒 Process
   - 複製資料到 User Space

優化方向：
- 步驟 1-4：使用 io_uring，減少 System call
- 步驟 5：使用 Direct I/O，避免 FTL 查詢
- 步驟 6-7：使用 Polling，避免中斷
```

**2. 理解頻寬瓶頸**

```
案例：為什麼我的 NVMe SSD 只跑到 2 GB/s？

SSD 規格：
- PCIe Gen3 x4
- 理論頻寬：4 GB/s
- 標稱讀取速度：3.5 GB/s

實測：
- 循序讀取（1 MB 塊）：2.0 GB/s  ← 為什麼？

分析：
1. PCIe 協議開銷：
   - TLP Header：20 Bytes / 128 Bytes Payload
   - 開銷：15.6%
   - 有效頻寬：4 GB/s × 84.4% = 3.38 GB/s

2. NVMe 協議開銷：
   - Command Fetch：每個 I/O 需要 Fetch Command
   - Completion Queue：每個 I/O 需要寫入 CQ
   - 開銷：~5%
   - 有效頻寬：3.38 GB/s × 95% = 3.21 GB/s

3. SSD 內部瓶頸：
   - DRAM Buffer 頻寬限制
   - NAND Channel 數量限制
   - 實際頻寬：2.0 GB/s

結論：
- 瓶頸在 SSD 內部，不是 PCIe
- 需要選擇更高階的 SSD（更多 NAND Channels）
```

---

## 二、這個系列涵蓋什麼？

### 2.1 系列結構總覽

本系列共 **12 篇文章**，分為 **5 個部分**：

**Part 1: Introduction（本文）**

- 為什麼要學習 Storage Architecture
- 系列結構與閱讀指南

**Part 2: Host Interfaces（主機接口）**

- **SATA/AHCI**：傳統接口的瓶頸
- **PCIe**：高速串列匯流排
- **NVMe**：為 SSD 設計的協議
- **CXL**：打破存儲與記憶體的界限

**Part 3: SSD Controller Internals（控制器內部）**

- **Flash Translation Layer (FTL)**：地址轉換的藝術
- **Garbage Collection & Wear Leveling**：SSD 的生命管理
- **Error Correction**：對抗 NAND Flash 的不可靠性

**Part 4: Advanced Topics（進階主題）**

- **Computational Storage**：在 SSD 內部執行計算
- **NVMe-oF**：透過網路存取 NVMe
- **Persistent Memory**：DRAM 與 SSD 之間的新選擇

**Part 5: Real-world Applications（實際應用）**

- **Database Optimization**：資料庫如何利用 SSD 特性
- **AI/ML Workloads**：訓練資料的儲存優化
- **Cloud Storage**：雲端環境中的 SSD 管理

---

### 2.2 知識地圖

```
Storage Architecture 知識體系：

┌─────────────────────────────────────────────────────────┐
│                    Application Layer                    │
│  (Database, File System, Object Storage, AI/ML)         │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                  Operating System                       │
│  (VFS, Block Layer, I/O Scheduler, Device Driver)       │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                  Host Interface                         │
│  ┌──────────┬──────────┬──────────┬──────────┐         │
│  │  SATA    │  PCIe    │  NVMe    │   CXL    │         │
│  │  AHCI    │  Gen3/4/5│  Queue   │  .io/.mem│         │
│  └──────────┴──────────┴──────────┴──────────┘         │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                 SSD Controller                          │
│  ┌─────────────────────────────────────────────┐       │
│  │  Front-End (FE)                             │       │
│  │  - PCIe Interface                           │       │
│  │  - NVMe Protocol Handler                    │       │
│  │  - Command Parser                           │       │
│  └─────────────────────────────────────────────┘       │
│                     │                                   │
│                     ▼                                   │
│  ┌─────────────────────────────────────────────┐       │
│  │  Flash Translation Layer (FTL)              │       │
│  │  - L2P Mapping                              │       │
│  │  - Garbage Collection                       │       │
│  │  - Wear Leveling                            │       │
│  └─────────────────────────────────────────────┘       │
│                     │                                   │
│                     ▼                                   │
│  ┌─────────────────────────────────────────────┐       │
│  │  Back-End (BE)                              │       │
│  │  - NAND Flash Interface (ONFI/Toggle)       │       │
│  │  - ECC (LDPC)                               │       │
│  │  - Multi-Channel Controller                 │       │
│  └─────────────────────────────────────────────┘       │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                  NAND Flash                             │
│  ┌────────┬────────┬────────┬────────┬────────┐        │
│  │ Die 0  │ Die 1  │ Die 2  │ Die 3  │ Die 4  │ ...    │
│  │ (TLC)  │ (TLC)  │ (TLC)  │ (TLC)  │ (TLC)  │        │
│  └────────┴────────┴────────┴────────┴────────┘        │
└─────────────────────────────────────────────────────────┘
```

**本系列的覆蓋範圍**：

- ✅ **Host Interface**：SATA、PCIe、NVMe、CXL（Part 2）
- ✅ **SSD Controller**：FTL、GC、WL、ECC（Part 3）
- ✅ **Advanced Topics**：Computational Storage、NVMe-oF、PM（Part 4）
- ✅ **Applications**：Database、AI/ML、Cloud（Part 5）

---

### 2.3 每篇文章的結構

每篇文章都遵循相同的結構：

**1. 前言：真實案例**

- 從我的實際經驗出發
- 引出技術問題
- 激發讀者興趣

**2. 技術深度解析**

- 原理講解
- 架構圖示
- 實測數據

**3. 實際應用**

- 如何在系統軟體中應用
- 效能優化技巧
- 故障診斷方法

**4. 總結與展望**

- 核心要點回顧
- 與其他主題的關聯
- 下一篇預告

---

## 三、如何閱讀這個系列？

### 3.1 依據背景選擇閱讀路徑

**路徑 1：系統軟體工程師**

```
建議閱讀順序：

1. 本文（Introduction）
   ↓
2. NVMe（了解現代存儲接口）
   ↓
3. FTL（理解 SSD 內部機制）
   ↓
4. GC & Wear Leveling（理解效能衰退原因）
   ↓
5. Error Correction（理解可靠性機制）
   ↓
6. Database Optimization（實際應用）

可選：
- SATA/AHCI（如果需要維護舊系統）
- PCIe（如果需要深入理解底層）
- CXL（如果對未來技術感興趣）
```

**路徑 2：嵌入式系統工程師**

```
建議閱讀順序：

1. 本文（Introduction）
   ↓
2. SATA/AHCI（了解傳統接口）
   ↓
3. PCIe（了解高速接口）
   ↓
4. NVMe（了解現代協議）
   ↓
5. FTL（理解 SSD 內部）
   ↓
6. Error Correction（理解可靠性）

重點關注：
- 硬體選型
- 電源管理
- 成本分析
```

**路徑 3：效能優化工程師**

```
建議閱讀順序：

1. 本文（Introduction）
   ↓
2. PCIe（理解頻寬瓶頸）
   ↓
3. NVMe（理解延遲來源）
   ↓
4. FTL（理解 WAF）
   ↓
5. GC & Wear Leveling（理解效能衰退）
   ↓
6. Database Optimization（實際優化案例）

重點關注：
- 延遲分解
- 頻寬優化
- I/O Pattern 優化
```

**路徑 4：研究人員 / 架構師**

```
建議按順序閱讀全部文章：

Part 1 → Part 2 → Part 3 → Part 4 → Part 5

重點關注：
- 技術演進脈絡
- 設計權衡
- 未來趨勢
```

---

### 3.2 前置知識要求

**必備知識**：

- ✅ **基礎 C/C++ 程式設計**
  - 指標、記憶體管理
  - 系統呼叫（open、read、write）

- ✅ **基礎作業系統概念**
  - Process、Thread
  - Virtual Memory
  - File System

- ✅ **基礎計算機組織**
  - CPU、Memory、I/O
  - Bus、Interrupt

**加分但非必須**：

- 🔶 **Linux Kernel 知識**
  - Block Layer
  - Device Driver

- 🔶 **硬體知識**
  - 數位電路
  - 訊號完整性

- 🔶 **效能分析工具**
  - perf、fio、blktrace

**不需要的知識**：

- ❌ NAND Flash 物理原理（我會在文章中解釋）
- ❌ PCIe 電氣特性（我們專注於協議層）
- ❌ FPGA 或 ASIC 設計（我們專注於軟體視角）

---

### 3.3 實驗環境建議

**最小環境**：

```
硬體：
- 任何有 SSD 的電腦（SATA 或 NVMe 都可以）
- 不需要特殊硬體

軟體：
- Linux（Ubuntu 20.04+ 或其他發行版）
- 基本工具：nvme-cli, smartmontools, fio

安裝：
sudo apt install nvme-cli smartmontools fio
```

**推薦環境**：

```
硬體：
- NVMe SSD（可以實驗 NVMe 命令）
- 至少 16 GB RAM（可以測試大檔案 I/O）

軟體：
- Linux Kernel 5.10+（支援 io_uring）
- 效能分析工具：perf, bpftrace, blktrace

安裝：
sudo apt install linux-tools-generic bpfcc-tools blktrace
```

**進階環境**：

```
硬體：
- 多顆 NVMe SSD（可以測試 RAID、多佇列）
- PCIe Gen4 主機板（可以測試高頻寬）

軟體：
- QEMU/KVM（可以模擬不同配置）
- SPDK（可以繞過 Kernel，直接存取 NVMe）

安裝：
git clone https://github.com/spdk/spdk
cd spdk
./scripts/pkgdep.sh
./configure
make
```

---

## 四、這個系列的特色

### 4.1 硬體感知的系統軟體視角

**不同於傳統教材**：

```
傳統教材：
- 專注於協議規範
- 理論為主
- 缺乏實際案例

本系列：
- 從系統軟體工程師的角度
- 理論 + 實測數據
- 大量真實案例
```

**範例**：

```
傳統教材講 NVMe：
"NVMe 支援 65,536 個佇列，每個佇列深度 65,536"

本系列講 NVMe：
"為什麼需要這麼多佇列？
- 單一佇列會成為瓶頸（鎖競爭）
- 多佇列可以分散到不同 CPU Core
- 實測：1 個佇列 vs 16 個佇列，IOPS 提升 8 倍

但實際上：
- 大部分應用只用 1-4 個佇列
- 超過 16 個佇列，收益遞減
- 需要根據工作負載調整"
```

---

### 4.2 完整的技術棧覆蓋

**從應用到硬體**：

```
Application
    ↓
System Call (read/write/io_uring)
    ↓
VFS (Virtual File System)
    ↓
File System (ext4/xfs/f2fs)
    ↓
Block Layer (I/O Scheduler)
    ↓
Device Driver (NVMe Driver)
    ↓
Host Interface (PCIe/NVMe)
    ↓
SSD Controller (FTL/ECC)
    ↓
NAND Flash

本系列涵蓋：Host Interface → SSD Controller → NAND Flash
```

---

### 4.3 實測數據與真實案例

**每篇文章都包含**：

1. **真實案例**
   - 來自我的實際工作經驗
   - 真實的問題與解決方案

2. **實測數據**
   - 使用 fio、perf、blktrace 等工具
   - 可重現的測試方法

3. **設計權衡**
   - 為什麼這樣設計？
   - 有什麼替代方案？
   - 各有什麼優缺點？

---

## 五、開始你的 Storage Architecture 之旅

### 5.1 學習目標

讀完這個系列，你將能夠：

**1. 理解存儲技術的演進**

- 從 HDD 到 SSD 的變革
- 從 SATA 到 NVMe 的演進
- 從 Block Storage 到 CXL 的未來

**2. 掌握 SSD 的內部機制**

- FTL 如何運作
- GC 如何影響效能
- ECC 如何保證可靠性

**3. 優化系統效能**

- 降低 I/O 延遲
- 提升吞吐量
- 延長 SSD 壽命

**4. 診斷存儲問題**

- 分析 SMART 資訊
- 使用效能分析工具
- 找出瓶頸根源

**5. 做出明智的技術決策**

- 選擇合適的存儲接口
- 設計高效的存儲架構
- 規劃容量與成本

---

### 5.2 下一步

**立即開始**：

如果你是：

- **系統軟體工程師**：直接跳到 [NVMe](04-nvme.md)
- **嵌入式工程師**：從 [SATA/AHCI](02-sata-ahci.md) 開始
- **效能優化工程師**：從 [PCIe](03-pcie.md) 開始
- **想系統學習**：按順序閱讀 Part 2

**實驗建議**：

```bash
# 1. 檢查你的 SSD 類型
lsblk -d -o name,rota,type
# rota=0：SSD
# rota=1：HDD

# 2. 如果是 NVMe SSD
sudo nvme list
sudo nvme smart-log /dev/nvme0n1

# 3. 如果是 SATA SSD
sudo smartctl -a /dev/sda

# 4. 簡單的效能測試
sudo fio --name=test --filename=/dev/nvme0n1 --rw=randread \
    --bs=4k --numjobs=1 --iodepth=32 --runtime=10 --time_based \
    --direct=1 --ioengine=libaio
```

---

## 六、總結

Storage Architecture 是一個迷人的領域，它連接了硬體與軟體，連接了理論與實踐。

這個系列的目標，不是讓你成為 SSD 控制器設計專家（那需要多年的硬體經驗），而是讓你：

1. **理解 SSD 的內部機制**，不再把它當成黑盒子
2. **掌握效能優化技巧**，寫出更快的程式
3. **學會診斷存儲問題**，快速定位根本原因
4. **做出明智的技術決策**，選擇合適的方案

讓我們開始這段旅程吧！

---

**下一篇預告**：

[SATA/AHCI - 傳統存儲接口的瓶頸](02-sata-ahci.md)

我們將探討：

- SATA 的歷史與演進
- AHCI 協議的設計
- 為什麼 SATA 無法發揮 SSD 的全部潛力
- NCQ (Native Command Queuing) 的限制

---

## 參考資料

1. "Understanding Modern Storage APIs", ACM Queue, 2019
2. "The Unwritten Contract of Solid State Drives", EuroSys 2017
3. NVMe Specification 1.4, NVM Express, Inc.
4. "Design Tradeoffs for SSD Performance", USENIX ATC 2008

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
