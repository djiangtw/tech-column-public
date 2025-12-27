---
author: Danny Jiang
date: 2025-12-11
---

# IoT 安全實戰 - 從晶片到雲端

## 前言

2015 年 5 月，我接到了一個緊急電話。

那是一個週五下午 4 點，我正準備下班，手機突然響起。是我們的產品經理 Lisa：

"Danny，我們有麻煩了。客戶的安全團隊發現我們的智慧手錶有安全漏洞，他們要求我們在 2 週內修復，否則取消訂單。"

我的心一沉："什麼漏洞？"

Lisa 說："他們發現可以透過 BLE 連線竊取使用者的健康資料，而且可以透過 OTA 更新安裝惡意韌體。他們要求我們加強安全性：Secure Boot、加密 OTA、資料加密。"

我深吸一口氣："2 週？這些功能至少需要 1 個月才能完成。"

Lisa 說："我知道很趕，但這是我們最大的客戶，訂單金額超過 100 萬美元。我們必須想辦法。"

我掛上電話，立刻召集團隊開會。我在白板上寫下客戶的要求：

1. **Secure Boot**：防止未經授權的韌體運行
2. **加密 OTA**：防止 OTA 更新被竄改
3. **資料加密**：防止 BLE 通訊被竊聽
4. **認證機制**：防止未經授權的裝置連線

我看著團隊說："我們有 2 週時間。這是一場硬仗，但我們必須贏。"

接下來的 2 週，是我職業生涯中最緊張的時光。我們每天工作 12 小時，週末也不休息。最終，我們成功實作了所有安全功能，通過了客戶的安全審查。

這篇文章，我想分享這次 IoT 安全實戰的經驗：從晶片層的 Secure Boot，到通訊層的加密，再到雲端的認證，完整的端到端安全方案。

---

## IoT 安全威脅模型

在開始實作之前，我們先分析了 IoT 裝置可能面臨的安全威脅：

```
┌─────────────────────────────────────────────────────────────────┐
│  IoT 安全威脅模型                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  裝置層 (Device Layer)                                   │   │
│  │  - 韌體竄改：安裝惡意韌體                                │   │
│  │  - 除錯接口：透過 SWD/UART 讀取記憶體                    │   │
│  │  - 側通道攻擊：透過功耗分析破解金鑰                      │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  網路層 (Network Layer)                                  │   │
│  │  - 竊聽：攔截 BLE 通訊                                   │   │
│  │  - 中間人攻擊：偽裝成合法裝置                            │   │
│  │  - 重放攻擊：重放舊的封包                                │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  雲端層 (Cloud Layer)                                    │   │
│  │  - 未授權存取：未經授權存取使用者資料                    │   │
│  │  - 資料洩漏：資料庫被駭客入侵                            │   │
│  │  - DDoS 攻擊：大量請求癱瘓伺服器                         │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

**安全目標**：

1. **機密性 (Confidentiality)**：防止資料被竊取
2. **完整性 (Integrity)**：防止資料被竄改
3. **可用性 (Availability)**：防止服務被中斷
4. **認證 (Authentication)**：確認身份
5. **授權 (Authorization)**：控制存取權限

---

## 裝置層安全：Secure Boot

**Secure Boot** 是 IoT 安全的第一道防線，確保只有經過簽名的韌體才能運行。

### Secure Boot 原理

```
┌─────────────────────────────────────────────────────────┐
│  Secure Boot 流程                                         │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  1. 上電 → Bootloader 啟動                               │
│       ↓                                                   │
│  2. 讀取 Application 韌體                                │
│       ↓                                                   │
│  3. 驗證數位簽章（使用公鑰）                             │
│       ↓                                                   │
│  4. 簽章正確？                                           │
│       ├─ Yes → 啟動 Application                          │
│       └─ No  → 停止啟動，進入 DFU 模式                   │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**數位簽章**：

- 使用 **ECDSA (Elliptic Curve Digital Signature Algorithm)**
- 私鑰：用於簽署韌體（保存在開發端）
- 公鑰：用於驗證簽章（燒錄在 Bootloader）

---

### Secure Boot 實作

**步驟 1：生成金鑰對**

```bash
# 使用 OpenSSL 生成 ECDSA 金鑰對（secp256r1）
openssl ecparam -name prime256v1 -genkey -noout -out private_key.pem
openssl ec -in private_key.pem -pubout -out public_key.pem

# 轉換為 C 陣列
xxd -i public_key.pem > public_key.h
```

**步驟 2：簽署韌體**

```python
# sign_firmware.py
import hashlib
from ecdsa import SigningKey, NIST256p

# 1. 讀取韌體
with open('application.bin', 'rb') as f:
    firmware = f.read()

# 2. 計算 SHA-256
firmware_hash = hashlib.sha256(firmware).digest()

# 3. 使用私鑰簽署
with open('private_key.pem', 'rb') as f:
    private_key = SigningKey.from_pem(f.read())

signature = private_key.sign_digest(firmware_hash, sigencode=lambda r, s, order: r.to_bytes(32, 'big') + s.to_bytes(32, 'big'))

# 4. 附加簽章到韌體
with open('application_signed.bin', 'wb') as f:
    f.write(firmware)
    f.write(signature)

print(f"Firmware signed: {len(firmware)} bytes + {len(signature)} bytes signature")
```

**步驟 3：Bootloader 驗證簽章**

```c
// Bootloader: 驗證韌體簽章
#include "uECC.h"  // micro-ecc library

// 公鑰（燒錄在 Bootloader）
const uint8_t public_key[64] = {
    0x04, 0x1e, 0x3f, 0x20, ...  // X coordinate (32 bytes)
    0x5a, 0x7b, 0x8c, 0x9d, ...  // Y coordinate (32 bytes)
};

bool verify_firmware_signature(uint32_t app_addr, uint32_t app_size) {
    // 1. 計算韌體 SHA-256
    uint8_t hash[32];
    sha256_calculate((uint8_t *)app_addr, app_size, hash);

    // 2. 讀取簽章（附加在韌體後面）
    uint8_t *signature = (uint8_t *)(app_addr + app_size);

    // 3. 驗證簽章
    int result = uECC_verify(public_key, hash, 32, signature, uECC_secp256r1());

    if (result == 1) {
        printf("Signature verification passed\n");
        return true;
    } else {
        printf("Signature verification failed\n");
        return false;
    }
}

void bootloader_main(void) {
    printf("Bootloader started\n");

    // 1. 驗證 Application 簽章
    if (verify_firmware_signature(APP_START_ADDR, APP_SIZE)) {
        // 2. 啟動 Application
        printf("Starting application...\n");
        start_application(APP_START_ADDR);
    } else {
        // 3. 簽章驗證失敗，進入 DFU 模式
        printf("Entering DFU mode...\n");
        dfu_mode_enter();
    }
}
```

**測試結果**：

- ✅ 正確簽章的韌體：可以啟動
- ✅ 錯誤簽章的韌體：無法啟動，進入 DFU 模式
- ✅ 未簽章的韌體：無法啟動，進入 DFU 模式

---

## 裝置層安全：Firmware Encryption

除了 Secure Boot，我們還需要加密韌體，防止韌體被逆向工程。

### Firmware Encryption 原理

**加密方式**：

- 使用 **AES-128-CTR** 加密韌體
- 金鑰：燒錄在 MCU 的 OTP (One-Time Programmable) 區域
- IV (Initialization Vector)：每次加密使用不同的 IV

**解密流程**：

```
┌─────────────────────────────────────────────────────────┐
│  Firmware Encryption 流程                                 │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  1. 開發端：使用 AES 加密韌體                            │
│       ↓                                                   │
│  2. 燒錄加密韌體到 Flash                                 │
│       ↓                                                   │
│  3. Bootloader：從 OTP 讀取金鑰                          │
│       ↓                                                   │
│  4. Bootloader：解密韌體到 RAM                           │
│       ↓                                                   │
│  5. Bootloader：啟動 Application（從 RAM 執行）          │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

### Firmware Encryption 實作

**步驟 1：加密韌體**

```python
# encrypt_firmware.py
from Crypto.Cipher import AES
from Crypto.Random import get_random_bytes

# 1. 讀取韌體
with open('application.bin', 'rb') as f:
    firmware = f.read()

# 2. 生成金鑰和 IV
key = get_random_bytes(16)  # 128-bit key
iv = get_random_bytes(16)   # 128-bit IV

# 3. AES-128-CTR 加密
cipher = AES.new(key, AES.MODE_CTR, nonce=iv[:8])
encrypted_firmware = cipher.encrypt(firmware)

# 4. 儲存加密韌體
with open('application_encrypted.bin', 'wb') as f:
    f.write(iv)  # 前 16 bytes 是 IV
    f.write(encrypted_firmware)

# 5. 儲存金鑰（需要燒錄到 OTP）
with open('encryption_key.bin', 'wb') as f:
    f.write(key)

print(f"Firmware encrypted: {len(firmware)} bytes")
print(f"Key: {key.hex()}")
print(f"IV: {iv.hex()}")
```

**步驟 2：Bootloader 解密韌體**

```c
// Bootloader: 解密韌體
#include "nrf_crypto.h"

// 從 OTP 讀取金鑰
void read_encryption_key(uint8_t *key) {
    // Nordic nRF52: OTP 位於 UICR
    memcpy(key, (uint8_t *)NRF_UICR->CUSTOMER, 16);
}

bool decrypt_firmware(uint32_t encrypted_addr, uint32_t decrypted_addr, uint32_t size) {
    uint8_t key[16];
    uint8_t iv[16];

    // 1. 從 OTP 讀取金鑰
    read_encryption_key(key);

    // 2. 讀取 IV（前 16 bytes）
    memcpy(iv, (uint8_t *)encrypted_addr, 16);

    // 3. AES-128-CTR 解密
    nrf_crypto_aes_context_t aes_ctx;
    nrf_crypto_aes_init(&aes_ctx, NRF_CRYPTO_AES_MODE_CTR, key, iv);

    nrf_crypto_aes_update(&aes_ctx,
                          (uint8_t *)(encrypted_addr + 16),  // 跳過 IV
                          size - 16,
                          (uint8_t *)decrypted_addr);

    nrf_crypto_aes_finalize(&aes_ctx);

    printf("Firmware decrypted: %d bytes\n", size - 16);
    return true;
}

void bootloader_main(void) {
    // 1. 解密韌體到 RAM
    decrypt_firmware(FLASH_APP_ADDR, RAM_APP_ADDR, APP_SIZE);

    // 2. 驗證簽章
    if (verify_firmware_signature(RAM_APP_ADDR, APP_SIZE - 16 - 64)) {
        // 3. 啟動 Application（從 RAM 執行）
        start_application(RAM_APP_ADDR);
    } else {
        printf("Signature verification failed\n");
        dfu_mode_enter();
    }
}
```

---

## 網路層安全：BLE 加密與認證

BLE 本身提供了加密和認證機制，但我們需要正確配置才能確保安全。

### BLE Pairing 與 Bonding

**Pairing**：建立加密連線的過程

**Bonding**：儲存配對資訊，下次連線時不需要重新配對

**Pairing 方式**：

1. **Just Works**：不需要使用者輸入，安全性最低
2. **Passkey Entry**：使用者輸入 6 位數字，安全性中等
3. **Numeric Comparison**：使用者確認兩個裝置顯示的數字是否相同，安全性高
4. **Out of Band (OOB)**：使用 NFC 等其他通道交換金鑰，安全性最高

**我們的選擇**：**Numeric Comparison**（智慧手錶有顯示器，可以顯示數字）

---

### BLE Pairing 實作

```c
// BLE Pairing 設定
void ble_pairing_init(void) {
    ble_gap_sec_params_t sec_params = {
        .bond = 1,                              // 啟用 Bonding
        .mitm = 1,                              // 啟用 MITM 保護
        .lesc = 1,                              // 啟用 LE Secure Connections
        .keypress = 0,
        .io_caps = BLE_GAP_IO_CAPS_DISPLAY_YESNO,  // 支援 Numeric Comparison
        .oob = 0,
        .min_key_size = 16,                     // 最小金鑰長度 128-bit
        .max_key_size = 16,
        .kdist_own = {
            .enc = 1,                           // 分發 LTK
            .id = 1,                            // 分發 IRK
        },
        .kdist_peer = {
            .enc = 1,
            .id = 1,
        },
    };

    pm_sec_params_set(&sec_params);
}

// Pairing 事件處理
void ble_evt_handler(ble_evt_t const *p_ble_evt) {
    switch (p_ble_evt->header.evt_id) {
        case BLE_GAP_EVT_SEC_PARAMS_REQUEST:
            // 1. 收到配對請求
            printf("Pairing request received\n");
            sd_ble_gap_sec_params_reply(p_ble_evt->evt.gap_evt.conn_handle,
                                        BLE_GAP_SEC_STATUS_SUCCESS,
                                        &m_sec_params,
                                        NULL);
            break;

        case BLE_GAP_EVT_LESC_DHKEY_REQUEST:
            // 2. 計算 ECDH 金鑰
            printf("Computing ECDH key...\n");
            compute_ecdh_key(p_ble_evt);
            break;

        case BLE_GAP_EVT_AUTH_KEY_REQUEST:
            // 3. 顯示 Passkey（Numeric Comparison）
            uint32_t passkey = generate_passkey();
            printf("Passkey: %06d\n", passkey);
            display_show_passkey(passkey);

            // 等待使用者確認
            break;

        case BLE_GAP_EVT_AUTH_STATUS:
            // 4. 配對完成
            if (p_ble_evt->evt.gap_evt.params.auth_status.auth_status == BLE_GAP_SEC_STATUS_SUCCESS) {
                printf("Pairing successful\n");

                // 儲存 Bonding 資訊
                pm_peer_id_t peer_id;
                pm_peer_id_get(p_ble_evt->evt.gap_evt.conn_handle, &peer_id);
                printf("Peer ID: %d\n", peer_id);
            } else {
                printf("Pairing failed: %d\n", p_ble_evt->evt.gap_evt.params.auth_status.auth_status);
            }
            break;
    }
}
```

**測試結果**：

- ✅ Pairing 成功：使用 Numeric Comparison
- ✅ 加密連線：使用 AES-128-CCM
- ✅ Bonding：下次連線自動加密，不需要重新配對

---

## 網路層安全：加密 OTA 更新

OTA 更新是 IoT 裝置的重要功能，但也是安全漏洞的來源。我們需要確保 OTA 更新的安全性。

### 加密 OTA 架構

```
┌─────────────────────────────────────────────────────────┐
│  加密 OTA 更新流程                                        │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  1. 雲端：加密韌體（AES-128-CBC）                        │
│       ↓                                                   │
│  2. 雲端：簽署加密韌體（ECDSA）                          │
│       ↓                                                   │
│  3. 手機 App：下載加密韌體                               │
│       ↓                                                   │
│  4. 手機 App：透過 BLE 傳送到裝置                        │
│       ↓                                                   │
│  5. 裝置：驗證簽章                                       │
│       ↓                                                   │
│  6. 裝置：解密韌體                                       │
│       ↓                                                   │
│  7. 裝置：寫入 Flash                                     │
│       ↓                                                   │
│  8. 裝置：重啟，Bootloader 驗證並啟動新韌體             │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

### 加密 OTA 實作

**步驟 1：雲端加密和簽署韌體**

```python
# cloud_ota_prepare.py
from Crypto.Cipher import AES
from Crypto.Random import get_random_bytes
from Crypto.Util.Padding import pad
import hashlib
from ecdsa import SigningKey, NIST256p

# 1. 讀取韌體
with open('application.bin', 'rb') as f:
    firmware = f.read()

# 2. AES-128-CBC 加密
key = get_random_bytes(16)
iv = get_random_bytes(16)
cipher = AES.new(key, AES.MODE_CBC, iv)
encrypted_firmware = cipher.encrypt(pad(firmware, AES.block_size))

# 3. 計算 SHA-256
firmware_hash = hashlib.sha256(encrypted_firmware).digest()

# 4. ECDSA 簽署
with open('private_key.pem', 'rb') as f:
    private_key = SigningKey.from_pem(f.read())
signature = private_key.sign_digest(firmware_hash, sigencode=lambda r, s, order: r.to_bytes(32, 'big') + s.to_bytes(32, 'big'))

# 5. 組合 OTA 封包
ota_packet = {
    'version': '1.1.0',
    'size': len(firmware),
    'encrypted_size': len(encrypted_firmware),
    'key': key.hex(),
    'iv': iv.hex(),
    'signature': signature.hex(),
    'firmware': encrypted_firmware.hex()
}

# 6. 儲存到雲端
import json
with open('ota_packet.json', 'w') as f:
    json.dump(ota_packet, f)

print(f"OTA packet prepared: {len(encrypted_firmware)} bytes")
```

**步驟 2：裝置接收和驗證 OTA**

```c
// 裝置：接收 OTA 更新
typedef struct {
    uint32_t version;
    uint32_t size;
    uint32_t encrypted_size;
    uint8_t key[16];
    uint8_t iv[16];
    uint8_t signature[64];
} ota_header_t;

ota_header_t ota_header;
uint32_t ota_offset = 0;

void ota_update_start(ota_header_t *header) {
    // 1. 儲存 OTA Header
    memcpy(&ota_header, header, sizeof(ota_header_t));

    // 2. 擦除 OTA Flash 區域
    flash_erase(OTA_FLASH_START, ota_header.encrypted_size);

    ota_offset = 0;
    printf("OTA update started: version=%d, size=%d\n",
           ota_header.version, ota_header.size);
}

void ota_update_write(uint8_t *data, uint16_t length) {
    // 寫入加密韌體到 Flash
    flash_write(OTA_FLASH_START + ota_offset, data, length);
    ota_offset += length;

    printf("OTA progress: %d%%\n", (ota_offset * 100) / ota_header.encrypted_size);
}

bool ota_update_verify_and_decrypt(void) {
    // 1. 驗證簽章
    uint8_t hash[32];
    sha256_calculate((uint8_t *)OTA_FLASH_START, ota_header.encrypted_size, hash);

    if (!uECC_verify(public_key, hash, 32, ota_header.signature, uECC_secp256r1())) {
        printf("OTA: Signature verification failed\n");
        return false;
    }

    printf("OTA: Signature verification passed\n");

    // 2. 解密韌體
    nrf_crypto_aes_context_t aes_ctx;
    nrf_crypto_aes_init(&aes_ctx, NRF_CRYPTO_AES_MODE_CBC, ota_header.key, ota_header.iv);

    // 分塊解密（避免 RAM 不足）
    #define DECRYPT_BLOCK_SIZE 256
    uint8_t decrypt_buffer[DECRYPT_BLOCK_SIZE];

    for (uint32_t i = 0; i < ota_header.encrypted_size; i += DECRYPT_BLOCK_SIZE) {
        uint32_t block_size = MIN(DECRYPT_BLOCK_SIZE, ota_header.encrypted_size - i);

        // 解密
        nrf_crypto_aes_update(&aes_ctx,
                              (uint8_t *)(OTA_FLASH_START + i),
                              block_size,
                              decrypt_buffer);

        // 寫回 Flash（覆蓋加密資料）
        flash_write(OTA_FLASH_START + i, decrypt_buffer, block_size);
    }

    nrf_crypto_aes_finalize(&aes_ctx);

    printf("OTA: Firmware decrypted\n");
    return true;
}

void ota_update_complete(void) {
    // 1. 驗證和解密
    if (!ota_update_verify_and_decrypt()) {
        printf("OTA: Update failed\n");
        return;
    }

    // 2. 設定 Bootloader Flag
    bootloader_set_flag(BOOTLOADER_FLAG_OTA_UPDATE);

    // 3. 重啟
    printf("OTA: Update complete. Rebooting...\n");
    vTaskDelay(pdMS_TO_TICKS(1000));
    NVIC_SystemReset();
}
```

---

## 雲端層安全：認證與授權

除了裝置層和網路層的安全，我們還需要確保雲端的安全。

### 雲端架構

```
┌─────────────────────────────────────────────────────────┐
│  IoT 雲端架構                                             │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────┐   HTTPS    ┌──────────┐   HTTPS   ┌─────┐│
│  │  手機    │ ────────→  │  API     │ ────────→ │ DB  ││
│  │  App     │            │  Gateway │           │     ││
│  └──────────┘            └──────────┘           └─────┘│
│                                 │                        │
│                                 │ JWT Token              │
│                                 ↓                        │
│                          ┌──────────┐                    │
│                          │  Auth    │                    │
│                          │  Service │                    │
│                          └──────────┘                    │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

### JWT (JSON Web Token) 認證

**JWT** 是一種無狀態的認證機制，適合 IoT 應用。

**JWT 結構**：

```
Header.Payload.Signature

Header: {"alg": "HS256", "typ": "JWT"}
Payload: {"user_id": "12345", "device_id": "ABCDE", "exp": 1234567890}
Signature: HMACSHA256(base64(Header) + "." + base64(Payload), secret_key)
```

**JWT 流程**：

```
┌─────────────────────────────────────────────────────────┐
│  JWT 認證流程                                             │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  1. 使用者登入 → 雲端驗證帳號密碼                        │
│       ↓                                                   │
│  2. 雲端生成 JWT Token                                   │
│       ↓                                                   │
│  3. 手機 App 儲存 JWT Token                              │
│       ↓                                                   │
│  4. 手機 App 發送 API 請求（附帶 JWT Token）             │
│       ↓                                                   │
│  5. 雲端驗證 JWT Token                                   │
│       ├─ Valid → 處理請求                                │
│       └─ Invalid → 拒絕請求（401 Unauthorized）          │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

### JWT 實作（雲端）

```python
# cloud_auth.py
import jwt
import datetime

SECRET_KEY = "your-secret-key-here"  # 應該從環境變數讀取

def generate_jwt_token(user_id, device_id):
    """生成 JWT Token"""
    payload = {
        'user_id': user_id,
        'device_id': device_id,
        'exp': datetime.datetime.utcnow() + datetime.timedelta(days=7),  # 7 天過期
        'iat': datetime.datetime.utcnow()
    }

    token = jwt.encode(payload, SECRET_KEY, algorithm='HS256')
    return token

def verify_jwt_token(token):
    """驗證 JWT Token"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
        return payload
    except jwt.ExpiredSignatureError:
        print("Token expired")
        return None
    except jwt.InvalidTokenError:
        print("Invalid token")
        return None

# API Gateway
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/api/login', methods=['POST'])
def login():
    """使用者登入"""
    data = request.json
    username = data.get('username')
    password = data.get('password')

    # 驗證帳號密碼（應該從資料庫查詢）
    if username == 'user@example.com' and password == 'password123':
        # 生成 JWT Token
        token = generate_jwt_token(user_id='12345', device_id='ABCDE')
        return jsonify({'token': token})
    else:
        return jsonify({'error': 'Invalid credentials'}), 401

@app.route('/api/health_data', methods=['GET'])
def get_health_data():
    """取得健康資料（需要認證）"""
    # 從 Header 取得 JWT Token
    auth_header = request.headers.get('Authorization')
    if not auth_header or not auth_header.startswith('Bearer '):
        return jsonify({'error': 'Missing token'}), 401

    token = auth_header.split(' ')[1]

    # 驗證 JWT Token
    payload = verify_jwt_token(token)
    if not payload:
        return jsonify({'error': 'Invalid token'}), 401

    # 從資料庫取得健康資料
    user_id = payload['user_id']
    health_data = get_health_data_from_db(user_id)

    return jsonify(health_data)

@app.route('/api/upload_health_data', methods=['POST'])
def upload_health_data():
    """上傳健康資料（需要認證）"""
    # 驗證 JWT Token
    auth_header = request.headers.get('Authorization')
    if not auth_header or not auth_header.startswith('Bearer '):
        return jsonify({'error': 'Missing token'}), 401

    token = auth_header.split(' ')[1]
    payload = verify_jwt_token(token)
    if not payload:
        return jsonify({'error': 'Invalid token'}), 401

    # 儲存健康資料到資料庫
    user_id = payload['user_id']
    data = request.json
    save_health_data_to_db(user_id, data)

    return jsonify({'status': 'success'})
```

---

## 真實案例：安全漏洞與防護

在這次安全實戰中，我們發現並修復了幾個安全漏洞：

### 漏洞 1：BLE Pairing 使用 Just Works

**問題**：

- 原本的 BLE Pairing 使用 **Just Works** 模式
- 攻擊者可以輕易連線到裝置，竊取健康資料

**攻擊場景**：

```
1. 攻擊者使用 BLE Sniffer 掃描附近的裝置
2. 發現智慧手錶的 BLE 廣播
3. 發送連線請求
4. 因為使用 Just Works，配對成功（不需要使用者確認）
5. 讀取 GATT Characteristic，竊取健康資料
```

**解決方案**：

- 改用 **Numeric Comparison** 模式
- 使用者必須確認配對碼才能連線

**程式碼修改**：

```c
// 修改前：Just Works
ble_gap_sec_params_t sec_params = {
    .io_caps = BLE_GAP_IO_CAPS_NONE,  // Just Works
    .mitm = 0,                         // 不需要 MITM 保護
};

// 修改後：Numeric Comparison
ble_gap_sec_params_t sec_params = {
    .io_caps = BLE_GAP_IO_CAPS_DISPLAY_YESNO,  // Numeric Comparison
    .mitm = 1,                                  // 需要 MITM 保護
    .lesc = 1,                                  // 使用 LE Secure Connections
};
```

---

### 漏洞 2：OTA 更新沒有簽章驗證

**問題**：

- 原本的 OTA 更新沒有驗證簽章
- 攻擊者可以透過 BLE 發送惡意韌體

**攻擊場景**：

```
1. 攻擊者連線到智慧手錶
2. 發送 OTA 更新請求
3. 傳送惡意韌體
4. 裝置接受並安裝惡意韌體
5. 惡意韌體竊取使用者資料並上傳到攻擊者的伺服器
```

**解決方案**：

- 加入 **ECDSA 簽章驗證**
- 只有經過簽名的韌體才能安裝

**程式碼修改**：

```c
// 修改前：沒有簽章驗證
void ota_update_complete(void) {
    // 直接安裝
    bootloader_set_flag(BOOTLOADER_FLAG_OTA_UPDATE);
    NVIC_SystemReset();
}

// 修改後：加入簽章驗證
void ota_update_complete(void) {
    // 1. 驗證簽章
    if (!verify_firmware_signature(OTA_FLASH_START, ota_header.size)) {
        printf("OTA: Signature verification failed\n");
        return;
    }

    // 2. 安裝
    bootloader_set_flag(BOOTLOADER_FLAG_OTA_UPDATE);
    NVIC_SystemReset();
}
```

---

### 漏洞 3：除錯接口未關閉

**問題**：

- 量產版本的 SWD 和 UART 除錯接口沒有關閉
- 攻擊者可以透過 SWD 讀取 Flash 和 RAM

**攻擊場景**：

```
1. 攻擊者拆開智慧手錶
2. 找到 SWD 接口（SWDIO, SWCLK）
3. 使用 J-Link 連線
4. 讀取 Flash 和 RAM
5. 取得韌體和金鑰
```

**解決方案**：

- 關閉 SWD 和 UART 除錯接口
- 啟用 **APPROTECT**（Access Port Protection）

**程式碼修改**：

```c
// 關閉除錯接口
void disable_debug_interface(void) {
    // 1. 關閉 UART
    NRF_UART0->ENABLE = 0;

    // 2. 啟用 APPROTECT（防止 SWD 存取）
    if (NRF_UICR->APPROTECT == 0xFFFFFFFF) {
        // 寫入 UICR（需要擦除整個 UICR）
        NRF_NVMC->CONFIG = NVMC_CONFIG_WEN_Wen;
        NRF_UICR->APPROTECT = 0x00000000;  // 啟用 APPROTECT
        NRF_NVMC->CONFIG = NVMC_CONFIG_WEN_Ren;

        // 重啟生效
        NVIC_SystemReset();
    }
}
```

**注意**：啟用 APPROTECT 後，SWD 將無法存取，只能透過 OTA 更新韌體。

---

## 實務建議

基於這次 IoT 安全實戰的經驗，我總結了以下的實務建議：

### 安全檢查清單

**裝置層安全**：

- [ ] 實作 Secure Boot（驗證韌體簽章）
- [ ] 加密韌體（防止逆向工程）
- [ ] 關閉除錯接口（SWD, UART）
- [ ] 啟用 APPROTECT（防止 SWD 存取）
- [ ] 使用 OTP 儲存金鑰（防止金鑰被讀取）

**網路層安全**：

- [ ] 使用 BLE Pairing（Numeric Comparison 或 OOB）
- [ ] 啟用 BLE 加密（AES-128-CCM）
- [ ] 實作 Bonding（儲存配對資訊）
- [ ] 驗證 OTA 更新簽章
- [ ] 加密 OTA 更新

**雲端層安全**：

- [ ] 使用 HTTPS（TLS 1.2+）
- [ ] 實作 JWT 認證
- [ ] 實作 API Rate Limiting（防止 DDoS）
- [ ] 加密資料庫（防止資料洩漏）
- [ ] 定期備份資料

---

### 安全開發流程

**設計階段**：

1. **威脅建模**：分析可能的安全威脅
2. **安全需求**：定義安全需求（機密性、完整性、可用性）
3. **安全架構**：設計端到端的安全方案

**開發階段**：

1. **安全編碼**：遵循安全編碼規範
2. **程式碼審查**：檢查安全漏洞
3. **靜態分析**：使用工具掃描安全漏洞

**測試階段**：

1. **滲透測試**：模擬攻擊者的攻擊
2. **模糊測試**：測試異常輸入
3. **安全審查**：第三方安全審查

**部署階段**：

1. **安全配置**：關閉除錯接口、啟用 APPROTECT
2. **金鑰管理**：安全地儲存和分發金鑰
3. **監控**：監控異常行為

---

## 總結

這次 IoT 安全實戰，讓我深刻體會到：**安全不是事後補救，而是從一開始就要考慮的**。

我們在 2 週內實作了完整的端到端安全方案：

- **裝置層**：Secure Boot, Firmware Encryption, APPROTECT
- **網路層**：BLE Pairing (Numeric Comparison), 加密 OTA
- **雲端層**：JWT 認證, HTTPS, API Rate Limiting

這些安全機制，成功通過了客戶的安全審查，保住了 100 萬美元的訂單。

**我的建議**：

1. **從設計階段就考慮安全**：不要等到產品完成才加入安全功能
2. **使用業界標準**：不要自己發明加密演算法
3. **多層防護**：不要只依賴單一安全機制
4. **定期更新**：透過 OTA 修復安全漏洞
5. **安全審查**：請第三方進行安全審查

IoT 安全是一個持續的過程，不是一次性的工作。只有持續關注安全，才能保護使用者的資料和隱私。

---

## 參考資料

- NIST Cybersecurity Framework: <https://www.nist.gov/cyberframework>
- OWASP IoT Security: <https://owasp.org/www-project-internet-of-things/>
- Bluetooth Security: <https://www.bluetooth.com/learn-about-bluetooth/key-attributes/bluetooth-security/>
- ARM TrustZone: <https://www.arm.com/technologies/trustzone-for-cortex-m>
- Nordic nRF52 Security: <https://infocenter.nordicsemi.com/topic/com.nordic.infocenter.nrf52/dita/nrf52/security/security.html>
- JWT: <https://jwt.io/>
- ECDSA: <https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

您可以自由地：

- **分享** — 以任何媒介或格式重製及散布本素材
- **修改** — 重混、轉換本素材、及依本素材建立新素材

但您必須遵守以下條款：

- **姓名標示** — 您必須給予適當表彰、提供指向本授權條款的連結，以及指出（本作品的原始版本）是否已被變更。您可以任何合理的方式為前述表彰，但不得以任何方式暗示授權人為您或您的使用方式背書。
