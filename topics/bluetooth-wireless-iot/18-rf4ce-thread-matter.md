---
author: Danny Jiang
date: 2025-12-11
---

# RF4CE, Thread, 與 Matter - IoT 協定的演進

## 前言：2014 年的遙控器專案

2014 年 12 月，我接到了一個來自電視廠商的專案需求。

"Danny，我們想做一個新的電視遙控器，"客戶的產品經理 Michael 說，"要支援語音控制、觸控板、體感操作。你覺得應該用什麼無線協定？"

我愣了一下。這是一個有趣的問題。

"傳統的紅外線（IR）遙控器肯定不行，"我說，"IR 需要對準電視，而且不支援雙向通訊。你們考慮過 Bluetooth 嗎？"

"考慮過，"Michael 說，"但 Bluetooth 有幾個問題：

1. **配對麻煩**：使用者需要手動配對，體驗不好
2. **延遲較高**：Bluetooth 的延遲大約 50-100 ms，對於遙控器來說太慢了
3. **功耗較高**：Bluetooth 的功耗比 IR 高，電池續航時間短

我們聽說有一個叫 **RF4CE** 的協定，專門為遙控器設計的。你了解嗎？"

"RF4CE？"我想了想，"我聽說過，但沒有實際用過。讓我研究一下。"

---

掛上電話後，我開始研究 RF4CE。

**RF4CE（Radio Frequency for Consumer Electronics）** 是一個基於 **IEEE 802.15.4** 的無線協定，專門為消費性電子產品（特別是遙控器）設計。

**RF4CE 的特點**：

- **低延遲**：< 30 ms（比 Bluetooth 快）
- **低功耗**：比 Bluetooth 低 50%
- **簡單配對**：按一個按鈕就能配對
- **專為遙控器設計**：支援按鍵、語音、體感

這看起來很適合電視遙控器！

但我也發現了一些問題：

- **生態系統小**：只有少數晶片廠商支援（主要是 Texas Instruments, Freescale）
- **應用範圍窄**：只適合遙控器，不適合其他 IoT 應用
- **未來不明**：RF4CE 的發展前景不明朗

我開始思考：**RF4CE 真的是最好的選擇嗎？還是應該用 Bluetooth？**

---

週一早上，我打電話給 Michael。

"我研究了 RF4CE，"我說，"它確實很適合遙控器。但我建議你們也考慮一下 **Bluetooth Low Energy（BLE）**。"

"為什麼？"Michael 問，"你剛才不是說 BLE 有延遲和功耗的問題嗎？"

"是的，"我說，"但 BLE 有一個巨大的優勢：**生態系統成熟**。

- **晶片廠商多**：Nordic, TI, Cypress, Dialog 都支援 BLE
- **開發工具完善**：SDK, IDE, Debugger 都很成熟
- **未來發展好**：BLE 會持續演進（BLE 4.2, 5.0, 5.1...）

而 RF4CE 的生態系統很小，未來發展不明朗。如果你們選擇 RF4CE，可能會面臨：

- **晶片供應風險**：只有少數廠商支援
- **開發資源少**：社群小，資源少
- **未來升級困難**：RF4CE 可能會被淘汰"

Michael 沉默了一會兒。

"你說得有道理，"他說，"但 BLE 的延遲和功耗問題怎麼解決？"

"延遲可以透過優化 Connection Interval 來降低，"我說，"功耗可以透過 Sleep 模式來降低。我可以幫你們做一個 PoC（Proof of Concept），證明 BLE 可以滿足你們的需求。"

"好，"Michael 說，"那就麻煩你了。"

---

兩週後，我完成了 BLE 遙控器的 PoC：

**測試結果**：

- **延遲**：< 30 ms（Connection Interval = 7.5 ms）
- **功耗**：平均 50 μA（使用 Sleep 模式）
- **電池續航**：> 1 年（2 顆 AAA 電池）
- **配對**：按一個按鈕就能配對（使用 Just Works Pairing）

Michael 非常滿意："這個結果比我預期的好！我們決定用 BLE。"

---

這次經歷讓我深刻理解了 **協定選擇** 的重要性。有時候，**最新的協定不一定是最好的選擇**。選擇協定時，要考慮：

- **技術需求**（延遲、功耗、通訊距離）
- **生態系統**（晶片廠商、開發工具、社群）
- **未來發展**（協定演進、市場趨勢）

只有綜合考慮這些因素，才能做出正確的決策。

---

## RF4CE 協定概覽

### RF4CE 是什麼？

**RF4CE（Radio Frequency for Consumer Electronics）** 是一個基於 **IEEE 802.15.4** 的無線協定，由 **Zigbee Alliance** 和 **RF4CE Consortium** 共同開發，專門為消費性電子產品（特別是遙控器）設計。

**RF4CE 的特點**：

- **工作頻段**：2.4 GHz
- **資料速率**：250 kbps
- **通訊距離**：10-50 m
- **延遲**：< 30 ms
- **功耗**：低功耗（Sleep 模式 < 1 μA）
- **配對**：簡單配對（按一個按鈕）

**RF4CE 的應用場景**：

- 電視遙控器
- 機上盒遙控器
- 遊戲手把
- 3D 眼鏡

---

### RF4CE 協定堆疊

RF4CE 協定堆疊分為三層：

```
┌─────────────────────────────────────────────────────────┐
│  RF4CE 協定堆疊                                           │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─────────────────────────────────────────────┐         │
│  │  Application Layer                          │         │
│  │  - ZRC (Zigbee Remote Control)              │         │
│  │  - ZID (Zigbee Input Device)                │         │
│  │  - MSO (Multiple System Operators)          │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                         │
│  ┌──────────────┴──────────────────────────────┐         │
│  │  RF4CE Network Layer                        │         │
│  │  - Pairing                                  │         │
│  │  - Discovery                                │         │
│  │  - Data Transfer                            │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                         │
│  ┌──────────────┴──────────────────────────────┐         │
│  │  MAC Layer (IEEE 802.15.4)                  │         │
│  │  - CSMA/CA                                  │         │
│  │  - Frame Format                             │         │
│  │  - ACK Mechanism                            │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                         │
│  ┌──────────────┴──────────────────────────────┐         │
│  │  PHY Layer (IEEE 802.15.4)                  │         │
│  │  - 2.4 GHz                                  │         │
│  │  - DSSS Modulation                          │         │
│  │  - Channel Selection                        │         │
│  └─────────────────────────────────────────────┘         │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**各層功能**：

1. **PHY Layer**：
   - 與 Zigbee 相同，使用 IEEE 802.15.4 PHY
   - 2.4 GHz 頻段，16 個 Channel

2. **MAC Layer**：
   - 與 Zigbee 相同，使用 IEEE 802.15.4 MAC
   - CSMA/CA 媒體存取控制

3. **RF4CE Network Layer**：
   - 負責配對、發現、資料傳輸
   - 比 Zigbee 簡單（不支援 Mesh Network）

4. **Application Layer**：
   - ZRC（Zigbee Remote Control）：遙控器應用
   - ZID（Zigbee Input Device）：輸入裝置（鍵盤、滑鼠）
   - MSO（Multiple System Operators）：有線電視運營商專用

### RF4CE vs BLE 對比

| 特性 | RF4CE | BLE |
|------|-------|-----|
| **延遲** | < 30 ms | 30-100 ms (可優化到 < 30 ms) |
| **功耗** | 低 | 中（可優化） |
| **配對** | 簡單（按一個按鈕） | 簡單（Just Works Pairing） |
| **通訊距離** | 10-50 m | 10-50 m |
| **生態系統** | 小（TI, Freescale） | 大（Nordic, TI, Cypress, Dialog） |
| **應用範圍** | 遙控器 | 廣泛（穿戴式、智慧家居、IoT） |
| **未來發展** | 不明朗 | 持續演進（BLE 5.0, 5.1, 5.2...） |

**分析**：

- **RF4CE 的優勢**：延遲低、功耗低、專為遙控器設計
- **BLE 的優勢**：生態系統大、應用範圍廣、未來發展好

**結論**：

- **2014-2016 年**：RF4CE 是遙控器的主流選擇
- **2016 年後**：BLE 逐漸取代 RF4CE，成為遙控器的主流選擇

---

## Thread 協定概覽

### Thread 是什麼？

**Thread** 是一個基於 **IEEE 802.15.4** 的無線協定，由 **Thread Group**（現在是 **CSA - Connectivity Standards Alliance**）開發，專門為智慧家居設計。

**Thread 的特點**：

- **工作頻段**：2.4 GHz
- **資料速率**：250 kbps
- **通訊距離**：10-100 m
- **網路拓撲**：Mesh Network
- **IP-based**：使用 IPv6，可以直接連接到網際網路
- **低功耗**：Sleep 模式 < 1 μA

**Thread 的應用場景**：

- 智慧家居（智慧燈泡、智慧插座、智慧門鎖）
- 智慧建築（HVAC 控制、能源管理）
- 工業 IoT

---

### Thread 協定堆疊

Thread 協定堆疊分為五層：

```
┌─────────────────────────────────────────────────────────┐
│  Thread 協定堆疊                                          │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─────────────────────────────────────────────┐         │
│  │  Application Layer                          │         │
│  │  - CoAP (Constrained Application Protocol)  │         │
│  │  - DTLS (Datagram TLS)                      │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                         │
│  ┌──────────────┴──────────────────────────────┐         │
│  │  Transport Layer                            │         │
│  │  - UDP                                      │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                         │
│  ┌──────────────┴──────────────────────────────┐         │
│  │  Network Layer                              │         │
│  │  - IPv6 (6LoWPAN)                           │         │
│  │  - ICMPv6                                   │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                         │
│  ┌──────────────┴──────────────────────────────┐         │
│  │  MAC Layer (IEEE 802.15.4)                  │         │
│  │  - CSMA/CA                                  │         │
│  │  - Frame Format                             │         │
│  │  - ACK Mechanism                            │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                         │
│  ┌──────────────┴──────────────────────────────┐         │
│  │  PHY Layer (IEEE 802.15.4)                  │         │
│  │  - 2.4 GHz                                  │         │
│  │  - DSSS Modulation                          │         │
│  │  - Channel Selection                        │         │
│  └─────────────────────────────────────────────┘         │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**各層功能**：

1. **PHY Layer**：
   - 與 Zigbee 相同，使用 IEEE 802.15.4 PHY

2. **MAC Layer**：
   - 與 Zigbee 相同，使用 IEEE 802.15.4 MAC

3. **Network Layer**：
   - 使用 **IPv6**（6LoWPAN - IPv6 over Low-Power Wireless Personal Area Networks）
   - 每個裝置都有一個 IPv6 地址
   - 可以直接連接到網際網路

4. **Transport Layer**：
   - 使用 **UDP**（User Datagram Protocol）

5. **Application Layer**：
   - 使用 **CoAP**（Constrained Application Protocol）
   - 使用 **DTLS**（Datagram TLS）進行加密

---

### Thread vs Zigbee 對比

| 特性 | Thread | Zigbee |
|------|--------|--------|
| **PHY/MAC** | IEEE 802.15.4 | IEEE 802.15.4 |
| **Network Layer** | IPv6 (6LoWPAN) | Zigbee NWK |
| **Application Layer** | CoAP | Zigbee Clusters |
| **IP-based** | 是 | 否 |
| **可連接網際網路** | 是 | 否（需要 Gateway） |
| **生態系統** | 中等（Google, Apple, Samsung） | 大（Philips, IKEA, Samsung） |
| **應用範圍** | 智慧家居 | 智慧家居、工業 IoT |

**分析**：

- **Thread 的優勢**：IP-based，可以直接連接到網際網路，不需要 Gateway
- **Zigbee 的優勢**：生態系統更大，應用範圍更廣

---

## Matter 協定概覽

### Matter 是什麼？

**Matter**（原名 **Project CHIP - Connected Home over IP**）是一個由 **CSA（Connectivity Standards Alliance）** 開發的智慧家居協定，旨在統一智慧家居的通訊標準。

**Matter 的特點**：

- **統一標準**：支援 WiFi, Thread, Ethernet
- **IP-based**：使用 IPv6
- **跨平台**：支援 Apple HomeKit, Google Home, Amazon Alexa, Samsung SmartThings
- **安全性高**：使用 TLS 1.3 加密
- **開源**：完全開源，任何人都可以使用

**Matter 的應用場景**：

- 智慧家居（智慧燈泡、智慧插座、智慧門鎖、智慧恆溫器）
- 智慧建築
- 工業 IoT

---

### Matter 協定堆疊

Matter 協定堆疊分為四層：

```
┌─────────────────────────────────────────────────────────┐
│  Matter 協定堆疊                                          │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─────────────────────────────────────────────┐         │
│  │  Application Layer                          │         │
│  │  - Matter Data Model                        │         │
│  │  - Clusters (On/Off, Level Control, etc.)   │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                         │
│  ┌──────────────┴──────────────────────────────┐         │
│  │  Interaction Layer                          │         │
│  │  - Read, Write, Subscribe, Invoke           │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                         │
│  ┌──────────────┴──────────────────────────────┐         │
│  │  Security Layer                             │         │
│  │  - TLS 1.3                                  │         │
│  │  - PASE (Password Authenticated Session)    │         │
│  │  - CASE (Certificate Authenticated Session) │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                         │
│  ┌──────────────┴──────────────────────────────┐         │
│  │  Transport Layer                            │         │
│  │  - WiFi, Thread, Ethernet                   │         │
│  │  - UDP, TCP                                 │         │
│  └─────────────────────────────────────────────┘         │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**各層功能**：

1. **Transport Layer**：
   - 支援多種傳輸方式：WiFi, Thread, Ethernet
   - 使用 UDP 或 TCP

2. **Security Layer**：
   - 使用 **TLS 1.3** 加密
   - PASE（Password Authenticated Session）：使用密碼配對
   - CASE（Certificate Authenticated Session）：使用憑證配對

3. **Interaction Layer**：
   - Read：讀取裝置狀態
   - Write：寫入裝置狀態
   - Subscribe：訂閱裝置狀態變化
   - Invoke：呼叫裝置命令

4. **Application Layer**：
   - Matter Data Model：定義裝置的資料模型
   - Clusters：定義裝置的功能（例如：On/Off, Level Control, Color Control）

---

### Matter 的優勢

**1. 統一標準**：

- 支援多種傳輸方式（WiFi, Thread, Ethernet）
- 不同廠商的裝置可以互相通訊
- 使用者不需要擔心相容性問題

**2. 跨平台**：

- 支援 Apple HomeKit, Google Home, Amazon Alexa, Samsung SmartThings
- 使用者可以用任何平台控制裝置

**3. 安全性高**：

- 使用 TLS 1.3 加密
- 使用憑證配對，防止中間人攻擊

**4. 開源**：

- 完全開源，任何人都可以使用
- 降低開發成本

---

## IoT 協定的演進

讓我們回顧一下 IoT 協定的演進歷程：

### 2010-2014 年：Zigbee 時代

**主流協定**：Zigbee

**特點**：

- Mesh Network
- 大量裝置
- 通訊距離遠

**應用**：

- 智慧家居（Philips Hue）
- 工業 IoT

**問題**：

- 手機不支援，需要專用 Hub
- 生態系統較小

---

### 2014-2017 年：BLE 崛起

**主流協定**：BLE, Zigbee

**特點**：

- BLE：手機原生支援，生態系統大
- Zigbee：Mesh Network，大量裝置

**應用**：

- BLE：穿戴式裝置、智慧門鎖
- Zigbee：智慧照明、工業 IoT

**問題**：

- BLE 不支援 Mesh Network
- Zigbee 需要專用 Hub

---

### 2017-2020 年：BLE Mesh 與 Thread

**主流協定**：BLE Mesh, Thread, Zigbee

**特點**：

- BLE Mesh：結合 BLE 的普及性和 Mesh Network 的可靠性
- Thread：IP-based，可以直接連接到網際網路

**應用**：

- BLE Mesh：智慧照明、智慧建築
- Thread：智慧家居（Google Nest）

**問題**：

- 協定太多，相容性問題
- 使用者需要選擇平台（Apple HomeKit, Google Home, Amazon Alexa）

---

### 2020 年後：Matter 時代

**主流協定**：Matter (WiFi, Thread, Ethernet)

**特點**：

- 統一標準
- 跨平台
- 安全性高
- 開源

**應用**：

- 智慧家居（所有裝置）

**優勢**：

- 解決了相容性問題
- 使用者不需要選擇平台

---

## 真實案例：從 RF4CE 到 BLE 的遙控器專案

回到文章開頭的故事。我為 Michael 的電視遙控器專案設計了一個基於 BLE 的方案：

**系統架構**：

```
┌─────────────────────────────────────────────────────────┐
│  BLE 遙控器系統架構                                       │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────┐                                        │
│  │  遙控器      │ (BLE Peripheral)                       │
│  │  - 按鍵      │                                        │
│  │  - 語音      │                                        │
│  │  - 觸控板    │                                        │
│  │  - 體感      │                                        │
│  └──────┬───────┘                                        │
│         │ BLE                                             │
│  ┌──────┴───────┐                                        │
│  │  電視        │ (BLE Central)                          │
│  │  - 接收命令  │                                        │
│  │  - 顯示畫面  │                                        │
│  └──────────────┘                                        │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**設計要點**：

1. **低延遲**：
   - Connection Interval = 7.5 ms
   - Slave Latency = 0
   - 延遲 < 30 ms

2. **低功耗**：
   - 使用 Sleep 模式
   - 按鍵按下時喚醒
   - 平均功耗 < 50 μA

3. **簡單配對**：
   - 使用 Just Works Pairing
   - 按一個按鈕就能配對

---

### 實作細節

#### 1. BLE 連線參數優化

**Connection Interval 優化**：

```c
// BLE 連線參數（低延遲）
void ble_conn_params_init(void) {
    ble_gap_conn_params_t conn_params = {
        .min_conn_interval = MSEC_TO_UNITS(7.5, UNIT_1_25_MS),   // 7.5 ms
        .max_conn_interval = MSEC_TO_UNITS(7.5, UNIT_1_25_MS),   // 7.5 ms
        .slave_latency = 0,                                       // 不使用 Latency
        .conn_sup_timeout = MSEC_TO_UNITS(4000, UNIT_10_MS),     // 4 s
    };

    ble_gap_ppcp_set(&conn_params);

    printf("Connection Interval: 7.5 ms\n");
    printf("Slave Latency: 0\n");
    printf("Expected latency: < 30 ms\n");
}
```

**功耗優化**：

```c
// 功耗優化：使用 Sleep 模式
void power_management_init(void) {
    // 1. 啟用 DC/DC 轉換器（降低功耗 30%）
    NRF_POWER->DCDCEN = 1;

    // 2. 設定 Sleep 模式
    sd_power_mode_set(NRF_POWER_MODE_LOWPWR);

    // 3. 設定 GPIO 為低功耗模式
    for (int i = 0; i < 32; i++) {
        nrf_gpio_cfg_default(i);
    }

    printf("Power management initialized\n");
    printf("Expected average current: < 50 μA\n");
}
```

---

#### 2. 按鍵處理

**按鍵事件處理**：

```c
// 按鍵事件處理
void button_handler(uint8_t button_id, uint8_t button_action) {
    if (button_action == BUTTON_PRESSED) {
        // 1. 喚醒裝置
        sd_nvic_SystemReset();  // 從 Sleep 喚醒

        // 2. 發送按鍵事件
        uint8_t key_code = button_id;
        ble_hids_inp_rep_send(&m_hids,
                              INPUT_REPORT_KEYS_INDEX,
                              INPUT_REPORT_KEYS_MAX_LEN,
                              &key_code);

        printf("Button %d pressed\n", button_id);
    }
}

// 按鍵初始化
void buttons_init(void) {
    // 1. 設定按鍵 GPIO
    for (int i = 0; i < BUTTON_COUNT; i++) {
        nrf_gpio_cfg_input(button_pins[i], NRF_GPIO_PIN_PULLUP);
    }

    // 2. 設定按鍵中斷
    nrf_drv_gpiote_in_config_t config = GPIOTE_CONFIG_IN_SENSE_HITOLO(true);
    config.pull = NRF_GPIO_PIN_PULLUP;

    for (int i = 0; i < BUTTON_COUNT; i++) {
        nrf_drv_gpiote_in_init(button_pins[i], &config, button_handler);
        nrf_drv_gpiote_in_event_enable(button_pins[i], true);
    }

    printf("Buttons initialized\n");
}
```

---

#### 3. 語音處理

**語音資料傳輸**：

```c
// 語音資料傳輸（使用 GATT Notification）
void voice_data_send(uint8_t *data, uint16_t length) {
    // 1. 分割資料（每次最多 20 bytes）
    uint16_t offset = 0;
    while (offset < length) {
        uint16_t chunk_len = MIN(20, length - offset);

        // 2. 發送 Notification
        ble_gatts_hvx_params_t hvx_params = {
            .handle = m_voice_char_handles.value_handle,
            .type = BLE_GATT_HVX_NOTIFICATION,
            .offset = 0,
            .p_len = &chunk_len,
            .p_data = &data[offset],
        };

        sd_ble_gatts_hvx(m_conn_handle, &hvx_params);

        offset += chunk_len;
    }

    printf("Voice data sent: %d bytes\n", length);
}

// 語音錄製
void voice_record_start(void) {
    // 1. 啟動 PDM（Pulse Density Modulation）麥克風
    nrf_drv_pdm_start();

    // 2. 設定 PDM 回調
    nrf_drv_pdm_buffer_set(m_pdm_buffer, PDM_BUFFER_SIZE);

    printf("Voice recording started\n");
}

// PDM 回調
void pdm_data_handler(uint16_t *buffer, uint32_t length) {
    // 1. 處理語音資料（降噪、壓縮）
    uint8_t compressed_data[100];
    uint16_t compressed_len = voice_compress(buffer, length, compressed_data);

    // 2. 發送語音資料
    voice_data_send(compressed_data, compressed_len);
}
```

---

### 實作結果

**測試結果**：

- ✅ **延遲**：< 30 ms（Connection Interval = 7.5 ms）
- ✅ **功耗**：平均 50 μA（使用 Sleep 模式）
- ✅ **電池續航**：> 1 年（2 顆 AAA 電池，每天使用 2 小時）
- ✅ **配對**：按一個按鈕就能配對（Just Works Pairing）
- ✅ **語音**：支援語音控制（使用 PDM 麥克風）
- ✅ **觸控板**：支援觸控板操作（使用 HID over GATT）
- ✅ **體感**：支援體感操作（使用 IMU 感測器）

**客戶反饋**：

Michael 非常滿意："這個遙控器比我預期的好！延遲低、功耗低、功能豐富。我們決定量產。"

---

### 為什麼選擇 BLE 而不是 RF4CE？

回顧這個專案，我選擇 BLE 而不是 RF4CE 的原因是：

1. **生態系統成熟**：
   - BLE 晶片廠商多（Nordic, TI, Cypress, Dialog）
   - 開發工具完善（SDK, IDE, Debugger）
   - 社群活躍，資源豐富

2. **未來發展好**：
   - BLE 會持續演進（BLE 4.2, 5.0, 5.1, 5.2...）
   - RF4CE 的發展前景不明朗

3. **應用範圍廣**：
   - BLE 不僅可以用於遙控器，還可以用於穿戴式裝置、智慧家居、IoT
   - RF4CE 只適合遙控器

4. **技術可行**：
   - 透過優化 Connection Interval，BLE 的延遲可以降到 < 30 ms
   - 透過 Sleep 模式，BLE 的功耗可以降到 < 50 μA

---

## 實務建議：如何選擇 IoT 協定？

基於多年的 IoT 開發經驗，我總結了以下的協定選擇指南：

### 選擇 BLE 的場景

✅ **穿戴式裝置**（智慧手環、智慧手錶、無線耳機）

- 需要與手機直接連線
- 電池供電，功耗敏感
- 資料速率要求高

✅ **智慧門鎖、智慧家電**

- 需要與手機直接連線
- 不需要 Mesh Network
- 安全性要求高

✅ **遙控器**（電視遙控器、遊戲手把）

- 延遲要求低（< 30 ms）
- 功耗要求低
- 生態系統成熟

---

### 選擇 Zigbee 的場景

✅ **工業 IoT**（感測器網路、設備監控）

- 需要 Mesh Network
- 通訊距離遠
- 可以使用 915 MHz 頻段避開干擾

✅ **農業 IoT**（土壤監測、灌溉控制）

- 通訊距離遠
- 需要 Mesh Network
- 環境惡劣（2.4 GHz 干擾多）

---

### 選擇 Thread 的場景

✅ **智慧家居**（新專案，需要 IP 連接）

- 需要 Mesh Network
- 需要直接連接到網際網路
- 不需要 Gateway

✅ **智慧建築**（HVAC 控制、能源管理）

- 需要 Mesh Network
- 需要 IP-based 通訊
- 安全性要求高

---

### 選擇 Matter 的場景（2020 年後）

✅ **所有智慧家居應用**

- 需要跨平台支援（Apple, Google, Amazon, Samsung）
- 需要統一標準
- 安全性要求高

---

### 決策流程圖

```
┌─────────────────────────────────────────────────────────┐
│  IoT 協定選擇決策流程圖                                   │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  應用場景？                                               │
│         │                                                 │
│         ├─ 穿戴式裝置 ─► BLE                             │
│         │                                                 │
│         ├─ 遙控器 ─► BLE (2016+) / RF4CE (2014-2016)    │
│         │                                                 │
│         ├─ 智慧家居 ─► Matter (2020+) / Thread / BLE Mesh│
│         │                                                 │
│         ├─ 工業 IoT ─► Zigbee / Thread                   │
│         │                                                 │
│         └─ 農業 IoT ─► Zigbee (915 MHz)                  │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

---

### 常見錯誤

❌ **錯誤 1：盲目追求新協定**

- 問題：因為協定新，就選擇新協定，忽略了生態系統和成熟度
- 後果：開發困難，資源少，專案延期
- 建議：選擇成熟的協定，除非新協定有明顯優勢

❌ **錯誤 2：忽略未來發展**

- 問題：只考慮當前需求，忽略了協定的未來發展
- 後果：協定被淘汰，需要重新設計
- 建議：選擇有持續演進的協定（例如：BLE, Matter）

❌ **錯誤 3：低估生態系統的重要性**

- 問題：只考慮技術指標，忽略了生態系統
- 後果：晶片供應風險、開發資源少
- 建議：選擇生態系統成熟的協定

❌ **錯誤 4：忽略相容性**

- 問題：選擇專有協定，導致相容性問題
- 後果：使用者體驗差，市場接受度低
- 建議：選擇開放標準（例如：Matter）

---

### 實務檢查清單

**協定選擇檢查清單**：

- [ ] 應用場景是什麼？（穿戴式 / 智慧家居 / 工業 IoT）
- [ ] 是否需要與手機直接連線？
- [ ] 是否需要 Mesh Network？
- [ ] 是否需要 IP-based 通訊？
- [ ] 是否需要跨平台支援？
- [ ] 延遲要求是多少？（< 30 ms, 30-100 ms, > 100 ms）
- [ ] 功耗要求是多少？（< 100 μA, 100-500 μA, > 500 μA）
- [ ] 通訊距離是多少？（< 10m, 10-50m, > 50m）
- [ ] 生態系統成熟度如何？（大 / 中 / 小）
- [ ] 未來發展如何？（持續演進 / 不明朗）

**根據檢查清單結果**：

- **BLE**：穿戴式裝置、遙控器、智慧門鎖
- **Zigbee**：工業 IoT、農業 IoT
- **Thread**：智慧家居（需要 IP 連接）
- **Matter**：所有智慧家居應用（2020 年後）

---

## 總結

IoT 協定經歷了多次演進：

**2010-2014 年：Zigbee 時代**

- Mesh Network、大量裝置、通訊距離遠
- 但需要專用 Hub，生態系統較小

**2014-2017 年：BLE 崛起**

- 手機原生支援、生態系統大
- 但不支援 Mesh Network

**2017-2020 年：BLE Mesh 與 Thread**

- BLE Mesh：結合 BLE 的普及性和 Mesh Network 的可靠性
- Thread：IP-based，可以直接連接到網際網路

**2020 年後：Matter 時代**

- 統一標準、跨平台、安全性高、開源
- 解決了相容性問題

---

### 我的建議

基於多年的 IoT 開發經驗，我的建議是：

1. **穿戴式裝置**：
   - **BLE**（一直是主流）

2. **遙控器**：
   - **2014-2016 年**：RF4CE
   - **2016 年後**：BLE

3. **智慧家居**：
   - **2014-2017 年**：Zigbee
   - **2017-2020 年**：BLE Mesh / Thread
   - **2020 年後**：Matter

4. **工業 IoT**：
   - **Zigbee**（特別是 915 MHz 頻段）
   - **Thread**（需要 IP 連接）

5. **新專案（2025 年）**：
   - **智慧家居**：Matter
   - **穿戴式裝置**：BLE
   - **工業 IoT**：Zigbee / Thread

---

### 延伸閱讀

回顧這次的協定選擇經歷，我深刻體會到：**協定選擇不僅是技術問題，更是生態系統和未來發展的問題**。

選擇協定時，要考慮：

- **技術需求**（延遲、功耗、通訊距離、Mesh Network）
- **生態系統**（晶片廠商、開發工具、社群）
- **未來發展**（協定演進、市場趨勢）
- **相容性**（跨平台支援、開放標準）

只有綜合考慮這些因素，才能做出正確的決策。

在下一篇文章中，我會介紹 **AIoT（AI + IoT）** 的演進，以及如何在 IoT 裝置上運行 AI 模型。

---

## 參考資料

- RF4CE Specification: <https://zigbeealliance.org/solution/rf4ce/>
- Thread Specification: <https://www.threadgroup.org/support#specifications>
- Matter Specification: <https://csa-iot.org/all-solutions/matter/>
- IEEE 802.15.4 Standard: <https://standards.ieee.org/standard/802_15_4-2020.html>
- 6LoWPAN: <https://datatracker.ietf.org/doc/html/rfc4944>
- CoAP: <https://datatracker.ietf.org/doc/html/rfc7252>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

您可以自由地：

- **分享** — 以任何媒介或格式重製及散布本素材
- **修改** — 重混、轉換本素材、及依本素材建立新素材

但您必須遵守以下條款：

- **姓名標示** — 您必須給予適當表彰、提供指向本授權條款的連結，以及指出（本作品的原始版本）是否已被變更。您可以任何合理的方式為前述表彰，但不得以任何方式暗示授權人為您或您的使用方式背書。
