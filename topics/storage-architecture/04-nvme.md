# NVMe - 為 SSD 而生的協議

**作者**: Danny Jiang  
**日期**: 2025-12-08  

---

## 前言：一次性能調校的頓悟

2018 年，我在一家雲端服務公司負責資料庫性能優化。我們的 MySQL 資料庫運行在配備 SATA SSD 的伺服器上，但隨著用戶增長，查詢延遲越來越高。

**當時的瓶頸**：

```
資料庫查詢延遲：
- 平均延遲：50 ms
- P99 延遲：200 ms

iostat 顯示：
- IOPS：~80,000
- 平均佇列深度：28-32  ← 接近 SATA 的上限（32）
```

硬體工程師建議：「換成 NVMe SSD 試試看。」

我當時的疑問：「NVMe 不就是更快的 SSD 嗎？能有多大差別？」

**升級到 NVMe 後的結果**：

```
資料庫查詢延遲：
- 平均延遲：8 ms   ← 降低 84%
- P99 延遲：25 ms  ← 降低 87.5%

iostat 顯示：
- IOPS：~450,000  ← 提升 5.6 倍
- 平均佇列深度：64
```

**震撼的發現**：

不只是頻寬提升（SATA 600 MB/s → NVMe 3,500 MB/s），更重要的是：

1. **多佇列架構**：每個 CPU core 有獨立的 I/O queue
2. **低延遲**：命令處理延遲從 2-3 μs 降到 0.5-1 μs
3. **高並行**：可以同時處理數十萬個 I/O 請求

這次經驗讓我理解：**NVMe 不只是更快的 SATA，它是為 SSD 重新設計的協議**。

本文將深入探討 NVMe 的設計哲學、架構，以及它如何徹底改變儲存系統的性能。

---

## 一、為什麼需要 NVMe？

### 1.1 SATA/AHCI 的歷史包袱

**AHCI 的設計目標**：為 HDD 優化

```
HDD 的特性：
- 尋道時間：5-10 ms
- 旋轉延遲：4-8 ms
- 循序讀取：100-200 MB/s
- 隨機 IOPS：100-200

AHCI 的設計：
- 單一佇列，深度 32  ← 對 HDD 夠用
- NCQ (Native Command Queuing)  ← 優化 HDD 尋道
- 命令處理延遲：2-3 μs  ← 相比 HDD 的 10 ms，可以忽略
```

**SSD 的特性**：完全不同

```
SSD 的特性：
- 無尋道時間
- 無旋轉延遲
- 循序讀取：500-7,000 MB/s
- 隨機 IOPS：100,000-1,000,000

AHCI 的問題：
- 單一佇列成為瓶頸  ← SSD 可以並行處理數千個請求
- 命令處理延遲變得明顯  ← 2-3 μs 佔總延遲的 20-30%
- 頻寬限制  ← SATA III 只有 600 MB/s
```

---

### 1.2 NVMe 的設計哲學

**NVMe (Non-Volatile Memory Express)** 的核心理念：

1. **為 SSD 設計**：不考慮 HDD 的相容性
2. **充分利用 PCIe**：高頻寬、低延遲
3. **多佇列架構**：每個 CPU core 有獨立的 queue
4. **簡化命令集**：減少不必要的複雜性

**AHCI vs NVMe 的對比**：

| 特性 | AHCI | NVMe |
|------|------|------|
| 發布年份 | 2004 | 2011 |
| 設計目標 | HDD | SSD |
| 物理介面 | SATA | PCIe |
| 最大佇列數 | 1 | 65,536 |
| 每個佇列深度 | 32 | 65,536 |
| 最大命令數 | 32 | 4,294,967,296 |
| 命令處理延遲 | 2-3 μs | 0.5-1 μs |
| 中斷機制 | 單一中斷 | MSI-X (2048 向量) |
| CPU 親和性 | 無 | 支援 |

---

## 二、NVMe 的核心架構

### 2.1 多佇列架構

**NVMe 的佇列模型**：

```
┌─────────────────────────────────────────────────┐
│                 作業系統                         │
│  ┌──────────────────────────────────────────┐   │
│  │         NVMe Driver                      │   │
│  │  ┌────────────────────────────────────┐  │   │
│  │  │ Admin Queue Pair                   │  │   │
│  │  │  - Admin SQ (Submission Queue)     │  │   │
│  │  │  - Admin CQ (Completion Queue)     │  │   │
│  │  ├────────────────────────────────────┤  │   │
│  │  │ I/O Queue Pair 0 (CPU 0)           │  │   │
│  │  │  - I/O SQ 0                        │  │   │
│  │  │  - I/O CQ 0                        │  │   │
│  │  ├────────────────────────────────────┤  │   │
│  │  │ I/O Queue Pair 1 (CPU 1)           │  │   │
│  │  │  - I/O SQ 1                        │  │   │
│  │  │  - I/O CQ 1                        │  │   │
│  │  ├────────────────────────────────────┤  │   │
│  │  │ ...                                │  │   │
│  │  │ (最多 65,535 個 I/O Queue Pairs)   │  │   │
│  │  └────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

**Queue Pair 的概念**：

```
┌─────────────────────────────────────────────────┐
│  Queue Pair                                     │
│  ┌──────────────────────────────────────────┐   │
│  │  Submission Queue (SQ)                   │   │
│  │  ┌────┬────┬────┬────┬────┬────┬────┐   │   │
│  │  │Cmd0│Cmd1│Cmd2│Cmd3│Cmd4│... │    │   │   │
│  │  └────┴────┴────┴────┴────┴────┴────┘   │   │
│  │  ↓ Host 寫入命令                         │   │
│  └──────────────────────────────────────────┘   │
│                      ↓                          │
│              NVMe Controller                    │
│                      ↓                          │
│  ┌──────────────────────────────────────────┐   │
│  │  Completion Queue (CQ)                   │   │
│  │  ┌────┬────┬────┬────┬────┬────┬────┐   │   │
│  │  │Cpl0│Cpl1│Cpl2│Cpl3│Cpl4│... │    │   │   │
│  │  └────┴────┴────┴────┴────┴────┴────┘   │   │
│  │  ↑ Controller 寫入完成狀態               │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

---

### 2.2 Submission Queue (SQ)：命令提交

**SQ 的結構**：

```c
// NVMe Specification 1.4
struct nvme_command {
    union {
        struct {
            __u8  opcode;       // 命令類型
            __u8  flags;
            __u16 command_id;   // 命令 ID
            __le32 nsid;        // Namespace ID
            __le64 rsvd2;
            __le64 metadata;
            __le64 prp1;        // Physical Region Page 1 (DMA 地址)
            __le64 prp2;        // Physical Region Page 2
            __le32 cdw10;       // Command Dword 10
            __le32 cdw11;
            __le32 cdw12;
            __le32 cdw13;
            __le32 cdw14;
            __le32 cdw15;
        } common;
        
        struct nvme_rw_command rw;      // Read/Write 命令
        struct nvme_admin_command admin; // Admin 命令
    };
};
```

**發送命令的流程**：

```
1. Host 準備 NVMe 命令
   ↓
2. Host 寫入 Submission Queue
   ↓
3. Host 寫入 Submission Queue Tail Doorbell Register
   ↓
4. NVMe Controller 收到 doorbell 通知
   ↓
5. NVMe Controller 從 SQ 讀取命令（透過 DMA）
   ↓
6. NVMe Controller 執行命令
```

---

### 2.3 Completion Queue (CQ)：命令完成

**CQ 的結構**：

```c
// NVMe Specification 1.4
struct nvme_completion {
    __le32 result;          // 命令結果
    __le32 rsvd;
    __le16 sq_head;         // SQ Head pointer
    __le16 sq_id;           // SQ ID
    __u16  command_id;      // 對應的命令 ID
    __u16  status;          // 狀態碼
};
```

**處理完成的流程**：

```
1. NVMe Controller 執行完命令
   ↓
2. Controller 寫入 Completion Queue（透過 DMA）
   ↓
3. Controller 發送 MSI-X 中斷
   ↓
4. Host 的中斷處理程式被喚醒
   ↓
5. Host 從 CQ 讀取完成狀態
   ↓
6. Host 更新 Completion Queue Head Doorbell Register
   ↓
7. Host 處理完成的命令（例如：喚醒等待的 process）
```

---

### 2.4 Doorbell Registers：通知機制

**Doorbell 的作用**：

```
┌─────────────────────────────────────────────────┐
│  NVMe Controller 的 MMIO 空間                   │
│  ┌──────────────────────────────────────────┐   │
│  │  0x0000: Controller Capabilities         │   │
│  │  0x0008: Version                         │   │
│  │  0x0014: Controller Configuration        │   │
│  │  0x001C: Controller Status               │   │
│  │  ...                                     │   │
│  ├──────────────────────────────────────────┤   │
│  │  0x1000: SQ 0 Tail Doorbell              │   │  ← Host 寫入
│  │  0x1004: CQ 0 Head Doorbell              │   │  ← Host 寫入
│  │  0x1008: SQ 1 Tail Doorbell              │   │
│  │  0x100C: CQ 1 Head Doorbell              │   │
│  │  ...                                     │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

**Doorbell 的實作**：

```c
// Linux kernel: drivers/nvme/host/pci.c
static void nvme_submit_cmd(struct nvme_queue *nvmeq, struct nvme_command *cmd)
{
    // 1. 寫入 Submission Queue
    memcpy(&nvmeq->sq_cmds[nvmeq->sq_tail], cmd, sizeof(*cmd));

    // 2. 更新 SQ tail
    if (++nvmeq->sq_tail == nvmeq->q_depth)
        nvmeq->sq_tail = 0;

    // 3. 寫入 Doorbell Register（通知 Controller）
    writel(nvmeq->sq_tail, nvmeq->q_db);
}
```

---

## 三、NVMe 的命令集

### 3.1 Admin Commands：管理命令

**Admin Queue 的用途**：

- 建立/刪除 I/O Queue
- 識別 Controller 和 Namespace
- 設定功能
- Firmware 更新

**常用的 Admin Commands**：

```c
// NVMe Admin Commands
enum nvme_admin_opcode {
    nvme_admin_delete_sq        = 0x00,  // 刪除 Submission Queue
    nvme_admin_create_sq        = 0x01,  // 建立 Submission Queue
    nvme_admin_delete_cq        = 0x04,  // 刪除 Completion Queue
    nvme_admin_create_cq        = 0x05,  // 建立 Completion Queue
    nvme_admin_identify         = 0x06,  // 識別 Controller/Namespace
    nvme_admin_abort_cmd        = 0x08,  // 中止命令
    nvme_admin_set_features     = 0x09,  // 設定功能
    nvme_admin_get_features     = 0x0a,  // 取得功能
    nvme_admin_async_event      = 0x0c,  // 非同步事件請求
    nvme_admin_fw_commit        = 0x10,  // Firmware Commit
    nvme_admin_fw_download      = 0x11,  // Firmware Download
    nvme_admin_format_nvm       = 0x80,  // 格式化 Namespace
    nvme_admin_security_send    = 0x81,  // 安全發送
    nvme_admin_security_recv    = 0x82,  // 安全接收
};
```

**範例：建立 I/O Queue Pair**：

```c
// Linux kernel: drivers/nvme/host/pci.c
static int nvme_create_queue(struct nvme_queue *nvmeq, int qid)
{
    struct nvme_dev *dev = nvmeq->dev;
    struct nvme_command c;

    // 1. 建立 Completion Queue
    memset(&c, 0, sizeof(c));
    c.create_cq.opcode = nvme_admin_create_cq;
    c.create_cq.prp1 = cpu_to_le64(nvmeq->cq_dma_addr);
    c.create_cq.cqid = cpu_to_le16(qid);
    c.create_cq.qsize = cpu_to_le16(nvmeq->q_depth - 1);
    c.create_cq.cq_flags = cpu_to_le16(flags);
    c.create_cq.irq_vector = cpu_to_le16(nvmeq->cq_vector);

    nvme_submit_sync_cmd(dev->admin_q, &c, NULL, 0);

    // 2. 建立 Submission Queue
    memset(&c, 0, sizeof(c));
    c.create_sq.opcode = nvme_admin_create_sq;
    c.create_sq.prp1 = cpu_to_le64(nvmeq->sq_dma_addr);
    c.create_sq.sqid = cpu_to_le16(qid);
    c.create_sq.qsize = cpu_to_le16(nvmeq->q_depth - 1);
    c.create_sq.sq_flags = cpu_to_le16(flags);
    c.create_sq.cqid = cpu_to_le16(qid);

    nvme_submit_sync_cmd(dev->admin_q, &c, NULL, 0);
}
```

---

### 3.2 I/O Commands：資料傳輸

**常用的 I/O Commands**：

```c
// NVMe I/O Commands
enum nvme_opcode {
    nvme_cmd_flush          = 0x00,  // Flush
    nvme_cmd_write          = 0x01,  // Write
    nvme_cmd_read           = 0x02,  // Read
    nvme_cmd_write_uncor    = 0x04,  // Write Uncorrectable
    nvme_cmd_compare        = 0x05,  // Compare
    nvme_cmd_write_zeroes   = 0x08,  // Write Zeroes
    nvme_cmd_dsm            = 0x09,  // Dataset Management (TRIM)
    nvme_cmd_resv_register  = 0x0d,  // Reservation Register
    nvme_cmd_resv_report    = 0x0e,  // Reservation Report
    nvme_cmd_resv_acquire   = 0x11,  // Reservation Acquire
    nvme_cmd_resv_release   = 0x15,  // Reservation Release
};
```

**Read 命令的結構**：

```c
struct nvme_rw_command {
    __u8  opcode;           // nvme_cmd_read
    __u8  flags;
    __u16 command_id;
    __le32 nsid;            // Namespace ID
    __le64 rsvd2;
    __le64 metadata;
    __le64 prp1;            // DMA 地址（第一個 page）
    __le64 prp2;            // DMA 地址（第二個 page 或 PRP List）
    __le64 slba;            // Starting LBA
    __le16 length;          // Number of logical blocks (0-based)
    __le16 control;
    __le32 dsmgmt;          // Dataset Management
    __le32 reftag;          // Reference Tag
    __le16 apptag;          // Application Tag
    __le16 appmask;         // Application Tag Mask
};
```

---

## 四、NVMe 的性能優化

### 4.1 CPU 親和性 (CPU Affinity)

**問題**：如果所有 CPU 共用一個 I/O queue，會有 lock contention。

**解決方案**：每個 CPU core 有獨立的 I/O queue。

```
傳統架構（AHCI）：
┌────┐ ┌────┐ ┌────┐ ┌────┐
│CPU0│ │CPU1│ │CPU2│ │CPU3│
└──┬─┘ └──┬─┘ └──┬─┘ └──┬─┘
   │      │      │      │
   └──────┴──────┴──────┘
          │
    ┌─────▼─────┐
    │ Single    │  ← Lock contention!
    │ Queue     │
    └───────────┘

NVMe 架構：
┌────┐ ┌────┐ ┌────┐ ┌────┐
│CPU0│ │CPU1│ │CPU2│ │CPU3│
└──┬─┘ └──┬─┘ └──┬─┘ └──┬─┘
   │      │      │      │
┌──▼──┐┌──▼──┐┌──▼──┐┌──▼──┐
│ Q0  ││ Q1  ││ Q2  ││ Q3  │  ← No lock!
└─────┘└─────┘└─────┘└─────┘
```

---

### 4.2 Namespace：邏輯分區

**Namespace 的概念**：

NVMe 支援將一個 SSD 分成多個 **Namespace**（類似分區）。

```
┌─────────────────────────────────────────────────┐
│  NVMe SSD (1 TB)                                │
│  ┌──────────────────────────────────────────┐   │
│  │  Namespace 1 (500 GB)                    │   │
│  │  - /dev/nvme0n1                          │   │
│  ├──────────────────────────────────────────┤   │
│  │  Namespace 2 (300 GB)                    │   │
│  │  - /dev/nvme0n2                          │   │
│  ├──────────────────────────────────────────┤   │
│  │  Namespace 3 (200 GB)                    │   │
│  │  - /dev/nvme0n3                          │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

**Namespace 的優勢**：

1. **隔離**：不同 namespace 的資料互不影響
2. **QoS**：可以為不同 namespace 設定不同的優先級
3. **安全**：可以為不同 namespace 設定不同的加密金鑰

---

## 五、實戰：NVMe 性能測試

### 5.1 測試環境

- **SSD**: Samsung 980 PRO 1TB (PCIe 4.0 x4)
- **主機板**: ASUS ROG Strix B550-F
- **CPU**: AMD Ryzen 7 5800X (8 cores, 16 threads)
- **記憶體**: 32 GB DDR4-3600
- **作業系統**: Ubuntu 22.04 LTS
- **Kernel**: 5.15.0

---

### 5.2 測試 1：循序讀寫

**測試命令**：

```bash
# 循序讀取
fio --name=seq_read --filename=/dev/nvme0n1 --rw=read --bs=128k \
    --iodepth=32 --direct=1 --runtime=60 --time_based --group_reporting

# 循序寫入
fio --name=seq_write --filename=/dev/nvme0n1 --rw=write --bs=128k \
    --iodepth=32 --direct=1 --runtime=60 --time_based --group_reporting
```

**結果**：

```
循序讀取：
  頻寬: 7,000 MB/s
  IOPS: 56,000
  延遲: 0.57 ms

循序寫入：
  頻寬: 5,000 MB/s
  IOPS: 40,000
  延遲: 0.80 ms
```

**觀察**：

- 讀取頻寬接近 PCIe 4.0 x4 的理論上限（8 GB/s）
- 寫入頻寬較低，因為 SSD 需要做 GC 和 Wear Leveling

---

### 5.3 測試 2：隨機讀寫

**測試命令**：

```bash
# 隨機讀取
fio --name=rand_read --filename=/dev/nvme0n1 --rw=randread --bs=4k \
    --iodepth=128 --direct=1 --runtime=60 --time_based --group_reporting

# 隨機寫入
fio --name=rand_write --filename=/dev/nvme0n1 --rw=randwrite --bs=4k \
    --iodepth=128 --direct=1 --runtime=60 --time_based --group_reporting
```

**結果**：

```
隨機讀取：
  頻寬: 3,900 MB/s
  IOPS: 1,000,000
  延遲: 0.13 ms

隨機寫入：
  頻寬: 1,600 MB/s
  IOPS: 410,000
  延遲: 0.31 ms
```

**觀察**：

- 隨機讀取 IOPS 達到 1M，展現 NVMe 的並行能力
- 隨機寫入 IOPS 較低，因為 SSD 的寫入放大（Write Amplification）

---

### 5.4 測試 3：Queue Depth 的影響

**測試不同的 Queue Depth**：

```bash
for qd in 1 4 16 64 256; do
    fio --name=rand_read_qd${qd} --filename=/dev/nvme0n1 --rw=randread \
        --bs=4k --iodepth=${qd} --direct=1 --runtime=30 --time_based
done
```

**結果**：

| Queue Depth | IOPS | 延遲 (ms) |
|-------------|------|-----------|
| 1 | 25,000 | 0.04 |
| 4 | 95,000 | 0.04 |
| 16 | 350,000 | 0.05 |
| 64 | 800,000 | 0.08 |
| 256 | 1,000,000 | 0.26 |

**觀察**：

- Queue Depth 從 1 增加到 256，IOPS 提升 40 倍
- 延遲隨 Queue Depth 增加而增加（排隊延遲）
- Queue Depth = 64 是最佳平衡點（高 IOPS + 低延遲）

---

## 六、總結

### NVMe 的優勢

✅ **高性能**：IOPS 可達 1M+，頻寬可達 7 GB/s+
✅ **低延遲**：命令處理延遲 < 1 μs
✅ **多佇列**：最多 65,536 個 queue，每個深度 65,536
✅ **CPU 親和性**：每個 CPU core 有獨立的 queue
✅ **可擴展**：支援 NVMe-oF，可用於網路儲存

### NVMe vs SATA 的對比

| 特性 | SATA/AHCI | NVMe |
|------|-----------|------|
| 頻寬 | 600 MB/s | 7,000 MB/s (11.7x) |
| IOPS | 90K | 1M (11.1x) |
| 延遲 | 2-3 μs | 0.5-1 μs (3x faster) |
| 佇列數 | 1 | 65,536 |
| 佇列深度 | 32 | 65,536 |

### 未來展望

- **PCIe 5.0/6.0**：頻寬將達到 16-32 GB/s
- **NVMe 2.0**：新增 Zoned Namespace, Key-Value 等功能
- **CXL**：Cache coherent 的記憶體擴展（下一篇文章）

**下一篇文章**，我們將探討 **CXL (Compute Express Link)**——基於 PCIe 的 cache coherent 互連，看看它如何改變儲存和記憶體的架構。

---

## 參考資料

1. NVM Express: <https://nvmexpress.org/>
2. NVMe Specification 1.4: <https://nvmexpress.org/specifications/>
3. Linux kernel NVMe driver: <https://www.kernel.org/doc/html/latest/driver-api/nvme/>
4. fio - Flexible I/O Tester: <https://fio.readthedocs.io/>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
