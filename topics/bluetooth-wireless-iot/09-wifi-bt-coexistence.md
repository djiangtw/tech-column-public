# WiFi/BT Coexistence - 2.4 GHz 的擁擠戰場

**作者**: Danny Jiang  
**日期**: 2025-12-11  

---

## 前言：BLE 連線每 30 秒斷一次

2015 年 3 月，我接到了一個緊急的客戶支援請求。

「我們的智慧手錶在連接 WiFi 後，BLE 連線每 30 秒就會斷開一次，」客戶的工程師在電話裡說，「但如果關閉 WiFi，BLE 就完全正常。」

「你們的 WiFi 和 BLE 是用同一顆晶片嗎？」我問。

「不是，」他說，「WiFi 是 Broadcom 的晶片，BLE 是我們的晶片。它們在同一塊 PCB 上，距離大約 2 公分。」

「那應該是 WiFi 和 BLE 互相干擾了，」我說，「你們有實作 Coexistence 機制嗎？」

「什麼是 Coexistence 機制？」他問。

我意識到，這是一個典型的 WiFi/BT Coexistence 問題。

第二天，我飛到客戶的辦公室，帶著頻譜分析儀和 BLE Sniffer。

我先用頻譜分析儀掃描 2.4 GHz 頻段，看到了這樣的畫面：

```
2.4 GHz 頻譜（WiFi Channel 6 開啟時）：

Power
(dBm)
  0  ┤                 ┌─────────┐
     │                 │ WiFi Ch6│
-20  ┤                 │         │
     │                 │         │
-40  ┤  ┌──┐  ┌──┐  ┌─┴─┐  ┌──┐  ┌──┐
     │  │BLE│  │BLE│  │   │  │BLE│  │BLE│
-60  ┤  │Ch │  │Ch │  │   │  │Ch │  │Ch │
     │  │37 │  │38 │  │   │  │39 │  │0  │
-80  ┴──┴───┴──┴───┴──┴───┴──┴───┴──┴───┴──
     2.4  2.41 2.42 2.43 2.44 2.45 2.46 2.47 2.48 GHz
```

問題一目了然：**WiFi Channel 6（中心頻率 2.437 GHz）完全覆蓋了 BLE 的 Channel 38（2.426 GHz）和部分 Data Channels**。

當 WiFi 在傳輸資料時，BLE 的封包會被完全淹沒。

「你們的 WiFi 每 30 秒傳輸一次資料嗎？」我問。

「對，」客戶的工程師說，「我們的 App 每 30 秒會透過 WiFi 上傳一次資料到雲端。」

「那就是問題所在，」我說，「WiFi 傳輸時，BLE 的封包會被干擾，導致連線斷開。」

「怎麼解決？」

「有三個方向，」我說，「第一，實作 PTA（Packet Traffic Arbitration）機制，讓 WiFi 和 BLE 協調使用頻譜。第二，優化天線設計，增加隔離度。第三，調整 BLE 的 Connection Parameters，避開 WiFi 的傳輸時間。」

接下來的兩週，我和客戶的團隊一起實作了 PTA 機制。最終，BLE 連線穩定性從 30 秒斷線一次，提升到連續運行 24 小時不斷線。

這次經歷讓我深刻理解了一個道理：**在 2.4 GHz 這個擁擠的戰場上，WiFi 和 Bluetooth 不是敵人，而是需要協調的鄰居。理解 Coexistence 機制，是設計穩定 IoT 產品的關鍵**。

---

## 為什麼 WiFi 和 Bluetooth 會互相干擾？

### 2.4 GHz ISM 頻段的擁擠程度

WiFi 和 Bluetooth 都使用 **2.4 GHz ISM 頻段**（2.400 - 2.4835 GHz），這是一個免授權的頻段，任何人都可以使用。

**2.4 GHz 頻段的使用者**：

- **WiFi (802.11 b/g/n)**：使用 20 MHz 或 40 MHz 頻寬
- **Bluetooth Classic**：使用 79 個 1 MHz 頻道
- **BLE (Bluetooth Low Energy)**：使用 40 個 2 MHz 頻道
- **Zigbee (802.15.4)**：使用 16 個 2 MHz 頻道
- **微波爐**：在 2.45 GHz 附近產生強烈干擾
- **無線電話**：部分使用 2.4 GHz

**頻譜重疊示意圖**：

```
┌─────────────────────────────────────────────────────────────────┐
│  2.4 GHz ISM Band (2.400 - 2.4835 GHz)                          │
├─────────────────────────────────────────────────────────────────┤
│  WiFi Channels (20 MHz each):                                   │
│  ├─ Channel 1:  2.412 GHz (2.401 - 2.423 GHz)                  │
│  ├─ Channel 6:  2.437 GHz (2.426 - 2.448 GHz)                  │
│  └─ Channel 11: 2.462 GHz (2.451 - 2.473 GHz)                  │
├─────────────────────────────────────────────────────────────────┤
│  BLE Channels (2 MHz each):                                     │
│  ├─ Channel 37: 2.402 GHz                                       │
│  ├─ Channel 0:  2.404 GHz                                       │
│  ├─ ...                                                          │
│  ├─ Channel 38: 2.426 GHz                                       │
│  ├─ ...                                                          │
│  └─ Channel 39: 2.480 GHz                                       │
└─────────────────────────────────────────────────────────────────┘
```

**重疊分析**：

- **WiFi Channel 1**（2.401 - 2.423 GHz）重疊：
  - BLE Channel 37（2.402 GHz）
  - BLE Channel 0-10（2.404 - 2.424 GHz）

- **WiFi Channel 6**（2.426 - 2.448 GHz）重疊：
  - BLE Channel 38（2.426 GHz）
  - BLE Channel 11-21（2.428 - 2.448 GHz）

- **WiFi Channel 11**（2.451 - 2.473 GHz）重疊：
  - BLE Channel 22-33（2.450 - 2.472 GHz）

**結論**：WiFi 的一個 Channel（20 MHz）會覆蓋 BLE 的 10 個 Channels（2 MHz × 10）。

### 干擾的類型

**1. 同頻干擾（Co-Channel Interference）**：

- WiFi 和 BLE 使用相同的頻率
- WiFi 的訊號功率通常比 BLE 高 10-20 dB
- BLE 的封包會被 WiFi 的訊號淹沒

**2. 鄰頻干擾（Adjacent Channel Interference）**：

- WiFi 的頻譜遮罩不完美，會洩漏到相鄰頻道
- BLE 的相鄰頻道也會受到影響

**3. 阻塞干擾（Blocking）**：

- WiFi 的強訊號會使 BLE 的接收器飽和
- 即使頻率不同，BLE 也無法正常接收

**4. 互調干擾（Intermodulation）**：

- WiFi 和 BLE 的訊號在非線性元件（如 PA、LNA）中混合
- 產生新的干擾頻率

### 干擾的影響

**對 BLE 的影響**：

- **封包遺失率增加**：從 1% 提升到 30-50%
- **連線斷開**：Supervision Timeout 導致連線斷開
- **吞吐量下降**：重傳導致有效吞吐量下降 50-80%
- **延遲增加**：重傳導致延遲增加 2-5 倍

**對 WiFi 的影響**：

- **吞吐量下降**：BLE 的封包會佔用頻譜
- **延遲增加**：WiFi 需要等待 BLE 傳輸完成
- **影響較小**：因為 WiFi 的訊號功率通常比 BLE 高

---

## Coexistence 機制：讓 WiFi 和 BLE 和平共處

### 1. PTA (Packet Traffic Arbitration)

**PTA** 是最常見的 Coexistence 機制，透過硬體訊號線讓 WiFi 和 BLE 協調使用頻譜。

**PTA 的硬體訊號**：

```
WiFi Chip                          BLE Chip
┌─────────┐                        ┌─────────┐
│         │  BT_PRIORITY ────────> │         │
│         │  <──────── WLAN_ACTIVE │         │
│         │  BT_ACTIVE ──────────> │         │
│         │  <──────── WLAN_PRIORITY│        │
└─────────┘                        └─────────┘
```

**訊號說明**：

- **BT_PRIORITY**：BLE 告訴 WiFi「我現在有重要的封包要傳」
- **WLAN_ACTIVE**：WiFi 告訴 BLE「我現在正在傳輸」
- **BT_ACTIVE**：BLE 告訴 WiFi「我現在正在傳輸」
- **WLAN_PRIORITY**：WiFi 告訴 BLE「我現在有重要的封包要傳」

**PTA 的仲裁規則**：

```c
// 簡化的 PTA 仲裁邏輯
if (BT_PRIORITY == HIGH && WLAN_PRIORITY == LOW) {
    // BLE 優先
    grant_access_to_BLE();
} else if (WLAN_PRIORITY == HIGH && BT_PRIORITY == LOW) {
    // WiFi 優先
    grant_access_to_WiFi();
} else if (BT_ACTIVE == HIGH) {
    // BLE 正在傳輸，WiFi 等待
    WiFi_wait();
} else if (WLAN_ACTIVE == HIGH) {
    // WiFi 正在傳輸，BLE 等待
    BLE_wait();
} else {
    // 預設：WiFi 優先（因為 WiFi 的延遲要求更嚴格）
    grant_access_to_WiFi();
}
```

**PTA 的優先權設定**：

| 場景 | BLE Priority | WiFi Priority | 結果 |
|------|--------------|---------------|------|
| BLE Advertising | HIGH | - | BLE 優先 |
| BLE Connection Event | HIGH | - | BLE 優先 |
| BLE Data Transfer | MEDIUM | - | 視 WiFi 狀態而定 |
| WiFi Beacon | - | HIGH | WiFi 優先 |
| WiFi Data Transfer | - | MEDIUM | 視 BLE 狀態而定 |
| WiFi Scan | - | LOW | BLE 優先 |

### 2. Time Division（時間分割）

**Time Division** 是另一種 Coexistence 機制，將時間分割成多個時間片（Time Slot），WiFi 和 BLE 輪流使用。

**Time Division 示意圖**：

```
Time:     0ms    10ms   20ms   30ms   40ms   50ms   60ms
          │      │      │      │      │      │      │
WiFi:     [████████]    [████████]    [████████]
BLE:              [████]        [████]        [████]
          │      │      │      │      │      │      │
          WiFi   BLE    WiFi   BLE    WiFi   BLE
          Slot   Slot   Slot   Slot   Slot   Slot
```

**Time Division 的優點**：

- ✅ **簡單**：不需要複雜的仲裁邏輯
- ✅ **公平**：WiFi 和 BLE 都有固定的時間片
- ✅ **可預測**：每個協定都知道自己什麼時候可以傳輸

**Time Division 的缺點**：

- ❌ **效率低**：即使沒有資料要傳，時間片也會被浪費
- ❌ **延遲高**：需要等待自己的時間片
- ❌ **不靈活**：無法根據實際需求動態調整

**Time Division 的實作**：

```c
// Time Division 的簡化實作
#define WIFI_SLOT_DURATION_MS  10
#define BLE_SLOT_DURATION_MS   5

void time_division_scheduler(void) {
    uint32_t current_time = get_time_ms();
    uint32_t slot_time = current_time % (WIFI_SLOT_DURATION_MS + BLE_SLOT_DURATION_MS);

    if (slot_time < WIFI_SLOT_DURATION_MS) {
        // WiFi 時間片
        enable_wifi();
        disable_ble();
    } else {
        // BLE 時間片
        disable_wifi();
        enable_ble();
    }
}
```

### 3. Adaptive Frequency Hopping（自適應跳頻）

**Adaptive Frequency Hopping (AFH)** 是 Bluetooth 的內建機制，可以避開被干擾的頻道。

**AFH 的工作原理**：

1. **監測頻道品質**：
   - 測量每個頻道的封包遺失率
   - 如果封包遺失率 > 30%，標記為「壞頻道」

2. **更新 Channel Map**：
   - 排除壞頻道
   - 只使用好頻道

3. **通知對方**：
   - Master 發送 `LL_CHANNEL_MAP_IND` 給 Slave
   - 雙方同時切換到新的 Channel Map

**AFH 範例**：

```
原始 Channel Map: 0x1FFFFFFFFF (所有 37 個頻道都可用)

WiFi Channel 6 開啟後，檢測到以下頻道被干擾：
  Channel 11-21 (2.428 - 2.448 GHz)

新 Channel Map: 0x1FFFC007FF (排除 Channel 11-21)

可用頻道數：37 - 11 = 26 個頻道
```

**AFH 的優點**：

- ✅ **自動適應**：不需要手動配置
- ✅ **提高可靠性**：避開干擾頻道
- ✅ **標準化**：所有 BLE 裝置都支援

**AFH 的缺點**：

- ❌ **反應慢**：需要時間檢測和更新
- ❌ **頻道減少**：可用頻道變少，跳頻效果變差
- ❌ **無法完全避免**：如果 WiFi 佔用太多頻道，BLE 仍會受影響

### 4. 混合策略：PTA + AFH

**最佳實踐**是結合 PTA 和 AFH：

- **PTA**：短期協調（微秒級）
  - WiFi 和 BLE 透過硬體訊號協調
  - 避免同時傳輸

- **AFH**：長期適應（秒級）
  - BLE 避開被 WiFi 佔用的頻道
  - 提高整體可靠性

**混合策略示意圖**：

```
短期（微秒級）：PTA 協調
┌─────────────────────────────────────────────────────────┐
│ Time: 0μs    100μs   200μs   300μs   400μs   500μs      │
│       │      │       │       │       │       │          │
│ WiFi: [████] wait    [████]  wait    [████]  wait       │
│ BLE:  wait   [██]    wait    [██]    wait    [██]       │
└─────────────────────────────────────────────────────────┘

長期（秒級）：AFH 適應
┌─────────────────────────────────────────────────────────┐
│ BLE 避開 WiFi Channel 6 佔用的頻道（Channel 11-21）     │
│ 只使用 Channel 0-10, 22-36（26 個頻道）                 │
└─────────────────────────────────────────────────────────┘
```

---

## 硬體設計考量

### 1. 天線隔離（Antenna Isolation）

**天線隔離** 是指兩個天線之間的訊號隔離度，單位是 **dB**。

**天線隔離的重要性**：

- WiFi 的 TX Power 通常是 +15 dBm 到 +20 dBm
- BLE 的 RX Sensitivity 是 -70 dBm
- 如果天線隔離度不足，WiFi 的訊號會洩漏到 BLE 的接收器，導致阻塞

**天線隔離度計算**：

```
WiFi TX Power:        +20 dBm
BLE RX Sensitivity:   -70 dBm
所需隔離度:           +20 - (-70) = 90 dB

實際隔離度（2 公分距離）：
  - 無隔離措施：30-40 dB ❌
  - 使用金屬隔板：50-60 dB ⚠️
  - 使用濾波器 + 隔板：70-80 dB ✅
```

**提高天線隔離度的方法**：

1. **增加天線距離**：
   - 距離加倍，隔離度增加 6 dB
   - 2 cm → 4 cm：+6 dB
   - 2 cm → 8 cm：+12 dB

2. **使用金屬隔板**：
   - 在兩個天線之間放置金屬隔板
   - 可增加 10-20 dB 隔離度

3. **天線方向正交**：
   - WiFi 天線垂直，BLE 天線水平
   - 可增加 10-15 dB 隔離度

4. **使用不同極化**：
   - WiFi 使用垂直極化，BLE 使用水平極化
   - 可增加 15-20 dB 隔離度

### 2. 濾波器（Filter）

**濾波器** 可以過濾掉不需要的頻率，減少干擾。

**常用的濾波器類型**：

1. **SAW Filter（Surface Acoustic Wave Filter）**：
   - 高選擇性（Selectivity）
   - 插入損耗（Insertion Loss）：1-3 dB
   - 成本：中等

2. **BAW Filter（Bulk Acoustic Wave Filter）**：
   - 更高選擇性
   - 插入損耗：0.5-1.5 dB
   - 成本：較高

3. **LC Filter**：
   - 低成本
   - 選擇性較差
   - 插入損耗：0.5-1 dB

**濾波器的應用**：

```
WiFi TX Path:
  PA → SAW Filter → Antenna
       (過濾掉 BLE 頻段的洩漏)

BLE RX Path:
  Antenna → SAW Filter → LNA
            (過濾掉 WiFi 的強訊號)
```

### 3. PCB Layout 優化

**PCB Layout** 對 Coexistence 的影響很大。

**PCB Layout 的最佳實踐**：

1. **分離 WiFi 和 BLE 的 RF 路徑**：
   - WiFi 和 BLE 的 RF 走線不要交叉
   - 使用不同的 PCB 層

2. **使用 Ground Plane**：
   - 在 WiFi 和 BLE 之間放置 Ground Plane
   - 提高隔離度

3. **最小化 RF 走線長度**：
   - RF 走線越短，損耗越小
   - 減少輻射和耦合

4. **使用屏蔽罩（Shield Can）**：
   - 將 WiFi 和 BLE 晶片分別放在屏蔽罩內
   - 可增加 20-30 dB 隔離度

---

## 真實案例：解決 WiFi/BT 共存問題

### 案例背景

2015 年 3 月，客戶的智慧手錶產品遇到 WiFi/BT Coexistence 問題：

- **問題**：WiFi 開啟後，BLE 連線每 30 秒斷開一次
- **硬體**：WiFi（Broadcom BCM43438）+ BLE（我們的晶片）
- **距離**：兩顆晶片距離 2 公分
- **沒有 PTA**：兩顆晶片之間沒有硬體訊號線

### 問題分析

**第一步：測量頻譜**

使用頻譜分析儀測量 2.4 GHz 頻段：

```
WiFi Channel 6 開啟時：
  - WiFi 佔用 2.426 - 2.448 GHz
  - BLE Channel 11-21 被完全覆蓋
  - WiFi TX Power: +18 dBm
  - BLE RX 端測量到 WiFi 洩漏：-30 dBm
```

**第二步：測量天線隔離度**

使用網路分析儀測量天線隔離度：

```
S21 (WiFi Antenna → BLE Antenna): -35 dB

計算：
  WiFi TX Power: +18 dBm
  天線隔離度: -35 dB
  BLE RX 端接收到的 WiFi 訊號: +18 - 35 = -17 dBm

BLE RX Sensitivity: -70 dBm
Margin: -17 - (-70) = 53 dB ❌ (遠超過 BLE 的動態範圍)
```

**結論**：天線隔離度不足，WiFi 的訊號會使 BLE 的接收器飽和。

### 解決方案

**方案 1：軟體優化（短期）**

由於硬體已經定型，無法修改 PCB，我們先用軟體優化：

1. **調整 BLE Connection Parameters**：
   - Connection Interval：從 30 ms 延長到 100 ms
   - Slave Latency：從 0 增加到 4
   - 減少 BLE 的傳輸頻率，避開 WiFi 的傳輸時間

2. **實作 AFH**：
   - 監測頻道品質
   - 排除被 WiFi 干擾的頻道（Channel 11-21）
   - 只使用 26 個頻道

3. **調整 WiFi 的傳輸時機**：
   - 將 WiFi 的資料上傳從「每 30 秒」改為「每 60 秒」
   - 減少 WiFi 的傳輸頻率

**結果**：

```
改善前：BLE 連線每 30 秒斷開一次
改善後：BLE 連線可以維持 5 分鐘（仍不理想）
```

**方案 2：硬體優化（長期）**

在下一版硬體中，我們做了以下改進：

1. **增加天線距離**：
   - 從 2 cm 增加到 5 cm
   - 天線隔離度從 -35 dB 提升到 -45 dB

2. **加入金屬隔板**：
   - 在兩個天線之間加入金屬隔板
   - 天線隔離度從 -45 dB 提升到 -55 dB

3. **加入 SAW Filter**：
   - 在 BLE RX 路徑加入 SAW Filter
   - 過濾掉 WiFi 的強訊號
   - 天線隔離度從 -55 dB 提升到 -70 dB

4. **實作 PTA**：
   - 在 WiFi 和 BLE 晶片之間加入 3 條訊號線
   - BT_PRIORITY, WLAN_ACTIVE, BT_ACTIVE
   - 實作硬體仲裁邏輯

**結果**：

```
天線隔離度：-70 dB
WiFi TX Power: +18 dBm
BLE RX 端接收到的 WiFi 訊號: +18 - 70 = -52 dBm
BLE RX Sensitivity: -70 dBm
Margin: -52 - (-70) = 18 dB ✅ (可接受)

BLE 連線穩定性：連續運行 24 小時不斷線 ✅
```

### 經驗總結

1. **硬體設計很重要**：
   - 天線隔離度是關鍵
   - 至少需要 60-70 dB 的隔離度

2. **PTA 是必需的**：
   - 軟體優化只能緩解問題
   - 硬體 PTA 才能根本解決

3. **AFH 是輔助**：
   - AFH 可以提高可靠性
   - 但無法完全避免干擾

4. **測試很重要**：
   - 使用頻譜分析儀和 BLE Sniffer
   - 在真實環境中測試（WiFi 開啟 + BLE 連線）

---

## 總結

這篇文章深入探討了 WiFi/BT Coexistence 的挑戰和解決方案：

1. **為什麼會互相干擾**：
   - WiFi 和 BLE 都使用 2.4 GHz ISM 頻段
   - WiFi 的 20 MHz 頻寬會覆蓋 BLE 的 10 個頻道
   - 同頻干擾、鄰頻干擾、阻塞干擾

2. **Coexistence 機制**：
   - **PTA**：硬體訊號協調，微秒級
   - **Time Division**：時間分割，簡單但效率低
   - **AFH**：自適應跳頻，避開干擾頻道
   - **混合策略**：PTA + AFH 是最佳實踐

3. **硬體設計考量**：
   - **天線隔離**：至少需要 60-70 dB
   - **濾波器**：SAW/BAW Filter 過濾干擾
   - **PCB Layout**：分離 RF 路徑、使用 Ground Plane、屏蔽罩

4. **真實案例**：
   - 軟體優化：調整 Connection Parameters、AFH
   - 硬體優化：增加天線距離、金屬隔板、SAW Filter、PTA
   - 最終達到 24 小時穩定運行

**關鍵要點**：

- WiFi 和 Bluetooth 不是敵人，而是需要協調的鄰居
- 硬體設計（天線隔離、濾波器）是基礎
- PTA 是必需的，AFH 是輔助
- 在真實環境中測試是關鍵

---

## 參考資料

1. Bluetooth SIG. (2013). *Bluetooth Core Specification Version 4.1 - Vol 6, Part B: Link Layer*. <https://www.bluetooth.com/specifications/specs/core-specification-4-1/>
2. IEEE. (2012). *IEEE 802.11-2012 - Wireless LAN Medium Access Control (MAC) and Physical Layer (PHY) Specifications*. <https://standards.ieee.org/>
3. Bluetooth SIG. (2015). *Coexistence with WiFi White Paper*. <https://www.bluetooth.com/>
4. Nordic Semiconductor. (2015). *nRF52 Series - Coexistence Application Note*. <https://www.nordicsemi.com/>
5. Broadcom. (2014). *BCM43438 - WiFi/BT Coexistence Design Guide*. <https://www.broadcom.com/>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
