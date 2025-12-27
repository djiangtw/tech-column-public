# Advanced Topics - 存儲技術的未來

**作者**: Danny Jiang  
**日期**: 2025-12-08  

---

## 前言：一次改變認知的技術演示

2020 年，我參加了一場存儲技術研討會。會上，一家新創公司展示了他們的 **Computational Storage** 產品：

**傳統方案**：

```
任務：在 10 TB 的日誌檔案中搜尋特定模式

流程：
1. 從 SSD 讀取資料到 Host Memory
2. CPU 執行正則表達式匹配
3. 將結果寫回 SSD

效能：
- 讀取頻寬：3 GB/s
- 處理時間：10 TB / 3 GB/s = 3,333 秒 ≈ 55 分鐘
- CPU 使用率：100%（16 Cores）
- 記憶體使用：64 GB
```

**Computational Storage 方案**：

```
任務：相同的搜尋任務

流程：
1. Host 發送搜尋命令到 SSD
2. SSD 內部的 FPGA 執行正則表達式匹配
3. 只將匹配結果傳回 Host

效能：
- 內部處理頻寬：20 GB/s（不受 PCIe 限制）
- 處理時間：10 TB / 20 GB/s = 500 秒 ≈ 8 分鐘
- CPU 使用率：< 5%（只處理結果）
- 記憶體使用：< 1 GB
- 加速比：6.7x
```

**我的第一反應**：「這不可能！」

**演示結果**：真的做到了。

這次經驗讓我意識到：**存儲不再只是「存儲」，它正在變成「計算 + 存儲」的融合體**。

本文將探討三個正在改變存儲架構的進階主題：

1. **Computational Storage**：在 SSD 內部執行計算
2. **NVMe-oF**：透過網路存取 NVMe SSD
3. **Persistent Memory**：DRAM 與 SSD 之間的新選擇

---

## 一、Computational Storage - 計算與存儲的融合

### 1.1 為什麼需要 Computational Storage？

**問題：資料移動的成本**

```
現代系統的瓶頸：

CPU 計算速度：
- 單核：~100 GFLOPS
- 16 核：~1,600 GFLOPS

記憶體頻寬：
- DDR5-4800：~76 GB/s

SSD 頻寬：
- PCIe Gen4 x4：~7 GB/s

問題：
- 資料從 SSD 到 CPU 的移動成本很高
- PCIe 頻寬成為瓶頸
- CPU 花費大量時間等待資料
```

**能量消耗分析**：

```
資料移動的能量成本（45nm 製程）：

操作                    能量消耗
─────────────────────────────────
32-bit 加法             0.1 pJ
32-bit 乘法             3.1 pJ
從 L1 Cache 讀取        0.5 pJ
從 L2 Cache 讀取        2.5 pJ
從 DRAM 讀取            640 pJ   ← 比計算貴 6,400 倍
從 SSD 讀取（PCIe）     10,000 pJ ← 比計算貴 100,000 倍

結論：
- 資料移動比計算貴得多
- 減少資料移動 = 降低能耗 + 提升效能
```

---

### 1.2 Computational Storage 的架構

**基本概念**：

```
傳統架構：
┌─────────┐      PCIe      ┌─────────┐
│   CPU   │ ←────────────→ │   SSD   │
│ (計算)  │   資料移動      │ (存儲)  │
└─────────┘                └─────────┘

Computational Storage：
┌─────────┐      PCIe      ┌─────────────────┐
│   CPU   │ ←────────────→ │   Smart SSD     │
│ (協調)  │   命令 + 結果   │ ┌─────┬───────┐ │
└─────────┘                │ │FPGA │ NAND  │ │
                           │ │(計算)│(存儲) │ │
                           │ └─────┴───────┘ │
                           └─────────────────┘
```

**實作方式**：

```
方式 1: FPGA-based
- 優勢：靈活、可重新配置
- 劣勢：功耗較高、成本較高
- 適用：需要客製化演算法

方式 2: ASIC-based
- 優勢：效能高、功耗低
- 劣勢：不可重新配置
- 適用：固定的演算法（如壓縮、加密）

方式 3: ARM Core-based
- 優勢：可程式化、生態系統成熟
- 劣勢：效能不如 FPGA/ASIC
- 適用：通用計算任務
```

---

### 1.3 實際應用案例

**案例 1：資料庫查詢加速**

```
任務：在 1 TB 的資料表中執行 SELECT 查詢

傳統方案：
1. 從 SSD 讀取整個資料表到 Memory
2. CPU 執行 WHERE 條件過濾
3. 返回結果

資料流：
SSD → PCIe → DRAM → CPU → DRAM → PCIe → Network
傳輸量：1 TB
時間：1 TB / 7 GB/s = 143 秒

Computational Storage 方案：
1. Host 發送 WHERE 條件到 SSD
2. SSD 內部的 FPGA 執行過濾
3. 只傳回符合條件的資料（假設 1%）

資料流：
SSD (內部處理) → PCIe → Network
傳輸量：10 GB
時間：10 GB / 7 GB/s = 1.4 秒

加速比：143 / 1.4 = 102x
```

**案例 2：影片轉碼**

```
任務：將 100 個 4K 影片從 H.264 轉碼為 H.265

傳統方案：
1. 從 SSD 讀取影片到 Memory
2. CPU/GPU 執行轉碼
3. 寫回 SSD

瓶頸：
- PCIe 頻寬（讀取 + 寫入）
- CPU/GPU 計算能力

Computational Storage 方案：
1. Host 發送轉碼命令到 SSD
2. SSD 內部的 ASIC 執行轉碼
3. 直接寫回 NAND Flash

優勢：
- 不佔用 PCIe 頻寬
- 不佔用 CPU/GPU
- 可以並行處理多個影片
```

**案例 3：機器學習推論**

```
任務：在 10 TB 的圖片資料集上執行物件偵測

傳統方案：
1. 從 SSD 讀取圖片到 GPU Memory
2. GPU 執行推論
3. 將結果寫回 SSD

瓶頸：
- PCIe 頻寬（GPU 與 SSD 競爭）
- GPU Memory 容量限制

Computational Storage 方案：
1. 將推論模型載入到 SSD 的 FPGA
2. SSD 內部執行推論
3. 只傳回偵測結果

優勢：
- GPU 可以專注於訓練
- 降低 PCIe 頻寬需求
- 可以處理超大資料集
```

---

### 1.4 標準化：SNIA Computational Storage TWG

**SNIA (Storage Networking Industry Association)** 成立了 Computational Storage Technical Work Group，定義標準：

**Computational Storage Processor (CSP)**：

```
CSP 的能力：
1. 資料處理（Data Processing）
   - 壓縮 / 解壓縮
   - 加密 / 解密
   - 雜湊計算

2. 資料轉換（Data Transformation）
   - 格式轉換
   - 編碼 / 解碼

3. 資料分析（Data Analytics）
   - 過濾 / 聚合
   - 模式匹配
   - 統計計算

4. 機器學習（Machine Learning）
   - 推論
   - 特徵提取
```

**Computational Storage Drive (CSD)**：

```
CSD = SSD + CSP

介面：
- NVMe（擴展命令集）
- 自定義命令（Vendor Specific Commands）

範例：壓縮命令
nvme_cmd.opcode = NVME_CMD_COMPRESS;
nvme_cmd.nsid = namespace_id;
nvme_cmd.cdw10 = input_lba;
nvme_cmd.cdw11 = output_lba;
nvme_cmd.cdw12 = length;
nvme_cmd.cdw13 = compression_algorithm;  // 0=LZ4, 1=ZSTD
```

---

## 二、NVMe-oF - 透過網路存取 NVMe

### 2.1 為什麼需要 NVMe-oF？

**問題：本地存儲的限制**

```
傳統架構：
┌─────────────────────────────────┐
│         Server                  │
│  ┌─────┐    PCIe    ┌────────┐ │
│  │ CPU │ ←────────→ │ NVMe   │ │
│  └─────┘            │ SSD    │ │
│                     └────────┘ │
└─────────────────────────────────┘

限制：
1. 容量受限於 PCIe 插槽數量
2. 無法共享存儲資源
3. 故障時資料無法存取
4. 擴展需要停機
```

**解決方案：NVMe over Fabrics (NVMe-oF)**

```
NVMe-oF 架構：
┌─────────┐   Network   ┌─────────────────┐
│ Server  │ ←─────────→ │ Storage Array   │
│  ┌───┐  │   NVMe-oF   │ ┌────┬────┬────┐│
│  │CPU│  │             │ │SSD │SSD │SSD ││
│  └───┘  │             │ └────┴────┴────┘│
└─────────┘             └─────────────────┘

優勢：
1. 容量不受本地限制
2. 多台 Server 共享存儲
3. 高可用性（Failover）
4. 線上擴展
```

---

### 2.2 NVMe-oF 的傳輸協議

**支援的 Fabric 類型**：

**1. NVMe over RDMA (RoCE / InfiniBand)**

```
特性：
- 延遲：~10 μs（接近本地 NVMe）
- 頻寬：100 Gbps+
- CPU 使用率：低（RDMA 卸載）

架構：
Host                    Target
┌────────┐   RDMA    ┌────────┐
│ NVMe   │ ←───────→ │ NVMe   │
│ Driver │  Queue    │ Target │
└────────┘  Pair     └────────┘
    ↓                     ↓
┌────────┐           ┌────────┐
│ RDMA   │           │ RDMA   │
│ NIC    │           │ NIC    │
└────────┘           └────────┘

優勢：
- 零拷貝（Zero-copy）
- Kernel bypass
- 最低延遲

劣勢：
- 需要特殊網卡（RDMA NIC）
- 成本較高
```

**2. NVMe over TCP**

```
特性：
- 延遲：~50 μs
- 頻寬：10-100 Gbps
- CPU 使用率：中等

架構：
Host                    Target
┌────────┐    TCP    ┌────────┐
│ NVMe   │ ←───────→ │ NVMe   │
│ Driver │           │ Target │
└────────┘           └────────┘
    ↓                     ↓
┌────────┐           ┌────────┐
│ TCP/IP │           │ TCP/IP │
│ Stack  │           │ Stack  │
└────────┘           └────────┘

優勢：
- 使用標準網卡
- 成本低
- 相容性好

劣勢：
- 延遲較高
- CPU 開銷較大
```

**3. NVMe over Fibre Channel (FC-NVMe)**

```
特性：
- 延遲：~20 μs
- 頻寬：32 Gbps（FC Gen 6）
- CPU 使用率：低

優勢：
- 企業級可靠性
- 成熟的生態系統
- 支援現有 FC 基礎設施

劣勢：
- 成本高
- 頻寬不如 RDMA
```

---

### 2.3 效能比較

**延遲對比**：

```
測試：4KB 隨機讀取

本地 NVMe SSD：
- 延遲：~50 μs
- IOPS：200,000

NVMe-oF over RDMA：
- 延遲：~60 μs  ← 只增加 10 μs
- IOPS：180,000
- 開銷：20%

NVMe-oF over TCP：
- 延遲：~100 μs  ← 增加 50 μs
- IOPS：100,000
- 開銷：100%

iSCSI（傳統方案）：
- 延遲：~500 μs  ← 增加 450 μs
- IOPS：20,000
- 開銷：900%

結論：
- NVMe-oF over RDMA 接近本地效能
- NVMe-oF over TCP 是成本與效能的平衡
- 遠優於傳統 iSCSI
```

**頻寬對比**：

```
測試：128KB 循序讀取

本地 NVMe SSD：
- 頻寬：7 GB/s（PCIe Gen4 x4）

NVMe-oF over RDMA (100 Gbps)：
- 頻寬：12 GB/s  ← 超過本地！
- 原因：可以聚合多顆 SSD

NVMe-oF over TCP (25 Gbps)：
- 頻寬：3 GB/s
- 瓶頸：網路頻寬

結論：
- RDMA 可以超越本地 PCIe 頻寬
- TCP 受限於網路頻寬
```

---

### 2.4 實際應用案例

**案例 1：虛擬化環境**

```
問題：
- 每台 VM 需要獨立的存儲
- 本地 SSD 無法共享
- VM 遷移需要複製資料

解決方案：NVMe-oF 共享存儲

架構：
┌─────────┐  ┌─────────┐  ┌─────────┐
│ Host 1  │  │ Host 2  │  │ Host 3  │
│ ┌──┬──┐ │  │ ┌──┬──┐ │  │ ┌──┬──┐ │
│ │VM│VM│ │  │ │VM│VM│ │  │ │VM│VM│ │
│ └──┴──┘ │  │ └──┴──┘ │  │ └──┴──┘ │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     └────────────┼────────────┘
                  │ NVMe-oF
          ┌───────┴────────┐
          │ Storage Array  │
          │ ┌────┬────┬────┐
          │ │SSD │SSD │SSD │
          │ └────┴────┴────┘
          └────────────────┘

優勢：
- VM 可以在任何 Host 上啟動
- Live Migration 不需要複製資料
- 集中管理存儲資源
```

**案例 2：容器化應用**

```
問題：
- Kubernetes Pod 需要持久化存儲
- 本地存儲無法跨節點

解決方案：NVMe-oF CSI Driver

apiVersion: v1
kind: PersistentVolume
metadata:
  name: nvmeof-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  csi:
    driver: nvmeof.csi.k8s.io
    volumeHandle: nvme-subsys-1
    volumeAttributes:
      nqn: nqn.2024-01.io.nvmeof:subsys1
      targetAddr: 192.168.1.100
      transport: tcp

優勢：
- Pod 可以在任何 Node 上調度
- 高效能持久化存儲
- 動態 Provisioning
```

**案例 3：高效能運算 (HPC)**

```
問題：
- 多個計算節點需要存取共享資料集
- 傳統 NFS 效能不足

解決方案：NVMe-oF 並行存取

架構：
┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐
│Node1│  │Node2│  │Node3│  │Node4│
└──┬──┘  └──┬──┘  └──┬──┘  └──┬──┘
   │        │        │        │
   └────────┴────────┴────────┘
            │ 100 Gbps RDMA
    ┌───────┴────────┐
    │ NVMe-oF Target │
    │ ┌────────────┐ │
    │ │ 16x NVMe   │ │
    │ │ SSD Array  │ │
    │ └────────────┘ │
    └────────────────┘

效能：
- 聚合頻寬：100 GB/s
- 延遲：< 20 μs
- 遠優於 NFS（~1 GB/s, ~1 ms）
```

---

## 三、Persistent Memory - DRAM 與 SSD 之間

### 3.1 記憶體階層的演進

**傳統記憶體階層**：

```
                速度      容量      成本      持久性
L1 Cache        ~1 ns     32 KB     極高      ✗
L2 Cache        ~3 ns     256 KB    極高      ✗
L3 Cache        ~10 ns    32 MB     高        ✗
DRAM            ~100 ns   128 GB    中        ✗
SSD             ~100 μs   4 TB      低        ✓

問題：
- DRAM 與 SSD 之間有巨大的延遲差距（1,000x）
- DRAM 不持久，斷電資料丟失
- SSD 延遲太高，無法當作記憶體使用
```

**Persistent Memory (PMEM) 填補空白**：

```
                速度      容量      成本      持久性
L1 Cache        ~1 ns     32 KB     極高      ✗
L2 Cache        ~3 ns     256 KB    極高      ✗
L3 Cache        ~10 ns    32 MB     高        ✗
DRAM            ~100 ns   128 GB    中        ✗
PMEM            ~300 ns   512 GB    中        ✓  ← 新增
SSD             ~100 μs   4 TB      低        ✓

優勢：
- 延遲只有 SSD 的 1/300
- 容量比 DRAM 大
- 持久化，斷電不丟失
```

---

### 3.2 Persistent Memory 技術

**Intel Optane DC Persistent Memory**：

```
技術：3D XPoint
- 非揮發性記憶體
- Byte-addressable（可以像 DRAM 一樣存取）
- 延遲：~300 ns（讀取），~1 μs（寫入）
- 耐久性：~10^7 次寫入（比 NAND Flash 低）

介面：
- DDR4 DIMM 插槽
- 與 DRAM 混合使用

容量：
- 單條：128 GB, 256 GB, 512 GB
- 遠大於 DRAM（單條最大 64 GB）
```

**使用模式**：

**模式 1: Memory Mode（記憶體模式）**

```
架構：
┌─────────────────────────────────┐
│         Application             │
└────────────┬────────────────────┘
             │ Load/Store
┌────────────┴────────────────────┐
│      Operating System           │
│  ┌──────────────────────────┐  │
│  │   DRAM (Cache)           │  │ ← 透明快取
│  │   16 GB                  │  │
│  └──────────┬───────────────┘  │
│             │                   │
│  ┌──────────┴───────────────┐  │
│  │   PMEM (Main Memory)     │  │
│  │   512 GB                 │  │
│  └──────────────────────────┘  │
└─────────────────────────────────┘

特性：
- DRAM 作為 PMEM 的快取
- 應用程式無需修改
- 總容量：512 GB
- 效能：接近 DRAM（熱資料在 DRAM）

使用場景：
- 需要大容量記憶體的應用
- 不需要持久化
```

**模式 2: App Direct Mode（應用直接模式）**

```
架構：
┌─────────────────────────────────┐
│         Application             │
│  ┌──────────┐  ┌──────────────┐ │
│  │ Volatile │  │ Persistent   │ │
│  │ Data     │  │ Data         │ │
│  └────┬─────┘  └──────┬───────┘ │
└───────┼────────────────┼─────────┘
        │                │
┌───────┴────┐   ┌───────┴────────┐
│   DRAM     │   │     PMEM       │
│   128 GB   │   │     512 GB     │
└────────────┘   └────────────────┘

特性：
- DRAM 和 PMEM 分別管理
- 應用程式需要修改（使用 PMEM API）
- PMEM 資料持久化

使用場景：
- 資料庫（持久化 Buffer Pool）
- 鍵值存儲（持久化 Hash Table）
- 檔案系統（持久化 Metadata）
```

---

### 3.3 程式設計模型

**PMDK (Persistent Memory Development Kit)**：

**範例 1：持久化變數**

```c
#include <libpmemobj.h>

// 定義持久化結構
struct my_data {
    uint64_t counter;
    char name[64];
};

// 開啟 PMEM Pool
PMEMobjpool *pop = pmemobj_open("/mnt/pmem/mypool", LAYOUT_NAME);

// 分配持久化物件
PMEMoid root = pmemobj_root(pop, sizeof(struct my_data));
struct my_data *data = pmemobj_direct(root);

// 原子更新（保證 Crash Consistency）
TX_BEGIN(pop) {
    TX_ADD(root);
    data->counter++;
    strcpy(data->name, "new_value");
} TX_END

// 即使系統當機，counter 和 name 的更新要麼都成功，要麼都失敗
```

**範例 2：持久化 Hash Table**

```c
#include <libpmemobj.h>

// 使用 PMDK 的 Hash Table
TOID(struct hashmap_tx) map;

// 插入
TX_BEGIN(pop) {
    hm_tx_insert(pop, map, key, value);
} TX_END

// 查詢
value = hm_tx_get(pop, map, key);

// 刪除
TX_BEGIN(pop) {
    hm_tx_remove(pop, map, key);
} TX_END

優勢：
- 斷電後資料不丟失
- 不需要 WAL (Write-Ahead Log)
- 效能遠優於 SSD
```

---

### 3.4 實際應用案例

**案例 1：Redis on PMEM**

```
傳統 Redis：
- 資料存在 DRAM
- 持久化：RDB (Snapshot) 或 AOF (Append-Only File)
- 問題：
  - RDB：資料可能丟失（最後一次 Snapshot 之後的資料）
  - AOF：效能開銷大（每次寫入都要寫檔案）

Redis on PMEM：
- 資料直接存在 PMEM
- 不需要 RDB 或 AOF
- 斷電後資料不丟失

效能對比：
                    傳統 Redis    Redis on PMEM
SET 延遲            ~100 μs       ~150 μs
GET 延遲            ~50 μs        ~80 μs
持久化開銷          50-200%       0%
恢復時間            分鐘級        秒級

優勢：
- 持久化無開銷
- 快速恢復
- 容量更大
```

**案例 2：資料庫 Buffer Pool**

```
傳統資料庫：
- Buffer Pool 在 DRAM
- 斷電後需要從 SSD 重新載入
- 冷啟動效能差

使用 PMEM：
- Buffer Pool 在 PMEM
- 斷電後資料仍在
- 重啟後立即可用

MySQL on PMEM：
- 冷啟動時間：10 分鐘 → 10 秒
- 查詢延遲（冷啟動後）：100 ms → 1 ms
```

**案例 3：檔案系統 Metadata**

```
傳統檔案系統（ext4）：
- Metadata 在 DRAM（Cache）
- 定期 Flush 到 SSD
- Crash 後需要 fsck（可能很慢）

PMEM 檔案系統（NOVA）：
- Metadata 直接在 PMEM
- 原子更新（Copy-on-Write）
- Crash 後立即可用，無需 fsck

效能：
                    ext4          NOVA (PMEM)
Metadata 操作       ~10 μs        ~1 μs
Crash 恢復時間      分鐘級        < 1 秒
```

---

## 四、三大技術的對比與展望

### 4.1 技術對比

```
                Computational    NVMe-oF         Persistent
                Storage                          Memory
─────────────────────────────────────────────────────────────
目標            減少資料移動      存儲池化        填補延遲空白
主要優勢        降低頻寬需求      靈活性          低延遲持久化
主要劣勢        可程式性受限      網路開銷        容量受限
成熟度          早期              成熟            中期
成本            高                中              高
適用場景        大資料分析        雲端/虛擬化      記憶體資料庫
```

---

### 4.2 未來趨勢

**1. Computational Storage 的演進**

```
當前：
- 固定功能（壓縮、加密）
- 有限的可程式性

未來：
- 完全可程式（類似 GPU）
- 支援複雜演算法（ML 推論、圖計算）
- 標準化 API（OpenCL、CUDA-like）
```

**2. NVMe-oF 的普及**

```
當前：
- 主要用於企業級
- 需要特殊網卡（RDMA）

未來：
- TCP 效能提升（Kernel bypass）
- 更低成本
- 成為雲端存儲標準
```

**3. Persistent Memory 的發展**

```
當前：
- Intel Optane（已停產）
- 容量有限

未來：
- 新技術（MRAM、ReRAM、PCM）
- 更大容量
- 更低成本
- 可能取代部分 DRAM
```

---

## 五、給系統軟體工程師的建議

### 5.1 如何選擇技術

**決策樹**：

```
問題：如何優化存儲系統？

1. 瓶頸在哪裡？
   ├─ PCIe 頻寬不足
   │  └─ 考慮 Computational Storage
   │     （在 SSD 內部處理資料）
   │
   ├─ 需要共享存儲
   │  └─ 考慮 NVMe-oF
   │     （透過網路存取）
   │
   └─ 需要低延遲持久化
      └─ 考慮 Persistent Memory
         （DRAM 與 SSD 之間）

2. 預算如何？
   ├─ 預算充足
   │  └─ 選擇最適合的技術
   │
   └─ 預算有限
      └─ 優先考慮 NVMe-oF over TCP
         （成本最低）
```

---

### 5.2 實驗建議

**Computational Storage**：

```bash
# 目前沒有開源的 Computational Storage 硬體
# 但可以用 FPGA 開發板模擬

# 使用 SPDK 的 Bdev 模組
git clone https://github.com/spdk/spdk
cd spdk
./scripts/pkgdep.sh
./configure
make

# 實作自定義的 Bdev（模擬 Computational Storage）
# 在 Bdev 層執行資料處理
```

**NVMe-oF**：

```bash
# 安裝 nvme-cli
sudo apt install nvme-cli

# Target 端（提供存儲）
sudo modprobe nvmet
sudo modprobe nvmet-tcp

# 建立 NVMe Subsystem
sudo mkdir /sys/kernel/config/nvmet/subsystems/nvme-test
echo 1 | sudo tee /sys/kernel/config/nvmet/subsystems/nvme-test/attr_allow_any_host

# 建立 Namespace
sudo mkdir /sys/kernel/config/nvmet/subsystems/nvme-test/namespaces/1
echo /dev/nvme0n1 | sudo tee /sys/kernel/config/nvmet/subsystems/nvme-test/namespaces/1/device_path
echo 1 | sudo tee /sys/kernel/config/nvmet/subsystems/nvme-test/namespaces/1/enable

# 建立 Port
sudo mkdir /sys/kernel/config/nvmet/ports/1
echo 192.168.1.100 | sudo tee /sys/kernel/config/nvmet/ports/1/addr_traddr
echo tcp | sudo tee /sys/kernel/config/nvmet/ports/1/addr_trtype
echo 4420 | sudo tee /sys/kernel/config/nvmet/ports/1/addr_trsvcid
echo ipv4 | sudo tee /sys/kernel/config/nvmet/ports/1/addr_adrfam

# 連接 Subsystem 到 Port
sudo ln -s /sys/kernel/config/nvmet/subsystems/nvme-test \
    /sys/kernel/config/nvmet/ports/1/subsystems/nvme-test

# Host 端（使用存儲）
sudo modprobe nvme-tcp
sudo nvme discover -t tcp -a 192.168.1.100 -s 4420
sudo nvme connect -t tcp -n nvme-test -a 192.168.1.100 -s 4420

# 測試效能
sudo fio --name=test --filename=/dev/nvme1n1 --rw=randread \
    --bs=4k --numjobs=4 --iodepth=32 --runtime=60 --time_based \
    --direct=1 --ioengine=libaio
```

**Persistent Memory**：

```bash
# 如果沒有實體 PMEM，可以用 DRAM 模擬
# 在 Kernel 啟動參數加入：
# memmap=4G!8G  （保留 8G-12G 的 DRAM 作為 PMEM）

# 建立 PMEM Namespace
sudo ndctl create-namespace --mode=fsdax --region=region0

# 掛載 PMEM 檔案系統
sudo mkfs.ext4 /dev/pmem0
sudo mount -o dax /dev/pmem0 /mnt/pmem

# 使用 PMDK
git clone https://github.com/pmem/pmdk
cd pmdk
make
sudo make install

# 測試程式
cat > test_pmem.c << 'EOF'
#include <libpmemobj.h>
#include <stdio.h>

POBJ_LAYOUT_BEGIN(test);
POBJ_LAYOUT_ROOT(test, struct root);
POBJ_LAYOUT_END(test);

struct root {
    uint64_t counter;
};

int main() {
    PMEMobjpool *pop = pmemobj_create("/mnt/pmem/pool",
        POBJ_LAYOUT_NAME(test), PMEMOBJ_MIN_POOL, 0666);

    TOID(struct root) root = POBJ_ROOT(pop, struct root);

    TX_BEGIN(pop) {
        TX_ADD(root);
        D_RW(root)->counter++;
    } TX_END

    printf("Counter: %lu\n", D_RO(root)->counter);

    pmemobj_close(pop);
    return 0;
}
EOF

gcc test_pmem.c -lpmemobj -o test_pmem
./test_pmem
```

---

## 六、總結

### 6.1 核心要點回顧

**Computational Storage**：

- 在 SSD 內部執行計算，減少資料移動
- 適用於大資料分析、影片轉碼、ML 推論
- 挑戰：可程式性、標準化

**NVMe-oF**：

- 透過網路存取 NVMe SSD，實現存儲池化
- RDMA 延遲 ~10 μs，TCP 延遲 ~50 μs
- 適用於雲端、虛擬化、HPC

**Persistent Memory**：

- 填補 DRAM 與 SSD 之間的延遲空白
- 延遲 ~300 ns，持久化
- 適用於記憶體資料庫、鍵值存儲

---

### 6.2 下一篇預告

本系列的最後三篇文章將探討實際應用：

**Part 5: Real-world Applications**

- **Database Optimization**：資料庫如何利用 SSD 特性
- **AI/ML Workloads**：訓練資料的儲存優化
- **Cloud Storage**：雲端環境中的 SSD 管理

---

## 參考資料

1. "Computational Storage: Where Are We Today?", SNIA, 2021
2. "NVMe over Fabrics Specification 1.1", NVM Express, 2021
3. "Intel Optane DC Persistent Memory Architecture", Intel, 2019
4. "PMDK: Persistent Memory Development Kit", pmem.io
5. "The Case for Computational Storage", ACM Queue, 2020

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
