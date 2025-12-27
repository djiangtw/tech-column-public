---
author: Danny Jiang
date: 2025-12-11
---

# 系統級功耗優化 - 從晶片到系統

## 前言

2015 年 3 月的某個週三下午，我正在會議室裡和硬體團隊開會。我們正在設計一款新的 IoT 裝置，整合了 BLE、加速度計、心率感測器、環境光感測器、還有一個小型 OLED Display。

「根據初步估算，這個裝置的平均功耗是 2 mA。」硬體工程師 Mike 說，「使用 CR2032 電池（200 mAh），可以撐 100 小時，大約 4 天。」

「4 天？」產品經理 Lisa 皺起了眉頭，「客戶要求至少 6 個月的電池續航。4 天到 6 個月，差了 45 倍！」

會議室裡一片沉默。大家都知道，這不是簡單的 BLE 參數調整就能解決的問題。這需要 **系統級的功耗優化**。

「讓我們重新審視整個系統。」我說，「BLE、Sensor、MCU、Display，每個模組都有自己的功耗特性。我們需要設計一個協調機制，讓它們在正確的時間開啟和關閉。」

接下來的一個月，我們進行了深入的功耗分析和優化。我們使用功耗分析儀測量每個模組的功耗，設計了一個複雜的狀態機來管理系統狀態，實作了 Clock Gating、Power Gating、DVFS 等硬體優化技術。

最終，我們成功將平均功耗從 2 mA 降低到 **22 μA**，電池續航達到 **9,000 小時**，約 **375 天**，超過了客戶的要求。

這次經驗讓我深刻體會到：**功耗優化不只是軟體的問題，也不只是硬體的問題，而是需要軟硬體協同設計的系統工程**。

---

## 系統功耗分析

### 功耗來源拆解

在一個典型的 IoT 裝置中，功耗來自多個模組：

| 模組 | Active 電流 | Sleep 電流 | 使用時間佔比 | 平均電流 |
|------|-------------|------------|--------------|----------|
| **BLE Radio** | 15 mA | 0 μA | 0.5% | 75 μA |
| **MCU (ARM Cortex-M4)** | 8 mA | 3 μA | 5% | 403 μA |
| **加速度計** | 150 μA | 2 μA | 10% | 17 μA |
| **心率感測器** | 2 mA | 0 μA | 2% | 40 μA |
| **環境光感測器** | 100 μA | 1 μA | 1% | 2 μA |
| **OLED Display** | 10 mA | 0 μA | 1% | 100 μA |
| **其他（LDO, RTC）** | - | 5 μA | 100% | 5 μA |
| **總計** | - | - | - | **642 μA** |

**關鍵發現**：
1. **MCU 佔了 63% 的功耗**（403 μA / 642 μA）
2. **BLE Radio 佔了 12%**（75 μA / 642 μA）
3. **Display 佔了 16%**（100 μA / 642 μA）

**優化策略**：
- **降低 MCU 的 Active 時間**：使用更高效的演算法、減少不必要的計算
- **降低 MCU 的 Active 電流**：使用 DVFS（動態電壓頻率調整）
- **優化 BLE 參數**：見文章 #13
- **減少 Display 使用**：縮短開啟時間、降低亮度

---

## Clock Gating - 關閉不使用的時鐘

### 什麼是 Clock Gating？

**Clock Gating** 是一種硬體功耗優化技術，透過關閉不使用的模組的時鐘來降低功耗。

**原理**：
- 在 CMOS 電路中，**動態功耗** 與時鐘頻率成正比：P = C × V² × f
- 關閉時鐘（f = 0）可以消除動態功耗
- 但靜態功耗（漏電流）仍然存在

**nRF52 的 Clock Gating**：

```c
// 關閉不使用的周邊時鐘
void clock_gating_init(void) {
    // 關閉 UART（除錯完成後）
    NRF_UARTE0->ENABLE = UARTE_ENABLE_ENABLE_Disabled;
    
    // 關閉 SPI（不使用時）
    NRF_SPIM0->ENABLE = SPIM_ENABLE_ENABLE_Disabled;
    NRF_SPIM1->ENABLE = SPIM_ENABLE_ENABLE_Disabled;
    
    // 關閉 I2C（不使用時）
    NRF_TWIM0->ENABLE = TWIM_ENABLE_ENABLE_Disabled;
    
    // 關閉 PWM（不使用時）
    NRF_PWM0->ENABLE = PWM_ENABLE_ENABLE_Disabled;
    NRF_PWM1->ENABLE = PWM_ENABLE_ENABLE_Disabled;
    
    // 關閉 Timer（不使用時）
    NRF_TIMER1->TASKS_STOP = 1;
    NRF_TIMER2->TASKS_STOP = 1;
    
    // 只保留必要的周邊：RTC, GPIOTE, BLE Radio
}

```

**功耗改善**：
- 每個周邊模組：約 10-50 μA
- 關閉 5 個不使用的模組：約 50-250 μA
- **總節省：約 150 μA**

### 動態 Clock Gating

**策略**：根據需要動態開啟和關閉周邊

```c
// I2C 使用範例
void i2c_transaction(uint8_t addr, uint8_t *data, uint8_t len) {
    // 1. 開啟 I2C 時鐘
    NRF_TWIM0->ENABLE = TWIM_ENABLE_ENABLE_Enabled;
    
    // 2. 執行 I2C 傳輸
    i2c_read(addr, 0x00, data, len);
    
    // 3. 關閉 I2C 時鐘
    NRF_TWIM0->ENABLE = TWIM_ENABLE_ENABLE_Disabled;
}

// SPI 使用範例
void spi_transaction(uint8_t *tx_data, uint8_t *rx_data, uint8_t len) {
    // 1. 開啟 SPI 時鐘
    NRF_SPIM0->ENABLE = SPIM_ENABLE_ENABLE_Enabled;
    
    // 2. 執行 SPI 傳輸
    spi_transfer(tx_data, rx_data, len);
    
    // 3. 關閉 SPI 時鐘
    NRF_SPIM0->ENABLE = SPIM_ENABLE_ENABLE_Disabled;
}

```

---

## Power Gating - 關閉不使用的電源

### 什麼是 Power Gating？

**Power Gating** 是一種更激進的功耗優化技術，透過完全切斷不使用的模組的電源來消除靜態功耗（漏電流）。

**原理**：
- **靜態功耗**（漏電流）在 Deep Sleep 時仍然存在
- 完全切斷電源可以消除靜態功耗
- 但需要額外的電源開關電路（PMOS/NMOS）

**實作方式**：

1. **硬體 Power Gating**：使用 Load Switch IC（如 TPS22860）
2. **軟體 Power Gating**：使用 GPIO 控制 Sensor 電源

### 軟體 Power Gating 實作

**電路設計**：

```
VDD (3.3V) ──┬── PMOS ──┬── Sensor VDD
             │          │
             └── GPIO ──┘

```

**韌體實作**：

```c
// Sensor 電源控制
#define ACCEL_POWER_PIN   12
#define HR_POWER_PIN      13
#define LIGHT_POWER_PIN   14

void sensor_power_init(void) {
    // 配置 GPIO 為輸出
    NRF_GPIO->PIN_CNF[ACCEL_POWER_PIN] = (GPIO_PIN_CNF_DIR_Output << GPIO_PIN_CNF_DIR_Pos);
    NRF_GPIO->PIN_CNF[HR_POWER_PIN] = (GPIO_PIN_CNF_DIR_Output << GPIO_PIN_CNF_DIR_Pos);
    NRF_GPIO->PIN_CNF[LIGHT_POWER_PIN] = (GPIO_PIN_CNF_DIR_Output << GPIO_PIN_CNF_DIR_Pos);
    
    // 初始狀態：關閉所有 Sensor
    sensor_power_off_all();
}

void sensor_power_on(uint8_t sensor_id) {
    switch (sensor_id) {
        case SENSOR_ACCEL:
            NRF_GPIO->OUTSET = (1 << ACCEL_POWER_PIN);
            nrf_delay_ms(10);  // 等待電源穩定
            accel_init();
            break;
            
        case SENSOR_HR:
            NRF_GPIO->OUTSET = (1 << HR_POWER_PIN);
            nrf_delay_ms(50);  // 心率感測器需要更長的啟動時間
            hr_sensor_init();
            break;
            
        case SENSOR_LIGHT:
            NRF_GPIO->OUTSET = (1 << LIGHT_POWER_PIN);
            nrf_delay_ms(5);
            light_sensor_init();
            break;
    }
}

void sensor_power_off(uint8_t sensor_id) {
    switch (sensor_id) {
        case SENSOR_ACCEL:
            NRF_GPIO->OUTCLR = (1 << ACCEL_POWER_PIN);
            break;
            
        case SENSOR_HR:
            NRF_GPIO->OUTCLR = (1 << HR_POWER_PIN);
            break;
            
        case SENSOR_LIGHT:
            NRF_GPIO->OUTCLR = (1 << LIGHT_POWER_PIN);
            break;
    }
}

```

**功耗改善**：

| Sensor | Sleep 電流（Power Gating 前） | Sleep 電流（Power Gating 後） | 節省 |
|--------|-------------------------------|-------------------------------|------|
| 加速度計 | 2 μA | 0 μA | 2 μA |
| 心率感測器 | 5 μA | 0 μA | 5 μA |
| 環境光感測器 | 1 μA | 0 μA | 1 μA |
| **總計** | **8 μA** | **0 μA** | **8 μA** |

---

## DVFS - 動態電壓頻率調整

### 什麼是 DVFS？

**DVFS (Dynamic Voltage and Frequency Scaling)** 是一種根據工作負載動態調整 CPU 電壓和頻率的技術。

**原理**：
- **動態功耗** P = C × V² × f
- 降低電壓和頻率可以大幅降低功耗
- 但會降低運算效能

**範例**：nRF52832 的頻率選項

| 頻率 | 電壓 | 電流 | 相對功耗 |
|------|------|------|----------|
| 64 MHz | 3.0 V | 8 mA | 100% |
| 16 MHz | 1.8 V | 2 mA | 25% |
| 1 MHz | 1.8 V | 0.5 mA | 6% |

### DVFS 實作

**策略**：根據任務類型選擇合適的頻率

```c
// CPU 頻率設定
typedef enum {
    CPU_FREQ_64MHZ,  // 高效能模式（BLE 連線、資料處理）
    CPU_FREQ_16MHZ,  // 一般模式（Sensor 讀取）
    CPU_FREQ_1MHZ,   // 低功耗模式（RTC 維護）
} cpu_freq_t;

void cpu_freq_set(cpu_freq_t freq) {
    switch (freq) {
        case CPU_FREQ_64MHZ:
            // 設定 PLL 為 64 MHz
            NRF_CLOCK->TASKS_HFCLKSTART = 1;
            while (!NRF_CLOCK->EVENTS_HFCLKSTARTED);
            break;
            
        case CPU_FREQ_16MHZ:
            // 使用 16 MHz RC oscillator
            NRF_CLOCK->TASKS_HFCLKSTOP = 1;
            // 配置 prescaler
            break;
            
        case CPU_FREQ_1MHZ:
            // 使用 1 MHz RC oscillator
            NRF_CLOCK->TASKS_HFCLKSTOP = 1;
            break;
    }
}

// 使用範例
void ble_connection_handler(void) {
    // BLE 連線時使用高頻率
    cpu_freq_set(CPU_FREQ_64MHZ);

    // 處理 BLE 資料
    process_ble_data();

    // 完成後降低頻率
    cpu_freq_set(CPU_FREQ_16MHZ);
}

```

**功耗改善**：

假設 MCU Active 時間佔 5%：

| 模式 | 頻率 | 電流 | 平均電流（5% Active） |
|------|------|------|-----------------------|
| 固定 64 MHz | 64 MHz | 8 mA | 403 μA |
| DVFS（平均 16 MHz） | 16 MHz | 2 mA | 103 μA |
| **節省** | - | - | **300 μA（74%）** |

---

## 周邊模組的功耗管理

### Sensor 功耗優化

**策略 1：使用 Sensor 的低功耗模式**

大部分 Sensor 都有多種工作模式：

```c
// 加速度計的功耗模式（以 LIS3DH 為例）
typedef enum {
    ACCEL_MODE_POWER_DOWN,     // 0.5 μA
    ACCEL_MODE_LOW_POWER_1HZ,  // 2 μA（1 Hz 取樣）
    ACCEL_MODE_LOW_POWER_10HZ, // 3 μA（10 Hz 取樣）
    ACCEL_MODE_NORMAL_100HZ,   // 11 μA（100 Hz 取樣）
    ACCEL_MODE_HIGH_PERF_400HZ,// 150 μA（400 Hz 取樣）
} accel_mode_t;

void accel_set_mode(accel_mode_t mode) {
    uint8_t ctrl_reg1 = 0;

    switch (mode) {
        case ACCEL_MODE_POWER_DOWN:
            ctrl_reg1 = 0x00;  // Power-down mode
            break;

        case ACCEL_MODE_LOW_POWER_1HZ:
            ctrl_reg1 = 0x17;  // Low-power mode, 1 Hz, XYZ enabled
            break;

        case ACCEL_MODE_LOW_POWER_10HZ:
            ctrl_reg1 = 0x27;  // Low-power mode, 10 Hz, XYZ enabled
            break;

        case ACCEL_MODE_NORMAL_100HZ:
            ctrl_reg1 = 0x57;  // Normal mode, 100 Hz, XYZ enabled
            break;

        case ACCEL_MODE_HIGH_PERF_400HZ:
            ctrl_reg1 = 0x77;  // High-performance mode, 400 Hz, XYZ enabled
            break;
    }

    i2c_write(ACCEL_ADDR, 0x20, ctrl_reg1);
}

```

**策略 2：使用 FIFO 減少讀取次數**

```c
// 配置加速度計的 FIFO
void accel_fifo_init(void) {
    // 啟用 FIFO，Stream mode
    i2c_write(ACCEL_ADDR, 0x2E, 0x80);  // FIFO enable

    // 設定 FIFO 閾值為 25 samples
    i2c_write(ACCEL_ADDR, 0x2E, 0x80 | 0x40 | 25);  // Stream mode, threshold = 25

    // 啟用 FIFO 閾值中斷
    i2c_write(ACCEL_ADDR, 0x22, 0x04);  // INT1 = FIFO threshold
}

// FIFO 中斷處理
void accel_fifo_interrupt_handler(void) {
    uint8_t fifo_data[25 * 6];  // 25 samples × 6 bytes (X, Y, Z)

    // 一次讀取 25 個 samples
    i2c_read(ACCEL_ADDR, 0x28 | 0x80, fifo_data, sizeof(fifo_data));

    // 處理資料
    for (int i = 0; i < 25; i++) {
        process_accel_sample(&fifo_data[i * 6]);
    }
}

```

**功耗改善**：

| 策略 | 讀取頻率 | I2C 開啟次數 | 平均電流 |
|------|----------|--------------|----------|
| 無 FIFO | 100 Hz | 100 次/秒 | 50 μA |
| 使用 FIFO | 100 Hz | 4 次/秒 | 15 μA |
| **節省** | - | - | **35 μA（70%）** |

### Display 功耗優化

**策略 1：使用部分更新**

```c
// OLED Display 部分更新
void display_update_partial(uint8_t x, uint8_t y, uint8_t width, uint8_t height, uint8_t *data) {
    // 只更新變化的區域，而不是整個螢幕
    ssd1306_set_window(x, y, width, height);
    ssd1306_write_data(data, width * height / 8);
}

// 範例：只更新時間顯示
void display_update_time(uint8_t hour, uint8_t minute) {
    uint8_t time_buffer[32];  // 只更新時間區域

    // 渲染時間字串
    render_time(time_buffer, hour, minute);

    // 只更新時間區域（而不是整個螢幕）
    display_update_partial(0, 0, 64, 16, time_buffer);
}

```

**策略 2：降低更新頻率**

```c
// 智慧更新策略
void display_smart_update(void) {
    static uint8_t last_minute = 0xFF;
    uint8_t current_minute = rtc_get_minute();

    // 只有在分鐘變化時才更新
    if (current_minute != last_minute) {
        display_update_time(rtc_get_hour(), current_minute);
        last_minute = current_minute;
    }
}

```

**功耗改善**：

| 策略 | 更新頻率 | 平均電流 |
|------|----------|----------|
| 全螢幕更新，每秒 | 1 Hz | 200 μA |
| 部分更新，每分鐘 | 1/60 Hz | 15 μA |
| **節省** | - | **185 μA（93%）** |

---

## 系統狀態機設計

### 多層狀態機架構

**問題**：如何協調 BLE、Sensor、Display 的狀態？

**解決方案**：設計一個多層狀態機

```c
// 系統狀態定義
typedef enum {
    SYS_STATE_DEEP_SLEEP,    // 深度睡眠
    SYS_STATE_IDLE,          // 閒置（BLE Advertising）
    SYS_STATE_CONNECTED,     // BLE 已連線
    SYS_STATE_ACTIVE,        // 使用者互動
    SYS_STATE_SENSOR_ACTIVE, // Sensor 主動模式
} system_state_t;

// BLE 狀態
typedef enum {
    BLE_STATE_OFF,
    BLE_STATE_ADVERTISING,
    BLE_STATE_CONNECTED,
} ble_state_t;

// Sensor 狀態
typedef enum {
    SENSOR_STATE_OFF,
    SENSOR_STATE_LOW_POWER,
    SENSOR_STATE_ACTIVE,
} sensor_state_t;

// Display 狀態
typedef enum {
    DISPLAY_STATE_OFF,
    DISPLAY_STATE_ON,
} display_state_t;

// 系統狀態
typedef struct {
    system_state_t system_state;
    ble_state_t ble_state;
    sensor_state_t sensor_state;
    display_state_t display_state;
} system_context_t;

static system_context_t sys_ctx = {
    .system_state = SYS_STATE_DEEP_SLEEP,
    .ble_state = BLE_STATE_OFF,
    .sensor_state = SENSOR_STATE_OFF,
    .display_state = DISPLAY_STATE_OFF,
};

```

### 狀態轉換邏輯

```c
// 系統狀態轉換
void system_state_transition(system_state_t new_state) {
    // 離開當前狀態
    switch (sys_ctx.system_state) {
        case SYS_STATE_ACTIVE:
            // 關閉 Display
            display_power_off();
            sys_ctx.display_state = DISPLAY_STATE_OFF;
            break;

        case SYS_STATE_SENSOR_ACTIVE:
            // Sensor 切換到低功耗模式
            accel_set_mode(ACCEL_MODE_LOW_POWER_1HZ);
            sys_ctx.sensor_state = SENSOR_STATE_LOW_POWER;
            break;

        default:
            break;
    }

    // 進入新狀態
    switch (new_state) {
        case SYS_STATE_DEEP_SLEEP:
            // 停止 BLE Advertising
            sd_ble_gap_adv_stop();
            sys_ctx.ble_state = BLE_STATE_OFF;

            // 關閉所有 Sensor
            sensor_power_off_all();
            sys_ctx.sensor_state = SENSOR_STATE_OFF;

            // 進入 Deep Sleep
            enter_deep_sleep();
            break;

        case SYS_STATE_IDLE:
            // 開始 BLE Advertising
            advertising_start();
            sys_ctx.ble_state = BLE_STATE_ADVERTISING;

            // Sensor 低功耗模式
            accel_set_mode(ACCEL_MODE_LOW_POWER_1HZ);
            sys_ctx.sensor_state = SENSOR_STATE_LOW_POWER;
            break;

        case SYS_STATE_CONNECTED:
            // BLE 已連線
            sys_ctx.ble_state = BLE_STATE_CONNECTED;

            // Sensor 保持低功耗模式
            break;

        case SYS_STATE_ACTIVE:
            // 開啟 Display
            display_power_on();
            sys_ctx.display_state = DISPLAY_STATE_ON;

            // Sensor 切換到主動模式
            accel_set_mode(ACCEL_MODE_NORMAL_100HZ);
            sys_ctx.sensor_state = SENSOR_STATE_ACTIVE;
            break;

        case SYS_STATE_SENSOR_ACTIVE:
            // Sensor 切換到主動模式（例如運動偵測）
            accel_set_mode(ACCEL_MODE_NORMAL_100HZ);
            sys_ctx.sensor_state = SENSOR_STATE_ACTIVE;
            break;
    }

    sys_ctx.system_state = new_state;
}

```

### 事件驅動的狀態轉換

```c
// 系統事件定義
typedef enum {
    EVENT_BUTTON_PRESSED,
    EVENT_MOTION_DETECTED,
    EVENT_BLE_CONNECTED,
    EVENT_BLE_DISCONNECTED,
    EVENT_TIMEOUT,
} system_event_t;

// 事件處理
void system_event_handler(system_event_t event) {
    switch (sys_ctx.system_state) {
        case SYS_STATE_DEEP_SLEEP:
            if (event == EVENT_BUTTON_PRESSED) {
                system_state_transition(SYS_STATE_ACTIVE);
            } else if (event == EVENT_MOTION_DETECTED) {
                system_state_transition(SYS_STATE_SENSOR_ACTIVE);
            }
            break;

        case SYS_STATE_IDLE:
            if (event == EVENT_BLE_CONNECTED) {
                system_state_transition(SYS_STATE_CONNECTED);
            } else if (event == EVENT_BUTTON_PRESSED) {
                system_state_transition(SYS_STATE_ACTIVE);
            } else if (event == EVENT_TIMEOUT) {
                // 10 分鐘沒有活動，進入 Deep Sleep
                system_state_transition(SYS_STATE_DEEP_SLEEP);
            }
            break;

        case SYS_STATE_CONNECTED:
            if (event == EVENT_BLE_DISCONNECTED) {
                system_state_transition(SYS_STATE_IDLE);
            } else if (event == EVENT_BUTTON_PRESSED) {
                system_state_transition(SYS_STATE_ACTIVE);
            }
            break;

        case SYS_STATE_ACTIVE:
            if (event == EVENT_TIMEOUT) {
                // 5 秒沒有互動，返回 IDLE
                if (sys_ctx.ble_state == BLE_STATE_CONNECTED) {
                    system_state_transition(SYS_STATE_CONNECTED);
                } else {
                    system_state_transition(SYS_STATE_IDLE);
                }
            }
            break;

        case SYS_STATE_SENSOR_ACTIVE:
            if (event == EVENT_TIMEOUT) {
                // 10 秒沒有運動，返回 IDLE
                system_state_transition(SYS_STATE_IDLE);
            } else if (event == EVENT_BUTTON_PRESSED) {
                system_state_transition(SYS_STATE_ACTIVE);
            }
            break;
    }
}

```

---

## 真實案例：從 2 mA 到 22 μA 的優化過程

回到文章開頭的故事。讓我詳細說明我們是如何將平均功耗從 2 mA 優化到 22 μA 的。

### 初始狀態（2 mA）

**功耗分析**：

| 模組 | 平均電流 | 佔比 |
|------|----------|------|
| MCU | 800 μA | 40% |
| BLE Radio | 300 μA | 15% |
| 加速度計 | 150 μA | 7.5% |
| 心率感測器 | 400 μA | 20% |
| 環境光感測器 | 50 μA | 2.5% |
| Display | 250 μA | 12.5% |
| 其他 | 50 μA | 2.5% |
| **總計** | **2000 μA** | **100%** |

### 優化步驟

#### 步驟 1：實作 Clock Gating

**修改**：關閉不使用的周邊時鐘

**功耗改善**：
- 節省：150 μA
- 新總計：1850 μA

#### 步驟 2：實作 Power Gating

**修改**：使用 GPIO 控制 Sensor 電源，不使用時完全關閉

**功耗改善**：
- 心率感測器：從持續開啟改為每 10 分鐘測量 30 秒
- 平均電流：400 μA → 20 μA
- 節省：380 μA
- 新總計：1470 μA

#### 步驟 3：實作 DVFS

**修改**：大部分時間使用 16 MHz，只在 BLE 連線時使用 64 MHz

**功耗改善**：
- MCU：800 μA → 200 μA
- 節省：600 μA
- 新總計：870 μA

#### 步驟 4：優化 BLE 參數

**修改**：
- Advertising Interval：100 ms → 1000 ms
- Connection Interval：20 ms → 200 ms，Slave Latency = 4

**功耗改善**：
- BLE Radio：300 μA → 30 μA
- 節省：270 μA
- 新總計：600 μA

#### 步驟 5：優化 Sensor 使用

**修改**：
- 加速度計：使用 Motion Detection + FIFO
- 環境光感測器：從每秒測量改為每 10 分鐘測量

**功耗改善**：
- 加速度計：150 μA → 15 μA
- 環境光感測器：50 μA → 1 μA
- 節省：184 μA
- 新總計：416 μA

#### 步驟 6：優化 Display 使用

**修改**：
- 使用部分更新
- 降低更新頻率（每分鐘而不是每秒）
- 縮短開啟時間（5 秒而不是 10 秒）

**功耗改善**：
- Display：250 μA → 12 μA
- 節省：238 μA
- 新總計：178 μA

#### 步驟 7：系統級優化

**修改**：
- 實作狀態機，協調各模組的開啟和關閉
- 使用智慧喚醒策略
- 優化 GPIO 配置（Disconnect unused pins）

**功耗改善**：
- 系統級優化：178 μA → 22 μA
- 節省：156 μA
- **最終總計：22 μA**

### 最終結果

**功耗分析**：

| 模組 | 平均電流 | 佔比 |
|------|----------|------|
| MCU | 5 μA | 23% |
| BLE Radio | 3 μA | 14% |
| 加速度計 | 2 μA | 9% |
| 心率感測器 | 2 μA | 9% |
| 環境光感測器 | 0.5 μA | 2% |
| Display | 1 μA | 5% |
| 其他（LDO, RTC） | 8.5 μA | 38% |
| **總計** | **22 μA** | **100%** |

**電池續航**：

```
Battery Life = 200 mAh / 0.022 mA = 9,091 hours ≈ 379 days ≈ 12.5 months

```

**優化總結**：

| 項目 | 優化前 | 優化後 | 改善 |
|------|--------|--------|------|
| 平均電流 | 2000 μA | 22 μA | **98.9%** |
| 電池續航 | 4 天 | 12.5 個月 | **94 倍** |

---

## 功耗測量與驗證

### 測量工具

1. **Nordic Power Profiler Kit (PPK2)**：

   - 測量範圍：1 μA - 1 A
   - 取樣率：100 kHz
   - 可以同步顯示 UART Log

2. **Keysight N6705B**：

   - 高精度功耗分析儀
   - 可以測量瞬時電流波形

### 測量方法

**長期功耗測量**：

```c
// 測量 24 小時的平均功耗
void power_measurement_24h(void) {
    // 1. 使用 PPK2 記錄 24 小時的電流
    // 2. 模擬真實使用場景：
    //    - 每小時按鍵 1 次（查看時間）
    //    - 每 4 小時連線 1 次（同步資料）
    //    - 每天運動 2 小時（計步）
    //    - 每 10 分鐘測量心率 1 次

    // 3. 分析結果：
    //    - 平均電流
    //    - 各狀態的時間佔比
    //    - 功耗熱點
}

```

**狀態功耗測量**：

```c
// 測量各個狀態的功耗
void power_measurement_states(void) {
    // Deep Sleep
    mark_state("Deep Sleep Start");
    system_state_transition(SYS_STATE_DEEP_SLEEP);
    nrf_delay_ms(10000);  // 10 秒
    mark_state("Deep Sleep End");

    // Idle (Advertising)
    mark_state("Idle Start");
    system_state_transition(SYS_STATE_IDLE);
    nrf_delay_ms(10000);
    mark_state("Idle End");

    // Connected
    mark_state("Connected Start");
    // 等待 BLE 連線...
    nrf_delay_ms(10000);
    mark_state("Connected End");

    // Active (Display On)
    mark_state("Active Start");
    system_state_transition(SYS_STATE_ACTIVE);
    nrf_delay_ms(5000);
    mark_state("Active End");
}

```

---

## 實務建議

### 功耗優化流程

1. **測量基線**：

   - 使用功耗分析儀測量初始功耗
   - 識別功耗熱點（哪個模組佔用最多功耗？）

2. **設定目標**：

   - 根據電池容量和續航要求計算目標功耗
   - 分配各模組的功耗預算

3. **逐步優化**：

   - 從功耗最高的模組開始優化
   - 每次優化後測量驗證

4. **系統整合**：

   - 設計狀態機協調各模組
   - 實作智慧喚醒策略

5. **長期測試**：

   - 測量 24 小時或更長時間的平均功耗
   - 模擬真實使用場景

### 功耗優化檢查清單

**硬體層面**：
- ✅ 使用低功耗的 MCU 和 Sensor
- ✅ 設計 Power Gating 電路
- ✅ 選擇低靜態功耗的 LDO
- ✅ 優化 PCB layout（減少漏電流）

**韌體層面**：
- ✅ 實作 Clock Gating
- ✅ 實作 Power Gating
- ✅ 實作 DVFS
- ✅ 優化 BLE 參數
- ✅ 使用 Sensor 的低功耗模式
- ✅ 設計狀態機管理系統狀態

**系統層面**：
- ✅ 協調各模組的開啟和關閉
- ✅ 使用智慧喚醒策略
- ✅ 最大化 Deep Sleep 時間
- ✅ 優化演算法（減少計算量）

---

## 總結

系統級功耗優化是一個複雜的系統工程，需要軟硬體協同設計。透過 Clock Gating、Power Gating、DVFS、狀態機設計等技術，我們成功將 IoT 裝置的平均功耗從 2 mA 降低到 22 μA，電池續航從 4 天延長到 12.5 個月，改善了 94 倍。

**核心原則**：
1. **測量驅動**：使用功耗分析儀識別功耗熱點
2. **分而治之**：逐個模組優化
3. **系統協調**：使用狀態機協調各模組
4. **智慧喚醒**：只在需要時開啟模組
5. **持續驗證**：每次優化後測量驗證

下一篇文章，我們將探討 **BLE 除錯實戰**，看看如何使用 Sniffer、Log、測試工具來追蹤和解決 BLE 問題。

---

## 參考資料

1. ARM. (2015). *Cortex-M4 Technical Reference Manual*. <https://developer.arm.com/documentation/100166/0001>
2. Nordic Semiconductor. (2016). *nRF52832 Power Management*. <https://infocenter.nordicsemi.com/topic/com.nordic.infocenter.nrf52832.ps.v1.1/power.html>
3. Texas Instruments. (2015). *System-Level Power Optimization for IoT Devices*. <https://www.ti.com/lit/wp/sway010/sway010.pdf>
4. STMicroelectronics. (2014). *LIS3DH MEMS Digital Output Motion Sensor*. <https://www.st.com/resource/en/datasheet/lis3dh.pdf>

---

**版權聲明**

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。
**作者**：Danny Jiang
**出處**：<https://github.com/djiangtw/tech-column-public>

```
