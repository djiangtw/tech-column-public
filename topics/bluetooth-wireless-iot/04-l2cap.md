# L2CAP - 邏輯鏈路控制與適配協定

**作者**: Danny Jiang  
**日期**: 2025-12-11  

---

## 前言：15 分鐘的煎熬

2014 年 10 月，我們的智慧手環準備發布第一次韌體更新。

「OTA 功能測試完了嗎？」主管問我。

「測試完了，」我說，「功能正常，可以成功更新韌體。」

「更新需要多久？」

「大概...」我看了一下測試記錄，「15 分鐘。」

「15 分鐘？！」主管的聲音提高了八度，「你確定嗎？」

「確定，」我說，「128 KB 的韌體檔案，透過 BLE 傳輸需要 15 分鐘。」

「這太慢了！」主管說，「使用者會以為手環壞了，然後拔掉電源，結果韌體更新失敗，手環變磚。」

我知道他說的對。15 分鐘的等待時間，對使用者來說是一種煎熬。

「有沒有辦法加快？」他問。

「我來分析一下，」我說。

我花了一整天時間，分析 BLE 的傳輸效率。我發現問題出在 **MTU（Maximum Transmission Unit）** 太小。

BLE 4.0 的 Link Layer MTU 是 27 bytes。聽起來不小，對吧？但實際上：

- L2CAP Header：4 bytes
- ATT Header：3 bytes
- **實際可用的 Payload**：只有 20 bytes

每次傳輸 20 bytes，需要等待下一個 Connection Event（我們設定的是 30 ms）。這意味著傳輸速度只有：

```
20 bytes / 30 ms = 666 bytes/s = 5.3 Kbps
```

128 KB 的韌體檔案，需要傳輸：

```
128 KB / 666 bytes/s = 196 秒 ≈ 3.3 分鐘
```

「等等，」我想，「理論上只需要 3.3 分鐘，為什麼實際上需要 15 分鐘？」

我檢查了程式碼，發現問題：我們的 OTA 實作中，每傳輸一個封包，都會等待 ACK（確認），然後才傳輸下一個封包。這導致了大量的等待時間。

「有兩個優化方向，」我對主管說，「第一，縮短 Connection Interval，從 30 ms 降到 7.5 ms，速度可以提升 4 倍。第二，使用 Write Command 而不是 Write Request，不需要等待 ACK，速度可以再提升 2 倍。」

「那還等什麼？」主管說，「趕快改！」

我花了兩天時間，重寫了 OTA 的程式碼。重新測試：

- **原始版本**：15 分鐘
- **優化後**：4 分鐘

「還是有點慢，」主管說，「但至少可以接受了。」

「如果我們升級到 BLE 4.2，」我說，「可以使用 Data Length Extension，MTU 可以提升到 251 bytes，速度可以再提升 10 倍，OTA 時間可以降到 30 秒。」

「那就等下一代晶片吧，」主管說。

這次經歷讓我深刻理解了 L2CAP 的重要性：**它負責資料的分段和重組，MTU 的大小直接影響傳輸效率。理解 L2CAP，就理解了 BLE 的性能瓶頸**。

---

## L2CAP 的設計目標

### 為什麼需要 L2CAP？

L2CAP（Logical Link Control and Adaptation Protocol）位於 Link Layer 和上層協定（ATT、SMP）之間，主要負責：

1. **Segmentation and Reassembly（分段和重組）**：
   - Link Layer 的 MTU 有限（27 bytes in BLE 4.0）
   - 上層協定可能需要傳輸更大的資料（如：512 bytes 的 ATT Read Response）
   - L2CAP 負責將大資料分段，接收端負責重組

2. **Multiplexing（多工）**：
   - 多個上層協定（ATT、SMP）共用同一個 Link Layer 連線
   - L2CAP 使用 Channel ID 區分不同的協定

3. **Flow Control（流量控制）**：
   - 避免發送端發送過快，接收端來不及處理

4. **Quality of Service（服務品質）**：
   - 為不同的應用提供不同的 QoS（Classic Bluetooth）

### L2CAP 在協定堆疊中的位置

```
┌─────────────────────────────────────────────────────────┐
│  Application Layer                                       │
│  └─ GATT, ATT, SMP                                      │
├─────────────────────────────────────────────────────────┤
│  L2CAP (Logical Link Control & Adaptation Protocol)     │
│  ├─ Segmentation and Reassembly                         │
│  ├─ Multiplexing (Channel ID)                           │
│  └─ Flow Control                                        │
├─────────────────────────────────────────────────────────┤
│  HCI (Host Controller Interface)                        │
├─────────────────────────────────────────────────────────┤
│  Link Layer                                             │
│  └─ MTU: 27 bytes (BLE 4.0), 251 bytes (BLE 4.2+)      │
└─────────────────────────────────────────────────────────┘
```

---

## L2CAP 封包格式

### Basic L2CAP Packet

```
┌─────────────────────────────────────────────────────────┐
│  L2CAP Packet (Basic Frame)                              │
├─────────────────────────────────────────────────────────┤
│  Length (2 bytes)    - Payload 長度（不包含 Header）     │
│  Channel ID (2 bytes) - 通道 ID                          │
│  Payload (N bytes)   - 上層協定的資料                     │
└─────────────────────────────────────────────────────────┘
```

**範例**：

```
Length: 0x0004 (4 bytes)
Channel ID: 0x0004 (ATT)
Payload: 01 02 03 04 (ATT Read Request)

完整封包: 04 00 04 00 01 02 03 04
```

### Channel ID

L2CAP 使用 Channel ID 區分不同的協定：

| Channel ID | 協定 | 說明 |
|-----------|------|------|
| 0x0001 | Signaling Channel | L2CAP 控制訊息（Classic Bluetooth） |
| 0x0002 | Connectionless Channel | 無連線模式（Classic Bluetooth） |
| 0x0003 | AMP Manager | AMP 管理（Classic Bluetooth） |
| 0x0004 | ATT | Attribute Protocol（BLE） |
| 0x0005 | LE Signaling Channel | L2CAP 控制訊息（BLE） |
| 0x0006 | SMP | Security Manager Protocol（BLE） |
| 0x0007 | SM over BR/EDR | Security Manager（Classic Bluetooth） |
| 0x0040-0x007F | Dynamically Allocated | 動態分配的通道 |

---

## L2CAP 的核心功能

### 1. Segmentation and Reassembly（分段和重組）

**問題**：Link Layer 的 MTU 是 27 bytes，但上層協定可能需要傳輸 512 bytes 的資料。

**解決方案**：L2CAP 將大資料分段，接收端重組。

**範例**：傳輸 100 bytes 的 ATT Read Response

```
發送端 (Server):
1. ATT 層產生 100 bytes 的 Read Response
2. L2CAP 將 100 bytes 分成 5 個分段：
   - Segment 1: 20 bytes (L2CAP Header + 16 bytes Payload)
   - Segment 2: 20 bytes (16 bytes Payload)
   - Segment 3: 20 bytes (16 bytes Payload)
   - Segment 4: 20 bytes (16 bytes Payload)
   - Segment 5: 20 bytes (16 bytes Payload)
3. Link Layer 逐一發送這 5 個分段

接收端 (Client):
1. Link Layer 接收 5 個分段
2. L2CAP 重組成 100 bytes 的完整資料
3. 傳遞給 ATT 層
```

**注意**：

- BLE 的 L2CAP 不支援分段（Segmentation）
- 所有的分段都由 Link Layer 處理（使用 LL Data PDU 的 LLID 欄位）
- L2CAP 只負責封裝和解封裝

### 2. MTU Negotiation（MTU 協商）

**MTU（Maximum Transmission Unit）**：L2CAP 層可以傳輸的最大資料單元。

**BLE 的 MTU**：

- **預設 MTU**：23 bytes（L2CAP Payload）
- **最大 MTU**：512 bytes（理論上）
- **實際 MTU**：取決於 Link Layer 的 MTU 和雙方的能力

**MTU Exchange 流程**：

```
Client (手機)                    Server (智慧手環)
     |                                |
     |  ATT_Exchange_MTU_Request      |
     |  (Client RX MTU = 512)         |
     |------------------------------->|
     |                                |
     |  ATT_Exchange_MTU_Response     |
     |  (Server RX MTU = 158)         |
     |<-------------------------------|
     |                                |
     |  (協商結果: MTU = min(512, 158) = 158) |
     |                                |
```

**程式碼範例**：

```c
// Client 發送 MTU Exchange Request
void att_exchange_mtu_request(uint16_t client_rx_mtu) {
    uint8_t packet[] = {
        0x02,                    // ATT Opcode: Exchange MTU Request
        client_rx_mtu & 0xFF,    // Client RX MTU (LSB)
        client_rx_mtu >> 8       // Client RX MTU (MSB)
    };
    l2cap_send(ATT_CHANNEL_ID, packet, 3);
}

// Server 處理 MTU Exchange Request
void att_exchange_mtu_response(uint16_t server_rx_mtu) {
    uint8_t packet[] = {
        0x03,                    // ATT Opcode: Exchange MTU Response
        server_rx_mtu & 0xFF,    // Server RX MTU (LSB)
        server_rx_mtu >> 8       // Server RX MTU (MSB)
    };
    l2cap_send(ATT_CHANNEL_ID, packet, 3);
    
    // 協商結果
    uint16_t negotiated_mtu = min(client_rx_mtu, server_rx_mtu);
    printf("Negotiated MTU: %d bytes\n", negotiated_mtu);
}
```

### 3. Flow Control（流量控制）

**問題**：發送端發送過快，接收端來不及處理。

**解決方案**：

- **Classic Bluetooth**：L2CAP 支援 Flow Control（使用 Credits）
- **BLE**：Link Layer 支援 Flow Control（使用 LL Flow Control PDU）

**BLE 的 Flow Control**：

```
發送端                          接收端
     |                                |
     |  LL Data PDU (Packet 1)        |
     |------------------------------->|
     |                                |
     |  LL Data PDU (Packet 2)        |
     |------------------------------->|
     |                                |
     |  (接收端 Buffer 滿了)           |
     |                                |
     |  LL Flow Control PDU (Stop)    |
     |<-------------------------------|
     |                                |
     |  (發送端停止發送)               |
     |                                |
     |  (接收端處理完 Buffer)          |
     |                                |
     |  LL Flow Control PDU (Go)      |
     |<-------------------------------|
     |                                |
     |  LL Data PDU (Packet 3)        |
     |------------------------------->|
     |                                |
```

---

## BLE 4.2 Data Length Extension

### 問題：BLE 4.0/4.1 的 MTU 限制

**BLE 4.0/4.1 的限制**：

- **Link Layer MTU**：27 bytes
- **L2CAP Header**：4 bytes
- **ATT Header**：3 bytes（Read Response）
- **實際 Payload**：20 bytes

**影響**：

- 傳輸大檔案（如：OTA 韌體更新）非常慢
- 每次傳輸只能傳 20 bytes，需要等待下一個 Connection Event

### 解決方案：Data Length Extension（BLE 4.2）

**BLE 4.2 引入的新功能**：

- **Link Layer MTU**：最大 251 bytes
- **L2CAP Header**：4 bytes
- **ATT Header**：3 bytes
- **實際 Payload**：最大 244 bytes

**速度提升**：

```
BLE 4.0/4.1: 20 bytes / 30 ms = 666 bytes/s
BLE 4.2:     244 bytes / 30 ms = 8,133 bytes/s

速度提升: 8,133 / 666 = 12.2 倍
```

### Data Length Extension 的協商流程

```
Central (手機)                    Peripheral (智慧手環)
     |                                |
     |  LL_LENGTH_REQ                 |
     |  (Max TX=251, Max RX=251)      |
     |------------------------------->|
     |                                |
     |  LL_LENGTH_RSP                 |
     |  (Max TX=251, Max RX=251)      |
     |<-------------------------------|
     |                                |
     |  (協商結果: 251 bytes)          |
     |                                |
```

**HCI 命令**：

```c
// 設定 Data Length
void hci_le_set_data_length(uint16_t handle, uint16_t tx_octets, uint16_t tx_time) {
    uint8_t cmd[] = {
        0x01,              // Packet Type: Command
        0x22, 0x20,        // Opcode: 0x2022 (HCI_LE_Set_Data_Length)
        0x06,              // Parameter Length: 6
        handle & 0xFF,     // Connection Handle (LSB)
        handle >> 8,       // Connection Handle (MSB)
        tx_octets & 0xFF,  // TX Octets (LSB) - 最大 251
        tx_octets >> 8,    // TX Octets (MSB)
        tx_time & 0xFF,    // TX Time (LSB) - 最大 2120 μs
        tx_time >> 8       // TX Time (MSB)
    };
    uart_send(cmd, 10);
}

// 範例：設定 Data Length 為 251 bytes
hci_le_set_data_length(connection_handle, 251, 2120);
```

---

## L2CAP Connection-Oriented Channels（BLE 4.1+）

### 什麼是 Connection-Oriented Channels？

**BLE 4.0 的限制**：

- 只支援固定的 Channel ID（0x0004 = ATT, 0x0006 = SMP）
- 無法動態建立新的通道

**BLE 4.1 引入的新功能**：

- **Connection-Oriented Channels**：動態建立的 L2CAP 通道
- **用途**：傳輸大量資料（如：檔案傳輸、音訊串流）

### Connection-Oriented Channels 的建立流程

```
Initiator (Client)                Responder (Server)
     |                                |
     |  LE Credit Based Connection    |
     |  Request                       |
     |  (SPSM, MTU, MPS, Credits)     |
     |------------------------------->|
     |                                |
     |  LE Credit Based Connection    |
     |  Response                      |
     |  (DCID, MTU, MPS, Credits)     |
     |<-------------------------------|
     |                                |
     |  (通道建立完成)                 |
     |                                |
     |  L2CAP Data (使用 DCID)        |
     |------------------------------->|
     |                                |
     |  LE Flow Control Credit        |
     |  (增加 Credits)                |
     |<-------------------------------|
     |                                |
```

**參數說明**：

- **SPSM（Simplified Protocol/Service Multiplexer）**：服務識別碼
- **MTU（Maximum Transmission Unit）**：最大傳輸單元
- **MPS（Maximum PDU Size）**：最大 PDU 大小
- **Credits**：流量控制的信用額度

---

## 實際案例：OTA 韌體更新的優化

### 問題分析

**原始方案（BLE 4.0）**：

- **MTU**：23 bytes（L2CAP Payload）
- **ATT Header**：3 bytes
- **實際 Payload**：20 bytes
- **Connection Interval**：30 ms
- **傳輸速度**：20 bytes / 30 ms = 666 bytes/s
- **128 KB 韌體**：128,000 / 666 = 192 秒 = 3.2 分鐘（理論值）
- **實際時間**：15 分鐘（考慮重傳、延遲）

### 優化方案 1：降低 Connection Interval

```c
// 將 Connection Interval 從 30 ms 降到 7.5 ms
void optimize_connection_interval(uint16_t handle) {
    // HCI_LE_Connection_Update
    uint8_t cmd[] = {
        0x01,              // Packet Type: Command
        0x13, 0x20,        // Opcode: 0x2013 (HCI_LE_Connection_Update)
        0x0E,              // Parameter Length: 14
        handle & 0xFF,     // Connection Handle (LSB)
        handle >> 8,       // Connection Handle (MSB)
        0x06, 0x00,        // Conn_Interval_Min: 6 * 1.25ms = 7.5ms
        0x06, 0x00,        // Conn_Interval_Max: 6 * 1.25ms = 7.5ms
        0x00, 0x00,        // Conn_Latency: 0
        0xC8, 0x00,        // Supervision_Timeout: 200 * 10ms = 2s
        0x00, 0x00,        // Minimum_CE_Length: 0
        0x00, 0x00         // Maximum_CE_Length: 0
    };
    uart_send(cmd, 18);
}

// 結果：
// 傳輸速度: 20 bytes / 7.5 ms = 2,666 bytes/s
// 128 KB 韌體: 128,000 / 2,666 = 48 秒 = 0.8 分鐘（理論值）
// 實際時間: 4 分鐘（考慮重傳、延遲）
```

### 優化方案 2：使用 Data Length Extension（BLE 4.2）

```c
// 設定 Data Length 為 251 bytes
hci_le_set_data_length(connection_handle, 251, 2120);

// 協商 MTU 為 512 bytes
att_exchange_mtu_request(512);

// 結果：
// MTU: 158 bytes (實際協商結果)
// ATT Header: 3 bytes
// 實際 Payload: 155 bytes
// Connection Interval: 30 ms
// 傳輸速度: 155 bytes / 30 ms = 5,166 bytes/s
// 128 KB 韌體: 128,000 / 5,166 = 24.8 秒 = 0.4 分鐘（理論值）
// 實際時間: 2 分鐘（考慮重傳、延遲）
```

### 優化方案 3：結合兩者

```c
// 1. 設定 Data Length 為 251 bytes
hci_le_set_data_length(connection_handle, 251, 2120);

// 2. 協商 MTU 為 512 bytes
att_exchange_mtu_request(512);

// 3. 降低 Connection Interval 到 7.5 ms
optimize_connection_interval(connection_handle);

// 結果：
// MTU: 158 bytes
// ATT Header: 3 bytes
// 實際 Payload: 155 bytes
// Connection Interval: 7.5 ms
// 傳輸速度: 155 bytes / 7.5 ms = 20,666 bytes/s
// 128 KB 韌體: 128,000 / 20,666 = 6.2 秒（理論值）
// 實際時間: 30 秒（考慮重傳、延遲）
```

**優化結果對比**：

| 方案 | Connection Interval | MTU | 傳輸速度 | 128 KB 韌體時間 |
|------|-------------------|-----|---------|---------------|
| 原始方案 | 30 ms | 23 bytes | 666 bytes/s | 15 分鐘 |
| 優化方案 1 | 7.5 ms | 23 bytes | 2,666 bytes/s | 4 分鐘 |
| 優化方案 2 | 30 ms | 158 bytes | 5,166 bytes/s | 2 分鐘 |
| 優化方案 3 | 7.5 ms | 158 bytes | 20,666 bytes/s | 30 秒 |

---

## 總結

L2CAP 是 Bluetooth 協定堆疊中的關鍵層級，負責資料的分段和重組、多工、流量控制。

**核心概念**：

- **Segmentation and Reassembly**：將大資料分段，接收端重組
- **Multiplexing**：使用 Channel ID 區分不同的協定
- **MTU Negotiation**：協商最大傳輸單元
- **Flow Control**：避免發送端發送過快

**BLE 4.2 的改進**：

- **Data Length Extension**：將 Link Layer MTU 從 27 bytes 提升到 251 bytes
- **速度提升**：12 倍以上

**實際應用**：

- **OTA 韌體更新**：透過優化 Connection Interval 和 MTU，將更新時間從 15 分鐘降到 30 秒
- **檔案傳輸**：使用 Connection-Oriented Channels 提升傳輸效率

在下一篇文章中，我們將深入探討 **ATT/GATT**，了解 BLE 的核心資料模型。

---

## 參考資料

1. Bluetooth SIG. (2013). *Bluetooth Core Specification Version 4.1 - Vol 3, Part A: L2CAP*. <https://www.bluetooth.com/specifications/specs/core-specification-4-1/>
2. Bluetooth SIG. (2014). *Bluetooth Core Specification Version 4.2 - Vol 3, Part A: L2CAP*. <https://www.bluetooth.com/specifications/specs/core-specification-4-2/>
3. Bluetooth SIG. (2014). *Data Length Extension White Paper*. <https://www.bluetooth.com/>
4. Nordic Semiconductor. (2015). *nRF52 Series - Data Length Extension Application Note*. <https://www.nordicsemi.com/>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
