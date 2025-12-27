# Bluetooth & Wireless IoT - 從 2014 年 IoT 浪潮說起

**作者**: Danny Jiang  
**日期**: 2025-12-11  

---

## 前言：一個改變世界的小晶片

2014 年 9 月，我加入了一家無線通訊晶片公司，擔任韌體工程師。入職的第一天，主管帶我參觀實驗室時，指著桌上一個比指甲還小的晶片說：

「這顆晶片，將會改變人們的生活方式。」

我當時半信半疑。一顆這麼小的晶片，能有多大的影響力？

但接下來的一年，我親眼見證了 IoT（物聯網）的爆發：

- **2014 年 1 月**：Google 以 32 億美元收購 Nest，智慧家居成為焦點
- **2014 年 6 月**：Apple 發布 HomeKit 和 HealthKit，並宣布 Apple Watch
- **2014 年 7 月**：小米手環 1 代發布，售價僅 79 元人民幣，30 天續航
- **2014 年 9 月**：我們的第一個客戶——一家智慧手錶廠商——要求我們在 6 個月內交付一顆 BLE 晶片

那顆晶片的核心技術，就是 **Bluetooth Low Energy (BLE)**。

當時的挑戰是巨大的：

- **功耗要求**：智慧手環要能用一個月，智慧手錶要能撐 7 天
- **連線穩定性**：在 2.4 GHz 擁擠的頻段中，與 WiFi 和平共處
- **成本壓力**：晶片成本要控制在 1 美元以內
- **開發時程**：6 個月內完成韌體開發、測試、認證

我們的團隊日以繼夜地工作。我負責 BLE 協定堆疊的實作，從 HCI、L2CAP、ATT/GATT 到 SMP，每一層都要從零開始。同時，我還要處理 SPI Flash 驅動（因為客戶要求支援 Quad SPI 以加快開機速度）、MIPI DSI 顯示驅動（智慧手錶需要彩色螢幕）、以及各種 Sensor 的 I2C 整合。

最困難的是功耗優化。第一版韌體跑起來後，智慧手環只能撐 3 天。我們花了 2 個月時間，逐一優化每個環節：

- 調整 BLE Connection Interval 從 7.5ms 延長到 100ms
- 實作 Deep Sleep Mode，讓 CPU 在閒置時進入最低功耗狀態
- 優化 Advertising Interval，從 100ms 延長到 1 秒
- 使用 Notification 而非 Indication，減少 ACK 的開銷

最終，我們將續航時間從 3 天提升到 30 天，達到了客戶的要求。

2015 年 3 月，那款智慧手環正式上市。半年內賣出了 500 萬支。

那一刻，我才真正理解主管當初說的話：**這顆小晶片，確實改變了世界**。

---

## 為什麼是 2014 年？IoT 元年的三大推手

回顧 2014 年，為什麼這一年被稱為「IoT 元年」？有三個關鍵因素：

### 1. 智慧型手機的普及：天然的 Gateway

到了 2014 年，全球智慧型手機的普及率已經超過 50%。更重要的是：

- **iPhone 4S 及後續機種**：全面支援 BLE（Bluetooth 4.0）
- **Android 4.3+**：原生支援 BLE
- **手機就是 Gateway**：使用者不需要額外購買 Hub 或 Gateway，手機就能直接連接 BLE 裝置

這是 BLE 相對於 Zigbee 的最大優勢。Zigbee 需要一個專用的 Gateway（如 Philips Hue Bridge），而 BLE 裝置可以直接與手機配對。這大幅降低了消費者的入門門檻。

### 2. BLE 4.0/4.1 的成熟：低功耗的革命

Bluetooth 4.0 在 2010 年發布，但直到 2013-2014 年才真正成熟：

- **2013 年 - Bluetooth 4.1**：改善與 LTE 的共存性，支援裝置同時作為 Hub 和 Peripheral
- **2014 年 - Bluetooth 4.2**：資料封包長度擴充，傳輸速度提升 2.5 倍

BLE 的核心優勢是**極低功耗**：

- **Classic Bluetooth**：持續連線，功耗約 1W，電池壽命數天
- **BLE**：睡眠/喚醒模式，功耗約 0.01-0.5W，電池壽命數月至數年

這使得鈕扣電池（CR2032）供電的裝置成為可能，例如：

- **iBeacon**：一顆 CR2032 電池可以用 1-2 年
- **智慧手環**：充電一次可以用 30 天
- **智慧門鎖**：4 顆 AA 電池可以用 1 年

### 3. 巨頭的入場：驗證市場價值

2014 年，科技巨頭紛紛入場，驗證了 IoT 的市場價值：

- **Google 收購 Nest**（32 億美元）：智慧家居的價值被認可
- **Apple 發布 HomeKit**：提供統一的智慧家居控制框架
- **Apple Watch 宣布**（2015 年上市）：穿戴式裝置成為主流

這些事件向開發者和投資者發出了明確的信號：**IoT 是下一個大平台**。

---

## BLE 為什麼成為 IoT 的主流？

在 2014 年，IoT 開發者面臨一個選擇：**Zigbee、WiFi，還是 BLE？**

讓我們看看三者的對比：

| 比較項目 | BLE (Bluetooth 4.x) | WiFi (802.11 b/g/n) | Zigbee (802.15.4) |
|---------|---------------------|---------------------|-------------------|
| **功耗** | **極低** | 高 | 低 |
| **傳輸距離** | 短 (10-50m) | 中長 (50-100m+) | 短中 (10-100m, 可 Mesh) |
| **資料速率** | 低 (1 Mbps) | **極高 (600 Mbps+)** | 極低 (250 Kbps) |
| **網路拓撲** | 點對點、星狀 | 星狀 | **網狀網路 (Mesh)** |
| **硬體成本** | 低 ($) | 中 ($$) | 中 ($$) |
| **生態系統** | **手機原生支援** | 基礎設施普及 | 工業/家庭自動化強 |
| **劣勢** | 2014 年尚不支援 Mesh | 耗電，不適合電池供電 | **手機無法直接連接** |

**BLE 的核心優勢**：

1. **手機就是 Gateway**：不需要額外的 Hub
2. **極低功耗**：鈕扣電池可以用數月至數年
3. **低成本**：晶片成本 < 1 美元
4. **快速連線**：連線延遲 < 6ms

**BLE 的劣勢**：

1. **不支援 Mesh**（直到 Bluetooth 5.0 才支援 Mesh）
2. **傳輸距離短**（10-50m）
3. **資料速率低**（1 Mbps）

但對於大多數 IoT 應用（智慧手環、智慧手錶、Beacon、智慧門鎖），BLE 的優勢遠大於劣勢。

---

## 本系列文章的結構

這個系列將涵蓋 Bluetooth & Wireless IoT 的完整技術棧，從協定堆疊到硬體接口，從功耗優化到除錯技巧。

### Part 1: 系列導讀

- **文章 #1**：Bluetooth & Wireless IoT - 從 2014 年 IoT 浪潮說起（本文）

### Part 2: Protocol Stack（協定堆疊）

- **文章 #2**：Bluetooth 協定堆疊概覽 - 從 PHY 到 Application
- **文章 #3**：HCI - 硬體與軟體的分界線
- **文章 #4**：L2CAP - 邏輯鏈路控制與適配協定
- **文章 #5**：ATT/GATT - BLE 的核心資料模型
- **文章 #6**：SMP - BLE 的安全機制
- **文章 #7**：BLE Beacon - 連接物理世界與數位世界的橋樑

### Part 3: PHY & RF（實體層與射頻）

- **文章 #8**：Bluetooth PHY & RF - 從 2.4 GHz 到空中封包
- **文章 #9**：WiFi/BT Coexistence - 2.4 GHz 的擁擠戰場

### Part 4: Peripheral Interfaces（周邊接口）

- **文章 #10**：SPI 深入解析 - 從 Standard 到 Dual/Quad SPI
- **文章 #11**：MIPI DSI - 智慧手錶的 Display 接口
- **文章 #12**：I2C, UART, GPIO - IoT 晶片的基礎接口

### Part 5: Power Optimization（功耗優化）

- **文章 #13**：BLE 功耗優化 - IoT 裝置的生命線
- **文章 #14**：系統級功耗優化 - 從晶片到系統

### Part 6: Debugging & Testing（除錯與測試）

- **文章 #15**：BLE 除錯實戰 - Sniffer, Log, 與測試工具
- **文章 #16**：BLE 認證與測試 - 從開發到量產

### Part 7: Comparison & Trends（對比與趨勢）

- **文章 #17**：Zigbee vs Bluetooth - IoT 協定的選擇
- **文章 #18**：RF4CE, Thread, 與 Matter - IoT 協定的演進
- **文章 #19**：AIoT - 從 IoT 到 AI + IoT 的演進

### Part 8: Case Studies（案例研究）

- **文章 #20**：智慧手錶案例研究 - 完整的 IoT 產品開發
- **文章 #21**：IoT 安全實戰 - 從晶片到雲端

---

## 不同背景的閱讀路徑

根據你的背景和興趣，我建議不同的閱讀順序：

### 路徑 1：韌體開發者（從協定堆疊開始）

1. 文章 #1（本文）→ #2 → #3 → #4 → #5 → #6
2. 文章 #13 → #14（功耗優化）
3. 文章 #15 → #16（除錯與測試）
4. 文章 #10 → #11 → #12（周邊接口）

### 路徑 2：系統架構師（從整體架構開始）

1. 文章 #1（本文）→ #2（協定堆疊概覽）
2. 文章 #8 → #9（PHY & RF）
3. 文章 #17 → #18 → #19（協定對比與趨勢）
4. 文章 #20 → #21（案例研究）

### 路徑 3：產品設計師（從應用場景開始）

1. 文章 #1（本文）→ #7（Beacon）
2. 文章 #13 → #14（功耗優化）
3. 文章 #20（智慧手錶案例）
4. 文章 #21（IoT 安全）

---

## 總結

2014 年是 IoT 的元年，BLE 憑藉其極低功耗、手機原生支援、低成本等優勢，成為 IoT 的主流協定。

在接下來的文章中，我們將深入探討：

- **協定堆疊**：從 PHY 到 Application 的完整架構
- **硬體接口**：SPI、MIPI DSI、I2C 等周邊接口的實作
- **功耗優化**：如何將電池壽命從 3 天提升到 30 天
- **除錯技巧**：Sniffer、HCI Log、測試工具的使用
- **案例研究**：智慧手錶的完整開發過程

讓我們開始這段旅程吧！

---

## 參考資料

1. Bluetooth SIG. (2013). *Bluetooth Core Specification 4.1*. <https://www.bluetooth.com/specifications/specs/core-specification-4-1/>
2. Bluetooth SIG. (2014). *Bluetooth Core Specification 4.2*. <https://www.bluetooth.com/specifications/specs/core-specification-4-2/>
3. Apple Inc. (2014). *HomeKit Framework*. <https://developer.apple.com/homekit/>
4. Google Inc. (2014). *Nest Acquisition Announcement*. <https://abc.xyz/investor/>
5. Xiaomi Inc. (2014). *Mi Band Product Specifications*. <https://www.mi.com/>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
