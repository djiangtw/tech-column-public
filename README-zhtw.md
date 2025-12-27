# Tech Column - 技術專欄

**深入淺出的系統架構與硬體設計**

[![Language](https://img.shields.io/badge/Language-繁體中文-blue)]()
[![Series](https://img.shields.io/badge/Series-6-blue)]()
[![Articles](https://img.shields.io/badge/Articles-90-blue)]()
[![License](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey)](https://creativecommons.org/licenses/by/4.0/)
[![Author](https://img.shields.io/badge/Author-Danny%20Jiang-orange)]()
[![Updated](https://img.shields.io/badge/Updated-Dec%202025-green)]()

---

## 📖 關於本專欄

Tech Column 是一個技術寫作專案，專注於系統架構、硬體設計和性能優化領域。本專欄的目標是用生動的比喻和真實的案例，將複雜的技術概念解釋得清晰易懂，讓讀者不僅知道「是什麼」，更理解「為什麼」。

**關於案例**：本專欄文章中的案例場景均為**模擬場景**，基於業界先進經驗並隱去機敏資訊撰寫。所有內容符合職業道德和保密協議要求，不涉及任何公司的專有技術或商業機密。

### 本專欄的特色

- **生動的比喻**：用圖書館理解 Cache、用停車場理解 Associativity、用城市交通理解 NoC
- **真實的案例**：來自 20+ 年產業經驗的實際問題和解決方案
- **循序漸進**：從入門到進階，系統性地建立知識體系
- **實務導向**：不只是理論，更提供可執行的優化建議和設計原則

---

## 📊 專案統計

| 系列 | 文章數 | 字數 |
|------|--------|------|
| Cache Architecture | 6 篇 | ~20,800 字 |
| Network-on-Chip | 6 篇 | ~14,100 字 |
| Storage Architecture | 9 篇 | ~39,200 字 |
| Embedded RTOS | 8 篇 | ~24,000 字 |
| Bluetooth & IoT | 21 篇 | ~70,000 字 |
| Building danieRTOS | 40 篇 | ~170,000 字 |

**總計**: 90 篇文章，~338,100 字

---

## 📚 文章系列

### 1. Cache Architecture 系列

深入探討 CPU Cache 的設計與優化，從基礎概念到實戰應用。

1. [Cache 基礎概念入門：用圖書館理解 CPU Cache](topics/cache-architecture/01-cache-basics.md)
2. [理解 Cache Associativity：停車場的智慧](topics/cache-architecture/02-cache-associativity.md)
3. [現代 CPU Cache 架構設計：從 L1 到 L3 的設計哲學](topics/cache-architecture/03-modern-cache-design.md)
4. [Cache Coherency 與 MESI 協議：多核心時代的一致性挑戰](topics/cache-architecture/04-cache-coherency-mesi.md)
5. [Cache 性能優化實戰：從理論到實踐](topics/cache-architecture/05-cache-optimization.md)
6. [False Sharing 與多執行緒優化：看不見的性能殺手](topics/cache-architecture/06-false-sharing.md)

---

### 2. Network-on-Chip 系列

探索晶片內部的通訊架構，從 Bus 到 Network 的演進。

1. [Network-on-Chip 入門：從 Bus 到 Network 的演進](topics/network-on-chip/01-noc-introduction.md)
2. [NoC 拓撲結構的圖論分析：從數學到硬體](topics/network-on-chip/02-topology-graph-theory.md)
3. [NoC 路由演算法與死鎖避免：從理論到實作](topics/network-on-chip/03-routing-deadlock.md)
4. [Router 微架構設計：從 Pipeline 到硬體實作](topics/network-on-chip/04-router-microarchitecture.md)
5. [NoC 與 Cache Coherency 整合：多核心的協調藝術](topics/network-on-chip/05-noc-cache-coherency.md)
6. [NoC 與先進封裝：突破物理邊界](topics/network-on-chip/06-noc-advanced-packaging.md)

---

### 3. Storage Architecture 系列

從硬體到軟體的完整視角，深入理解現代儲存系統。

1. [儲存系統導論：從硬碟到 SSD 的演進](topics/storage-architecture/01-introduction.md)
2. [SATA 與 AHCI：傳統儲存介面深度解析](topics/storage-architecture/02-sata-ahci.md)
3. [PCIe 架構：高速互連的基石](topics/storage-architecture/03-pcie.md)
4. [NVMe 協議：為 SSD 而生的介面](topics/storage-architecture/04-nvme.md)
5. [CXL 技術：記憶體與儲存的融合](topics/storage-architecture/05-cxl.md)
6. [FTL 深度解析：SSD 的靈魂](topics/storage-architecture/06-ftl.md)
7. [GC 與 Wear Leveling：SSD 的長壽秘訣](topics/storage-architecture/07-gc-wear-leveling.md)
8. [錯誤校正碼：資料完整性的守護者](topics/storage-architecture/08-error-correction.md)
9. [進階主題：ZNS、Computational Storage](topics/storage-architecture/09-advanced-topics.md)

---

### 4. Embedded RTOS 系列

實務導向的嵌入式 RTOS 開發，以 FreeRTOS + RISC-V 為學習平台。

1. [RTOS 入門：為什麼需要即時作業系統](topics/embedded-rtos/01-rtos-introduction.md)
2. [Scheduler 深度解析：任務調度的藝術](topics/embedded-rtos/02-scheduler-deep-dive.md)
3. [中斷處理：即時系統的心跳](topics/embedded-rtos/03-interrupt-handling.md)
4. [記憶體管理：從 heap_1 到 heap_5](topics/embedded-rtos/04-memory-management.md)
5. [GDB + QEMU 除錯實戰](topics/embedded-rtos/05-debugging-with-gdb-qemu.md)
6. [RTOS SMP：多核心的挑戰](topics/embedded-rtos/06-rtos-smp.md)
7. [Context Switch 組合語言深度解析](topics/embedded-rtos/07-context-switch-assembly.md)
8. [RISC-V 特權模式：M/S/U mode](topics/embedded-rtos/08-privilege-modes.md)

---

### 5. Bluetooth & IoT 系列

BLE 協議棧、無線通訊、IoT 系統整合。

1. [藍牙技術入門](topics/bluetooth-wireless-iot/01-introduction.md)
2. [藍牙協議棧架構](topics/bluetooth-wireless-iot/02-protocol-stack.md)
3. [HCI 層深度解析](topics/bluetooth-wireless-iot/03-hci.md)
4. [L2CAP 協議](topics/bluetooth-wireless-iot/04-l2cap.md)
5. [ATT 與 GATT 協議](topics/bluetooth-wireless-iot/05-att-gatt.md)
6. [SMP 安全管理](topics/bluetooth-wireless-iot/06-smp.md)
7. [Beacon 技術](topics/bluetooth-wireless-iot/07-beacon.md)
8. [PHY 與 RF 基礎](topics/bluetooth-wireless-iot/08-phy-rf.md)
9. [WiFi/BT 共存](topics/bluetooth-wireless-iot/09-wifi-bt-coexistence.md)
10. [SPI 介面](topics/bluetooth-wireless-iot/10-spi.md)
11. [MIPI DSI 介面](topics/bluetooth-wireless-iot/11-mipi-dsi.md)
12. [I2C/UART/GPIO 介面](topics/bluetooth-wireless-iot/12-i2c-uart-gpio.md)
13. [BLE 功耗優化](topics/bluetooth-wireless-iot/13-ble-power-optimization.md)
14. [系統功耗優化](topics/bluetooth-wireless-iot/14-system-power-optimization.md)
15. [BLE 除錯技巧](topics/bluetooth-wireless-iot/15-ble-debugging.md)
16. [Bluetooth SIG 認證](topics/bluetooth-wireless-iot/16-bluetooth-sig-certification.md)
17. [Zigbee vs Bluetooth 比較](topics/bluetooth-wireless-iot/17-zigbee-vs-bluetooth.md)
18. [RF4CE、Thread、Matter](topics/bluetooth-wireless-iot/18-rf4ce-thread-matter.md)
19. [AIoT 整合](topics/bluetooth-wireless-iot/19-aiot.md)
20. [智慧手錶案例研究](topics/bluetooth-wireless-iot/20-smartwatch-case-study.md)
21. [IoT 安全](topics/bluetooth-wireless-iot/21-iot-security.md)

---

### 6. Building danieRTOS 系列

從零打造 RISC-V RTOS，故事化寫作風格，40 篇完整教學。

**danieRTOS** 是一個教育用途的 minimal RTOS，運行於 RISC-V 架構。本系列記錄從零開始打造 RTOS 的完整過程。

| 版本 | 別名 | 章節 | 核心功能 |
|------|------|------|----------|
| v0.x | Nano | 01-12 | 基礎 RTOS：Task, Scheduler, Semaphore, Mutex, Queue |
| v1.x | Secure | 13-19 | User Mode：PMP, Syscall, Fault Handling |
| v2.x | MSMP | 20-30 | SMP：Spinlock, IPI, Multi-core Scheduler |
| v3.x | SMP | 31-40 | 整合：SMP + User Mode + Fault Isolation |

**文章列表**：參見 [topics/building-daniertos/README.md](topics/building-daniertos/README.md)

---

## 🎯 目標讀者

本專欄適合：

- **系統軟體工程師**：想要理解硬體如何影響軟體性能
- **嵌入式工程師**：從事 RTOS、驅動程式、韌體開發
- **硬體工程師**：從事 CPU、SoC 設計和驗證
- **IoT 開發者**：從事藍牙、無線通訊、物聯網開發
- **電腦架構學生**：在真實世界情境中學習系統架構

**先備知識**：
- 基本的計算機組織概念
- 理解 CPU、記憶體、匯流排等基本元件
- 有 C 語言程式設計經驗（部分系列需要）

---

## 📄 授權

**版權所有 © 2025 Danny Jiang**

本專欄所有文章採用 **Creative Commons Attribution 4.0 International License (CC BY 4.0)** 授權。

**您可以自由地：**

- **分享** — 以任何媒介或格式複製及散布本素材
- **修改** — 重混、轉換本素材，及依本素材建立新素材，且為任何目的，包含商業性質之使用

**惟需遵守下列條件：**

- **姓名標示** — 您必須給予適當表彰、提供指向本授權條款的連結，以及指出（本作品的原始版本）是否已被變更

**授權條款**：https://creativecommons.org/licenses/by/4.0/

---

## 📖 如何使用本專欄

### 線上閱讀

直接在 GitHub 上瀏覽 Markdown 檔案，從各系列的第一篇開始。

### 離線閱讀

Clone 此 repository：
```bash
git clone https://github.com/djiangtw/tech-column-public.git
cd tech-column-public
```

### 推薦閱讀順序

**硬體架構入門**：Cache Architecture → Network-on-Chip → Storage Architecture

**嵌入式系統**：Embedded RTOS → Building danieRTOS

**無線通訊**：Bluetooth & IoT 系列

---

## 🤝 貢獻

這是一個唯讀的公開 repository。本專欄在私有 repository 中開發。

**歡迎回饋**：
- 針對錯字、錯誤或建議開 issue
- 鼓勵討論和提問

**注意**：無法接受 pull request，因為這是從私有開發 repository 單向同步的。

---

## 👨‍💻 關於作者

**Danny Jiang**

系統軟體工程師，專注於 RISC-V 架構、嵌入式系統、性能優化。20+ 年產業經驗，熱愛用生動的比喻解釋複雜的技術概念。

**其他作品**：

- [See RISC-V Run: Fundamentals](https://github.com/djiangtw/see-riscv-run-public) - RISC-V 架構完整指南
- [Data Structures in Practice](https://github.com/djiangtw/data-structures-in-practice-public) - 硬體導向的資料結構

---

## 🔗 連結

- **GitHub**: <https://github.com/djiangtw/tech-column-public>
- **Email**: djiang.tw@gmail.com
- **LinkedIn**: [linkedin.com/in/danny-jiang-26359644](https://www.linkedin.com/in/danny-jiang-26359644/)

---

## 📝 引用

如果您在研究、教學或文章中引用本專欄，請使用：

```text
Danny Jiang. (2025). Tech Column - 技術專欄：深入淺出的系統架構與硬體設計.
採用 CC BY 4.0 授權. https://github.com/djiangtw/tech-column-public
```

---

**祝閱讀愉快！** 📖

如有任何問題或建議，歡迎透過 GitHub Issues 與我聯繫。
