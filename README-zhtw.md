# Tech Column - 技術專欄

**深入淺出的系統架構與硬體設計**

[![Language](https://img.shields.io/badge/Language-繁體中文-blue)]()
[![Series](https://img.shields.io/badge/Series-6-blue)]()
[![Articles](https://img.shields.io/badge/Articles-93-blue)]()
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
| Storage Architecture | 12 篇 | ~52,000 字 |
| Embedded RTOS | 8 篇 | ~24,000 字 |
| Bluetooth & IoT | 21 篇 | ~70,000 字 |
| Building danieRTOS | 40 篇 | ~170,000 字 |

**總計**: 93 篇文章，~350,900 字

---

## 📚 文章系列

### 1. Cache Architecture 系列（6 篇）

深入探討 CPU Cache 的設計與優化，從基礎概念到實戰應用。

主題：Cache 基礎、Associativity、現代 Cache 設計（L1-L3）、MESI 協議、性能優化、False Sharing

---

### 2. Network-on-Chip 系列（6 篇）

探索晶片內部的通訊架構，從 Bus 到 Network 的演進。

主題：NoC 入門、圖論拓撲分析、路由與死鎖、Router 微架構、Cache Coherency 整合、先進封裝

---

### 3. Storage Architecture 系列（12 篇）

從硬體到軟體的完整視角，深入理解現代儲存系統。

主題：HDD 到 SSD 演進、SATA/AHCI、PCIe 架構、NVMe 協議、CXL 技術、FTL、GC 與 Wear Leveling、錯誤校正、ZNS、資料庫優化、AI/ML 工作負載、雲端儲存

---

### 4. Embedded RTOS 系列（8 篇）

實務導向的嵌入式 RTOS 開發，以 FreeRTOS + RISC-V 為學習平台。

主題：RTOS 入門、Scheduler 深度解析、中斷處理、記憶體管理、GDB+QEMU 除錯、SMP 挑戰、Context Switch 組合語言、RISC-V 特權模式

---

### 5. Bluetooth & IoT 系列（21 篇）

BLE 協議棧、無線通訊、IoT 系統整合。

主題：BLE 協議棧（HCI、L2CAP、ATT/GATT、SMP）、PHY/RF、WiFi/BT 共存、硬體介面（SPI、MIPI、I2C/UART/GPIO）、功耗優化、除錯、認證、Zigbee 比較、Thread/Matter、AIoT、安全

---

### 6. Building danieRTOS 系列（40 篇）

從零打造 RISC-V RTOS，故事化寫作風格，40 篇完整教學。

**danieRTOS** 是一個教育用途的 minimal RTOS，運行於 RISC-V 架構。

| 版本 | 別名 | 章節 | 核心功能 |
|------|------|------|----------|
| v0.x | Nano | 01-12 | 基礎 RTOS：Task, Scheduler, Semaphore, Mutex, Queue |
| v1.x | Secure | 13-19 | User Mode：PMP, Syscall, Fault Handling |
| v2.x | MSMP | 20-30 | SMP：Spinlock, IPI, Multi-core Scheduler |
| v3.x | SMP | 31-40 | 整合：SMP + User Mode + Fault Isolation |

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
