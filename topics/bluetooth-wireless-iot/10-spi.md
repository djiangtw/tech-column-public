# SPI 深入解析 - 從 Standard 到 Dual/Quad SPI

**作者**: Danny Jiang  
**日期**: 2025-12-11  

---

## 前言：一個從零實作 Quad SPI 的挑戰

2014 年 10 月，我們的智慧手錶專案遇到了一個瓶頸：開機時間太長。

客戶的要求是「從按下電源鍵到顯示 Logo，不能超過 2 秒」。但我們的第一版韌體，開機時間長達 8 秒。

問題出在哪裡？

經過分析，我們發現瓶頸在 **SPI Flash 的讀取速度**。智慧手錶的韌體（包含 BLE Stack、Display Driver、UI 資源）大約 2 MB，全部存放在 SPI Flash 中。開機時，CPU 需要從 Flash 讀取韌體到 RAM，然後才能執行。

當時我們使用的是 **Standard SPI**（單線傳輸），時脈頻率 25 MHz，理論速度是：

```
速度 = 25 MHz / 8 bits = 3.125 MB/s
實際速度（考慮 overhead）= 2.5 MB/s

讀取 2 MB 韌體的時間 = 2 MB / 2.5 MB/s = 0.8 秒
```

但這只是 Flash 讀取的時間，加上 CPU 初始化、Display 初始化、BLE 初始化，總共需要 8 秒。

客戶說：「能不能加快 Flash 讀取速度？」

我查了 Flash 的 datasheet，發現它支援 **Quad SPI**（四線並行傳輸），理論速度可以提升 4 倍。但問題是：**我們的 SoC 晶片不支援 Quad SPI**。

「那就自己實作吧！」主管說。

於是，我花了 3 週時間，從零開始實作 Quad SPI 驅動。這是我第一次深入理解 SPI 的每個細節：時脈相位、資料對齊、Pin Mux、DMA 傳輸。

最終，我們將 Flash 讀取速度提升到 10 MB/s，開機時間縮短到 1.5 秒，達到了客戶的要求。

這段經歷讓我深刻體會到：**理解底層接口的細節，是優化性能的關鍵**。

---

## SPI 基本原理：Master-Slave 架構

### 什麼是 SPI？

**SPI (Serial Peripheral Interface)** 是一種同步串列通訊協定，由 Motorola 在 1980 年代發明。它是嵌入式系統中最常用的接口之一。

**SPI 的特性**：

- **同步傳輸**：使用時脈訊號（SCLK）同步資料傳輸
- **全雙工**：可以同時發送和接收資料
- **Master-Slave 架構**：一個 Master 控制多個 Slave
- **高速**：時脈頻率可達 50 MHz 以上

### SPI 的 4 條訊號線

SPI 使用 4 條訊號線：

```
┌─────────────────────────────────────────────────────────┐
│  SPI 訊號線                                               │
├─────────────────────────────────────────────────────────┤
│  1. SCLK (Serial Clock)        - 時脈訊號（Master 產生）  │
│  2. MOSI (Master Out Slave In) - Master 發送資料         │
│  3. MISO (Master In Slave Out) - Slave 發送資料          │
│  4. CS/SS (Chip Select)        - 選擇 Slave              │
└─────────────────────────────────────────────────────────┘
```

**訊號線說明**：

1. **SCLK (Serial Clock)**：
   - 由 Master 產生的時脈訊號
   - 用於同步資料傳輸
   - 頻率範圍：1 MHz - 50 MHz（甚至更高）

2. **MOSI (Master Out Slave In)**：
   - Master 發送資料到 Slave
   - 也稱為 SDO (Serial Data Out) 或 TX

3. **MISO (Master In Slave Out)**：
   - Slave 發送資料到 Master
   - 也稱為 SDI (Serial Data In) 或 RX

4. **CS/SS (Chip Select / Slave Select)**：
   - 選擇要通訊的 Slave
   - 低電位有效（Active Low）
   - 一個 Master 可以連接多個 Slave，每個 Slave 有獨立的 CS

### SPI 的連接方式

**單一 Slave**：

```
┌─────────────┐                    ┌─────────────┐
│   Master    │                    │   Slave     │
│             │                    │             │
│  SCLK   ────┼────────────────────┼────  SCLK   │
│  MOSI   ────┼────────────────────┼────  MOSI   │
│  MISO   ────┼────────────────────┼────  MISO   │
│  CS     ────┼────────────────────┼────  CS     │
│             │                    │             │
└─────────────┘                    └─────────────┘
```

**多個 Slave**：

```
┌─────────────┐
│   Master    │
│             │
│  SCLK   ────┼────────┬───────────┬───────────┐
│  MOSI   ────┼────────┼───────────┼───────────┤
│  MISO   ────┼────────┼───────────┼───────────┤
│  CS0    ────┼────────┘           │           │
│  CS1    ────┼────────────────────┘           │
│  CS2    ────┼────────────────────────────────┘
│             │
└─────────────┘
       │              │              │
       ▼              ▼              ▼
   ┌───────┐      ┌───────┐      ┌───────┐
   │Slave 0│      │Slave 1│      │Slave 2│
   └───────┘      └───────┘      └───────┘
```

**注意**：

- SCLK、MOSI、MISO 是共用的（Bus）
- 每個 Slave 有獨立的 CS
- 同一時間只能與一個 Slave 通訊（CS 為低電位）

---

## SPI 的四種模式：CPOL 與 CPHA

SPI 有 4 種工作模式，由兩個參數決定：

- **CPOL (Clock Polarity)**：時脈極性
  - CPOL = 0：閒置時 SCLK 為低電位
  - CPOL = 1：閒置時 SCLK 為高電位

- **CPHA (Clock Phase)**：時脈相位
  - CPHA = 0：在時脈的第一個邊緣取樣資料
  - CPHA = 1：在時脈的第二個邊緣取樣資料

### 四種模式的時序圖

**Mode 0 (CPOL=0, CPHA=0)**：

```
CS    ────┐                                    ┌────
          └────────────────────────────────────┘

SCLK  ────┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌────
          └───┘   └───┘   └───┘   └───┘   └───┘
          ↑       ↑       ↑       ↑       ↑
MOSI  ────┤ D7 ┤ D6 ┤ D5 ┤ D4 ┤ D3 ┤ D2 ┤ D1 ┤ D0 ┤
          └───────────────────────────────────────┘
          ↑       ↑       ↑       ↑       ↑
          取樣    取樣    取樣    取樣    取樣
```

**Mode 1 (CPOL=0, CPHA=1)**：

```
CS    ────┐                                    ┌────
          └────────────────────────────────────┘

SCLK  ────┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌────
          └───┘   └───┘   └───┘   └───┘   └───┘
              ↑       ↑       ↑       ↑       ↑
MOSI  ────┤ D7 ┤ D6 ┤ D5 ┤ D4 ┤ D3 ┤ D2 ┤ D1 ┤ D0 ┤
          └───────────────────────────────────────┘
              ↑       ↑       ↑       ↑       ↑
              取樣    取樣    取樣    取樣    取樣
```

**Mode 2 (CPOL=1, CPHA=0)**：

```
CS    ────┐                                    ┌────
          └────────────────────────────────────┘

SCLK  ────┘   └───┐   └───┐   └───┐   └───┐   └────
              ┌───┘   ┌───┘   ┌───┘   ┌───┘
          ↑       ↑       ↑       ↑       ↑
MOSI  ────┤ D7 ┤ D6 ┤ D5 ┤ D4 ┤ D3 ┤ D2 ┤ D1 ┤ D0 ┤
          └───────────────────────────────────────┘
          ↑       ↑       ↑       ↑       ↑
          取樣    取樣    取樣    取樣    取樣
```

**Mode 3 (CPOL=1, CPHA=1)**：

```
CS    ────┐                                    ┌────
          └────────────────────────────────────┘

SCLK  ────┘   └───┐   └───┐   └───┐   └───┐   └────
              ┌───┘   ┌───┘   ┌───┘   ┌───┘
              ↑       ↑       ↑       ↑       ↑
MOSI  ────┤ D7 ┤ D6 ┤ D5 ┤ D4 ┤ D3 ┤ D2 ┤ D1 ┤ D0 ┤
          └───────────────────────────────────────┘
              ↑       ↑       ↑       ↑       ↑
              取樣    取樣    取樣    取樣    取樣
```

### 如何選擇 SPI 模式？

**選擇原則**：

1. **查看 Slave 的 datasheet**：Slave 裝置會指定支援哪些模式
2. **最常用的模式**：Mode 0 和 Mode 3
3. **相容性考量**：如果有多個 Slave，選擇它們都支援的模式

**常見裝置的 SPI 模式**：

| 裝置類型 | 常用模式 |
|---------|---------|
| SPI Flash (如 W25Q128) | Mode 0, Mode 3 |
| SD Card | Mode 0 |
| OLED Display (如 SSD1306) | Mode 0, Mode 3 |
| ADC (如 MCP3008) | Mode 0 |
| Accelerometer (如 ADXL345) | Mode 3 |

---

## Standard SPI vs Dual SPI vs Quad SPI

### Standard SPI：單線傳輸

**Standard SPI** 使用 1 條 MOSI 和 1 條 MISO，每次傳輸 1 bit。

**速度計算**：

```
假設 SCLK = 25 MHz

每個時脈週期傳輸 1 bit
速度 = 25 MHz / 8 bits/byte = 3.125 MB/s
```

**優點**：

- 簡單，硬體成本低
- 相容性好，所有 SPI 裝置都支援

**缺點**：

- 速度慢（對於大容量 Flash 讀取）

### Dual SPI：2 線並行傳輸

**Dual SPI** 使用 2 條資料線（IO0, IO1），每次傳輸 2 bits。

**Pin 定義**：

```
Standard SPI:
- MOSI (IO0): Master → Slave
- MISO (IO1): Slave → Master

Dual SPI:
- IO0: 雙向（可以是 Master → Slave 或 Slave → Master）
- IO1: 雙向
```

**速度計算**：

```
假設 SCLK = 25 MHz

每個時脈週期傳輸 2 bits
速度 = 25 MHz * 2 bits / 8 bits/byte = 6.25 MB/s
```

**優點**：

- 速度提升 2 倍
- Pin 數量不變（IO0, IO1 原本就存在）

**缺點**：

- 需要 Flash 支援 Dual SPI
- 需要軟體配置 Pin Mux（將 MOSI/MISO 切換為雙向模式）

### Quad SPI：4 線並行傳輸

**Quad SPI** 使用 4 條資料線（IO0, IO1, IO2, IO3），每次傳輸 4 bits。

**Pin 定義**：

```
Standard SPI:
- MOSI (IO0): Master → Slave
- MISO (IO1): Slave → Master
- WP (IO2): Write Protect（寫保護）
- HOLD (IO3): Hold（暫停）

Quad SPI:
- IO0: 雙向
- IO1: 雙向
- IO2: 雙向（原本是 WP）
- IO3: 雙向（原本是 HOLD）
```

**速度計算**：

```
假設 SCLK = 25 MHz

每個時脈週期傳輸 4 bits
速度 = 25 MHz * 4 bits / 8 bits/byte = 12.5 MB/s
```

**優點**：

- 速度提升 4 倍
- 適合大容量 Flash 讀取

**缺點**：

- 需要 Flash 支援 Quad SPI
- 需要額外的 Pin（IO2, IO3）
- 需要軟體配置 Pin Mux

### 對比表

| 模式 | 資料線數量 | 每次傳輸 | 速度（25 MHz） | Pin 數量 |
|------|-----------|---------|---------------|---------|
| **Standard SPI** | 1 (MOSI) + 1 (MISO) | 1 bit | 3.125 MB/s | 4 (SCLK, MOSI, MISO, CS) |
| **Dual SPI** | 2 (IO0, IO1) | 2 bits | 6.25 MB/s | 4 (SCLK, IO0, IO1, CS) |
| **Quad SPI** | 4 (IO0-IO3) | 4 bits | 12.5 MB/s | 6 (SCLK, IO0-IO3, CS) |

---

## Quad SPI 的實作細節

### Quad SPI 的三個階段

Quad SPI 的讀取操作分為 3 個階段：

```
┌─────────────────────────────────────────────────────────┐
│  Quad SPI Read Operation                                 │
├─────────────────────────────────────────────────────────┤
│  Phase 1: Command (Standard SPI, 1-bit)                 │
│  ├─ 發送命令：0xEB (Quad I/O Fast Read)                 │
│  └─ 使用 MOSI (IO0) 傳輸                                 │
├─────────────────────────────────────────────────────────┤
│  Phase 2: Address (Quad SPI, 4-bit)                     │
│  ├─ 發送地址：24-bit address (3 bytes)                  │
│  └─ 使用 IO0-IO3 並行傳輸                                │
├─────────────────────────────────────────────────────────┤
│  Phase 3: Data (Quad SPI, 4-bit)                        │
│  ├─ 接收資料：N bytes                                    │
│  └─ 使用 IO0-IO3 並行傳輸                                │
└─────────────────────────────────────────────────────────┘
```

**為什麼 Command 階段使用 Standard SPI？**

- Flash 在上電後，預設是 Standard SPI 模式
- 需要先發送命令（0xEB），告訴 Flash「接下來要使用 Quad SPI」
- Flash 收到命令後，切換到 Quad SPI 模式

### Quad SPI 的時序圖

```
CS    ────┐                                                ┌────
          └────────────────────────────────────────────────┘

SCLK  ────┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌────
          └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘

          ◄─ Command ─►◄──── Address ────►◄──── Data ────►
          (Standard)    (Quad)             (Quad)

IO0   ────┤1┤0┤1┤1┤1┤0┤1┤1┤A23┤A19┤A15┤...┤D7┤D3┤...
IO1   ────────────────────┤A22┤A18┤A14┤...┤D6┤D2┤...
IO2   ────────────────────┤A21┤A17┤A13┤...┤D5┤D1┤...
IO3   ────────────────────┤A20┤A16┤A12┤...┤D4┤D0┤...
```

**說明**：

1. **Command 階段**（8 bits）：
   - 使用 Standard SPI（只有 IO0）
   - 發送命令：0xEB（Quad I/O Fast Read）

2. **Address 階段**（24 bits = 6 個時脈週期）：
   - 使用 Quad SPI（IO0-IO3）
   - 每個時脈週期傳輸 4 bits
   - 24 bits / 4 bits = 6 個時脈週期

3. **Data 階段**（N bytes）：
   - 使用 Quad SPI（IO0-IO3）
   - 每個時脈週期傳輸 4 bits
   - 每 2 個時脈週期傳輸 1 byte

### Quad SPI 的 Pin Mux 配置

**問題**：

IO2 和 IO3 原本是 WP（Write Protect）和 HOLD（Hold）功能，如何切換為資料線？

**解決方案**：

1. **在 Flash 內部配置**：
   - 發送命令 0x01（Write Status Register）
   - 設定 QE (Quad Enable) bit = 1
   - Flash 會將 IO2 和 IO3 切換為資料線

2. **在 SoC 內部配置**：
   - 配置 GPIO Pin Mux，將 IO2 和 IO3 設定為 SPI 功能
   - 配置 SPI 控制器，啟用 Quad SPI 模式

**程式碼範例**（啟用 Quad SPI）：

```c
// 1. 讀取 Flash 的 Status Register
uint8_t status = spi_flash_read_status_register();

// 2. 設定 QE (Quad Enable) bit
status |= (1 << 6);  // QE bit 在 bit 6

// 3. 寫入 Status Register
spi_flash_write_status_register(status);

// 4. 等待 Flash 完成配置
while (spi_flash_is_busy());

// 5. 配置 SoC 的 Pin Mux
gpio_set_function(IO2_PIN, GPIO_FUNC_SPI);
gpio_set_function(IO3_PIN, GPIO_FUNC_SPI);

// 6. 配置 SPI 控制器
spi_set_mode(SPI_MODE_QUAD);
```

---

## SPI Flash 的操作命令

SPI Flash 支援多種操作命令，以 W25Q128 為例：

### 常用命令表

| 命令 | 代碼 | 說明 | 資料階段 |
|------|------|------|---------|
| **Read Data** | 0x03 | 標準讀取（Standard SPI） | Address (3 bytes) + Data (N bytes) |
| **Fast Read** | 0x0B | 快速讀取（Standard SPI） | Address (3 bytes) + Dummy (1 byte) + Data (N bytes) |
| **Dual Output Fast Read** | 0x3B | Dual SPI 讀取 | Address (3 bytes) + Dummy (1 byte) + Data (N bytes, Dual) |
| **Quad Output Fast Read** | 0x6B | Quad SPI 讀取（Address 仍是 Standard） | Address (3 bytes) + Dummy (1 byte) + Data (N bytes, Quad) |
| **Quad I/O Fast Read** | 0xEB | Quad SPI 讀取（Address 和 Data 都是 Quad） | Address (3 bytes, Quad) + Dummy (2 bytes) + Data (N bytes, Quad) |
| **Page Program** | 0x02 | 寫入資料（最多 256 bytes） | Address (3 bytes) + Data (N bytes, ≤ 256) |
| **Sector Erase** | 0x20 | 擦除 Sector（4 KB） | Address (3 bytes) |
| **Block Erase (32KB)** | 0x52 | 擦除 Block（32 KB） | Address (3 bytes) |
| **Block Erase (64KB)** | 0xD8 | 擦除 Block（64 KB） | Address (3 bytes) |
| **Chip Erase** | 0xC7 | 擦除整個 Flash | 無 |
| **Read Status Register** | 0x05 | 讀取狀態暫存器 | Data (1 byte) |
| **Write Enable** | 0x06 | 啟用寫入 | 無 |
| **Write Disable** | 0x04 | 禁用寫入 | 無 |

### 讀取操作的程式碼範例

**Standard SPI Read**：

```c
void spi_flash_read_standard(uint32_t address, uint8_t *buffer, uint32_t length) {
    // 1. 拉低 CS
    gpio_set_low(CS_PIN);
    
    // 2. 發送命令：0x03 (Read Data)
    spi_transfer_byte(0x03);
    
    // 3. 發送地址（24-bit，MSB first）
    spi_transfer_byte((address >> 16) & 0xFF);
    spi_transfer_byte((address >> 8) & 0xFF);
    spi_transfer_byte(address & 0xFF);
    
    // 4. 接收資料
    for (uint32_t i = 0; i < length; i++) {
        buffer[i] = spi_transfer_byte(0xFF);  // 發送 dummy byte，接收資料
    }
    
    // 5. 拉高 CS
    gpio_set_high(CS_PIN);
}
```

**Quad SPI Read**：

```c
void spi_flash_read_quad(uint32_t address, uint8_t *buffer, uint32_t length) {
    // 1. 拉低 CS
    gpio_set_low(CS_PIN);
    
    // 2. 發送命令：0xEB (Quad I/O Fast Read)（Standard SPI）
    spi_set_mode(SPI_MODE_STANDARD);
    spi_transfer_byte(0xEB);
    
    // 3. 切換到 Quad SPI 模式
    spi_set_mode(SPI_MODE_QUAD);
    
    // 4. 發送地址（24-bit，Quad SPI）
    spi_transfer_quad_byte((address >> 16) & 0xFF);
    spi_transfer_quad_byte((address >> 8) & 0xFF);
    spi_transfer_quad_byte(address & 0xFF);
    
    // 5. 發送 Dummy bytes（2 bytes）
    spi_transfer_quad_byte(0xFF);
    spi_transfer_quad_byte(0xFF);
    
    // 6. 接收資料（Quad SPI）
    for (uint32_t i = 0; i < length; i++) {
        buffer[i] = spi_transfer_quad_byte(0xFF);
    }
    
    // 7. 拉高 CS
    gpio_set_high(CS_PIN);
    
    // 8. 切換回 Standard SPI 模式
    spi_set_mode(SPI_MODE_STANDARD);
}
```

### 寫入操作的程式碼範例

**Page Program**（寫入資料）：

```c
void spi_flash_page_program(uint32_t address, const uint8_t *data, uint32_t length) {
    // 1. 啟用寫入
    gpio_set_low(CS_PIN);
    spi_transfer_byte(0x06);  // Write Enable
    gpio_set_high(CS_PIN);
    
    // 2. 拉低 CS
    gpio_set_low(CS_PIN);
    
    // 3. 發送命令：0x02 (Page Program)
    spi_transfer_byte(0x02);
    
    // 4. 發送地址（24-bit）
    spi_transfer_byte((address >> 16) & 0xFF);
    spi_transfer_byte((address >> 8) & 0xFF);
    spi_transfer_byte(address & 0xFF);
    
    // 5. 發送資料（最多 256 bytes）
    for (uint32_t i = 0; i < length && i < 256; i++) {
        spi_transfer_byte(data[i]);
    }
    
    // 6. 拉高 CS
    gpio_set_high(CS_PIN);
    
    // 7. 等待寫入完成
    while (spi_flash_is_busy());
}
```

**Sector Erase**（擦除 Sector）：

```c
void spi_flash_sector_erase(uint32_t address) {
    // 1. 啟用寫入
    gpio_set_low(CS_PIN);
    spi_transfer_byte(0x06);  // Write Enable
    gpio_set_high(CS_PIN);
    
    // 2. 拉低 CS
    gpio_set_low(CS_PIN);
    
    // 3. 發送命令：0x20 (Sector Erase)
    spi_transfer_byte(0x20);
    
    // 4. 發送地址（24-bit）
    spi_transfer_byte((address >> 16) & 0xFF);
    spi_transfer_byte((address >> 8) & 0xFF);
    spi_transfer_byte(address & 0xFF);
    
    // 5. 拉高 CS
    gpio_set_high(CS_PIN);
    
    // 6. 等待擦除完成（可能需要數百 ms）
    while (spi_flash_is_busy());
}
```

---

## SPI 的性能優化

### 1. 提高時脈頻率

**策略**：

- 將 SCLK 從 25 MHz 提高到 50 MHz
- 速度提升 2 倍

**限制**：

- Flash 的最大時脈頻率（查看 datasheet）
- PCB 走線長度（過長會導致訊號衰減）
- SoC 的 SPI 控制器最大頻率

**程式碼範例**：

```c
// 設定 SPI 時脈頻率為 50 MHz
spi_set_clock_frequency(50000000);
```

### 2. 使用 DMA 傳輸

**策略**：

- 使用 DMA（Direct Memory Access）傳輸資料
- CPU 不需要參與每個 byte 的傳輸，可以做其他事情

**優點**：

- 降低 CPU 負載
- 提高傳輸效率

**程式碼範例**：

```c
void spi_flash_read_dma(uint32_t address, uint8_t *buffer, uint32_t length) {
    // 1. 配置 DMA
    dma_config_t dma_config = {
        .src = SPI_DATA_REGISTER,
        .dst = buffer,
        .length = length,
        .direction = DMA_PERIPH_TO_MEM,
    };
    dma_configure(DMA_CHANNEL_0, &dma_config);
    
    // 2. 拉低 CS
    gpio_set_low(CS_PIN);
    
    // 3. 發送命令和地址（使用 CPU）
    spi_transfer_byte(0x03);  // Read Data
    spi_transfer_byte((address >> 16) & 0xFF);
    spi_transfer_byte((address >> 8) & 0xFF);
    spi_transfer_byte(address & 0xFF);
    
    // 4. 啟動 DMA 傳輸
    dma_start(DMA_CHANNEL_0);
    
    // 5. 等待 DMA 完成
    while (!dma_is_complete(DMA_CHANNEL_0));
    
    // 6. 拉高 CS
    gpio_set_high(CS_PIN);
}
```

### 3. 使用 Burst Mode

**策略**：

- 一次讀取大量資料（如 4 KB），而不是每次讀取幾個 bytes
- 減少 Command 和 Address 階段的 overhead

**範例**：

```
不好的做法：
for (int i = 0; i < 1000; i++) {
    spi_flash_read(address + i, &buffer[i], 1);  // 每次讀取 1 byte
}
// 總共需要 1000 次 Command + Address + Data

好的做法：
spi_flash_read(address, buffer, 1000);  // 一次讀取 1000 bytes
// 只需要 1 次 Command + Address + Data
```

### 4. 使用 Quad SPI

**策略**：

- 使用 Quad SPI 而非 Standard SPI
- 速度提升 4 倍

**實測結果**：

```
Standard SPI (25 MHz):
- 讀取 2 MB 韌體：0.8 秒

Quad SPI (25 MHz):
- 讀取 2 MB 韌體：0.2 秒

Quad SPI (50 MHz):
- 讀取 2 MB 韌體：0.1 秒
```

---

## 總結

SPI 是嵌入式系統中最常用的接口之一，理解其細節對於性能優化至關重要。

**核心概念**：

- **Master-Slave 架構**：一個 Master 控制多個 Slave
- **4 條訊號線**：SCLK, MOSI, MISO, CS
- **4 種模式**：CPOL 和 CPHA 的組合
- **Standard/Dual/Quad SPI**：1-bit、2-bit、4-bit 並行傳輸

**Quad SPI 的實作**：

- **3 個階段**：Command (Standard) → Address (Quad) → Data (Quad)
- **Pin Mux 配置**：將 IO2 和 IO3 切換為資料線
- **速度提升**：4 倍（相對於 Standard SPI）

**性能優化**：

- **提高時脈頻率**：25 MHz → 50 MHz
- **使用 DMA 傳輸**：降低 CPU 負載
- **使用 Burst Mode**：減少 overhead
- **使用 Quad SPI**：速度提升 4 倍

在下一篇文章中，我們將探討 **MIPI DSI**，了解智慧手錶如何驅動高解析度彩色螢幕。

---

## 參考資料

1. Motorola Inc. (1985). *SPI Block Guide*. <https://www.nxp.com/docs/en/data-sheet/SPI.pdf>
2. Winbond Electronics. (2014). *W25Q128FV Datasheet*. <https://www.winbond.com/resource-files/w25q128fv_revhh1_100913_website1.pdf>
3. STMicroelectronics. (2015). *SPI Protocol Specification*. <https://www.st.com/resource/en/application_note/dm00054618.pdf>
4. NXP Semiconductors. (2016). *SPI Flash Programming Guide*. <https://www.nxp.com/docs/en/application-note/AN4852.pdf>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
