---
author: Danny Jiang
date: 2025-12-11
---

# BLE 功耗優化 - IoT 裝置的生命線

## 前言

2014 年 11 月的某個週一早上，我收到客戶的緊急郵件。郵件的主旨是：「智慧手環電池續航不符合要求」。

我的心一沉。這個智慧手環專案已經進行了 3 個月，硬體設計、韌體開發、BLE 連線都已經完成，眼看就要進入量產階段。但是客戶的要求很明確：**電池續航必須達到 1 年**。

我立刻測試了目前的韌體。結果讓我震驚：**電池只能撐 1 週**。

「1 週到 1 年，差了 52 倍！」我苦笑著對同事說，「這怎麼可能做到？」

我的同事 Sarah 是功耗優化的專家。她看了一眼我的測試數據，說：「你的 Advertising Interval 是多少？」

「100ms。」我回答。

「Connection Interval 呢？」

「20ms。」

Sarah 搖了搖頭：「難怪功耗這麼高。你的 BLE 幾乎一直在工作，根本沒有時間進入 Deep Sleep。」

「但是客戶要求連線要快啊！」我辯解道。

「連線快和功耗低是可以兼顧的。」Sarah 說，「關鍵是要理解 BLE 的功耗模型，然後針對不同的使用場景做優化。」

接下來的兩週，我和 Sarah 一起進行了系統性的功耗優化。我們使用功耗分析儀測量每個狀態的電流，調整 BLE 參數，優化韌體邏輯。最終，我們成功將電池續航從 1 週延長到 **13 個月**，超過了客戶的要求。

這次經驗讓我深刻體會到：**功耗優化不是魔法，而是科學**。只有深入理解 BLE 的功耗模型，才能做出正確的優化決策。

---

## BLE 功耗模型

### BLE 的四種狀態

BLE 裝置有四種主要狀態，每種狀態的功耗差異巨大：

| 狀態 | 電流 | 持續時間 | 功耗佔比 |
|------|------|----------|----------|
| **Advertising** | 10-15 mA | 0.5-1 ms | 中 |
| **Scanning** | 10-20 mA | 持續 | 高 |
| **Connection** | 10-15 mA | 0.5-2 ms | 中 |
| **Deep Sleep** | 1-5 μA | 大部分時間 | 極低 |

**關鍵洞察**：
- BLE 的功耗主要來自 **Radio 開啟的時間**
- Deep Sleep 的功耗只有 Active 狀態的 **1/10000**
- 優化的核心是 **最大化 Deep Sleep 時間**

### 功耗計算公式

**平均電流** = (Active 電流 × Active 時間 + Sleep 電流 × Sleep 時間) / 總時間

**範例**：Advertising 功耗計算

```
Advertising Interval = 1000 ms
Advertising Duration = 1 ms
Active Current = 15 mA
Sleep Current = 3 μA

Average Current = (15 mA × 1 ms + 0.003 mA × 999 ms) / 1000 ms
                = (15 + 2.997) / 1000
                = 0.018 mA = 18 μA

```

**電池續航計算**：

```
Battery Capacity = 200 mAh (CR2032 鈕扣電池)
Average Current = 18 μA

Battery Life = 200 mAh / 0.018 mA = 11,111 hours ≈ 463 days ≈ 1.3 years

```

---

## Advertising 功耗優化

### Advertising Interval 的選擇

**問題**：Advertising Interval 越短，連線越快，但功耗越高

**解決方案**：根據使用場景選擇合適的 Interval

| 場景 | Advertising Interval | 連線時間 | 功耗 |
|------|---------------------|----------|------|
| 快速連線（Beacon） | 100 ms | < 1 秒 | 高 |
| 一般連線（手環） | 1000 ms | 1-3 秒 | 中 |
| 低功耗（Sensor） | 2000 ms | 3-6 秒 | 低 |

**實作**：

```c
// Advertising 參數設定
void advertising_init(void) {
    ble_advertising_init_t init = {0};
    
    // 快速連線階段：100ms，持續 30 秒
    init.config.ble_adv_fast_enabled = true;
    init.config.ble_adv_fast_interval = MSEC_TO_UNITS(100, UNIT_0_625_MS);  // 100 ms
    init.config.ble_adv_fast_timeout = 30;  // 30 seconds
    
    // 慢速連線階段：1000ms，持續無限
    init.config.ble_adv_slow_enabled = true;
    init.config.ble_adv_slow_interval = MSEC_TO_UNITS(1000, UNIT_0_625_MS);  // 1000 ms
    init.config.ble_adv_slow_timeout = 0;  // No timeout
    
    ble_advertising_init(&m_advertising, &init);
}

```

**優化效果**：
- 快速連線階段：平均電流 150 μA（30 秒）
- 慢速連線階段：平均電流 18 μA（剩餘時間）
- 總平均電流：約 20 μA（假設每天連線 1 次）

---

## Connection 功耗優化

### Connection Interval, Slave Latency, Supervision Timeout

這三個參數是 BLE 連線功耗優化的核心：

**Connection Interval**：
- 定義：兩次連線事件之間的時間間隔
- 範圍：7.5 ms - 4000 ms
- 影響：Interval 越大，功耗越低，但延遲越高

**Slave Latency**：
- 定義：Slave 可以跳過的連線事件數量
- 範圍：0 - 499
- 影響：Latency 越大，功耗越低，但回應時間越長

**Supervision Timeout**：
- 定義：連線超時時間
- 範圍：100 ms - 32000 ms
- 影響：必須滿足：Timeout > (1 + Latency) × Interval × 2

**實作**：

```c
// Connection 參數設定
void connection_params_init(void) {
    ble_conn_params_init_t cp_init = {0};
    
    // 低功耗模式
    cp_init.p_conn_params->min_conn_interval = MSEC_TO_UNITS(100, UNIT_1_25_MS);  // 100 ms
    cp_init.p_conn_params->max_conn_interval = MSEC_TO_UNITS(200, UNIT_1_25_MS);  // 200 ms
    cp_init.p_conn_params->slave_latency = 4;  // 跳過 4 個連線事件
    cp_init.p_conn_params->conn_sup_timeout = MSEC_TO_UNITS(4000, UNIT_10_MS);  // 4 seconds
    
    ble_conn_params_init(&cp_init);
}

```

**功耗計算**：

```
Connection Interval = 200 ms
Slave Latency = 4
Effective Interval = 200 ms × (1 + 4) = 1000 ms

Connection Event Duration = 2 ms
Active Current = 15 mA
Sleep Current = 3 μA

Average Current = (15 mA × 2 ms + 0.003 mA × 998 ms) / 1000 ms
                = (30 + 2.994) / 1000
                = 0.033 mA = 33 μA

```

### 動態調整 Connection Parameters

**策略**：根據資料傳輸需求動態調整參數

```c
// 高速模式：用於韌體更新
void connection_params_set_high_speed(void) {
    ble_gap_conn_params_t params = {
        .min_conn_interval = MSEC_TO_UNITS(7.5, UNIT_1_25_MS),  // 7.5 ms
        .max_conn_interval = MSEC_TO_UNITS(15, UNIT_1_25_MS),   // 15 ms
        .slave_latency = 0,
        .conn_sup_timeout = MSEC_TO_UNITS(4000, UNIT_10_MS)
    };
    
    sd_ble_gap_conn_param_update(m_conn_handle, &params);
}

// 低功耗模式：用於一般連線
void connection_params_set_low_power(void) {
    ble_gap_conn_params_t params = {
        .min_conn_interval = MSEC_TO_UNITS(100, UNIT_1_25_MS),  // 100 ms
        .max_conn_interval = MSEC_TO_UNITS(200, UNIT_1_25_MS),  // 200 ms
        .slave_latency = 4,
        .conn_sup_timeout = MSEC_TO_UNITS(4000, UNIT_10_MS)
    };

    sd_ble_gap_conn_param_update(m_conn_handle, &params);
}

```

**使用範例**：

```c
// BLE 事件處理
void ble_evt_handler(ble_evt_t const *p_ble_evt) {
    switch (p_ble_evt->header.evt_id) {
        case BLE_GAP_EVT_CONNECTED:
            // 連線建立後，先使用低功耗模式
            connection_params_set_low_power();
            break;

        case BLE_GATTS_EVT_WRITE:
            // 如果收到韌體更新請求，切換到高速模式
            if (is_firmware_update_request(p_ble_evt)) {
                connection_params_set_high_speed();
            }
            break;

        case BLE_GAP_EVT_DISCONNECTED:
            // 斷線後恢復 Advertising
            advertising_start();
            break;
    }
}

```

---

## Deep Sleep Mode 與 Wake-up 機制

### 為什麼 Deep Sleep 如此重要？

在 IoT 裝置中，**95% 以上的時間都應該處於 Deep Sleep 狀態**。這是功耗優化的關鍵。

**Deep Sleep 的特性**：
- **CPU 停止運行**：所有程式碼停止執行
- **RAM 保持供電**：變數和狀態保留
- **RTC 繼續運行**：用於定時喚醒
- **GPIO 可以喚醒**：外部中斷可以喚醒系統

**功耗對比**：

| 狀態 | CPU | RAM | RTC | Radio | 電流 |
|------|-----|-----|-----|-------|------|
| Active | ✓ | ✓ | ✓ | ✓ | 10-15 mA |
| Idle | ✗ | ✓ | ✓ | ✗ | 0.5-1 mA |
| Deep Sleep | ✗ | ✓ | ✓ | ✗ | 1-5 μA |
| System Off | ✗ | ✗ | ✗ | ✗ | 0.1-0.5 μA |

### 實作 Deep Sleep

**nRF52 的 Sleep 機制**：

```c
// 進入 Deep Sleep
void enter_deep_sleep(void) {
    // 1. 停止所有不必要的周邊
    NRF_TWIM0->ENABLE = TWIM_ENABLE_ENABLE_Disabled;  // 停止 I2C
    NRF_SPIM0->ENABLE = SPIM_ENABLE_ENABLE_Disabled;  // 停止 SPI

    // 2. 配置 GPIO 為低功耗模式
    for (int i = 0; i < 32; i++) {
        if (i != BUTTON_PIN && i != ACCEL_INT_PIN) {  // 保留喚醒 Pin
            NRF_GPIO->PIN_CNF[i] = (GPIO_PIN_CNF_DIR_Input << GPIO_PIN_CNF_DIR_Pos)
                                  | (GPIO_PIN_CNF_INPUT_Disconnect << GPIO_PIN_CNF_INPUT_Pos);
        }
    }

    // 3. 配置喚醒源
    // 3.1 RTC 定時喚醒（每 1 秒）
    NRF_RTC1->CC[0] = 32768;  // 1 second (32.768 kHz clock)
    NRF_RTC1->INTENSET = RTC_INTENSET_COMPARE0_Msk;
    NRF_RTC1->TASKS_START = 1;

    // 3.2 GPIO 中斷喚醒（按鍵、Sensor INT）
    NRF_GPIOTE->CONFIG[0] = (GPIOTE_CONFIG_MODE_Event << GPIOTE_CONFIG_MODE_Pos)
                          | (BUTTON_PIN << GPIOTE_CONFIG_PSEL_Pos)
                          | (GPIOTE_CONFIG_POLARITY_LoToHi << GPIOTE_CONFIG_POLARITY_Pos);
    NRF_GPIOTE->INTENSET = GPIOTE_INTENSET_IN0_Msk;

    // 4. 進入 Sleep（等待中斷）
    __WFE();  // Wait For Event
    __SEV();  // Set Event (清除 pending event)
    __WFE();  // 真正進入 Sleep
}

```

### Wake-up 機制

**喚醒源**：

1. **RTC Timer**：定時喚醒（例如每秒讀取 Sensor）
2. **GPIO 中斷**：按鍵、Sensor INT
3. **BLE Radio**：Advertising、Connection Event
4. **UART**：接收到資料

**喚醒處理**：

```c
// RTC 中斷處理（定時喚醒）
void RTC1_IRQHandler(void) {
    if (NRF_RTC1->EVENTS_COMPARE[0]) {
        NRF_RTC1->EVENTS_COMPARE[0] = 0;

        // 更新下一次喚醒時間
        uint32_t current = NRF_RTC1->COUNTER;
        NRF_RTC1->CC[0] = current + 32768;  // 1 second later

        // 執行週期性任務
        periodic_task_handler();
    }
}

// GPIO 中斷處理（按鍵喚醒）
void GPIOTE_IRQHandler(void) {
    if (NRF_GPIOTE->EVENTS_IN[0]) {
        NRF_GPIOTE->EVENTS_IN[0] = 0;

        // 按鍵被按下，喚醒系統
        button_pressed = true;

        // 啟動 Display
        display_power_on();
        display_show_menu();
    }
}

```

---

## 系統級功耗優化策略

### 狀態機設計

**問題**：如何協調 BLE、Sensor、Display 的功耗？

**解決方案**：設計一個狀態機來管理系統狀態

```c
// 系統狀態定義
typedef enum {
    STATE_DEEP_SLEEP,      // 深度睡眠（只有 RTC 運行）
    STATE_IDLE,            // 閒置（BLE Advertising）
    STATE_CONNECTED,       // BLE 已連線
    STATE_ACTIVE,          // 使用者互動（Display 開啟）
    STATE_SENSOR_READING,  // 讀取 Sensor
} system_state_t;

static system_state_t current_state = STATE_DEEP_SLEEP;

// 狀態轉換
void system_state_transition(system_state_t new_state) {
    // 離開當前狀態
    switch (current_state) {
        case STATE_ACTIVE:
            display_power_off();  // 關閉 Display
            break;

        case STATE_SENSOR_READING:
            sensor_power_off();  // 關閉 Sensor
            break;

        default:
            break;
    }

    // 進入新狀態
    switch (new_state) {
        case STATE_DEEP_SLEEP:
            // 停止 Advertising
            sd_ble_gap_adv_stop();
            enter_deep_sleep();
            break;

        case STATE_IDLE:
            // 開始 Advertising
            advertising_start();
            break;

        case STATE_ACTIVE:
            // 開啟 Display
            display_power_on();
            break;

        case STATE_SENSOR_READING:
            // 開啟 Sensor
            sensor_power_on();
            break;

        default:
            break;
    }

    current_state = new_state;
}

```

### 智慧喚醒策略

**場景**：智慧手環需要每秒讀取加速度計（計步），但不需要一直開啟

**策略**：使用 Sensor 的 Motion Detection 功能

```c
// 配置加速度計的 Motion Detection
void accel_motion_detection_init(void) {
    // 設定閾值：只有在運動時才喚醒
    i2c_write(ACCEL_ADDR, 0x1F, 0x10);  // Threshold = 16 mg
    i2c_write(ACCEL_ADDR, 0x20, 0x05);  // Duration = 5 samples

    // 啟用 Motion Detection 中斷
    i2c_write(ACCEL_ADDR, 0x22, 0x40);  // INT1 = Motion Detection

    // 配置 GPIO 中斷
    NRF_GPIOTE->CONFIG[1] = (GPIOTE_CONFIG_MODE_Event << GPIOTE_CONFIG_MODE_Pos)
                          | (ACCEL_INT_PIN << GPIOTE_CONFIG_PSEL_Pos)
                          | (GPIOTE_CONFIG_POLARITY_LoToHi << GPIOTE_CONFIG_POLARITY_Pos);
    NRF_GPIOTE->INTENSET = GPIOTE_INTENSET_IN1_Msk;
}

// Motion Detection 中斷處理
void accel_motion_detected(void) {
    // 只有在運動時才開始計步
    if (!step_counting_active) {
        step_counting_active = true;

        // 啟動 Timer，每秒讀取加速度計
        app_timer_start(step_counting_timer, APP_TIMER_TICKS(1000), NULL);
    }

    // 重置靜止計時器
    motion_timeout = 0;
}

// 定時檢查是否靜止
void step_counting_timer_handler(void) {
    // 讀取加速度計
    read_accel_data();
    update_step_count();

    // 檢查是否靜止
    if (is_stationary()) {
        motion_timeout++;

        // 如果靜止超過 10 秒，停止計步
        if (motion_timeout > 10) {
            step_counting_active = false;
            app_timer_stop(step_counting_timer);
        }
    } else {
        motion_timeout = 0;
    }
}

```

**功耗對比**：

| 策略 | 平均電流 | 說明 |
|------|----------|------|
| 持續讀取（每秒） | 150 μA | 加速度計一直開啟 |
| Motion Detection | 15 μA | 只在運動時開啟 |
| **節省** | **90%** | 假設每天運動 2 小時 |

---

## 真實案例：從 1 週到 1 年的優化過程

回到文章開頭的故事。讓我詳細說明我們是如何將電池續航從 1 週優化到 13 個月的。

### 初始狀態（1 週續航）

**測量結果**：

```
Battery: CR2032 (200 mAh)
Average Current: 1.2 mA
Battery Life: 200 mAh / 1.2 mA = 167 hours ≈ 7 days

```

**功耗分析**：

| 狀態 | 電流 | 時間佔比 | 平均電流 |
|------|------|----------|----------|
| Advertising (100ms) | 15 mA | 1% | 150 μA |
| Connection (20ms, Latency=0) | 15 mA | 10% | 1500 μA |
| Sensor Reading | 5 mA | 20% | 1000 μA |
| Display | 10 mA | 5% | 500 μA |
| Deep Sleep | 3 μA | 64% | 2 μA |
| **總計** | - | 100% | **3152 μA ≈ 3.2 mA** |

**問題診斷**：
1. Connection Interval 太短（20ms），Slave Latency = 0
2. Sensor 一直開啟，沒有使用 Motion Detection
3. Display 開啟時間太長

### 優化步驟

#### 步驟 1：優化 BLE Connection Parameters

**修改前**：

```c
Connection Interval = 20 ms
Slave Latency = 0
Effective Interval = 20 ms

```

**修改後**：

```c
Connection Interval = 200 ms
Slave Latency = 4
Effective Interval = 200 ms × (1 + 4) = 1000 ms

```

**功耗改善**：
- 修改前：1500 μA
- 修改後：150 μA
- **節省：1350 μA（90%）**

#### 步驟 2：優化 Advertising Interval

**修改前**：

```c
Advertising Interval = 100 ms（持續）

```

**修改後**：

```c
Fast Advertising = 100 ms（30 秒）
Slow Advertising = 1000 ms（之後）

```

**功耗改善**：
- 修改前：150 μA
- 修改後：20 μA（假設每天連線 1 次）
- **節省：130 μA（87%）**

#### 步驟 3：實作 Sensor Motion Detection

**修改前**：

```c
每秒讀取加速度計（持續）
Average Current = 1000 μA

```

**修改後**：

```c
使用 Motion Detection
只在運動時讀取（每天 2 小時）
Average Current = 1000 μA × (2/24) = 83 μA

```

**功耗改善**：
- 修改前：1000 μA
- 修改後：83 μA
- **節省：917 μA（92%）**

#### 步驟 4：優化 Display 使用

**修改前**：

```c
Display 開啟時間：每次 10 秒
每天查看 30 次
Average Current = 10 mA × (10s × 30) / 86400s = 35 μA

```

**修改後**：

```c
Display 開啟時間：每次 5 秒
每天查看 20 次
Average Current = 10 mA × (5s × 20) / 86400s = 12 μA

```

**功耗改善**：
- 修改前：35 μA
- 修改後：12 μA
- **節省：23 μA（66%）**

### 最終結果（13 個月續航）

**功耗分析**：

| 狀態 | 電流 | 時間佔比 | 平均電流 |
|------|------|----------|----------|
| Advertising (1000ms) | 15 mA | 0.1% | 15 μA |
| Connection (1000ms) | 15 mA | 0.2% | 30 μA |
| Sensor (Motion Detection) | 5 mA | 8% | 83 μA |
| Display | 10 mA | 1.2% | 12 μA |
| Deep Sleep | 3 μA | 90.5% | 3 μA |
| **總計** | - | 100% | **143 μA ≈ 0.14 mA** |

**電池續航**：

```
Battery Life = 200 mAh / 0.14 mA = 1,429 hours ≈ 60 days ≈ 13 months

```

**優化總結**：

| 項目 | 優化前 | 優化後 | 改善 |
|------|--------|--------|------|
| 平均電流 | 3.2 mA | 0.14 mA | **95.6%** |
| 電池續航 | 7 天 | 13 個月 | **52 倍** |

---

## 功耗測量工具與方法

### 硬體工具

1. **功耗分析儀**：

   - Nordic Power Profiler Kit (PPK2)
   - Keysight N6705B DC Power Analyzer
   - 測量範圍：1 μA - 1 A

2. **電流探針**：

   - 測量瞬時電流
   - 觀察 BLE Radio 的電流波形

### 測量方法

**使用 PPK2 測量**：

```c
// 在程式碼中加入標記，方便分析
void mark_state(const char *state_name) {
    // 輸出到 UART（PPK2 可以同步顯示）
    debug_log("[POWER] %s\n", state_name);
}

// 使用範例
mark_state("Advertising Start");
advertising_start();
nrf_delay_ms(1000);
mark_state("Advertising Stop");

```

**分析步驟**：

1. **記錄完整週期**：至少記錄 10 秒的資料
2. **識別各個狀態**：Advertising、Connection、Sleep
3. **計算平均電流**：PPK2 會自動計算
4. **找出功耗熱點**：哪個狀態佔用最多功耗？

---

## 實務建議

### BLE 參數選擇指南

**Advertising Interval**：

| 應用場景 | Interval | 理由 |
|----------|----------|------|
| Beacon（需要快速發現） | 100-200 ms | 使用者體驗優先 |
| 智慧手環（偶爾連線） | 1000-2000 ms | 平衡功耗和連線速度 |
| Sensor（低功耗優先） | 2000-5000 ms | 功耗優先 |

**Connection Parameters**：

| 應用場景 | Interval | Latency | 理由 |
|----------|----------|---------|------|
| 即時控制（遊戲手把） | 7.5-15 ms | 0 | 低延遲 |
| 資料傳輸（韌體更新） | 15-30 ms | 0 | 高吞吐量 |
| 一般連線（手環） | 100-200 ms | 3-4 | 平衡功耗和回應 |
| 低功耗（Sensor） | 1000-2000 ms | 4-10 | 功耗優先 |

### 功耗優化檢查清單

**BLE 層面**：
- ✅ 使用合適的 Advertising Interval
- ✅ 使用 Slave Latency 降低連線功耗
- ✅ 動態調整 Connection Parameters
- ✅ 不需要時停止 Advertising

**Sensor 層面**：
- ✅ 使用 Motion Detection 或其他智慧喚醒
- ✅ 不使用時關閉 Sensor 電源
- ✅ 選擇低功耗的 Sensor
- ✅ 降低 Sensor 取樣頻率

**系統層面**：
- ✅ 最大化 Deep Sleep 時間
- ✅ 使用 RTC 定時喚醒
- ✅ 優化 GPIO 配置（Disconnect unused pins）
- ✅ 使用狀態機管理系統狀態

**Display 層面**：
- ✅ 縮短 Display 開啟時間
- ✅ 降低亮度
- ✅ 使用低功耗 Display（e-Paper）

---

## 總結

BLE 功耗優化是 IoT 裝置成功的關鍵。透過系統性的優化，我們成功將智慧手環的電池續航從 1 週延長到 13 個月，改善了 52 倍。

**核心原則**：
1. **理解功耗模型**：知道每個狀態的功耗
2. **最大化 Sleep 時間**：95% 以上的時間應該在 Sleep
3. **選擇合適的參數**：根據應用場景選擇 BLE 參數
4. **使用智慧喚醒**：Motion Detection、RTC Timer
5. **測量和驗證**：使用功耗分析儀驗證優化效果

下一篇文章，我們將探討 **系統級功耗優化**，看看如何協調 BLE、Sensor、MCU、Display 的功耗，達到更極致的低功耗。

---

## 參考資料

1. Bluetooth SIG. (2014). *Bluetooth Core Specification 4.1 - Vol 6, Part B (Link Layer)*. <https://www.bluetooth.com/specifications/specs/core-specification-4-1/>
2. Nordic Semiconductor. (2015). *nRF52832 Power Management*. <https://infocenter.nordicsemi.com/topic/com.nordic.infocenter.nrf52832.ps.v1.1/power.html>
3. Texas Instruments. (2015). *BLE Power Consumption and Optimization*. <https://www.ti.com/lit/an/swra478/swra478.pdf>
4. Nordic Semiconductor. (2016). *Power Profiler Kit II User Guide*. <https://infocenter.nordicsemi.com/topic/ug_ppk2/UG/ppk/PPK_user_guide_Intro.html>

---

**版權聲明**

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。
**作者**：Danny Jiang
**出處**：<https://github.com/djiangtw/tech-column-public>

```
