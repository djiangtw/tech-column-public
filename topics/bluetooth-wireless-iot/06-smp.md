# SMP - BLE 的安全機制

**作者**: Danny Jiang  
**日期**: 2025-12-11  

---

## 前言：凌晨三點的緊急會議

2015 年 2 月的一個週五晚上，我正準備下班，手機突然響了。

「Danny，你看到 Twitter 上的影片了嗎？」主管的聲音聽起來很緊張。

「什麼影片？」我一邊收拾東西，一邊問。

「有人破解了我們的智慧手環，」他說，「他在咖啡廳裡用筆電攔截了心率資料。影片已經有 5 萬次觀看了。」

我的心一沉。我立刻打開 Twitter，看到了那段影片：一位安全研究員坐在星巴克裡，筆電螢幕上顯示著即時的心率數據——75 bpm、78 bpm、82 bpm。他甚至沒有碰過那支手環，只是坐在 10 公尺外。

影片的標題是：「BLE 智慧手環的安全漏洞：任何人都能讀取你的心率」。

「這怎麼可能？」我想，「我們不是有加密嗎？」

週六凌晨三點，我們召開了緊急會議。我花了整晚檢查程式碼，終於找到了問題的根源。

我們使用的配對方式叫做「Just Works」——聽起來很方便，對吧？裝置自動配對，使用者不需要做任何事情。但它有一個致命的缺陷：**它不能防止中間人攻擊**。

想像一下這個場景：你以為你在和朋友說話，但實際上有個陌生人站在你們中間，假裝是你的朋友，然後偷聽你們的對話。這就是中間人攻擊。

攻擊者可以偽裝成手機，與手環配對，然後讀取所有資料。手環以為它在和手機說話，手機以為它在和手環說話，但實際上它們都在和攻擊者說話。

「我們需要改用更安全的配對方式，」我在會議上說。

「有哪些選擇？」主管問。

「有三種，」我說，「第一種是 Passkey Entry——手環顯示一個 6 位數字，使用者在手機上輸入。第二種是 Numeric Comparison——雙方都顯示一個 6 位數字，使用者確認是否相同。第三種是 Out of Band——透過 NFC 或 QR Code 傳送配對金鑰。」

「但我們的手環沒有螢幕和按鍵，」主管說，「怎麼顯示數字？怎麼讓使用者輸入？」

這是一個兩難的問題：**安全性 vs 使用者體驗**。

「我們可以在手環上預設一個固定的數字，」我建議，「例如 000000，然後在 App 上提示使用者輸入。」

「這樣還是不夠安全，」主管說，「攻擊者可以猜到。」

我們陷入了沉默。

經過一週的研究和測試，我們找到了解決方案：**使用 NFC（近場通訊）來傳送配對金鑰**。

使用者只需要把手機靠近手環（距離小於 5 公分），配對金鑰就會透過 NFC 傳送。這樣既安全（攻擊者無法在 5 公分內攔截），又方便（使用者只需要靠近一下）。

兩週後，我們發布了韌體更新。那位安全研究員在 Twitter 上發文：「已修復。做得好。」

這次經歷讓我深刻理解了一個道理：**安全不是可選項，而是必需品。但安全也不能犧牲使用者體驗——好的設計，應該讓安全變得無感**。

---

## SMP：Security Manager Protocol

### 什麼是 SMP？

**SMP（Security Manager Protocol）** 是 BLE 的安全管理協定，負責：

1. **Pairing（配對）**：建立安全連線
2. **Bonding（綁定）**：儲存配對金鑰，下次連線時自動加密
3. **Encryption（加密）**：使用 AES-128 加密資料
4. **Privacy（隱私）**：使用隨機位址保護裝置隱私

### SMP 在協定堆疊中的位置

```
┌─────────────────────────────────────────────────────────┐
│  Application Layer                                       │
├─────────────────────────────────────────────────────────┤
│  GATT, ATT                                              │
├─────────────────────────────────────────────────────────┤
│  SMP (Security Manager Protocol)                        │
│  ├─ Pairing (配對)                                      │
│  ├─ Bonding (綁定)                                      │
│  ├─ Encryption (加密)                                   │
│  └─ Privacy (隱私)                                      │
├─────────────────────────────────────────────────────────┤
│  L2CAP (Channel ID: 0x0006)                             │
├─────────────────────────────────────────────────────────┤
│  HCI                                                    │
├─────────────────────────────────────────────────────────┤
│  Link Layer                                             │
└─────────────────────────────────────────────────────────┘
```

---

## Pairing：配對方式

### BLE 的配對方式

BLE 支援四種配對方式：

| 配對方式 | MITM 保護 | 需要的 I/O 能力 | 說明 |
|---------|----------|---------------|------|
| **Just Works** | ❌ | 無 | 自動配對，無需使用者互動 |
| **Passkey Entry** | ✅ | 顯示或輸入 | 一方顯示 6 位數字，另一方輸入 |
| **Numeric Comparison** | ✅ | 顯示 + 按鍵 | 雙方顯示 6 位數字，使用者確認是否相同 |
| **Out of Band (OOB)** | ✅ | NFC、QR Code | 透過其他通道傳送配對金鑰 |

### Just Works

**流程**：

```
Initiator (手機)                Responder (智慧手環)
     |                                |
     |  Pairing Request               |
     |------------------------------->|
     |                                |
     |  Pairing Response              |
     |<-------------------------------|
     |                                |
     |  (自動產生 TK = 0)              |
     |                                |
     |  Pairing Confirm               |
     |------------------------------->|
     |                                |
     |  Pairing Confirm               |
     |<-------------------------------|
     |                                |
     |  Pairing Random                |
     |------------------------------->|
     |                                |
     |  Pairing Random                |
     |<-------------------------------|
     |                                |
     |  (產生 STK，啟用加密)           |
     |                                |
```

**優點**：

- 無需使用者互動
- 適合沒有螢幕和按鍵的裝置

**缺點**：

- **不提供 MITM 保護**
- 攻擊者可以偽裝成配對裝置

### Passkey Entry

**流程**：

```
Initiator (手機)                Responder (智慧手環)
     |                                |
     |  Pairing Request               |
     |------------------------------->|
     |                                |
     |  Pairing Response              |
     |<-------------------------------|
     |                                |
     |  (手環顯示 Passkey: 123456)     |
     |                                |
     |  (使用者在手機上輸入 123456)     |
     |                                |
     |  Pairing Confirm               |
     |------------------------------->|
     |                                |
     |  Pairing Confirm               |
     |<-------------------------------|
     |                                |
     |  Pairing Random                |
     |------------------------------->|
     |                                |
     |  Pairing Random                |
     |<-------------------------------|
     |                                |
     |  (產生 STK，啟用加密)           |
     |                                |
```

**優點**：

- **提供 MITM 保護**
- 攻擊者無法猜到 Passkey（6 位數字，1,000,000 種組合）

**缺點**：

- 需要螢幕或按鍵
- 使用者體驗較差

### Numeric Comparison（BLE 4.2+）

**流程**：

```
Initiator (手機)                Responder (智慧手環)
     |                                |
     |  Pairing Request               |
     |------------------------------->|
     |                                |
     |  Pairing Response              |
     |<-------------------------------|
     |                                |
     |  Public Key Exchange           |
     |<------------------------------>|
     |                                |
     |  (雙方顯示 6 位數字: 123456)    |
     |                                |
     |  (使用者確認數字相同)           |
     |                                |
     |  Pairing Confirm               |
     |------------------------------->|
     |                                |
     |  Pairing Confirm               |
     |<-------------------------------|
     |                                |
     |  Pairing Random                |
     |------------------------------->|
     |                                |
     |  Pairing Random                |
     |<-------------------------------|
     |                                |
     |  (產生 LTK，啟用加密)           |
     |                                |
```

**優點**：

- **提供 MITM 保護**
- 使用者體驗較好（只需確認，不需輸入）

**缺點**：

- 需要螢幕和按鍵
- 只支援 BLE 4.2+

### Out of Band (OOB)

**流程**：

```
Initiator (手機)                Responder (智慧手環)
     |                                |
     |  (透過 NFC 傳送 OOB 資料)       |
     |<-------------------------------|
     |                                |
     |  Pairing Request               |
     |  (OOB Flag = 1)                |
     |------------------------------->|
     |                                |
     |  Pairing Response              |
     |  (OOB Flag = 1)                |
     |<-------------------------------|
     |                                |
     |  (使用 OOB 資料產生 TK)         |
     |                                |
     |  Pairing Confirm               |
     |------------------------------->|
     |                                |
     |  Pairing Confirm               |
     |<-------------------------------|
     |                                |
     |  Pairing Random                |
     |------------------------------->|
     |                                |
     |  Pairing Random                |
     |<-------------------------------|
     |                                |
     |  (產生 STK，啟用加密)           |
     |                                |
```

**優點**：

- **提供 MITM 保護**
- 無需螢幕和按鍵
- 使用者體驗好

**缺點**：

- 需要額外的硬體（NFC、QR Code）

---

## Bonding：綁定與金鑰儲存

### 什麼是 Bonding？

**Bonding（綁定）** 是將配對金鑰儲存起來，下次連線時自動加密，無需重新配對。

**Bonding 的好處**：

- 提升使用者體驗（無需每次都配對）
- 提升安全性（使用長期金鑰 LTK）

### Bonding 流程

```
第一次配對：
Initiator (手機)                Responder (智慧手環)
     |                                |
     |  Pairing Request               |
     |  (Bonding Flag = 1)            |
     |------------------------------->|
     |                                |
     |  Pairing Response              |
     |  (Bonding Flag = 1)            |
     |<-------------------------------|
     |                                |
     |  (配對流程...)                  |
     |                                |
     |  (產生 LTK)                     |
     |                                |
     |  Encryption Information        |
     |  (LTK, EDIV, Rand)             |
     |<-------------------------------|
     |                                |
     |  (儲存 LTK)                     |
     |                                |

下次連線：
Initiator (手機)                Responder (智慧手環)
     |                                |
     |  LL_ENC_REQ                    |
     |  (EDIV, Rand)                  |
     |------------------------------->|
     |                                |
     |  (使用儲存的 LTK)               |
     |                                |
     |  LL_ENC_RSP                    |
     |<-------------------------------|
     |                                |
     |  (啟用加密)                     |
     |                                |
```

### 金鑰的儲存

**需要儲存的金鑰**：

| 金鑰 | 說明 | 大小 |
|------|------|------|
| **LTK（Long Term Key）** | 長期金鑰，用於加密 | 128 bits |
| **EDIV（Encrypted Diversifier）** | 金鑰識別碼 | 16 bits |
| **Rand（Random Number）** | 隨機數 | 64 bits |
| **IRK（Identity Resolving Key）** | 身份解析金鑰，用於解析隨機位址 | 128 bits |
| **CSRK（Connection Signature Resolving Key）** | 簽名金鑰 | 128 bits |

**程式碼範例**：

```c
// Bonding 資料結構
typedef struct {
    uint8_t peer_address[6];
    uint8_t peer_address_type;
    uint8_t ltk[16];
    uint16_t ediv;
    uint8_t rand[8];
    uint8_t irk[16];
    uint8_t csrk[16];
} bonding_info_t;

// 儲存 Bonding 資訊
void save_bonding_info(bonding_info_t *info) {
    // 儲存到 Flash
    flash_write(BONDING_INFO_ADDRESS, (uint8_t*)info, sizeof(bonding_info_t));
}

// 讀取 Bonding 資訊
void load_bonding_info(bonding_info_t *info) {
    // 從 Flash 讀取
    flash_read(BONDING_INFO_ADDRESS, (uint8_t*)info, sizeof(bonding_info_t));
}

// 檢查是否已綁定
bool is_bonded(uint8_t *peer_address) {
    bonding_info_t info;
    load_bonding_info(&info);

    return memcmp(info.peer_address, peer_address, 6) == 0;
}
```

---

## Encryption：加密

### AES-128 加密

BLE 使用 **AES-128** 加密演算法：

- **金鑰長度**：128 bits
- **區塊大小**：128 bits
- **模式**：CCM（Counter with CBC-MAC）

### 加密流程

```
Initiator (手機)                Responder (智慧手環)
     |                                |
     |  LL_ENC_REQ                    |
     |  (Rand, EDIV, SKDm, IVm)       |
     |------------------------------->|
     |                                |
     |  LL_ENC_RSP                    |
     |  (SKDs, IVs)                   |
     |<-------------------------------|
     |                                |
     |  (產生 Session Key: SK = LTK)  |
     |                                |
     |  LL_START_ENC_REQ              |
     |------------------------------->|
     |                                |
     |  LL_START_ENC_RSP              |
     |<-------------------------------|
     |                                |
     |  (啟用加密)                     |
     |                                |
```

### 程式碼範例

```c
// 啟用加密
void enable_encryption(uint16_t handle, uint8_t *ltk, uint16_t ediv, uint8_t *rand) {
    uint8_t cmd[] = {
        0x01,              // Packet Type: Command
        0x19, 0x20,        // Opcode: 0x2019 (HCI_LE_Start_Encryption)
        0x1C,              // Parameter Length: 28
        handle & 0xFF,     // Connection Handle (LSB)
        handle >> 8,       // Connection Handle (MSB)
        // Random Number (8 bytes)
        rand[0], rand[1], rand[2], rand[3], rand[4], rand[5], rand[6], rand[7],
        // EDIV (2 bytes)
        ediv & 0xFF, ediv >> 8,
        // LTK (16 bytes)
        ltk[0], ltk[1], ltk[2], ltk[3], ltk[4], ltk[5], ltk[6], ltk[7],
        ltk[8], ltk[9], ltk[10], ltk[11], ltk[12], ltk[13], ltk[14], ltk[15]
    };
    uart_send(cmd, 32);
}

// 處理加密完成事件
void handle_encryption_change_event(uint8_t *event) {
    uint8_t status = event[3];
    uint16_t handle = event[4] | (event[5] << 8);
    uint8_t encryption_enabled = event[6];

    if (status == 0x00 && encryption_enabled == 0x01) {
        printf("Encryption enabled on connection 0x%04X\n", handle);
    } else {
        printf("Encryption failed: status=0x%02X\n", status);
    }
}
```

---

## Privacy：隱私保護

### 為什麼需要 Privacy？

**問題**：BLE 裝置使用固定的 MAC 位址，可以被追蹤。

**解決方案**：使用**隨機位址（Random Address）**，定期更換。

### 隨機位址的類型

| 類型 | 說明 | 格式 |
|------|------|------|
| **Static Random Address** | 靜態隨機位址，開機時產生，不會更換 | 最高 2 bits = 11 |
| **Private Resolvable Address** | 可解析的隨機位址，使用 IRK 解析 | 最高 2 bits = 01 |
| **Private Non-Resolvable Address** | 不可解析的隨機位址 | 最高 2 bits = 00 |

### Private Resolvable Address

**產生流程**：

```
1. 產生 24-bit 隨機數 (prand)
2. 使用 IRK 和 prand 計算 hash = AES-128(IRK, prand)
3. 取 hash 的最高 24 bits
4. 組合成 48-bit 位址: [hash (24 bits)] [prand (24 bits)]
```

**解析流程**：

```
1. 從位址中提取 hash 和 prand
2. 使用儲存的 IRK 和 prand 計算 hash' = AES-128(IRK, prand)
3. 比較 hash 和 hash'，如果相同則解析成功
```

**程式碼範例**：

```c
// 產生 Private Resolvable Address
void generate_resolvable_address(uint8_t *irk, uint8_t *address) {
    // 1. 產生 24-bit 隨機數
    uint8_t prand[3];
    random_generate(prand, 3);
    prand[2] |= 0x40;  // 設定最高 2 bits = 01

    // 2. 計算 hash = AES-128(IRK, prand)
    uint8_t plaintext[16] = {0};
    memcpy(plaintext, prand, 3);

    uint8_t hash[16];
    aes_128_encrypt(irk, plaintext, hash);

    // 3. 組合成位址
    memcpy(address, hash, 3);      // hash (24 bits)
    memcpy(address + 3, prand, 3); // prand (24 bits)
}

// 解析 Private Resolvable Address
bool resolve_resolvable_address(uint8_t *irk, uint8_t *address) {
    // 1. 提取 hash 和 prand
    uint8_t hash[3];
    uint8_t prand[3];
    memcpy(hash, address, 3);
    memcpy(prand, address + 3, 3);

    // 2. 計算 hash' = AES-128(IRK, prand)
    uint8_t plaintext[16] = {0};
    memcpy(plaintext, prand, 3);

    uint8_t hash_calculated[16];
    aes_128_encrypt(irk, plaintext, hash_calculated);

    // 3. 比較 hash 和 hash'
    return memcmp(hash, hash_calculated, 3) == 0;
}

// 設定隨機位址
void set_random_address(uint8_t *address) {
    uint8_t cmd[] = {
        0x01,              // Packet Type: Command
        0x05, 0x20,        // Opcode: 0x2005 (HCI_LE_Set_Random_Address)
        0x06,              // Parameter Length: 6
        address[0], address[1], address[2], address[3], address[4], address[5]
    };
    uart_send(cmd, 10);
}
```

---

## 安全最佳實踐

### 1. 選擇正確的配對方式

| 裝置類型 | 建議的配對方式 | 原因 |
|---------|--------------|------|
| 智慧手環（無螢幕） | OOB (NFC) | 提供 MITM 保護，無需螢幕 |
| 智慧手錶（有螢幕） | Numeric Comparison | 提供 MITM 保護，使用者體驗好 |
| 智慧門鎖 | Passkey Entry | 提供 MITM 保護，安全性高 |
| Beacon | Just Works | 無需配對，只廣播資料 |

### 2. 啟用 Bonding

- 儲存 LTK，下次連線時自動加密
- 定期檢查 Bonding 資訊的有效性

### 3. 使用隨機位址

- 使用 Private Resolvable Address 保護隱私
- 定期更換位址（建議 15 分鐘）

### 4. 實作安全的金鑰儲存

```c
// 使用 Flash 的安全區域儲存金鑰
#define BONDING_INFO_ADDRESS  0x0007F000  // Flash 的最後一個 Page

// 加密儲存（使用裝置唯一的金鑰）
void secure_save_bonding_info(bonding_info_t *info) {
    uint8_t encrypted_data[sizeof(bonding_info_t)];

    // 使用裝置唯一的金鑰加密
    aes_128_encrypt(device_unique_key, (uint8_t*)info, encrypted_data);

    // 儲存到 Flash
    flash_write(BONDING_INFO_ADDRESS, encrypted_data, sizeof(bonding_info_t));
}
```

### 5. 防止重放攻擊

```c
// 使用 Nonce（Number used once）防止重放攻擊
typedef struct {
    uint32_t nonce;
    uint8_t data[20];
} secure_packet_t;

// 發送端
void send_secure_packet(uint8_t *data, uint16_t len) {
    secure_packet_t packet;
    packet.nonce = get_next_nonce();  // 遞增的 Nonce
    memcpy(packet.data, data, len);

    // 發送
    att_send_notification((uint8_t*)&packet, sizeof(secure_packet_t));
}

// 接收端
bool verify_secure_packet(secure_packet_t *packet) {
    // 檢查 Nonce 是否大於上次的 Nonce
    if (packet->nonce <= last_nonce) {
        printf("Replay attack detected!\n");
        return false;
    }

    last_nonce = packet->nonce;
    return true;
}
```

---

## 總結

SMP 是 BLE 的安全管理協定，提供配對、綁定、加密、隱私保護等功能。

**核心概念**：

- **Pairing**：Just Works、Passkey Entry、Numeric Comparison、OOB
- **Bonding**：儲存 LTK，下次連線時自動加密
- **Encryption**：AES-128 加密
- **Privacy**：使用隨機位址保護隱私

**安全最佳實踐**：

- 選擇正確的配對方式（根據裝置的 I/O 能力）
- 啟用 Bonding（提升使用者體驗）
- 使用隨機位址（保護隱私）
- 實作安全的金鑰儲存（加密儲存）
- 防止重放攻擊（使用 Nonce）

在下一篇文章中，我們將深入探討 **BLE Beacon**，了解如何使用 BLE 廣播實現室內定位和接近行銷。

---

## 參考資料

1. Bluetooth SIG. (2013). *Bluetooth Core Specification Version 4.1 - Vol 3, Part H: SMP*. <https://www.bluetooth.com/specifications/specs/core-specification-4-1/>
2. Bluetooth SIG. (2014). *Bluetooth Core Specification Version 4.2 - Vol 3, Part H: SMP*. <https://www.bluetooth.com/specifications/specs/core-specification-4-2/>
3. Bluetooth SIG. (2014). *Bluetooth Security White Paper*. <https://www.bluetooth.com/>
4. Mike Ryan. (2013). *Bluetooth: With Low Energy Comes Low Security*. USENIX WOOT'13.

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
