---
author: Danny Jiang
date: 2025-12-11
---

# BLE 除錯實戰 - Sniffer, Log, 與測試工具

## 前言

2015 年 5 月的某個週四晚上，我正準備下班，手機突然響了。是客戶打來的，聲音聽起來很緊張。

「我們的智慧手環在測試時遇到了一個詭異的問題。」客戶說，「手環和手機可以正常連線，但是每隔 10-15 分鐘就會自動斷線。我們試了 10 支手環，都有同樣的問題。」

我的心一沉。這種隨機斷線的問題最難除錯，因為你不知道是什麼觸發的，也不知道是哪一層出了問題。是 BLE 協定的問題？還是韌體的 bug？還是硬體的干擾？

「我們明天有一個重要的展示會。」客戶繼續說，「如果這個問題不能解決，我們可能會失去這個訂單。」

我看了一眼時鐘，晚上 8 點。「我馬上過去。」我說。

我帶著筆電、BLE Sniffer、邏輯分析儀趕到客戶的辦公室。接下來的 6 個小時，我使用各種除錯工具追蹤這個問題：

1. **HCI Log**：查看 BLE 事件序列
2. **BLE Sniffer**：捕捉空中封包
3. **邏輯分析儀**：分析 HCI UART 訊號
4. **功耗分析儀**：觀察斷線時的電流波形

凌晨 2 點，我終於找到了問題：**Connection Supervision Timeout 設定太短**。當手機和手環之間有短暫的干擾（例如使用者把手環放在口袋裡），超過 4 秒沒有收到封包，連線就會超時斷開。

我修改了 Supervision Timeout 從 4 秒延長到 6 秒，重新測試。手環穩定運行了 2 個小時，沒有再斷線。

「太好了！」客戶鬆了一口氣，「你是怎麼找到這個問題的？」

「使用正確的工具，加上系統性的除錯方法。」我說，「BLE 除錯不是靠猜測，而是靠證據。」

這次經驗讓我深刻體會到：**掌握 BLE 除錯工具和方法，是每個 BLE 開發者的必備技能**。

---

## BLE 除錯工具概覽

### 工具分類

| 工具類型 | 用途 | 代表工具 |
|----------|------|----------|
| **BLE Sniffer** | 捕捉空中封包（Over-the-Air） | Ellisys, Frontline, Nordic nRF Sniffer |
| **HCI Log** | 查看 Host-Controller 通訊 | Wireshark, Android HCI Log |
| **邏輯分析儀** | 分析數位訊號（UART, SPI, I2C） | Saleae Logic, Rigol |
| **功耗分析儀** | 測量電流波形 | Nordic PPK2, Keysight N6705B |
| **Protocol Tester** | 自動化測試 | Bluetooth SIG PTS |

### 工具選擇指南

| 問題類型 | 推薦工具 | 理由 |
|----------|----------|------|
| 連線失敗 | BLE Sniffer | 查看空中封包，確認連線請求是否發送 |
| 配對失敗 | BLE Sniffer + HCI Log | 查看 SMP 配對流程 |
| 資料遺失 | BLE Sniffer | 查看 L2CAP 封包是否正確傳輸 |
| 隨機斷線 | BLE Sniffer + 功耗分析儀 | 查看斷線前的封包和電流波形 |
| HCI 通訊問題 | 邏輯分析儀 | 分析 UART 訊號時序 |

---

## BLE Sniffer - 捕捉空中封包

### 什麼是 BLE Sniffer？

**BLE Sniffer** 是一種專門用於捕捉 BLE 空中封包的工具。它可以：

1. **監聽 Advertising 封包**：查看裝置的 Advertising 內容
2. **追蹤 Connection**：捕捉連線建立和資料傳輸
3. **解析協定**：自動解析 L2CAP、ATT、GATT、SMP 等協定
4. **時間戳記**：精確記錄每個封包的時間

### 常見的 BLE Sniffer

#### 1. **Nordic nRF Sniffer**

**優點**：
- 免費（需要 nRF52840 Dongle）
- 與 Wireshark 整合
- 支援 BLE 4.x 和 5.x

**缺點**：
- 只能追蹤一個連線
- 需要手動配置 Access Address

**使用方法**：

```bash
# 1. 安裝 nRF Sniffer for Bluetooth LE
# 2. 啟動 Wireshark
# 3. 選擇 nRF Sniffer 介面
# 4. 開始捕捉

# 過濾 Advertising 封包
btle.advertising_address == 12:34:56:78:9a:bc

# 過濾特定 Connection
btle.access_address == 0x8e89bed6

# 過濾 ATT 封包
btatt

```

#### 2. **Ellisys Bluetooth Analyzer**

**優點**：
- 專業級工具
- 可以同時追蹤多個連線
- 自動追蹤 Connection（不需要手動配置）
- 強大的過濾和分析功能

**缺點**：
- 價格昂貴（$5,000 - $20,000）

#### 3. **Frontline ComProbe**

**優點**：
- 支援 Bluetooth Classic 和 BLE
- 可以捕捉 HCI 和 Over-the-Air 封包
- 強大的協定分析功能

**缺點**：
- 價格昂貴

### BLE Sniffer 使用範例

**場景 1：查看 Advertising 內容**

```
# Wireshark 過濾器
btle.advertising_address == 12:34:56:78:9a:bc && btle.advertising_header.pdu_type == 0x00

# 捕捉結果
Frame 1: ADV_IND
  Advertising Address: 12:34:56:78:9a:bc
  Advertising Data:
    Flags: 0x06 (LE General Discoverable, BR/EDR Not Supported)
    Complete Local Name: "MyDevice"
    TX Power Level: 0 dBm
    Service UUID: 0x180D (Heart Rate Service)

```

**場景 2：追蹤連線建立**

```
# 連線建立流程
Frame 10: ADV_IND (Peripheral advertising)
Frame 11: SCAN_REQ (Central scanning)
Frame 12: SCAN_RSP (Peripheral response)
Frame 13: CONNECT_REQ (Central initiating connection)
  Access Address: 0x8e89bed6
  CRC Init: 0x123456
  WinSize: 10
  WinOffset: 5
  Interval: 24 (30 ms)
  Latency: 0
  Timeout: 200 (2000 ms)
  Channel Map: 0x1FFFFFFFFF (all channels)
  Hop: 9
  SCA: 50 ppm

Frame 14: LL_DATA (First data packet)
  Access Address: 0x8e89bed6
  LLID: 0x02 (LL Data PDU, start of L2CAP message)
  Data: [ATT Exchange MTU Request]

```

**場景 3：分析配對流程**

```
# SMP 配對流程
Frame 20: ATT Write Request (Pairing Request)
  Handle: 0x0001
  Value: [SMP Pairing Request]
    IO Capability: Display Only
    OOB: Not Present
    Auth Req: Bonding, MITM
    Max Key Size: 16
    Initiator Key Distribution: EncKey, IdKey
    Responder Key Distribution: EncKey, IdKey

Frame 21: ATT Write Response (Pairing Response)
  Value: [SMP Pairing Response]
    IO Capability: Keyboard Only
    OOB: Not Present
    Auth Req: Bonding, MITM
    Max Key Size: 16

Frame 22: SMP Pairing Confirm
Frame 23: SMP Pairing Random
Frame 24: SMP Pairing Confirm
Frame 25: SMP Pairing Random
Frame 26: SMP Encryption Information
Frame 27: SMP Master Identification
Frame 28: SMP Identity Information
Frame 29: SMP Identity Address Information

```

---

## HCI Log 分析

### 什麼是 HCI Log？

**HCI Log** 記錄了 Host 和 Controller 之間的所有通訊，包括：

1. **HCI Commands**：Host 發送給 Controller 的命令
2. **HCI Events**：Controller 發送給 Host 的事件
3. **HCI ACL Data**：應用層資料

### 啟用 HCI Log

#### Android

```bash
# 1. 開啟開發者選項
# 2. 啟用「藍牙 HCI 信息收集記錄」
# 3. 重現問題
# 4. 匯出 Log

adb pull /sdcard/btsnoop_hci.log

```

#### iOS

```bash
# iOS 不直接支援 HCI Log
# 需要使用 PacketLogger（需要 Apple Developer Account）

```

#### nRF52 (Nordic SDK)

```c
// 在 sdk_config.h 中啟用 HCI Log
#define NRF_LOG_ENABLED 1
#define NRF_LOG_BACKEND_UART_ENABLED 1

// 在程式碼中輸出 HCI Log
void ble_evt_handler(ble_evt_t const *p_ble_evt) {
    NRF_LOG_INFO("BLE Event: 0x%04X", p_ble_evt->header.evt_id);
    
    switch (p_ble_evt->header.evt_id) {
        case BLE_GAP_EVT_CONNECTED:
            NRF_LOG_INFO("Connected: conn_handle=%d, role=%d",
                         p_ble_evt->evt.gap_evt.conn_handle,
                         p_ble_evt->evt.gap_evt.params.connected.role);
            break;
            
        case BLE_GAP_EVT_DISCONNECTED:
            NRF_LOG_INFO("Disconnected: reason=0x%02X",
                         p_ble_evt->evt.gap_evt.params.disconnected.reason);
            break;
            
        case BLE_GATTS_EVT_WRITE:
            NRF_LOG_HEXDUMP_INFO(p_ble_evt->evt.gatts_evt.params.write.data,
                                 p_ble_evt->evt.gatts_evt.params.write.len);
            break;
    }
}

```

### HCI Log 分析範例

**場景：連線斷線分析**

```
# HCI Log (Wireshark 格式)
Frame 100: HCI Event - Connection Complete
  Status: Success
  Connection Handle: 0x0001
  Role: Peripheral
  Peer Address: AA:BB:CC:DD:EE:FF

Frame 101-500: HCI ACL Data (正常資料傳輸)

Frame 501: HCI Event - Disconnection Complete
  Status: Success
  Connection Handle: 0x0001
  Reason: 0x08 (Connection Timeout)

```

**分析**：
- 斷線原因是 **Connection Timeout (0x08)**
- 表示 Peripheral 在 Supervision Timeout 時間內沒有收到任何封包
- 可能原因：

  1. 訊號干擾
  2. Supervision Timeout 設定太短
  3. Central 進入 Deep Sleep 太久

---

## 常見 BLE 問題除錯

### 問題 1：連線失敗

**症狀**：裝置可以被掃描到，但無法連線

**除錯步驟**：

1. **使用 BLE Sniffer 捕捉 Advertising 和 Connection 封包**

```
# 檢查 Advertising 封包
- ADV_IND 是否正常發送？
- Advertising Interval 是否合理？
- Advertising Data 是否正確？

# 檢查 Connection 封包
- CONNECT_REQ 是否發送？
- Connection Parameters 是否合理？
- 是否有 LL_DATA 封包？

```

2. **檢查 HCI Log**

```c
// 查看連線事件
BLE_GAP_EVT_CONNECTED: 是否收到？
BLE_GAP_EVT_CONN_PARAM_UPDATE: 參數是否合理？

```

3. **常見原因**：

| 原因 | 解決方案 |
|------|----------|
| Advertising 停止 | 檢查 Advertising Timeout 設定 |
| Connection Parameters 不合理 | 調整 Connection Interval, Latency, Timeout |
| MTU Exchange 失敗 | 檢查 MTU 設定 |
| 配對失敗 | 檢查 SMP 配對流程 |

### 問題 2：配對失敗

**症狀**：配對過程中斷，或配對後無法通訊

**除錯步驟**：

1. **使用 BLE Sniffer 捕捉 SMP 封包**

```
# SMP 配對流程
1. Pairing Request
2. Pairing Response
3. Pairing Confirm (Initiator)
4. Pairing Random (Initiator)
5. Pairing Confirm (Responder)
6. Pairing Random (Responder)
7. Encryption Information
8. Master Identification
9. Identity Information
10. Identity Address Information

# 檢查哪一步失敗

```

2. **檢查 IO Capability 和 Auth Req**

```c
// Peripheral 的 SMP 設定
ble_gap_sec_params_t sec_params = {
    .bond = 1,
    .mitm = 1,  // Man-In-The-Middle protection
    .lesc = 0,  // LE Secure Connections (BLE 4.2+)
    .keypress = 0,
    .io_caps = BLE_GAP_IO_CAPS_DISPLAY_ONLY,  // 只有 Display
    .oob = 0,
    .min_key_size = 7,
    .max_key_size = 16,
};

// 檢查 Central 和 Peripheral 的 IO Capability 是否匹配

```

3. **常見原因**：

| 原因 | 解決方案 |
|------|----------|
| IO Capability 不匹配 | 調整 IO Caps（例如都使用 NoInputNoOutput） |
| MITM 要求不匹配 | 調整 Auth Req |
| Key Size 不匹配 | 調整 Min/Max Key Size |
| Passkey 輸入錯誤 | 檢查 Passkey 顯示和輸入 |

### 問題 3：資料遺失

**症狀**：部分資料沒有傳輸成功

**除錯步驟**：

1. **使用 BLE Sniffer 捕捉 L2CAP 封包**

```
# 檢查 L2CAP 封包
- 是否有封包遺失？
- 是否有重傳？
- MTU 是否足夠？

```

2. **檢查 ATT MTU**

```c
// 查看 MTU Exchange
BLE_GATTS_EVT_EXCHANGE_MTU_REQUEST:
  Client RX MTU: 247
  
// 設定 Server MTU
ble_gap_conn_params_t conn_params = {
    .min_conn_interval = MSEC_TO_UNITS(7.5, UNIT_1_25_MS),
    .max_conn_interval = MSEC_TO_UNITS(30, UNIT_1_25_MS),
    .slave_latency = 0,
    .conn_sup_timeout = MSEC_TO_UNITS(4000, UNIT_10_MS),
};

// 設定 GATT MTU
#define NRF_SDH_BLE_GATT_MAX_MTU_SIZE 247

```

3. **檢查 Connection Parameters**

```c
// Connection Interval 太大會導致吞吐量低
// 計算吞吐量：
// Throughput = (MTU - 3) / Connection Interval
// 例如：MTU = 247, Interval = 30 ms
// Throughput = (247 - 3) / 0.03 = 8.13 KB/s

```

4. **常見原因**：

| 原因 | 解決方案 |
|------|----------|
| MTU 太小 | 增加 MTU（最大 247 for BLE 4.2） |
| Connection Interval 太大 | 減小 Connection Interval |
| 訊號干擾 | 改善天線設計、遠離干擾源 |
| Buffer 溢位 | 增加 Buffer 大小、使用流控 |

### 問題 4：隨機斷線

**症狀**：連線一段時間後隨機斷開

**除錯步驟**：

1. **使用 BLE Sniffer 捕捉斷線前的封包**

```
# 查看最後幾個封包
Frame 498: LL_DATA (正常資料)
Frame 499: LL_DATA (正常資料)
Frame 500: (沒有封包)
...
Frame 600: (沒有封包)

# 斷線原因
- Connection Timeout: 在 Supervision Timeout 時間內沒有收到封包

```

2. **使用功耗分析儀觀察電流波形**

```c
// 檢查斷線時的電流
- 是否有異常的電流尖峰？（可能是干擾）
- 是否進入 Deep Sleep 太久？（超過 Supervision Timeout）

```

3. **檢查 Connection Parameters**

```c
// Supervision Timeout 必須滿足：
// Timeout > (1 + Latency) × Interval × 2

// 範例：
Connection Interval = 200 ms
Slave Latency = 4
Supervision Timeout = 4000 ms

// 檢查：
(1 + 4) × 200 ms × 2 = 2000 ms < 4000 ms  ✓

// 如果 Timeout 太短，可能導致隨機斷線

```

4. **常見原因**：

| 原因 | 解決方案 |
|------|----------|
| Supervision Timeout 太短 | 增加 Timeout（建議 4-6 秒） |
| 訊號干擾 | 改善天線設計、使用 AFH |
| Peripheral 進入 Deep Sleep 太久 | 調整 Sleep 策略 |
| Central 進入 Deep Sleep 太久 | 無法控制（iOS/Android 系統行為） |

---

## 進階除錯技巧

### 技巧 1：使用 Logic Analyzer 分析 HCI UART

**場景**：懷疑 HCI UART 通訊有問題

**工具**：Saleae Logic Analyzer

**步驟**：

1. **連接 Logic Analyzer 到 UART TX/RX**

```
nRF52 UART TX (Pin 6) -> Logic Analyzer Channel 0
nRF52 UART RX (Pin 8) -> Logic Analyzer Channel 1
GND -> Logic Analyzer GND

```

2. **配置 Logic Analyzer**

```
Baud Rate: 115200
Data Bits: 8
Parity: None
Stop Bits: 1

```

3. **捕捉 HCI 封包**

```
# HCI Command: Reset
01 03 0C 00

# HCI Event: Command Complete
04 0E 04 01 03 0C 00

# 檢查：
- Baud Rate 是否正確？
- 是否有 Framing Error？
- 是否有資料遺失？

```

### 技巧 2：使用 Power Profiler 觀察 BLE 電流波形

**場景**：分析 BLE 功耗或除錯斷線問題

**工具**：Nordic Power Profiler Kit (PPK2)

**步驟**：

1. **連接 PPK2**

```
PPK2 VOUT -> nRF52 VDD
PPK2 GND -> nRF52 GND

```

2. **觀察電流波形**

```
# Advertising 波形
每 1000 ms 一個尖峰（15 mA，持續 1 ms）

# Connection 波形
每 200 ms 一個尖峰（15 mA，持續 2 ms）

# 斷線時的波形
- 是否有異常的電流尖峰？
- 是否進入 Deep Sleep 太久？

```

### 技巧 3：使用 Wireshark 過濾器

**常用過濾器**：

```bash
# 過濾特定裝置的 Advertising
btle.advertising_address == 12:34:56:78:9a:bc

# 過濾特定 Connection
btle.access_address == 0x8e89bed6

# 過濾 ATT 封包
btatt

# 過濾 ATT Write Request
btatt.opcode == 0x12

# 過濾 ATT Notification
btatt.opcode == 0x1b

# 過濾 SMP 封包
btsmp

# 過濾 L2CAP 封包
btl2cap

```

---

## Bluetooth SIG 認證測試

### 什麼是 Bluetooth SIG 認證？

**Bluetooth SIG 認證** 是確保 Bluetooth 產品符合規範的測試流程。包括：

1. **RF Testing**：射頻測試（發射功率、接收靈敏度、調變特性）
2. **Protocol Testing**：協定測試（使用 PTS 工具）
3. **Profile Testing**：Profile 測試（GATT Services）
4. **Interoperability Testing**：互通性測試

### Profile Tuning Test Suite (PTS)

**PTS** 是 Bluetooth SIG 提供的自動化測試工具。

**測試項目**：

```
# GAP 測試
GAP/DISC/GENP/BV-01-C: General Discoverable Mode
GAP/CONN/NCON/BV-01-C: Non-Connectable Mode
GAP/CONN/DCON/BV-01-C: Directed Connectable Mode

# GATT 測試
GATT/SR/GAD/BV-01-C: Discover All Primary Services
GATT/SR/GAR/BV-01-C: Read Characteristic Value
GATT/SR/GAW/BV-01-C: Write Characteristic Value

# SMP 測試
SM/MAS/PROT/BV-01-C: Pairing - Just Works
SM/MAS/PROT/BV-02-C: Pairing - Passkey Entry

```

**常見失敗原因**：

| 測試項目 | 失敗原因 | 解決方案 |
|----------|----------|----------|
| GAP/DISC/GENP/BV-01-C | Advertising Data 不正確 | 檢查 Flags, Local Name |
| GATT/SR/GAR/BV-01-C | Read Response 超時 | 增加 Response Timeout |
| SM/MAS/PROT/BV-01-C | Pairing 失敗 | 檢查 IO Capability, Auth Req |

---

## 實務建議

### 除錯流程

1. **重現問題**：

   - 確保問題可以穩定重現
   - 記錄重現步驟

2. **收集資訊**：

   - HCI Log
   - BLE Sniffer 捕捉
   - 功耗波形
   - 邏輯分析儀資料

3. **分析資料**：

   - 識別問題發生的時間點
   - 查看問題前後的封包和事件
   - 對比正常和異常的差異

4. **提出假設**：

   - 根據資料提出可能的原因
   - 設計實驗驗證假設

5. **修復和驗證**：

   - 修改程式碼或參數
   - 重新測試驗證

### 除錯工具箱

**必備工具**：
- ✅ BLE Sniffer（Nordic nRF Sniffer 或 Ellisys）
- ✅ Wireshark（分析 HCI Log 和 Sniffer 捕捉）
- ✅ 邏輯分析儀（Saleae Logic）
- ✅ 功耗分析儀（Nordic PPK2）

**可選工具**：
- ✅ Protocol Tester（Bluetooth SIG PTS）
- ✅ Spectrum Analyzer（分析射頻干擾）
- ✅ 示波器（分析類比訊號）

### 常見錯誤碼

| 錯誤碼 | 名稱 | 說明 | 常見原因 |
|--------|------|------|----------|
| 0x08 | Connection Timeout | 連線超時 | Supervision Timeout 太短、訊號干擾 |
| 0x13 | Remote User Terminated Connection | 遠端使用者終止連線 | 正常斷線 |
| 0x16 | Connection Terminated by Local Host | 本地主機終止連線 | 正常斷線 |
| 0x22 | LMP Response Timeout | LMP 回應超時 | 訊號問題 |
| 0x3D | Connection Failed to be Established | 連線建立失敗 | Connection Parameters 不合理 |

---

## 總結

BLE 除錯是一門藝術，也是一門科學。透過正確的工具和系統性的方法，我們可以快速定位和解決 BLE 問題。

**核心原則**：
1. **使用正確的工具**：BLE Sniffer, HCI Log, Logic Analyzer, Power Profiler
2. **系統性的方法**：重現問題 → 收集資訊 → 分析資料 → 提出假設 → 驗證修復
3. **理解協定**：深入理解 BLE 協定堆疊（見文章 #2-6）
4. **積累經驗**：記錄常見問題和解決方案

掌握這些技能，你就能成為 BLE 除錯的專家，快速解決各種詭異的問題。

---

## 參考資料

1. Bluetooth SIG. (2014). *Bluetooth Core Specification 4.1*. <https://www.bluetooth.com/specifications/specs/core-specification-4-1/>
2. Nordic Semiconductor. (2016). *nRF Sniffer for Bluetooth LE User Guide*. <https://infocenter.nordicsemi.com/topic/ug_sniffer_ble/UG/sniffer_ble/intro.html>
3. Ellisys. (2015). *Bluetooth Analyzer User Guide*. <https://www.ellisys.com/products/bta/>
4. Bluetooth SIG. (2015). *Profile Tuning Suite (PTS) User Guide*. <https://www.bluetooth.com/develop-with-bluetooth/qualification-listing/qualification-test-tools/profile-tuning-suite/>

---

**版權聲明**

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。  
**作者**：Danny Jiang  
**出處**：<https://github.com/djiangtw/tech-column-public>
