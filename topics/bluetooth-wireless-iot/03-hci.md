# HCI - 硬體與軟體的分界線

**作者**: Danny Jiang  
**日期**: 2025-12-11  

---

## 前言：iPhone 連不上的謎團

2014 年 8 月，我遇到了一個讓我抓狂的問題：智慧手環無法連接到 iPhone。

「在 Android 手機上明明可以正常連線，」我對主管說，「為什麼 iPhone 就不行？」

「iPhone 上是什麼狀況？」他問。

「連線建立後，幾秒鐘就斷開，」我說，「然後 iPhone 顯示『連線失敗』。」

「你檢查過 HCI 命令序列了嗎？」

「還沒，」我說，「我該怎麼檢查？」

「用示波器量測 UART 訊號，」他說，「看看 iPhone 發送了什麼命令，我們的韌體回應了什麼。」

我花了一整天時間，用示波器量測 UART 訊號。我把每個 byte 都記錄下來，然後對照 Bluetooth Core Specification，逐一解析每個 HCI 命令。

終於，我找到了問題。

「你看這裡，」我指著示波器上的波形，「iPhone 發送了一個命令：`01 22 20 04 FB 00 F4 01`。」

「這是什麼命令？」主管問。

我翻開規格文件，查找 Opcode `0x2022`：

「這是 `HCI_LE_Set_Data_Length` 命令，」我說，「Bluetooth 4.2 的新功能，用來設定 Link Layer 的 MTU。」

「我們的韌體支援這個命令嗎？」

「不支援，」我說，「我們的晶片是 Bluetooth 4.1，還沒有這個功能。」

「那我們的韌體回應了什麼？」

我看了一下示波器的記錄：「什麼都沒有。我們的韌體收到這個命令後，直接忽略了。」

「這就是問題所在，」主管說，「iPhone 發送命令後，等待回應。如果沒有收到回應，iPhone 會認為連線失敗，然後斷開。」

「那怎麼辦？」我問。

「有兩個選擇，」他說，「第一，實作這個功能，支援 Data Length Extension。第二，回傳一個錯誤碼，告訴 iPhone 我們不支援這個功能。」

「實作這個功能需要多久？」

「至少兩週，」他說，「而且需要修改 Link Layer 的韌體，風險很高。」

「那我選第二個方案，」我說。

我修改了 HCI 命令處理函數，加入了對 `HCI_LE_Set_Data_Length` 的處理：

```c
case HCI_LE_SET_DATA_LENGTH:
    // 回傳錯誤碼：不支援此功能
    hci_send_command_complete(HCI_ERROR_UNSUPPORTED_FEATURE);
    break;
```

重新編譯、燒錄、測試。

這次，iPhone 成功連接了！

我興奮地跑去找主管：「成功了！iPhone 可以連線了！」

「很好，」他說，「這就是 HCI 的重要性。它是硬體與軟體的分界線，也是不同廠商晶片互通的關鍵。只要正確實作 HCI，你的晶片就能和所有支援 Bluetooth 的裝置互通。」

這次經歷讓我深刻理解了 HCI 的設計哲學：**標準化介面，讓不同廠商的晶片能夠互通。即使不支援某個功能，也要正確回應錯誤碼，而不是直接忽略**。

---

## HCI 的設計哲學

### 為什麼需要 HCI？

在 Bluetooth 的早期，每個晶片廠商都有自己的 API。如果你要從 CSR 的晶片換到 Broadcom 的晶片，你需要重寫所有的程式碼。

這對開發者來說是一場災難。

**HCI（Host Controller Interface）** 的設計目標就是解決這個問題：

1. **標準化介面**：所有晶片廠商都必須支援相同的 HCI 命令
2. **硬體抽象**：Host 不需要知道 Controller 的實作細節
3. **可移植性**：Host 的程式碼可以在不同的 Controller 上運行

### HCI 的位置

HCI 位於 Host 和 Controller 之間：

```
┌─────────────────────────────────────────────────────────┐
│  Host (主機 - 通常在主處理器上運行)                       │
│  ├─ GAP, GATT, SMP, ATT, L2CAP                          │
│  └─ HCI Driver                                          │
├─────────────────────────────────────────────────────────┤
│  HCI (Host Controller Interface)                        │
│  ├─ Command Packet (Host → Controller)                  │
│  ├─ Event Packet (Controller → Host)                    │
│  └─ Data Packet (雙向)                                  │
├─────────────────────────────────────────────────────────┤
│  Controller (控制器 - 通常在藍牙晶片上運行)               │
│  ├─ Link Layer                                          │
│  └─ PHY                                                 │
└─────────────────────────────────────────────────────────┘
```

### HCI 的實作方式

HCI 可以透過不同的物理介面實作：

| 介面 | 速度 | Pin 數量 | 功耗 | 應用場景 |
|------|------|---------|------|---------|
| **UART** | 115200 - 3000000 bps | 4 (TX, RX, CTS, RTS) | 低 | 最常見，適合低速應用 |
| **USB** | 12 Mbps (Full Speed) | 2 (D+, D-) | 中 | PC、筆電 |
| **SPI** | 10 - 50 Mbps | 4 (SCLK, MOSI, MISO, CS) | 低 | 高速應用 |
| **SDIO** | 25 - 50 Mbps | 6 (CLK, CMD, DAT0-3) | 中 | 高速應用 |
| **Function Call** | N/A | 0 | 極低 | 單晶片方案（SoC） |

**單晶片方案（SoC）**：

- Host 和 Controller 在同一晶片上
- HCI 是虛擬的（Function Call）
- 優點：成本低、功耗低、體積小
- 缺點：靈活性低（無法更換 Controller）

**雙晶片方案**：

- Host 和 Controller 在不同晶片上
- HCI 透過 UART、USB、SPI 等物理介面實作
- 優點：靈活性高（可以更換 Controller）
- 缺點：成本高、功耗高、體積大

---

## HCI 封包格式

HCI 定義了三種封包類型：

### 1. Command Packet（Host → Controller）

**格式**：

```
┌─────────────────────────────────────────────────────────┐
│  HCI Command Packet                                      │
├─────────────────────────────────────────────────────────┤
│  Packet Type (1 byte)    - 0x01 (Command)               │
│  Opcode (2 bytes)        - 命令代碼                      │
│  Parameter Length (1 byte) - 參數長度                   │
│  Parameters (N bytes)    - 參數                          │
└─────────────────────────────────────────────────────────┘
```

**Opcode 結構**：

```
┌─────────────────────────────────────────────────────────┐
│  Opcode (16 bits)                                        │
├─────────────────────────────────────────────────────────┤
│  OGF (6 bits)  - Opcode Group Field (命令群組)          │
│  OCF (10 bits) - Opcode Command Field (命令編號)        │
└─────────────────────────────────────────────────────────┘
```

**常見的 OGF**：

| OGF | 群組名稱 | 說明 |
|-----|---------|------|
| 0x01 | Link Control | 連線控制（Classic Bluetooth） |
| 0x03 | Controller & Baseband | 控制器設定 |
| 0x04 | Informational Parameters | 讀取資訊 |
| 0x08 | LE Controller | BLE 控制器命令 |

**範例：HCI_Reset**

```
Packet Type: 0x01
Opcode: 0x0C03 (OGF=0x03, OCF=0x0003)
Parameter Length: 0x00
Parameters: (無)

完整封包: 01 03 0C 00
```

### 2. Event Packet（Controller → Host）

**格式**：

```
┌─────────────────────────────────────────────────────────┐
│  HCI Event Packet                                        │
├─────────────────────────────────────────────────────────┤
│  Packet Type (1 byte)    - 0x04 (Event)                 │
│  Event Code (1 byte)     - 事件代碼                      │
│  Parameter Length (1 byte) - 參數長度                   │
│  Parameters (N bytes)    - 參數                          │
└─────────────────────────────────────────────────────────┘
```

**常見的 Event Code**：

| Event Code | 事件名稱 | 說明 |
|-----------|---------|------|
| 0x0E | Command Complete | 命令執行完成 |
| 0x0F | Command Status | 命令狀態（用於非同步命令） |
| 0x05 | Disconnection Complete | 連線斷開 |
| 0x3E | LE Meta Event | BLE 事件（包含多種子事件） |

**範例：Command Complete Event**

```
Packet Type: 0x04
Event Code: 0x0E (Command Complete)
Parameter Length: 0x04
Parameters:
  - Num_HCI_Command_Packets: 0x01 (可以發送下一個命令)
  - Command_Opcode: 0x0C03 (HCI_Reset)
  - Return_Parameters: 0x00 (成功)

完整封包: 04 0E 04 01 03 0C 00
```

### 3. Data Packet（雙向）

**格式（ACL Data）**：

```
┌─────────────────────────────────────────────────────────┐
│  HCI ACL Data Packet                                     │
├─────────────────────────────────────────────────────────┤
│  Packet Type (1 byte)    - 0x02 (ACL Data)              │
│  Handle + Flags (2 bytes) - 連線 Handle + 封包邊界標記   │
│  Data Length (2 bytes)   - 資料長度                      │
│  Data (N bytes)          - L2CAP 封包                    │
└─────────────────────────────────────────────────────────┘
```

**Handle + Flags 結構**：

```
┌─────────────────────────────────────────────────────────┐
│  Handle + Flags (16 bits)                                │
├─────────────────────────────────────────────────────────┤
│  Handle (12 bits)  - 連線 Handle (0x0000 - 0x0EFF)      │
│  PB Flag (2 bits)  - Packet Boundary Flag               │
│  BC Flag (2 bits)  - Broadcast Flag                     │
└─────────────────────────────────────────────────────────┘
```

**PB Flag（Packet Boundary Flag）**：

| 值 | 說明 |
|----|------|
| 0b00 | 保留 |
| 0b01 | 繼續分段（Continuing fragment） |
| 0b10 | 第一個分段（First fragment） |
| 0b11 | 完整封包（Complete L2CAP PDU） |

---

## 常用的 HCI 命令

### 控制器初始化

**HCI_Reset**：

```c
// 重置 Controller
uint8_t hci_reset[] = {0x01, 0x03, 0x0C, 0x00};
uart_send(hci_reset, 4);

// 預期回應：Command Complete Event
// 04 0E 04 01 03 0C 00
```

**HCI_Read_Local_Version_Information**：

```c
// 讀取 Controller 版本資訊
uint8_t hci_read_version[] = {0x01, 0x01, 0x10, 0x00};
uart_send(hci_read_version, 4);

// 預期回應：Command Complete Event
// 04 0E 0C 01 01 10 00 [HCI_Version] [HCI_Revision] [LMP_Version] [Manufacturer_Name] [LMP_Subversion]
```

### BLE 掃描

**HCI_LE_Set_Scan_Parameters**：

```c
// 設定掃描參數
uint8_t hci_set_scan_params[] = {
    0x01,              // Packet Type: Command
    0x0B, 0x20,        // Opcode: 0x200B (HCI_LE_Set_Scan_Parameters)
    0x07,              // Parameter Length: 7
    0x01,              // Scan_Type: Active Scanning
    0x10, 0x00,        // Scan_Interval: 16 * 0.625ms = 10ms
    0x10, 0x00,        // Scan_Window: 16 * 0.625ms = 10ms
    0x00,              // Own_Address_Type: Public Device Address
    0x00               // Scanning_Filter_Policy: Accept all
};
uart_send(hci_set_scan_params, 11);
```

**HCI_LE_Set_Scan_Enable**：

```c
// 開始掃描
uint8_t hci_scan_enable[] = {
    0x01,              // Packet Type: Command
    0x0C, 0x20,        // Opcode: 0x200C (HCI_LE_Set_Scan_Enable)
    0x02,              // Parameter Length: 2
    0x01,              // LE_Scan_Enable: Enable
    0x00               // Filter_Duplicates: Disable
};
uart_send(hci_scan_enable, 6);

// 預期回應：Command Complete Event
// 04 0E 04 01 0C 20 00

// 之後會收到 LE Advertising Report Event
// 04 3E [Length] 02 [Num_Reports] [Event_Type] [Address_Type] [Address] [Data_Length] [Data] [RSSI]
```

### BLE 連線

**HCI_LE_Create_Connection**：

```c
// 建立連線
uint8_t hci_create_connection[] = {
    0x01,              // Packet Type: Command
    0x0D, 0x20,        // Opcode: 0x200D (HCI_LE_Create_Connection)
    0x19,              // Parameter Length: 25
    0x60, 0x00,        // LE_Scan_Interval: 96 * 0.625ms = 60ms
    0x30, 0x00,        // LE_Scan_Window: 48 * 0.625ms = 30ms
    0x00,              // Initiator_Filter_Policy: Use Peer_Address
    0x00,              // Peer_Address_Type: Public Device Address
    0x12, 0x34, 0x56, 0x78, 0x9A, 0xBC,  // Peer_Address
    0x00,              // Own_Address_Type: Public Device Address
    0x18, 0x00,        // Conn_Interval_Min: 24 * 1.25ms = 30ms
    0x28, 0x00,        // Conn_Interval_Max: 40 * 1.25ms = 50ms
    0x00, 0x00,        // Conn_Latency: 0
    0xC8, 0x00,        // Supervision_Timeout: 200 * 10ms = 2s
    0x00, 0x00,        // Minimum_CE_Length: 0
    0x00, 0x00         // Maximum_CE_Length: 0
};
uart_send(hci_create_connection, 29);

// 預期回應：Command Status Event
// 04 0F 04 00 01 0D 20

// 連線建立後會收到 LE Connection Complete Event
// 04 3E 13 01 00 [Handle] [Role] [Peer_Address_Type] [Peer_Address] [Conn_Interval] [Conn_Latency] [Supervision_Timeout] [Master_Clock_Accuracy]
```

### BLE 廣播

**HCI_LE_Set_Advertising_Parameters**：

```c
// 設定廣播參數
uint8_t hci_set_adv_params[] = {
    0x01,              // Packet Type: Command
    0x06, 0x20,        // Opcode: 0x2006 (HCI_LE_Set_Advertising_Parameters)
    0x0F,              // Parameter Length: 15
    0x00, 0x08,        // Advertising_Interval_Min: 2048 * 0.625ms = 1280ms
    0x00, 0x08,        // Advertising_Interval_Max: 2048 * 0.625ms = 1280ms
    0x00,              // Advertising_Type: ADV_IND (Connectable undirected)
    0x00,              // Own_Address_Type: Public Device Address
    0x00,              // Peer_Address_Type: Public Device Address
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00,  // Peer_Address (unused)
    0x07,              // Advertising_Channel_Map: All 3 channels
    0x00               // Advertising_Filter_Policy: Process all
};
uart_send(hci_set_adv_params, 19);
```

**HCI_LE_Set_Advertising_Data**：

```c
// 設定廣播資料
uint8_t hci_set_adv_data[] = {
    0x01,              // Packet Type: Command
    0x08, 0x20,        // Opcode: 0x2008 (HCI_LE_Set_Advertising_Data)
    0x20,              // Parameter Length: 32
    0x0C,              // Advertising_Data_Length: 12
    // Advertising Data (12 bytes)
    0x02, 0x01, 0x06,  // Flags: LE General Discoverable Mode
    0x09, 0x09, 'M', 'i', ' ', 'B', 'a', 'n', 'd',  // Complete Local Name: "Mi Band"
    // Padding (20 bytes)
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    0x00, 0x00, 0x00, 0x00
};
uart_send(hci_set_adv_data, 36);
```

**HCI_LE_Set_Advertising_Enable**：

```c
// 開始廣播
uint8_t hci_adv_enable[] = {
    0x01,              // Packet Type: Command
    0x0A, 0x20,        // Opcode: 0x200A (HCI_LE_Set_Advertising_Enable)
    0x01,              // Parameter Length: 1
    0x01               // Advertising_Enable: Enable
};
uart_send(hci_adv_enable, 5);

// 預期回應：Command Complete Event
// 04 0E 04 01 0A 20 00
```

---

## HCI 命令序列：完整的初始化流程

讓我們看一個完整的 BLE Peripheral 初始化流程：

```c
void ble_peripheral_init(void) {
    // 1. 重置 Controller
    hci_send_command(HCI_Reset);
    hci_wait_event(Command_Complete);

    // 2. 讀取版本資訊
    hci_send_command(HCI_Read_Local_Version_Information);
    hci_wait_event(Command_Complete);

    // 3. 讀取支援的功能
    hci_send_command(HCI_LE_Read_Local_Supported_Features);
    hci_wait_event(Command_Complete);

    // 4. 讀取 Buffer 大小
    hci_send_command(HCI_LE_Read_Buffer_Size);
    hci_wait_event(Command_Complete);

    // 5. 設定 Event Mask（啟用需要的事件）
    hci_send_command(HCI_Set_Event_Mask, 0xFFFFFFFFFFFFFFFF);
    hci_wait_event(Command_Complete);

    // 6. 設定 LE Event Mask
    hci_send_command(HCI_LE_Set_Event_Mask, 0x000000000000001F);
    hci_wait_event(Command_Complete);

    // 7. 設定隨機位址（如果使用 Random Address）
    hci_send_command(HCI_LE_Set_Random_Address, random_address);
    hci_wait_event(Command_Complete);

    // 8. 設定廣播參數
    hci_send_command(HCI_LE_Set_Advertising_Parameters, ...);
    hci_wait_event(Command_Complete);

    // 9. 設定廣播資料
    hci_send_command(HCI_LE_Set_Advertising_Data, ...);
    hci_wait_event(Command_Complete);

    // 10. 設定 Scan Response 資料
    hci_send_command(HCI_LE_Set_Scan_Response_Data, ...);
    hci_wait_event(Command_Complete);

    // 11. 開始廣播
    hci_send_command(HCI_LE_Set_Advertising_Enable, 0x01);
    hci_wait_event(Command_Complete);

    printf("BLE Peripheral initialized and advertising\n");
}
```

---

## HCI 的錯誤處理

### 常見的錯誤碼

| 錯誤碼 | 名稱 | 說明 |
|-------|------|------|
| 0x00 | Success | 成功 |
| 0x01 | Unknown HCI Command | 不支援的命令 |
| 0x02 | Unknown Connection Identifier | 無效的連線 Handle |
| 0x0C | Command Disallowed | 命令不允許（狀態錯誤） |
| 0x12 | Invalid HCI Command Parameters | 參數錯誤 |
| 0x1A | Unsupported Feature or Parameter Value | 不支援的功能 |
| 0x3E | Connection Failed to be Established | 連線建立失敗 |

### 錯誤處理範例

```c
void hci_command_complete_handler(uint8_t *event) {
    uint8_t num_packets = event[3];
    uint16_t opcode = (event[5] << 8) | event[4];
    uint8_t status = event[6];

    if (status != 0x00) {
        printf("HCI Command 0x%04X failed with status 0x%02X\n", opcode, status);

        switch (status) {
            case 0x01:
                printf("Error: Unknown HCI Command\n");
                break;
            case 0x0C:
                printf("Error: Command Disallowed (check state)\n");
                break;
            case 0x12:
                printf("Error: Invalid HCI Command Parameters\n");
                break;
            case 0x1A:
                printf("Error: Unsupported Feature\n");
                // 回退到舊版本的功能
                use_legacy_feature();
                break;
            default:
                printf("Error: Unknown error code\n");
                break;
        }
    } else {
        printf("HCI Command 0x%04X completed successfully\n", opcode);
    }
}
```

---

## HCI Transport 的選擇

### UART vs SPI vs USB

在實際專案中，如何選擇 HCI Transport？

**UART**：

- **優點**：
  - 簡單、成熟、穩定
  - Pin 數量少（4 個）
  - 功耗低
  - 成本低
- **缺點**：
  - 速度慢（通常 115200 - 1000000 bps）
  - 不適合高速應用
- **應用場景**：
  - 智慧手環、智慧手錶
  - IoT 裝置
  - 低速應用

**SPI**：

- **優點**：
  - 速度快（10 - 50 Mbps）
  - 功耗低
- **缺點**：
  - Pin 數量多（4 個）
  - 實作複雜（需要處理 CS 訊號）
- **應用場景**：
  - 高速應用
  - 需要低延遲的應用

**USB**：

- **優點**：
  - 速度快（12 Mbps Full Speed, 480 Mbps High Speed）
  - 標準化（PC、筆電都支援）
- **缺點**：
  - 功耗高
  - 成本高
  - 不適合電池供電裝置
- **應用場景**：
  - PC、筆電的藍牙適配器
  - 開發板

**Function Call（單晶片方案）**：

- **優點**：
  - 速度極快（無需序列化）
  - 功耗極低（無需 UART/SPI 硬體）
  - 成本極低（無需額外 Pin）
- **缺點**：
  - 靈活性低（無法更換 Controller）
- **應用場景**：
  - 大部分的 BLE 應用（智慧手環、Beacon、IoT 裝置）

### 實際案例：智慧手環的選擇

在我們的智慧手環專案中，我們選擇了**單晶片方案（SoC）**：

- **晶片**：Nordic nRF51822（Cortex-M0 + BLE Controller）
- **HCI**：虛擬的（Function Call）
- **優勢**：
  - 成本低（單一晶片）
  - 功耗低（無需 UART）
  - 體積小（無需額外晶片）
- **劣勢**：
  - 無法更換 Controller（但我們不需要）

---

## 總結

HCI 是 Bluetooth 協定堆疊中的關鍵層級，它定義了 Host 和 Controller 之間的標準介面。

**核心概念**：

- **三種封包類型**：Command、Event、Data
- **標準化介面**：所有晶片廠商都必須支援相同的 HCI 命令
- **硬體抽象**：Host 不需要知道 Controller 的實作細節

**實作方式**：

- **UART**：最常見，適合低速應用
- **SPI**：速度快，適合高速應用
- **USB**：速度快，適合 PC、筆電
- **Function Call**：單晶片方案，成本低、功耗低

**開發建議**：

- 理解 HCI 命令序列（初始化、掃描、連線、廣播）
- 正確處理錯誤碼
- 根據應用場景選擇合適的 HCI Transport

在下一篇文章中，我們將深入探討 **L2CAP（Logical Link Control & Adaptation Protocol）**，了解資料的分段和重組。

---

## 參考資料

1. Bluetooth SIG. (2013). *Bluetooth Core Specification Version 4.1 - Vol 2, Part E: Host Controller Interface*. <https://www.bluetooth.com/specifications/specs/core-specification-4-1/>
2. Bluetooth SIG. (2014). *Bluetooth Core Specification Version 4.2 - Vol 2, Part E: Host Controller Interface*. <https://www.bluetooth.com/specifications/specs/core-specification-4-2/>
3. Nordic Semiconductor. (2014). *nRF51 Series Reference Manual - SoftDevice S110*. <https://www.nordicsemi.com/>
4. Texas Instruments. (2014). *CC2540/41 Bluetooth Low Energy Software Developer's Guide*. <https://www.ti.com/>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
