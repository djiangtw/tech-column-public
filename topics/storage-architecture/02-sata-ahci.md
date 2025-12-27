# SATA/AHCI - 傳統存儲接口的瓶頸

**作者**: Danny Jiang  
**日期**: 2025-12-08  

---

## 前言：一個被 SATA 限制的 SSD

2016 年，我在一家嵌入式系統公司工作。有一天，產品經理興沖沖地拿著一顆新的 SATA SSD 來找我：

「Danny，我們換了這顆新的 SSD，讀寫速度應該會快很多吧？」

我看了一下規格：這是一顆高階的 SATA SSD，標稱讀取速度 550 MB/s，寫入速度 520 MB/s。理論上確實比舊的 HDD（~120 MB/s）快了 4 倍。

但實際測試後，我們發現：

- **循序讀取**：確實達到了 550 MB/s ✅
- **循序寫入**：也達到了 520 MB/s ✅
- **隨機讀取（4K）**：只有 ~40 MB/s ❌
- **隨機寫入（4K）**：只有 ~80 MB/s ❌

產品經理很困惑：「為什麼隨機存取這麼慢？這顆 SSD 不是號稱可以達到 90,000 IOPS 嗎？」

我解釋道：「問題不在 SSD，而在 **SATA 接口**。SATA 是為 HDD 設計的，它的協議（AHCI）有一個致命的限制：**單一命令佇列，深度只有 32**。」

「這意味著什麼？」

「意味著即使 SSD 內部可以同時處理數千個請求，但 SATA 接口一次只能送 32 個命令進去。就像一個超市有 100 個收銀台，但門口只開了一個小門，顧客只能排隊進去。」

這就是 **SATA 的瓶頸**。

本文將深入探討 SATA 和 AHCI 的設計、限制，以及為什麼它們無法發揮 SSD 的全部潛力。

---

## 一、SATA 的歷史：從 PATA 到 SATA III

### 1.1 PATA 時代：平行傳輸的黃金年代

在 SATA 出現之前，硬碟使用的是 **PATA (Parallel ATA)**，也叫 IDE 接口。

**PATA 的特性**：

```
┌─────────────────────────────────────────┐
│  PATA 接口（40-pin 或 80-pin 排線）      │
│                                         │
│  ┌───┐ ┌───┐ ┌───┐     ┌───┐ ┌───┐    │
│  │D0 │ │D1 │ │D2 │ ... │D15│ │Ctrl│    │
│  └───┘ └───┘ └───┘     └───┘ └───┘    │
│                                         │
│  16-bit 平行傳輸（一次傳 16 bits）       │
└─────────────────────────────────────────┘
```

**PATA 的問題**：

1. **排線太粗**：40 或 80 條線，佔空間、影響散熱
2. **訊號干擾**：平行線之間會互相干擾（crosstalk）
3. **速度限制**：最高只能到 133 MB/s（ATA-133）
4. **主從架構**：一條排線只能接 2 個裝置（Master/Slave），配置麻煩

---

### 1.2 SATA 的誕生：串列傳輸的革命

2003 年，**SATA (Serial ATA)** 正式發布，取代了 PATA。

**SATA 的改進**：

```
┌─────────────────────────────────────────┐
│  SATA 接口（7-pin 細線）                 │
│                                         │
│  ┌───┐ ┌───┐ ┌───┐ ┌───┐              │
│  │TX+│ │TX-│ │RX+│ │RX-│ + GND        │
│  └───┘ └───┘ └───┘ └───┘              │
│                                         │
│  差分訊號（Differential Signaling）      │
│  串列傳輸（一次傳 1 bit，但速度更快）    │
└─────────────────────────────────────────┘
```

**為什麼串列比平行快？**

這聽起來很反直覺：一次傳 1 bit 怎麼會比一次傳 16 bits 快？

答案是：**時脈頻率**。

- **PATA**：平行線之間會互相干擾，時脈頻率只能到 ~66 MHz
  - 頻寬 = 66 MHz × 16 bits = 133 MB/s
  
- **SATA**：差分訊號抗干擾能力強，時脈頻率可以到 1.5 GHz（SATA I）
  - 頻寬 = 1.5 GHz × 1 bit × 8b/10b 編碼效率 = 150 MB/s

**SATA 的世代演進**：

| 世代 | 發布年份 | 原始速率 | 實際頻寬 | 備註 |
|------|---------|---------|---------|------|
| SATA I | 2003 | 1.5 Gbps | 150 MB/s | 初代 |
| SATA II | 2004 | 3.0 Gbps | 300 MB/s | 加入 NCQ |
| SATA III | 2009 | 6.0 Gbps | 600 MB/s | 目前主流 |

> **註**：8b/10b 編碼會損失 20% 的頻寬（10 bits 傳輸 8 bits 資料），所以 6 Gbps 的實際頻寬是 600 MB/s。

---

## 二、AHCI 協議：為 HDD 設計的軟體接口

SATA 只是物理層的接口，軟體要如何與 SATA 裝置溝通呢？答案是 **AHCI (Advanced Host Controller Interface)**。

### 2.1 AHCI 的設計理念

AHCI 是 Intel 在 2004 年提出的標準，目的是：

1. **統一接口**：讓所有 SATA 控制器都用同一套軟體接口
2. **取代舊的 IDE 模式**：IDE 模式需要為每個廠商寫不同的驅動
3. **支援進階功能**：熱插拔（Hot-plug）、NCQ（Native Command Queuing）

**AHCI 的架構**：

```
┌─────────────────────────────────────────────────────┐
│                   作業系統                           │
│  ┌─────────────────────────────────────────────┐   │
│  │          AHCI Driver (libata)               │   │
│  └─────────────────┬───────────────────────────┘   │
│                    │                                │
│  ┌─────────────────▼───────────────────────────┐   │
│  │          AHCI Host Controller               │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │   │
│  │  │  Port 0  │  │  Port 1  │  │  Port 2  │  │   │
│  │  │ (32 cmd) │  │ (32 cmd) │  │ (32 cmd) │  │   │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  │   │
│  └───────┼─────────────┼─────────────┼────────┘   │
└──────────┼─────────────┼─────────────┼────────────┘
           │             │             │
      ┌────▼────┐   ┌────▼────┐   ┌────▼────┐
      │ SATA    │   │ SATA    │   │ SATA    │
      │ SSD 0   │   │ HDD 1   │   │ SSD 2   │
      └─────────┘   └─────────┘   └─────────┘
```

**關鍵概念**：

- **Port**：每個 SATA 裝置連接到一個 Port
- **Command List**：每個 Port 有一個命令列表，最多 **32 個命令**
- **Command Table**：每個命令對應一個 Command Table，描述要讀寫的資料

---

## 三、NCQ (Native Command Queuing)：AHCI 的救命稻草

### 3.1 為什麼需要 NCQ？

在沒有 NCQ 之前，SATA 一次只能處理一個命令：

```
時間軸：
┌────┐     ┌────┐     ┌────┐     ┌────┐
│Cmd1│ ... │Cmd2│ ... │Cmd3│ ... │Cmd4│
└────┘     └────┘     └────┘     └────┘
  ↓          ↓          ↓          ↓
 等待       等待       等待       等待
 完成       完成       完成       完成
```

**問題**：如果 Cmd1 需要等待 10ms（HDD 尋道時間），CPU 就要空等 10ms，效率很低。

**NCQ 的解決方案**：允許同時發送多個命令（最多 32 個），讓硬碟自己決定執行順序。

```
時間軸：
┌────┬────┬────┬────┐
│Cmd1│Cmd2│Cmd3│Cmd4│ ← 一次發送 4 個命令
└────┴────┴────┴────┘
  ↓    ↓    ↓    ↓
  └────┴────┴────┴──→ HDD 重新排序，優化尋道路徑
```

---

### 3.2 NCQ 的工作原理

**HDD 的尋道優化**：

假設有 4 個讀取請求：

```
Cmd1: LBA 1000
Cmd2: LBA 5000
Cmd3: LBA 2000
Cmd4: LBA 4000

HDD 磁頭目前在 LBA 0
```

**沒有 NCQ**：按順序執行

```
0 → 1000 → 5000 → 2000 → 4000
   (1000) (4000) (3000) (2000) = 總共移動 10,000 個 LBA
```

**有 NCQ**：重新排序

```
0 → 1000 → 2000 → 4000 → 5000
   (1000) (1000) (2000) (1000) = 總共移動 5,000 個 LBA
```

**效能提升**：減少了 50% 的尋道時間！

---

### 3.3 NCQ 對 SSD 有用嗎？

**答案**：有用，但不是因為尋道優化。

SSD 沒有機械尋道，但 NCQ 仍然有幫助：

1. **並行處理**：SSD 內部有多個 NAND channel，可以同時處理多個請求
2. **減少延遲**：不用等待一個命令完成才發送下一個

**實測數據**（SATA SSD）：

```
測試環境：
- SSD: Samsung 860 EVO (SATA III)
- 測試工具: fio
- 測試項目: 4K 隨機讀取

結果：
- Queue Depth = 1:  ~10,000 IOPS
- Queue Depth = 4:  ~30,000 IOPS
- Queue Depth = 32: ~90,000 IOPS
```

**觀察**：Queue Depth 從 1 增加到 32，IOPS 提升了 9 倍！

---

## 四、SATA 的限制：為什麼無法發揮 SSD 的潛力？

### 4.1 限制 1：頻寬瓶頸

**SATA III 的理論頻寬**：6 Gbps = 600 MB/s

**現代 SSD 的能力**：

```
高階 SATA SSD (Samsung 870 EVO):
- 循序讀取: 560 MB/s  ← 已經接近 SATA III 上限
- 循序寫入: 530 MB/s

NVMe SSD (Samsung 980 PRO):
- 循序讀取: 7,000 MB/s  ← 是 SATA 的 11 倍！
- 循序寫入: 5,000 MB/s
```

**結論**：SATA III 的 600 MB/s 已經成為瓶頸。

---

### 4.2 限制 2：單一佇列，深度只有 32

**AHCI 的佇列限制**：

```
┌─────────────────────────────────┐
│  AHCI Host Controller           │
│  ┌───────────────────────────┐  │
│  │  Command List (32 slots)  │  │
│  │  ┌──┐┌──┐┌──┐    ┌──┐    │  │
│  │  │ 0││ 1││ 2│... │31│    │  │
│  │  └──┘└──┘└──┘    └──┘    │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
         ↓
    單一佇列，深度 32
```

**NVMe 的佇列能力**：

```
┌─────────────────────────────────┐
│  NVMe Controller                │
│  ┌───────┐ ┌───────┐ ┌───────┐ │
│  │Queue 0│ │Queue 1│ │Queue 2│ │
│  │(64K)  │ │(64K)  │ │(64K)  │ │
│  └───────┘ └───────┘ └───────┘ │
│       ...  (最多 64K 個佇列)    │
└─────────────────────────────────┘
         ↓
    多佇列，每個深度 64K
```

**影響**：

- **AHCI**：最多同時處理 32 個命令
- **NVMe**：最多同時處理 64K × 64K = 4,194,304,000 個命令（理論值）

**實際案例**：

```
測試：4K 隨機讀取，Queue Depth = 128

SATA SSD (受限於 QD=32):
- IOPS: ~90,000

NVMe SSD (QD=128):
- IOPS: ~500,000  ← 是 SATA 的 5.5 倍！
```

---

### 4.3 限制 3：高延遲

**AHCI 的命令處理流程**：

```
1. 驅動程式準備 Command Header
2. 驅動程式準備 Command Table
3. 驅動程式寫入 Command Issue Register
4. AHCI Controller 讀取 Command Header
5. AHCI Controller 讀取 Command Table
6. AHCI Controller 發送 FIS (Frame Information Structure) 到 SATA PHY
7. SATA PHY 傳輸到 SSD
8. SSD 執行命令
9. SSD 回傳結果
10. AHCI Controller 產生中斷
11. 驅動程式處理中斷

總延遲：~2-3 μs (微秒)
```

**NVMe 的命令處理流程**：

```
1. 驅動程式寫入 Submission Queue
2. 驅動程式寫入 Doorbell Register
3. NVMe Controller 讀取 Submission Queue
4. NVMe Controller 發送命令到 SSD
5. SSD 執行命令
6. SSD 回傳結果到 Completion Queue
7. NVMe Controller 產生中斷

總延遲：~0.5-1 μs (微秒)
```

**結論**：NVMe 的延遲是 AHCI 的 1/3 到 1/2。

---

## 五、SATA 的未來：被 NVMe 取代

### 5.1 SATA Express 的失敗

2013 年，SATA-IO 組織推出了 **SATA Express**，試圖結合 SATA 和 PCIe：

```
SATA Express 接口：
┌─────────────────────────────────┐
│  ┌─────┐  ┌─────┐               │
│  │SATA │  │SATA │  ← 向下相容   │
│  │Port │  │Port │               │
│  └─────┘  └─────┘               │
│  ┌─────────────────┐            │
│  │   PCIe x2       │  ← 新功能  │
│  └─────────────────┘            │
└─────────────────────────────────┘

頻寬：PCIe 3.0 x2 = 2 GB/s
```

**為什麼失敗？**

1. **接口太大**：需要佔用 2 個 SATA port 的空間
2. **成本高**：需要支援 SATA 和 PCIe 兩種協議
3. **NVMe 更好**：M.2 接口更小，NVMe 協議更高效

**結果**：SATA Express 幾乎沒有產品採用，很快就被市場淘汰。

---

### 5.2 SATA 的定位：低成本、大容量

雖然 SATA 在性能上已經被 NVMe 超越，但它仍然有存在的價值：

**SATA 的優勢**：

1. **成本低**：SATA SSD 比 NVMe SSD 便宜 20-30%
2. **相容性好**：所有電腦都支援 SATA
3. **夠用**：對於一般使用者，600 MB/s 已經足夠

**適用場景**：

- **大容量儲存**：2TB、4TB 的 SATA SSD 比 NVMe 便宜
- **舊電腦升級**：沒有 M.2 插槽的電腦
- **伺服器**：需要大量 2.5" 硬碟槽的伺服器

---

## 六、系統軟體視角：AHCI Driver 實作

### 6.1 Linux 的 libata 架構

Linux kernel 使用 **libata** 來支援 SATA/AHCI：

```
┌─────────────────────────────────────────┐
│         Block Layer (VFS)               │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│         SCSI Layer (sd driver)          │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│         libata (ATA/SATA core)          │
│  ┌──────────────────────────────────┐   │
│  │  ata_qc_issue() - 發送命令       │   │
│  │  ata_qc_complete() - 完成處理    │   │
│  └──────────────────────────────────┘   │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│         ahci driver                     │
│  ┌──────────────────────────────────┐   │
│  │  ahci_qc_prep() - 準備命令       │   │
│  │  ahci_qc_issue() - 發送到硬體    │   │
│  │  ahci_interrupt() - 中斷處理     │   │
│  └──────────────────────────────────┘   │
└─────────────────┬───────────────────────┘
                  │
            ┌─────▼─────┐
            │   AHCI    │
            │ Hardware  │
            └───────────┘
```

---

### 6.2 關鍵函式解析

**發送命令**：

```c
// drivers/ata/libahci.c
static unsigned int ahci_qc_issue(struct ata_queued_cmd *qc)
{
    struct ata_port *ap = qc->ap;
    void __iomem *port_mmio = ahci_port_base(ap);

    // 寫入 Command Issue Register
    writel(1 << qc->tag, port_mmio + PORT_CMD_ISSUE);

    return 0;
}
```

**中斷處理**：

```c
static irqreturn_t ahci_single_level_irq_intr(int irq, void *dev_instance)
{
    struct ata_host *host = dev_instance;
    struct ahci_host_priv *hpriv = host->private_data;
    unsigned int irq_masked;

    // 讀取 Interrupt Status
    irq_masked = readl(hpriv->mmio + HOST_IRQ_STAT);

    // 處理每個 port 的中斷
    for (i = 0; i < host->n_ports; i++) {
        if (irq_masked & (1 << i)) {
            ahci_port_intr(host->ports[i]);
        }
    }

    return IRQ_HANDLED;
}
```

---

## 七、實戰：測量 SATA SSD 的性能

### 7.1 使用 fio 測試

**安裝 fio**：

```bash
sudo apt install fio
```

**測試循序讀取**：

```bash
fio --name=seq_read \
    --filename=/dev/sda \
    --rw=read \
    --bs=128k \
    --iodepth=32 \
    --direct=1 \
    --runtime=60 \
    --time_based \
    --group_reporting
```

**測試隨機讀取**：

```bash
fio --name=rand_read \
    --filename=/dev/sda \
    --rw=randread \
    --bs=4k \
    --iodepth=32 \
    --direct=1 \
    --runtime=60 \
    --time_based \
    --group_reporting
```

---

### 7.2 實測結果分析

**我的測試環境**：

- SSD: Samsung 870 EVO 500GB (SATA III)
- 主機板: ASUS ROG Strix B550-F
- CPU: AMD Ryzen 7 5800X

**結果**：

```
循序讀取 (128K, QD=32):
- 頻寬: 558 MB/s
- IOPS: 4,464
- 延遲: 7.2 ms

隨機讀取 (4K, QD=32):
- 頻寬: 352 MB/s
- IOPS: 90,112
- 延遲: 0.35 ms

循序寫入 (128K, QD=32):
- 頻寬: 524 MB/s
- IOPS: 4,192
- 延遲: 7.6 ms

隨機寫入 (4K, QD=32):
- 頻寬: 312 MB/s
- IOPS: 79,872
- 延遲: 0.40 ms
```

**觀察**：

1. 循序讀取接近 SATA III 的理論上限（600 MB/s）
2. 隨機 IOPS 達到 ~90K，符合 NCQ 深度 32 的預期
3. 延遲在 0.35-0.40 ms，比 HDD（~10 ms）快 25 倍

---

## 八、總結

### SATA/AHCI 的優點

✅ **成熟穩定**：20 年的發展，相容性極佳
✅ **成本低**：控制器和 SSD 都便宜
✅ **夠用**：對於一般使用者，600 MB/s 足夠

### SATA/AHCI 的限制

❌ **頻寬瓶頸**：600 MB/s 無法發揮高階 SSD 的潛力
❌ **單一佇列**：Queue Depth 只有 32，限制並行能力
❌ **高延遲**：命令處理流程複雜，延遲是 NVMe 的 2-3 倍

### 未來展望

SATA 不會消失，但會逐漸被 NVMe 取代：

- **消費市場**：NVMe 已經成為主流（M.2 接口）
- **企業市場**：U.2 NVMe 取代 2.5" SATA
- **SATA 的定位**：低成本、大容量儲存

**下一篇文章**，我們將探討 **PCIe**——NVMe 的物理層基礎，看看它如何突破 SATA 的限制。

---

## 參考資料

1. Serial ATA International Organization (SATA-IO): <https://www.sata-io.org/>
2. AHCI Specification 1.3.1: <https://www.intel.com/content/www/us/en/io/serial-ata/ahci.html>
3. Linux kernel libata documentation: <https://www.kernel.org/doc/html/latest/driver-api/libata.html>
4. fio - Flexible I/O Tester: <https://fio.readthedocs.io/>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
