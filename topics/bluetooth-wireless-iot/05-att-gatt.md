# ATT/GATT - BLE 的核心資料模型

**作者**: Danny Jiang  
**日期**: 2025-12-11  

---

## 前言：第一次實作 Heart Rate Profile

2014 年 7 月，我接到了第一個完整的任務：為智慧手環實作心率監測功能。

「這應該不難吧？」我想，「就是讀取心率感測器的數值，然後傳送給手機。」

主管給了我一份文件：「這是 Bluetooth SIG 定義的 Heart Rate Profile 規格。你需要按照這個規格來實作。」

我打開文件，看到的第一頁就是一個複雜的樹狀結構圖：

```
Heart Rate Service (UUID: 0x180D)
├─ Heart Rate Measurement (UUID: 0x2A37)
│   ├─ Properties: Notify
│   └─ Value: [Flags] [Heart Rate] [Energy Expended] [RR-Interval]
└─ Body Sensor Location (UUID: 0x2A38)
    ├─ Properties: Read
    └─ Value: 0x01 (Chest)
```

「這些 UUID、Properties、Notify 是什麼意思？」我問。

「這就是 GATT 的資料模型，」主管說，「你可以把它想像成一個樹狀的資料庫。Service 是資料庫，Characteristic 是資料表，Descriptor 是欄位。手機透過讀寫這些 Characteristic 來取得心率數值。」

「那 Notify 是什麼？」

「Notify 是一種推送機制，」他解釋，「手環主動把心率數值推送給手機，而不是等手機來讀取。這樣更即時，也更省電。」

「聽起來很簡單，」我說，「就是定義幾個資料結構，然後實作讀寫函數。」

「試試看吧，」主管笑了笑，「有問題再來找我。」

我花了整整一週時間，才搞懂 ATT/GATT 的運作方式。

第一天，我卡在「什麼是 Handle」。我以為 Handle 是指標，結果發現它只是一個 16-bit 的編號，用來識別每個 Attribute。

第二天，我卡在「如何啟用 Notification」。我以為只要設定 Properties 為 Notify 就可以了，結果發現還需要一個叫做 CCCD（Client Characteristic Configuration Descriptor）的東西，手機必須先寫入 0x0001 到 CCCD，才能啟用 Notification。

第三天，我卡在「封包格式」。我不知道心率數值要怎麼編碼。是直接傳送一個整數嗎？還是要加上 Flags？結果發現規格文件中有詳細的定義：第一個 byte 是 Flags（表示數值格式），第二個 byte 是心率數值。

第四天，我終於把程式碼寫完了。我燒錄到手環上，打開手機 App，連線、配對、啟用 Notification...

螢幕上出現了心率數值：**75 bpm**。

我按住手環，心率數值開始跳動：76、78、82、85...

「成功了！」我興奮地喊出來。

這次經歷讓我深刻理解了 ATT/GATT 的重要性：**它們定義了 BLE 的核心資料模型，是所有 BLE 應用的基礎。理解 ATT/GATT，就理解了 BLE 的靈魂**。

---

## ATT：Attribute Protocol

### 什麼是 Attribute？

**Attribute（屬性）** 是 BLE 資料模型的基本單元，包含四個要素：

1. **Handle**：16-bit 的唯一識別碼（0x0001 - 0xFFFF）
2. **Type**：UUID（16-bit 或 128-bit），定義 Attribute 的類型
3. **Value**：實際的資料（0 - 512 bytes）
4. **Permissions**：讀寫權限（Read、Write、Read/Write）

**Attribute 結構**：

```
┌─────────────────────────────────────────────────────────┐
│  Attribute                                               │
├─────────────────────────────────────────────────────────┤
│  Handle: 0x0010                                          │
│  Type: 0x2A37 (Heart Rate Measurement)                  │
│  Value: [0x00, 0x4B] (75 bpm)                           │
│  Permissions: Read, Notify                              │
└─────────────────────────────────────────────────────────┘
```

### ATT 的操作

ATT 定義了多種操作，用於讀寫 Attribute：

| 操作 | 方向 | 需要 ACK | 說明 |
|------|------|---------|------|
| **Read Request** | Client → Server | 是 | 讀取 Attribute 的值 |
| **Read Response** | Server → Client | - | 回傳 Attribute 的值 |
| **Write Request** | Client → Server | 是 | 寫入 Attribute 的值 |
| **Write Response** | Server → Client | - | 確認寫入成功 |
| **Write Command** | Client → Server | 否 | 寫入 Attribute 的值（不需要 ACK） |
| **Notification** | Server → Client | 否 | 主動推送資料（不需要 ACK） |
| **Indication** | Server → Client | 是 | 主動推送資料（需要 ACK） |
| **Read By Type Request** | Client → Server | 是 | 根據 Type 讀取 Attribute |
| **Read By Group Type Request** | Client → Server | 是 | 根據 Group Type 讀取 Attribute |

### ATT 封包格式

**Read Request**：

```
┌─────────────────────────────────────────────────────────┐
│  ATT Read Request                                        │
├─────────────────────────────────────────────────────────┤
│  Opcode: 0x0A (Read Request)                            │
│  Attribute Handle: 0x0010 (2 bytes)                     │
└─────────────────────────────────────────────────────────┘

完整封包: 0A 10 00
```

**Read Response**：

```
┌─────────────────────────────────────────────────────────┐
│  ATT Read Response                                       │
├─────────────────────────────────────────────────────────┤
│  Opcode: 0x0B (Read Response)                           │
│  Attribute Value: [0x00, 0x4B] (75 bpm)                 │
└─────────────────────────────────────────────────────────┘

完整封包: 0B 00 4B
```

**Write Request**：

```
┌─────────────────────────────────────────────────────────┐
│  ATT Write Request                                       │
├─────────────────────────────────────────────────────────┤
│  Opcode: 0x12 (Write Request)                           │
│  Attribute Handle: 0x0012 (2 bytes)                     │
│  Attribute Value: [0x01, 0x00] (Enable Notification)    │
└─────────────────────────────────────────────────────────┘

完整封包: 12 12 00 01 00
```

**Notification**：

```
┌─────────────────────────────────────────────────────────┐
│  ATT Notification                                        │
├─────────────────────────────────────────────────────────┤
│  Opcode: 0x1B (Notification)                            │
│  Attribute Handle: 0x0011 (2 bytes)                     │
│  Attribute Value: [0x00, 0x4B] (75 bpm)                 │
└─────────────────────────────────────────────────────────┘

完整封包: 1B 11 00 00 4B
```

---

## GATT：Generic Attribute Profile

### GATT 的層次結構

GATT 定義了 Attribute 的組織方式，形成層次結構：

```
┌─────────────────────────────────────────────────────────┐
│  Profile (例如：Heart Rate Profile)                      │
│  └─ Service (例如：Heart Rate Service, UUID=0x180D)     │
│      ├─ Characteristic (Heart Rate Measurement)         │
│      │   ├─ Characteristic Declaration                  │
│      │   ├─ Characteristic Value                        │
│      │   └─ Descriptor (CCCD - 啟用 Notification)       │
│      └─ Characteristic (Body Sensor Location)           │
│          ├─ Characteristic Declaration                  │
│          └─ Characteristic Value                        │
└─────────────────────────────────────────────────────────┘
```

### Service

**Service** 是一組相關的 Characteristics。

**Service 的定義**：

```
┌─────────────────────────────────────────────────────────┐
│  Service Declaration                                     │
├─────────────────────────────────────────────────────────┤
│  Handle: 0x0001                                          │
│  Type: 0x2800 (Primary Service)                         │
│  Value: 0x180D (Heart Rate Service UUID)                │
│  Permissions: Read                                      │
└─────────────────────────────────────────────────────────┘
```

**常見的 Service UUID**：

| UUID | Service 名稱 | 說明 |
|------|------------|------|
| 0x1800 | Generic Access | 裝置名稱、外觀 |
| 0x1801 | Generic Attribute | Service Changed |
| 0x180D | Heart Rate | 心率測量 |
| 0x180F | Battery Service | 電池電量 |
| 0x1810 | Blood Pressure | 血壓測量 |
| 0x181A | Environmental Sensing | 環境感測（溫度、濕度） |

### Characteristic

**Characteristic** 是一個具體的資料項目。

**Characteristic 的結構**：

```
┌─────────────────────────────────────────────────────────┐
│  Characteristic Declaration                              │
├─────────────────────────────────────────────────────────┤
│  Handle: 0x0010                                          │
│  Type: 0x2803 (Characteristic Declaration)              │
│  Value: [Properties] [Value Handle] [UUID]              │
│  Permissions: Read                                      │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  Characteristic Value                                    │
├─────────────────────────────────────────────────────────┤
│  Handle: 0x0011                                          │
│  Type: 0x2A37 (Heart Rate Measurement)                  │
│  Value: [0x00, 0x4B] (75 bpm)                           │
│  Permissions: Read, Notify                              │
└─────────────────────────────────────────────────────────┘
```

**Characteristic Properties**：

| Bit | Property | 說明 |
|-----|---------|------|
| 0 | Broadcast | 支援廣播 |
| 1 | Read | 支援讀取 |
| 2 | Write Without Response | 支援寫入（不需要 ACK） |
| 3 | Write | 支援寫入（需要 ACK） |
| 4 | Notify | 支援通知（不需要 ACK） |
| 5 | Indicate | 支援指示（需要 ACK） |
| 6 | Authenticated Signed Writes | 支援簽名寫入 |
| 7 | Extended Properties | 擴展屬性 |

### Descriptor

**Descriptor** 提供 Characteristic 的額外資訊。

**常見的 Descriptor UUID**：

| UUID | Descriptor 名稱 | 說明 |
|------|---------------|------|
| 0x2900 | Characteristic Extended Properties | 擴展屬性 |
| 0x2901 | Characteristic User Description | 使用者描述 |
| 0x2902 | Client Characteristic Configuration (CCCD) | 啟用 Notification/Indication |
| 0x2903 | Server Characteristic Configuration | 伺服器配置 |
| 0x2904 | Characteristic Presentation Format | 資料格式 |

**CCCD（Client Characteristic Configuration Descriptor）**：

```
┌─────────────────────────────────────────────────────────┐
│  CCCD                                                    │
├─────────────────────────────────────────────────────────┤
│  Handle: 0x0012                                          │
│  Type: 0x2902 (CCCD)                                    │
│  Value: [0x01, 0x00] (Enable Notification)              │
│  Permissions: Read, Write                               │
└─────────────────────────────────────────────────────────┘

Value 的意義：
- 0x0000: Notification 和 Indication 都停用
- 0x0001: 啟用 Notification
- 0x0002: 啟用 Indication
- 0x0003: 同時啟用 Notification 和 Indication
```

---

## Service Discovery：探索 GATT 結構

### 為什麼需要 Service Discovery？

當 Client（手機）連線到 Server（智慧手環）時，Client 不知道 Server 有哪些 Services 和 Characteristics。

**Service Discovery** 是一個自動探索的過程：

1. **探索所有 Services**
2. **探索每個 Service 的 Characteristics**
3. **探索每個 Characteristic 的 Descriptors**

### Service Discovery 流程

```
Client (手機)                    Server (智慧手環)
     |                                |
     |  Read By Group Type Request    |
     |  (Type = Primary Service)      |
     |------------------------------->|
     |                                |
     |  Read By Group Type Response   |
     |  (Service 1: 0x1800, Handle 0x0001-0x0005) |
     |  (Service 2: 0x180D, Handle 0x0010-0x0015) |
     |<-------------------------------|
     |                                |
     |  Read By Type Request          |
     |  (Type = Characteristic, Range 0x0010-0x0015) |
     |------------------------------->|
     |                                |
     |  Read By Type Response         |
     |  (Char 1: 0x2A37, Handle 0x0011) |
     |  (Char 2: 0x2A38, Handle 0x0013) |
     |<-------------------------------|
     |                                |
     |  Find Information Request      |
     |  (Range 0x0012-0x0012)         |
     |------------------------------->|
     |                                |
     |  Find Information Response     |
     |  (Descriptor: 0x2902, Handle 0x0012) |
     |<-------------------------------|
     |                                |
```

### 程式碼範例

```c
// 1. 探索所有 Primary Services
void discover_primary_services(void) {
    uint8_t packet[] = {
        0x10,              // Opcode: Read By Group Type Request
        0x01, 0x00,        // Starting Handle: 0x0001
        0xFF, 0xFF,        // Ending Handle: 0xFFFF
        0x00, 0x28         // Attribute Type: Primary Service (0x2800)
    };
    l2cap_send(ATT_CHANNEL_ID, packet, 7);
}

// 2. 探索 Service 的 Characteristics
void discover_characteristics(uint16_t start_handle, uint16_t end_handle) {
    uint8_t packet[] = {
        0x08,              // Opcode: Read By Type Request
        start_handle & 0xFF,    // Starting Handle (LSB)
        start_handle >> 8,      // Starting Handle (MSB)
        end_handle & 0xFF,      // Ending Handle (LSB)
        end_handle >> 8,        // Ending Handle (MSB)
        0x03, 0x28         // Attribute Type: Characteristic (0x2803)
    };
    l2cap_send(ATT_CHANNEL_ID, packet, 7);
}

// 3. 探索 Characteristic 的 Descriptors
void discover_descriptors(uint16_t start_handle, uint16_t end_handle) {
    uint8_t packet[] = {
        0x04,              // Opcode: Find Information Request
        start_handle & 0xFF,    // Starting Handle (LSB)
        start_handle >> 8,      // Starting Handle (MSB)
        end_handle & 0xFF,      // Ending Handle (LSB)
        end_handle >> 8         // Ending Handle (MSB)
    };
    l2cap_send(ATT_CHANNEL_ID, packet, 5);
}
```

---

## 實際案例：Heart Rate Profile 的實作

### Heart Rate Service 的結構

```
Heart Rate Service (UUID: 0x180D)
├─ Handle 0x0010: Service Declaration
│   └─ Value: 0x180D (Heart Rate Service UUID)
├─ Handle 0x0011: Characteristic Declaration (Heart Rate Measurement)
│   └─ Value: [Properties=0x10 (Notify)] [Handle=0x0012] [UUID=0x2A37]
├─ Handle 0x0012: Characteristic Value (Heart Rate Measurement)
│   └─ Value: [Flags] [Heart Rate] [Energy Expended] [RR-Interval]
├─ Handle 0x0013: CCCD (Client Characteristic Configuration)
│   └─ Value: [0x00, 0x00] (Notification Disabled)
├─ Handle 0x0014: Characteristic Declaration (Body Sensor Location)
│   └─ Value: [Properties=0x02 (Read)] [Handle=0x0015] [UUID=0x2A38]
└─ Handle 0x0015: Characteristic Value (Body Sensor Location)
    └─ Value: 0x01 (Chest)
```

### Server 端實作

```c
// Heart Rate Service 的 Attribute Table
typedef struct {
    uint16_t handle;
    uint16_t type;
    uint8_t permissions;
    uint16_t value_len;
    uint8_t *value;
} attribute_t;

attribute_t heart_rate_service[] = {
    // Service Declaration
    {
        .handle = 0x0010,
        .type = 0x2800,  // Primary Service
        .permissions = ATT_PERMISSIONS_READ,
        .value_len = 2,
        .value = (uint8_t[]){0x0D, 0x18}  // Heart Rate Service UUID
    },

    // Characteristic Declaration (Heart Rate Measurement)
    {
        .handle = 0x0011,
        .type = 0x2803,  // Characteristic Declaration
        .permissions = ATT_PERMISSIONS_READ,
        .value_len = 5,
        .value = (uint8_t[]){0x10, 0x12, 0x00, 0x37, 0x2A}  // [Properties] [Handle] [UUID]
    },

    // Characteristic Value (Heart Rate Measurement)
    {
        .handle = 0x0012,
        .type = 0x2A37,  // Heart Rate Measurement
        .permissions = ATT_PERMISSIONS_READ | ATT_PERMISSIONS_NOTIFY,
        .value_len = 2,
        .value = (uint8_t[]){0x00, 0x4B}  // 75 bpm
    },

    // CCCD
    {
        .handle = 0x0013,
        .type = 0x2902,  // CCCD
        .permissions = ATT_PERMISSIONS_READ | ATT_PERMISSIONS_WRITE,
        .value_len = 2,
        .value = (uint8_t[]){0x00, 0x00}  // Notification Disabled
    },

    // Characteristic Declaration (Body Sensor Location)
    {
        .handle = 0x0014,
        .type = 0x2803,  // Characteristic Declaration
        .permissions = ATT_PERMISSIONS_READ,
        .value_len = 5,
        .value = (uint8_t[]){0x02, 0x15, 0x00, 0x38, 0x2A}  // [Properties] [Handle] [UUID]
    },

    // Characteristic Value (Body Sensor Location)
    {
        .handle = 0x0015,
        .type = 0x2A38,  // Body Sensor Location
        .permissions = ATT_PERMISSIONS_READ,
        .value_len = 1,
        .value = (uint8_t[]){0x01}  // Chest
    }
};

// ATT Read Request Handler
void att_read_request_handler(uint16_t handle) {
    for (int i = 0; i < sizeof(heart_rate_service) / sizeof(attribute_t); i++) {
        if (heart_rate_service[i].handle == handle) {
            // 檢查權限
            if (!(heart_rate_service[i].permissions & ATT_PERMISSIONS_READ)) {
                att_send_error_response(handle, ATT_ERROR_READ_NOT_PERMITTED);
                return;
            }

            // 發送 Read Response
            att_send_read_response(heart_rate_service[i].value, heart_rate_service[i].value_len);
            return;
        }
    }

    // Handle 不存在
    att_send_error_response(handle, ATT_ERROR_INVALID_HANDLE);
}

// ATT Write Request Handler
void att_write_request_handler(uint16_t handle, uint8_t *value, uint16_t value_len) {
    for (int i = 0; i < sizeof(heart_rate_service) / sizeof(attribute_t); i++) {
        if (heart_rate_service[i].handle == handle) {
            // 檢查權限
            if (!(heart_rate_service[i].permissions & ATT_PERMISSIONS_WRITE)) {
                att_send_error_response(handle, ATT_ERROR_WRITE_NOT_PERMITTED);
                return;
            }

            // 寫入值
            memcpy(heart_rate_service[i].value, value, value_len);
            heart_rate_service[i].value_len = value_len;

            // 如果是 CCCD，檢查是否啟用 Notification
            if (heart_rate_service[i].type == 0x2902) {
                uint16_t cccd_value = value[0] | (value[1] << 8);
                if (cccd_value & 0x0001) {
                    printf("Notification enabled\n");
                    start_heart_rate_notification();
                } else {
                    printf("Notification disabled\n");
                    stop_heart_rate_notification();
                }
            }

            // 發送 Write Response
            att_send_write_response();
            return;
        }
    }

    // Handle 不存在
    att_send_error_response(handle, ATT_ERROR_INVALID_HANDLE);
}

// 發送 Heart Rate Notification
void send_heart_rate_notification(uint8_t heart_rate) {
    uint8_t packet[] = {
        0x1B,              // Opcode: Notification
        0x12, 0x00,        // Attribute Handle: 0x0012
        0x00,              // Flags: Heart Rate Value Format (uint8)
        heart_rate         // Heart Rate Value
    };
    l2cap_send(ATT_CHANNEL_ID, packet, 5);
}
```

### Client 端實作

```c
// 1. Service Discovery
discover_primary_services();

// 2. 收到 Service Discovery Response
void handle_read_by_group_type_response(uint8_t *data, uint16_t len) {
    // 解析 Response
    // Service 1: UUID=0x1800, Handle Range=0x0001-0x0005
    // Service 2: UUID=0x180D, Handle Range=0x0010-0x0015

    // 探索 Heart Rate Service 的 Characteristics
    discover_characteristics(0x0010, 0x0015);
}

// 3. 收到 Characteristic Discovery Response
void handle_read_by_type_response(uint8_t *data, uint16_t len) {
    // 解析 Response
    // Char 1: UUID=0x2A37, Handle=0x0012, Properties=0x10 (Notify)
    // Char 2: UUID=0x2A38, Handle=0x0015, Properties=0x02 (Read)

    // 探索 Heart Rate Measurement 的 Descriptors
    discover_descriptors(0x0012, 0x0013);
}

// 4. 收到 Descriptor Discovery Response
void handle_find_information_response(uint8_t *data, uint16_t len) {
    // 解析 Response
    // Descriptor: UUID=0x2902 (CCCD), Handle=0x0013

    // 啟用 Notification
    enable_notification(0x0013);
}

// 5. 啟用 Notification
void enable_notification(uint16_t cccd_handle) {
    uint8_t packet[] = {
        0x12,              // Opcode: Write Request
        cccd_handle & 0xFF,     // CCCD Handle (LSB)
        cccd_handle >> 8,       // CCCD Handle (MSB)
        0x01, 0x00         // Value: Enable Notification
    };
    l2cap_send(ATT_CHANNEL_ID, packet, 5);
}

// 6. 接收 Notification
void handle_notification(uint8_t *data, uint16_t len) {
    uint16_t handle = data[1] | (data[2] << 8);
    uint8_t flags = data[3];
    uint8_t heart_rate = data[4];

    printf("Heart Rate: %d bpm\n", heart_rate);
}
```

---

## 總結

ATT/GATT 是 BLE 的核心資料模型，定義了資料的組織方式和存取方法。

**核心概念**：

- **Attribute**：資料的基本單元（Handle、Type、Value、Permissions）
- **GATT 層次結構**：Profile → Service → Characteristic → Descriptor
- **ATT 操作**：Read、Write、Notify、Indicate
- **Service Discovery**：自動探索 GATT 結構

**實際應用**：

- **標準 Profile**：Heart Rate、Battery、Blood Pressure
- **自訂 Profile**：根據應用需求設計
- **Notification**：即時推送資料（如：心率、步數）

在下一篇文章中，我們將深入探討 **SMP（Security Manager Protocol）**，了解 BLE 的安全機制。

---

## 參考資料

1. Bluetooth SIG. (2013). *Bluetooth Core Specification Version 4.1 - Vol 3, Part F: ATT*. <https://www.bluetooth.com/specifications/specs/core-specification-4-1/>
2. Bluetooth SIG. (2013). *Bluetooth Core Specification Version 4.1 - Vol 3, Part G: GATT*. <https://www.bluetooth.com/specifications/specs/core-specification-4-1/>
3. Bluetooth SIG. (2012). *Heart Rate Profile Specification*. <https://www.bluetooth.com/specifications/specs/heart-rate-profile-1-0/>
4. Nordic Semiconductor. (2014). *nRF51 Series - GATT Services Tutorial*. <https://www.nordicsemi.com/>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
