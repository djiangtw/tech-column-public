# PCIe - 高速互連的基石

**作者**: Danny Jiang  
**日期**: 2025-12-08  

---

## 前言：一個被 PCIe 拯救的專案

2017 年，我在一家網路設備公司負責一個高速封包處理系統。系統需要處理 10 Gbps 的網路流量，並將封包資料寫入 SSD 做記錄。

最初的設計使用 SATA SSD，但很快就遇到瓶頸：

```
網路流量：10 Gbps = 1,250 MB/s
SATA III：6 Gbps = 600 MB/s

問題：SATA 頻寬不夠！
```

硬體工程師建議：「換成 PCIe SSD 吧，PCIe 3.0 x4 有 4 GB/s 的頻寬。」

我當時很困惑：「PCIe 不是用來插顯示卡的嗎？怎麼 SSD 也能用 PCIe？」

後來我才理解：**PCIe 不只是一個插槽，它是一個通用的高速互連協議**。從顯示卡、網卡、SSD，到 FPGA、GPU，幾乎所有高速裝置都用 PCIe。

更重要的是，PCIe 提供了：

- **高頻寬**：PCIe 3.0 x4 = 4 GB/s（是 SATA 的 6.7 倍）
- **低延遲**：點對點連接，沒有仲裁延遲
- **可擴展**：從 x1 到 x16，靈活配置

這個專案最後成功了，系統穩定運行了 3 年。這次經驗讓我深刻體會到：**理解 PCIe，就理解了現代計算機的互連架構**。

本文將深入探討 PCIe 的設計、演進，以及它如何成為 NVMe SSD 的基礎。

---

## 一、PCIe 的誕生：取代 PCI 的革命

### 1.1 PCI 時代：共享匯流排的限制

在 PCIe 出現之前，電腦使用 **PCI (Peripheral Component Interconnect)** 來連接擴充卡。

**PCI 的架構**：

```
┌─────────────────────────────────────────────────┐
│                    CPU                          │
└──────────────────┬──────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────┐
│              North Bridge                       │
│  (Memory Controller + PCI Controller)           │
└──────────────────┬──────────────────────────────┘
                   │
    ┌──────────────┴──────────────┐
    │      Shared PCI Bus         │
    │  (32-bit, 33 MHz = 133 MB/s)│
    └──┬────────┬────────┬────────┬┘
       │        │        │        │
   ┌───▼───┐┌───▼───┐┌───▼───┐┌───▼───┐
   │ 網卡  ││ 音效卡││ 顯示卡││ SCSI  │
   └───────┘└───────┘└───────┘└───────┘
```

**PCI 的問題**：

1. **共享頻寬**：所有裝置共用 133 MB/s（PCI 32-bit 33 MHz）
   - 4 個裝置 → 每個只有 33 MB/s

2. **仲裁延遲**：同一時間只有一個裝置可以使用 bus

3. **無法擴展**：頻寬固定，無法隨裝置數量增加

**PCI-X 的嘗試**：

```
PCI-X 1.0: 64-bit, 133 MHz = 1,066 MB/s
PCI-X 2.0: 64-bit, 266 MHz = 2,133 MB/s

問題：仍然是共享 bus，無法解決根本問題
```

---

### 1.2 PCIe 的革命：點對點連接

2003 年，**PCIe (PCI Express)** 正式發布，徹底改變了遊戲規則。

**PCIe 的核心概念**：

1. **點對點連接**：每個裝置都有專屬的連接
2. **串列傳輸**：用差分訊號（Differential Signaling）取代平行 bus
3. **可擴展**：從 x1 到 x16，靈活配置頻寬

**PCIe 的架構**：

```
┌─────────────────────────────────────────────────┐
│                    CPU                          │
│  ┌──────────────────────────────────────────┐   │
│  │         PCIe Root Complex                │   │
│  └──┬────────┬────────┬────────┬────────────┘   │
└─────┼────────┼────────┼────────┼────────────────┘
      │        │        │        │
      │ x16    │ x4     │ x1     │ x4
      │        │        │        │
  ┌───▼───┐┌───▼───┐┌───▼───┐┌───▼───┐
  │  GPU  ││ NVMe  ││ 網卡  ││ NVMe  │
  │ (x16) ││ SSD   ││ (x1)  ││ SSD   │
  │       ││ (x4)  ││       ││ (x4)  │
  └───────┘└───────┘└───────┘└───────┘

每個裝置都有專屬的 PCIe lanes
```

**關鍵優勢**：

- GPU 獨佔 x16（16 GB/s）
- NVMe SSD 獨佔 x4（4 GB/s）
- 網卡獨佔 x1（1 GB/s）
- **沒有頻寬競爭！**

---

## 二、PCIe 的分層架構

PCIe 採用分層設計，類似網路的 OSI 模型：

```
┌─────────────────────────────────────────┐
│  Transaction Layer (TL)                 │  ← 軟體層
│  - 封包格式 (TLP)                       │
│  - 流量控制                             │
│  - 錯誤處理                             │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│  Data Link Layer (DLL)                  │  ← 可靠傳輸
│  - 序號 (Sequence Number)               │
│  - ACK/NAK                              │
│  - CRC 校驗                             │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│  Physical Layer (PHY)                   │  ← 硬體層
│  - 8b/10b 或 128b/130b 編碼             │
│  - 差分訊號                             │
│  - Lane 管理                            │
└─────────────────────────────────────────┘
```

---

### 2.1 Transaction Layer (TL)：封包格式

**TLP (Transaction Layer Packet)** 是 PCIe 的基本傳輸單位。

**TLP 的類型**：

1. **Memory Read/Write**：讀寫記憶體（最常用）
2. **I/O Read/Write**：讀寫 I/O 空間（已很少用）
3. **Configuration Read/Write**：讀寫配置空間
4. **Message**：中斷、錯誤通知等

**Memory Write TLP 格式**：

```
┌────────────────────────────────────────────────┐
│  Header (12 or 16 bytes)                       │
│  ┌──────────────────────────────────────────┐  │
│  │ Fmt/Type │ Length │ Requester ID │ Tag  │  │
│  ├──────────────────────────────────────────┤  │
│  │ Address (32-bit or 64-bit)               │  │
│  └──────────────────────────────────────────┘  │
├────────────────────────────────────────────────┤
│  Data Payload (0 - 4096 bytes)                 │
│  ┌──────────────────────────────────────────┐  │
│  │ Data[0] │ Data[1] │ ... │ Data[N]        │  │
│  └──────────────────────────────────────────┘  │
├────────────────────────────────────────────────┤
│  ECRC (4 bytes, optional)                      │
└────────────────────────────────────────────────┘
```

**關鍵欄位**：

- **Fmt/Type**：封包類型（Memory Read/Write, Config, Message）
- **Length**：資料長度（以 DWord = 4 bytes 為單位）
- **Requester ID**：發送者的 Bus/Device/Function 編號
- **Tag**：用於配對 Request 和 Completion
- **Address**：目標記憶體位址

---

### 2.2 Data Link Layer (DLL)：可靠傳輸

DLL 負責確保資料正確傳輸：

**功能**：

1. **序號 (Sequence Number)**：每個 TLP 都有序號
2. **ACK/NAK**：接收端回覆確認或否認
3. **重傳機制**：如果收到 NAK，重新發送
4. **CRC 校驗**：檢測傳輸錯誤

**DLLP (Data Link Layer Packet)** 格式：

```
┌────────────────────────────────────┐
│  Type (1 byte)                     │  ← ACK, NAK, Flow Control
├────────────────────────────────────┤
│  Data (2 bytes)                    │  ← Sequence Number, Credits
├────────────────────────────────────┤
│  CRC (2 bytes)                     │
└────────────────────────────────────┘
```

**流量控制 (Flow Control)**：

```
發送端：
  1. 檢查 Credits（接收端的緩衝區空間）
  2. 如果 Credits > 0，發送 TLP
  3. Credits -= TLP_size

接收端：
  1. 接收 TLP，放入緩衝區
  2. 處理完 TLP，釋放緩衝區
  3. 發送 Flow Control Update DLLP，增加 Credits
```

---

### 2.3 Physical Layer (PHY)：差分訊號

**Lane 的概念**：

一個 PCIe Lane 包含 4 條線：

```
┌─────────────────────────────────────┐
│  PCIe Lane                          │
│  ┌───────────────────────────────┐  │
│  │  TX+ (Transmit Positive)      │  │  ← 發送差分訊號
│  │  TX- (Transmit Negative)      │  │
│  ├───────────────────────────────┤  │
│  │  RX+ (Receive Positive)       │  │  ← 接收差分訊號
│  │  RX- (Receive Negative)       │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

**差分訊號的優勢**：

```
單端訊號 (Single-Ended):
  ┌─┐   ┌─┐
  │ │   │ │
──┘ └───┘ └──  ← 容易受干擾

差分訊號 (Differential):
  ┌─┐   ┌─┐
  │ │   │ │  TX+
──┘ └───┘ └──
  
──┐ ┌───┐ ┌──  TX-
  └─┘   └─┘

接收端計算 TX+ - TX-，干擾被抵消！
```

**編碼方式**：

- **PCIe 1.0/2.0**：8b/10b 編碼
  - 8 bits 資料 → 10 bits 傳輸
  - 效率：80%
  
- **PCIe 3.0+**：128b/130b 編碼
  - 128 bits 資料 → 130 bits 傳輸
  - 效率：98.5%

---

## 三、PCIe 的世代演進

### 3.1 頻寬的飛躍

| 世代 | 發布年份 | 編碼 | 每 Lane 速率 | x1 頻寬 | x4 頻寬 | x16 頻寬 |
|------|---------|------|-------------|---------|---------|----------|
| PCIe 1.0 | 2003 | 8b/10b | 2.5 GT/s | 250 MB/s | 1 GB/s | 4 GB/s |
| PCIe 2.0 | 2007 | 8b/10b | 5.0 GT/s | 500 MB/s | 2 GB/s | 8 GB/s |
| PCIe 3.0 | 2010 | 128b/130b | 8.0 GT/s | 1 GB/s | 4 GB/s | 16 GB/s |
| PCIe 4.0 | 2017 | 128b/130b | 16.0 GT/s | 2 GB/s | 8 GB/s | 32 GB/s |
| PCIe 5.0 | 2019 | 128b/130b | 32.0 GT/s | 4 GB/s | 16 GB/s | 64 GB/s |
| PCIe 6.0 | 2022 | PAM4 | 64.0 GT/s | 8 GB/s | 32 GB/s | 128 GB/s |

> **註**：GT/s = Giga Transfers per second（每秒十億次傳輸）

**觀察**：

- 每一代頻寬翻倍
- PCIe 5.0 x4 = 16 GB/s（是 SATA III 的 26 倍！）

---

### 3.2 PCIe 3.0 的關鍵改進

**128b/130b 編碼**：

```
PCIe 2.0 (8b/10b):
  傳輸 1 GB 資料需要：
  1 GB × (10/8) = 1.25 GB 的實際傳輸
  效率：80%

PCIe 3.0 (128b/130b):
  傳輸 1 GB 資料需要：
  1 GB × (130/128) = 1.016 GB 的實際傳輸
  效率：98.5%
```

**為什麼不用 100% 效率的編碼？**

因為需要保證 **DC balance**（直流平衡）：

```
不平衡的訊號：
  ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
  │ │ │ │ │ │ │ │ │ │  ← 連續的 1
──┘ └─┘ └─┘ └─┘ └─┘ └──

  問題：接收端無法判斷時脈邊界

平衡的訊號：
  ┌─┐   ┌─┐   ┌─┐   ┌─┐
  │ │   │ │   │ │   │ │  ← 1 和 0 交替
──┘ └───┘ └───┘ └───┘ └──

  優點：接收端可以從訊號變化中恢復時脈
```

---

### 3.3 PCIe 4.0/5.0：頻寬翻倍

**PCIe 4.0 的挑戰**：

```
PCIe 3.0: 8 GT/s
PCIe 4.0: 16 GT/s  ← 頻率翻倍

問題：
1. 訊號完整性 (Signal Integrity) 變差
2. 需要更好的 PCB 設計
3. 需要更好的 PHY 電路
```

**解決方案**：

1. **更嚴格的 PCB 規範**：
   - 阻抗控制：100Ω ± 10%
   - 走線長度匹配：± 5 mil
   - 避免 via（過孔）

2. **更好的等化器 (Equalizer)**：
   - 補償高頻訊號衰減
   - 減少 ISI (Inter-Symbol Interference)

3. **更低的抖動 (Jitter)**：
   - 時脈抖動 < 1 ps

---

### 3.4 PCIe 6.0：PAM4 調變

**PCIe 6.0 的革命**：使用 **PAM4 (Pulse Amplitude Modulation 4-level)** 調變。

**NRZ vs PAM4**：

```
NRZ (Non-Return-to-Zero) - PCIe 5.0 使用：
  每個符號傳輸 1 bit

  ┌─┐     ┌─┐
  │1│     │1│
──┘ └─────┘ └──
    0       0

  1 符號 = 1 bit

PAM4 - PCIe 6.0 使用：
  每個符號傳輸 2 bits

  ┌─────┐
  │ 11  │  ← 最高電壓
  ├─────┤
  │ 10  │  ← 中高電壓
  ├─────┤
  │ 01  │  ← 中低電壓
  ├─────┤
  │ 00  │  ← 最低電壓
  └─────┘

  1 符號 = 2 bits
```

**優勢**：

```
PCIe 5.0 (NRZ):
  32 GT/s × 1 bit/symbol = 32 Gbps per lane

PCIe 6.0 (PAM4):
  32 GT/s × 2 bits/symbol = 64 Gbps per lane

頻寬翻倍，但符號速率不變！
```

**挑戰**：

- PAM4 需要更高的 SNR (Signal-to-Noise Ratio)
- 需要更複雜的 DSP (Digital Signal Processing)

---

## 四、PCIe 的關鍵機制

### 4.1 Configuration Space：裝置識別

每個 PCIe 裝置都有一個 **Configuration Space**（配置空間），用來儲存裝置資訊。

**Configuration Space 結構**：

```
┌────────────────────────────────────────────────┐
│  Header (64 bytes)                             │
│  ┌──────────────────────────────────────────┐  │
│  │ 0x00: Vendor ID (2B) │ Device ID (2B)   │  │
│  ├──────────────────────────────────────────┤  │
│  │ 0x04: Command (2B)   │ Status (2B)      │  │
│  ├──────────────────────────────────────────┤  │
│  │ 0x08: Revision ID (1B) │ Class Code (3B)│  │
│  ├──────────────────────────────────────────┤  │
│  │ 0x10: BAR0 (Base Address Register 0)    │  │
│  │ 0x14: BAR1                               │  │
│  │ 0x18: BAR2                               │  │
│  │ ...                                      │  │
│  └──────────────────────────────────────────┘  │
├────────────────────────────────────────────────┤
│  Capabilities (192 bytes)                      │
│  - MSI/MSI-X                                   │
│  - Power Management                            │
│  - PCIe Extended Capabilities                  │
└────────────────────────────────────────────────┘
```

**讀取 Configuration Space**：

```c
// Linux kernel: drivers/pci/access.c
int pci_read_config_word(struct pci_dev *dev, int where, u16 *val)
{
    // 透過 MMIO 或 I/O port 讀取 Configuration Space
    return pci_bus_read_config_word(dev->bus, dev->devfn, where, val);
}

// 範例：讀取 Vendor ID 和 Device ID
u16 vendor_id, device_id;
pci_read_config_word(dev, PCI_VENDOR_ID, &vendor_id);
pci_read_config_word(dev, PCI_DEVICE_ID, &device_id);

printk("Vendor: 0x%04x, Device: 0x%04x\n", vendor_id, device_id);
```

**lspci 的輸出**：

```bash
$ lspci -v
01:00.0 Non-Volatile memory controller: Samsung Electronics Co Ltd NVMe SSD Controller SM981/PM981/PM983
        Subsystem: Samsung Electronics Co Ltd NVMe SSD Controller SM981/PM981/PM983
        Flags: bus master, fast devsel, latency 0, IRQ 16, NUMA node 0
        Memory at f7d00000 (64-bit, non-prefetchable) [size=16K]
        Capabilities: [40] Power Management version 3
        Capabilities: [50] MSI: Enable+ Count=1/32 Maskable- 64bit+
        Capabilities: [70] Express Endpoint, MSI 00
        Capabilities: [b0] MSI-X: Enable- Count=33 Masked-
```

---

### 4.2 MMIO (Memory-Mapped I/O)：記憶體映射

PCIe 裝置的暫存器通常透過 **MMIO** 存取。

**MMIO 的概念**：

```
CPU 的記憶體空間：
┌────────────────────────────────────┐
│  0x00000000 - 0x7FFFFFFF           │  ← RAM
│  (2 GB)                            │
├────────────────────────────────────┤
│  0x80000000 - 0xF7CFFFFF           │  ← RAM
│  (1.9 GB)                          │
├────────────────────────────────────┤
│  0xF7D00000 - 0xF7D03FFF           │  ← NVMe SSD 的暫存器
│  (16 KB)                           │
├────────────────────────────────────┤
│  0xF8000000 - 0xFFFFFFFF           │  ← 其他裝置
└────────────────────────────────────┘
```

**存取 MMIO**：

```c
// Linux kernel: drivers/nvme/host/pci.c
struct nvme_dev {
    void __iomem *bar;  // MMIO 基址
};

// 映射 MMIO
dev->bar = ioremap(pci_resource_start(pdev, 0),
                   pci_resource_len(pdev, 0));

// 讀取暫存器
u32 cap = readl(dev->bar + NVME_REG_CAP);

// 寫入暫存器
writel(0x00460001, dev->bar + NVME_REG_CC);
```

**為什麼用 MMIO 而不是 I/O Port？**

1. **地址空間大**：MMIO 可以用 64-bit 地址，I/O Port 只有 16-bit
2. **效率高**：MMIO 可以用 CPU 的 cache，I/O Port 不行
3. **簡單**：MMIO 就是普通的記憶體存取，不需要特殊指令

---

### 4.3 DMA (Direct Memory Access)：零拷貝傳輸

**DMA 的概念**：

```
沒有 DMA：
┌─────┐     ┌─────┐     ┌─────┐
│ SSD │ ──→ │ CPU │ ──→ │ RAM │
└─────┘     └─────┘     └─────┘
  1. SSD 讀取資料到內部緩衝區
  2. CPU 從 SSD 讀取資料（透過 MMIO）
  3. CPU 寫入 RAM

  問題：CPU 要參與每個 byte 的傳輸，效率低

有 DMA：
┌─────┐                 ┌─────┐
│ SSD │ ──────────────→ │ RAM │
└─────┘                 └─────┘
  1. CPU 告訴 SSD：「把資料寫到 RAM 的 0x12345000」
  2. SSD 直接透過 PCIe 寫入 RAM
  3. 完成後，SSD 發送中斷通知 CPU

  優點：CPU 不用參與資料傳輸，可以做其他事
```

**DMA 的實作**：

```c
// Linux kernel: drivers/nvme/host/pci.c

// 1. 分配 DMA 緩衝區
dma_addr_t dma_addr;
void *buffer = dma_alloc_coherent(&pdev->dev, size, &dma_addr, GFP_KERNEL);

// 2. 設定 NVMe 命令
struct nvme_command cmd;
cmd.rw.opcode = nvme_cmd_read;
cmd.rw.prp1 = cpu_to_le64(dma_addr);  // DMA 地址
cmd.rw.length = cpu_to_le16((size >> 9) - 1);  // 以 512B 為單位

// 3. 發送命令到 NVMe SSD
nvme_submit_cmd(nvmeq, &cmd);

// 4. 等待中斷
wait_for_completion(&nvmeq->cq_wait);

// 5. 資料已經在 buffer 中了！
```

---

### 4.4 MSI/MSI-X：高效中斷機制

**傳統中斷 (INTx) 的問題**：

```
┌─────┐     ┌─────┐     ┌─────┐
│ SSD │ ──→ │ APIC│ ──→ │ CPU │
└─────┘     └─────┘     └─────┘

  1. SSD 拉高 IRQ 線
  2. APIC 通知 CPU
  3. CPU 讀取 SSD 的狀態暫存器，確認是哪個裝置

  問題：
  - 需要額外的 IRQ 線
  - 多個裝置可能共用 IRQ，需要輪詢
  - 延遲高
```

**MSI (Message Signaled Interrupts)**：

```
┌─────┐                 ┌─────┐
│ SSD │ ──────────────→ │ CPU │
└─────┘                 └─────┘

  1. SSD 寫入一個特殊的記憶體地址（MSI address）
  2. CPU 的中斷控制器收到這個寫入，產生中斷

  優點：
  - 不需要 IRQ 線
  - 每個裝置有獨立的中斷向量
  - 延遲低
```

**MSI-X 的改進**：

```
MSI:
  - 最多 32 個中斷向量
  - 所有中斷共用一個 MSI address

MSI-X:
  - 最多 2048 個中斷向量
  - 每個中斷有獨立的 MSI address 和 data
  - 可以指定中斷送到哪個 CPU core
```

**NVMe 使用 MSI-X**：

```c
// Linux kernel: drivers/nvme/host/pci.c
static int nvme_setup_irqs(struct nvme_dev *dev)
{
    // 分配 MSI-X 向量
    int nr_io_queues = num_online_cpus();
    int result = pci_alloc_irq_vectors(pdev, 1, nr_io_queues + 1,
                                       PCI_IRQ_MSIX | PCI_IRQ_AFFINITY);

    // 為每個 I/O queue 分配一個中斷
    for (i = 0; i < nr_io_queues; i++) {
        int irq = pci_irq_vector(pdev, i + 1);
        request_irq(irq, nvme_irq, 0, "nvme_queue", &dev->queues[i]);
    }
}
```

---

## 五、PCIe 在 NVMe SSD 中的應用

### 5.1 為什麼 NVMe 選擇 PCIe？

**SATA 的限制**：

```
SATA III:
  頻寬：600 MB/s
  佇列：單一佇列，深度 32
  延遲：~2-3 μs
```

**PCIe 的優勢**：

```
PCIe 3.0 x4:
  頻寬：4 GB/s  ← 是 SATA 的 6.7 倍
  佇列：多佇列，每個深度 64K
  延遲：~0.5-1 μs  ← 是 SATA 的 1/3
```

---

### 5.2 NVMe over PCIe 的架構

```
┌─────────────────────────────────────────────────┐
│                 作業系統                         │
│  ┌──────────────────────────────────────────┐   │
│  │         NVMe Driver                      │   │
│  │  ┌────────────────────────────────────┐  │   │
│  │  │ Admin Queue (1 個)                 │  │   │
│  │  ├────────────────────────────────────┤  │   │
│  │  │ I/O Queue 0 (CPU 0)                │  │   │
│  │  │ I/O Queue 1 (CPU 1)                │  │   │
│  │  │ I/O Queue 2 (CPU 2)                │  │   │
│  │  │ ...                                │  │   │
│  │  └────────────────────────────────────┘  │   │
│  └──────────────────┬───────────────────────┘   │
└─────────────────────┼───────────────────────────┘
                      │ PCIe TLP (Memory Write)
                      │
┌─────────────────────▼───────────────────────────┐
│              PCIe Root Complex                  │
└─────────────────────┬───────────────────────────┘
                      │ PCIe x4
                      │
┌─────────────────────▼───────────────────────────┐
│              NVMe SSD Controller                │
│  ┌──────────────────────────────────────────┐   │
│  │  PCIe Endpoint                           │   │
│  │  - 接收 TLP                              │   │
│  │  - 解析 NVMe 命令                        │   │
│  │  - DMA 資料到 Host Memory                │   │
│  │  - 發送 MSI-X 中斷                       │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

---

### 5.3 實測：PCIe 世代對 NVMe 性能的影響

**測試環境**：

- SSD: Samsung 980 PRO 1TB (支援 PCIe 4.0 x4)
- 主機板: ASUS ROG Strix B550-F
- CPU: AMD Ryzen 7 5800X

**測試 1：循序讀取**

```bash
fio --name=seq_read --filename=/dev/nvme0n1 --rw=read --bs=128k \
    --iodepth=32 --direct=1 --runtime=60 --time_based
```

**結果**：

```
PCIe 3.0 x4 (4 GB/s):
  頻寬: 3,500 MB/s  ← 接近 PCIe 3.0 上限

PCIe 4.0 x4 (8 GB/s):
  頻寬: 7,000 MB/s  ← 翻倍！
```

**測試 2：隨機讀取**

```bash
fio --name=rand_read --filename=/dev/nvme0n1 --rw=randread --bs=4k \
    --iodepth=128 --direct=1 --runtime=60 --time_based
```

**結果**：

```
PCIe 3.0 x4:
  IOPS: 450,000
  延遲: 0.28 ms

PCIe 4.0 x4:
  IOPS: 1,000,000  ← 提升 2.2 倍
  延遲: 0.13 ms
```

**觀察**：

- 循序讀取受限於 PCIe 頻寬
- 隨機讀取受限於 SSD 內部並行能力和 PCIe 延遲

---

## 六、PCIe 的進階主題

### 6.1 PCIe Bifurcation：拆分 lanes

**Bifurcation 的概念**：

```
原始配置：
┌─────────────────────────────────┐
│  PCIe Slot (x16)                │
│  ┌───────────────────────────┐  │
│  │  GPU (x16)                │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘

Bifurcation 後：
┌─────────────────────────────────┐
│  PCIe Slot (x16)                │
│  ┌─────────┐┌─────────┐┌──────┐ │
│  │ NVMe x4 ││ NVMe x4 ││GPU x8│ │
│  └─────────┘└─────────┘└──────┘ │
└─────────────────────────────────┘
```

**應用場景**：

- 在一個 x16 插槽上接 4 個 NVMe SSD (x4 each)
- 需要主機板和 BIOS 支援

---

### 6.2 PCIe Hot-Plug：熱插拔

**Hot-Plug 的流程**：

```
1. 插入裝置
   ↓
2. PCIe 偵測到 Presence Detect 訊號
   ↓
3. 作業系統收到 Hot-Plug 事件
   ↓
4. 執行 PCIe enumeration
   ↓
5. 載入驅動程式
   ↓
6. 裝置可用
```

**Linux 的 Hot-Plug 支援**：

```bash
# 查看 PCIe 裝置
$ lspci

# 移除裝置
$ echo 1 > /sys/bus/pci/devices/0000:01:00.0/remove

# 重新掃描
$ echo 1 > /sys/bus/pci/rescan
```

---

### 6.3 PCIe AER (Advanced Error Reporting)

**AER 的錯誤類型**：

1. **Correctable Errors**：可修正的錯誤
   - Receiver Error
   - Bad TLP
   - Bad DLLP

2. **Uncorrectable Non-Fatal Errors**：不可修正但不致命
   - Completion Timeout
   - Flow Control Protocol Error

3. **Uncorrectable Fatal Errors**：不可修正且致命
   - Malformed TLP
   - ECRC Error

**查看 AER 錯誤**：

```bash
$ dmesg | grep -i aer
pcieport 0000:00:01.1: AER: Corrected error received: 0000:01:00.0
nvme 0000:01:00.0: PCIe Bus Error: severity=Corrected, type=Physical Layer, (Receiver ID)
```

---

## 七、總結

### PCIe 的優勢

✅ **高頻寬**：PCIe 5.0 x4 = 16 GB/s（是 SATA 的 26 倍）
✅ **低延遲**：點對點連接，沒有仲裁延遲
✅ **可擴展**：從 x1 到 x16，靈活配置
✅ **通用性**：GPU、NVMe、網卡、FPGA 都用 PCIe

### PCIe 的挑戰

⚠️ **訊號完整性**：高速訊號需要精密的 PCB 設計
⚠️ **功耗**：PCIe 5.0/6.0 的 PHY 功耗較高
⚠️ **成本**：高速 PCIe 的控制器和 PHY 較貴

### 未來展望

- **PCIe 6.0**：64 GT/s，使用 PAM4 調變
- **PCIe 7.0**：預計 2025 年發布，128 GT/s
- **CXL**：基於 PCIe 的 cache coherent 互連（下一篇文章）

**下一篇文章**，我們將探討 **NVMe**——專為 SSD 設計的協議，看看它如何充分利用 PCIe 的能力。

---

## 參考資料

1. PCI-SIG: <https://pcisig.com/>
2. PCIe Base Specification 5.0: <https://pcisig.com/specifications>
3. Linux kernel PCIe documentation: <https://www.kernel.org/doc/html/latest/PCI/>
4. Understanding PCIe Configuration Space: <https://wiki.osdev.org/PCI_Express>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
