---
author: Danny Jiang
date: 2025-12-11
---

# Bluetooth SIG 認證實戰 - 從 RF 測試到 Protocol 驗證

## 前言：2015 年 3 月的認證馬拉松

2015 年 3 月，我們的智慧手環專案進入了最關鍵的階段：**Bluetooth SIG 認證**。

那是一個週一早上 9 點，我坐在 Bluetooth SIG 認證實驗室的會議室裡，看著測試工程師 Mike 準備測試設備。桌上擺滿了各種儀器：**Spectrum Analyzer**（頻譜分析儀）、**Signal Generator**（訊號產生器）、**Anechoic Chamber**（電波暗室）的控制面板。

"Danny，我們今天要完成所有的 RF 測試，"Mike 說，"TX Power、RX Sensitivity、Frequency Accuracy、Modulation Characteristics... 一共 15 項測試。"

我點點頭，心裡卻有些緊張。這是我第一次參與 Bluetooth SIG 認證，之前只在公司內部做過 Pre-compliance 測試。雖然內部測試都通過了，但正式認證的標準更嚴格，容錯率幾乎為零。

"我們先從 TX Power 開始，"Mike 說著，將我們的智慧手環放入電波暗室，"根據 Bluetooth Core Spec 5.0，BLE 的 TX Power 應該在 -20 dBm 到 +10 dBm 之間，而且每個 Channel 的功率差異不能超過 ±6 dB。"

我看著螢幕上的波形，心裡默默祈禱。

第一個 Channel（Channel 0, 2402 MHz）的測試結果出來了：**+3.2 dBm**。

"很好，"Mike 說，"在規範範圍內。"

接下來是 Channel 19（2440 MHz）：**+3.5 dBm**。

Channel 39（2480 MHz）：**+2.8 dBm**。

我鬆了一口氣。TX Power 測試順利通過。

但接下來的 **RX Sensitivity** 測試就沒那麼順利了。

Mike 將 Signal Generator 連接到我們的裝置，開始發送微弱的 BLE 訊號。根據規範，BLE 裝置的 RX Sensitivity 應該至少達到 **-70 dBm**（也就是說，能夠接收到 -70 dBm 的微弱訊號）。

測試開始。Mike 逐漸降低訊號強度：-60 dBm、-65 dBm、-70 dBm...

當訊號強度降到 **-68 dBm** 時，我們的裝置開始出現 **Packet Error Rate (PER) > 30.8%**。

"這不行，"Mike 搖搖頭，"規範要求在 -70 dBm 時，PER 應該 ≤ 30.8%。你們的裝置在 -68 dBm 就超標了，這意味著 RX Sensitivity 只有 -68 dBm，不符合規範。"

我的心一沉。這是最不想看到的情況。

"可能是什麼問題？"我問。

"通常是 RF 前端的設計問題，"Mike 說，"可能是 LNA (Low Noise Amplifier) 的增益不夠，或是 Matching Network 沒有調好。你們需要回去檢查硬體設計。"

我立刻打電話給硬體工程師 Kevin。

"Kevin，我們的 RX Sensitivity 測試失敗了，只有 -68 dBm，"我說，"Mike 說可能是 LNA 或 Matching Network 的問題。"

"讓我看看 Layout，"Kevin 說。

10 分鐘後，Kevin 回電了。

"我找到問題了，"他說，"Matching Network 的電容值錯了。我們用的是 **2.2 pF**，但根據 nRF52 的 Reference Design，應該用 **1.8 pF**。這會影響 RF 前端的阻抗匹配，導致接收靈敏度下降。"

"那我們需要重新做板子？"我問。

"是的，"Kevin 說，"我會立刻下單做新的 PCB。大概需要 3 天。"

我看著 Mike，苦笑著說："我們需要暫停測試，3 天後再來。"

Mike 點點頭："沒問題。這種情況很常見。記得帶新的樣品來。"

---

3 天後，我們拿到了新的 PCB。Kevin 將 Matching Network 的電容從 2.2 pF 改成了 1.8 pF，並重新調整了 RF Layout。

我們再次來到認證實驗室。

這次，RX Sensitivity 測試的結果是：**-72 dBm**（PER = 28.5%）。

"完美！"Mike 說，"不僅符合規範，還有 2 dB 的餘裕。"

我終於鬆了一口氣。

接下來的測試都很順利：Frequency Accuracy（頻率準確度）、Modulation Characteristics（調變特性）、Spurious Emissions（雜散輻射）... 一項一項通過。

到了下午 5 點，所有的 RF 測試都完成了。

"恭喜，"Mike 說，"RF 測試全部通過。明天我們進行 Protocol 測試。"

第二天，我們開始了 **Protocol Conformance Testing**（協定一致性測試）。這部分使用 Bluetooth SIG 的 **PTS (Profile Tuning Suite)** 工具，測試我們的裝置是否符合 BLE 協定規範。

測試項目包括：

- **GAP (Generic Access Profile)** 測試：Advertising、Scanning、Connection
- **GATT (Generic Attribute Profile)** 測試：Service Discovery、Read/Write Characteristics
- **L2CAP** 測試：Connection Parameter Update、MTU Exchange
- **SMP (Security Manager Protocol)** 測試：Pairing、Bonding、Encryption

每一項測試都有數十個 Test Case。整個過程就像是在玩一個超級複雜的「找碴遊戲」。

最讓我印象深刻的是 **Connection Parameter Update** 測試。

PTS 發送了一個 Connection Parameter Update Request，要求將 Connection Interval 從 30 ms 改成 100 ms。我們的裝置應該回應 Connection Parameter Update Response，並更新參數。

但測試失敗了。

"你們的裝置沒有回應 Update Response，"Mike 說。

我立刻檢查程式碼。原來是我們的 L2CAP 層沒有正確處理 Connection Parameter Update Request。我們只實作了「主動發起 Update Request」的功能，卻忘了實作「回應對方的 Update Request」。

我花了 2 小時修改程式碼，重新編譯韌體，燒錄到裝置上。

再次測試，通過了。

經過兩天的馬拉松式測試，我們終於完成了所有的認證項目。

"恭喜，"Mike 說，"你們的產品通過了 Bluetooth SIG 認證。我會在一週內發送正式的認證報告和 QDID (Qualified Design ID)。"

我握著 Mike 的手，心裡充滿了成就感。

這次認證經歷讓我深刻理解了 Bluetooth SIG 認證的嚴格性和重要性。每一個細節都不能馬虎，從 RF 硬體設計到協定實作，都需要精確符合規範。

---

## Bluetooth SIG 認證概覽

### 為什麼需要 Bluetooth SIG 認證？

**Bluetooth SIG (Special Interest Group)** 是 Bluetooth 技術的管理組織，負責制定 Bluetooth 規範和認證流程。

**認證的目的**：

1. **互操作性 (Interoperability)**：
   - 確保不同廠商的 Bluetooth 裝置可以互相通訊
   - 避免「只能連自家裝置」的問題

2. **符合規範 (Compliance)**：
   - 確保產品符合 Bluetooth Core Spec 和 Profile Spec
   - 保證 RF 性能、協定實作、安全機制都符合標準

3. **品牌保護 (Brand Protection)**：
   - 只有通過認證的產品才能使用 Bluetooth® 商標
   - 保護 Bluetooth 品牌的聲譽

4. **法規要求 (Regulatory)**：
   - 許多國家要求 Bluetooth 產品必須通過認證才能上市
   - 例如：FCC (美國)、CE (歐盟)、TELEC (日本)

**認證的好處**：

- ✅ 獲得 **QDID (Qualified Design ID)**：產品的唯一識別碼
- ✅ 可以使用 **Bluetooth® 商標**
- ✅ 列入 Bluetooth SIG 的 **Qualified Products List**
- ✅ 提升產品的市場信任度

---

### 認證流程概覽

Bluetooth SIG 認證分為兩大部分：

```
┌─────────────────────────────────────────────────────────┐
│  Bluetooth SIG 認證流程                                   │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─────────────────────────────────────────────┐         │
│  │  1. RF Testing (RF 測試)                    │         │
│  │     - TX Power                              │         │
│  │     - RX Sensitivity                        │         │
│  │     - Frequency Accuracy                    │         │
│  │     - Modulation Characteristics            │         │
│  │     - Spurious Emissions                    │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                         │
│                 ▼                                         │
│  ┌─────────────────────────────────────────────┐         │
│  │  2. Protocol Testing (協定測試)             │         │
│  │     - GAP (Generic Access Profile)          │         │
│  │     - GATT (Generic Attribute Profile)      │         │
│  │     - L2CAP (Logical Link Control)          │         │
│  │     - SMP (Security Manager Protocol)       │         │
│  │     - Profile-specific tests                │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                         │
│                 ▼                                         │
│  ┌─────────────────────────────────────────────┐         │
│  │  3. Interoperability Testing (互操作性測試) │         │
│  │     - 與不同廠商的裝置測試連線              │         │
│  │     - 測試常見的使用場景                    │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                         │
│                 ▼                                         │
│  ┌─────────────────────────────────────────────┐         │
│  │  4. Documentation Review (文件審查)         │         │
│  │     - Declaration of Conformity             │         │
│  │     - Test Reports                          │         │
│  │     - Product Listing                       │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                         │
│                 ▼                                         │
│  ┌─────────────────────────────────────────────┐         │
│  │  5. QDID Issued (獲得認證)                  │         │
│  │     - Qualified Design ID                   │         │
│  │     - Bluetooth® Trademark License          │         │
│  └─────────────────────────────────────────────┘         │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**認證時程**：

- **Pre-compliance 測試**：1-2 週（公司內部）
- **正式認證測試**：3-5 天（認證實驗室）
- **文件審查**：1-2 週（Bluetooth SIG）
- **總時程**：4-6 週

**認證費用**：

- **Bluetooth SIG 會員費**：
  - Adopter Member：$7,500/年（小公司）
  - Associate Member：$35,000/年（大公司）
- **認證測試費用**：$4,000-$8,000（依產品複雜度）
- **實驗室測試費用**：$2,000-$5,000/天

---

## RF 測試 (RF Testing)

RF 測試是認證的第一關，也是最容易失敗的部分。測試項目包括：

### 1. TX Power（發射功率）

**測試目的**：確保裝置的發射功率符合規範

**測試方法**：

1. 將裝置放入 **Anechoic Chamber**（電波暗室）
2. 使用 **Spectrum Analyzer** 測量每個 Channel 的發射功率
3. 測試所有 40 個 BLE Channels（3 個 Advertising Channels + 37 個 Data Channels）

**規範要求**：

- **BLE 1M PHY**：-20 dBm 到 +10 dBm
- **BLE 2M PHY**：-20 dBm 到 +10 dBm
- **BLE Coded PHY (Long Range)**：-20 dBm 到 +10 dBm
- **功率差異**：每個 Channel 的功率差異不能超過 ±6 dB

**測試範例**：

```
Channel 0 (2402 MHz):  +3.2 dBm  ✅
Channel 1 (2404 MHz):  +3.4 dBm  ✅
Channel 2 (2406 MHz):  +3.1 dBm  ✅
...
Channel 37 (2402 MHz): +3.3 dBm  ✅ (Advertising Channel)
Channel 38 (2426 MHz): +3.5 dBm  ✅ (Advertising Channel)
Channel 39 (2480 MHz): +2.9 dBm  ✅ (Advertising Channel)

最大功率差異: 3.5 - 2.9 = 0.6 dB  ✅ (< 6 dB)
```

**常見問題**：

- ❌ **功率過高**：超過 +10 dBm，可能違反法規
- ❌ **功率過低**：低於 -20 dBm，影響通訊距離
- ❌ **功率不穩定**：不同 Channel 的功率差異過大

**解決方案**：

```c
// 調整 TX Power（以 nRF52 為例）
void set_tx_power(int8_t power_dbm) {
    // nRF52 支援的 TX Power: -40, -20, -16, -12, -8, -4, 0, +2, +3, +4, +5, +6, +7, +8 dBm

    if (power_dbm > 8) power_dbm = 8;
    if (power_dbm < -40) power_dbm = -40;

    // 設定 TX Power
    NRF_RADIO->TXPOWER = (power_dbm << RADIO_TXPOWER_TXPOWER_Pos);
}

// 使用範例
void ble_init(void) {
    // 設定 TX Power 為 +4 dBm（符合大部分應用需求）
    set_tx_power(4);
}
```

---

### 2. RX Sensitivity（接收靈敏度）

**測試目的**：確保裝置能夠接收微弱的訊號

**測試方法**：

1. 使用 **Signal Generator** 產生 BLE 訊號
2. 逐漸降低訊號強度，測量 **Packet Error Rate (PER)**
3. 找出 PER = 30.8% 時的訊號強度（這就是 RX Sensitivity）

**規範要求**：

- **BLE 1M PHY**：≤ -70 dBm（PER ≤ 30.8%）
- **BLE 2M PHY**：≤ -67 dBm（PER ≤ 30.8%）
- **BLE Coded PHY S=8**：≤ -94 dBm（PER ≤ 30.8%）
- **BLE Coded PHY S=2**：≤ -91 dBm（PER ≤ 30.8%）

**測試範例**：

```
訊號強度: -60 dBm, PER: 0.0%   ✅
訊號強度: -65 dBm, PER: 5.2%   ✅
訊號強度: -70 dBm, PER: 28.5%  ✅ (< 30.8%)
訊號強度: -72 dBm, PER: 45.3%  ❌ (> 30.8%)

RX Sensitivity: -70 dBm  ✅ (符合規範)
```

**常見問題**：

- ❌ **RX Sensitivity 不足**：只能接收 -65 dBm 的訊號，不符合 -70 dBm 的規範
- ❌ **PER 過高**：在 -70 dBm 時，PER > 30.8%

**原因分析**：

1. **LNA (Low Noise Amplifier) 增益不夠**
2. **Matching Network 阻抗不匹配**
3. **RF Layout 設計不良**（例如：Ground plane 不完整）
4. **干擾源**（例如：DC-DC Converter 的雜訊）

**解決方案**：

```c
// 檢查 Matching Network 設計
// 以 nRF52 為例，參考 Nordic 的 Reference Design

/*
  Matching Network 電路：

  ANT
   │
   ├─── C1 (1.8 pF) ─── L1 (3.9 nH) ─── RF_IN (nRF52)
   │
   └─── C2 (1.0 pF) ─── GND

  阻抗匹配目標：50 Ω (Antenna) → 50 Ω (nRF52 RF_IN)
*/

// 軟體優化：啟用 DC-DC Converter（減少雜訊）
void enable_dcdc(void) {
    NRF_POWER->DCDCEN = POWER_DCDCEN_DCDCEN_Enabled;
}

// 軟體優化：調整 LNA 增益
void optimize_rx_sensitivity(void) {
    // nRF52 的 LNA 增益可以透過 RADIO->OVERRIDE 調整
    // 但通常使用預設值即可

    // 確保 DC-DC Converter 已啟用（減少雜訊）
    enable_dcdc();
}
```

**硬體優化建議**：

1. **使用正確的 Matching Network 元件值**：
   - 參考晶片廠商的 Reference Design
   - 使用 Smith Chart 工具計算阻抗匹配

2. **優化 RF Layout**：
   - RF trace 使用 50 Ω 阻抗控制
   - Ground plane 完整，避免切割
   - 遠離高速數位訊號（如 SPI、UART）

3. **減少干擾源**：
   - DC-DC Converter 使用 Spread Spectrum 模式
   - 在電源線上加入 Ferrite Bead 和去耦電容

---

### 3. Frequency Accuracy（頻率準確度）

**測試目的**：確保裝置的頻率準確，避免頻率偏移

**測試方法**：

1. 使用 **Frequency Counter** 測量裝置的實際頻率
2. 與標稱頻率比較，計算頻率偏移

**規範要求**：

- **Initial Frequency Tolerance**：±20 ppm（初始頻率容差）
- **Drift Rate**：±50 ppm（頻率漂移率）

**測試範例**：

```
標稱頻率: 2402 MHz (Channel 0)
實際頻率: 2402.012 MHz

頻率偏移: (2402.012 - 2402) / 2402 × 10^6 = 5.0 ppm  ✅ (< 20 ppm)
```

**常見問題**：

- ❌ **頻率偏移過大**：超過 ±20 ppm
- ❌ **溫度漂移**：溫度變化導致頻率偏移

**原因分析**：

1. **Crystal Oscillator 精度不夠**：使用低精度的晶振（例如：±50 ppm）
2. **Load Capacitor 值不正確**：影響晶振的頻率
3. **溫度補償不足**：溫度變化導致頻率漂移

**解決方案**：

```c
// 使用高精度晶振
// 推薦：±10 ppm 或更好的 TCXO (Temperature Compensated Crystal Oscillator)

// 校準晶振頻率（以 nRF52 為例）
void calibrate_crystal(void) {
    // nRF52 支援軟體校準晶振頻率
    // 透過調整 CLOCK->LFCLKSRC 的 CAL 欄位

    // 範例：調整 +5 ppm
    NRF_CLOCK->LFCLKSRC = (CLOCK_LFCLKSRC_SRC_Xtal << CLOCK_LFCLKSRC_SRC_Pos) |
                          (5 << CLOCK_LFCLKSRC_CAL_Pos);  // +5 ppm
}

// 溫度補償（如果使用 Temperature Sensor）
void temperature_compensation(void) {
    int32_t temp = read_temperature();  // 讀取溫度（單位：°C）

    // 根據溫度調整頻率
    // 假設晶振的溫度係數為 -0.04 ppm/°C²
    int32_t freq_offset = calculate_freq_offset(temp);

    // 調整晶振頻率
    NRF_CLOCK->LFCLKSRC = (CLOCK_LFCLKSRC_SRC_Xtal << CLOCK_LFCLKSRC_SRC_Pos) |
                          (freq_offset << CLOCK_LFCLKSRC_CAL_Pos);
}
```

---

### 4. Modulation Characteristics（調變特性）

**測試目的**：確保裝置的調變符合 GFSK (Gaussian Frequency Shift Keying) 規範

**測試方法**：

1. 使用 **Vector Signal Analyzer** 分析調變波形
2. 測量 **Frequency Deviation**（頻率偏移）和 **Modulation Index**（調變指數）

**規範要求**：

- **Frequency Deviation**：185 kHz ± 75 kHz（對於 1M PHY）
- **Modulation Index**：0.45 到 0.55

**測試範例**：

```
Frequency Deviation: 250 kHz  ✅ (185 ± 75 kHz)
Modulation Index: 0.50  ✅ (0.45 到 0.55)
```

**常見問題**：

- ❌ **Frequency Deviation 過大或過小**
- ❌ **Modulation Index 不符合規範**

**解決方案**：

```c
// 調整 Frequency Deviation（以 nRF52 為例）
void set_frequency_deviation(void) {
    // nRF52 的 Frequency Deviation 由硬體自動控制
    // 通常不需要調整

    // 如果需要微調，可以透過 RADIO->MODECNF0 調整
    NRF_RADIO->MODECNF0 = (RADIO_MODECNF0_RU_Default << RADIO_MODECNF0_RU_Pos) |
                          (RADIO_MODECNF0_DTX_B1 << RADIO_MODECNF0_DTX_Pos);
}
```

---

### 5. Spurious Emissions（雜散輻射）

**測試目的**：確保裝置不會產生過多的雜散輻射，干擾其他頻段

**測試方法**：

1. 使用 **Spectrum Analyzer** 掃描 30 MHz 到 12.75 GHz 的頻段
2. 測量所有頻段的輻射功率

**規範要求**：

- **In-band Spurious Emissions**（2400-2483.5 MHz）：< -20 dBm
- **Out-of-band Spurious Emissions**（其他頻段）：< -36 dBm

**測試範例**：

```
2400-2483.5 MHz (In-band):  -25 dBm  ✅ (< -20 dBm)
900 MHz (Out-of-band):      -42 dBm  ✅ (< -36 dBm)
1800 MHz (Out-of-band):     -40 dBm  ✅ (< -36 dBm)
```

**常見問題**：

- ❌ **Harmonic Emissions**（諧波輻射）：2.4 GHz 的 2 倍頻（4.8 GHz）、3 倍頻（7.2 GHz）
- ❌ **DC-DC Converter 雜訊**：開關頻率的諧波

**解決方案**：

1. **使用 RF Filter**：在 Antenna 和 RF_OUT 之間加入 Band-pass Filter
2. **優化 PCB Layout**：減少 RF 訊號洩漏
3. **使用 Spread Spectrum DC-DC Converter**：減少開關頻率的諧波

---

## Protocol 測試 (Protocol Conformance Testing)

Protocol 測試使用 Bluetooth SIG 的 **PTS (Profile Tuning Suite)** 工具，測試裝置是否符合 BLE 協定規範。

### 測試環境設定

```
┌─────────────────────────────────────────────────────────┐
│  PTS 測試環境                                             │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────┐         USB/UART        ┌───────────┐ │
│  │  PTS PC      │◄──────────────────────►│  DUT      │ │
│  │  (Tester)    │                         │  (Device  │ │
│  │              │                         │   Under   │ │
│  │              │                         │   Test)   │ │
│  └──────────────┘                         └───────────┘ │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**測試步驟**：

1. **連接 DUT 到 PTS PC**：
   - 使用 USB Dongle 或 UART 連接
   - 配置 HCI Transport

2. **選擇測試項目**：
   - GAP、GATT、L2CAP、SMP 等

3. **執行測試**：
   - PTS 自動發送測試命令
   - DUT 回應測試命令
   - PTS 檢查回應是否符合規範

4. **查看測試結果**：
   - Pass / Fail / Inconclusive

---

### GAP (Generic Access Profile) 測試

**測試項目**：

1. **Advertising**：
   - 測試 Advertising 封包格式
   - 測試 Advertising Interval
   - 測試 Scan Response

2. **Scanning**：
   - 測試 Scan Request/Response
   - 測試 Active/Passive Scanning

3. **Connection**：
   - 測試 Connection Request/Response
   - 測試 Connection Parameter Update

**測試範例**：

```
Test Case: GAP/DISC/GENP/BV-01-C (General Discoverable Mode)

步驟：
1. PTS 發送 Scan Request
2. DUT 回應 Scan Response（包含 Device Name）
3. PTS 檢查 Scan Response 格式

預期結果：
- Scan Response 包含 Complete Local Name
- AD Type = 0x09 (Complete Local Name)
- Device Name = "Smart Band"

實際結果：✅ Pass
```

**常見失敗原因**：

- ❌ **Advertising Interval 不符合規範**：
  - 規範要求：20 ms 到 10.24 s
  - 常見錯誤：設定為 10 ms（太短）

- ❌ **Scan Response 格式錯誤**：
  - 缺少 Complete Local Name
  - AD Type 錯誤

**解決方案**：

```c
// 設定 Advertising Data
void set_advertising_data(void) {
    uint8_t adv_data[] = {
        // Flags
        0x02,  // Length
        0x01,  // AD Type: Flags
        0x06,  // LE General Discoverable Mode + BR/EDR Not Supported

        // Complete Local Name
        0x0B,  // Length (11 bytes)
        0x09,  // AD Type: Complete Local Name
        'S', 'm', 'a', 'r', 't', ' ', 'B', 'a', 'n', 'd',  // "Smart Band"
    };

    // 設定 Advertising Data
    ble_gap_adv_data_set(adv_data, sizeof(adv_data));
}

// 設定 Advertising Interval
void set_advertising_interval(void) {
    // Advertising Interval: 100 ms (符合規範)
    uint32_t interval_ms = 100;
    uint32_t interval_units = (interval_ms * 1000) / 625;  // 轉換為 0.625 ms 單位

    ble_gap_adv_params_t adv_params = {
        .interval = interval_units,  // 100 ms
        .timeout = 0,  // 無限制
        .type = BLE_GAP_ADV_TYPE_CONNECTABLE_SCANNABLE_UNDIRECTED,
    };

    ble_gap_adv_start(&adv_params);
}
```

---

### GATT (Generic Attribute Profile) 測試

**測試項目**：

1. **Service Discovery**：
   - 測試 Primary Service Discovery
   - 測試 Characteristic Discovery

2. **Read/Write Characteristics**：
   - 測試 Read Characteristic Value
   - 測試 Write Characteristic Value
   - 測試 Write Without Response

3. **Notifications/Indications**：
   - 測試 Characteristic Notifications
   - 測試 Characteristic Indications

**測試範例**：

```
Test Case: GATT/SR/GAR/BV-01-C (Read Characteristic Value)

步驟：
1. PTS 連線到 DUT
2. PTS 發送 Read Request（讀取 Heart Rate Measurement Characteristic）
3. DUT 回應 Read Response（包含心率資料）
4. PTS 檢查 Read Response 格式

預期結果：
- Read Response 包含 Heart Rate Measurement 資料
- 格式符合 Heart Rate Service Specification

實際結果：✅ Pass
```

**常見失敗原因**：

- ❌ **Characteristic UUID 錯誤**：
  - 使用自定義 UUID 而非標準 UUID

- ❌ **Characteristic Properties 錯誤**：
  - 例如：Characteristic 宣告為 Read，但實際不支援 Read

- ❌ **ATT MTU 處理錯誤**：
  - 資料長度超過 ATT MTU，但沒有正確分割

**解決方案**：

```c
// 定義 Heart Rate Service
#define BLE_UUID_HEART_RATE_SERVICE              0x180D
#define BLE_UUID_HEART_RATE_MEASUREMENT_CHAR     0x2A37

// 註冊 Heart Rate Service
void register_heart_rate_service(void) {
    // 1. 建立 Service
    ble_uuid_t service_uuid = {
        .uuid = BLE_UUID_HEART_RATE_SERVICE,
        .type = BLE_UUID_TYPE_BLE,  // 標準 UUID
    };

    uint16_t service_handle;
    ble_gatts_service_add(BLE_GATTS_SRVC_TYPE_PRIMARY, &service_uuid, &service_handle);

    // 2. 建立 Characteristic
    ble_uuid_t char_uuid = {
        .uuid = BLE_UUID_HEART_RATE_MEASUREMENT_CHAR,
        .type = BLE_UUID_TYPE_BLE,
    };

    ble_gatts_char_md_t char_md = {
        .char_props.read = 0,  // 不支援 Read
        .char_props.notify = 1,  // 支援 Notify
    };

    ble_gatts_attr_t attr = {
        .p_uuid = &char_uuid,
        .p_attr_md = &attr_md,
        .init_len = sizeof(uint8_t),
        .max_len = 20,  // 最大 20 bytes
    };

    ble_gatts_char_handles_t char_handles;
    ble_gatts_characteristic_add(service_handle, &char_md, &attr, &char_handles);
}

// 發送 Heart Rate Notification
void send_heart_rate_notification(uint8_t heart_rate) {
    uint8_t data[2] = {
        0x00,  // Flags: Heart Rate Value Format = UINT8
        heart_rate,  // Heart Rate Measurement Value
    };

    ble_gatts_hvx_params_t hvx_params = {
        .handle = char_handles.value_handle,
        .type = BLE_GATT_HVX_NOTIFICATION,
        .offset = 0,
        .p_len = sizeof(data),
        .p_data = data,
    };

    ble_gatts_hvx(conn_handle, &hvx_params);
}
```

---

### L2CAP 測試

**測試項目**：

1. **Connection Parameter Update**：
   - 測試 Connection Parameter Update Request/Response

2. **MTU Exchange**：
   - 測試 MTU Exchange Request/Response

**測試範例**：

```
Test Case: L2CAP/COS/CED/BV-01-C (MTU Exchange)

步驟：
1. PTS 連線到 DUT
2. PTS 發送 MTU Exchange Request（MTU = 247）
3. DUT 回應 MTU Exchange Response（MTU = 247）
4. PTS 檢查 MTU Exchange Response

預期結果：
- DUT 回應 MTU Exchange Response
- MTU = 247（或更小）

實際結果：✅ Pass
```

**常見失敗原因**：

- ❌ **沒有回應 MTU Exchange Request**
- ❌ **MTU 值錯誤**（例如：回應 MTU > 請求 MTU）

**解決方案**：

```c
// 處理 MTU Exchange Request
void on_mtu_exchange_request(uint16_t conn_handle, uint16_t client_mtu) {
    // 選擇較小的 MTU
    uint16_t server_mtu = 247;  // 我們支援的最大 MTU
    uint16_t negotiated_mtu = (client_mtu < server_mtu) ? client_mtu : server_mtu;

    // 回應 MTU Exchange Response
    ble_gattc_exchange_mtu_response(conn_handle, negotiated_mtu);

    // 更新連線的 MTU
    update_connection_mtu(conn_handle, negotiated_mtu);
}
```

---

### SMP (Security Manager Protocol) 測試

**測試項目**：

1. **Pairing**：
   - 測試 Just Works Pairing
   - 測試 Passkey Entry Pairing
   - 測試 Numeric Comparison Pairing

2. **Bonding**：
   - 測試 Bonding Information 儲存
   - 測試 Reconnection with Bonding

3. **Encryption**：
   - 測試 Link Encryption
   - 測試 Re-encryption

**測試範例**：

```
Test Case: SM/MAS/PROT/BV-01-C (Just Works Pairing)

步驟：
1. PTS 連線到 DUT
2. PTS 發送 Pairing Request
3. DUT 回應 Pairing Response
4. 執行 Just Works Pairing
5. PTS 檢查 Pairing 是否成功

預期結果：
- Pairing 成功
- Link 已加密

實際結果：✅ Pass
```

**常見失敗原因**：

- ❌ **Pairing 失敗**：
  - IO Capabilities 不匹配
  - Pairing Method 選擇錯誤

- ❌ **Bonding Information 沒有儲存**：
  - 重新連線時需要重新 Pairing

**解決方案**：

```c
// 設定 Security Parameters
void set_security_params(void) {
    ble_gap_sec_params_t sec_params = {
        .bond = 1,  // 啟用 Bonding
        .mitm = 0,  // 不需要 MITM Protection（Just Works）
        .lesc = 0,  // 不使用 LE Secure Connections
        .keypress = 0,
        .io_caps = BLE_GAP_IO_CAPS_NONE,  // No Input, No Output
        .oob = 0,
        .min_key_size = 7,
        .max_key_size = 16,
    };

    ble_gap_sec_params_set(&sec_params);
}

// 儲存 Bonding Information
void store_bonding_info(uint16_t conn_handle) {
    // 取得 Bonding Information
    ble_gap_sec_keyset_t keyset;
    ble_gap_sec_keys_get(conn_handle, &keyset);

    // 儲存到 Flash
    flash_write(BONDING_INFO_ADDR, &keyset, sizeof(keyset));
}

// 載入 Bonding Information
void load_bonding_info(uint16_t conn_handle) {
    // 從 Flash 讀取
    ble_gap_sec_keyset_t keyset;
    flash_read(BONDING_INFO_ADDR, &keyset, sizeof(keyset));

    // 設定 Bonding Information
    ble_gap_sec_keys_set(conn_handle, &keyset);
}
```

---

## Interoperability 測試

Interoperability 測試確保裝置可以與不同廠商的裝置互相通訊。

**測試方法**：

1. **與不同手機測試**：
   - iPhone (iOS)
   - Android 手機（Samsung, Google Pixel）
   - Windows PC

2. **測試常見場景**：
   - Advertising 和 Scanning
   - Connection 和 Disconnection
   - Service Discovery
   - Read/Write Characteristics
   - Notifications
   - Pairing 和 Bonding

**測試範例**：

```
測試裝置: iPhone 12 (iOS 14.5)

測試項目:
1. Advertising: ✅ iPhone 可以掃描到裝置
2. Connection: ✅ iPhone 可以連線到裝置
3. Service Discovery: ✅ iPhone 可以發現 Heart Rate Service
4. Read Characteristic: ✅ iPhone 可以讀取 Device Name
5. Notification: ✅ iPhone 可以接收 Heart Rate Notification
6. Pairing: ✅ iPhone 可以與裝置 Pairing
7. Reconnection: ✅ iPhone 可以重新連線（使用 Bonding）

結果: ✅ 全部通過
```

**常見問題**：

- ❌ **iOS 無法連線**：
  - 可能是 Advertising Data 格式錯誤
  - 可能是 Connection Parameters 不符合 Apple 的要求

- ❌ **Android 連線不穩定**：
  - 可能是 Connection Interval 太長
  - 可能是 Supervision Timeout 太短

**解決方案**：

```c
// 符合 Apple 的 Connection Parameters 要求
void set_apple_compatible_conn_params(void) {
    ble_gap_conn_params_t conn_params = {
        .min_conn_interval = MSEC_TO_UNITS(15, UNIT_1_25_MS),  // 15 ms
        .max_conn_interval = MSEC_TO_UNITS(30, UNIT_1_25_MS),  // 30 ms
        .slave_latency = 0,
        .conn_sup_timeout = MSEC_TO_UNITS(4000, UNIT_10_MS),  // 4 s
    };

    ble_gap_ppcp_set(&conn_params);
}
```

---

## 真實案例：認證過程中的常見問題

### 案例 1：RX Sensitivity 測試失敗

**問題**：RX Sensitivity 只有 -68 dBm，不符合 -70 dBm 的規範

**原因**：Matching Network 的電容值錯誤（2.2 pF → 應該是 1.8 pF）

**解決方案**：
1. 修改 PCB Layout，將電容從 2.2 pF 改成 1.8 pF
2. 重新測試，RX Sensitivity 達到 -72 dBm

**學到的經驗**：
- 嚴格遵循晶片廠商的 Reference Design
- 使用 Smith Chart 工具驗證阻抗匹配
- Pre-compliance 測試要測量 RX Sensitivity

---

### 案例 2：Connection Parameter Update 測試失敗

**問題**：PTS 發送 Connection Parameter Update Request，但裝置沒有回應

**原因**：L2CAP 層沒有實作「回應對方的 Update Request」功能

**解決方案**：
1. 修改 L2CAP 層程式碼，實作 Connection Parameter Update Response
2. 重新測試，通過

**學到的經驗**：
- 不只要實作「主動發起」的功能，也要實作「被動回應」的功能
- 仔細閱讀 Bluetooth Core Spec，確保所有必要的功能都有實作

---

### 案例 3：iOS 無法連線

**問題**：iPhone 可以掃描到裝置，但無法連線

**原因**：Connection Parameters 不符合 Apple 的要求（Connection Interval > 30 ms）

**解決方案**：
1. 修改 Connection Parameters，將 max_conn_interval 從 100 ms 改成 30 ms
2. 重新測試，iPhone 可以正常連線

**學到的經驗**：
- 不同平台（iOS、Android、Windows）對 Connection Parameters 有不同的要求
- 參考平台廠商的開發文件（例如：Apple Accessory Design Guidelines）

---

## 實務建議

### 認證前的準備

✅ **Pre-compliance 測試**：
- 在公司內部進行 Pre-compliance 測試
- 使用與認證實驗室相同的測試設備（如果可能）
- 確保所有測試項目都通過

✅ **文件準備**：
- Declaration of Conformity
- Test Plan
- Product Specification
- User Manual

✅ **樣品準備**：
- 準備 3-5 個樣品（以防測試過程中損壞）
- 確保樣品與量產版本一致

---

### 認證過程中的注意事項

✅ **與測試工程師溝通**：
- 清楚說明產品的功能和特性
- 詢問測試失敗的原因
- 請求測試工程師的建議

✅ **記錄測試結果**：
- 記錄每一項測試的結果
- 拍照記錄測試設備的設定
- 保存測試報告

✅ **快速修復問題**：
- 如果測試失敗，立刻分析原因
- 準備備用樣品（如果需要修改硬體）
- 與團隊溝通，快速修復問題

---

### 認證後的維護

✅ **保存認證文件**：
- QDID
- Test Reports
- Declaration of Conformity

✅ **產品變更管理**：
- 如果產品有重大變更（例如：更換晶片、修改 RF 設計），需要重新認證
- 小幅變更（例如：韌體更新）通常不需要重新認證

✅ **年度審查**：
- Bluetooth SIG 會員需要每年續費
- 確保產品仍然符合最新的規範

---

## 總結

Bluetooth SIG 認證是一個嚴格但必要的過程。通過認證不僅能確保產品的品質和互操作性，還能提升產品的市場信任度。

### 核心要點

**RF 測試**：
- ✅ TX Power：確保發射功率符合規範（-20 到 +10 dBm）
- ✅ RX Sensitivity：確保接收靈敏度達到 -70 dBm
- ✅ Frequency Accuracy：確保頻率偏移 < ±20 ppm
- ✅ Modulation Characteristics：確保調變符合 GFSK 規範
- ✅ Spurious Emissions：確保雜散輻射符合規範

**Protocol 測試**：
- ✅ GAP：Advertising、Scanning、Connection
- ✅ GATT：Service Discovery、Read/Write、Notifications
- ✅ L2CAP：Connection Parameter Update、MTU Exchange
- ✅ SMP：Pairing、Bonding、Encryption

**Interoperability 測試**：
- ✅ 與不同平台測試（iOS、Android、Windows）
- ✅ 測試常見使用場景

### 實戰經驗

從這次認證經歷中，我學到了：

1. **Pre-compliance 測試很重要**：可以提前發現問題，避免正式認證失敗
2. **硬體設計要嚴格遵循 Reference Design**：Matching Network、RF Layout 都不能馬虎
3. **協定實作要完整**：不只要實作「主動發起」的功能，也要實作「被動回應」的功能
4. **與測試工程師溝通**：他們的經驗可以幫助你快速找到問題

下一篇文章，我們將探討 **Zigbee vs Bluetooth**，比較兩種 IoT 協定的優缺點，以及如何選擇適合的協定。

---

## 參考資料

- Bluetooth Core Specification 5.0: <https://www.bluetooth.com/specifications/bluetooth-core-specification/>
- Bluetooth SIG Qualification Process: <https://www.bluetooth.com/develop-with-bluetooth/qualification-listing/>
- Apple Accessory Design Guidelines: <https://developer.apple.com/accessories/>
- Nordic nRF52 Reference Design: <https://www.nordicsemi.com/Products/nRF52832>

---

**版權聲明**

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。歡迎分享、改作，但請註明出處。
