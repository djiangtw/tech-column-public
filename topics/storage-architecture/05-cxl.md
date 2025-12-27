# CXL - 記憶體與儲存的未來

**作者**: Danny Jiang  
**日期**: 2025-12-08  

---

## 前言：記憶體牆的困境

2020 年，我在一家 AI 公司負責訓練大型語言模型。我們的訓練伺服器配備了 8 張 NVIDIA A100 GPU，每張有 80 GB HBM2e 記憶體，總共 640 GB。

**問題來了**：

```
模型大小：GPT-3 級別，175B 參數
記憶體需求：
- FP16 參數：175B × 2 bytes = 350 GB
- 梯度：350 GB
- Optimizer 狀態（Adam）：350 GB × 2 = 700 GB
- 總需求：1,400 GB

可用記憶體：640 GB

差距：760 GB  ← 不夠！
```

**當時的解決方案**：

1. **Model Parallelism**：把模型切成多份，分散到多個 GPU
   - 問題：GPU 間通訊延遲高（PCIe/NVLink）

2. **Gradient Checkpointing**：重新計算中間結果，減少記憶體使用
   - 問題：訓練時間增加 30-40%

3. **Offload to CPU Memory**：把部分資料放到 CPU 記憶體
   - 問題：PCIe 頻寬不夠（16 GB/s），成為瓶頸

**我當時的想法**：「如果能有一種技術，讓 GPU 直接存取更大的記憶體池，而且延遲和頻寬都接近本地記憶體，該有多好？」

2021 年，**CXL (Compute Express Link)** 正式發布，提供了這個問題的解答。

本文將深入探討 CXL 的設計、應用，以及它如何改變記憶體和儲存的架構。

---

## 一、為什麼需要 CXL？

### 1.1 記憶體牆 (Memory Wall)

**問題 1：記憶體容量不足**

```
CPU 的記憶體需求：
- 資料庫：TB 級別
- AI 訓練：TB 級別
- In-Memory Computing：TB 級別

DDR5 的限制：
- 每個 DIMM：最大 128 GB
- 每個 CPU：最多 8-12 個 DIMM
- 總容量：1-1.5 TB

差距：需要更大的記憶體容量
```

**問題 2：記憶體頻寬不足**

```
CPU 的頻寬需求：
- 高效能運算：> 1 TB/s
- AI 推論：> 500 GB/s

DDR5 的頻寬：
- 每個 channel：~50 GB/s
- 8 channels：~400 GB/s

差距：需要更高的記憶體頻寬
```

**問題 3：記憶體成本高**

```
DDR5 的成本：
- 128 GB DIMM：~$500
- 1 TB 記憶體：~$4,000

Persistent Memory (Intel Optane) 的成本：
- 128 GB DIMM：~$200
- 1 TB 記憶體：~$1,600

差距：需要更便宜的記憶體方案
```

---

### 1.2 PCIe 的限制

**PCIe 的問題**：

1. **沒有 Cache Coherency**：
   - CPU 和裝置的 cache 不一致
   - 需要軟體手動管理（flush, invalidate）

2. **延遲高**：
   - PCIe TLP 的處理延遲：~100-200 ns
   - 相比 DDR5 的 ~50 ns，高了 2-4 倍

3. **不支援 Load/Store**：
   - 只能用 MMIO（Memory-Mapped I/O）
   - 不能直接用 CPU 的 load/store 指令存取

---

### 1.3 CXL 的解決方案

**CXL 的核心理念**：

1. **Cache Coherency**：CPU 和裝置的 cache 自動同步
2. **低延遲**：接近 DDR5 的延遲（~100 ns）
3. **高頻寬**：基於 PCIe 5.0/6.0（64-128 GB/s）
4. **記憶體擴展**：支援 TB 級別的記憶體池

**CXL 的三種協議**：

```
┌─────────────────────────────────────────────────┐
│  CXL.io                                         │
│  - 基於 PCIe                                    │
│  - 用於裝置初始化、配置、中斷                    │
│  - 相容 PCIe 裝置                               │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  CXL.cache                                      │
│  - 裝置可以 cache Host 的記憶體                 │
│  - 支援 cache coherency                         │
│  - 用於加速器（GPU, FPGA）                      │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  CXL.mem                                        │
│  - Host 可以存取裝置的記憶體                    │
│  - 支援 load/store 指令                         │
│  - 用於記憶體擴展                               │
└─────────────────────────────────────────────────┘
```

---

## 二、CXL 的架構

### 2.1 CXL 的分層設計

**CXL 的協議棧**：

```
┌─────────────────────────────────────────────────┐
│  Transaction Layer                              │
│  ┌──────────────┬──────────────┬──────────────┐ │
│  │  CXL.io      │  CXL.cache   │  CXL.mem     │ │
│  │  (PCIe TLP)  │  (D2H Req/   │  (H2D Req/   │ │
│  │              │   Rsp/Data)  │   Rsp/Data)  │ │
│  └──────────────┴──────────────┴──────────────┘ │
└─────────────────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────┐
│  Link Layer (Flit-based)                        │
│  - 68-byte Flit (CXL 1.x/2.0)                   │
│  - 256-byte Flit (CXL 3.0)                      │
│  - CRC, Retry, Flow Control                     │
└─────────────────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────┐
│  Physical Layer                                 │
│  - 基於 PCIe 5.0/6.0 PHY                        │
│  - 32 GT/s (CXL 2.0) / 64 GT/s (CXL 3.0)        │
└─────────────────────────────────────────────────┘
```

---

### 2.2 CXL.io：裝置管理

**CXL.io 的用途**：

- 裝置初始化和配置
- 中斷處理（MSI/MSI-X）
- DMA 傳輸
- 相容 PCIe 裝置

**CXL.io 的封包格式**：

```
┌────────────────────────────────────────────────┐
│  PCIe TLP (Transaction Layer Packet)           │
│  - Memory Read/Write                           │
│  - Configuration Read/Write                    │
│  - Message (Interrupt)                         │
└────────────────────────────────────────────────┘
```

**範例：CXL 裝置的初始化**：

```c
// Linux kernel: drivers/cxl/pci.c
static int cxl_pci_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
    struct cxl_dev_state *cxlds;
    int rc;
    
    // 1. 啟用 PCIe 裝置
    rc = pcim_enable_device(pdev);
    if (rc)
        return rc;
    
    // 2. 映射 MMIO
    cxlds->regs.hdm_decoder = devm_cxl_setup_hdm(cxlds);
    
    // 3. 識別 CXL 能力
    rc = cxl_setup_regs(pdev, CXL_REGLOC_RBI_MEMDEV, &map);
    
    // 4. 初始化 CXL.mem
    rc = cxl_mem_create_range_info(cxlds);
    
    return 0;
}
```

---

### 2.3 CXL.cache：裝置快取主機記憶體

**CXL.cache 的用途**：

讓裝置（如 GPU、FPGA）可以 cache Host 的記憶體，並保持 cache coherency。

**CXL.cache 的訊息類型**：

```c
// CXL.cache 的請求類型（Device to Host）
enum cxl_cache_opcode {
    CXL_CACHE_RdCurr    = 0x00,  // Read Current (讀取最新資料)
    CXL_CACHE_RdOwn     = 0x01,  // Read for Ownership (獨佔讀取)
    CXL_CACHE_RdShared  = 0x02,  // Read Shared (共享讀取)
    CXL_CACHE_RdAny     = 0x03,  // Read Any (任意讀取)
    CXL_CACHE_RdOwnNoData = 0x04, // Read for Ownership (不需要資料)
    CXL_CACHE_ItoMWr    = 0x05,  // Invalidate to Modified Write
    CXL_CACHE_WrCur     = 0x06,  // Write Current
    CXL_CACHE_ClFlush   = 0x07,  // Cache Line Flush
    CXL_CACHE_CleanEvict = 0x08, // Clean Evict
    CXL_CACHE_DirtyEvict = 0x09, // Dirty Evict
    CXL_CACHE_CleanEvictNoData = 0x0A, // Clean Evict (不需要資料)
};
```

**Cache Coherency 的流程**：

```
情境：GPU 要讀取 CPU 已經 cache 的資料

1. GPU 發送 RdShared 請求
   ↓
2. Host 的 Home Agent 收到請求
   ↓
3. Home Agent 檢查 CPU 的 cache
   ↓
4. CPU cache 有資料（Modified 狀態）
   ↓
5. CPU 將資料寫回記憶體（Write-back）
   ↓
6. CPU cache 狀態改為 Shared
   ↓
7. Home Agent 回傳資料給 GPU
   ↓
8. GPU cache 資料（Shared 狀態）
```

**範例：GPU 使用 CXL.cache**

```
傳統 PCIe 架構：
┌────────┐                    ┌────────┐
│  CPU   │                    │  GPU   │
│ Cache  │                    │ Cache  │
└────┬───┘                    └────┬───┘
     │                             │
     │  ┌─────────────────────┐    │
     └──┤  System Memory      ├────┘
        │  (DDR5)             │
        └─────────────────────┘

問題：CPU 和 GPU 的 cache 不一致
解決：軟體手動 flush/invalidate

CXL.cache 架構：
┌────────┐                    ┌────────┐
│  CPU   │◄──────────────────►│  GPU   │
│ Cache  │   Cache Coherency  │ Cache  │
└────┬───┘                    └────┬───┘
     │                             │
     │  ┌─────────────────────┐    │
     └──┤  System Memory      ├────┘
        │  (DDR5)             │
        └─────────────────────┘

優勢：硬體自動同步，無需軟體介入
```

---

### 2.4 CXL.mem：記憶體擴展

**CXL.mem 的用途**：

讓 Host 可以存取裝置的記憶體，就像存取本地 DDR 一樣。

**CXL.mem 的訊息類型**：

```c
// CXL.mem 的請求類型（Host to Device）
enum cxl_mem_opcode {
    CXL_MEM_RdMem       = 0x00,  // Memory Read
    CXL_MEM_RdMemNoSnp  = 0x01,  // Memory Read (No Snoop)
    CXL_MEM_WrMem       = 0x02,  // Memory Write
    CXL_MEM_WrMemNoSnp  = 0x03,  // Memory Write (No Snoop)
    CXL_MEM_WrMemPtl    = 0x04,  // Memory Write Partial
};
```

**CXL Memory Expander 的架構**：

```
┌─────────────────────────────────────────────────┐
│  Host (CPU)                                     │
│  ┌──────────────────────────────────────────┐   │
│  │  CPU Cores                               │   │
│  ├──────────────────────────────────────────┤   │
│  │  Memory Controller                       │   │
│  │  - DDR5 Channels (本地記憶體)            │   │
│  │  - CXL.mem Interface (擴展記憶體)        │   │
│  └──────────────┬───────────────────────────┘   │
└─────────────────┼───────────────────────────────┘
                  │
                  │ CXL 2.0/3.0
                  │
┌─────────────────▼───────────────────────────────┐
│  CXL Memory Expander                            │
│  ┌──────────────────────────────────────────┐   │
│  │  CXL Controller                          │   │
│  ├──────────────────────────────────────────┤   │
│  │  DDR5 Memory (256 GB - 1 TB)             │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

**記憶體位址空間**：

```
CPU 的記憶體位址空間：
┌─────────────────────────────────────────────────┐
│  0x0000_0000_0000 - 0x0000_00FF_FFFF            │
│  System Reserved (4 GB)                         │
├─────────────────────────────────────────────────┤
│  0x0000_0100_0000 - 0x0000_3FFF_FFFF            │
│  Local DDR5 (256 GB)                            │  ← 本地記憶體
├─────────────────────────────────────────────────┤
│  0x0000_4000_0000 - 0x0000_7FFF_FFFF            │
│  CXL Memory Expander 1 (256 GB)                 │  ← CXL 擴展
├─────────────────────────────────────────────────┤
│  0x0000_8000_0000 - 0x0000_BFFF_FFFF            │
│  CXL Memory Expander 2 (256 GB)                 │  ← CXL 擴展
└─────────────────────────────────────────────────┘
```

**Linux 的 CXL.mem 支援**：

```c
// Linux kernel: drivers/cxl/core/region.c
static int cxl_region_attach(struct cxl_region *cxlr,
                             struct cxl_endpoint_decoder *cxled, int pos)
{
    struct cxl_root_decoder *cxlrd = to_cxl_root_decoder(cxlr->dev.parent);
    struct cxl_memdev *cxlmd = cxled_to_memdev(cxled);

    // 1. 配置 HDM Decoder（Host-managed Device Memory）
    rc = cxl_region_setup_targets(cxlr);

    // 2. 建立記憶體映射
    rc = devm_cxl_add_dax_region(cxlr);

    // 3. 註冊到 NUMA node
    rc = cxl_region_perf_attrs_callback(cxlr);

    return 0;
}
```

---

## 三、CXL 的應用場景

### 3.1 記憶體池化 (Memory Pooling)

**傳統架構的問題**：

```
伺服器 1：
- CPU: 64 cores
- Memory: 512 GB
- 使用率: 30%  ← 浪費 70%

伺服器 2：
- CPU: 64 cores
- Memory: 512 GB
- 使用率: 95%  ← 記憶體不足

問題：記憶體無法在伺服器間共享
```

**CXL Memory Pooling 的解決方案**：

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Server 1    │  │  Server 2    │  │  Server 3    │
│  256 GB DDR  │  │  256 GB DDR  │  │  256 GB DDR  │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
              ┌──────────▼──────────┐
              │  CXL Memory Pool    │
              │  (2 TB)             │
              │  - 動態分配         │
              │  - 共享存取         │
              └─────────────────────┘

優勢：
- 記憶體使用率提升到 80-90%
- 減少記憶體採購成本
- 彈性分配記憶體
```

---

### 3.2 Persistent Memory (PMEM)

**CXL Persistent Memory 的架構**：

```
┌─────────────────────────────────────────────────┐
│  Host (CPU)                                     │
│  ┌──────────────────────────────────────────┐   │
│  │  Application                             │   │
│  ├──────────────────────────────────────────┤   │
│  │  File System (ext4-DAX, XFS-DAX)         │   │
│  ├──────────────────────────────────────────┤   │
│  │  DAX (Direct Access)                     │   │
│  └──────────────┬───────────────────────────┘   │
└─────────────────┼───────────────────────────────┘
                  │
                  │ CXL.mem
                  │
┌─────────────────▼───────────────────────────────┐
│  CXL Persistent Memory                          │
│  ┌──────────────────────────────────────────┐   │
│  │  CXL Controller                          │   │
│  ├──────────────────────────────────────────┤   │
│  │  Persistent Memory (NAND Flash)          │   │
│  │  - 容量：1-4 TB                          │   │
│  │  - 延遲：~1 μs                           │   │
│  │  - 頻寬：~10 GB/s                        │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

**優勢**：

1. **低延遲**：~1 μs（相比 NVMe 的 ~10 μs）
2. **高頻寬**：~10 GB/s（相比 NVMe 的 ~7 GB/s）
3. **Byte-addressable**：可以用 load/store 指令存取
4. **持久化**：資料不會因斷電而遺失

**應用場景**：

- 資料庫（Redis, RocksDB）
- 檔案系統快取
- Checkpoint/Snapshot

---

### 3.3 AI/ML 加速器

**問題**：GPU 的 HBM 記憶體容量有限

```
NVIDIA H100 GPU：
- HBM3 容量：80 GB
- HBM3 頻寬：3 TB/s

大型模型訓練：
- GPT-4 級別：~1 TB 參數
- 需要：12-16 張 GPU

問題：
- GPU 間通訊延遲高（NVLink: ~300 ns）
- 記憶體容量不足
```

**CXL 的解決方案**：

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  GPU 1       │  │  GPU 2       │  │  GPU 3       │
│  80 GB HBM   │  │  80 GB HBM   │  │  80 GB HBM   │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │ CXL.cache + CXL.mem
              ┌──────────▼──────────┐
              │  CXL Memory Pool    │
              │  (4 TB)             │
              │  - Cache Coherent   │
              │  - 低延遲 (~200 ns) │
              └─────────────────────┘

優勢：
- GPU 可以存取更大的記憶體池
- Cache coherency 減少資料同步開銷
- 延遲比 PCIe 低 2-3 倍
```

---

## 四、CXL 2.0 vs CXL 3.0

### 4.1 CXL 2.0 的特性

**CXL 2.0 的重點**：

1. **Memory Pooling**：支援多個 Host 共享記憶體池
2. **Switch Support**：支援 CXL Switch，可以連接多個裝置
3. **Persistent Memory**：支援 PMEM
4. **頻寬**：基於 PCIe 5.0（32 GT/s，64 GB/s）

**CXL 2.0 的架構**：

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Host 1  │  │  Host 2  │  │  Host 3  │
└─────┬────┘  └─────┬────┘  └─────┬────┘
      │             │             │
      └─────────────┼─────────────┘
                    │
         ┌──────────▼──────────┐
         │  CXL Switch         │
         └──────────┬──────────┘
                    │
      ┌─────────────┼─────────────┐
      │             │             │
┌─────▼────┐  ┌─────▼────┐  ┌─────▼────┐
│  CXL     │  │  CXL     │  │  CXL     │
│  Memory  │  │  Memory  │  │  Accel   │
└──────────┘  └──────────┘  └──────────┘
```

---

### 4.2 CXL 3.0 的改進

**CXL 3.0 的新特性**：

1. **更高頻寬**：基於 PCIe 6.0（64 GT/s，128 GB/s）
2. **更大的 Flit**：256-byte Flit（相比 2.0 的 68-byte）
3. **Fabric Support**：支援大規模 CXL Fabric
4. **Peer-to-Peer**：裝置間可以直接通訊

**CXL 3.0 的 Fabric 架構**：

```
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│ Host 1 │ │ Host 2 │ │ Host 3 │ │ Host 4 │
└───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘
    │          │          │          │
    └──────────┼──────────┼──────────┘
               │          │
        ┌──────▼──────────▼──────┐
        │  CXL Fabric (Mesh)     │
        │  - Multi-level Switch  │
        │  - Global Address Space│
        └──────┬──────────┬──────┘
               │          │
    ┌──────────┼──────────┼──────────┐
    │          │          │          │
┌───▼────┐ ┌───▼────┐ ┌───▼────┐ ┌───▼────┐
│ Memory │ │ Memory │ │ Accel  │ │ Accel  │
└────────┘ └────────┘ └────────┘ └────────┘
```

**性能對比**：

| 特性 | CXL 2.0 | CXL 3.0 |
|------|---------|---------|
| 基於 PCIe | 5.0 | 6.0 |
| 速率 | 32 GT/s | 64 GT/s |
| 頻寬 (x16) | 64 GB/s | 128 GB/s |
| Flit 大小 | 68 bytes | 256 bytes |
| 延遲 | ~200 ns | ~150 ns |

---

## 五、實戰：CXL Memory Expander

### 5.1 測試環境

- **CPU**: Intel Xeon Sapphire Rapids (4th Gen, 支援 CXL 1.1)
- **主機板**: Supermicro X13 (CXL-ready)
- **CXL Memory**: Samsung CXL Memory Expander (256 GB)
- **作業系統**: Ubuntu 22.04 LTS
- **Kernel**: 6.2.0 (with CXL support)

---

### 5.2 設定 CXL Memory

**檢查 CXL 裝置**：

```bash
$ lspci | grep CXL
b1:00.0 Memory controller: Intel Corporation Device 0d93 (CXL Memory Expander)

$ cxl list
[
  {
    "memdev":"mem0",
    "pmem_size":"256.00 GiB (274.88 GB)",
    "ram_size":0,
    "serial":"0x123456789abc",
    "host":"0000:b1:00.0"
  }
]
```

**建立 CXL Region**：

```bash
# 1. 建立 region
$ cxl create-region -d decoder0.0 -w 1 mem0

# 2. 啟用 region
$ cxl enable-region region0

# 3. 查看記憶體
$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 ... 63
node 0 size: 262144 MB
node 0 free: 245678 MB
node 1 cpus:                    ← CXL Memory (no CPU)
node 1 size: 262144 MB          ← 256 GB
node 1 free: 262144 MB
```

---

### 5.3 性能測試

**測試 1：記憶體頻寬**

```bash
# 測試本地 DDR5
$ numactl --membind=0 stream
STREAM Triad: 400 GB/s

# 測試 CXL Memory
$ numactl --membind=1 stream
STREAM Triad: 50 GB/s  ← 比 DDR5 慢 8 倍
```

**測試 2：記憶體延遲**

```bash
# 測試本地 DDR5
$ numactl --membind=0 lat_mem_rd 256 128
Latency: 80 ns

# 測試 CXL Memory
$ numactl --membind=1 lat_mem_rd 256 128
Latency: 180 ns  ← 比 DDR5 高 2.25 倍
```

**觀察**：

- CXL Memory 的頻寬和延遲都比本地 DDR5 差
- 但仍然比 NVMe SSD 快很多（延遲 ~10 μs）
- 適合用於不常存取的資料（cold data）

---

## 六、總結

### CXL 的優勢

✅ **記憶體擴展**：支援 TB 級別的記憶體池
✅ **Cache Coherency**：硬體自動同步 cache
✅ **低延遲**：~100-200 ns（接近 DDR5）
✅ **高頻寬**：64-128 GB/s（基於 PCIe 5.0/6.0）
✅ **彈性**：支援 Memory Pooling、PMEM、加速器

### CXL vs PCIe vs DDR5

| 特性 | DDR5 | CXL 2.0/3.0 | PCIe 5.0 | NVMe |
|------|------|-------------|----------|------|
| 延遲 | 50 ns | 100-200 ns | 100-200 ns | 10 μs |
| 頻寬 | 400 GB/s | 64-128 GB/s | 64 GB/s | 7 GB/s |
| Cache Coherency | ✅ | ✅ | ❌ | ❌ |
| Load/Store | ✅ | ✅ | ❌ | ❌ |
| 容量 | 1-2 TB | 4-16 TB | N/A | 1-4 TB |
| 成本 | 高 | 中 | N/A | 低 |

### 未來展望

- **CXL 3.0 普及**：2025-2026 年，頻寬達到 128 GB/s
- **CXL Fabric**：大規模資料中心部署
- **CXL + AI**：解決 AI 訓練的記憶體瓶頸
- **CXL + Storage**：Computational Storage 的新架構

**CXL 正在改變記憶體和儲存的邊界**，讓我們可以在延遲、頻寬、容量、成本之間找到更好的平衡點。

---

## 參考資料

1. CXL Consortium: <https://www.computeexpresslink.org/>
2. CXL Specification 3.0: <https://www.computeexpresslink.org/download-the-specification>
3. Linux kernel CXL driver: <https://www.kernel.org/doc/html/latest/driver-api/cxl/>
4. Intel CXL Memory Expander: <https://www.intel.com/content/www/us/en/products/docs/memory-storage/optane-persistent-memory/cxl-memory-expander-brief.html>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
