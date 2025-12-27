---
author: Danny Jiang
date: 2025-12-11
---

# 智慧手錶案例研究 - 完整的 IoT 產品開發

## 前言

2014 年 6 月，我接到了一個令人興奮又充滿挑戰的任務：帶領團隊開發一款智慧手錶。

那是一個週一早上，我剛到公司，就被叫進了會議室。產品經理 Lisa 正在投影幕上展示一份市場分析報告：

"Apple Watch 預計明年發布，Samsung Gear 已經上市，Pebble 在 Kickstarter 上募資成功。智慧手錶市場正在爆發，我們必須抓住這個機會。"

她轉向我："Danny，我們需要在 9 個月內推出一款智慧手錶。你覺得可行嗎？"

我愣了一下。9 個月？從零開始開發一款智慧手錶？這意味著我們需要整合：

- **BLE 通訊**：與手機連線，同步通知、電話、訊息
- **MIPI Display**：1.3 吋圓形 AMOLED 顯示器
- **多種感測器**：心率、加速度、陀螺儀、氣壓計
- **SPI Flash**：儲存字型、圖片、資料
- **電池管理**：200 mAh 電池，續航 > 3 天
- **防水設計**：IP67 防水防塵
- **OTA 更新**：支援無線韌體更新

而且，這還只是硬體部分。我們還需要開發：

- **RTOS 韌體**：多工處理、記憶體管理、功耗優化
- **UI 框架**：觸控、動畫、字型渲染
- **BLE Stack**：GAP, GATT, Profile
- **手機 App**：iOS 和 Android 雙平台

我深吸一口氣，看著 Lisa 說："可行，但我們需要一個強大的團隊，而且必須做好充分的技術規劃。"

Lisa 笑了："我就知道你會接受這個挑戰。團隊已經組好了：你負責韌體，Michael 負責硬體，Sarah 負責 RF，Alex 負責測試。我們週三開始第一次技術會議。"

接下來的 9 個月，是我職業生涯中最緊張、最充實、也最有成就感的時光。我們從零開始，經歷了無數次的設計迭代、除錯、測試，最終成功推出了這款智慧手錶。

這篇文章，我想分享這個完整的開發過程：從需求分析到產品上市，從技術挑戰到解決方案，從跨團隊協作到經驗教訓。

---

## 智慧手錶的系統架構

在第一次技術會議上，我在白板上畫出了智慧手錶的系統架構：

```
┌─────────────────────────────────────────────────────────────────┐
│  智慧手錶系統架構                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  MCU (ARM Cortex-M4 @ 64 MHz)                            │   │
│  │  - 256 KB RAM                                            │   │
│  │  - 1 MB Flash                                            │   │
│  │  - FreeRTOS                                              │   │
│  └────┬─────────┬─────────┬─────────┬─────────┬────────────┘   │
│       │         │         │         │         │                 │
│  ┌────┴────┐ ┌─┴──────┐ ┌┴────────┐ ┌┴───────┐ ┌┴──────────┐  │
│  │  BLE    │ │ MIPI   │ │ Sensors │ │  SPI   │ │  Battery  │  │
│  │  Stack  │ │ Display│ │         │ │  Flash │ │  Mgmt     │  │
│  └────┬────┘ └─┬──────┘ └┬────────┘ └┬───────┘ └┬──────────┘  │
│       │         │         │           │           │             │
│  ┌────┴────┐ ┌─┴──────┐ ┌┴────────┐ ┌┴───────┐ ┌┴──────────┐  │
│  │  nRF52  │ │ 1.3"   │ │ - HR    │ │ 16 MB  │ │ 200 mAh   │  │
│  │  832    │ │ AMOLED │ │ - Accel │ │ Flash  │ │ Li-Po     │  │
│  │         │ │ 360x360│ │ - Gyro  │ │        │ │           │  │
│  │         │ │        │ │ - Baro  │ │        │ │           │  │
│  └─────────┘ └────────┘ └─────────┘ └────────┘ └───────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

**核心元件選擇**：

1. **MCU**：Nordic nRF52832
   - ARM Cortex-M4 @ 64 MHz
   - 256 KB RAM, 1 MB Flash
   - 內建 BLE 5.0
   - 低功耗：< 5 μA Sleep, < 10 mA Active

2. **Display**：1.3 吋圓形 AMOLED
   - 解析度：360 x 360
   - 介面：MIPI DSI (1-lane)
   - 功耗：< 20 mA @ 50% 亮度

3. **Sensors**：
   - 心率感測器：Maxim MAX30102 (PPG)
   - 加速度計 + 陀螺儀：InvenSense MPU-6050 (I2C)
   - 氣壓計：Bosch BMP280 (I2C)

4. **Storage**：
   - SPI Flash：16 MB (Winbond W25Q128)
   - 儲存字型、圖片、使用者資料

5. **Battery**：
   - 200 mAh Li-Po
   - 充電：USB 磁吸充電座
   - 目標續航：> 3 天

---

## 從需求到產品：完整的開發流程

### Phase 1: 需求分析與規劃（Week 1-2）

**產品需求**：

- **核心功能**：
  - 通知同步（電話、訊息、App 通知）
  - 健康監測（心率、步數、睡眠）
  - 運動追蹤（跑步、騎車、游泳）
  - 時間顯示（錶盤、鬧鐘、計時器）

- **性能要求**：
  - 電池續航：> 3 天（正常使用）
  - 充電時間：< 2 小時
  - BLE 連線距離：> 10 公尺
  - 觸控反應時間：< 100 ms
  - UI 流暢度：> 30 FPS

- **品質要求**：
  - 防水等級：IP67
  - 工作溫度：-10°C ~ 50°C
  - 通過 Bluetooth SIG 認證
  - 通過 FCC/CE 認證

**技術規劃**：

我們將開發工作分為 5 個階段：

1. **EVT (Engineering Validation Test)**：驗證核心功能
2. **DVT (Design Validation Test)**：驗證設計可靠性
3. **PVT (Production Validation Test)**：驗證量產可行性
4. **MP (Mass Production)**：量產
5. **Post-MP**：售後支援和韌體更新

---

### Phase 2: EVT - 核心功能驗證（Week 3-12）

**目標**：驗證核心技術可行性

**硬體設計**：

Michael 設計了第一版 EVT 板：

- 使用 Nordic nRF52832 DK 作為基礎
- 外接 MIPI Display 模組
- 外接 Sensor 模組（I2C）
- 外接 SPI Flash
- 使用 USB 供電（方便除錯）

**韌體開發**：

我開始開發韌體架構：

```c
// FreeRTOS 任務架構
void main(void) {
    // 1. 硬體初始化
    hardware_init();

    // 2. 建立 FreeRTOS 任務
    xTaskCreate(ble_task, "BLE", 512, NULL, 2, NULL);
    xTaskCreate(display_task, "Display", 1024, NULL, 1, NULL);
    xTaskCreate(sensor_task, "Sensor", 512, NULL, 1, NULL);
    xTaskCreate(ui_task, "UI", 2048, NULL, 3, NULL);

    // 3. 啟動 FreeRTOS
    vTaskStartScheduler();
}

// BLE 任務：處理 BLE 連線和資料傳輸
void ble_task(void *pvParameters) {
    while (1) {
        // 處理 BLE 事件
        ble_stack_process();

        // 等待事件
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
    }
}

// Display 任務：更新顯示器
void display_task(void *pvParameters) {
    while (1) {
        // 更新顯示器
        display_update();

        // 30 FPS = 33 ms
        vTaskDelay(pdMS_TO_TICKS(33));
    }
}

// Sensor 任務：讀取感測器資料
void sensor_task(void *pvParameters) {
    while (1) {
        // 讀取心率
        uint8_t hr = heart_rate_read();

        // 讀取加速度
        int16_t accel[3];
        accelerometer_read(accel);

        // 計算步數
        step_count_update(accel);

        // 1 Hz
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

// UI 任務：處理使用者輸入和 UI 邏輯
void ui_task(void *pvParameters) {
    while (1) {
        // 處理觸控事件
        touch_event_t event;
        if (touch_get_event(&event)) {
            ui_handle_event(&event);
        }

        // 10 ms
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

**EVT 階段的挑戰**：

#### 挑戰 1：MIPI Display 初始化失敗

**問題**：Display 無法正常顯示，只有白屏

**除錯過程**：

1. 檢查 MIPI DSI 時序：使用 Logic Analyzer 抓取 D0+/D0- 訊號
2. 發現問題：Clock Lane 沒有進入 HS Mode
3. 根因：MIPI DSI 初始化序列錯誤

**解決方案**：

```c
// 正確的 MIPI DSI 初始化序列
void mipi_dsi_init(void) {
    // 1. 進入 LP Mode
    mipi_dsi_set_mode(MIPI_DSI_MODE_LP);

    // 2. 發送初始化命令
    mipi_dsi_dcs_write(0x11, NULL, 0);  // Sleep Out
    vTaskDelay(pdMS_TO_TICKS(120));

    mipi_dsi_dcs_write(0x29, NULL, 0);  // Display On
    vTaskDelay(pdMS_TO_TICKS(20));

    // 3. 進入 HS Mode
    mipi_dsi_set_mode(MIPI_DSI_MODE_HS);

    printf("MIPI DSI initialized\n");
}
```

---

#### 挑戰 2：心率感測器讀數不穩定

**問題**：心率讀數在 60-180 bpm 之間跳動

**除錯過程**：

1. 檢查 PPG 訊號：使用示波器觀察 LED 和 PD 訊號
2. 發現問題：訊號雜訊很大，SNR < 10 dB
3. 根因：LED 電流太小，PD 增益太低

**解決方案**：

```c
// 優化 PPG 參數
void heart_rate_init(void) {
    // 1. 設定 LED 電流（增加到 50 mA）
    max30102_set_led_current(MAX30102_LED_CURRENT_50MA);

    // 2. 設定 PD 增益（增加到最大）
    max30102_set_adc_range(MAX30102_ADC_RANGE_16384);

    // 3. 設定取樣率（100 Hz）
    max30102_set_sample_rate(MAX30102_SAMPLE_RATE_100HZ);

    // 4. 啟用數位濾波器
    max30102_enable_filter(true);

    printf("Heart rate sensor initialized\n");
}

// 心率演算法：使用 FFT 找出心率
uint8_t heart_rate_calculate(int32_t *ppg_data, uint16_t length) {
    // 1. 移除 DC 分量
    int32_t mean = 0;
    for (int i = 0; i < length; i++) {
        mean += ppg_data[i];
    }
    mean /= length;

    for (int i = 0; i < length; i++) {
        ppg_data[i] -= mean;
    }

    // 2. FFT
    arm_rfft_fast_instance_f32 fft;
    arm_rfft_fast_init_f32(&fft, length);

    float32_t fft_input[length];
    float32_t fft_output[length];

    for (int i = 0; i < length; i++) {
        fft_input[i] = (float32_t)ppg_data[i];
    }

    arm_rfft_fast_f32(&fft, fft_input, fft_output, 0);

    // 3. 找出最大頻率（0.8-3 Hz = 48-180 bpm）
    uint16_t max_index = 0;
    float32_t max_value = 0;

    for (int i = 8; i < 30; i++) {  // 0.8-3 Hz @ 100 Hz sample rate
        float32_t magnitude = sqrt(fft_output[i*2] * fft_output[i*2] +
                                   fft_output[i*2+1] * fft_output[i*2+1]);
        if (magnitude > max_value) {
            max_value = magnitude;
            max_index = i;
        }
    }

    // 4. 計算心率
    float32_t freq = (float32_t)max_index * 100.0f / length;  // Hz
    uint8_t hr = (uint8_t)(freq * 60.0f);  // bpm

    return hr;
}
```

---

#### 挑戰 3：記憶體不足

**問題**：韌體編譯後，RAM 使用率 > 90%

**除錯過程**：

1. 使用 `arm-none-eabi-nm` 分析記憶體使用
2. 發現問題：Display Frame Buffer 佔用 256 KB（360x360x2 bytes）
3. 根因：AMOLED 使用 RGB565 格式，每個像素 2 bytes

**解決方案**：

```c
// 使用 Partial Update 減少 Frame Buffer 大小
#define DISPLAY_WIDTH  360
#define DISPLAY_HEIGHT 360
#define TILE_SIZE      60  // 60x60 tile

// 只分配一個 Tile 的 Frame Buffer
uint16_t frame_buffer[TILE_SIZE * TILE_SIZE];  // 7.2 KB

void display_update(void) {
    // 逐 Tile 更新顯示器
    for (int y = 0; y < DISPLAY_HEIGHT; y += TILE_SIZE) {
        for (int x = 0; x < DISPLAY_WIDTH; x += TILE_SIZE) {
            // 1. 渲染 Tile
            ui_render_tile(frame_buffer, x, y, TILE_SIZE, TILE_SIZE);

            // 2. 傳送到 Display
            mipi_dsi_write_tile(frame_buffer, x, y, TILE_SIZE, TILE_SIZE);
        }
    }
}
```

**結果**：

- RAM 使用率：從 90% 降到 60%
- 顯示更新時間：從 16 ms 增加到 50 ms（仍然 > 20 FPS）

---

### Phase 3: DVT - 設計驗證（Week 13-24）

**目標**：驗證設計可靠性和功耗

**硬體改進**：

Michael 設計了第二版 DVT 板：

- 縮小 PCB 尺寸（從 50x50 mm 到 40x40 mm）
- 整合電池管理（充電、保護、電量計）
- 優化天線設計（PCB 天線 + Matching Network）
- 加入防水設計（密封圈、防水膠）

**韌體優化**：

我開始優化功耗：

```c
// 功耗優化策略
void power_management_init(void) {
    // 1. 啟用 DC/DC 轉換器（降低功耗 30%）
    NRF_POWER->DCDCEN = 1;

    // 2. 設定 Sleep 模式
    sd_power_mode_set(NRF_POWER_MODE_LOWPWR);

    // 3. 關閉不使用的週邊
    NRF_UART0->ENABLE = 0;  // 關閉 UART（除錯時才開啟）

    printf("Power management initialized\n");
}

// 動態調整 BLE 連線參數
void ble_conn_params_update(bool low_latency) {
    ble_gap_conn_params_t conn_params;

    if (low_latency) {
        // 低延遲模式（通知同步時）
        conn_params.min_conn_interval = MSEC_TO_UNITS(7.5, UNIT_1_25_MS);
        conn_params.max_conn_interval = MSEC_TO_UNITS(7.5, UNIT_1_25_MS);
        conn_params.slave_latency = 0;
    } else {
        // 省電模式（待機時）
        conn_params.min_conn_interval = MSEC_TO_UNITS(100, UNIT_1_25_MS);
        conn_params.max_conn_interval = MSEC_TO_UNITS(200, UNIT_1_25_MS);
        conn_params.slave_latency = 4;
    }

    conn_params.conn_sup_timeout = MSEC_TO_UNITS(4000, UNIT_10_MS);

    sd_ble_gap_conn_param_update(m_conn_handle, &conn_params);
}

// 動態調整 Display 亮度
void display_brightness_auto_adjust(void) {
    // 根據環境光線調整亮度
    uint16_t ambient_light = ambient_light_sensor_read();

    uint8_t brightness;
    if (ambient_light < 100) {
        brightness = 20;  // 暗環境：20%
    } else if (ambient_light < 500) {
        brightness = 50;  // 室內：50%
    } else {
        brightness = 100; // 戶外：100%
    }

    display_set_brightness(brightness);
}

// 動態調整感測器取樣率
void sensor_sampling_rate_adjust(bool active) {
    if (active) {
        // 運動模式：高取樣率
        heart_rate_set_sample_rate(100);  // 100 Hz
        accelerometer_set_sample_rate(100);  // 100 Hz
    } else {
        // 待機模式：低取樣率
        heart_rate_set_sample_rate(1);  // 1 Hz
        accelerometer_set_sample_rate(10);  // 10 Hz
    }
}
```

**功耗測試結果**：

| 模式 | 電流 | 續航時間 |
|------|------|----------|
| **Sleep**（螢幕關閉，BLE 斷線） | 50 μA | > 30 天 |
| **Idle**（螢幕開啟，BLE 連線） | 5 mA | > 7 天 |
| **Active**（通知同步） | 15 mA | > 3 天 |
| **Sport**（運動追蹤） | 25 mA | > 1.5 天 |

**結論**：✅ 符合 > 3 天續航的要求

---

### Phase 4: PVT - 量產驗證（Week 25-32）

**目標**：驗證量產可行性

**量產挑戰**：

#### 挑戰 1：良率問題

**問題**：第一批 PVT 樣品良率只有 70%

**根因分析**：

1. **MIPI Display 不良**：20%
   - 原因：FPC 連接器接觸不良
   - 解決：更換更可靠的連接器

2. **BLE 連線失敗**：10%
   - 原因：天線 Matching 不佳
   - 解決：調整 Matching Network（重新調整 L/C 值）

**改進後良率**：> 95%

---

#### 挑戰 2：認證測試

**Bluetooth SIG 認證**：

- RF 測試：✅ 通過
- Protocol 測試：✅ 通過
- Profile 測試：❌ 失敗（HRS Profile）

**問題**：Heart Rate Service 的 Body Sensor Location 特性沒有實作

**解決方案**：

```c
// 加入 Body Sensor Location 特性
#define BLE_HRS_BODY_SENSOR_LOCATION_WRIST  1

void hrs_init(void) {
    ble_hrs_init_t hrs_init = {0};

    // 設定 Body Sensor Location
    hrs_init.body_sensor_location = BLE_HRS_BODY_SENSOR_LOCATION_WRIST;

    // 初始化 HRS
    ble_hrs_init(&m_hrs, &hrs_init);
}
```

**重新測試**：✅ 通過

---

**FCC/CE 認證**：

- RF Emission：✅ 通過
- RF Immunity：✅ 通過
- ESD：❌ 失敗（± 8 kV Contact Discharge）

**問題**：ESD 測試時，MCU 會重啟

**解決方案**：

1. 加入 TVS Diode 保護 USB 接口
2. 加入 ESD 保護電路在按鍵和觸控面板
3. 改善 PCB Layout（加大 GND Plane）

**重新測試**：✅ 通過

---

### Phase 5: MP - 量產（Week 33-36）

**目標**：開始量產並上市

**量產準備**：

1. **韌體凍結**：
   - 版本：v1.0.0
   - 功能：完整實作所有核心功能
   - 測試：通過所有測試案例

2. **生產工具**：
   - 燒錄工具：使用 J-Link 批量燒錄韌體
   - 測試工具：自動化測試（BLE, Display, Sensor, Battery）
   - 校準工具：心率感測器校準

3. **品質控制**：
   - 來料檢驗（IQC）
   - 製程檢驗（IPQC）
   - 出貨檢驗（OQC）

**量產流程**：

```
┌─────────────────────────────────────────────────────────┐
│  智慧手錶量產流程                                         │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  1. SMT 貼片 → 2. 回焊 → 3. AOI 檢測                     │
│       ↓                                                   │
│  4. 組裝（Display, Battery, 外殼）                        │
│       ↓                                                   │
│  5. 燒錄韌體                                              │
│       ↓                                                   │
│  6. 功能測試（BLE, Display, Sensor, Battery）             │
│       ↓                                                   │
│  7. 校準（心率感測器）                                    │
│       ↓                                                   │
│  8. 老化測試（24 小時）                                   │
│       ↓                                                   │
│  9. 包裝出貨                                              │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**第一批量產**：

- 數量：10,000 隻
- 良率：96%
- 週期：4 週

---

### Phase 6: Post-MP - 售後支援（Week 37+）

**OTA 更新**：

產品上市後，我們收到了很多使用者反饋。我們需要透過 OTA 更新來修復問題和新增功能。

**OTA 架構**：

```c
// OTA 更新流程
typedef enum {
    OTA_STATE_IDLE,
    OTA_STATE_DOWNLOADING,
    OTA_STATE_VERIFYING,
    OTA_STATE_INSTALLING,
    OTA_STATE_COMPLETE,
    OTA_STATE_ERROR
} ota_state_t;

ota_state_t ota_state = OTA_STATE_IDLE;

void ota_update_start(uint32_t firmware_size, uint32_t firmware_crc) {
    // 1. 檢查空間
    if (firmware_size > FLASH_SIZE - BOOTLOADER_SIZE - APP_SIZE) {
        printf("OTA: Not enough space\n");
        ota_state = OTA_STATE_ERROR;
        return;
    }

    // 2. 擦除 OTA 區域
    flash_erase(OTA_FLASH_START, firmware_size);

    // 3. 開始下載
    ota_state = OTA_STATE_DOWNLOADING;
    ota_offset = 0;
    ota_total_size = firmware_size;
    ota_expected_crc = firmware_crc;

    printf("OTA: Started (size=%d, crc=0x%08X)\n", firmware_size, firmware_crc);
}

void ota_update_write(uint8_t *data, uint16_t length) {
    if (ota_state != OTA_STATE_DOWNLOADING) {
        return;
    }

    // 寫入 Flash
    flash_write(OTA_FLASH_START + ota_offset, data, length);
    ota_offset += length;

    // 更新進度
    uint8_t progress = (ota_offset * 100) / ota_total_size;
    printf("OTA: Progress %d%%\n", progress);

    // 檢查是否完成
    if (ota_offset >= ota_total_size) {
        ota_state = OTA_STATE_VERIFYING;
        ota_update_verify();
    }
}

void ota_update_verify(void) {
    // 計算 CRC
    uint32_t crc = crc32_calculate(OTA_FLASH_START, ota_total_size);

    if (crc == ota_expected_crc) {
        printf("OTA: Verification passed\n");
        ota_state = OTA_STATE_INSTALLING;
        ota_update_install();
    } else {
        printf("OTA: Verification failed (expected=0x%08X, actual=0x%08X)\n",
               ota_expected_crc, crc);
        ota_state = OTA_STATE_ERROR;
    }
}

void ota_update_install(void) {
    // 1. 設定 Bootloader Flag
    bootloader_set_flag(BOOTLOADER_FLAG_OTA_UPDATE);

    // 2. 重啟
    printf("OTA: Installing... Rebooting...\n");
    vTaskDelay(pdMS_TO_TICKS(1000));
    NVIC_SystemReset();
}
```

**OTA 更新記錄**：

| 版本 | 日期 | 更新內容 |
|------|------|----------|
| v1.0.0 | 2015-03 | 初始版本 |
| v1.0.1 | 2015-04 | 修復心率讀數不穩定問題 |
| v1.0.2 | 2015-05 | 新增睡眠監測功能 |
| v1.1.0 | 2015-06 | 新增游泳模式（防水優化） |
| v1.2.0 | 2015-08 | 新增 GPS 軌跡記錄（配合手機 GPS） |

---

## 跨團隊協作

這個專案的成功，離不開跨團隊的緊密協作：

### 韌體 ↔ 硬體

**溝通重點**：

- **GPIO 分配**：哪些 GPIO 用於哪些功能
- **電源管理**：哪些元件可以關閉以省電
- **時序要求**：MIPI DSI, I2C, SPI 的時序參數
- **除錯接口**：保留 SWD 和 UART 接口

**協作工具**：

- 硬體原理圖（Altium Designer）
- GPIO 分配表（Excel）
- 週會（每週一早上 10:00）

---

### 韌體 ↔ RF

**溝通重點**：

- **天線設計**：PCB 天線 vs 陶瓷天線
- **RF 測試**：TX Power, Sensitivity, Frequency Offset
- **Coexistence**：BLE 與 WiFi 的共存策略
- **認證測試**：Bluetooth SIG, FCC, CE

**協作工具**：

- RF 測試報告（PDF）
- Spectrum Analyzer 截圖
- 認證測試報告

---

### 韌體 ↔ 測試

**溝通重點**：

- **測試案例**：功能測試、壓力測試、相容性測試
- **Bug 回報**：使用 JIRA 追蹤 Bug
- **自動化測試**：開發自動化測試工具

**協作工具**：

- JIRA（Bug 追蹤）
- 測試報告（Excel）
- 自動化測試腳本（Python）

---

## 經驗教訓

回顧這 9 個月的開發過程，我學到了很多寶貴的經驗：

### 1. 提前規劃，預留緩衝

**教訓**：

- 我們原本計劃 9 個月完成，實際花了 10 個月
- 原因：認證測試失敗，需要重新設計和測試

**建議**：

- 預留 20-30% 的緩衝時間
- 提前進行風險評估
- 關鍵路徑要有備案

---

### 2. 早期驗證，降低風險

**教訓**：

- EVT 階段發現 MIPI Display 初始化問題，花了 2 週除錯
- 如果更早驗證，可以節省時間

**建議**：

- 使用 Development Kit 提前驗證核心技術
- 不要等到 EVT 才開始驗證
- 高風險的技術要優先驗證

---

### 3. 功耗優化要從一開始

**教訓**：

- DVT 階段才開始優化功耗，發現很多設計需要修改
- 例如：Display 的 Frame Buffer 太大，需要重新設計

**建議**：

- 從 EVT 階段就開始考慮功耗
- 使用 Power Profiler 持續監控功耗
- 設定功耗預算，每個模組都要符合預算

---

### 4. 自動化測試很重要

**教訓**：

- 手動測試很耗時，而且容易遺漏
- 量產時發現一些 Bug 是因為測試不完整

**建議**：

- 開發自動化測試工具
- 每次 Commit 都要跑自動化測試
- 量產前要進行完整的回歸測試

---

### 5. 跨團隊溝通要頻繁

**教訓**：

- 有些問題是因為溝通不足導致的
- 例如：硬體改了 GPIO 分配，但沒有通知韌體

**建議**：

- 每週固定開會
- 使用共享文件（Google Docs, Confluence）
- 重要變更要發郵件通知所有人

---

## 實務建議

基於這次智慧手錶開發的經驗，我總結了以下的實務建議：

### 開發流程檢查清單

**EVT 階段**：

- [ ] 核心功能驗證（BLE, Display, Sensor）
- [ ] 硬體原理圖 Review
- [ ] 韌體架構設計
- [ ] 功耗初步評估
- [ ] 風險評估和備案

**DVT 階段**：

- [ ] 設計可靠性驗證
- [ ] 功耗優化
- [ ] 天線設計和 RF 測試
- [ ] 認證測試準備
- [ ] 量產工具開發

**PVT 階段**：

- [ ] 量產可行性驗證
- [ ] 良率分析和改善
- [ ] 認證測試（Bluetooth SIG, FCC, CE）
- [ ] 韌體凍結
- [ ] 生產線測試

**MP 階段**：

- [ ] 量產啟動
- [ ] 品質控制（IQC, IPQC, OQC）
- [ ] 良率監控
- [ ] 售後支援準備

---

### 技術選型建議

**MCU 選擇**：

- **低功耗優先**：選擇支援 Sleep Mode 的 MCU
- **整合度優先**：選擇內建 BLE 的 MCU（例如：Nordic nRF52）
- **生態系統**：選擇有完整 SDK 和社群支援的 MCU

**Display 選擇**：

- **功耗**：AMOLED < LCD（黑色畫面時）
- **介面**：MIPI DSI > SPI（高解析度時）
- **成本**：SPI < MIPI DSI

**Sensor 選擇**：

- **整合度**：選擇整合多種感測器的模組（例如：MPU-6050 = Accel + Gyro）
- **功耗**：選擇支援 Low Power Mode 的感測器
- **精度**：根據應用需求選擇合適的精度

---

## 總結

這次智慧手錶開發專案，是我職業生涯中最具挑戰性的專案之一。我們從零開始，在 9 個月內完成了：

- **硬體設計**：從 EVT 到 PVT，3 次迭代
- **韌體開發**：20,000+ 行程式碼
- **認證測試**：Bluetooth SIG, FCC, CE
- **量產**：10,000 隻，良率 96%

這個專案讓我深刻體會到：

1. **系統思維很重要**：IoT 產品不只是韌體，還包括硬體、RF、測試、量產
2. **功耗優化要從一開始**：不要等到 DVT 才開始優化
3. **自動化測試很重要**：可以節省大量時間和提高品質
4. **跨團隊協作很重要**：溝通頻繁可以避免很多問題
5. **預留緩衝時間**：計劃永遠趕不上變化

如果你也在開發 IoT 產品，希望這篇文章能給你一些啟發和幫助。

---

## 參考資料

- Nordic nRF52832: <https://www.nordicsemi.com/Products/nRF52832>
- FreeRTOS: <https://www.freertos.org/>
- MIPI DSI Specification: <https://www.mipi.org/specifications/dsi>
- Bluetooth SIG Qualification: <https://www.bluetooth.com/develop-with-bluetooth/qualification-listing/>
- FCC Certification: <https://www.fcc.gov/engineering-technology/laboratory-division/general/equipment-authorization>
- ARM CMSIS-DSP: <https://github.com/ARM-software/CMSIS-DSP>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

您可以自由地：

- **分享** — 以任何媒介或格式重製及散布本素材
- **修改** — 重混、轉換本素材、及依本素材建立新素材

但您必須遵守以下條款：

- **姓名標示** — 您必須給予適當表彰、提供指向本授權條款的連結，以及指出（本作品的原始版本）是否已被變更。您可以任何合理的方式為前述表彰，但不得以任何方式暗示授權人為您或您的使用方式背書。
