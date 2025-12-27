# MIPI DSI - 智慧手錶的 Display 接口

**作者**: Danny Jiang  
**日期**: 2025-12-11  

---

## 前言：從 SPI Display 到 MIPI DSI 的遷移

2015 年 3 月，我們的智慧手錶專案進入第二代開發。

第一代產品使用的是 **SPI Display**（240x240 解析度，16-bit color），雖然成本低，但有幾個明顯的問題：

1. **更新速度慢**：60 FPS 需要的頻寬是 `240 * 240 * 2 bytes * 60 = 6.9 MB/s`，但 SPI 的實際速度只有 3 MB/s，導致畫面更新只能達到 25 FPS，滑動時有明顯卡頓。

2. **功耗高**：SPI 需要持續傳輸資料，即使畫面沒有變化，也要不斷刷新，導致功耗居高不下。

3. **Pin 數量多**：SPI 需要 4 條訊號線（SCLK, MOSI, MISO, CS），加上 Display 的控制訊號（DC, RESET），總共需要 6 個 GPIO。

客戶對第二代產品的要求更高：

- **解析度**：從 240x240 提升到 320x320
- **更新率**：60 FPS（流暢的動畫效果）
- **功耗**：降低 30%（延長電池壽命）
- **成本**：不能增加太多

我們評估了幾種方案：

| 方案 | 頻寬需求 | 優點 | 缺點 |
|------|---------|------|------|
| **SPI (Standard)** | 12.3 MB/s | 成本低 | 速度不夠（最大 3 MB/s） |
| **SPI (Quad)** | 12.3 MB/s | 速度提升 4 倍 | 仍然不夠（最大 12 MB/s，沒有餘裕） |
| **Parallel RGB** | 12.3 MB/s | 速度快 | Pin 數量太多（24 個 GPIO） |
| **MIPI DSI** | 12.3 MB/s | 速度快、Pin 少、功耗低 | 需要 MIPI DSI 控制器 |

最終，我們選擇了 **MIPI DSI**。

MIPI DSI (Display Serial Interface) 是 MIPI Alliance 定義的一種高速串列接口，專門用於連接 Display。它的特點是：

- **高速**：支援 1 Gbps 以上的傳輸速度
- **低 Pin 數**：只需要 2 條資料線（Data Lane 0, Data Lane 1）
- **低功耗**：支援 Low-Power Mode，畫面靜止時功耗極低
- **標準化**：MIPI Alliance 定義的標準，相容性好

但問題是：**我們的 SoC 沒有 MIPI DSI 控制器**。

「那就用軟體模擬吧！」主管說。

於是，我花了 4 週時間，從零開始實作 MIPI DSI 驅動。這是我第一次接觸 MIPI 協定，也是我第一次理解「為什麼智慧手機的螢幕可以這麼流暢」。

最終，我們成功將畫面更新率提升到 60 FPS，功耗降低了 40%，Pin 數量從 6 個減少到 4 個。

這段經歷讓我深刻體會到：**選擇正確的接口，是產品成功的關鍵**。

---

## MIPI DSI 的核心概念

### 什麼是 MIPI DSI？

**MIPI DSI (Display Serial Interface)** 是 MIPI Alliance 在 2006 年發布的一種高速串列接口標準，專門用於連接 Display（LCD、OLED）。

**MIPI DSI 的特性**：

- **高速串列傳輸**：使用差分訊號（Differential Signaling），速度可達 1-2.5 Gbps per lane
- **多 Lane 架構**：支援 1-4 條 Data Lane，頻寬可擴展
- **雙模式**：High-Speed Mode（高速傳輸）+ Low-Power Mode（低功耗控制）
- **封包化傳輸**：使用封包格式傳輸資料，支援多種資料類型
- **標準化**：MIPI Alliance 定義的標準，相容性好

### MIPI DSI vs SPI Display

| 比較項目 | SPI Display | MIPI DSI |
|---------|-------------|----------|
| **傳輸方式** | 單端訊號（Single-ended） | 差分訊號（Differential） |
| **速度** | 3-12 MB/s（Standard/Quad SPI） | 100-300 MB/s（1-4 Lane） |
| **Pin 數量** | 4-6 個（SCLK, MOSI, MISO, CS, DC, RESET） | 4-10 個（Clock Lane + 1-4 Data Lane） |
| **功耗** | 高（持續傳輸） | 低（支援 Low-Power Mode） |
| **成本** | 低 | 中（需要 MIPI DSI 控制器） |
| **應用** | 小尺寸 Display（< 2 吋） | 中大尺寸 Display（> 2 吋） |

**為什麼 MIPI DSI 速度更快？**

1. **差分訊號**：抗干擾能力強，可以使用更高的時脈頻率
2. **多 Lane 架構**：可以並行傳輸，頻寬可擴展
3. **專用硬體**：MIPI DSI 控制器使用硬體加速，效率高

---

## MIPI DSI 的架構

### 三層架構

MIPI DSI 採用三層架構：

```
┌─────────────────────────────────────────────────────────┐
│  MIPI DSI 三層架構                                        │
├─────────────────────────────────────────────────────────┤
│  Layer 3: Application Layer                             │
│  ├─ DCS (Display Command Set)                           │
│  ├─ Generic Interface                                   │
│  └─ 定義 Display 的控制命令（如：設定亮度、進入睡眠）      │
├─────────────────────────────────────────────────────────┤
│  Layer 2: Protocol Layer                                │
│  ├─ Packet Format                                       │
│  ├─ Error Detection (ECC, CRC)                          │
│  └─ 定義封包格式、錯誤檢測                                │
├─────────────────────────────────────────────────────────┤
│  Layer 1: Physical Layer (D-PHY)                        │
│  ├─ High-Speed Mode (HS)                                │
│  ├─ Low-Power Mode (LP)                                 │
│  └─ 定義電氣特性、差分訊號                                │
└─────────────────────────────────────────────────────────┘
```

### Layer 1: D-PHY（Physical Layer）

**D-PHY** 是 MIPI DSI 的物理層，定義了電氣特性和訊號傳輸方式。

**D-PHY 的兩種模式**：

1. **High-Speed Mode (HS)**：
   - 使用差分訊號（Differential Signaling）
   - 速度：80 Mbps - 2.5 Gbps per lane
   - 用於傳輸大量資料（如：畫面資料）

2. **Low-Power Mode (LP)**：
   - 使用單端訊號（Single-ended Signaling）
   - 速度：< 10 Mbps
   - 用於傳輸控制命令（如：設定亮度、進入睡眠）
   - 功耗極低

**D-PHY 的 Lane 架構**：

```
┌─────────────────────────────────────────────────────────┐
│  D-PHY Lane 架構                                         │
├─────────────────────────────────────────────────────────┤
│  Clock Lane (1 條)                                       │
│  ├─ CLK+  (差分訊號正極)                                 │
│  └─ CLK-  (差分訊號負極)                                 │
├─────────────────────────────────────────────────────────┤
│  Data Lane 0 (必須)                                      │
│  ├─ D0+   (差分訊號正極)                                 │
│  └─ D0-   (差分訊號負極)                                 │
├─────────────────────────────────────────────────────────┤
│  Data Lane 1 (可選)                                      │
│  ├─ D1+   (差分訊號正極)                                 │
│  └─ D1-   (差分訊號負極)                                 │
├─────────────────────────────────────────────────────────┤
│  Data Lane 2 (可選)                                      │
│  ├─ D2+   (差分訊號正極)                                 │
│  └─ D2-   (差分訊號負極)                                 │
├─────────────────────────────────────────────────────────┤
│  Data Lane 3 (可選)                                      │
│  ├─ D3+   (差分訊號正極)                                 │
│  └─ D3-   (差分訊號負極)                                 │
└─────────────────────────────────────────────────────────┘
```

**Pin 數量計算**：

```
1 Lane:  Clock Lane (2 pins) + Data Lane 0 (2 pins) = 4 pins
2 Lane:  Clock Lane (2 pins) + Data Lane 0-1 (4 pins) = 6 pins
3 Lane:  Clock Lane (2 pins) + Data Lane 0-2 (6 pins) = 8 pins
4 Lane:  Clock Lane (2 pins) + Data Lane 0-3 (8 pins) = 10 pins
```

### Layer 2: Protocol Layer

**Protocol Layer** 定義了封包格式和錯誤檢測機制。

**MIPI DSI 的封包格式**：

```
┌─────────────────────────────────────────────────────────┐
│  Short Packet (4 bytes)                                  │
├─────────────────────────────────────────────────────────┤
│  Byte 0: Data ID (DI)                                    │
│  ├─ Virtual Channel (2 bits)                            │
│  └─ Data Type (6 bits)                                  │
├─────────────────────────────────────────────────────────┤
│  Byte 1-2: Data (16 bits)                                │
├─────────────────────────────────────────────────────────┤
│  Byte 3: ECC (Error Correction Code)                    │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  Long Packet (6 + N bytes)                               │
├─────────────────────────────────────────────────────────┤
│  Byte 0: Data ID (DI)                                    │
│  ├─ Virtual Channel (2 bits)                            │
│  └─ Data Type (6 bits)                                  │
├─────────────────────────────────────────────────────────┤
│  Byte 1-2: Word Count (WC) - 資料長度                    │
├─────────────────────────────────────────────────────────┤
│  Byte 3: ECC (Error Correction Code)                    │
├─────────────────────────────────────────────────────────┤
│  Byte 4 ~ (4+N-1): Payload (資料)                        │
├─────────────────────────────────────────────────────────┤
│  Byte (4+N) ~ (4+N+1): CRC (Checksum)                    │
└─────────────────────────────────────────────────────────┘
```

**常見的 Data Type**：

| Data Type | 代碼 | 說明 |
|-----------|------|------|
| **DCS Short Write (0 parameter)** | 0x05 | 發送 DCS 命令（無參數） |
| **DCS Short Write (1 parameter)** | 0x15 | 發送 DCS 命令（1 個參數） |
| **DCS Long Write** | 0x39 | 發送 DCS 命令（多個參數） |
| **Generic Short Write (0 parameter)** | 0x03 | 發送通用命令（無參數） |
| **Generic Short Write (1 parameter)** | 0x13 | 發送通用命令（1 個參數） |
| **Generic Long Write** | 0x29 | 發送通用命令（多個參數） |
| **DCS Read** | 0x06 | 讀取 DCS 暫存器 |
| **Packed Pixel Stream (16-bit RGB)** | 0x0E | 傳輸 16-bit RGB 畫面資料 |
| **Packed Pixel Stream (18-bit RGB)** | 0x1E | 傳輸 18-bit RGB 畫面資料 |
| **Packed Pixel Stream (24-bit RGB)** | 0x3E | 傳輸 24-bit RGB 畫面資料 |

### Layer 3: Application Layer

**Application Layer** 定義了 Display 的控制命令，主要有兩種：

1. **DCS (Display Command Set)**：
   - MIPI Alliance 定義的標準命令集
   - 所有 MIPI DSI Display 都支援
   - 範例：設定亮度、進入睡眠、設定顯示方向

2. **Generic Interface**：
   - 廠商自定義的命令
   - 用於特殊功能（如：調整 Gamma、設定 VCOM）

---

## MIPI DCS：Display Command Set

### 常用的 DCS 命令

| 命令 | 代碼 | 參數 | 說明 |
|------|------|------|------|
| **NOP** | 0x00 | 無 | 無操作 |
| **Software Reset** | 0x01 | 無 | 軟體重置 Display |
| **Get Display ID** | 0x04 | 無 | 讀取 Display ID |
| **Get Display Status** | 0x09 | 無 | 讀取 Display 狀態 |
| **Enter Sleep Mode** | 0x10 | 無 | 進入睡眠模式（功耗 < 1 mA） |
| **Exit Sleep Mode** | 0x11 | 無 | 退出睡眠模式 |
| **Set Display On** | 0x29 | 無 | 開啟 Display |
| **Set Display Off** | 0x28 | 無 | 關閉 Display |
| **Set Column Address** | 0x2A | 4 bytes | 設定 X 座標範圍（SC, EC） |
| **Set Page Address** | 0x2B | 4 bytes | 設定 Y 座標範圍（SP, EP） |
| **Write Memory Start** | 0x2C | N bytes | 開始寫入畫面資料 |
| **Write Memory Continue** | 0x3C | N bytes | 繼續寫入畫面資料 |
| **Set Tear On** | 0x35 | 1 byte | 啟用 Tearing Effect（同步訊號） |
| **Set Pixel Format** | 0x3A | 1 byte | 設定像素格式（16/18/24-bit） |
| **Set Brightness** | 0x51 | 1 byte | 設定亮度（0-255） |

### DCS 命令的發送方式

**範例 1：進入睡眠模式**

```c
// DCS 命令：0x10 (Enter Sleep Mode)
// 無參數

// 使用 Short Packet
uint8_t packet[4] = {
    0x05,  // Data ID: DCS Short Write (0 parameter)
    0x10,  // DCS Command: Enter Sleep Mode
    0x00,  // 填充
    0x00   // ECC (由硬體計算)
};

mipi_dsi_send_packet(packet, 4);
```

**範例 2：設定亮度**

```c
// DCS 命令：0x51 (Set Brightness)
// 參數：0x80 (亮度 128/255)

// 使用 Short Packet
uint8_t packet[4] = {
    0x15,  // Data ID: DCS Short Write (1 parameter)
    0x51,  // DCS Command: Set Brightness
    0x80,  // Parameter: 128
    0x00   // ECC (由硬體計算)
};

mipi_dsi_send_packet(packet, 4);
```

**範例 3：設定顯示區域**

```c
// DCS 命令：0x2A (Set Column Address)
// 參數：SC=0, EC=319 (X 座標範圍 0-319)

// 使用 Long Packet
uint8_t packet[10] = {
    0x39,        // Data ID: DCS Long Write
    0x04, 0x00,  // Word Count: 4 bytes
    0x00,        // ECC (由硬體計算)
    0x2A,        // DCS Command: Set Column Address
    0x00, 0x00,  // SC (Start Column): 0
    0x01, 0x3F,  // EC (End Column): 319
    0x00, 0x00   // CRC (由硬體計算)
};

mipi_dsi_send_packet(packet, 10);
```

---

## MIPI DSI 的初始化流程

### 典型的初始化序列

```c
void mipi_dsi_display_init(void) {
    // 1. 硬體重置 Display
    gpio_set_low(RESET_PIN);
    delay_ms(10);
    gpio_set_high(RESET_PIN);
    delay_ms(120);  // 等待 Display 啟動
    
    // 2. 軟體重置 Display
    mipi_dsi_dcs_write(0x01);  // Software Reset
    delay_ms(120);
    
    // 3. 退出睡眠模式
    mipi_dsi_dcs_write(0x11);  // Exit Sleep Mode
    delay_ms(120);
    
    // 4. 設定像素格式（16-bit RGB565）
    mipi_dsi_dcs_write_param(0x3A, 0x55);  // Set Pixel Format: 16-bit
    
    // 5. 設定顯示方向（Portrait）
    mipi_dsi_dcs_write_param(0x36, 0x00);  // Set Address Mode: Portrait
    
    // 6. 設定亮度
    mipi_dsi_dcs_write_param(0x51, 0xFF);  // Set Brightness: 255 (最亮)
    
    // 7. 啟用 Tearing Effect（同步訊號）
    mipi_dsi_dcs_write_param(0x35, 0x00);  // Set Tear On: V-Blanking only
    
    // 8. 開啟 Display
    mipi_dsi_dcs_write(0x29);  // Set Display On
    delay_ms(20);
    
    // 9. 初始化完成
    printf("MIPI DSI Display initialized\n");
}
```

### 初始化流程圖

```
┌─────────────────────────────────────────────────────────┐
│  MIPI DSI Display 初始化流程                             │
├─────────────────────────────────────────────────────────┤
│  1. 硬體重置 (RESET Pin)                                 │
│     ├─ 拉低 RESET Pin (10 ms)                           │
│     ├─ 拉高 RESET Pin                                   │
│     └─ 等待 120 ms (Display 啟動)                       │
├─────────────────────────────────────────────────────────┤
│  2. 軟體重置 (DCS 0x01)                                  │
│     └─ 等待 120 ms                                      │
├─────────────────────────────────────────────────────────┤
│  3. 退出睡眠模式 (DCS 0x11)                              │
│     └─ 等待 120 ms                                      │
├─────────────────────────────────────────────────────────┤
│  4. 設定像素格式 (DCS 0x3A)                              │
│     └─ 16-bit RGB565 / 18-bit RGB666 / 24-bit RGB888    │
├─────────────────────────────────────────────────────────┤
│  5. 設定顯示方向 (DCS 0x36)                              │
│     └─ Portrait / Landscape / Flip                      │
├─────────────────────────────────────────────────────────┤
│  6. 設定亮度 (DCS 0x51)                                  │
│     └─ 0-255                                            │
├─────────────────────────────────────────────────────────┤
│  7. 啟用 Tearing Effect (DCS 0x35)                       │
│     └─ 同步訊號（避免畫面撕裂）                          │
├─────────────────────────────────────────────────────────┤
│  8. 開啟 Display (DCS 0x29)                              │
│     └─ 等待 20 ms                                       │
├─────────────────────────────────────────────────────────┤
│  9. 初始化完成                                           │
└─────────────────────────────────────────────────────────┘
```

---

## MIPI DSI 的資料傳輸模式

### Video Mode vs Command Mode

MIPI DSI 支援兩種資料傳輸模式：

| 模式 | Video Mode | Command Mode |
|------|-----------|--------------|
| **傳輸方式** | 連續傳輸（類似 RGB 接口） | 按需傳輸（只傳輸變化的區域） |
| **Frame Buffer** | Display 內部無 Frame Buffer | Display 內部有 Frame Buffer |
| **功耗** | 高（持續傳輸） | 低（只在畫面變化時傳輸） |
| **應用** | 影片播放、遊戲 | 靜態畫面、UI |
| **同步訊號** | 需要 VSYNC, HSYNC | 不需要 |

**Video Mode 的時序**：

```
┌─────────────────────────────────────────────────────────┐
│  Video Mode 時序                                         │
├─────────────────────────────────────────────────────────┤
│  VSYNC ────┐     ┌─────────────────────────────────┐    │
│            └─────┘                                 └────│
│                                                          │
│  HSYNC ────┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌────│
│            └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘    │
│                                                          │
│  Data  ────┤ Line 0 ┤ Line 1 ┤ Line 2 ┤ ... ┤ Line N ┤ │
│            └────────────────────────────────────────────│
└─────────────────────────────────────────────────────────┘
```

**Command Mode 的時序**：

```
┌─────────────────────────────────────────────────────────┐
│  Command Mode 時序                                       │
├─────────────────────────────────────────────────────────┤
│  1. 設定顯示區域 (DCS 0x2A, 0x2B)                        │
│     ├─ Set Column Address: X0=0, X1=319                 │
│     └─ Set Page Address: Y0=0, Y1=319                   │
├─────────────────────────────────────────────────────────┤
│  2. 開始寫入資料 (DCS 0x2C)                              │
│     └─ Write Memory Start                               │
├─────────────────────────────────────────────────────────┤
│  3. 傳輸畫面資料                                         │
│     └─ Pixel Data (320 * 320 * 2 bytes = 204,800 bytes) │
├─────────────────────────────────────────────────────────┤
│  4. 傳輸完成                                             │
│     └─ Display 自動更新畫面                              │
└─────────────────────────────────────────────────────────┘
```

### Partial Update：只更新變化的區域

**Command Mode 的優勢**：可以只更新畫面的一部分，降低功耗。

**範例**：智慧手錶的時間顯示

```c
// 假設時間顯示在螢幕的 (100, 100) 到 (220, 140) 區域

void update_time_display(const char *time_str) {
    // 1. 設定顯示區域（只更新時間區域）
    mipi_dsi_set_column_address(100, 220);  // X: 100-220
    mipi_dsi_set_page_address(100, 140);    // Y: 100-140
    
    // 2. 開始寫入資料
    mipi_dsi_dcs_write(0x2C);  // Write Memory Start
    
    // 3. 傳輸時間區域的畫面資料
    uint16_t *pixel_data = render_time_string(time_str);
    uint32_t pixel_count = (220 - 100 + 1) * (140 - 100 + 1);  // 121 * 41 = 4,961 pixels
    
    mipi_dsi_send_pixel_data(pixel_data, pixel_count);
    
    // 4. 傳輸完成（只傳輸了 4,961 * 2 = 9,922 bytes，而不是整個螢幕的 204,800 bytes）
}
```

**功耗對比**：

```
全螢幕更新（320x320）：
- 資料量：320 * 320 * 2 = 204,800 bytes
- 傳輸時間（100 MB/s）：2.05 ms
- 功耗：高

部分更新（121x41）：
- 資料量：121 * 41 * 2 = 9,922 bytes
- 傳輸時間（100 MB/s）：0.1 ms
- 功耗：低（降低 95%）
```

---

## MIPI DSI 的功耗優化

### 1. 使用 Command Mode

**策略**：

- 對於靜態畫面（如：時鐘、通知），使用 Command Mode
- 只在畫面變化時傳輸資料

**範例**：

```c
// 智慧手錶的時鐘顯示（每秒更新一次）

void clock_display_loop(void) {
    while (1) {
        // 1. 獲取當前時間
        char time_str[16];
        get_current_time(time_str);
        
        // 2. 只更新時間區域（Partial Update）
        update_time_display(time_str);
        
        // 3. 等待 1 秒
        delay_ms(1000);
    }
}
```

### 2. 使用 Partial Update

**策略**：

- 只更新變化的區域，而不是整個螢幕
- 降低資料傳輸量，降低功耗

**範例**：

```c
// 智慧手錶的通知顯示（只更新通知區域）

void notification_display(const char *message) {
    // 1. 設定通知區域（螢幕下方 1/3）
    mipi_dsi_set_column_address(0, 319);    // X: 0-319
    mipi_dsi_set_page_address(213, 319);    // Y: 213-319 (下方 1/3)
    
    // 2. 開始寫入資料
    mipi_dsi_dcs_write(0x2C);  // Write Memory Start
    
    // 3. 傳輸通知區域的畫面資料
    uint16_t *pixel_data = render_notification(message);
    uint32_t pixel_count = 320 * 107;  // 320 * 107 = 34,240 pixels
    
    mipi_dsi_send_pixel_data(pixel_data, pixel_count);
    
    // 4. 傳輸完成（只傳輸了 34,240 * 2 = 68,480 bytes，而不是整個螢幕的 204,800 bytes）
}
```

### 3. 使用 Frame Rate Control

**策略**：

- 降低更新率（從 60 FPS 降到 30 FPS 或更低）
- 對於靜態畫面，甚至可以降到 1 FPS

**範例**：

```c
// 智慧手錶的省電模式（降低更新率）

void enter_power_saving_mode(void) {
    // 1. 降低更新率到 1 FPS
    mipi_dsi_set_frame_rate(1);
    
    // 2. 降低亮度
    mipi_dsi_dcs_write_param(0x51, 0x40);  // Set Brightness: 64/255
    
    // 3. 只顯示時間（不顯示其他資訊）
    display_time_only();
}
```

### 4. 進入睡眠模式

**策略**：

- 當螢幕不需要顯示時，進入睡眠模式
- 功耗從 10 mA 降到 < 1 mA

**範例**：

```c
// 智慧手錶的螢幕關閉（進入睡眠模式）

void display_sleep(void) {
    // 1. 關閉 Display
    mipi_dsi_dcs_write(0x28);  // Set Display Off
    delay_ms(20);
    
    // 2. 進入睡眠模式
    mipi_dsi_dcs_write(0x10);  // Enter Sleep Mode
    delay_ms(120);
    
    // 3. 功耗降到 < 1 mA
}

void display_wakeup(void) {
    // 1. 退出睡眠模式
    mipi_dsi_dcs_write(0x11);  // Exit Sleep Mode
    delay_ms(120);
    
    // 2. 開啟 Display
    mipi_dsi_dcs_write(0x29);  // Set Display On
    delay_ms(20);
    
    // 3. 恢復正常顯示
}
```

---

## MIPI DSI vs SPI Display：實測對比

### 測試環境

- **Display**：320x320 解析度，16-bit RGB565
- **SPI Display**：Standard SPI，25 MHz
- **MIPI DSI**：1 Lane，500 Mbps

### 測試結果

| 測試項目 | SPI Display | MIPI DSI | 改善幅度 |
|---------|-------------|----------|---------|
| **全螢幕更新時間** | 82 ms | 3.3 ms | **25 倍** |
| **部分更新時間（10%）** | 8.2 ms | 0.33 ms | **25 倍** |
| **60 FPS 可行性** | ❌ 不可行（最多 12 FPS） | ✅ 可行 | - |
| **功耗（全螢幕更新）** | 15 mA | 12 mA | 降低 20% |
| **功耗（部分更新）** | 15 mA | 2 mA | **降低 87%** |
| **功耗（睡眠模式）** | 5 mA | 0.5 mA | **降低 90%** |
| **Pin 數量** | 6 個 | 4 個 | 減少 2 個 |

**結論**：

- **速度**：MIPI DSI 比 SPI Display 快 25 倍
- **功耗**：MIPI DSI 的功耗降低 87%（部分更新）
- **Pin 數量**：MIPI DSI 減少 2 個 GPIO

---

## 總結

MIPI DSI 是智慧手錶、智慧手機等裝置的標準 Display 接口，它提供了高速、低功耗、低 Pin 數的優勢。

**核心概念**：

- **三層架構**：D-PHY (Physical Layer) + Protocol Layer + Application Layer
- **雙模式**：High-Speed Mode（高速傳輸）+ Low-Power Mode（低功耗控制）
- **DCS (Display Command Set)**：標準化的 Display 控制命令
- **Video Mode vs Command Mode**：連續傳輸 vs 按需傳輸

**功耗優化**：

- **使用 Command Mode**：只在畫面變化時傳輸資料
- **使用 Partial Update**：只更新變化的區域
- **使用 Frame Rate Control**：降低更新率
- **進入睡眠模式**：功耗降到 < 1 mA

**實測結果**：

- **速度提升 25 倍**：從 82 ms 降到 3.3 ms（全螢幕更新）
- **功耗降低 87%**：從 15 mA 降到 2 mA（部分更新）
- **Pin 數量減少**：從 6 個降到 4 個

在下一篇文章中，我們將探討 **I2C, UART, GPIO**，了解 IoT 晶片的基礎接口。

---

## 參考資料

1. MIPI Alliance. (2011). *MIPI DSI Specification Version 1.2*. <https://www.mipi.org/specifications/dsi>
2. MIPI Alliance. (2014). *MIPI D-PHY Specification Version 1.2*. <https://www.mipi.org/specifications/d-phy>
3. MIPI Alliance. (2010). *MIPI DCS Specification Version 1.02*. <https://www.mipi.org/specifications/dcs>
4. Sitronix Technology. (2014). *ST7789V Datasheet*. <https://www.sitronix.com.tw/en/product/Driver/mobile_display.html>
5. Solomon Systech. (2015). *SSD1963 Datasheet*. <http://www.solomon-systech.com/en/product/advanced-display/tft-lcd-driver-controller/ssd1963/>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
