# Bluetooth PHY & RF - 從 2.4 GHz 到空中封包

**作者**: Danny Jiang  
**日期**: 2025-12-11  

---

## 前言：第一次看到頻譜分析儀

2014 年 9 月，我第一次走進 RF 實驗室。

「這是頻譜分析儀，」RF 工程師指著一台比電視還大的儀器說，「我們用它來測量 Bluetooth 晶片的 RF 性能。」

螢幕上顯示著一條波動的曲線，橫軸是頻率（2.4 GHz 到 2.5 GHz），縱軸是功率（dBm）。

「這條曲線代表什麼？」我問。

「這是我們的 BLE 晶片發送封包時的頻譜，」他說，「你看這裡，2.402 GHz、2.426 GHz、2.480 GHz，這三個峰值就是 BLE 的 Advertising Channels。」

「為什麼只有三個峰值？」我問，「BLE 不是有 40 個頻道嗎？」

「BLE 有 40 個頻道，」他解釋，「但 Advertising 只使用 3 個頻道（37、38、39），這樣可以避開 WiFi 的主要頻道。其他 37 個頻道用於 Data Connection。」

「那這個峰值的高度代表什麼？」

「這是 TX Power（發射功率），」他說，「我們的規格要求是 0 dBm（1 mW），但實際測量只有 -3 dBm（0.5 mW）。這不符合規格。」

「為什麼會這樣？」

「可能是 PA（Power Amplifier）的增益不夠，」他說，「也可能是 Matching Network 沒有調好。我們需要調校。」

接下來的兩週，我和 RF 工程師一起調校 TX Power。我們調整了 PA 的偏壓、修改了 Matching Network 的電容值、優化了 PCB Layout。

最終，TX Power 從 -3 dBm 提升到 +1 dBm，符合規格要求。

但這只是開始。接下來，我們還需要測試 RX Sensitivity（接收靈敏度）、Frequency Accuracy（頻率準確度）、Modulation Characteristics（調變特性）...

這次經歷讓我深刻理解了一個道理：**Bluetooth 不只是軟體協定，更是硬體與 RF 的結合。理解 PHY 和 RF，才能真正掌握 Bluetooth 的全貌**。

---

## 2.4 GHz ISM 頻段：擁擠的無線電波世界

### 為什麼是 2.4 GHz？

Bluetooth 使用 **2.4 GHz ISM（Industrial, Scientific, Medical）頻段**，範圍是 2.400 GHz 到 2.4835 GHz。

**為什麼選擇 2.4 GHz？**

1. **全球免授權**：
   - 2.4 GHz ISM 頻段在全球大部分國家都是免授權的
   - 不需要向政府申請頻譜使用許可
   - 降低了產品的認證成本和時間

2. **天線尺寸適中**：
   - 波長 λ = c / f = 3×10⁸ / 2.4×10⁹ = 12.5 cm
   - 1/4 波長天線 = 3.1 cm（適合嵌入式裝置）
   - 1/2 波長天線 = 6.25 cm（適合手機、手錶）

3. **穿透力適中**：
   - 比 5 GHz 更好的穿牆能力
   - 比 900 MHz 更小的天線尺寸
   - 適合室內應用（10-100 公尺範圍）

**但 2.4 GHz 也有缺點**：

- **擁擠**：WiFi、Bluetooth、Zigbee、微波爐都使用 2.4 GHz
- **干擾**：多個裝置同時使用會互相干擾
- **頻譜效率低**：需要跳頻機制來避免干擾

### BLE 的頻道分配

BLE 將 2.4 GHz 頻段分為 **40 個頻道**，每個頻道寬度為 **2 MHz**：

```
┌─────────────────────────────────────────────────────────────────┐
│  2.4 GHz ISM Band (2.400 - 2.4835 GHz)                          │
├─────────────────────────────────────────────────────────────────┤
│  BLE Channels (40 channels, 2 MHz each)                         │
│                                                                  │
│  Advertising Channels (3 channels):                             │
│  ├─ Channel 37: 2.402 GHz                                       │
│  ├─ Channel 38: 2.426 GHz                                       │
│  └─ Channel 39: 2.480 GHz                                       │
│                                                                  │
│  Data Channels (37 channels):                                   │
│  ├─ Channel 0:  2.404 GHz                                       │
│  ├─ Channel 1:  2.406 GHz                                       │
│  ├─ ...                                                          │
│  └─ Channel 36: 2.478 GHz                                       │
└─────────────────────────────────────────────────────────────────┘
```

**為什麼 Advertising Channels 選擇 37、38、39？**

這三個頻道被精心選擇，以**避開 WiFi 的主要頻道**：

```
WiFi Channel 1:  2.412 GHz (中心頻率)
WiFi Channel 6:  2.437 GHz (中心頻率)
WiFi Channel 11: 2.462 GHz (中心頻率)

BLE Channel 37:  2.402 GHz ← 避開 WiFi Channel 1
BLE Channel 38:  2.426 GHz ← 在 WiFi Channel 1 和 6 之間
BLE Channel 39:  2.480 GHz ← 避開 WiFi Channel 11
```

這樣設計的好處：

- **降低干擾**：Advertising 封包不會被 WiFi 干擾
- **提高可靠性**：即使 WiFi 很忙碌，Advertising 仍然可以正常工作
- **快速連線**：Advertising 是連線的第一步，必須可靠

---

## GFSK 調變：BLE 的 1 Mbps PHY

### 什麼是 GFSK？

BLE 使用 **GFSK（Gaussian Frequency Shift Keying）** 調變，資料速率為 **1 Mbps**。

**GFSK 的工作原理**：

- **Frequency Shift Keying (FSK)**：用頻率的變化來表示 0 和 1
  - 0 → 低頻（-250 kHz）
  - 1 → 高頻（+250 kHz）

- **Gaussian Filter**：平滑頻率變化，減少頻譜擴散
  - 降低對相鄰頻道的干擾
  - 符合 FCC/ETSI 的頻譜遮罩要求

**GFSK 調變示意圖**：

```
Data:     0   1   1   0   1   0   0   1
          │   │   │   │   │   │   │   │
Frequency:│   ┌───┬───┐   ┌───┐   │   ┌───
          │   │   │   │   │   │   │   │
          └───┘   │   └───┘   └───┘   │
          -250kHz     +250kHz
```

**為什麼選擇 1 Mbps？**

1. **功耗低**：
   - 1 Mbps 的調變器比 2 Mbps 或 3 Mbps 更簡單
   - 更簡單的電路 = 更低的功耗

2. **範圍遠**：
   - 較低的資料速率 = 較高的接收靈敏度
   - BLE 4.0 的 RX Sensitivity 要求：-70 dBm @ 1 Mbps

3. **抗干擾**：
   - 較低的資料速率 = 較寬的符號時間
   - 較寬的符號時間 = 較好的抗干擾能力

**BLE 5.0 的新 PHY**：

BLE 5.0 引入了兩種新的 PHY：

- **2 Mbps PHY**：速度提升 2 倍，範圍減少
- **Coded PHY (125 kbps / 500 kbps)**：範圍提升 4 倍，速度降低

---

## Frequency Hopping：跳頻機制

### 為什麼需要跳頻？

在 2.4 GHz 頻段，有很多裝置同時工作：

- WiFi：持續佔用某些頻道
- 微波爐：在 2.45 GHz 附近產生強烈干擾
- 其他 Bluetooth 裝置：也在使用相同頻段

如果 BLE 一直使用同一個頻道，很容易被干擾。

**解決方案：Frequency Hopping（跳頻）**

BLE 使用 **Adaptive Frequency Hopping (AFH)**：

- 每個 Connection Event 使用不同的頻道
- 如果某個頻道被干擾，就跳過它
- 動態調整跳頻序列，避開干擾頻道

### 跳頻序列的產生

BLE 使用一個偽隨機序列來決定下一個頻道：

```c
// 跳頻演算法（簡化版）
uint8_t hop_increment = 5 + (access_address & 0x1F);  // 5-16
uint8_t next_channel = (current_channel + hop_increment) % 37;

// 如果 next_channel 被標記為干擾頻道，則跳過
while (channel_map[next_channel] == 0) {
    next_channel = (next_channel + hop_increment) % 37;
}
```

**跳頻示意圖**：

```
Time:     0ms   7.5ms  15ms  22.5ms  30ms  37.5ms
          │     │      │     │       │     │
Channel:  5  →  12  →  19  →  26  →  33  →  3  →  ...
          │     │      │      │      │     │
Packet:  [Data][Data] [Data] [Data] [Data][Data]
```

**Adaptive Frequency Hopping (AFH)**：

如果 Master 檢測到某些頻道的封包遺失率很高，它會：

1. 標記這些頻道為「壞頻道」
2. 更新 Channel Map（37-bit，每個 bit 代表一個頻道）
3. 發送 `LL_CHANNEL_MAP_IND` 給 Slave
4. 雙方同時切換到新的 Channel Map

**範例**：

```
原始 Channel Map: 0x1FFFFFFFFF (所有 37 個頻道都可用)
檢測到 Channel 5, 12, 19 被干擾
新 Channel Map:   0x1FFFEF7FDF (排除 Channel 5, 12, 19)
```

---

## Link Layer 封包格式

### BLE 封包的結構

BLE 的 Link Layer 封包格式如下：

```
┌─────────────────────────────────────────────────────────────────┐
│  BLE Link Layer Packet                                          │
├─────────────────────────────────────────────────────────────────┤
│  Preamble (1 byte)                                              │
│  ├─ 0xAA (if Access Address starts with 1)                     │
│  └─ 0x55 (if Access Address starts with 0)                     │
├─────────────────────────────────────────────────────────────────┤
│  Access Address (4 bytes)                                       │
│  ├─ 0x8E89BED6 (Advertising)                                   │
│  └─ Random (Data Connection, unique per connection)            │
├─────────────────────────────────────────────────────────────────┤
│  PDU (2-257 bytes)                                              │
│  ├─ Header (2 bytes)                                            │
│  │   ├─ PDU Type (4 bits)                                      │
│  │   ├─ RFU (2 bits)                                           │
│  │   ├─ TxAdd / RxAdd (2 bits)                                 │
│  │   └─ Length (8 bits)                                        │
│  └─ Payload (0-255 bytes)                                       │
├─────────────────────────────────────────────────────────────────┤
│  CRC (3 bytes)                                                  │
│  └─ 24-bit CRC for error detection                             │
└─────────────────────────────────────────────────────────────────┘
```

**各欄位說明**：

1. **Preamble（前導碼）**：
   - 用於接收端的頻率同步和時序恢復
   - 1 byte，交替的 0 和 1（0xAA 或 0x55）

2. **Access Address（存取位址）**：
   - Advertising：固定為 0x8E89BED6
   - Data Connection：每個連線都有唯一的隨機值
   - 用於識別封包屬於哪個連線

3. **PDU（Protocol Data Unit）**：
   - Header：PDU 類型、長度
   - Payload：實際的資料

4. **CRC（循環冗餘檢查）**：
   - 24-bit CRC
   - 用於檢測傳輸錯誤

### Advertising 封包範例

**iBeacon Advertising 封包**：

```
Preamble:        0xAA
Access Address:  0x8E89BED6
PDU Header:      0x40 0x1E (ADV_NONCONN_IND, Length=30)
PDU Payload:     [30 bytes of iBeacon data]
CRC:             0xXXXXXX (calculated)

完整封包長度: 1 + 4 + 2 + 30 + 3 = 40 bytes
傳輸時間: 40 bytes × 8 bits/byte / 1 Mbps = 320 μs
```

---

## TX Power, RX Sensitivity, Link Budget

### TX Power（發射功率）

**TX Power** 是發射端的功率，單位是 **dBm**（decibel-milliwatts）。

**BLE 的 TX Power 範圍**：

| 等級 | TX Power | 範圍 | 應用 |
|------|----------|------|------|
| **Class 1** | +20 dBm (100 mW) | ~100 m | 工業應用、長距離通訊 |
| **Class 2** | +4 dBm (2.5 mW) | ~10 m | 手機、筆電 |
| **Class 3** | 0 dBm (1 mW) | ~1 m | 穿戴式裝置、Beacon |
| **Ultra Low** | -20 dBm (0.01 mW) | ~0.1 m | 極低功耗應用 |

**dBm 與 mW 的轉換**：

```
P(dBm) = 10 × log₁₀(P(mW))

範例：
  0 dBm = 10 × log₁₀(1 mW) = 0
 +3 dBm = 10 × log₁₀(2 mW) ≈ 3
 +10 dBm = 10 × log₁₀(10 mW) = 10
 +20 dBm = 10 × log₁₀(100 mW) = 20
```

**TX Power 的權衡**：

- **高 TX Power**：
  - ✅ 範圍更遠
  - ❌ 功耗更高
  - ❌ 對其他裝置的干擾更大

- **低 TX Power**：
  - ✅ 功耗更低
  - ✅ 對其他裝置的干擾更小
  - ❌ 範圍更短

### RX Sensitivity（接收靈敏度）

**RX Sensitivity** 是接收端能夠正確解調的最小訊號功率，單位是 **dBm**。

**BLE 的 RX Sensitivity 要求**：

- **BLE 4.0/4.1/4.2**：-70 dBm @ 1 Mbps
- **BLE 5.0 (1 Mbps)**：-70 dBm
- **BLE 5.0 (2 Mbps)**：-67 dBm
- **BLE 5.0 (Coded PHY, 125 kbps)**：-82 dBm

**RX Sensitivity 越低越好**：

- -70 dBm 比 -60 dBm 更好（能接收更弱的訊號）
- -82 dBm 比 -70 dBm 更好（範圍更遠）

### Link Budget（鏈路預算）

**Link Budget** 是計算無線鏈路的最大傳輸距離。

**Link Budget 公式**：

```
RX Power (dBm) = TX Power (dBm) + TX Antenna Gain (dBi)
                 - Path Loss (dB) + RX Antenna Gain (dBi)

如果 RX Power ≥ RX Sensitivity，則連線成功
```

**Path Loss（路徑損耗）**：

在自由空間中，Path Loss 的計算公式：

```
Path Loss (dB) = 20 × log₁₀(d) + 20 × log₁₀(f) + 32.44

其中：
  d = 距離（公里）
  f = 頻率（MHz）

範例（2.4 GHz，10 公尺）：
  Path Loss = 20 × log₁₀(0.01) + 20 × log₁₀(2400) + 32.44
            = -40 + 67.6 + 32.44
            = 60 dB
```

**Link Budget 範例**：

```
TX Power:          0 dBm
TX Antenna Gain:   0 dBi (內建天線)
Path Loss:         60 dB (10 公尺)
RX Antenna Gain:   0 dBi (內建天線)
RX Power:          0 + 0 - 60 + 0 = -60 dBm

RX Sensitivity:    -70 dBm
Margin:            -60 - (-70) = 10 dB ✅ (連線成功)
```

**如果距離增加到 100 公尺**：

```
Path Loss:         80 dB (100 公尺)
RX Power:          0 + 0 - 80 + 0 = -80 dBm
Margin:            -80 - (-70) = -10 dB ❌ (連線失敗)
```

**如何增加範圍？**

1. **提高 TX Power**：從 0 dBm 提升到 +10 dBm（增加 10 dB）
2. **使用外部天線**：增加 Antenna Gain（+2 dBi 到 +5 dBi）
3. **改善 RX Sensitivity**：使用更好的接收器（-70 dBm → -82 dBm）
4. **使用 BLE 5.0 Coded PHY**：RX Sensitivity 可達 -82 dBm

---

## 真實案例：RF 性能調校與認證測試

### 案例背景

2014 年 10 月，我們的 BLE 晶片準備進行 Bluetooth SIG 認證測試。

認證測試包含三大項：

1. **RF 測試**：TX Power, RX Sensitivity, Frequency Accuracy
2. **Protocol 測試**：使用 PTS (Profile Tuning Suite)
3. **Interoperability 測試**：與其他廠商的裝置互通

我負責 RF 測試的準備工作。

### 問題 1：TX Power 不足

**測試結果**：

```
要求：0 dBm ± 2 dB
實測：-3 dBm ❌
```

**原因分析**：

我和 RF 工程師一起檢查：

1. **PA（Power Amplifier）偏壓不足**：
   - 檢查 PA 的 DC 偏壓：應該是 1.8V，實測只有 1.5V
   - 原因：電源穩壓器的輸出電壓偏低
   - 解決：調整穩壓器的回授電阻，提升輸出電壓到 1.8V

2. **Matching Network 不匹配**：
   - 使用網路分析儀測量 S11（反射係數）
   - 發現在 2.4 GHz 的 S11 = -10 dB（應該 < -15 dB）
   - 原因：Matching Network 的電容值不對
   - 解決：調整電容值，使 S11 < -20 dB

**調校後結果**：

```
TX Power: +1 dBm ✅
```

### 問題 2：RX Sensitivity 不佳

**測試結果**：

```
要求：-70 dBm @ 1 Mbps
實測：-65 dBm ❌
```

**原因分析**：

1. **LNA（Low Noise Amplifier）增益不足**：
   - 檢查 LNA 的偏壓和增益
   - 發現 LNA 的增益只有 10 dB（應該 > 15 dB）
   - 原因：LNA 的偏壓電流太小
   - 解決：增加偏壓電流，提升增益到 18 dB

2. **ADC 的量化噪聲太大**：
   - 檢查 ADC 的 SNR（Signal-to-Noise Ratio）
   - 發現 SNR 只有 30 dB（應該 > 40 dB）
   - 原因：ADC 的參考電壓不穩定
   - 解決：加入濾波電容，穩定參考電壓

**調校後結果**：

```
RX Sensitivity: -72 dBm ✅
```

### 問題 3：Frequency Accuracy 超標

**測試結果**：

```
要求：±20 ppm
實測：±35 ppm ❌
```

**原因分析**：

1. **Crystal Oscillator 的頻率偏移**：
   - 測量 Crystal 的頻率：應該是 32 MHz，實測是 32.0011 MHz
   - 偏移：(32.0011 - 32) / 32 × 10⁶ = 34 ppm
   - 原因：Crystal 的負載電容不對
   - 解決：調整負載電容，使頻率偏移 < 10 ppm

2. **溫度漂移**：
   - 在不同溫度下測量頻率（-40°C 到 +85°C）
   - 發現溫度漂移達 ±15 ppm
   - 解決：使用溫度補償演算法（TCXO, Temperature Compensated Crystal Oscillator）

**調校後結果**：

```
Frequency Accuracy: ±12 ppm ✅
```

### 認證測試結果

經過兩週的調校，我們的 BLE 晶片通過了所有 RF 測試：

- ✅ TX Power: +1 dBm
- ✅ RX Sensitivity: -72 dBm
- ✅ Frequency Accuracy: ±12 ppm
- ✅ Modulation Characteristics: 符合規格
- ✅ Spurious Emissions: 符合 FCC/ETSI 要求

2014 年 11 月，我們獲得了 Bluetooth SIG 的認證（QDID: XXXXXX）。

---

## 總結

這篇文章深入探討了 Bluetooth 的 PHY 和 RF 層：

1. **2.4 GHz ISM 頻段**：
   - 全球免授權，但擁擠
   - BLE 使用 40 個頻道（3 個 Advertising + 37 個 Data）
   - Advertising Channels 精心選擇，避開 WiFi

2. **GFSK 調變**：
   - 1 Mbps 資料速率
   - 功耗低、範圍遠、抗干擾
   - BLE 5.0 引入 2 Mbps 和 Coded PHY

3. **Frequency Hopping**：
   - 每個 Connection Event 使用不同頻道
   - Adaptive Frequency Hopping 避開干擾頻道
   - 提高可靠性和抗干擾能力

4. **Link Layer 封包格式**：
   - Preamble + Access Address + PDU + CRC
   - Advertising 使用固定 Access Address (0x8E89BED6)
   - Data Connection 使用隨機 Access Address

5. **TX Power, RX Sensitivity, Link Budget**：
   - TX Power：0 dBm 到 +20 dBm
   - RX Sensitivity：-70 dBm (BLE 4.x) 到 -82 dBm (BLE 5.0 Coded PHY)
   - Link Budget 決定最大傳輸距離

6. **RF 性能調校**：
   - TX Power：調整 PA 偏壓、Matching Network
   - RX Sensitivity：調整 LNA 增益、ADC SNR
   - Frequency Accuracy：調整 Crystal 負載電容、溫度補償

**關鍵要點**：

- Bluetooth 不只是軟體協定，更是硬體與 RF 的結合
- 理解 PHY 和 RF，才能真正掌握 Bluetooth 的全貌
- RF 性能調校需要硬體知識、測試儀器、耐心和經驗

---

## 參考資料

1. Bluetooth SIG. (2013). *Bluetooth Core Specification Version 4.1 - Vol 6: Low Energy Controller*. <https://www.bluetooth.com/specifications/specs/core-specification-4-1/>
2. Bluetooth SIG. (2016). *Bluetooth Core Specification Version 5.0 - Vol 6: Low Energy Controller*. <https://www.bluetooth.com/specifications/specs/core-specification-5-0/>
3. Bluetooth SIG. (2014). *Bluetooth RF Test Specification*. <https://www.bluetooth.com/>
4. Nordic Semiconductor. (2015). *nRF52 Series - Radio Specification*. <https://www.nordicsemi.com/>
5. Texas Instruments. (2014). *CC2640 Bluetooth Low Energy RF Performance*. <https://www.ti.com/>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
