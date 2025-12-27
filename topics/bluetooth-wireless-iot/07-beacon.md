# BLE Beacon - 連接物理世界與數位世界的橋樑

**作者**: Danny Jiang  
**日期**: 2025-12-11  

---

## 前言：一個改變零售業的小裝置

2014 年 11 月，一家連鎖零售商找上我們，提出了一個有趣的需求：

「我們希望當顧客走到貨架旁邊時，手機能自動彈出折扣通知。不需要顧客掃描 QR Code，也不需要打開 App，就能收到優惠券。」

這聽起來像科幻電影的場景，但技術上是可行的——答案就是 **BLE Beacon**。

Beacon 的概念很簡單：在店內的每個貨架上放置一個小裝置（比火柴盒還小），它會不斷廣播一個訊號。當顧客的手機接收到這個訊號時，就知道「我現在在 A 貨架旁邊」，然後觸發相應的動作（例如推送優惠券）。

但實作起來，挑戰重重：

- **功耗要求**：Beacon 要能用 2 年（使用一顆 CR2032 鈕扣電池）
- **精確度要求**：要能區分「顧客在 A 貨架」還是「B 貨架」（距離約 5 公尺）
- **成本要求**：每個 Beacon 的成本要低於 5 美元（店內要部署 200 個）
- **穩定性要求**：在擁擠的商場環境中，訊號不能被干擾

我們花了 3 個月時間，解決了這些挑戰：

- **功耗優化**：將 Advertising Interval 調整到 1 秒，TX Power 降到 -12 dBm
- **精確度優化**：使用 RSSI（訊號強度）+ 三邊測量法，精確度達到 3-5 公尺
- **成本優化**：使用最簡單的 BLE 晶片，只實作 Advertising 功能（不需要 Connection）
- **穩定性優化**：使用 Advertising Channel 37, 38, 39（避開 WiFi 頻道）

2015 年 2 月，系統正式上線。第一個月，優惠券的點擊率提升了 300%，轉換率提升了 150%。

這就是 Beacon 的魔力：**它讓物理世界與數位世界無縫連接**。

---

## Beacon 的核心概念：數位燈塔

### 什麼是 Beacon？

**Beacon** 的字面意思是「燈塔」或「信標」。在 BLE 的世界裡，Beacon 是一個**單向廣播器**，它不斷發送訊號，但不接收任何回應。

**比喻**：

想像一座燈塔，它不斷旋轉，發出光束。船隻（手機）看到光束後，就知道「我在燈塔附近」，然後可以決定要不要靠近。燈塔不需要知道有多少船隻看到了它的光，它只管不斷發光。

**技術定義**：

Beacon 是一個 BLE 裝置，工作在 **Advertising-only mode**（只廣播模式）。它不建立連線，只發送 Advertising 封包。

### Advertising-only Mode vs Connection Mode

BLE 裝置有兩種工作模式：

| 模式 | Advertising-only Mode | Connection Mode |
|------|----------------------|-----------------|
| **功能** | 只廣播，不連線 | 可以建立連線，雙向通訊 |
| **功耗** | **極低**（可用 2 年） | 較高（需要維持連線） |
| **資料量** | 小（每次最多 31 bytes） | 大（可以傳輸大量資料） |
| **應用** | Beacon、室內定位、Proximity Marketing | 智慧手環、智慧手錶、心率帶 |
| **範例** | iBeacon、Eddystone | Fitbit、Apple Watch |

**Beacon 的優勢**：

1. **極低功耗**：不需要維持連線，只需要定期廣播
2. **簡單**：不需要配對，不需要處理連線狀態
3. **可擴展**：一個手機可以同時接收數百個 Beacon 的訊號

---

## iBeacon：Apple 的室內定位方案

### iBeacon 的誕生

2013 年，Apple 在 WWDC 上發布了 **iBeacon** 技術。這是 Apple 對 BLE Advertising 的一個應用規範。

**iBeacon 的核心概念**：

- 每個 Beacon 廣播一個 **UUID**（唯一識別碼）
- 手機接收到 UUID 後，可以判斷「我在哪個 Beacon 附近」
- App 可以根據 Beacon 的位置，觸發相應的動作

### iBeacon 封包格式

iBeacon 使用 BLE Advertising 封包，格式如下：

```
┌─────────────────────────────────────────────────────────┐
│  iBeacon Advertising Packet (31 bytes)                  │
├─────────────────────────────────────────────────────────┤
│  Prefix (9 bytes)                                        │
│  ├─ 0x02 0x01 0x06  (Flags)                             │
│  ├─ 0x1A 0xFF       (Manufacturer Specific Data)        │
│  └─ 0x4C 0x00       (Apple Company ID)                  │
├─────────────────────────────────────────────────────────┤
│  iBeacon Data (21 bytes)                                 │
│  ├─ 0x02 0x15       (iBeacon Type & Length)             │
│  ├─ UUID (16 bytes) ← 唯一識別碼                         │
│  ├─ Major (2 bytes) ← 區域識別碼                         │
│  ├─ Minor (2 bytes) ← 子區域識別碼                       │
│  └─ TX Power (1 byte) ← 發射功率（用於距離估算）          │
└─────────────────────────────────────────────────────────┘
```

**欄位說明**：

1. **UUID (16 bytes)**：
   - 唯一識別碼，通常代表一個「應用」或「組織」
   - 範例：`E2C56DB5-DFFB-48D2-B060-D0F5A71096E0`（某零售商的 UUID）

2. **Major (2 bytes)**：
   - 區域識別碼，通常代表一個「店面」或「樓層」
   - 範例：`0x0001`（第 1 號店）

3. **Minor (2 bytes)**：
   - 子區域識別碼，通常代表一個「貨架」或「區域」
   - 範例：`0x0005`（第 5 號貨架）

4. **TX Power (1 byte)**：
   - 發射功率（單位：dBm），用於估算距離
   - 範例：`0xC5`（-59 dBm）

### iBeacon 的應用場景

**場景 1：零售店的 Proximity Marketing**

```
店內部署：
- UUID: E2C56DB5-DFFB-48D2-B060-D0F5A71096E0（零售商 A）
- Major: 0x0001（台北信義店）
- Minor: 0x0001（電子產品區）
- Minor: 0x0002（服飾區）
- Minor: 0x0003（食品區）

顧客走到電子產品區：
1. 手機接收到 Beacon 訊號（UUID + Major + Minor）
2. App 判斷：「顧客在台北信義店的電子產品區」
3. 推送通知：「電子產品 8 折優惠，限時 30 分鐘！」
```

**場景 2：博物館的導覽系統**

```
博物館部署：
- UUID: A1B2C3D4-E5F6-7890-ABCD-EF1234567890（博物館 B）
- Major: 0x0001（1 樓）
- Major: 0x0002（2 樓）
- Minor: 0x0001（展品 A）
- Minor: 0x0002（展品 B）

遊客走到展品 A：
1. 手機接收到 Beacon 訊號
2. App 判斷：「遊客在 1 樓的展品 A 旁邊」
3. 播放語音導覽：「這是清朝的瓷器...」
```

---

## Eddystone：Google 的開放標準

### Eddystone 的誕生

2015 年，Google 發布了 **Eddystone**，這是一個開放的 Beacon 標準（相對於 Apple 的 iBeacon）。

**Eddystone 的特色**：

1. **開放標準**：任何人都可以使用，不需要 Apple 的授權
2. **多種 Frame Type**：支援 4 種不同的廣播格式
3. **跨平台**：Android 和 iOS 都支援

### Eddystone 的 4 種 Frame Type

Eddystone 定義了 4 種 Frame Type，每種有不同的用途：

| Frame Type | 用途 | 資料內容 |
|-----------|------|---------|
| **Eddystone-UID** | 唯一識別碼（類似 iBeacon） | Namespace (10 bytes) + Instance (6 bytes) |
| **Eddystone-URL** | 廣播 URL（可以直接開啟網頁） | URL（最多 17 bytes，壓縮格式） |
| **Eddystone-TLM** | 遙測資料（Telemetry） | 電池電壓、溫度、廣播次數 |
| **Eddystone-EID** | 加密識別碼（Ephemeral ID） | 定期變化的加密 ID（隱私保護） |

### Eddystone-UID：唯一識別碼

**封包格式**：

```
┌─────────────────────────────────────────────────────────┐
│  Eddystone-UID Packet                                    │
├─────────────────────────────────────────────────────────┤
│  Service UUID (2 bytes): 0xFEAA (Eddystone)             │
│  Frame Type (1 byte): 0x00 (UID)                         │
│  TX Power (1 byte): -59 dBm                              │
│  Namespace (10 bytes): 唯一命名空間                       │
│  Instance (6 bytes): 實例識別碼                           │
│  RFU (2 bytes): Reserved for Future Use                 │
└─────────────────────────────────────────────────────────┘
```

**與 iBeacon 的對比**：

| 欄位 | iBeacon | Eddystone-UID |
|------|---------|---------------|
| 唯一識別碼 | UUID (16 bytes) | Namespace (10 bytes) + Instance (6 bytes) |
| 區域識別碼 | Major (2 bytes) | Namespace 的一部分 |
| 子區域識別碼 | Minor (2 bytes) | Instance (6 bytes) |

### Eddystone-URL：廣播 URL

**Eddystone-URL 的創新**：

Beacon 可以直接廣播一個 URL，手機接收到後，可以直接開啟網頁，**不需要安裝 App**！

**封包格式**：

```
┌─────────────────────────────────────────────────────────┐
│  Eddystone-URL Packet                                    │
├─────────────────────────────────────────────────────────┤
│  Service UUID (2 bytes): 0xFEAA (Eddystone)             │
│  Frame Type (1 byte): 0x10 (URL)                         │
│  TX Power (1 byte): -59 dBm                              │
│  URL Scheme (1 byte): 0x00 (http://www.)                │
│  Encoded URL (最多 17 bytes): example.com/promo          │
└─────────────────────────────────────────────────────────┘
```

**URL 壓縮**：

由於 Advertising 封包只有 31 bytes，Eddystone-URL 使用壓縮格式：

| URL Scheme | 編碼 |
|-----------|------|
| `http://www.` | 0x00 |
| `https://www.` | 0x01 |
| `http://` | 0x02 |
| `https://` | 0x03 |

| URL 後綴 | 編碼 |
|---------|------|
| `.com/` | 0x00 |
| `.org/` | 0x01 |
| `.edu/` | 0x02 |
| `.net/` | 0x03 |

**範例**：

```
原始 URL: https://www.example.com/promo
壓縮後:
- URL Scheme: 0x01 (https://www.)
- Encoded URL: example 0x00 promo
- 總長度: 1 + 7 + 1 + 5 = 14 bytes
```

### Eddystone-TLM：遙測資料

**Eddystone-TLM** 用於廣播 Beacon 的狀態資訊：

```
┌─────────────────────────────────────────────────────────┐
│  Eddystone-TLM Packet                                    │
├─────────────────────────────────────────────────────────┤
│  Service UUID (2 bytes): 0xFEAA (Eddystone)             │
│  Frame Type (1 byte): 0x20 (TLM)                         │
│  TLM Version (1 byte): 0x00                              │
│  Battery Voltage (2 bytes): 3000 mV                      │
│  Temperature (2 bytes): 22.5 °C                          │
│  Advertising Count (4 bytes): 1,234,567                  │
│  Uptime (4 bytes): 86400 seconds (1 day)                 │
└─────────────────────────────────────────────────────────┘
```

**用途**：

- 監控 Beacon 的電池電量（提前更換電池）
- 監控 Beacon 的溫度（避免過熱）
- 統計 Beacon 的廣播次數（分析使用情況）

---

## 室內定位技術：RSSI 與三邊測量法

### RSSI：訊號強度指示器

**RSSI (Received Signal Strength Indicator)** 是接收到的訊號強度，單位是 dBm。

**RSSI 與距離的關係**：

訊號強度會隨著距離衰減，遵循 **Path Loss Model**：

```
RSSI = TX Power - 10 * n * log10(d) + X

其中：
- TX Power: 發射功率（dBm）
- n: Path Loss Exponent（路徑損耗指數，通常為 2-4）
- d: 距離（公尺）
- X: 隨機雜訊（Gaussian noise）
```

**範例**：

```
假設：
- TX Power = -59 dBm（1 公尺處的 RSSI）
- n = 2（自由空間）

距離 1 公尺：RSSI = -59 dBm
距離 2 公尺：RSSI = -59 - 10 * 2 * log10(2) = -59 - 6 = -65 dBm
距離 5 公尺：RSSI = -59 - 10 * 2 * log10(5) = -59 - 14 = -73 dBm
距離 10 公尺：RSSI = -59 - 10 * 2 * log10(10) = -59 - 20 = -79 dBm
```

**RSSI 的問題**：

1. **環境干擾**：牆壁、人群、金屬物體會影響訊號
2. **多路徑效應**：訊號反射會導致 RSSI 不穩定
3. **裝置差異**：不同手機的 RSSI 測量值可能不同

**解決方案**：

- **平均濾波**：對多次測量取平均值
- **卡爾曼濾波**：使用卡爾曼濾波器平滑 RSSI
- **校準**：在實際環境中校準 Path Loss Exponent

### 三邊測量法（Trilateration）

**三邊測量法** 使用 3 個 Beacon 的 RSSI，計算手機的位置。

**原理**：

```
已知：
- Beacon A 的位置 (x1, y1)，距離 d1
- Beacon B 的位置 (x2, y2)，距離 d2
- Beacon C 的位置 (x3, y3)，距離 d3

求解：手機的位置 (x, y)

方程式：
(x - x1)² + (y - y1)² = d1²
(x - x2)² + (y - y2)² = d2²
(x - x3)² + (y - y3)² = d3²
```

**圖示**：

```
        Beacon A (x1, y1)
           ●
          /|\
         / | \
        /  |  \
    d1 /   |   \ d2
      /    |    \
     /     |     \
    /      |      \
   ●───────●───────●
  手機    Beacon B   Beacon C
  (x, y)  (x2, y2)   (x3, y3)
           d3
```

**精確度**：

- **理想情況**：精確度可達 1-2 公尺
- **實際情況**：精確度約 3-5 公尺（受環境干擾影響）

---

## Beacon 的應用場景

### 1. 室內導航（Indoor Navigation）

**場景**：大型商場、機場、醫院

**實作**：

```
部署：
- 在每個關鍵位置放置 Beacon（每 10 公尺一個）
- 每個 Beacon 廣播其位置資訊（UUID + Major + Minor）

導航：
1. 手機接收到多個 Beacon 的訊號
2. 使用三邊測量法計算手機位置
3. 在地圖上顯示「你在這裡」
4. 規劃路線到目的地
```

### 2. Proximity Marketing（近場行銷）

**場景**：零售店、餐廳、展覽

**實作**：

```
部署：
- 在每個貨架/展位放置 Beacon
- 每個 Beacon 對應一個促銷活動

行銷：
1. 顧客走到貨架旁邊
2. 手機接收到 Beacon 訊號
3. App 推送優惠券：「電子產品 8 折，限時 30 分鐘！」
4. 顧客點擊優惠券，完成購買
```

### 3. 資產追蹤（Asset Tracking）

**場景**：倉庫、醫院、工廠

**實作**：

```
部署：
- 在每個資產（設備、貨物）上貼 Beacon
- 在倉庫的固定位置放置 Gateway（接收 Beacon 訊號）

追蹤：
1. Beacon 定期廣播訊號
2. Gateway 接收訊號，記錄 RSSI
3. 計算資產的位置
4. 在系統中顯示「設備 A 在倉庫 B 區」
```

### 4. 存取控制（Access Control）

**場景**：辦公室、停車場、飯店

**實作**：

```
部署：
- 在門口放置 Beacon
- 員工的手機安裝 App

存取：
1. 員工走到門口
2. 手機接收到 Beacon 訊號
3. App 自動發送開門請求（包含員工 ID）
4. 伺服器驗證權限，開門
```

---

## Beacon 的實作挑戰

### 1. 功耗優化：2 年電池壽命

**挑戰**：

Beacon 通常使用 CR2032 鈕扣電池（容量約 220 mAh），要能用 2 年。

**計算**：

```
電池容量：220 mAh
使用時間：2 年 = 730 天 = 17,520 小時

平均電流：220 mAh / 17,520 小時 = 12.6 μA
```

**優化策略**：

1. **延長 Advertising Interval**：
   - 從 100ms 延長到 1 秒（降低 10 倍功耗）
   - 權衡：延遲增加（手機需要等待 1 秒才能接收到訊號）

2. **降低 TX Power**：
   - 從 0 dBm 降到 -12 dBm（降低 4 倍功耗）
   - 權衡：範圍縮小（從 50 公尺降到 10 公尺）

3. **使用 Deep Sleep Mode**：
   - 在 Advertising 之間進入 Deep Sleep
   - 功耗從 1 mA 降到 1 μA

**實測結果**：

```
配置：
- Advertising Interval: 1 秒
- TX Power: -12 dBm
- Deep Sleep: 啟用

功耗：
- Advertising: 10 mA（持續 1 ms）
- Deep Sleep: 1 μA（持續 999 ms）

平均電流：
(10 mA * 1 ms + 1 μA * 999 ms) / 1000 ms = 11 μA

電池壽命：
220 mAh / 11 μA = 20,000 小時 = 833 天 ≈ 2.3 年 ✅
```

### 2. 精確度優化：3-5 公尺

**挑戰**：

RSSI 受環境干擾影響，精確度不穩定。

**優化策略**：

1. **平均濾波**：

   ```c
   float rssi_filtered = 0;
   for (int i = 0; i < 10; i++) {
       rssi_filtered += rssi_samples[i];
   }
   rssi_filtered /= 10;
   ```

2. **卡爾曼濾波**：

   ```c
   // 卡爾曼濾波器（簡化版）
   float kalman_filter(float rssi_measured, float rssi_estimated, float error_estimate) {
       float kalman_gain = error_estimate / (error_estimate + measurement_noise);
       float rssi_new = rssi_estimated + kalman_gain * (rssi_measured - rssi_estimated);
       float error_new = (1 - kalman_gain) * error_estimate;
       return rssi_new;
   }
   ```

3. **環境校準**：
   - 在實際環境中測量 Path Loss Exponent
   - 建立 RSSI-Distance 對照表

### 3. 成本優化：< 5 美元

**挑戰**：

Beacon 要大量部署（數百個），成本要低。

**優化策略**：

1. **使用最簡單的 BLE 晶片**：
   - 只需要 Advertising 功能，不需要 Connection
   - 選擇低成本晶片（如 Nordic nRF51822）

2. **簡化硬體設計**：
   - 不需要 Display、Button、Sensor
   - 只需要 BLE 晶片 + 電池座 + PCB

3. **大量採購**：
   - 一次採購 1000 個，降低單價

**成本分析**：

```
BLE 晶片：$1.5
PCB + 電池座：$0.5
外殼：$1.0
組裝：$1.0
總成本：$4.0 ✅
```

### 4. 穩定性優化：抗干擾

**挑戰**：

商場環境中，WiFi、Bluetooth、微波爐都會干擾 2.4 GHz 訊號。

**優化策略**：

1. **使用 Advertising Channel 37, 38, 39**：
   - 這 3 個頻道避開了 WiFi 的主要頻道
   - 頻道 37: 2402 MHz
   - 頻道 38: 2426 MHz
   - 頻道 39: 2480 MHz

2. **增加 Advertising 次數**：
   - 每次 Advertising 重複 3 次（在 3 個頻道上）
   - 提高接收成功率

3. **使用 Adaptive Frequency Hopping**：
   - 動態避開干擾頻道

---

## iBeacon vs Eddystone：如何選擇？

| 比較項目 | iBeacon | Eddystone |
|---------|---------|-----------|
| **開發者** | Apple | Google |
| **授權** | 需要 Apple 授權 | 開放標準 |
| **平台支援** | iOS 優先 | Android 優先，iOS 也支援 |
| **識別碼** | UUID + Major + Minor | Namespace + Instance |
| **URL 廣播** | ❌ 不支援 | ✅ 支援（Eddystone-URL） |
| **遙測資料** | ❌ 不支援 | ✅ 支援（Eddystone-TLM） |
| **隱私保護** | ❌ 不支援 | ✅ 支援（Eddystone-EID） |
| **生態系統** | Apple 生態系統強 | Google 生態系統強 |

**選擇建議**：

1. **如果目標用戶主要是 iOS**：選擇 iBeacon
2. **如果目標用戶主要是 Android**：選擇 Eddystone
3. **如果需要 URL 廣播**：選擇 Eddystone-URL
4. **如果需要監控 Beacon 狀態**：選擇 Eddystone-TLM
5. **如果需要隱私保護**：選擇 Eddystone-EID

**最佳實踐**：

- **同時支援 iBeacon 和 Eddystone**：Beacon 可以交替廣播兩種格式
- **使用 Eddystone-TLM 監控狀態**：定期廣播電池電量和溫度

---

## 總結

Beacon 是 BLE 技術的一個經典應用，它讓物理世界與數位世界無縫連接。

**核心概念**：

- **Advertising-only Mode**：只廣播，不連線，極低功耗
- **iBeacon**：Apple 的室內定位方案（UUID + Major + Minor）
- **Eddystone**：Google 的開放標準（UID, URL, TLM, EID）
- **RSSI + 三邊測量法**：室內定位技術

**應用場景**：

- **室內導航**：商場、機場、醫院
- **Proximity Marketing**：零售店、餐廳、展覽
- **資產追蹤**：倉庫、醫院、工廠
- **存取控制**：辦公室、停車場、飯店

**實作挑戰**：

- **功耗優化**：2 年電池壽命（Advertising Interval 1 秒，TX Power -12 dBm）
- **精確度優化**：3-5 公尺（平均濾波、卡爾曼濾波、環境校準）
- **成本優化**：< 5 美元（簡化硬體、大量採購）
- **穩定性優化**：抗干擾（使用 Advertising Channel 37, 38, 39）

在下一篇文章中，我們將深入探討 **Bluetooth PHY & RF**，了解 BLE 如何在 2.4 GHz 頻段中生存。

---

## 參考資料

1. Apple Inc. (2013). *iBeacon for Developers*. <https://developer.apple.com/ibeacon/>
2. Google Inc. (2015). *Eddystone Protocol Specification*. <https://github.com/google/eddystone>
3. Bluetooth SIG. (2013). *Bluetooth Core Specification 4.1*. <https://www.bluetooth.com/specifications/specs/core-specification-4-1/>
4. Faragher, R., & Harle, R. (2015). *Location Fingerprinting With Bluetooth Low Energy Beacons*. IEEE Journal on Selected Areas in Communications.
5. Zafari, F., Gkelias, A., & Leung, K. K. (2019). *A Survey of Indoor Localization Systems and Technologies*. IEEE Communications Surveys & Tutorials.

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
