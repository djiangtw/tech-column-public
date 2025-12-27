---
author: Danny Jiang
date: 2025-12-11
---

# Zigbee vs Bluetooth - IoT 協定的選擇

## 前言：2014 年的協定選擇難題

2014 年 10 月，我接到了一個來自智慧家居客戶的電話。

"Danny，我們正在開發一個智慧照明系統，"客戶的產品經理 Sarah 說，"我們需要選擇一個無線協定。你覺得應該用 Bluetooth 還是 Zigbee？"

我愣了一下。這是一個經典的問題，但答案並不簡單。

"這取決於你們的應用場景，"我說，"讓我先了解一下你們的需求。"

Sarah 開始描述他們的產品：

"我們要做一個智慧照明系統，包括：
- **20-30 個智慧燈泡**（分佈在整個房子）
- **1 個中央控制器**（連接到 WiFi，可以透過手機 App 控制）
- **3-5 個牆壁開關**（可以直接控制燈泡）

需求是：
- 使用者可以透過手機 App 控制所有燈泡
- 牆壁開關可以直接控制附近的燈泡（不需要經過中央控制器）
- 燈泡之間可以互相通訊（例如：一個燈泡的狀態改變，其他燈泡也跟著改變）
- 系統要穩定，不能經常斷線
- 成本要低，每個燈泡的無線模組成本要 < $2"

我聽完後，心裡開始分析。

這個應用場景有幾個關鍵特點：

1. **裝置數量多**（20-30 個燈泡）
2. **需要 Mesh Network**（燈泡之間互相通訊）
3. **不需要與手機直接連線**（只需要與中央控制器連線）
4. **成本敏感**（< $2/模組）

"根據你們的需求，我建議使用 **Zigbee**，"我說。

"為什麼不用 Bluetooth？"Sarah 問，"Bluetooth 不是更普及嗎？"

"Bluetooth 確實更普及，"我說，"但 Zigbee 更適合你們的應用場景。讓我解釋一下。"

我打開筆記本，開始畫圖。

"Zigbee 和 Bluetooth 都是基於 **IEEE 802.15.4** 的無線協定，都工作在 2.4 GHz 頻段。但它們的設計理念不同：

- **Bluetooth（特別是 BLE）**：設計用於 **點對點** 通訊，例如：手機連接耳機、手機連接智慧手環。雖然 BLE 5.0 開始支援 Mesh，但當時（2014 年）還沒有。

- **Zigbee**：設計用於 **Mesh Network**，適合大量裝置互相通訊的場景，例如：智慧家居、工業 IoT。"

"那 Zigbee 的優勢是什麼？"Sarah 問。

"主要有三個優勢，"我說：

"1. **Mesh Network**：Zigbee 原生支援 Mesh Network。每個裝置都可以作為 Router，轉發其他裝置的訊息。這樣即使某個裝置故障，訊息也可以透過其他路徑傳遞。

2. **大量裝置**：Zigbee 可以支援 **65,000 個裝置** 在同一個網路中。BLE（2014 年）只能支援 **1 對 1** 或 **1 對多** 的連線。

3. **低功耗**：Zigbee 的功耗比 BLE 稍高，但對於智慧燈泡來說不是問題（因為燈泡有電源供應）。"

"那 Bluetooth 的優勢呢？"Sarah 問。

"Bluetooth 的優勢是 **普及性** 和 **手機直連**，"我說，"如果你們的產品需要直接與手機連線（例如：智慧手環、無線耳機），那 Bluetooth 是更好的選擇。但對於智慧照明系統，Zigbee 更適合。"

Sarah 點點頭："我明白了。那 Zigbee 的成本呢？"

"Zigbee 模組的成本大約 **$1.5-$2**，"我說，"符合你們的預算。而且 Zigbee 的晶片廠商很多（例如：Texas Instruments, Silicon Labs, NXP），競爭激烈，價格會持續下降。"

"好，那我們就用 Zigbee，"Sarah 說，"你能幫我們設計嗎？"

"當然，"我說，"我會準備一份詳細的技術方案。"

---

掛上電話後，我開始準備技術方案。我需要深入比較 Zigbee 和 Bluetooth 的技術細節，才能給客戶一個完整的建議。

這次經歷讓我深刻理解了 **協定選擇** 的重要性。選對協定，可以事半功倍；選錯協定，可能會導致專案失敗。

---

## Zigbee 協定概覽

### Zigbee 是什麼？

**Zigbee** 是一個基於 **IEEE 802.15.4** 的無線通訊協定，專為 **低功耗、低成本、低資料速率** 的應用設計。

**Zigbee 的特點**：

- **工作頻段**：2.4 GHz（全球）、915 MHz（美國）、868 MHz（歐洲）
- **資料速率**：250 kbps（2.4 GHz）、40 kbps（915 MHz）、20 kbps（868 MHz）
- **通訊距離**：10-100 m（視環境而定）
- **網路拓撲**：Star、Tree、Mesh
- **裝置數量**：最多 65,000 個裝置
- **功耗**：低功耗（Sleep 模式 < 1 μA）

**Zigbee 的應用場景**：

- 智慧家居（智慧燈泡、智慧插座、智慧門鎖）
- 工業 IoT（感測器網路、設備監控）
- 醫療 IoT（病患監測、醫療設備）
- 農業 IoT（土壤監測、灌溉控制）

---

### Zigbee 協定堆疊

Zigbee 協定堆疊分為四層：

```
┌─────────────────────────────────────────────────────────┐
│  Zigbee 協定堆疊                                          │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─────────────────────────────────────────────┐         │
│  │  Application Layer (APL)                    │         │
│  │  - Application Framework                    │         │
│  │  - Zigbee Device Objects (ZDO)              │         │
│  │  - Application Support Sub-layer (APS)      │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                         │
│  ┌──────────────┴──────────────────────────────┐         │
│  │  Network Layer (NWK)                        │         │
│  │  - Routing                                  │         │
│  │  - Network Formation                        │         │
│  │  - Address Assignment                       │         │
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
│  │  - 2.4 GHz / 915 MHz / 868 MHz              │         │
│  │  - DSSS Modulation                          │         │
│  │  - Channel Selection                        │         │
│  └─────────────────────────────────────────────┘         │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**各層功能**：

1. **PHY Layer**：
   - 負責無線訊號的發送和接收
   - 使用 DSSS (Direct Sequence Spread Spectrum) 調變
   - 2.4 GHz 頻段有 16 個 Channel（Channel 11-26）

2. **MAC Layer**：
   - 負責媒體存取控制
   - 使用 CSMA/CA (Carrier Sense Multiple Access with Collision Avoidance)
   - 提供 ACK 機制確保資料傳輸可靠

3. **Network Layer**：
   - 負責網路路由和拓撲管理
   - 支援 Star、Tree、Mesh 拓撲
   - 自動路由選擇（AODV - Ad hoc On-Demand Distance Vector）

4. **Application Layer**：
   - 負責應用邏輯
   - Zigbee Device Objects (ZDO) 管理裝置的加入、離開、綁定
   - Application Support Sub-layer (APS) 提供資料傳輸服務

### Zigbee 網路拓撲

Zigbee 支援三種網路拓撲：

#### 1. Star Topology（星狀拓撲）

```
┌─────────────────────────────────────────────────────────┐
│  Star Topology                                            │
├─────────────────────────────────────────────────────────┤
│                                                           │
│                  ┌──────────┐                            │
│                  │Coordinator│                           │
│                  └─────┬────┘                            │
│                        │                                  │
│          ┌─────────────┼─────────────┐                   │
│          │             │             │                   │
│     ┌────┴───┐    ┌───┴────┐   ┌───┴────┐              │
│     │End Dev │    │End Dev │   │End Dev │              │
│     └────────┘    └────────┘   └────────┘              │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**特點**：

- 所有裝置都直接連接到 Coordinator
- 簡單、易於管理
- 但 Coordinator 故障會導致整個網路癱瘓

#### 2. Tree Topology（樹狀拓撲）

```
┌─────────────────────────────────────────────────────────┐
│  Tree Topology                                            │
├─────────────────────────────────────────────────────────┤
│                                                           │
│                  ┌──────────┐                            │
│                  │Coordinator│                           │
│                  └─────┬────┘                            │
│                        │                                  │
│          ┌─────────────┼─────────────┐                   │
│          │             │             │                   │
│     ┌────┴───┐    ┌───┴────┐   ┌───┴────┐              │
│     │Router  │    │Router  │   │End Dev │              │
│     └────┬───┘    └───┬────┘   └────────┘              │
│          │            │                                   │
│     ┌────┴───┐   ┌───┴────┐                             │
│     │End Dev │   │End Dev │                             │
│     └────────┘   └────────┘                             │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**特點**：

- Router 可以轉發訊息
- 擴展性好，可以支援更多裝置
- 但路由路徑固定，不夠靈活

#### 3. Mesh Topology（網狀拓撲）

```
┌─────────────────────────────────────────────────────────┐
│  Mesh Topology                                            │
├─────────────────────────────────────────────────────────┤
│                                                           │
│                  ┌──────────┐                            │
│                  │Coordinator│                           │
│                  └─────┬────┘                            │
│                        │                                  │
│          ┌─────────────┼─────────────┐                   │
│          │             │             │                   │
│     ┌────┴───┐    ┌───┴────┐   ┌───┴────┐              │
│     │Router  │◄───┤Router  │───┤Router  │              │
│     └────┬───┘    └───┬────┘   └───┬────┘              │
│          │            │            │                     │
│          │       ┌────┴────┐       │                     │
│          └───────┤End Dev  │───────┘                     │
│                  └─────────┘                             │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**特點**：

- Router 之間可以互相連接
- 自動路由選擇，找到最佳路徑
- 高可靠性：即使某個 Router 故障，訊息也可以透過其他路徑傳遞
- **最適合智慧家居應用**

---

### Zigbee 裝置類型

Zigbee 網路中有三種裝置類型：

1. **Coordinator（協調器）**：
   - 每個 Zigbee 網路只有一個 Coordinator
   - 負責建立網路、分配地址、管理安全金鑰
   - 通常是中央控制器（例如：智慧家居的 Hub）

2. **Router（路由器）**：
   - 可以轉發訊息
   - 可以讓 End Device 加入網路
   - 通常是有電源供應的裝置（例如：智慧燈泡、智慧插座）

3. **End Device（終端裝置）**：
   - 不能轉發訊息
   - 只能與 Parent（Coordinator 或 Router）通訊
   - 可以進入 Sleep 模式節省功耗
   - 通常是電池供電的裝置（例如：感測器、遙控器）

---

## BLE vs Zigbee 技術對比

現在讓我們深入比較 BLE 和 Zigbee 的技術細節：

### 1. 協定堆疊對比

| 層級 | BLE | Zigbee |
|------|-----|--------|
| **Application** | GATT Profiles | Zigbee Clusters |
| **Transport** | ATT, GATT | APS |
| **Network** | GAP | NWK (Routing) |
| **Link** | L2CAP | - |
| **MAC** | Link Layer | IEEE 802.15.4 MAC |
| **PHY** | BLE PHY (1M, 2M, Coded) | IEEE 802.15.4 PHY |

---

### 2. PHY 層對比

| 特性 | BLE | Zigbee |
|------|-----|--------|
| **頻段** | 2.4 GHz | 2.4 GHz / 915 MHz / 868 MHz |
| **Channel 數量** | 40 (3 Adv + 37 Data) | 16 (2.4 GHz) |
| **Channel 寬度** | 2 MHz | 5 MHz |
| **調變方式** | GFSK | O-QPSK (2.4 GHz) |
| **資料速率** | 1 Mbps / 2 Mbps / 125/500 kbps | 250 kbps |
| **TX Power** | -20 到 +10 dBm | -3 到 +20 dBm |
| **RX Sensitivity** | -70 dBm (1M PHY) | -85 dBm |

**分析**：

- **BLE 的資料速率更高**（1-2 Mbps vs 250 kbps），適合傳輸較大的資料
- **Zigbee 的 RX Sensitivity 更好**（-85 dBm vs -70 dBm），通訊距離更遠
- **Zigbee 支援多個頻段**，可以避開 2.4 GHz 的干擾

---

### 3. 網路拓撲對比

| 特性 | BLE | Zigbee |
|------|-----|--------|
| **拓撲** | Star (Point-to-Point) | Star / Tree / Mesh |
| **Mesh 支援** | BLE Mesh (2017 年後) | 原生支援 |
| **最大裝置數** | 7 (Piconet) | 65,000 |
| **路由** | 不支援 | AODV 自動路由 |
| **多跳** | 不支援（BLE Mesh 支援） | 支援 |

**分析**：

- **Zigbee 原生支援 Mesh Network**，適合大量裝置互相通訊
- **BLE 主要用於點對點通訊**（例如：手機連接耳機）
- **BLE Mesh**（2017 年發布）讓 BLE 也可以支援 Mesh，但當時（2014 年）還沒有

---

### 4. 功耗對比

| 模式 | BLE | Zigbee |
|------|-----|--------|
| **TX (0 dBm)** | 10-15 mA | 15-20 mA |
| **RX** | 10-15 mA | 15-20 mA |
| **Sleep** | 0.5-1 μA | 0.5-1 μA |
| **Advertising** | 50-200 μA (平均) | - |
| **Connected (100 ms interval)** | 50-100 μA (平均) | - |

**分析**：

- **BLE 的功耗稍低**，特別是在 Advertising 和 Connected 模式
- **Zigbee 的功耗稍高**，但對於有電源供應的裝置（例如：智慧燈泡）不是問題
- **兩者的 Sleep 功耗都很低**（< 1 μA），適合電池供電的裝置

---

### 5. 安全性對比

| 特性 | BLE | Zigbee |
|------|-----|--------|
| **加密演算法** | AES-128 CCM | AES-128 CCM |
| **金鑰管理** | SMP (Security Manager Protocol) | Trust Center |
| **Pairing 方式** | Just Works / Passkey / Numeric Comparison | Install Code / Default Link Key |
| **網路金鑰** | 不支援 | Network Key (所有裝置共享) |
| **Link 金鑰** | Link Key (每個連線獨立) | Link Key (每個裝置獨立) |

**分析**：

- **兩者都使用 AES-128 加密**，安全性相當
- **Zigbee 使用 Network Key**，所有裝置共享同一個金鑰，方便管理
- **BLE 使用 Link Key**，每個連線獨立，安全性更高

---

### 6. 成本對比

| 項目 | BLE | Zigbee |
|------|-----|--------|
| **晶片成本** | $0.5-$1.5 | $1.0-$2.0 |
| **認證費用** | $4,000-$8,000 | $3,000-$5,000 |
| **開發難度** | 中等 | 較高 |
| **生態系統** | 非常成熟（手機、電腦都支援） | 較小（需要專用 Hub） |

**分析**：

- **BLE 的晶片成本稍低**，但差異不大
- **BLE 的生態系統更成熟**，手機、電腦都原生支援
- **Zigbee 需要專用 Hub**，增加了系統成本

---

## 應用場景分析

### 場景 1：智慧手環（穿戴式裝置）

**需求**：

- 與手機直接連線
- 傳輸心率、步數等資料
- 電池供電，需要低功耗
- 成本敏感

**推薦協定**：**BLE**

**原因**：

- ✅ 手機原生支援 BLE，不需要額外的 Hub
- ✅ BLE 功耗低，適合電池供電
- ✅ BLE 資料速率高，可以快速傳輸資料
- ✅ BLE 晶片成本低

---

### 場景 2：智慧照明系統

**需求**：

- 20-30 個智慧燈泡
- 燈泡之間互相通訊（Mesh Network）
- 中央控制器連接到 WiFi
- 成本敏感

**推薦協定**：**Zigbee**

**原因**：

- ✅ Zigbee 原生支援 Mesh Network
- ✅ Zigbee 可以支援大量裝置（65,000 個）
- ✅ Zigbee 的通訊距離更遠（RX Sensitivity -85 dBm）
- ✅ 燈泡有電源供應，功耗不是問題

---

### 場景 3：智慧門鎖

**需求**：

- 與手機直接連線（開鎖）
- 電池供電，需要低功耗
- 安全性要求高

**推薦協定**：**BLE**

**原因**：

- ✅ 手機原生支援 BLE，使用方便
- ✅ BLE 功耗低，電池可以用 1-2 年
- ✅ BLE 的 Link Key 機制安全性高
- ✅ 不需要 Mesh Network

---

### 場景 4：工業感測器網路

**需求**：

- 100+ 個感測器
- 感測器之間互相通訊
- 需要高可靠性（Mesh Network）
- 通訊距離遠（工廠環境）

**推薦協定**：**Zigbee**

**原因**：

- ✅ Zigbee 支援大量裝置
- ✅ Zigbee 的 Mesh Network 可靠性高
- ✅ Zigbee 的通訊距離更遠
- ✅ Zigbee 支援 915 MHz 頻段，避開 2.4 GHz 的干擾

---

## 為什麼 BLE 在消費性 IoT 勝出？

雖然 Zigbee 在技術上有很多優勢（Mesh Network、大量裝置、通訊距離），但在消費性 IoT 市場，**BLE 逐漸成為主流**。

**原因**：

### 1. 手機原生支援

**BLE**：

- ✅ 所有智慧手機都原生支援 BLE
- ✅ 使用者可以直接用手機控制裝置，不需要額外的 Hub
- ✅ 開發 App 容易（iOS 和 Android 都有完整的 BLE API）

**Zigbee**：

- ❌ 手機不支援 Zigbee
- ❌ 需要專用的 Hub（例如：Philips Hue Bridge）
- ❌ 增加了系統成本和複雜度

---

### 2. 生態系統成熟

**BLE**：

- ✅ 晶片廠商多（Nordic, TI, Cypress, Dialog, Qualcomm）
- ✅ 開發工具完善（SDK, IDE, Debugger）
- ✅ 社群活躍，資源豐富

**Zigbee**：

- ✅ 晶片廠商也不少（TI, Silicon Labs, NXP）
- ❌ 但生態系統相對較小
- ❌ 開發難度較高

---

### 3. BLE Mesh 的出現（2017 年）

2017 年，Bluetooth SIG 發布了 **BLE Mesh** 規範，讓 BLE 也可以支援 Mesh Network。

**BLE Mesh 的優勢**：

- ✅ 結合了 BLE 的普及性和 Mesh Network 的可靠性
- ✅ 可以支援大量裝置（理論上無限制）
- ✅ 與現有的 BLE 裝置相容

**BLE Mesh 的應用**：

- 智慧照明（例如：Bluetooth Mesh Lighting）
- 智慧建築（例如：Sensor Network）
- 工業 IoT

---

### 4. 成本優勢

**BLE**：

- ✅ 晶片成本低（$0.5-$1.5）
- ✅ 不需要專用 Hub，降低系統成本
- ✅ 開發成本低（工具和資源豐富）

**Zigbee**：

- ❌ 晶片成本稍高（$1.0-$2.0）
- ❌ 需要專用 Hub，增加系統成本
- ❌ 開發成本較高

---

## 真實案例：智慧照明系統的協定選擇

回到文章開頭的故事。我為 Sarah 的智慧照明系統設計了一個基於 Zigbee 的方案：

**系統架構**：

```
┌─────────────────────────────────────────────────────────┐
│  智慧照明系統架構（Zigbee）                               │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────┐                                        │
│  │  手機 App    │                                        │
│  └──────┬───────┘                                        │
│         │ WiFi                                            │
│  ┌──────┴───────┐                                        │
│  │ 中央控制器   │ (Zigbee Coordinator + WiFi)           │
│  │ (Hub)        │                                        │
│  └──────┬───────┘                                        │
│         │ Zigbee Mesh                                     │
│         │                                                 │
│    ┌────┼────┬────┬────┬────┐                           │
│    │    │    │    │    │    │                           │
│  ┌─┴─┐┌─┴─┐┌─┴─┐┌─┴─┐┌─┴─┐┌─┴─┐                       │
│  │燈1││燈2││燈3││燈4││燈5││開關│                       │
│  └───┘└───┘└───┘└───┘└───┘└───┘                       │
│  (Router) (Router) (Router) (End Device)                │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**設計要點**：

1. **中央控制器**：
   - 使用 Zigbee Coordinator + WiFi 模組
   - 負責建立 Zigbee 網路
   - 連接到 WiFi，接收手機 App 的控制命令

2. **智慧燈泡**：
   - 使用 Zigbee Router
   - 可以轉發訊息，形成 Mesh Network
   - 有電源供應，不需要考慮功耗

3. **牆壁開關**：
   - 使用 Zigbee End Device
   - 電池供電，進入 Sleep 模式節省功耗
   - 按下按鈕時，發送控制命令到燈泡

---

### 實作細節

#### 1. Zigbee 網路建立

**Coordinator 初始化**：

```c
// Zigbee Coordinator 初始化
void zigbee_coordinator_init(void) {
    // 1. 設定 PAN ID（Personal Area Network ID）
    uint16_t pan_id = 0x1234;
    zigbee_set_pan_id(pan_id);

    // 2. 設定 Channel（使用 Channel 15，避開 WiFi）
    uint8_t channel = 15;
    zigbee_set_channel(channel);

    // 3. 設定 Network Key（128-bit AES 金鑰）
    uint8_t network_key[16] = {
        0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08,
        0x09, 0x0A, 0x0B, 0x0C, 0x0D, 0x0E, 0x0F, 0x10
    };
    zigbee_set_network_key(network_key);

    // 4. 啟動網路
    zigbee_start_network();

    printf("Zigbee Coordinator started\n");
    printf("PAN ID: 0x%04X\n", pan_id);
    printf("Channel: %d\n", channel);
}
```

**Router 加入網路**：

```c
// Zigbee Router 加入網路
void zigbee_router_join(void) {
    // 1. 掃描可用的網路
    zigbee_scan_networks();

    // 2. 選擇 PAN ID = 0x1234 的網路
    uint16_t target_pan_id = 0x1234;
    zigbee_join_network(target_pan_id);

    // 3. 等待加入成功
    while (!zigbee_is_joined()) {
        delay_ms(100);
    }

    // 4. 取得分配的 Short Address
    uint16_t short_addr = zigbee_get_short_address();
    printf("Joined network, Short Address: 0x%04X\n", short_addr);

    // 5. 允許其他裝置加入（Router 功能）
    zigbee_permit_join(true);
}
```

---

#### 2. 燈泡控制

**開關燈泡**：

```c
// 開關燈泡（使用 Zigbee On/Off Cluster）
void zigbee_light_control(uint16_t dest_addr, bool on) {
    // 1. 建立 ZCL (Zigbee Cluster Library) Frame
    uint8_t zcl_frame[10];
    zcl_frame[0] = 0x01;  // Frame Control: Cluster-specific command
    zcl_frame[1] = 0x00;  // Transaction Sequence Number
    zcl_frame[2] = on ? 0x01 : 0x00;  // Command: On (0x01) or Off (0x00)

    // 2. 發送到目標裝置
    zigbee_send_zcl_command(
        dest_addr,           // Destination Address
        0x0006,              // Cluster ID: On/Off
        zcl_frame,           // ZCL Frame
        3                    // Frame Length
    );

    printf("Light %s sent to 0x%04X\n", on ? "ON" : "OFF", dest_addr);
}

// 調整燈泡亮度（使用 Zigbee Level Control Cluster）
void zigbee_light_set_level(uint16_t dest_addr, uint8_t level) {
    // 1. 建立 ZCL Frame
    uint8_t zcl_frame[10];
    zcl_frame[0] = 0x01;  // Frame Control
    zcl_frame[1] = 0x00;  // Transaction Sequence Number
    zcl_frame[2] = 0x04;  // Command: Move to Level
    zcl_frame[3] = level; // Level (0-255)
    zcl_frame[4] = 0x00;  // Transition Time (LSB)
    zcl_frame[5] = 0x00;  // Transition Time (MSB)

    // 2. 發送到目標裝置
    zigbee_send_zcl_command(
        dest_addr,           // Destination Address
        0x0008,              // Cluster ID: Level Control
        zcl_frame,           // ZCL Frame
        6                    // Frame Length
    );

    printf("Light level %d sent to 0x%04X\n", level, dest_addr);
}
```

---

#### 3. 牆壁開關實作

**按鈕事件處理**：

```c
// 牆壁開關（End Device）
void zigbee_switch_init(void) {
    // 1. 加入網路
    zigbee_router_join();  // 使用相同的加入流程

    // 2. 設定為 End Device（不轉發訊息）
    zigbee_set_device_type(ZIGBEE_END_DEVICE);

    // 3. 設定 Sleep 模式（節省功耗）
    zigbee_enable_sleep_mode(true);

    // 4. 綁定到燈泡（Binding）
    uint16_t light_addr = 0x0001;  // 燈泡的 Short Address
    zigbee_bind_device(light_addr, 0x0006);  // Bind to On/Off Cluster

    printf("Switch initialized\n");
}

// 按鈕按下事件
void button_pressed_handler(void) {
    // 1. 喚醒裝置
    zigbee_wake_up();

    // 2. 切換燈泡狀態
    static bool light_on = false;
    light_on = !light_on;

    // 3. 發送控制命令
    uint16_t light_addr = 0x0001;
    zigbee_light_control(light_addr, light_on);

    // 4. 等待 ACK
    delay_ms(100);

    // 5. 進入 Sleep 模式
    zigbee_sleep();

    printf("Button pressed, light %s\n", light_on ? "ON" : "OFF");
}
```

---

### 實作結果

**測試結果**：

- ✅ **系統穩定**：30 個燈泡都能正常通訊，沒有斷線問題
- ✅ **Mesh Network 可靠性高**：即使某個燈泡故障，其他燈泡仍能正常工作
- ✅ **成本符合預算**：每個燈泡的 Zigbee 模組成本 $1.8
- ✅ **功耗符合要求**：牆壁開關的電池可以用 2 年以上
- ✅ **客戶非常滿意**

**遇到的問題**：

1. **Channel 選擇**：
   - 問題：Zigbee 和 WiFi 都工作在 2.4 GHz，會互相干擾
   - 解決：使用 Channel 15（2.425 GHz），避開 WiFi Channel 1, 6, 11

2. **Mesh 路由延遲**：
   - 問題：多跳路由會增加延遲（每跳約 50-100 ms）
   - 解決：優化網路拓撲，減少跳數（最多 3 跳）

3. **End Device Sleep**：
   - 問題：End Device 進入 Sleep 後，無法接收訊息
   - 解決：使用 Polling 機制，定期喚醒檢查是否有訊息

---

### 如果是現在（2025 年）

如果是現在（2025 年），我可能會推薦使用 **BLE Mesh**：

**原因**：

- ✅ BLE Mesh 已經成熟（2017 年發布）
- ✅ 晶片成本更低（$0.5-$1.0）
- ✅ 生態系統更成熟
- ✅ 可以直接用手機控制（不需要專用 Hub）

**BLE Mesh 架構**：

```
┌─────────────────────────────────────────────────────────┐
│  智慧照明系統架構（BLE Mesh）                             │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────┐                                        │
│  │  手機 App    │ (直接控制，不需要 Hub)                │
│  └──────┬───────┘                                        │
│         │ BLE Mesh                                        │
│         │                                                 │
│    ┌────┼────┬────┬────┬────┐                           │
│    │    │    │    │    │    │                           │
│  ┌─┴─┐┌─┴─┐┌─┴─┐┌─┴─┐┌─┴─┐┌─┴─┐                       │
│  │燈1││燈2││燈3││燈4││燈5││開關│                       │
│  └───┘└───┘└───┘└───┘└───┘└───┘                       │
│  (Relay) (Relay) (Relay) (Low Power Node)               │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**優勢**：

- ✅ 不需要專用 Hub，降低系統成本
- ✅ 手機可以直接控制燈泡
- ✅ BLE Mesh 的 Flooding 機制可靠性高
- ✅ 與現有的 BLE 裝置相容

---

## 實務建議：如何選擇 BLE 或 Zigbee？

基於多年的 IoT 開發經驗，我總結了以下的協定選擇指南：

### 選擇 BLE 的場景

✅ **穿戴式裝置**（智慧手環、智慧手錶、無線耳機）

- 需要與手機直接連線
- 電池供電，功耗敏感
- 資料速率要求高（心率、音訊）

✅ **智慧門鎖、智慧家電**

- 需要與手機直接連線
- 不需要 Mesh Network
- 安全性要求高

✅ **Beacon 應用**（室內定位、廣告推播）

- 單向廣播
- 不需要雙向通訊
- 成本敏感

✅ **醫療裝置**（血糖儀、血壓計）

- 需要與手機直接連線
- 資料量小
- 功耗敏感

---

### 選擇 Zigbee 的場景

✅ **智慧照明系統**（大量燈泡）

- 需要 Mesh Network
- 裝置數量多（20+ 個）
- 有電源供應

✅ **工業感測器網路**

- 需要高可靠性
- 通訊距離遠
- 可以使用 915 MHz 頻段避開干擾

✅ **智慧建築**（HVAC 控制、能源管理）

- 需要 Mesh Network
- 裝置數量多
- 有電源供應

✅ **農業 IoT**（土壤監測、灌溉控制）

- 通訊距離遠
- 需要 Mesh Network
- 環境惡劣（2.4 GHz 干擾多）

---

### 選擇 BLE Mesh 的場景（2017 年後）

✅ **智慧照明系統**（新專案）

- 需要 Mesh Network
- 希望用手機直接控制（不需要 Hub）
- 成本敏感

✅ **智慧建築**（新專案）

- 需要 Mesh Network
- 希望與現有的 BLE 裝置相容
- 生態系統成熟

---

### 決策流程圖

```
┌─────────────────────────────────────────────────────────┐
│  BLE vs Zigbee 決策流程圖                                 │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  需要與手機直接連線？                                     │
│         │                                                 │
│         ├─ Yes ─► 需要 Mesh Network？                    │
│         │              │                                  │
│         │              ├─ Yes ─► BLE Mesh (2017+)        │
│         │              │                                  │
│         │              └─ No ──► BLE                      │
│         │                                                 │
│         └─ No ──► 需要 Mesh Network？                    │
│                        │                                  │
│                        ├─ Yes ─► 裝置數量 > 50？         │
│                        │              │                   │
│                        │              ├─ Yes ─► Zigbee   │
│                        │              │                   │
│                        │              └─ No ──► BLE Mesh │
│                        │                                  │
│                        └─ No ──► 通訊距離 > 50m？        │
│                                       │                   │
│                                       ├─ Yes ─► Zigbee   │
│                                       │                   │
│                                       └─ No ──► BLE      │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

---

### 常見錯誤

❌ **錯誤 1：盲目選擇 BLE**

- 問題：因為 BLE 普及，就選擇 BLE，忽略了 Mesh Network 的需求
- 後果：系統無法擴展，需要重新設計
- 建議：先分析需求，再選擇協定

❌ **錯誤 2：忽略 Channel 干擾**

- 問題：Zigbee 和 WiFi 都工作在 2.4 GHz，會互相干擾
- 後果：通訊不穩定，經常斷線
- 建議：使用 Channel 15-26，避開 WiFi Channel 1, 6, 11

❌ **錯誤 3：低估開發難度**

- 問題：認為 Zigbee 和 BLE 的開發難度相同
- 後果：專案延期，成本超支
- 建議：Zigbee 的開發難度較高，需要更多時間和資源

❌ **錯誤 4：忽略成本**

- 問題：只考慮晶片成本，忽略了 Hub 成本
- 後果：系統成本超出預算
- 建議：Zigbee 需要專用 Hub，要計入系統成本

---

### 實務檢查清單

**協定選擇檢查清單**：

- [ ] 是否需要與手機直接連線？
- [ ] 是否需要 Mesh Network？
- [ ] 裝置數量是多少？（< 10, 10-50, > 50）
- [ ] 通訊距離是多少？（< 10m, 10-50m, > 50m）
- [ ] 是否有電源供應？（電池 / 電源）
- [ ] 資料速率要求？（< 100 kbps, 100-500 kbps, > 500 kbps）
- [ ] 功耗要求？（< 100 μA, 100-500 μA, > 500 μA）
- [ ] 成本預算？（< $1, $1-$2, > $2）
- [ ] 開發時間？（< 3 個月, 3-6 個月, > 6 個月）
- [ ] 是否需要認證？（Bluetooth SIG / Zigbee Alliance）

**根據檢查清單結果**：

- **BLE**：手機直連、低功耗、成本低、開發時間短
- **Zigbee**：Mesh Network、大量裝置、通訊距離遠、有電源供應
- **BLE Mesh**：Mesh Network + 手機直連、成本低、生態系統成熟（2017 年後）

---

## 總結

Zigbee 和 BLE 都是優秀的無線協定，各有優缺點：

**Zigbee 的優勢**：

- ✅ 原生支援 Mesh Network
- ✅ 可以支援大量裝置（65,000 個）
- ✅ 通訊距離遠（RX Sensitivity -85 dBm）
- ✅ 支援多個頻段（2.4 GHz / 915 MHz / 868 MHz）

**Zigbee 的劣勢**：

- ❌ 手機不支援，需要專用 Hub
- ❌ 生態系統較小
- ❌ 開發難度較高
- ❌ 成本稍高

**BLE 的優勢**：

- ✅ 手機原生支援
- ✅ 生態系統成熟
- ✅ 開發容易
- ✅ 成本低

**BLE 的劣勢**：

- ❌ 不支援 Mesh Network（2017 年前）
- ❌ 裝置數量有限（7 個 Piconet）
- ❌ 通訊距離較短

**BLE Mesh（2017 年後）**：

- ✅ 結合了 BLE 的普及性和 Mesh Network 的可靠性
- ✅ 成為智慧照明、智慧建築的主流選擇

---

### 我的建議

基於多年的 IoT 開發經驗，我的建議是：

1. **2014-2017 年**：
   - 穿戴式裝置、智慧門鎖 → **BLE**
   - 智慧照明、工業 IoT → **Zigbee**

2. **2017 年後**：
   - 穿戴式裝置、智慧門鎖 → **BLE**
   - 智慧照明、智慧建築 → **BLE Mesh**
   - 工業 IoT、農業 IoT → **Zigbee**（特別是需要 915 MHz 頻段的場景）

3. **2025 年（現在）**：
   - 大部分消費性 IoT → **BLE / BLE Mesh**
   - 工業 IoT、特殊場景 → **Zigbee**

---

### 延伸閱讀

回顧這次的協定選擇經歷，我深刻體會到：**沒有最好的協定，只有最適合的協定**。

選擇協定時，要考慮：

- 應用場景（穿戴式 / 智慧家居 / 工業 IoT）
- 技術需求（Mesh Network / 通訊距離 / 功耗）
- 成本預算（晶片成本 / Hub 成本 / 開發成本）
- 時間限制（開發時間 / 認證時間）

只有綜合考慮這些因素，才能做出正確的決策。

在下一篇文章中，我會介紹 **RF4CE, Thread, 與 Matter** 等其他 IoT 協定，以及它們與 BLE、Zigbee 的關係。

---

## 參考資料

- Zigbee Alliance: <https://zigbeealliance.org/>
- Bluetooth SIG: <https://www.bluetooth.com/>
- IEEE 802.15.4 Standard: <https://standards.ieee.org/standard/802_15_4-2020.html>
- BLE Mesh Specification: <https://www.bluetooth.com/specifications/mesh-specifications/>
- Zigbee Cluster Library: <https://zigbeealliance.org/wp-content/uploads/2019/12/07-5123-06-zigbee-cluster-library-specification.pdf>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

您可以自由地：

- **分享** — 以任何媒介或格式重製及散布本素材
- **修改** — 重混、轉換本素材、及依本素材建立新素材

但您必須遵守以下條款：

- **姓名標示** — 您必須給予適當表彰、提供指向本授權條款的連結，以及指出（本作品的原始版本）是否已被變更。您可以任何合理的方式為前述表彰，但不得以任何方式暗示授權人為您或您的使用方式背書。
