---
author: Danny Jiang
date: 2025-12-11
---

# I2C, UART, GPIO - IoT 晶片的基礎接口

## 前言：一個週五晚上的 I2C 除錯馬拉松

2014 年 10 月的某個週五下午 5 點，我正準備下班。這週的工作很順利，智慧手環的新版韌體已經完成了 90%，只剩下最後的整合測試。

「應該沒問題吧。」我心想，「三個 Sensor 都已經單獨測試過了，整合起來應該很簡單。」

我信心滿滿地燒錄了韌體，按下電源鍵。手環的螢幕亮了起來，顯示「正在初始化...」，然後就卡住了。

「怎麼回事？」我皺起眉頭。

我打開 UART 除錯介面，看到一連串的錯誤訊息：

```
[00:00.123] [INFO] System boot...
[00:00.234] [INFO] Initializing I2C bus...
[00:00.345] [INFO] Scanning I2C devices...
[00:00.456] [ERROR] I2C: Device 0x68 not responding
[00:00.567] [ERROR] I2C: SCL stuck low
[00:00.678] [ERROR] I2C: Bus timeout
[00:00.789] [ERROR] System initialization failed!
```

我的心一沉。「又是 I2C 問題。」

這個智慧手環需要整合三個 Sensor：

1. **MPU6050 加速度計/陀螺儀**（地址 0x68）- 用於計步和手勢識別
2. **MAX30102 心率感測器**（地址 0x57）- 用於心率和血氧監測
3. **APDS-9960 環境光/接近感測器**（地址 0x39）- 用於自動調整螢幕亮度和手勢控制

所有的 Sensor 都使用 I2C 接口，看起來很簡單——只需要兩條線（SDA 和 SCL），就能連接多個裝置。但現在，整個 I2C Bus 都卡住了。

我看了一眼時鐘，下午 5:15。「看來今晚要加班了。」

---

### 第一個問題：SCL Stuck Low

我拿起示波器，開始測量 I2C 的 SDA 和 SCL 訊號。

果然，**SCL 被拉低了**，整個 I2C Bus 都卡住了。但是是哪個裝置造成的？加速度計？心率感測器？還是環境光感測器？

我逐一斷開每個 Sensor 的電源，測試 I2C Bus 的狀態：

1. **斷開 MPU6050**：SCL 仍然是 Low
2. **斷開 MAX30102**：SCL 仍然是 Low
3. **斷開 APDS-9960**：SCL 恢復正常！

「原來是環境光感測器的問題。」我鬆了一口氣。

但為什麼 APDS-9960 會拉低 SCL？我檢查了電路圖，發現 **Pull-up 電阻是 4.7kΩ**。

這時候，我的同事 Alex 走了過來。他是硬體工程師，對 I2C 的電氣特性非常熟悉。

「讓我看看你的電路圖。」他說。

我把電路圖調出來。Alex 看了一眼，立刻指出問題：

「你的 Pull-up 電阻太大了。4.7kΩ 對於 400kHz 的 I2C 來說太大了，應該用 2.2kΩ。」

「但是 datasheet 上說 4.7kΩ 就可以了啊。」我有點不服氣。

「那是針對 100kHz 的標準模式。」Alex 解釋道，「你用的是 Fast Mode（400kHz），需要更小的電阻。而且你有三個裝置掛在同一條 Bus 上，總電容更大，上升時間會更慢。」

他在白板上畫了一個公式：

```
t_rise = 0.8473 × R_pullup × C_bus

對於 Fast Mode (400kHz)，t_rise 必須 < 300ns

假設 C_bus = 150pF（3 個裝置 + PCB trace）
R_pullup < 300ns / (0.8473 × 150pF) = 2.36kΩ

所以應該選擇 2.2kΩ
```

我恍然大悟。I2C 的時序不只是軟體的問題，還涉及硬體的電氣特性。

「但是現在已經是晚上 6 點了，硬體實驗室已經關門了，我去哪裡找 2.2kΩ 的電阻？」我有點沮喪。

「我桌上有一些樣品。」Alex 說，「我去拿給你。」

10 分鐘後，我拿著烙鐵，小心翼翼地更換 Pull-up 電阻。焊接完成後，我重新上電。

這次，I2C Bus 正常工作了！UART 輸出：

```
[00:00.123] [INFO] System boot...
[00:00.234] [INFO] Initializing I2C bus...
[00:00.345] [INFO] Scanning I2C devices...
[00:00.456] [INFO] Found device at 0x39 (APDS-9960)
[00:00.567] [INFO] Found device at 0x57 (MAX30102)
[00:00.678] [ERROR] I2C: Address conflict at 0x68!
[00:00.789] [ERROR] System initialization failed!
```

「什麼？地址衝突？」我仔細檢查了每個 Sensor 的 datasheet：

- MPU6050：地址 0x68（可以透過 AD0 pin 改為 0x69）
- MAX30102：地址 0x57
- APDS-9960：地址 0x39

「等等，我明明只有一個 0x68 的裝置啊？」

我突然想起來，**我在測試時連接了兩個 MPU6050**（一個在手環上，一個在測試板上），忘記拔掉測試板了！

我拔掉測試板，重新上電。這次，系統正常啟動了！

```
[00:00.123] [INFO] System boot...
[00:00.234] [INFO] Initializing I2C bus...
[00:00.345] [INFO] Scanning I2C devices...
[00:00.456] [INFO] Found device at 0x39 (APDS-9960)
[00:00.567] [INFO] Found device at 0x57 (MAX30102)
[00:00.678] [INFO] Found device at 0x68 (MPU6050)
[00:00.789] [INFO] All sensors initialized successfully!
[00:00.890] [INFO] System ready!
```

我看了一眼時鐘，晚上 8:30。「終於搞定了！」

但這次經驗讓我深刻體會到：**I2C、UART、GPIO 這些看似簡單的基礎接口，實際上充滿了細節和陷阱**。只有真正理解它們的原理和特性，才能在 IoT 專案中游刃有餘。

---

## I2C 協定深入解析

### 為什麼選擇 I2C？

在 IoT 裝置中，**I2C (Inter-Integrated Circuit)** 是連接 Sensor 的首選協定。為什麼？

**優點**：

1. **只需要兩條線**：SDA (Serial Data) 和 SCL (Serial Clock)
   - 節省 PCB 空間和 GPIO pins
   - 降低佈線複雜度

2. **支援多個裝置**：一條 Bus 可以連接多達 127 個裝置（7-bit 地址）
   - 不需要額外的 Chip Select 訊號
   - 動態添加/移除裝置

3. **硬體簡單**：只需要 Pull-up 電阻
   - 成本低
   - 易於實作

4. **功耗低**：適合電池供電的 IoT 裝置
   - Open-Drain 架構，只在傳輸時消耗電流
   - 支援 Clock Stretching（Slave 可以暫停傳輸）

5. **生態系統成熟**：幾乎所有的 Sensor 都支援 I2C
   - 加速度計、陀螺儀、磁力計
   - 溫度、濕度、氣壓感測器
   - 心率、血氧感測器
   - EEPROM、RTC

**缺點**：

1. **速度較慢**：最高 3.4 MHz（High Speed Mode），但實際很少使用
2. **距離限制**：通常 < 1 公尺（受電容影響）
3. **地址衝突**：同型號 Sensor 可能有相同地址
4. **時序敏感**：需要精確的 Pull-up 電阻和時序控制

---

### I2C 的基本原理

I2C 是一個 **Master-Slave** 架構的同步串列通訊協定：

```
┌─────────────────────────────────────────────────────────┐
│  I2C Bus 架構                                             │
├─────────────────────────────────────────────────────────┤
│                                                           │
│         VDD                                               │
│          │                                                │
│          ├─── R_pullup (2.2kΩ - 4.7kΩ)                   │
│          │                                                │
│  ┌───────┴───────┬───────────┬───────────┐               │
│  │               │           │           │               │
│  │ SDA           │ SDA       │ SDA       │ SDA           │
│  │ SCL           │ SCL       │ SCL       │ SCL           │
│  │               │           │           │               │
│ ┌┴──────┐   ┌───┴────┐ ┌───┴────┐ ┌───┴────┐           │
│ │Master │   │Slave 1 │ │Slave 2 │ │Slave 3 │           │
│ │(MCU)  │   │(0x68)  │ │(0x57)  │ │(0x39)  │           │
│ └───────┘   └────────┘ └────────┘ └────────┘           │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**角色定義**：

- **Master**：通常是 MCU 或 SoC
  - 產生 Clock (SCL)
  - 發起通訊（START condition）
  - 結束通訊（STOP condition）
  - 控制資料傳輸方向

- **Slave**：Sensor 或其他周邊裝置
  - 被動回應 Master 的請求
  - 有唯一的 7-bit 或 10-bit 地址
  - 可以使用 Clock Stretching 暫停傳輸

**訊號線**：

- **SDA (Serial Data)**：
  - 雙向資料線
  - **Open-Drain** 架構（需要 Pull-up 電阻）
  - 資料在 SCL 為 High 時穩定，在 SCL 為 Low 時可以改變

- **SCL (Serial Clock)**：
  - 時鐘線（由 Master 產生）
  - **Open-Drain** 架構（需要 Pull-up 電阻）
  - Slave 可以拉低 SCL 來暫停傳輸（Clock Stretching）

---

### I2C 傳輸協定詳解

#### 1. **START 和 STOP Condition**

**START Condition**：SDA 從 High 變 Low（SCL 保持 High）

```
SCL: ────────┐         ┌────────
             │         │
SDA: ────────┘         └────────
             ↑
           START
```

**STOP Condition**：SDA 從 Low 變 High（SCL 保持 High）

```
SCL: ────────┐         ┌────────
             │         │
SDA: ────────┘         └────────
                       ↑
                     STOP
```

#### 2. **資料傳輸格式**

**完整的 I2C 傳輸**：

```
START - [Slave Address (7-bit)] - [R/W bit] - ACK - [Data Byte] - ACK - ... - STOP
```

**範例 1**：寫入 MPU6050 的 PWR_MGMT_1 暫存器（喚醒裝置）

```
START - 0x68 - W - ACK - 0x6B - ACK - 0x00 - ACK - STOP

說明：
- 0x68: MPU6050 的 I2C 地址
- W: 寫入模式 (R/W bit = 0)
- 0x6B: PWR_MGMT_1 暫存器地址
- 0x00: 寫入的資料（清除 SLEEP bit）
```

**範例 2**：讀取 MPU6050 的加速度計資料（6 bytes）

```
START - 0x68 - W - ACK - 0x3B - ACK - RESTART - 0x68 - R - ACK - [ACCEL_XOUT_H] - ACK - [ACCEL_XOUT_L] - ACK - [ACCEL_YOUT_H] - ACK - [ACCEL_YOUT_L] - ACK - [ACCEL_ZOUT_H] - ACK - [ACCEL_ZOUT_L] - NACK - STOP

說明：
- 第一階段：寫入暫存器地址 0x3B
- RESTART: 重新開始（不發送 STOP）
- 第二階段：讀取 6 bytes 資料
- NACK: Master 發送 NACK 表示讀取結束
```

#### 3. **ACK 和 NACK**

**ACK (Acknowledge)**：

- Receiver 拉低 SDA（在第 9 個 Clock Pulse）
- 表示成功接收資料

**NACK (Not Acknowledge)**：

- Receiver 保持 SDA 為 High（在第 9 個 Clock Pulse）
- 表示：
  - 讀取結束（Master 發送 NACK）
  - 接收失敗（Slave 發送 NACK）
  - 地址不匹配（Slave 不回應）

```
資料傳輸時序：

SCL: ─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌───
      └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘
SDA: ──D7──D6──D5──D4──D3──D2──D1──D0──ACK──
      ↑                                 ↑
    資料位元                          ACK/NACK
```

---

### I2C 的速度模式

| 模式 | 速度 | t_rise (max) | 應用場景 | Pull-up 電阻 |
|------|------|--------------|----------|--------------|
| **Standard Mode** | 100 kHz | 1000 ns | 低速 Sensor（溫度、濕度） | 4.7kΩ |
| **Fast Mode** | 400 kHz | 300 ns | 加速度計、陀螺儀 | 2.2kΩ - 3.3kΩ |
| **Fast Mode Plus** | 1 MHz | 120 ns | 高速 Sensor、EEPROM | 1kΩ - 2.2kΩ |
| **High Speed Mode** | 3.4 MHz | 80 ns | 很少使用 | < 1kΩ |

**速度選擇建議**：

1. **Standard Mode (100 kHz)**：
   - 適合低速 Sensor（溫度、濕度、RTC）
   - 線長可以較長（< 1 公尺）
   - Pull-up 電阻 4.7kΩ

2. **Fast Mode (400 kHz)**：
   - 適合大多數 Sensor（加速度計、陀螺儀、心率感測器）
   - 線長應 < 30 cm
   - Pull-up 電阻 2.2kΩ - 3.3kΩ

3. **Fast Mode Plus (1 MHz)**：
   - 適合高速 Sensor 或 EEPROM
   - 線長應 < 10 cm
   - Pull-up 電阻 1kΩ - 2.2kΩ
   - 需要 Sensor 支援

### I2C 驅動實作（以 nRF52 為例）

#### 1. **初始化 I2C**

```c
// I2C 配置
#define SDA_PIN 26
#define SCL_PIN 27
#define I2C_FREQUENCY TWIM_FREQUENCY_FREQUENCY_K400  // 400 kHz

// I2C 初始化
void i2c_init(void) {
    // 1. 配置 GPIO 為 I2C 功能
    // SDA pin 配置
    NRF_GPIO->PIN_CNF[SDA_PIN] =
        (GPIO_PIN_CNF_DIR_Input << GPIO_PIN_CNF_DIR_Pos) |        // 輸入模式
        (GPIO_PIN_CNF_INPUT_Connect << GPIO_PIN_CNF_INPUT_Pos) |  // 連接輸入 buffer
        (GPIO_PIN_CNF_PULL_Pullup << GPIO_PIN_CNF_PULL_Pos) |     // 內部 Pull-up（可選）
        (GPIO_PIN_CNF_DRIVE_S0D1 << GPIO_PIN_CNF_DRIVE_Pos);      // Open-Drain 輸出

    // SCL pin 配置
    NRF_GPIO->PIN_CNF[SCL_PIN] =
        (GPIO_PIN_CNF_DIR_Input << GPIO_PIN_CNF_DIR_Pos) |
        (GPIO_PIN_CNF_INPUT_Connect << GPIO_PIN_CNF_INPUT_Pos) |
        (GPIO_PIN_CNF_PULL_Pullup << GPIO_PIN_CNF_PULL_Pos) |
        (GPIO_PIN_CNF_DRIVE_S0D1 << GPIO_PIN_CNF_DRIVE_Pos);

    // 2. 配置 I2C 模組（TWIM = Two-Wire Interface Master）
    NRF_TWIM0->PSEL.SDA = SDA_PIN;
    NRF_TWIM0->PSEL.SCL = SCL_PIN;
    NRF_TWIM0->FREQUENCY = I2C_FREQUENCY;

    // 3. 啟用 I2C 模組
    NRF_TWIM0->ENABLE = TWIM_ENABLE_ENABLE_Enabled;

    // 4. 清除錯誤狀態
    NRF_TWIM0->ERRORSRC = 0xFFFFFFFF;
}
```

**重點說明**：

1. **Open-Drain 輸出**：I2C 使用 Open-Drain 架構，需要外部 Pull-up 電阻
2. **內部 Pull-up**：nRF52 的內部 Pull-up 電阻約 13kΩ，通常不夠強，建議使用外部 Pull-up
3. **頻率設定**：400 kHz 是最常用的速度

---

#### 2. **寫入資料**

```c
// I2C 寫入（寫入暫存器）
bool i2c_write_reg(uint8_t slave_addr, uint8_t reg_addr, uint8_t data) {
    uint8_t tx_buf[2] = {reg_addr, data};

    // 1. 設定 Slave 地址
    NRF_TWIM0->ADDRESS = slave_addr;

    // 2. 設定傳輸資料
    NRF_TWIM0->TXD.PTR = (uint32_t)tx_buf;
    NRF_TWIM0->TXD.MAXCNT = 2;

    // 3. 清除事件
    NRF_TWIM0->EVENTS_STOPPED = 0;
    NRF_TWIM0->EVENTS_ERROR = 0;

    // 4. 啟動傳輸
    NRF_TWIM0->TASKS_STARTTX = 1;

    // 5. 等待傳輸完成
    while (!NRF_TWIM0->EVENTS_STOPPED && !NRF_TWIM0->EVENTS_ERROR);

    // 6. 檢查錯誤
    if (NRF_TWIM0->EVENTS_ERROR) {
        uint32_t error = NRF_TWIM0->ERRORSRC;
        NRF_TWIM0->ERRORSRC = 0xFFFFFFFF;  // 清除錯誤
        NRF_TWIM0->EVENTS_ERROR = 0;

        // 解析錯誤類型
        if (error & TWIM_ERRORSRC_ANACK_Msk) {
            // Address NACK: Slave 不回應地址
            return false;
        }
        if (error & TWIM_ERRORSRC_DNACK_Msk) {
            // Data NACK: Slave 不回應資料
            return false;
        }
        return false;
    }

    NRF_TWIM0->EVENTS_STOPPED = 0;
    return true;
}

// I2C 寫入多個 bytes
bool i2c_write_bytes(uint8_t slave_addr, uint8_t reg_addr, uint8_t *data, uint8_t len) {
    uint8_t tx_buf[len + 1];
    tx_buf[0] = reg_addr;
    memcpy(&tx_buf[1], data, len);

    NRF_TWIM0->ADDRESS = slave_addr;
    NRF_TWIM0->TXD.PTR = (uint32_t)tx_buf;
    NRF_TWIM0->TXD.MAXCNT = len + 1;

    NRF_TWIM0->EVENTS_STOPPED = 0;
    NRF_TWIM0->EVENTS_ERROR = 0;
    NRF_TWIM0->TASKS_STARTTX = 1;

    while (!NRF_TWIM0->EVENTS_STOPPED && !NRF_TWIM0->EVENTS_ERROR);

    if (NRF_TWIM0->EVENTS_ERROR) {
        NRF_TWIM0->ERRORSRC = 0xFFFFFFFF;
        NRF_TWIM0->EVENTS_ERROR = 0;
        return false;
    }

    NRF_TWIM0->EVENTS_STOPPED = 0;
    return true;
}
```

---

#### 3. **讀取資料**

```c
// I2C 讀取（讀取暫存器）
bool i2c_read_reg(uint8_t slave_addr, uint8_t reg_addr, uint8_t *data, uint8_t len) {
    // 1. 先寫入暫存器地址
    NRF_TWIM0->ADDRESS = slave_addr;
    NRF_TWIM0->TXD.PTR = (uint32_t)&reg_addr;
    NRF_TWIM0->TXD.MAXCNT = 1;

    // 2. 配置接收
    NRF_TWIM0->RXD.PTR = (uint32_t)data;
    NRF_TWIM0->RXD.MAXCNT = len;

    // 3. 設定 SHORTS：自動從 TX 切換到 RX（產生 RESTART condition）
    NRF_TWIM0->SHORTS = TWIM_SHORTS_LASTTX_STARTRX_Msk;

    // 4. 清除事件
    NRF_TWIM0->EVENTS_STOPPED = 0;
    NRF_TWIM0->EVENTS_ERROR = 0;

    // 5. 啟動傳輸
    NRF_TWIM0->TASKS_STARTTX = 1;

    // 6. 等待傳輸完成
    while (!NRF_TWIM0->EVENTS_STOPPED && !NRF_TWIM0->EVENTS_ERROR);

    // 7. 清除 SHORTS
    NRF_TWIM0->SHORTS = 0;

    // 8. 檢查錯誤
    if (NRF_TWIM0->EVENTS_ERROR) {
        NRF_TWIM0->ERRORSRC = 0xFFFFFFFF;
        NRF_TWIM0->EVENTS_ERROR = 0;
        return false;
    }

    NRF_TWIM0->EVENTS_STOPPED = 0;
    return true;
}
```

**SHORTS 機制說明**：

- `TWIM_SHORTS_LASTTX_STARTRX_Msk`：當 TX 完成後，自動啟動 RX
- 這樣可以產生 **RESTART condition**（不發送 STOP），符合 I2C 讀取協定

---

#### 4. **實際使用範例**

```c
// 初始化 MPU6050 加速度計
void mpu6050_init(void) {
    // 1. 喚醒裝置（清除 SLEEP bit）
    i2c_write_reg(0x68, 0x6B, 0x00);  // PWR_MGMT_1 = 0x00
    nrf_delay_ms(100);

    // 2. 配置加速度計範圍（±2g）
    i2c_write_reg(0x68, 0x1C, 0x00);  // ACCEL_CONFIG = 0x00

    // 3. 配置陀螺儀範圍（±250 deg/s）
    i2c_write_reg(0x68, 0x1B, 0x00);  // GYRO_CONFIG = 0x00

    // 4. 配置 Sample Rate（1 kHz）
    i2c_write_reg(0x68, 0x19, 0x07);  // SMPLRT_DIV = 7 (125 Hz)
}

// 讀取加速度計資料
void mpu6050_read_accel(int16_t *accel_x, int16_t *accel_y, int16_t *accel_z) {
    uint8_t data[6];

    // 讀取 6 bytes（ACCEL_XOUT_H/L, ACCEL_YOUT_H/L, ACCEL_ZOUT_H/L）
    if (i2c_read_reg(0x68, 0x3B, data, 6)) {
        *accel_x = (int16_t)((data[0] << 8) | data[1]);
        *accel_y = (int16_t)((data[2] << 8) | data[3]);
        *accel_z = (int16_t)((data[4] << 8) | data[5]);
    }
}
```

### I2C 的常見問題與解決方案

#### 1. **地址衝突**

**問題**：兩個裝置使用相同的 I2C 地址

**常見案例**：

- 兩個 MPU6050 都是 0x68
- 兩個 OLED Display 都是 0x3C
- 同型號 Sensor 的地址無法改變

**解決方案 1：使用 AD0/SA0 pin 改變地址**

很多 Sensor 提供一個 pin 來改變地址：

```c
// MPU6050 地址配置
// AD0 = GND: 地址 = 0x68
// AD0 = VDD: 地址 = 0x69

#define MPU6050_ADDR_1 0x68  // AD0 = GND
#define MPU6050_ADDR_2 0x69  // AD0 = VDD

// 初始化兩個 MPU6050
void dual_mpu6050_init(void) {
    // 初始化第一個 MPU6050
    i2c_write_reg(MPU6050_ADDR_1, 0x6B, 0x00);

    // 初始化第二個 MPU6050
    i2c_write_reg(MPU6050_ADDR_2, 0x6B, 0x00);
}
```

**解決方案 2：使用 I2C Multiplexer**

當 Sensor 無法改變地址時，使用 **TCA9548A I2C Multiplexer**：

```
┌─────────────────────────────────────────────────────────┐
│  I2C Multiplexer 架構                                     │
├─────────────────────────────────────────────────────────┤
│                                                           │
│         Master (MCU)                                      │
│              │                                            │
│              │ I2C Bus                                    │
│              │                                            │
│         ┌────┴────┐                                       │
│         │TCA9548A │                                       │
│         │  (0x70) │                                       │
│         └─┬─┬─┬─┬─┘                                       │
│           │ │ │ │                                         │
│     CH0   │ │ │ │   CH7                                   │
│     ┌─────┘ │ │ └─────┐                                   │
│     │       │ │       │                                   │
│  ┌──┴──┐ ┌─┴─┴──┐ ┌──┴──┐                               │
│  │0x68 │ │ 0x68 │ │0x68 │                               │
│  │MPU1 │ │ MPU2 │ │MPU3 │                               │
│  └─────┘ └──────┘ └─────┘                               │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**TCA9548A 驅動實作**：

```c
// TCA9548A I2C Multiplexer
#define MUX_ADDR 0x70

// 選擇 I2C Channel（0-7）
void i2c_mux_select_channel(uint8_t channel) {
    if (channel > 7) return;

    uint8_t channel_mask = 1 << channel;

    // TCA9548A 只需要寫入一個 byte（不需要暫存器地址）
    NRF_TWIM0->ADDRESS = MUX_ADDR;
    NRF_TWIM0->TXD.PTR = (uint32_t)&channel_mask;
    NRF_TWIM0->TXD.MAXCNT = 1;

    NRF_TWIM0->EVENTS_STOPPED = 0;
    NRF_TWIM0->TASKS_STARTTX = 1;

    while (!NRF_TWIM0->EVENTS_STOPPED);
    NRF_TWIM0->EVENTS_STOPPED = 0;
}

// 關閉所有 Channel
void i2c_mux_disable_all(void) {
    uint8_t channel_mask = 0x00;

    NRF_TWIM0->ADDRESS = MUX_ADDR;
    NRF_TWIM0->TXD.PTR = (uint32_t)&channel_mask;
    NRF_TWIM0->TXD.MAXCNT = 1;

    NRF_TWIM0->EVENTS_STOPPED = 0;
    NRF_TWIM0->TASKS_STARTTX = 1;

    while (!NRF_TWIM0->EVENTS_STOPPED);
    NRF_TWIM0->EVENTS_STOPPED = 0;
}

// 使用範例
void read_multiple_sensors(void) {
    int16_t accel_x, accel_y, accel_z;

    // 讀取 Channel 0 的 MPU6050
    i2c_mux_select_channel(0);
    mpu6050_read_accel(&accel_x, &accel_y, &accel_z);
    printf("MPU1: X=%d, Y=%d, Z=%d\n", accel_x, accel_y, accel_z);

    // 讀取 Channel 1 的 MPU6050
    i2c_mux_select_channel(1);
    mpu6050_read_accel(&accel_x, &accel_y, &accel_z);
    printf("MPU2: X=%d, Y=%d, Z=%d\n", accel_x, accel_y, accel_z);

    // 關閉所有 Channel（節省功耗）
    i2c_mux_disable_all();
}
```

---

#### 2. **Bus Stuck（Bus 卡住）**

**問題**：SDA 或 SCL 被拉低，整個 Bus 無法工作

**症狀**：

- I2C 傳輸超時
- 所有 Slave 都無法回應
- 示波器顯示 SDA 或 SCL 持續為 Low

**原因**：

1. **Slave 裝置當機**：持續拉低 SDA
2. **電源不穩定**：導致 Slave 進入異常狀態
3. **軟體錯誤**：Master 沒有正確結束傳輸
4. **熱插拔**：在傳輸過程中拔掉 Slave

**解決方案：I2C Bus Recovery**

根據 I2C 規範，可以發送 **9 個 Clock Pulse** 來復位 Bus：

```c
// I2C Bus Recovery（符合 I2C 規範）
bool i2c_bus_recovery(void) {
    // 1. 禁用 I2C 模組
    NRF_TWIM0->ENABLE = TWIM_ENABLE_ENABLE_Disabled;

    // 2. 配置 SCL 和 SDA 為 GPIO
    NRF_GPIO->PIN_CNF[SCL_PIN] =
        (GPIO_PIN_CNF_DIR_Output << GPIO_PIN_CNF_DIR_Pos) |
        (GPIO_PIN_CNF_DRIVE_S0D1 << GPIO_PIN_CNF_DRIVE_Pos);

    NRF_GPIO->PIN_CNF[SDA_PIN] =
        (GPIO_PIN_CNF_DIR_Input << GPIO_PIN_CNF_DIR_Pos) |
        (GPIO_PIN_CNF_INPUT_Connect << GPIO_PIN_CNF_INPUT_Pos);

    // 3. 檢查 SDA 是否被拉低
    if (NRF_GPIO->IN & (1 << SDA_PIN)) {
        // SDA 是 High，Bus 正常
        i2c_init();
        return true;
    }

    // 4. 發送 9 個 Clock Pulse
    // 目的：讓 Slave 完成當前的 Byte 傳輸並釋放 SDA
    for (int i = 0; i < 9; i++) {
        NRF_GPIO->OUTSET = (1 << SCL_PIN);  // SCL = High
        nrf_delay_us(5);
        NRF_GPIO->OUTCLR = (1 << SCL_PIN);  // SCL = Low
        nrf_delay_us(5);

        // 檢查 SDA 是否已經釋放
        if (NRF_GPIO->IN & (1 << SDA_PIN)) {
            break;  // SDA 已經是 High，可以停止
        }
    }

    // 5. 發送 STOP condition
    // SDA 從 Low 變 High（SCL 保持 High）
    NRF_GPIO->PIN_CNF[SDA_PIN] =
        (GPIO_PIN_CNF_DIR_Output << GPIO_PIN_CNF_DIR_Pos) |
        (GPIO_PIN_CNF_DRIVE_S0D1 << GPIO_PIN_CNF_DRIVE_Pos);

    NRF_GPIO->OUTCLR = (1 << SDA_PIN);  // SDA = Low
    nrf_delay_us(5);
    NRF_GPIO->OUTSET = (1 << SCL_PIN);  // SCL = High
    nrf_delay_us(5);
    NRF_GPIO->OUTSET = (1 << SDA_PIN);  // SDA = High (STOP)
    nrf_delay_us(5);

    // 6. 檢查 Bus 是否恢復
    bool recovered = (NRF_GPIO->IN & (1 << SDA_PIN)) &&
                     (NRF_GPIO->IN & (1 << SCL_PIN));

    // 7. 重新初始化 I2C
    i2c_init();

    return recovered;
}

// 在 I2C 傳輸失敗時自動執行 Bus Recovery
bool i2c_write_reg_with_recovery(uint8_t slave_addr, uint8_t reg_addr, uint8_t data) {
    if (i2c_write_reg(slave_addr, reg_addr, data)) {
        return true;
    }

    // 傳輸失敗，嘗試 Bus Recovery
    printf("[WARN] I2C write failed, attempting bus recovery...\n");

    if (i2c_bus_recovery()) {
        printf("[INFO] Bus recovery successful, retrying...\n");
        return i2c_write_reg(slave_addr, reg_addr, data);
    } else {
        printf("[ERROR] Bus recovery failed!\n");
        return false;
    }
}
```

**Bus Recovery 時序圖**：

```
SCL: ─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─────
      └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘
       1   2   3   4   5   6   7   8   9

SDA: ─────────────────────────────────┐ ┌─────
                                       └─┘
                                      STOP
```

---

#### 3. **時序問題（Pull-up 電阻選擇）**

**問題**：資料傳輸錯誤、ACK 失敗、波形失真

**原因**：

1. **Pull-up 電阻太大**：上升時間太慢，違反 I2C 時序規範
2. **Pull-up 電阻太小**：電流太大，功耗增加，可能損壞 GPIO
3. **Bus 電容太大**：線太長、裝置太多、PCB trace 電容

**Pull-up 電阻計算**：

```
上升時間公式：
t_rise = 0.8473 × R_pullup × C_bus

I2C 規範要求：
- Standard Mode (100 kHz): t_rise < 1000 ns
- Fast Mode (400 kHz): t_rise < 300 ns
- Fast Mode Plus (1 MHz): t_rise < 120 ns

計算 R_pullup：
R_pullup < t_rise / (0.8473 × C_bus)

範例 1：Fast Mode (400 kHz)，3 個裝置
C_bus = 3 × 10pF (裝置) + 50pF (PCB trace) = 80pF
R_pullup < 300ns / (0.8473 × 80pF) = 4.4kΩ
選擇：2.2kΩ 或 3.3kΩ

範例 2：Fast Mode (400 kHz)，5 個裝置 + 長線
C_bus = 5 × 10pF (裝置) + 150pF (PCB trace) = 200pF
R_pullup < 300ns / (0.8473 × 200pF) = 1.77kΩ
選擇：1.5kΩ 或 1kΩ
```

**Pull-up 電阻選擇表**：

| I2C 速度 | 裝置數量 | Bus 電容 | 推薦 R_pullup |
|----------|----------|----------|---------------|
| 100 kHz | 1-3 | < 100 pF | 4.7kΩ |
| 100 kHz | 4-7 | 100-200 pF | 3.3kΩ |
| 400 kHz | 1-3 | < 100 pF | 2.2kΩ - 3.3kΩ |
| 400 kHz | 4-7 | 100-200 pF | 1.5kΩ - 2.2kΩ |
| 1 MHz | 1-3 | < 100 pF | 1kΩ - 1.5kΩ |

**實務建議**：

1. **使用示波器測量上升時間**：

   ```c
   // 測量 SDA 和 SCL 的上升時間
   // 從 30% VDD 到 70% VDD 的時間
   ```

2. **降低時鐘頻率**（如果 Pull-up 電阻無法更換）：

   ```c
   // 從 400 kHz 降到 100 kHz
   NRF_TWIM0->FREQUENCY = TWIM_FREQUENCY_FREQUENCY_K100;
   ```

3. **減少 Bus 電容**：
   - 縮短 PCB trace 長度
   - 減少裝置數量
   - 使用 I2C Multiplexer 分離 Bus

---

## UART 協定 - 除錯與 HCI Transport

### 為什麼需要 UART？

在 IoT 專案中，**UART (Universal Asynchronous Receiver/Transmitter)** 有兩個主要用途：

1. **除錯介面**：輸出 Log 訊息，方便開發和除錯
   - 系統啟動訊息
   - 錯誤訊息和警告
   - Sensor 資料監控
   - 性能分析（時間戳記）

2. **HCI Transport**：Bluetooth Controller 和 Host 之間的通訊（見文章 #3）
   - HCI Commands
   - HCI Events
   - HCI ACL Data

### UART 基本原理

UART 是一個 **非同步** 串列通訊協定：

- **非同步**：不需要時鐘訊號，使用 Start bit 和 Stop bit 同步
- **全雙工**：可以同時發送和接收資料
- **點對點**：一對一通訊（不像 I2C 可以多個裝置）

**UART 訊號線**：

```
┌─────────────────────────────────────────────────────────┐
│  UART 連接                                                │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─────────┐                    ┌─────────┐             │
│  │ Device A│                    │ Device B│             │
│  │         │                    │         │             │
│  │  TX  ───┼────────────────────┼───  RX  │             │
│  │  RX  ───┼────────────────────┼───  TX  │             │
│  │  GND ───┼────────────────────┼───  GND │             │
│  │         │                    │         │             │
│  └─────────┘                    └─────────┘             │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**UART 資料格式**：

```
┌─────────────────────────────────────────────────────────┐
│  UART 資料格式（8N1）                                     │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  Idle  Start  D0  D1  D2  D3  D4  D5  D6  D7  Stop Idle  │
│  ─────┐    ┌───┬───┬───┬───┬───┬───┬───┬───┐    ┌─────  │
│        └────┘   │   │   │   │   │   │   │   └────┘       │
│         ↑       └───┴───┴───┴───┴───┴───┘    ↑           │
│      Start bit      8 Data bits          Stop bit        │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**常見配置**：

- **8N1**：8 data bits, No parity, 1 stop bit（最常用）
- **Baud Rate**：115200 bps（除錯）、921600 bps（HCI）

---

### UART 驅動實作（以 nRF52 為例）

#### 1. **初始化 UART**

```c
// UART 配置
#define TX_PIN 6
#define RX_PIN 8
#define UART_BAUDRATE UARTE_BAUDRATE_BAUDRATE_Baud115200  // 115200 bps

// UART 初始化
void uart_init(void) {
    // 1. 配置 GPIO
    NRF_UARTE0->PSEL.TXD = TX_PIN;
    NRF_UARTE0->PSEL.RXD = RX_PIN;
    NRF_UARTE0->PSEL.CTS = UARTE_PSEL_CTS_CONNECT_Disconnected;  // 不使用流控
    NRF_UARTE0->PSEL.RTS = UARTE_PSEL_RTS_CONNECT_Disconnected;

    // 2. 配置 Baud Rate
    NRF_UARTE0->BAUDRATE = UART_BAUDRATE;

    // 3. 配置格式（8N1）
    NRF_UARTE0->CONFIG = (UARTE_CONFIG_HWFC_Disabled << UARTE_CONFIG_HWFC_Pos) |  // 禁用硬體流控
                         (UARTE_CONFIG_PARITY_Excluded << UARTE_CONFIG_PARITY_Pos);  // 無 Parity

    // 4. 啟用 UART
    NRF_UARTE0->ENABLE = UARTE_ENABLE_ENABLE_Enabled;
}
```

#### 2. **發送資料**

```c
// UART 發送（Blocking）
void uart_send(const char *str) {
    uint32_t len = strlen(str);

    // 設定傳輸資料
    NRF_UARTE0->TXD.PTR = (uint32_t)str;
    NRF_UARTE0->TXD.MAXCNT = len;

    // 清除事件
    NRF_UARTE0->EVENTS_ENDTX = 0;

    // 啟動傳輸
    NRF_UARTE0->TASKS_STARTTX = 1;

    // 等待傳輸完成
    while (!NRF_UARTE0->EVENTS_ENDTX);

    // 停止傳輸
    NRF_UARTE0->TASKS_STOPTX = 1;
}

// 除錯 Log 輸出（支援 printf 格式）
void debug_log(const char *format, ...) {
    char buffer[256];
    va_list args;

    va_start(args, format);
    vsnprintf(buffer, sizeof(buffer), format, args);
    va_end(args);

    uart_send(buffer);
}

// 使用範例
void test_uart_log(void) {
    debug_log("[INFO] System boot...\n");
    debug_log("[INFO] CPU Frequency: %d MHz\n", 64);
    debug_log("[INFO] Free RAM: %d bytes\n", 32768);
}
```

#### 3. **接收資料（使用 DMA）**

```c
// UART 接收 Buffer
#define RX_BUFFER_SIZE 256
static uint8_t rx_buffer[RX_BUFFER_SIZE];
static volatile bool rx_ready = false;

// UART 接收初始化（使用中斷）
void uart_rx_init(void) {
    // 配置接收 Buffer
    NRF_UARTE0->RXD.PTR = (uint32_t)rx_buffer;
    NRF_UARTE0->RXD.MAXCNT = RX_BUFFER_SIZE;

    // 啟用接收中斷
    NRF_UARTE0->INTENSET = UARTE_INTENSET_ENDRX_Msk;
    NVIC_EnableIRQ(UARTE0_UART0_IRQn);

    // 啟動接收
    NRF_UARTE0->TASKS_STARTRX = 1;
}

// UART 中斷處理函式
void UARTE0_UART0_IRQHandler(void) {
    if (NRF_UARTE0->EVENTS_ENDRX) {
        NRF_UARTE0->EVENTS_ENDRX = 0;

        // 接收完成
        rx_ready = true;

        // 重新啟動接收
        NRF_UARTE0->TASKS_STARTRX = 1;
    }
}

// 處理接收資料
void process_uart_rx(void) {
    if (rx_ready) {
        rx_ready = false;

        uint32_t len = NRF_UARTE0->RXD.AMOUNT;

        // 處理接收到的資料
        for (uint32_t i = 0; i < len; i++) {
            char c = rx_buffer[i];
            // 處理字元...
        }
    }
}
```

---

## GPIO 管理 - 中斷處理與電源控制

### GPIO 的用途

在 IoT 裝置中，GPIO (General Purpose Input/Output) 有多種用途：

1. **中斷輸入**：
   - Sensor 資料就緒訊號（INT pin）
   - 按鈕輸入
   - 外部事件觸發

2. **電源控制**：
   - Sensor 電源開關
   - LED 指示燈
   - 外部模組控制

3. **狀態指示**：
   - LED 閃爍
   - 除錯訊號

---

### GPIO 中斷處理（以 nRF52 為例）

#### 1. **配置 GPIO 中斷**

```c
// GPIO 配置
#define ACCEL_INT_PIN 12   // 加速度計中斷 pin
#define BUTTON_PIN 13      // 按鈕 pin

// GPIO 中斷初始化
void gpio_interrupt_init(void) {
    // 1. 配置 GPIO 為輸入
    NRF_GPIO->PIN_CNF[ACCEL_INT_PIN] =
        (GPIO_PIN_CNF_DIR_Input << GPIO_PIN_CNF_DIR_Pos) |
        (GPIO_PIN_CNF_INPUT_Connect << GPIO_PIN_CNF_INPUT_Pos) |
        (GPIO_PIN_CNF_PULL_Disabled << GPIO_PIN_CNF_PULL_Pos);  // 外部 Pull-down

    // 2. 配置 GPIOTE（GPIO Task and Event）
    // Channel 0: 加速度計中斷（Low to High）
    NRF_GPIOTE->CONFIG[0] =
        (GPIOTE_CONFIG_MODE_Event << GPIOTE_CONFIG_MODE_Pos) |
        (ACCEL_INT_PIN << GPIOTE_CONFIG_PSEL_Pos) |
        (GPIOTE_CONFIG_POLARITY_LoToHi << GPIOTE_CONFIG_POLARITY_Pos);

    // 3. 啟用中斷
    NRF_GPIOTE->INTENSET = GPIOTE_INTENSET_IN0_Msk;
    NVIC_SetPriority(GPIOTE_IRQn, 3);  // 優先級 3（較低）
    NVIC_EnableIRQ(GPIOTE_IRQn);
}

// GPIO 中斷處理函式
void GPIOTE_IRQHandler(void) {
    // 檢查 Channel 0（加速度計中斷）
    if (NRF_GPIOTE->EVENTS_IN[0]) {
        NRF_GPIOTE->EVENTS_IN[0] = 0;  // 清除事件

        // 設定旗標，在主迴圈中處理
        accel_data_ready = true;
    }
}

// 主迴圈處理
void main_loop(void) {
    while (1) {
        if (accel_data_ready) {
            accel_data_ready = false;

            // 讀取加速度計資料
            int16_t accel_x, accel_y, accel_z;
            mpu6050_read_accel(&accel_x, &accel_y, &accel_z);

            // 處理資料...
        }

        // 進入 Sleep 模式
        __WFE();  // Wait For Event
    }
}
```

#### 2. **中斷優先級管理**

在 IoT 裝置中，合理分配中斷優先級非常重要：

```c
// 中斷優先級配置（數字越小，優先級越高）
void interrupt_priority_init(void) {
    // 1. BLE Radio（最高優先級）
    NVIC_SetPriority(RADIO_IRQn, 0);

    // 2. RTC（時間關鍵）
    NVIC_SetPriority(RTC1_IRQn, 1);

    // 3. UART（HCI Transport）
    NVIC_SetPriority(UARTE0_UART0_IRQn, 2);

    // 4. GPIO（Sensor 中斷）
    NVIC_SetPriority(GPIOTE_IRQn, 3);

    // 5. I2C（最低優先級）
    NVIC_SetPriority(SPIM0_SPIS0_TWIM0_TWIS0_SPI0_TWI0_IRQn, 4);
}
```

---

### GPIO 電源控制

#### 1. **Sensor 電源管理**

```c
// Sensor 電源 pin 配置
#define SENSOR_POWER_PIN 14

// 初始化電源控制 pin
void sensor_power_init(void) {
    // 配置為輸出，初始為 Low（關閉電源）
    NRF_GPIO->PIN_CNF[SENSOR_POWER_PIN] =
        (GPIO_PIN_CNF_DIR_Output << GPIO_PIN_CNF_DIR_Pos) |
        (GPIO_PIN_CNF_DRIVE_S0S1 << GPIO_PIN_CNF_DRIVE_Pos);  // Standard drive

    NRF_GPIO->OUTCLR = (1 << SENSOR_POWER_PIN);  // 初始為 Low
}

// 開啟 Sensor 電源
void sensor_power_on(void) {
    NRF_GPIO->OUTSET = (1 << SENSOR_POWER_PIN);  // 設為 High
    nrf_delay_ms(10);  // 等待電源穩定
}

// 關閉 Sensor 電源
void sensor_power_off(void) {
    NRF_GPIO->OUTCLR = (1 << SENSOR_POWER_PIN);  // 設為 Low
}

// 使用範例：節省功耗
void read_sensor_with_power_control(void) {
    // 1. 開啟電源
    sensor_power_on();

    // 2. 初始化 Sensor
    mpu6050_init();

    // 3. 讀取資料
    int16_t accel_x, accel_y, accel_z;
    mpu6050_read_accel(&accel_x, &accel_y, &accel_z);

    // 4. 關閉電源（節省功耗）
    sensor_power_off();
}
```

#### 2. **LED 指示燈控制**

```c
// LED pin 配置
#define LED_RED_PIN 17
#define LED_GREEN_PIN 18
#define LED_BLUE_PIN 19

// LED 初始化
void led_init(void) {
    // 配置為輸出，初始為 High（LED 關閉，假設 Active Low）
    NRF_GPIO->PIN_CNF[LED_RED_PIN] =
        (GPIO_PIN_CNF_DIR_Output << GPIO_PIN_CNF_DIR_Pos);
    NRF_GPIO->PIN_CNF[LED_GREEN_PIN] =
        (GPIO_PIN_CNF_DIR_Output << GPIO_PIN_CNF_DIR_Pos);
    NRF_GPIO->PIN_CNF[LED_BLUE_PIN] =
        (GPIO_PIN_CNF_DIR_Output << GPIO_PIN_CNF_DIR_Pos);

    NRF_GPIO->OUTSET = (1 << LED_RED_PIN) | (1 << LED_GREEN_PIN) | (1 << LED_BLUE_PIN);
}

// LED 控制
void led_on(uint8_t led_pin) {
    NRF_GPIO->OUTCLR = (1 << led_pin);  // Active Low
}

void led_off(uint8_t led_pin) {
    NRF_GPIO->OUTSET = (1 << led_pin);
}

void led_toggle(uint8_t led_pin) {
    NRF_GPIO->OUT ^= (1 << led_pin);
}

// LED 閃爍（用於狀態指示）
void led_blink(uint8_t led_pin, uint32_t duration_ms) {
    led_on(led_pin);
    nrf_delay_ms(duration_ms);
    led_off(led_pin);
}
```

---

## 真實案例：智慧手環的 Sensor 整合

回到文章開頭的故事。在解決了 Pull-up 電阻和地址衝突的問題後，我面臨了一個新的挑戰：**如何高效地管理三個 Sensor 的資料讀取？**

### 問題分析

智慧手環需要整合三個 Sensor：

1. **MPU6050 加速度計/陀螺儀**（地址 0x68）
   - 用途：計步、手勢識別
   - 更新頻率：100 Hz
   - 資料量：12 bytes（6 軸資料）

2. **MAX30102 心率感測器**（地址 0x57）
   - 用途：心率和血氧監測
   - 更新頻率：25 Hz
   - 資料量：6 bytes（紅光 + 紅外光）

3. **APDS-9960 環境光/接近感測器**（地址 0x39）
   - 用途：自動調整螢幕亮度、手勢控制
   - 更新頻率：10 Hz
   - 資料量：8 bytes（RGBC + 接近度）

**挑戰**：

- 三個 Sensor 的更新頻率不同
- I2C Bus 是共享的，需要避免衝突
- 需要最小化功耗（使用中斷而非輪詢）

### 解決方案：I2C Bus Manager + GPIO 中斷

我設計了一個 **I2C Bus Manager** 來管理多個 Sensor 的請求：

```c
// I2C 請求類型
typedef enum {
    I2C_REQ_WRITE,
    I2C_REQ_READ,
} i2c_request_type_t;

// I2C 請求結構
typedef struct {
    i2c_request_type_t type;
    uint8_t slave_addr;
    uint8_t reg_addr;
    uint8_t *data;
    uint8_t len;
    void (*callback)(bool success);  // 完成後的回調函式
} i2c_request_t;

// I2C 請求佇列（Ring Buffer）
#define I2C_QUEUE_SIZE 16
static i2c_request_t i2c_queue[I2C_QUEUE_SIZE];
static uint8_t queue_head = 0;
static uint8_t queue_tail = 0;
static volatile bool i2c_busy = false;

// 加入請求到佇列
bool i2c_request_add(i2c_request_t *request) {
    uint8_t next_tail = (queue_tail + 1) % I2C_QUEUE_SIZE;

    // 檢查佇列是否已滿
    if (next_tail == queue_head) {
        debug_log("[ERROR] I2C queue full!\n");
        return false;
    }

    // 複製請求到佇列
    memcpy(&i2c_queue[queue_tail], request, sizeof(i2c_request_t));
    queue_tail = next_tail;

    // 如果 I2C 不忙，立即處理
    if (!i2c_busy) {
        i2c_process_queue();
    }

    return true;
}

// 處理佇列中的請求
void i2c_process_queue(void) {
    // 檢查佇列是否為空
    if (queue_head == queue_tail) {
        i2c_busy = false;
        return;
    }

    i2c_busy = true;

    // 取出請求
    i2c_request_t *req = &i2c_queue[queue_head];
    queue_head = (queue_head + 1) % I2C_QUEUE_SIZE;

    // 執行請求
    bool success = false;
    if (req->type == I2C_REQ_WRITE) {
        success = i2c_write_bytes(req->slave_addr, req->reg_addr, req->data, req->len);
    } else {
        success = i2c_read_reg(req->slave_addr, req->reg_addr, req->data, req->len);
    }

    // 呼叫回調函式
    if (req->callback) {
        req->callback(success);
    }

    // 處理下一個請求
    i2c_process_queue();
}
```

### Sensor 驅動整合

使用 I2C Bus Manager 後，每個 Sensor 的驅動變得非常簡潔：

```c
// MPU6050 驅動
static uint8_t mpu6050_data[12];

void mpu6050_data_ready_callback(bool success) {
    if (success) {
        // 解析加速度計和陀螺儀資料
        int16_t accel_x = (int16_t)((mpu6050_data[0] << 8) | mpu6050_data[1]);
        int16_t accel_y = (int16_t)((mpu6050_data[2] << 8) | mpu6050_data[3]);
        int16_t accel_z = (int16_t)((mpu6050_data[4] << 8) | mpu6050_data[5]);

        // 處理資料...
        process_accel_data(accel_x, accel_y, accel_z);
    }
}

// MPU6050 中斷處理（資料就緒）
void GPIOTE_IRQHandler(void) {
    if (NRF_GPIOTE->EVENTS_IN[0]) {  // MPU6050 INT pin
        NRF_GPIOTE->EVENTS_IN[0] = 0;

        // 加入讀取請求到佇列
        i2c_request_t req = {
            .type = I2C_REQ_READ,
            .slave_addr = 0x68,
            .reg_addr = 0x3B,  // ACCEL_XOUT_H
            .data = mpu6050_data,
            .len = 12,
            .callback = mpu6050_data_ready_callback,
        };
        i2c_request_add(&req);
    }
}
```

### 系統架構

最終的系統架構如下：

```
┌─────────────────────────────────────────────────────────┐
│  智慧手環 Sensor 整合架構                                 │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─────────────────────────────────────────────┐         │
│  │  Application Layer                          │         │
│  │  - 計步演算法                                │         │
│  │  - 心率分析                                  │         │
│  │  - 手勢識別                                  │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                         │
│  ┌──────────────┴──────────────────────────────┐         │
│  │  I2C Bus Manager                            │         │
│  │  - 請求佇列（Ring Buffer）                   │         │
│  │  - 優先級管理                                │         │
│  │  - 錯誤處理和重試                            │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                         │
│  ┌──────────────┴──────────────────────────────┐         │
│  │  I2C Driver (HAL)                           │         │
│  │  - 硬體抽象層                                │         │
│  │  - DMA 傳輸                                  │         │
│  │  - Bus Recovery                             │         │
│  └──────────────┬──────────────────────────────┘         │
│                 │                                         │
│  ┌──────────────┴──────────────────────────────┐         │
│  │  I2C Bus (SDA, SCL)                         │         │
│  │  - 400 kHz                                  │         │
│  │  - Pull-up: 2.2kΩ                           │         │
│  └─────┬────────┬────────┬───────────────────┘         │
│        │        │        │                               │
│   ┌────┴───┐ ┌─┴────┐ ┌─┴────────┐                     │
│   │MPU6050 │ │MAX30 │ │APDS-9960 │                     │
│   │(0x68)  │ │(0x57)│ │(0x39)    │                     │
│   │INT->P12│ │INT->P│ │INT->P14  │                     │
│   └────────┘ └──────┘ └──────────┘                     │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

### 性能優化結果

使用 I2C Bus Manager 後，系統性能顯著提升：

| 指標 | 優化前 | 優化後 | 改善 |
|------|--------|--------|------|
| **CPU 使用率** | 45% | 12% | 73% ↓ |
| **I2C Bus 利用率** | 60% | 25% | 58% ↓ |
| **平均延遲** | 15 ms | 3 ms | 80% ↓ |
| **功耗** | 3.5 mA | 2.1 mA | 40% ↓ |

**關鍵優化**：

1. **使用中斷而非輪詢**：CPU 可以進入 Sleep 模式
2. **請求佇列**：避免 I2C Bus 衝突
3. **DMA 傳輸**：減少 CPU 負擔
4. **電源管理**：不使用時關閉 Sensor 電源

---

## 進階技巧

### 1. **I2C Clock Stretching**

某些 Sensor（如 BME280）支援 **Clock Stretching**：

```c
// 啟用 Clock Stretching（nRF52 預設支援）
// Slave 可以拉低 SCL 來暫停傳輸，等待資料準備好

// 範例：讀取 BME280 溫度資料
// BME280 在測量時會使用 Clock Stretching
bool bme280_read_temperature(int32_t *temp) {
    uint8_t data[3];

    // 觸發測量
    i2c_write_reg(0x76, 0xF4, 0x2E);  // CTRL_MEAS: Force mode

    // 讀取資料（BME280 會使用 Clock Stretching 等待測量完成）
    if (i2c_read_reg(0x76, 0xFA, data, 3)) {
        *temp = (int32_t)((data[0] << 12) | (data[1] << 4) | (data[2] >> 4));
        return true;
    }
    return false;
}
```

### 2. **I2C Multi-Master**

在某些應用中，可能有多個 Master（例如 MCU + FPGA）：

```c
// I2C Multi-Master 仲裁
// 當兩個 Master 同時發送 START condition 時，
// 透過 SDA 的電平來決定優先權（低電平優先）

// nRF52 不支援 Multi-Master，但可以透過軟體實作
bool i2c_multi_master_write(uint8_t slave_addr, uint8_t reg_addr, uint8_t data) {
    // 1. 檢查 Bus 是否空閒
    if (!(NRF_GPIO->IN & (1 << SDA_PIN)) || !(NRF_GPIO->IN & (1 << SCL_PIN))) {
        return false;  // Bus 忙碌
    }

    // 2. 發送 START condition
    // 3. 檢查仲裁（如果失敗，重試）
    // ...
}
```

### 3. **UART 流控（Flow Control）**

對於高速 UART（如 HCI Transport），建議使用硬體流控：

```c
// 啟用硬體流控（RTS/CTS）
void uart_init_with_flow_control(void) {
    NRF_UARTE0->PSEL.TXD = TX_PIN;
    NRF_UARTE0->PSEL.RXD = RX_PIN;
    NRF_UARTE0->PSEL.RTS = RTS_PIN;  // Request To Send
    NRF_UARTE0->PSEL.CTS = CTS_PIN;  // Clear To Send

    NRF_UARTE0->BAUDRATE = UARTE_BAUDRATE_BAUDRATE_Baud921600;  // 921600 bps
    NRF_UARTE0->CONFIG = (UARTE_CONFIG_HWFC_Enabled << UARTE_CONFIG_HWFC_Pos);  // 啟用流控

    NRF_UARTE0->ENABLE = UARTE_ENABLE_ENABLE_Enabled;
}
```

---

## 實務建議

### I2C 設計檢查清單

#### I2C 硬體設計

✅ **Pull-up 電阻選擇**：

- 100 kHz：4.7kΩ
- 400 kHz：2.2kΩ - 3.3kΩ
- 1 MHz：1kΩ - 1.5kΩ
- 使用示波器測量上升時間（應 < 規範要求）

✅ **PCB 佈線**：

- SDA 和 SCL 走線盡量短（< 30 cm for 400 kHz）
- 避免與高速訊號（如 SPI、UART）平行走線
- 使用 Ground plane 減少干擾
- 在 VDD 和 GND 之間加入去耦電容（0.1μF + 10μF）

✅ **地址規劃**：

- 列出所有 Sensor 的 I2C 地址
- 檢查是否有衝突
- 預留 I2C Multiplexer 的位置（如果需要）

#### I2C 軟體設計

✅ **錯誤處理**：

```c
// 完整的錯誤處理範例
bool i2c_write_with_retry(uint8_t slave_addr, uint8_t reg_addr, uint8_t data) {
    const int MAX_RETRIES = 3;

    for (int retry = 0; retry < MAX_RETRIES; retry++) {
        if (i2c_write_reg(slave_addr, reg_addr, data)) {
            return true;  // 成功
        }

        // 失敗，嘗試 Bus Recovery
        debug_log("[WARN] I2C write failed (retry %d/%d)\n", retry + 1, MAX_RETRIES);

        if (i2c_bus_recovery()) {
            nrf_delay_ms(10);  // 等待 Bus 穩定
        } else {
            debug_log("[ERROR] Bus recovery failed!\n");
            return false;
        }
    }

    debug_log("[ERROR] I2C write failed after %d retries\n", MAX_RETRIES);
    return false;
}
```

✅ **超時保護**：

```c
// 使用 Timer 實作超時保護
#define I2C_TIMEOUT_MS 100

bool i2c_write_with_timeout(uint8_t slave_addr, uint8_t reg_addr, uint8_t data) {
    uint32_t start_time = get_system_time_ms();

    // 啟動傳輸
    NRF_TWIM0->TASKS_STARTTX = 1;

    // 等待完成或超時
    while (!NRF_TWIM0->EVENTS_STOPPED && !NRF_TWIM0->EVENTS_ERROR) {
        if (get_system_time_ms() - start_time > I2C_TIMEOUT_MS) {
            debug_log("[ERROR] I2C timeout!\n");
            NRF_TWIM0->TASKS_STOP = 1;  // 停止傳輸
            return false;
        }
    }

    // 檢查錯誤...
}
```

✅ **請求佇列**：

- 使用 Ring Buffer 管理多個 Sensor 的請求
- 避免 I2C Bus 衝突
- 支援優先級（例如心率感測器優先於環境光感測器）

---

### UART 設計檢查清單

#### UART 硬體設計

✅ **Baud Rate 選擇**：

- 除錯 Log：115200 bps
- HCI Transport：921600 bps 或 1 Mbps
- 確保 MCU 的時鐘頻率可以精確產生 Baud Rate

✅ **流控**：

- 低速（< 115200 bps）：可以不使用流控
- 高速（≥ 921600 bps）：建議使用硬體流控（RTS/CTS）

#### UART 軟體設計

✅ **使用 DMA**：

```c
// 使用 DMA 減少 CPU 負擔
void uart_send_dma(const char *str, uint32_t len) {
    NRF_UARTE0->TXD.PTR = (uint32_t)str;
    NRF_UARTE0->TXD.MAXCNT = len;

    NRF_UARTE0->EVENTS_ENDTX = 0;
    NRF_UARTE0->TASKS_STARTTX = 1;

    // 不需要等待，DMA 會自動傳輸
}
```

✅ **Log 分級**：

```c
// 實作 Log 分級（減少 UART 輸出）
typedef enum {
    LOG_LEVEL_ERROR,
    LOG_LEVEL_WARN,
    LOG_LEVEL_INFO,
    LOG_LEVEL_DEBUG,
} log_level_t;

static log_level_t current_log_level = LOG_LEVEL_INFO;

void debug_log_level(log_level_t level, const char *format, ...) {
    if (level > current_log_level) return;  // 過濾低優先級 Log

    char buffer[256];
    va_list args;

    va_start(args, format);
    vsnprintf(buffer, sizeof(buffer), format, args);
    va_end(args);

    uart_send(buffer);
}

// 使用範例
debug_log_level(LOG_LEVEL_ERROR, "[ERROR] I2C failed!\n");
debug_log_level(LOG_LEVEL_DEBUG, "[DEBUG] Accel: X=%d\n", accel_x);  // 只在 DEBUG 模式輸出
```

---

### GPIO 設計檢查清單

#### 中斷管理

✅ **優先級分配**：

```c
// 推薦的中斷優先級（數字越小，優先級越高）
NVIC_SetPriority(RADIO_IRQn, 0);           // BLE Radio（最高）
NVIC_SetPriority(RTC1_IRQn, 1);            // RTC（時間關鍵）
NVIC_SetPriority(UARTE0_UART0_IRQn, 2);    // UART（HCI）
NVIC_SetPriority(GPIOTE_IRQn, 3);          // GPIO（Sensor 中斷）
NVIC_SetPriority(SPIM0_SPIS0_TWIM0_TWIS0_SPI0_TWI0_IRQn, 4);  // I2C（最低）
```

✅ **中斷處理原則**：

- **快速處理**：中斷處理函式應盡量簡短（< 10 μs）
- **設定旗標**：在中斷中設定旗標，在主迴圈中處理
- **避免阻塞**：不要在中斷中呼叫 `nrf_delay_ms()` 或 I2C/UART 傳輸

```c
// 錯誤範例（不要這樣做）
void GPIOTE_IRQHandler(void) {
    if (NRF_GPIOTE->EVENTS_IN[0]) {
        NRF_GPIOTE->EVENTS_IN[0] = 0;

        // ❌ 不要在中斷中執行耗時操作
        int16_t accel_x, accel_y, accel_z;
        mpu6050_read_accel(&accel_x, &accel_y, &accel_z);  // I2C 傳輸，耗時 > 100 μs
        process_accel_data(accel_x, accel_y, accel_z);     // 複雜計算
    }
}

// 正確範例
volatile bool accel_data_ready = false;

void GPIOTE_IRQHandler(void) {
    if (NRF_GPIOTE->EVENTS_IN[0]) {
        NRF_GPIOTE->EVENTS_IN[0] = 0;

        // ✅ 只設定旗標
        accel_data_ready = true;
    }
}

void main_loop(void) {
    while (1) {
        if (accel_data_ready) {
            accel_data_ready = false;

            // 在主迴圈中處理
            int16_t accel_x, accel_y, accel_z;
            mpu6050_read_accel(&accel_x, &accel_y, &accel_z);
            process_accel_data(accel_x, accel_y, accel_z);
        }

        __WFE();  // 進入 Sleep 模式
    }
}
```

#### 電源管理

✅ **Sensor 電源控制**：

- 不使用時關閉 Sensor 電源（節省功耗）
- 開啟電源後等待穩定（通常 10-100 ms）
- 使用 MOSFET 或 Load Switch 控制電源

✅ **GPIO 功耗優化**：

```c
// 未使用的 GPIO 應配置為輸入 + Pull-down（減少漏電流）
void gpio_unused_pins_init(void) {
    for (int pin = 0; pin < 32; pin++) {
        // 跳過已使用的 pin
        if (pin == SDA_PIN || pin == SCL_PIN || pin == TX_PIN || pin == RX_PIN) {
            continue;
        }

        // 配置為輸入 + Pull-down
        NRF_GPIO->PIN_CNF[pin] =
            (GPIO_PIN_CNF_DIR_Input << GPIO_PIN_CNF_DIR_Pos) |
            (GPIO_PIN_CNF_INPUT_Disconnect << GPIO_PIN_CNF_INPUT_Pos) |
            (GPIO_PIN_CNF_PULL_Pulldown << GPIO_PIN_CNF_PULL_Pos);
    }
}
```

---

## 總結

這篇文章從一個週五晚上的 I2C 除錯經歷出發，深入探討了 **I2C、UART、GPIO** 這三個 IoT 晶片的基礎接口。

### 核心要點

**I2C**：

- ✅ 只需兩條線（SDA, SCL），可連接多個裝置
- ✅ Pull-up 電阻選擇：根據速度和 Bus 電容計算
- ✅ 常見問題：地址衝突、Bus Stuck、時序問題
- ✅ 解決方案：I2C Multiplexer、Bus Recovery、請求佇列

**UART**：

- ✅ 非同步串列通訊，適合除錯和 HCI Transport
- ✅ Baud Rate 選擇：115200 bps（除錯）、921600 bps（HCI）
- ✅ 使用 DMA 減少 CPU 負擔
- ✅ 高速傳輸建議使用硬體流控（RTS/CTS）

**GPIO**：

- ✅ 用於中斷、電源控制、狀態指示
- ✅ 合理分配中斷優先級：BLE Radio > RTC > UART > GPIO > I2C
- ✅ 中斷處理原則：快速處理、設定旗標、避免阻塞
- ✅ 功耗優化：關閉未使用的 Sensor、未使用的 GPIO 配置為 Pull-down

### 實戰經驗

從這次智慧手環的 Sensor 整合專案中，我學到了：

1. **理解底層原理**：Pull-up 電阻不只是「接上去就好」，需要根據速度和電容計算
2. **系統性思考**：使用 I2C Bus Manager 管理多個 Sensor，而非各自為政
3. **錯誤處理**：實作 Bus Recovery、超時保護、重試機制
4. **功耗優化**：使用中斷而非輪詢、關閉未使用的 Sensor

只有深入理解這些接口的原理和特性，才能在 IoT 專案中游刃有餘。

下一篇文章，我們將探討 **BLE 功耗優化**，看看如何讓智慧手環的電池從 1 週延長到 13 個月。

---

## 參考資料

1. NXP. (2014). *I2C-bus specification and user manual*. <https://www.nxp.com/docs/en/user-guide/UM10204.pdf>
2. Texas Instruments. (2015). *Understanding the I2C Bus*. <https://www.ti.com/lit/an/slva704/slva704.pdf>
3. Nordic Semiconductor. (2016). *nRF52832 Product Specification*. <https://infocenter.nordicsemi.com/pdf/nRF52832_PS_v1.4.pdf>
4. Maxim Integrated. (2015). *UART: A Hardware Communication Protocol*. <https://www.maximintegrated.com/en/design/technical-documents/tutorials/2/2141.html>

---

**版權聲明**

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。
**作者**：Danny Jiang
**出處**：<https://github.com/djiangtw/tech-column-public>
