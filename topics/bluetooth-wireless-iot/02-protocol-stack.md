# Bluetooth 協定堆疊概覽 - 從 PHY 到 Application

**作者**: Danny Jiang  
**日期**: 2025-12-11  

---

## 前言：2,800 頁的聖經

2014 年 6 月，我剛加入無線晶片公司的第一週，主管給了我一份厚達 2,800 頁的文件。

「這是什麼？」我看著這本比字典還厚的文件，問道。

「**Bluetooth Core Specification Version 4.1**，」他說，「藍牙的聖經。」

「我需要全部讀完嗎？」我有點擔心。

「不用，」他笑了笑，「你不需要全部讀完，但至少要理解協定堆疊的架構。從第 2 卷開始看，那裡有完整的架構圖。」

我打開文件，翻到第 2 卷，看到的第一張圖就是 **Bluetooth Protocol Stack Architecture**。

那是一個複雜的分層架構圖，從最底層的 Physical Layer（PHY）一路往上到 Application Layer，中間還有 Link Layer、L2CAP、ATT、GATT、GAP、SMP 等一堆縮寫。

我盯著這張圖看了 10 分鐘，完全看不懂。

「這些層級到底是做什麼的？」我問。

主管走過來，指著圖說：「簡單來說，PHY 負責發送無線電波，Link Layer 負責封包的傳輸和確認，L2CAP 負責資料的分段和重組，ATT/GATT 負責資料的讀寫，GAP 負責裝置的發現和連線，SMP 負責安全性。」

「聽起來很複雜，」我說。

「確實很複雜，」他說，「但每一層都有自己的職責，透過定義好的介面互相溝通。你可以把它想像成一個工廠的生產線：PHY 是原料供應，Link Layer 是組裝線，L2CAP 是包裝部門，ATT/GATT 是品管部門，GAP 是物流部門，SMP 是保全部門。每個部門只負責自己的工作，但整個工廠才能運作。」

「那我要寫的韌體是在哪一層？」

「你主要會在 Link Layer 和 L2CAP 之間工作，」他說，「但你必須理解整個堆疊，因為上層的需求會影響下層的設計，下層的限制也會影響上層的實作。」

那一週，我花了大量時間研讀協定堆疊的架構。我把每一層的職責都寫在筆記本上，畫了無數張圖，試圖理解它們之間的關係。

我發現，理解 Bluetooth 的關鍵不在於記住每個層級的細節，而在於理解**為什麼需要這些層級**，以及**它們如何協同工作**。

這篇文章將帶你從宏觀的角度理解 Bluetooth 協定堆疊，從 PHY 到 Application，從硬體到軟體，從 Classic Bluetooth 到 BLE。就像我當年第一次看到那張架構圖一樣，我們從整體開始，逐步深入每一層。

---

## Bluetooth 協定堆疊的分層架構

### 為什麼需要分層？

在深入細節之前，讓我們先理解**為什麼需要分層架構**。

想像一下，如果沒有分層，你要寫一個智慧手環的韌體，你需要：

1. **控制無線電波的發送和接收**（調變、解調、頻率跳躍）
2. **處理封包的傳輸和確認**（重傳、錯誤檢測）
3. **管理資料的分段和重組**（MTU、分段）
4. **定義資料的格式和讀寫方式**（Services、Characteristics）
5. **處理裝置的發現和連線**（Advertising、Scanning、Connection）
6. **實作安全機制**（Pairing、Encryption）

這些任務的複雜度和時間敏感度差異極大。如果全部混在一起，程式碼會變得難以維護，也難以重用。

**分層架構的優勢**：

- **職責分離**：每一層只負責特定的任務
- **介面標準化**：層與層之間透過定義好的介面溝通
- **可重用性**：下層的實作可以被多個上層重用
- **可測試性**：每一層可以獨立測試

### Bluetooth 協定堆疊的完整架構

Bluetooth 協定堆疊分為兩大部分：**Controller（控制器）** 和 **Host（主機）**，中間透過 **HCI（Host Controller Interface）** 溝通。

```
┌─────────────────────────────────────────────────────────┐
│  Application Layer                                       │
│  ├─ User Application (智慧手環 App)                      │
│  └─ Profiles (Heart Rate Profile, Proximity Profile)    │
├─────────────────────────────────────────────────────────┤
│  Host (主機 - 通常在主處理器上運行)                       │
│  ├─ GAP (Generic Access Profile)                        │
│  ├─ GATT (Generic Attribute Profile)                    │
│  ├─ SMP (Security Manager Protocol)                     │
│  ├─ ATT (Attribute Protocol)                            │
│  └─ L2CAP (Logical Link Control & Adaptation Protocol)  │
├─────────────────────────────────────────────────────────┤
│  HCI (Host Controller Interface)                        │
│  └─ UART, USB, SPI, 或 Function Call                    │
├─────────────────────────────────────────────────────────┤
│  Controller (控制器 - 通常在藍牙晶片上運行)               │
│  ├─ Link Layer (BLE) / Baseband (Classic)               │
│  └─ PHY (Physical Layer - Radio)                        │
└─────────────────────────────────────────────────────────┘
```

### Controller vs Host：硬體與軟體的分界

**Controller（控制器）**：

- **位置**：通常在藍牙晶片（SoC）上運行
- **實作方式**：硬體（PHY）+ 韌體（Link Layer）
- **職責**：
  - **PHY**：無線電波的發送和接收、調變解調、頻率跳躍
  - **Link Layer**：封包的傳輸和確認、連線管理、時序控制
- **特性**：時間敏感度極高（微秒級），需要硬體加速

**Host（主機）**：

- **位置**：通常在主處理器（如 Cortex-M4）上運行
- **實作方式**：軟體（C/C++ Libraries）
- **職責**：
  - **L2CAP**：資料的分段和重組、流量控制
  - **ATT/GATT**：資料的讀寫、Services 和 Characteristics 的管理
  - **GAP**：裝置的發現和連線、角色管理
  - **SMP**：安全機制（Pairing、Encryption）
- **特性**：時間敏感度較低（毫秒級），可以用軟體實作

**HCI（Host Controller Interface）**：

- **作用**：連接 Host 和 Controller 的橋樑
- **實作方式**：
  - **物理介面**：UART、USB、SPI（當 Host 和 Controller 在不同晶片上）
  - **虛擬介面**：Function Call（當 Host 和 Controller 在同一晶片上）
- **封包類型**：
  - **Command**：Host → Controller（如：開始掃描、建立連線）
  - **Event**：Controller → Host（如：掃描到裝置、連線建立）
  - **Data**：雙向傳輸（如：傳輸應用資料）

---

## 從下到上：每一層的職責

### Layer 1: PHY (Physical Layer)

**職責**：無線電波的發送和接收

**核心技術**：

- **頻段**：2.4 GHz ISM Band（2.400 - 2.4835 GHz）
- **調變方式**：GFSK（Gaussian Frequency Shift Keying）
- **頻道數量**：
  - **Classic Bluetooth**：79 個頻道（1 MHz 間隔）
  - **BLE**：40 個頻道（2 MHz 間隔，其中 3 個是 Advertising Channel）
- **傳輸速率**：
  - **Classic Bluetooth**：1-3 Mbps（EDR）
  - **BLE**：1 Mbps（BLE 4.x）、2 Mbps（BLE 5.0）

**實作方式**：

- **類比電路**：射頻前端（RF Front-end）、功率放大器（PA）、低噪聲放大器（LNA）
- **數位電路**：調變器（Modulator）、解調器（Demodulator）、CRC 校驗

**開發者需要知道的**：

- PHY 層通常由晶片廠商實作，開發者不需要直接操作
- 但需要理解 PHY 的限制（如：傳輸距離、功耗、干擾）

### Layer 2: Link Layer (BLE) / Baseband (Classic)

**職責**：封包的傳輸和確認、連線管理、時序控制

**核心功能**：

1. **Advertising（廣播）**：
   - Peripheral 定期發送 Advertising Packet
   - Central 掃描並發現 Peripheral

2. **Scanning（掃描）**：
   - Central 監聽 Advertising Channel
   - 發現 Peripheral 並讀取 Advertising Data

3. **Connection（連線）**：
   - Central 發送 Connection Request
   - 建立連線後，雙方進入 Connected State

4. **Data Transfer（資料傳輸）**：
   - 使用 Data Channel（37 個頻道）
   - 頻率跳躍（Frequency Hopping）避免干擾

5. **Acknowledgment（確認）**：
   - 每個封包都需要 ACK
   - 如果沒有收到 ACK，會自動重傳

**時序控制**：

- **Connection Interval**：兩次資料交換的間隔（7.5 ms - 4 s）
- **Slave Latency**：Slave 可以跳過幾次 Connection Event（降低功耗）
- **Supervision Timeout**：如果超過這個時間沒有收到封包，連線會斷開

**實作方式**：

- **韌體**：運行在藍牙晶片的即時作業系統（RTOS）上
- **狀態機**：管理 Advertising、Scanning、Connection 等狀態

**開發者需要知道的**：

- Link Layer 的參數（Connection Interval、Slave Latency）會直接影響功耗和延遲
- 需要根據應用場景調整這些參數

### Layer 3: HCI (Host Controller Interface)

**職責**：連接 Host 和 Controller

**封包類型**：

1. **Command Packet**（Host → Controller）：
   - 格式：`[Opcode (2 bytes)] [Parameter Length (1 byte)] [Parameters (N bytes)]`
   - 範例：`HCI_LE_Set_Scan_Enable`（開始掃描）

2. **Event Packet**（Controller → Host）：
   - 格式：`[Event Code (1 byte)] [Parameter Length (1 byte)] [Parameters (N bytes)]`
   - 範例：`HCI_LE_Advertising_Report`（掃描到裝置）

3. **Data Packet**（雙向）：
   - 格式：`[Handle (2 bytes)] [Length (2 bytes)] [Data (N bytes)]`
   - 用於傳輸應用資料

**實作方式**：

- **UART**：最常見，速度 115200 - 3000000 bps
- **USB**：速度快，但功耗高
- **SPI**：速度快，Pin 數量多
- **Function Call**：單晶片方案（SoC），Host 和 Controller 在同一晶片上

**開發者需要知道的**：

- 在單晶片方案中，HCI 通常是虛擬的（Function Call）
- 在雙晶片方案中，HCI 的速度會影響整體性能

### Layer 4: L2CAP (Logical Link Control & Adaptation Protocol)

**職責**：資料的分段和重組、流量控制、多工

**核心功能**：

1. **Segmentation and Reassembly（分段和重組）**：
   - Link Layer 的 MTU（Maximum Transmission Unit）是 27 bytes（BLE 4.x）
   - 如果應用資料超過 27 bytes，L2CAP 會自動分段
   - 接收端會自動重組

2. **Multiplexing（多工）**：
   - 多個上層協定（ATT、SMP）可以共用同一個 Link Layer 連線
   - L2CAP 使用 Channel ID 區分不同的協定

3. **Flow Control（流量控制）**：
   - 避免發送端發送過快，接收端來不及處理

**封包格式**：

```
┌─────────────────────────────────────────────────────────┐
│  L2CAP Packet                                            │
├─────────────────────────────────────────────────────────┤
│  Length (2 bytes)    - 資料長度                          │
│  Channel ID (2 bytes) - 通道 ID（0x0004 = ATT）         │
│  Data (N bytes)      - 應用資料                          │
└─────────────────────────────────────────────────────────┘
```

**開發者需要知道的**：

- L2CAP 的 MTU 會影響資料傳輸的效率
- BLE 4.2 引入了 Data Length Extension，可以將 MTU 提升到 251 bytes

### Layer 5: ATT (Attribute Protocol)

**職責**：定義資料的讀寫方式

**核心概念**：

- **Attribute（屬性）**：一個資料單元，包含：
  - **Handle**：16-bit 的唯一識別碼
  - **Type**：UUID（16-bit 或 128-bit）
  - **Value**：實際的資料
  - **Permissions**：讀寫權限

**ATT 操作**：

| 操作 | 方向 | 說明 |
|------|------|------|
| **Read Request** | Client → Server | 讀取 Attribute 的值 |
| **Read Response** | Server → Client | 回傳 Attribute 的值 |
| **Write Request** | Client → Server | 寫入 Attribute 的值（需要 ACK） |
| **Write Response** | Server → Client | 確認寫入成功 |
| **Write Command** | Client → Server | 寫入 Attribute 的值（不需要 ACK） |
| **Notification** | Server → Client | 主動推送資料（不需要 ACK） |
| **Indication** | Server → Client | 主動推送資料（需要 ACK） |

**範例：讀取心率**

```
Client (手機)                    Server (心率帶)
     |                                |
     |  Read Request (Handle=0x0010)  |
     |------------------------------->|
     |                                |
     |  Read Response (Value=75 bpm)  |
     |<-------------------------------|
     |                                |
```

**開發者需要知道的**：

- ATT 是 GATT 的基礎
- Notification 和 Indication 的差異：Notification 不需要 ACK，速度快但不可靠；Indication 需要 ACK，速度慢但可靠

### Layer 6: GATT (Generic Attribute Profile)

**職責**：定義資料的組織方式（Services 和 Characteristics）

**核心概念**：

1. **Service（服務）**：
   - 一組相關的 Characteristics
   - 範例：Heart Rate Service（心率服務）

2. **Characteristic（特徵值）**：
   - 一個具體的資料項目
   - 範例：Heart Rate Measurement（心率測量值）

3. **Descriptor（描述符）**：
   - Characteristic 的額外資訊
   - 範例：Client Characteristic Configuration Descriptor（CCCD，用於啟用 Notification）

**GATT 層次結構**：

```
┌─────────────────────────────────────────────────────────┐
│  Profile (例如：Heart Rate Profile)                      │
│  └─ Service (例如：Heart Rate Service, UUID=0x180D)     │
│      ├─ Characteristic (Heart Rate Measurement)         │
│      │   ├─ Value (75 bpm)                              │
│      │   └─ Descriptor (CCCD - 啟用 Notification)       │
│      └─ Characteristic (Body Sensor Location)           │
│          └─ Value (Chest)                               │
└─────────────────────────────────────────────────────────┘
```

**範例：Heart Rate Service**

```c
// Heart Rate Service (UUID: 0x180D)
Service {
    UUID: 0x180D,
    Characteristics: [
        // Heart Rate Measurement (UUID: 0x2A37)
        Characteristic {
            UUID: 0x2A37,
            Properties: Notify,
            Value: [0x00, 0x4B],  // 75 bpm
            Descriptors: [
                // CCCD (UUID: 0x2902)
                Descriptor {
                    UUID: 0x2902,
                    Value: [0x01, 0x00]  // Enable Notification
                }
            ]
        },
        // Body Sensor Location (UUID: 0x2A38)
        Characteristic {
            UUID: 0x2A38,
            Properties: Read,
            Value: [0x01]  // Chest
        }
    ]
}
```

**開發者需要知道的**：

- GATT 是 BLE 應用開發的核心
- Bluetooth SIG 定義了許多標準的 Services 和 Characteristics（如：Heart Rate、Battery、Temperature）
- 也可以自定義 Services 和 Characteristics（使用 128-bit UUID）

### Layer 7: GAP (Generic Access Profile)

**職責**：裝置的發現和連線、角色管理

**核心角色**：

1. **Broadcaster（廣播者）**：
   - 只發送 Advertising Packet，不接受連線
   - 範例：Beacon

2. **Observer（觀察者）**：
   - 只掃描 Advertising Packet，不發起連線
   - 範例：掃描器

3. **Peripheral（周邊裝置）**：
   - 發送 Advertising Packet，接受連線
   - 連線後作為 Slave
   - 範例：智慧手環、心率帶

4. **Central（中心裝置）**：
   - 掃描 Advertising Packet，發起連線
   - 連線後作為 Master
   - 範例：手機、平板

**Advertising Data**：

- **Advertising Data**：最多 31 bytes
- **Scan Response Data**：最多 31 bytes（當 Central 發送 Scan Request 時回傳）

**常見的 Advertising Data 類型**：

| Type | 說明 | 範例 |
|------|------|------|
| **Flags** | 裝置的能力 | LE General Discoverable Mode |
| **Complete Local Name** | 裝置名稱 | "Mi Band 2" |
| **TX Power Level** | 發射功率 | 0 dBm |
| **Service UUIDs** | 支援的 Services | Heart Rate Service (0x180D) |
| **Manufacturer Data** | 廠商自定義資料 | iBeacon 的 UUID, Major, Minor |

**開發者需要知道的**：

- GAP 定義了裝置的角色和行為
- Advertising Data 的大小限制（31 bytes）需要精心設計

### Layer 8: SMP (Security Manager Protocol)

**職責**：安全機制（Pairing、Encryption）

**核心概念**：

1. **Pairing（配對）**：
   - 建立長期金鑰（Long Term Key, LTK）
   - 配對方法：
     - **Just Works**：無需使用者互動（不安全）
     - **Passkey Entry**：輸入 6 位數字（中等安全）
     - **Numeric Comparison**：比較 6 位數字（BLE 4.2+，安全）
     - **Out of Band (OOB)**：使用 NFC 等其他通道（最安全）

2. **Bonding（綁定）**：
   - 儲存 LTK，下次連線時可以直接使用（不需要重新配對）

3. **Encryption（加密）**：
   - 使用 AES-128 加密資料傳輸

4. **Privacy（隱私）**：
   - 使用隨機位址（Random Address）避免被追蹤

**Pairing 流程**：

```
Central (手機)                    Peripheral (智慧手環)
     |                                |
     |  Pairing Request               |
     |------------------------------->|
     |                                |
     |  Pairing Response              |
     |<-------------------------------|
     |                                |
     |  (使用者輸入 Passkey: 123456)   |
     |                                |
     |  Pairing Confirm               |
     |------------------------------->|
     |                                |
     |  Pairing Random                |
     |<-------------------------------|
     |                                |
     |  (驗證成功，生成 LTK)           |
     |                                |
     |  Encryption Information        |
     |<-------------------------------|
     |                                |
     |  (連線已加密)                   |
     |                                |
```

**開發者需要知道的**：

- Pairing 方法的選擇取決於裝置的 I/O 能力（有無螢幕、鍵盤）
- Bonding 可以提升使用者體驗（不需要每次都配對）
- Privacy 功能可以避免被追蹤（但會增加功耗）

---

## Classic Bluetooth vs BLE：協定堆疊的差異

雖然都叫 Bluetooth，但 Classic Bluetooth 和 BLE 的協定堆疊有很大的差異。

### 架構對比

| 層級 | Classic Bluetooth | BLE |
|------|------------------|-----|
| **Application** | Profiles (A2DP, HFP, SPP) | Profiles (Heart Rate, Proximity) |
| **Host** | SDP, RFCOMM, L2CAP | GATT, ATT, L2CAP |
| **HCI** | HCI | HCI |
| **Controller** | Baseband, LMP | Link Layer |
| **PHY** | 79 channels, 1 MHz spacing | 40 channels, 2 MHz spacing |

### 主要差異

1. **連線模式**：
   - **Classic**：持續連線（Always Connected）
   - **BLE**：睡眠/喚醒模式（Duty Cycling）

2. **資料模型**：
   - **Classic**：Stream-based（如：音訊串流）
   - **BLE**：Attribute-based（如：讀寫 Characteristics）

3. **功耗**：
   - **Classic**：高（數十 mA）
   - **BLE**：極低（數十 μA）

4. **應用場景**：
   - **Classic**：音訊、檔案傳輸
   - **BLE**：感測器、控制、Beacon

---

## 一個完整的 BLE 連線生命週期

讓我們用一個實際的例子，看看 BLE 協定堆疊如何協同工作。

**場景**：手機連接智慧手環，讀取心率

### 階段 1：Advertising（廣播）

```
智慧手環 (Peripheral):
1. Link Layer: 每 100 ms 在 3 個 Advertising Channel 發送 Advertising Packet
2. GAP: Advertising Data 包含裝置名稱 "Mi Band 2" 和 Heart Rate Service UUID
```

### 階段 2：Scanning（掃描）

```
手機 (Central):
1. Link Layer: 監聽 Advertising Channel
2. GAP: 發現 "Mi Band 2"，顯示在 App 的裝置列表中
```

### 階段 3：Connection（連線）

```
手機 (Central):
1. Link Layer: 發送 Connection Request
2. 協商 Connection Parameters (Interval=30ms, Latency=0, Timeout=5s)

智慧手環 (Peripheral):
1. Link Layer: 接受連線，進入 Connected State
2. 停止 Advertising
```

### 階段 4：Service Discovery（服務發現）

```
手機 (Central):
1. GATT: 發送 "Discover All Primary Services" 請求
2. ATT: Read Request (Handle=0x0001)

智慧手環 (Peripheral):
1. ATT: Read Response (Heart Rate Service, UUID=0x180D, Handle=0x0010)
2. GATT: 回傳 Service 列表
```

### 階段 5：Enable Notification（啟用通知）

```
手機 (Central):
1. GATT: 寫入 CCCD (Handle=0x0012, Value=0x0001)
2. ATT: Write Request

智慧手環 (Peripheral):
1. ATT: Write Response
2. 開始定期發送心率資料
```

### 階段 6：Data Transfer（資料傳輸）

```
智慧手環 (Peripheral):
1. 每 1 秒測量心率
2. GATT: 更新 Heart Rate Measurement Characteristic (Value=75 bpm)
3. ATT: Notification (Handle=0x0011, Value=[0x00, 0x4B])
4. L2CAP: 封裝成 L2CAP Packet
5. Link Layer: 發送 Data Packet

手機 (Central):
1. Link Layer: 接收 Data Packet
2. L2CAP: 解析 L2CAP Packet
3. ATT: 解析 Notification
4. GATT: 更新 UI，顯示心率 75 bpm
```

### 階段 7：Disconnection（斷線）

```
手機 (Central):
1. Link Layer: 發送 Disconnect Request

智慧手環 (Peripheral):
1. Link Layer: 進入 Standby State
2. GAP: 重新開始 Advertising
```

---

## 總結

Bluetooth 協定堆疊是一個精心設計的分層架構，每一層都有明確的職責：

- **PHY**：無線電波的發送和接收
- **Link Layer**：封包的傳輸和確認、連線管理
- **HCI**：連接 Host 和 Controller
- **L2CAP**：資料的分段和重組、流量控制
- **ATT**：定義資料的讀寫方式
- **GATT**：定義資料的組織方式（Services 和 Characteristics）
- **GAP**：裝置的發現和連線、角色管理
- **SMP**：安全機制（Pairing、Encryption）

理解這些層級的職責和互動方式，是開發 BLE 應用的基礎。

在下一篇文章中，我們將深入探討 **HCI（Host Controller Interface）**，了解硬體與軟體的分界線。

---

## 參考資料

1. Bluetooth SIG. (2013). *Bluetooth Core Specification Version 4.1*. <https://www.bluetooth.com/specifications/specs/core-specification-4-1/>
2. Bluetooth SIG. (2014). *Bluetooth Core Specification Version 4.2*. <https://www.bluetooth.com/specifications/specs/core-specification-4-2/>
3. Townsend, K., Cufí, C., Akiba, & Davidson, R. (2014). *Getting Started with Bluetooth Low Energy*. O'Reilly Media.
4. Heydon, R. (2013). *Bluetooth Low Energy: The Developer's Handbook*. Prentice Hall.
5. Nordic Semiconductor. (2014). *nRF51 Series Reference Manual*. <https://www.nordicsemi.com/>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
