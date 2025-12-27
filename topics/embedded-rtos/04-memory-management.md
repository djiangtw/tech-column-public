# RTOS 記憶體管理：5 種 Heap 方案的選擇與權衡

**作者**: Danny Jiang
**日期**: 2025-12-09

---

## 前言：一個讓人抓狂的記憶體洩漏

2018 年，我在一家物聯網設備公司擔任韌體工程師。有一天，客服部門轉來一個嚴重的客訴：

**客戶的智慧家居閘道在運行 72 小時後會隨機崩潰，錯誤訊息是 "Out of Memory"。**

我們的產品是一個智慧家居閘道，運行 FreeRTOS，需要同時處理 10 個感測器的資料採集（每個 100 ms 一次）、MQTT 網路通訊、資料統計計算，以及 LCD 顯示更新。總共 13 個 Tasks 在系統中運行。

我接手這個案子後，第一件事就是檢查記憶體使用情況。我們使用 FreeRTOS 的 `heap_3`（直接呼叫 `malloc/free`），並在程式碼中大量使用動態記憶體分配：

```c
// 感測器資料處理
void vTaskSensorProcess(void *pvParameters) {
    while (1) {
        // 動態分配緩衝區
        sensor_data_t *data = pvPortMalloc(sizeof(sensor_data_t));
        
        // 讀取感測器
        read_sensor(data);
        
        // 處理資料
        process_data(data);
        
        // 釋放記憶體
        vPortFree(data);
        
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

**看起來沒問題，對吧？** 每次都有 `vPortFree()`，應該不會洩漏記憶體。

### 真正的問題

使用 FreeRTOS 的記憶體分析工具，我發現：

```
After 72 hours:
  Total heap size: 128 KB
  Used memory: 96 KB
  Free memory: 32 KB
  Largest free block: 512 bytes  ← 問題在這裡！
```

**記憶體碎片化 (Fragmentation)！**

雖然總共有 32 KB 的空閒記憶體，但最大的連續空閒區塊只有 512 bytes。當我們需要分配 1 KB 的緩衝區時，就會失敗。

### 解決方案

最後，我們改用 **heap_4**（First-Fit with Coalescing），並重新設計記憶體分配策略：

1. **靜態分配長期存在的物件**（如網路緩衝區）
2. **使用記憶體池 (Memory Pool)** 給固定大小的物件（如感測器資料）
3. **只在必要時使用動態分配**

結果：系統穩定運行超過 3 個月，沒有任何記憶體問題。

---

## 一、FreeRTOS 的 5 種 Heap 方案

FreeRTOS 提供了 5 種不同的 Heap 實作，每種都有不同的特性和適用場景。

### 1.1 為什麼需要不同的 Heap 方案？

**一個 Heap 方案無法滿足所有需求**：

- **簡單系統**：只在啟動時分配記憶體，之後不再釋放
- **複雜系統**：頻繁分配和釋放，需要處理碎片化
- **多記憶體區域**：有些 MCU 有多個不連續的 RAM 區域
- **即時性要求**：需要確定性的分配時間

**FreeRTOS 的解決方案**：提供 5 種 Heap 實作，讓開發者根據需求選擇。

### 1.2 5 種 Heap 方案總覽

| Heap | 名稱 | 特性 | 適用場景 |
|------|------|------|---------|
| **heap_1** | Bump Allocator | 只能分配，不能釋放 | 簡單系統，只在啟動時分配 |
| **heap_2** | Best-Fit | 可以釋放，但不合併 | 固定大小的物件，較少碎片化 |
| **heap_3** | malloc/free | 直接呼叫標準庫 | 需要與標準庫整合 |
| **heap_4** | First-Fit with Coalescing | 可以釋放並合併 | 一般用途，最常用 |
| **heap_5** | Multiple Regions | 支援多個記憶體區域 | 多個不連續的 RAM 區域 |

---

## 二、heap_1：Bump Allocator

### 2.1 原理

**Bump Allocator** 是最簡單的記憶體分配器，只需要一個指標指向下一個可用的記憶體位址。

**實作**：

```c
static uint8_t heap[configTOTAL_HEAP_SIZE];
static size_t next_free_byte = 0;

void *pvPortMalloc(size_t size) {
    void *ptr = NULL;
    
    // 對齊到 8 bytes
    size = (size + 7) & ~7;
    
    if (next_free_byte + size < configTOTAL_HEAP_SIZE) {
        ptr = &heap[next_free_byte];
        next_free_byte += size;
    }
    
    return ptr;
}

void vPortFree(void *ptr) {
    // No-op: 不能釋放個別區塊
}
```

### 2.2 特性

**優點**：

- **極快**：只需要增加指標（~5 cycles）
- **零碎片化**：分配是連續的
- **簡單**：只有 10 行程式碼
- **確定性**：O(1) 時間複雜度

**缺點**：

- **不能釋放個別區塊**：只能重置整個 Heap
- **記憶體浪費**：一旦分配就無法回收

### 2.3 適用場景

**場景 #1：簡單的嵌入式系統**

- 只在啟動時建立 Tasks 和 Queues
- 之後不再分配記憶體

**場景 #2：分階段的系統**

- 例如：Bootloader
- 階段 1：解析 Device Tree（分配記憶體）
- 階段 2：載入 Kernel（重置 Heap，重新分配）

**範例**：

```c
void main(void) {
    // 階段 1：建立 Tasks
    xTaskCreate(vTaskSensor, "Sensor", 256, NULL, 1, NULL);
    xTaskCreate(vTaskNetwork, "Network", 512, NULL, 2, NULL);
    
    // 階段 2：建立 Queues
    sensor_queue = xQueueCreate(10, sizeof(sensor_data_t));
    
    // 啟動 Scheduler（之後不再分配記憶體）
    vTaskStartScheduler();
}
```

---

## 三、heap_2：Best-Fit without Coalescing

### 3.1 原理

**Best-Fit** 演算法會找到「最適合」的空閒區塊（大小最接近需求的區塊）。

**實作**：

```c
typedef struct A_BLOCK_LINK {
    struct A_BLOCK_LINK *pxNextFreeBlock;
    size_t xBlockSize;
} BlockLink_t;

void *pvPortMalloc(size_t xWantedSize) {
    BlockLink_t *pxBlock, *pxPreviousBlock, *pxBestBlock = NULL;
    size_t xBestSize = SIZE_MAX;
    
    // 找到最適合的區塊
    pxPreviousBlock = &xStart;
    pxBlock = xStart.pxNextFreeBlock;
    
    while (pxBlock != pxEnd) {
        if (pxBlock->xBlockSize >= xWantedSize) {
            if (pxBlock->xBlockSize < xBestSize) {
                xBestSize = pxBlock->xBlockSize;
                pxBestBlock = pxBlock;
            }
        }
        pxPreviousBlock = pxBlock;
        pxBlock = pxBlock->pxNextFreeBlock;
    }
    
    // 分割區塊並返回
    // ...
}
```

### 3.2 特性

**優點**：

- **可以釋放記憶體**：支援 `vPortFree()`
- **減少碎片化**：Best-Fit 策略減少浪費
- **確定性**：時間複雜度固定（雖然較慢）

**缺點**：

- **不合併空閒區塊**：釋放的區塊不會與相鄰區塊合併
- **碎片化問題**：長時間運行後會產生大量小區塊
- **較慢**：需要遍歷整個空閒鏈表（O(n)）

### 3.3 適用場景

**場景：固定大小的物件**

如果你的系統只分配幾種固定大小的物件（例如：256 bytes 的感測器資料、512 bytes 的網路封包），heap_2 可以有效避免碎片化。

**為什麼？**

因為釋放的 256 bytes 區塊可以被下一個 256 bytes 的分配重用，不會產生碎片。

**範例**：

```c
// 系統只分配兩種大小的物件
#define SENSOR_DATA_SIZE  256
#define NETWORK_PACKET_SIZE  512

void vTaskSensor(void *pvParameters) {
    while (1) {
        // 總是分配 256 bytes
        sensor_data_t *data = pvPortMalloc(SENSOR_DATA_SIZE);
        read_sensor(data);
        process_data(data);
        vPortFree(data);  // 釋放後可以被下一個 256 bytes 重用

        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

**注意**：heap_2 在 FreeRTOS V9.0.0 之後已經**不推薦使用**，建議改用 heap_4。

---

## 四、heap_3：Wrapper for malloc/free

### 4.1 原理

**heap_3** 不是一個真正的記憶體分配器，它只是對標準庫的 `malloc/free` 的包裝。

**實作**：

```c
void *pvPortMalloc(size_t xWantedSize) {
    void *pvReturn;

    vTaskSuspendAll();  // 暫停 Scheduler
    {
        pvReturn = malloc(xWantedSize);
    }
    (void) xTaskResumeAll();  // 恢復 Scheduler

    return pvReturn;
}

void vPortFree(void *pv) {
    if (pv != NULL) {
        vTaskSuspendAll();
        {
            free(pv);
        }
        (void) xTaskResumeAll();
    }
}
```

### 4.2 特性

**優點**：

- **與標準庫整合**：可以與使用 `malloc/free` 的第三方庫整合
- **成熟的實作**：標準庫的 `malloc/free` 經過充分測試
- **支援複雜的分配模式**：標準庫通常使用更複雜的演算法

**缺點**：

- **需要標準庫支援**：嵌入式系統可能沒有完整的標準庫
- **較大的程式碼體積**：標準庫的 `malloc/free` 通常較大（~10 KB）
- **不確定性**：標準庫的 `malloc/free` 時間複雜度不固定
- **需要額外的 Heap 配置**：需要在 Linker Script 中配置 Heap

### 4.3 適用場景

**場景 #1：與第三方庫整合**

如果你使用的第三方庫（例如：JSON 解析器、加密庫）內部使用 `malloc/free`，使用 heap_3 可以確保所有記憶體分配都來自同一個 Heap。

**場景 #2：移植現有程式碼**

如果你正在將現有的程式碼移植到 FreeRTOS，而程式碼中大量使用 `malloc/free`，使用 heap_3 可以減少修改。

**範例**：

```c
// 第三方 JSON 庫內部使用 malloc
#include "cJSON.h"

void vTaskParseJSON(void *pvParameters) {
    while (1) {
        char *json_string = receive_json();

        // cJSON_Parse 內部會呼叫 malloc
        cJSON *json = cJSON_Parse(json_string);

        // 處理 JSON
        process_json(json);

        // cJSON_Delete 內部會呼叫 free
        cJSON_Delete(json);

        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

**注意**：heap_3 需要在 Linker Script 中配置 Heap：

```ld
/* Linker Script */
MEMORY {
    RAM (xrw) : ORIGIN = 0x20000000, LENGTH = 128K
}

SECTIONS {
    .heap : {
        . = ALIGN(8);
        __heap_start = .;
        . = . + 64K;  /* 64 KB Heap */
        __heap_end = .;
    } > RAM
}
```

---

## 五、heap_4：First-Fit with Coalescing（最常用）

### 5.1 原理

**heap_4** 是 FreeRTOS 最常用的 Heap 方案，它使用 **First-Fit** 演算法，並支援 **Coalescing（合併）** 空閒區塊。

**First-Fit**：找到第一個足夠大的空閒區塊

**Coalescing**：當釋放記憶體時，如果相鄰的區塊也是空閒的，就合併成一個大區塊

**實作**：

```c
typedef struct A_BLOCK_LINK {
    struct A_BLOCK_LINK *pxNextFreeBlock;
    size_t xBlockSize;
} BlockLink_t;

static BlockLink_t xStart;
static BlockLink_t *pxEnd = NULL;

void *pvPortMalloc(size_t xWantedSize) {
    BlockLink_t *pxBlock, *pxPreviousBlock, *pxNewBlockLink;

    // 1. 找到第一個足夠大的區塊
    pxPreviousBlock = &xStart;
    pxBlock = xStart.pxNextFreeBlock;

    while ((pxBlock->xBlockSize < xWantedSize) &&
           (pxBlock->pxNextFreeBlock != NULL)) {
        pxPreviousBlock = pxBlock;
        pxBlock = pxBlock->pxNextFreeBlock;
    }

    if (pxBlock != pxEnd) {
        // 2. 分割區塊（如果區塊夠大）
        if (pxBlock->xBlockSize > xWantedSize + heapMINIMUM_BLOCK_SIZE) {
            pxNewBlockLink = (BlockLink_t *)(((uint8_t *)pxBlock) + xWantedSize);
            pxNewBlockLink->xBlockSize = pxBlock->xBlockSize - xWantedSize;
            pxNewBlockLink->pxNextFreeBlock = pxBlock->pxNextFreeBlock;

            pxBlock->xBlockSize = xWantedSize;
            pxPreviousBlock->pxNextFreeBlock = pxNewBlockLink;
        } else {
            // 區塊不夠大，不分割
            pxPreviousBlock->pxNextFreeBlock = pxBlock->pxNextFreeBlock;
        }

        return (void *)(((uint8_t *)pxBlock) + xHeapStructSize);
    }

    return NULL;
}

void vPortFree(void *pv) {
    uint8_t *puc = (uint8_t *)pv;
    BlockLink_t *pxLink;

    puc -= xHeapStructSize;
    pxLink = (BlockLink_t *)puc;

    // 插入到空閒鏈表中（按地址排序）
    BlockLink_t *pxIterator;
    for (pxIterator = &xStart;
         pxIterator->pxNextFreeBlock < pxLink;
         pxIterator = pxIterator->pxNextFreeBlock) {
        // 找到插入位置
    }

    // 合併相鄰的空閒區塊
    uint8_t *pucEnd = puc + pxLink->xBlockSize;
    if (pucEnd == (uint8_t *)pxIterator->pxNextFreeBlock) {
        // 與下一個區塊合併
        pxLink->xBlockSize += pxIterator->pxNextFreeBlock->xBlockSize;
        pxLink->pxNextFreeBlock = pxIterator->pxNextFreeBlock->pxNextFreeBlock;
    } else {
        pxLink->pxNextFreeBlock = pxIterator->pxNextFreeBlock;
    }

    if ((uint8_t *)pxIterator + pxIterator->xBlockSize == puc) {
        // 與前一個區塊合併
        pxIterator->xBlockSize += pxLink->xBlockSize;
        pxIterator->pxNextFreeBlock = pxLink->pxNextFreeBlock;
    } else {
        pxIterator->pxNextFreeBlock = pxLink;
    }
}
```

### 5.2 特性

**優點**：

- **支援合併**：釋放記憶體時會合併相鄰的空閒區塊
- **減少碎片化**：合併機制大幅減少碎片化
- **一般用途**：適合大多數應用場景
- **確定性**：時間複雜度固定（O(n)，n 是空閒區塊數量）

**缺點**：

- **較慢**：需要遍歷空閒鏈表並合併區塊
- **仍有碎片化風險**：如果分配模式不規則，仍可能產生碎片

### 5.3 適用場景

**場景：一般用途的嵌入式系統**

heap_4 是 **FreeRTOS 最推薦的 Heap 方案**，適合大多數應用場景：

- 需要頻繁分配和釋放記憶體
- 分配大小不固定
- 需要長時間穩定運行

**範例：我們的 QEMU Demo**

在我們的 FreeRTOS + QEMU + RISC-V Demo 中，我們使用 heap_4：

```c
// FreeRTOSConfig.h
#define configTOTAL_HEAP_SIZE  ((size_t)(64 * 1024))  // 64 KB

// main.c
void main(void) {
    // 建立 Tasks（動態分配 Task Stack）
    xTaskCreate(vTaskBlink, "Blink", 256, NULL, 1, NULL);
    xTaskCreate(vTaskPrint, "Print", 512, NULL, 2, NULL);

    // 建立 Queues（動態分配 Queue 緩衝區）
    print_queue = xQueueCreate(10, sizeof(char *));

    // 啟動 Scheduler
    vTaskStartScheduler();
}
```

**為什麼選擇 heap_4？**

1. **支援動態建立 Tasks**：我們可能在運行時建立和刪除 Tasks
2. **支援動態建立 Queues**：我們可能在運行時建立和刪除 Queues
3. **合併機制**：確保長時間運行不會產生碎片化

### 5.4 記憶體佈局

**heap_4 的記憶體佈局**：

```
Heap 開始
┌─────────────────────────────────────────────────────────┐
│ BlockLink_t (8 bytes)                                   │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ pxNextFreeBlock: 指向下一個空閒區塊                  │ │
│ │ xBlockSize: 區塊大小                                 │ │
│ └─────────────────────────────────────────────────────┘ │
│ User Data (xBlockSize - 8 bytes)                        │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ 使用者資料                                           │ │
│ │ ...                                                  │ │
│ └─────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────┤
│ BlockLink_t (8 bytes)                                   │
│ ...                                                     │
└─────────────────────────────────────────────────────────┘
Heap 結束
```

**Overhead**：每個區塊需要 8 bytes 的 metadata（`BlockLink_t`）

---

## 六、heap_5：Multiple Regions

### 6.1 原理

**heap_5** 支援多個不連續的記憶體區域，這在某些 MCU 上非常有用。

**為什麼需要多個記憶體區域？**

某些 MCU 有多個不連續的 RAM 區域：

- **內部 SRAM**：速度快，但容量小（例如：64 KB）
- **外部 SDRAM**：容量大，但速度慢（例如：8 MB）
- **CCM (Core-Coupled Memory)**：速度極快，但只能被 CPU 存取（例如：64 KB）

**heap_5 可以將這些區域統一管理**。

**實作**：

```c
typedef struct HeapRegion {
    uint8_t *pucStartAddress;
    size_t xSizeInBytes;
} HeapRegion_t;

void vPortDefineHeapRegions(const HeapRegion_t * const pxHeapRegions) {
    BlockLink_t *pxFirstFreeBlockInRegion = NULL, *pxPreviousFreeBlock;
    size_t xAlignedHeap;
    size_t xTotalRegionSize, xTotalHeapSize = 0;
    BaseType_t xDefinedRegions = 0;
    size_t xAddress;
    const HeapRegion_t *pxHeapRegion;

    // 遍歷所有區域
    pxHeapRegion = &(pxHeapRegions[xDefinedRegions]);
    while (pxHeapRegion->xSizeInBytes > 0) {
        xTotalRegionSize = pxHeapRegion->xSizeInBytes;

        // 對齊到 8 bytes
        xAddress = (size_t)pxHeapRegion->pucStartAddress;
        if ((xAddress & portBYTE_ALIGNMENT_MASK) != 0) {
            xAddress += (portBYTE_ALIGNMENT - 1);
            xAddress &= ~portBYTE_ALIGNMENT_MASK;
            xTotalRegionSize -= xAddress - (size_t)pxHeapRegion->pucStartAddress;
        }

        // 建立空閒區塊
        pxFirstFreeBlockInRegion = (BlockLink_t *)xAddress;
        pxFirstFreeBlockInRegion->xBlockSize = xTotalRegionSize;
        pxFirstFreeBlockInRegion->pxNextFreeBlock = NULL;

        // 連結到空閒鏈表
        if (xDefinedRegions == 0) {
            xStart.pxNextFreeBlock = pxFirstFreeBlockInRegion;
        } else {
            pxPreviousFreeBlock->pxNextFreeBlock = pxFirstFreeBlockInRegion;
        }

        pxPreviousFreeBlock = pxFirstFreeBlockInRegion;
        xTotalHeapSize += pxFirstFreeBlockInRegion->xBlockSize;

        xDefinedRegions++;
        pxHeapRegion = &(pxHeapRegions[xDefinedRegions]);
    }

    // 結束標記
    pxEnd = pxPreviousFreeBlock;
    pxEnd->xBlockSize = 0;
    pxEnd->pxNextFreeBlock = NULL;
}
```

### 6.2 特性

**優點**：

- **支援多個記憶體區域**：可以統一管理不連續的 RAM
- **靈活配置**：可以根據需求配置不同的區域
- **與 heap_4 相同的演算法**：支援合併，減少碎片化

**缺點**：

- **需要手動配置**：需要在程式碼中定義記憶體區域
- **較複雜**：配置錯誤可能導致系統崩潰

### 6.3 適用場景

**場景：多個 RAM 區域的 MCU**

**範例：STM32F7 系列**

STM32F7 有多個 RAM 區域：

- **DTCM RAM**：128 KB，速度極快（0 wait state）
- **SRAM1**：368 KB，速度快
- **SRAM2**：16 KB，速度快
- **外部 SDRAM**：8 MB，速度慢

**配置**：

```c
// 定義記憶體區域（必須按地址排序）
const HeapRegion_t xHeapRegions[] = {
    // DTCM RAM: 用於高頻率分配的小物件
    { (uint8_t *)0x20000000UL, 64 * 1024 },

    // SRAM1: 用於一般用途
    { (uint8_t *)0x20020000UL, 256 * 1024 },

    // 外部 SDRAM: 用於大緩衝區
    { (uint8_t *)0xC0000000UL, 4 * 1024 * 1024 },

    // 結束標記
    { NULL, 0 }
};

void main(void) {
    // 初始化 Heap
    vPortDefineHeapRegions(xHeapRegions);

    // 建立 Tasks
    xTaskCreate(vTaskBlink, "Blink", 256, NULL, 1, NULL);

    // 啟動 Scheduler
    vTaskStartScheduler();
}
```

**策略**：

- **小物件**（< 1 KB）：優先從 DTCM RAM 分配（速度快）
- **中等物件**（1-64 KB）：從 SRAM1 分配
- **大緩衝區**（> 64 KB）：從外部 SDRAM 分配

**注意**：heap_5 會自動從第一個有足夠空間的區域分配，無法手動指定區域。如果需要更精細的控制，建議使用多個獨立的 Heap。

---

## 七、如何選擇 Heap 方案？

### 7.1 決策樹

```
開始
  │
  ├─ 是否需要釋放記憶體？
  │   ├─ 否 → heap_1（最簡單、最快）
  │   └─ 是 ↓
  │
  ├─ 是否有多個不連續的 RAM 區域？
  │   ├─ 是 → heap_5（支援多區域）
  │   └─ 否 ↓
  │
  ├─ 是否需要與標準庫整合？
  │   ├─ 是 → heap_3（使用 malloc/free）
  │   └─ 否 ↓
  │
  └─ 一般用途 → heap_4（最推薦）
```

### 7.2 比較表

| 特性 | heap_1 | heap_2 | heap_3 | heap_4 | heap_5 |
|------|--------|--------|--------|--------|--------|
| **可以釋放** | ❌ | ✅ | ✅ | ✅ | ✅ |
| **合併空閒區塊** | N/A | ❌ | ✅ | ✅ | ✅ |
| **碎片化風險** | 無 | 高 | 低 | 低 | 低 |
| **時間複雜度** | O(1) | O(n) | 不定 | O(n) | O(n) |
| **程式碼大小** | 極小 | 小 | 大 | 中 | 中 |
| **多記憶體區域** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **推薦度** | ⭐⭐⭐ | ⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

### 7.3 實務建議

**建議 #1：優先使用 heap_4**

除非有特殊需求，否則使用 heap_4。它是 FreeRTOS 最推薦的方案，適合大多數應用場景。

**建議 #2：避免使用 heap_2**

heap_2 在 FreeRTOS V9.0.0 之後已經不推薦使用，建議改用 heap_4。

**建議 #3：只在必要時使用 heap_3**

heap_3 會增加程式碼體積（~10 KB），只在需要與標準庫整合時使用。

**建議 #4：考慮靜態分配**

如果可能，優先使用靜態分配（`xTaskCreateStatic`、`xQueueCreateStatic`），可以完全避免動態記憶體分配。

**建議 #5：使用記憶體池**

對於固定大小的物件（如感測器資料、網路封包），使用記憶體池可以避免碎片化並提升效能。

---

## 八、記憶體碎片化問題

### 8.1 什麼是碎片化？

**碎片化 (Fragmentation)**：雖然總共有足夠的空閒記憶體，但沒有足夠大的連續空閒區塊。

**範例**：

```
初始狀態（64 KB Heap）：
┌────────────────────────────────────────────────────────┐
│ Free: 64 KB                                            │
└────────────────────────────────────────────────────────┘

分配 3 個 16 KB 區塊：
┌──────────┬──────────┬──────────┬──────────────────────┐
│ Used: 16 │ Used: 16 │ Used: 16 │ Free: 16 KB          │
└──────────┴──────────┴──────────┴──────────────────────┘

釋放第 1 和第 3 個區塊：
┌──────────┬──────────┬──────────┬──────────────────────┐
│ Free: 16 │ Used: 16 │ Free: 16 │ Free: 16 KB          │
└──────────┴──────────┴──────────┴──────────────────────┘

問題：總共有 48 KB 空閒記憶體，但最大連續區塊只有 16 KB！
如果需要分配 32 KB，會失敗。
```

### 8.2 碎片化的原因

**原因 #1：不規則的分配模式**

如果分配和釋放的順序不規則，會產生碎片化。

**原因 #2：不同大小的物件混合分配**

如果同時分配 256 bytes、1 KB、4 KB 的物件，會產生碎片化。

**原因 #3：長期運行**

系統運行時間越長，碎片化越嚴重。

### 8.3 如何避免碎片化？

**方法 #1：使用 heap_4 或 heap_5（支援合併）**

heap_4 和 heap_5 會在釋放記憶體時合併相鄰的空閒區塊，大幅減少碎片化。

**方法 #2：使用記憶體池**

對於固定大小的物件，使用記憶體池可以完全避免碎片化。

**範例**：

```c
// 記憶體池：256 bytes x 32 個
typedef struct {
    uint8_t data[256];
} sensor_data_t;

sensor_data_t sensor_pool[32];
uint32_t sensor_pool_bitmap = 0xFFFFFFFF;  // 所有區塊都空閒

sensor_data_t *sensor_alloc(void) {
    // 找到第一個空閒區塊
    int idx = __builtin_ffs(sensor_pool_bitmap) - 1;
    if (idx < 0) {
        return NULL;  // Pool 已滿
    }

    // 標記為已使用
    sensor_pool_bitmap &= ~(1U << idx);

    return &sensor_pool[idx];
}

void sensor_free(sensor_data_t *data) {
    int idx = data - sensor_pool;
    sensor_pool_bitmap |= (1U << idx);
}
```

**優點**：

- **零碎片化**：所有區塊大小相同
- **O(1) 分配**：只需要找到第一個 set bit
- **極快**：~5 cycles（vs. ~200 cycles for malloc）

**方法 #3：靜態分配長期存在的物件**

如果物件在整個系統生命週期中都存在，使用靜態分配。

**範例**：

```c
// 不好：動態分配
void main(void) {
    network_buffer_t *buffer = pvPortMalloc(sizeof(network_buffer_t));
    // buffer 在整個系統生命週期中都存在
}

// 好：靜態分配
static network_buffer_t g_network_buffer;

void main(void) {
    // 直接使用 g_network_buffer
}
```

**方法 #4：分層記憶體管理**

將記憶體分成多個區域，每個區域用於不同大小的物件。

**範例**：

```c
// 小物件池：32 bytes x 128 個
// 中等物件池：256 bytes x 32 個
// 大物件池：4 KB x 8 個

void *smart_alloc(size_t size) {
    if (size <= 32) {
        return pool_alloc(&small_pool);
    } else if (size <= 256) {
        return pool_alloc(&medium_pool);
    } else if (size <= 4096) {
        return pool_alloc(&large_pool);
    } else {
        return NULL;  // 太大
    }
}
```

---

## 九、記憶體使用分析與除錯

### 9.1 FreeRTOS 記憶體分析 API

FreeRTOS 提供了幾個 API 來分析記憶體使用：

**API #1：`xPortGetFreeHeapSize()`**

返回當前可用的 Heap 大小。

```c
size_t free_heap = xPortGetFreeHeapSize();
printf("Free heap: %zu bytes\n", free_heap);
```

**API #2：`xPortGetMinimumEverFreeHeapSize()`**

返回歷史最小的可用 Heap 大小（只有 heap_4 和 heap_5 支援）。

```c
size_t min_free_heap = xPortGetMinimumEverFreeHeapSize();
printf("Minimum ever free heap: %zu bytes\n", min_free_heap);
```

**這個 API 非常有用**：可以用來檢測記憶體使用的峰值。

**範例**：

```c
void vTaskMonitor(void *pvParameters) {
    while (1) {
        size_t free_heap = xPortGetFreeHeapSize();
        size_t min_free_heap = xPortGetMinimumEverFreeHeapSize();

        printf("Free: %zu bytes, Min: %zu bytes\n", free_heap, min_free_heap);

        if (min_free_heap < 1024) {
            printf("WARNING: Low memory!\n");
        }

        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}
```

**API #3：`configUSE_MALLOC_FAILED_HOOK`**

當記憶體分配失敗時，FreeRTOS 會呼叫 `vApplicationMallocFailedHook()`。

```c
// FreeRTOSConfig.h
#define configUSE_MALLOC_FAILED_HOOK  1

// main.c
void vApplicationMallocFailedHook(void) {
    printf("ERROR: Malloc failed!\n");
    printf("Free heap: %zu bytes\n", xPortGetFreeHeapSize());

    // 進入無限迴圈或重啟系統
    while (1) {
        // Halt
    }
}
```

### 9.2 記憶體洩漏檢測

**記憶體洩漏 (Memory Leak)**：分配的記憶體沒有被釋放。

**檢測方法**：

**方法 #1：監控 `xPortGetFreeHeapSize()`**

如果 `xPortGetFreeHeapSize()` 持續下降，可能有記憶體洩漏。

```c
void vTaskMonitor(void *pvParameters) {
    size_t prev_free_heap = xPortGetFreeHeapSize();

    while (1) {
        vTaskDelay(pdMS_TO_TICKS(10000));

        size_t curr_free_heap = xPortGetFreeHeapSize();

        if (curr_free_heap < prev_free_heap) {
            printf("WARNING: Memory leak detected!\n");
            printf("Previous: %zu, Current: %zu\n", prev_free_heap, curr_free_heap);
        }

        prev_free_heap = curr_free_heap;
    }
}
```

**方法 #2：使用 Heap Tracing**

FreeRTOS 提供了 Heap Tracing 功能，可以記錄所有的記憶體分配和釋放。

```c
// FreeRTOSConfig.h
#define configUSE_TRACE_FACILITY  1

// main.c
#include "heap_trace.h"

void main(void) {
    // 啟動 Heap Tracing
    vTraceEnable(TRC_START);

    // 建立 Tasks
    xTaskCreate(vTaskBlink, "Blink", 256, NULL, 1, NULL);

    // 啟動 Scheduler
    vTaskStartScheduler();
}
```

**方法 #3：手動追蹤**

在 `pvPortMalloc()` 和 `vPortFree()` 中加入追蹤程式碼。

```c
// 追蹤結構
typedef struct {
    void *ptr;
    size_t size;
    const char *file;
    int line;
} alloc_record_t;

alloc_record_t alloc_records[256];
int alloc_count = 0;

// 包裝 pvPortMalloc
#define pvPortMalloc(size)  pvPortMalloc_traced(size, __FILE__, __LINE__)

void *pvPortMalloc_traced(size_t size, const char *file, int line) {
    void *ptr = pvPortMalloc(size);

    if (ptr != NULL) {
        alloc_records[alloc_count].ptr = ptr;
        alloc_records[alloc_count].size = size;
        alloc_records[alloc_count].file = file;
        alloc_records[alloc_count].line = line;
        alloc_count++;
    }

    return ptr;
}

// 包裝 vPortFree
void vPortFree_traced(void *ptr) {
    // 從 alloc_records 中移除
    for (int i = 0; i < alloc_count; i++) {
        if (alloc_records[i].ptr == ptr) {
            alloc_records[i] = alloc_records[alloc_count - 1];
            alloc_count--;
            break;
        }
    }

    vPortFree(ptr);
}

// 列印所有未釋放的記憶體
void print_leaks(void) {
    printf("Unreleased allocations:\n");
    for (int i = 0; i < alloc_count; i++) {
        printf("  %p: %zu bytes at %s:%d\n",
               alloc_records[i].ptr,
               alloc_records[i].size,
               alloc_records[i].file,
               alloc_records[i].line);
    }
}
```

### 9.3 Stack Overflow 檢測

**Stack Overflow**：Task 的 Stack 不夠大，導致覆蓋其他記憶體。

**檢測方法**：

**方法 #1：`configCHECK_FOR_STACK_OVERFLOW`**

FreeRTOS 提供了 Stack Overflow 檢測功能。

```c
// FreeRTOSConfig.h
#define configCHECK_FOR_STACK_OVERFLOW  2  // 最嚴格的檢查

// main.c
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName) {
    printf("ERROR: Stack overflow in task %s!\n", pcTaskName);

    // 進入無限迴圈或重啟系統
    while (1) {
        // Halt
    }
}
```

**方法 #2：`uxTaskGetStackHighWaterMark()`**

返回 Task Stack 的最小剩餘空間（歷史最小值）。

```c
void vTaskMonitor(void *pvParameters) {
    while (1) {
        UBaseType_t watermark = uxTaskGetStackHighWaterMark(NULL);
        printf("Stack high water mark: %u words\n", watermark);

        if (watermark < 32) {
            printf("WARNING: Stack almost full!\n");
        }

        vTaskDelay(pdMS_TO_TICKS(5000));
    }
}
```

---

## 十、總結

### 10.1 核心要點

1. **FreeRTOS 提供 5 種 Heap 方案**：
   - heap_1：只能分配，不能釋放（最簡單）
   - heap_2：可以釋放，但不合併（不推薦）
   - heap_3：使用標準庫的 malloc/free（需要整合）
   - heap_4：支援合併，減少碎片化（最推薦）
   - heap_5：支援多個記憶體區域（進階）

2. **選擇 Heap 方案的原則**：
   - 一般用途：使用 heap_4
   - 簡單系統：使用 heap_1
   - 多記憶體區域：使用 heap_5
   - 需要與標準庫整合：使用 heap_3

3. **避免碎片化的方法**：
   - 使用 heap_4 或 heap_5（支援合併）
   - 使用記憶體池（固定大小的物件）
   - 靜態分配長期存在的物件
   - 分層記憶體管理

4. **記憶體分析與除錯**：
   - 使用 `xPortGetFreeHeapSize()` 監控可用記憶體
   - 使用 `xPortGetMinimumEverFreeHeapSize()` 檢測峰值
   - 使用 `configUSE_MALLOC_FAILED_HOOK` 捕捉分配失敗
   - 使用 `configCHECK_FOR_STACK_OVERFLOW` 檢測 Stack Overflow

### 10.2 實務建議

**建議 #1：優先使用靜態分配**

如果可能，使用 `xTaskCreateStatic()`、`xQueueCreateStatic()` 等靜態 API，可以完全避免動態記憶體分配。

**建議 #2：使用記憶體池**

對於固定大小的物件（如感測器資料、網路封包），使用記憶體池可以避免碎片化並提升效能。

**建議 #3：監控記憶體使用**

在開發階段，使用 `xPortGetFreeHeapSize()` 和 `xPortGetMinimumEverFreeHeapSize()` 監控記憶體使用，確保有足夠的餘裕。

**建議 #4：設定合理的 Heap 大小**

`configTOTAL_HEAP_SIZE` 應該設定為：

```
configTOTAL_HEAP_SIZE = (所有 Task Stack 大小總和) +
                        (所有 Queue 緩衝區大小總和) +
                        (動態分配的最大值) +
                        (20% 餘裕)
```

**建議 #5：啟用所有檢查**

在開發階段，啟用所有檢查：

```c
#define configUSE_MALLOC_FAILED_HOOK  1
#define configCHECK_FOR_STACK_OVERFLOW  2
#define configASSERT(x)  if (!(x)) { printf("ASSERT: %s:%d\n", __FILE__, __LINE__); while(1); }
```

### 10.3 回到 Mock Scenario

回到我們的 Mock Scenario：

**問題**：系統在運行 72 小時後會隨機崩潰，錯誤訊息是 "Out of Memory"。

**原因**：使用 heap_3（malloc/free），產生嚴重的記憶體碎片化。

**解決方案**：

1. **改用 heap_4**：支援合併，減少碎片化
2. **使用記憶體池**：感測器資料使用固定大小的記憶體池
3. **靜態分配**：網路緩衝區使用靜態分配

**結果**：系統穩定運行超過 3 個月，沒有任何記憶體問題。

### 10.4 下一篇預告

在下一篇文章中，我們將探討 **RTOS 除錯實戰：GDB + QEMU 的完整工作流程**：

- 如何在 QEMU 中使用 GDB 除錯 FreeRTOS
- 如何設定中斷點、單步執行、檢視變數
- 如何除錯 Context Switch 和中斷處理
- 如何分析 Task 狀態和記憶體使用

敬請期待！

---

## 參考資料

**官方文檔**：

- [FreeRTOS Memory Management](https://www.freertos.org/a00111.html)
- [FreeRTOS Heap Implementations](https://www.freertos.org/a00111.html#heap_1)
- [FreeRTOS API Reference](https://www.freertos.org/a00106.html)

**原始碼**：

- [FreeRTOS heap_1.c](https://github.com/FreeRTOS/FreeRTOS-Kernel/blob/main/portable/MemMang/heap_1.c)
- [FreeRTOS heap_4.c](https://github.com/FreeRTOS/FreeRTOS-Kernel/blob/main/portable/MemMang/heap_4.c)
- [FreeRTOS heap_5.c](https://github.com/FreeRTOS/FreeRTOS-Kernel/blob/main/portable/MemMang/heap_5.c)

**延伸閱讀**：

- [Memory Fragmentation in Embedded Systems](https://interrupt.memfault.com/blog/memory-fragmentation)
- [Best Practices for Memory Management in FreeRTOS](https://www.freertos.org/FAQMem.html)

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
