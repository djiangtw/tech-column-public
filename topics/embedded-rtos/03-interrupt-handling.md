# RTOS 中斷處理：從硬體到軟體的完整流程

**作者**: Danny Jiang
**日期**: 2025-12-09

---

## 前言：一個讓人崩潰的 Hard Fault

2019 年，我在一家智慧家居公司擔任韌體工程師。有一天，測試團隊報告了一個嚴重的問題：

**系統在高負載時會隨機 Hard Fault，導致設備重啟。**

我們的產品是一個智慧門鎖，需要處理：

- UART 中斷（接收藍牙模組的資料）
- Timer 中斷（馬達控制的 PWM）
- GPIO 中斷（按鍵和感測器）

經過兩週的除錯，我發現問題出在 **ISR 中錯誤地呼叫了 FreeRTOS API**：

```c
// 錯誤的實作！
void UART_IRQHandler(void) {
    char data = UART_ReadByte();
    
    // ❌ 錯誤：在 ISR 中呼叫一般 API
    xQueueSend(uart_queue, &data, 0);  // 這會導致 Hard Fault！
}
```

**問題**：

- `xQueueSend()` 是一般 API，不能在 ISR 中呼叫
- 應該使用 `xQueueSendFromISR()`

**正確的實作**：

```c
void UART_IRQHandler(void) {
    char data = UART_ReadByte();
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    
    // ✅ 正確：使用 FromISR API
    xQueueSendFromISR(uart_queue, &data, &xHigherPriorityTaskWoken);
    
    // ✅ 正確：檢查是否需要 Context Switch
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

修正後，系統穩定運行至今，再也沒有 Hard Fault。

這個慘痛的教訓讓我深刻體會到：**在 ISR 中呼叫 RTOS API，必須遵守嚴格的規則。**

本文將深入探討 RTOS 的中斷處理機制。讀完這篇文章，你將能夠：

- 理解 RISC-V Timer Interrupt 的設定（mtime, mtimecmp, CSRs）
- 知道 ISR 中可以安全呼叫的 FreeRTOS API
- 掌握 FromISR 系列 API 與一般 API 的三大差異
- 了解 QEMU 虛擬硬體（CLINT, PLIC, UART）
- 實作正確的 ISR

---

## 一、RISC-V 中斷架構

在深入 FreeRTOS 的中斷處理之前，我們需要先理解 RISC-V 的中斷架構。

### 1.1 RISC-V 的中斷類型

RISC-V 有兩種中斷：

**1. Exception（例外）**

- 由 CPU 內部事件觸發
- 例如：非法指令、記憶體存取錯誤、系統呼叫（ecall）
- 同步（Synchronous）：與指令執行同步

**2. Interrupt（中斷）**

- 由外部硬體觸發
- 例如：Timer, UART, GPIO
- 非同步（Asynchronous）：與指令執行無關

### 1.2 RISC-V 的中斷來源

在 M-mode（Machine Mode）下，RISC-V 有三種中斷來源：

**1. Software Interrupt（軟體中斷）**

- 由軟體觸發（寫入 msip 暫存器）
- 用於 IPI（Inter-Processor Interrupt）

**2. Timer Interrupt（計時器中斷）**

- 由 Timer 觸發（mtime >= mtimecmp）
- FreeRTOS 的 Tick Interrupt 使用這個

**3. External Interrupt（外部中斷）**

- 由外部硬體觸發（UART, GPIO, etc.）
- 透過 PLIC（Platform-Level Interrupt Controller）管理

### 1.3 關鍵的 CSRs（Control and Status Registers）

**mstatus**（Machine Status）：

- 控制中斷的全域啟用/禁用
- MIE bit（bit 3）：Machine Interrupt Enable

**mie**（Machine Interrupt Enable）：

- 控制個別中斷的啟用/禁用
- MSIE（bit 3）：Software Interrupt Enable
- MTIE（bit 7）：Timer Interrupt Enable
- MEIE（bit 11）：External Interrupt Enable

**mip**（Machine Interrupt Pending）：

- 顯示哪些中斷正在 Pending
- MSIP（bit 3）：Software Interrupt Pending
- MTIP（bit 7）：Timer Interrupt Pending
- MEIP（bit 11）：External Interrupt Pending

**mcause**（Machine Cause）：

- 記錄中斷/例外的原因
- bit 31：Interrupt bit（1 = Interrupt, 0 = Exception）
- bits 0-30：Exception Code

**mepc**（Machine Exception PC）：

- 記錄中斷/例外發生時的 PC（Program Counter）
- 中斷返回時，會跳回這個位址

**mtvec**（Machine Trap Vector）：

- 中斷/例外處理函式的位址
- Mode bits（bits 0-1）：
  - 0 = Direct（所有中斷跳到同一個位址）
  - 1 = Vectored（不同中斷跳到不同位址）

### 1.4 中斷處理的完整流程

**當中斷發生時，CPU 自動執行以下步驟**：

1. **保存 PC**：mepc = PC（當前指令的位址）
2. **保存狀態**：mstatus.MPIE = mstatus.MIE（保存中斷啟用狀態）
3. **禁用中斷**：mstatus.MIE = 0（禁用全域中斷）
4. **設定 Cause**：mcause = 中斷原因
5. **跳轉**：PC = mtvec（跳到中斷處理函式）

**ISR 執行完畢後，執行 mret 指令**：

1. **恢復 PC**：PC = mepc（返回中斷前的位址）
2. **恢復狀態**：mstatus.MIE = mstatus.MPIE（恢復中斷啟用狀態）
3. **繼續執行**：從中斷前的位址繼續執行

### 1.5 重要觀念：RISC-V 預設不支援巢狀中斷

**關鍵點**：在步驟 3 中，硬體自動將 `mstatus.MIE = 0`，這表示：

**進入 ISR 後，全域中斷被關閉，因此預設不支援巢狀中斷（Nested Interrupts）。**

**這是什麼意思？**

- **巢狀中斷 (Nested Interrupt)**：高優先級中斷打斷正在執行的 ISR
- **RISC-V 預設行為**：進入 ISR 後，所有中斷都被禁用，直到 `mret` 執行

**這會影響 FreeRTOS 的任務搶佔嗎？**

**答案：不會！** 這是一個常見的誤解。

- **任務搶佔 (Task Preemption)**：中斷打斷 Task，並切換到另一個 Task
- **流程**：Task A → ISR → Task B
- **需求**：只需要「中斷」能打斷「任務」，RISC-V 完全支援

**FreeRTOS 的 Context Switch 發生在 Task → ISR → Task 之間，不需要 ISR → ISR 的巢狀中斷。**

**什麼時候需要巢狀中斷？**

是否需要巢狀中斷，取決於系統對 **「延遲 (Latency)」** 的容忍度。以下是必須或強烈建議使用巢狀中斷的場景：

**場景 #1：高階馬達控制 (FOC - Field Oriented Control)**

- **需求**：電流環 (Current Loop) 必須在每個 PWM 週期內完成（例如 20kHz, 即 50μs 內）
- **衝突**：如果正在處理耗時的 UART 字串列印或 I2C 讀取（可能耗時 100μs+），此時 PWM 中斷來了
- **後果**：PWM 中斷被迫等待，導致馬達控制迴圈抖動 (Jitter)，馬達會發出噪音、發熱甚至失控

**場景 #2：即時音訊處理 (Real-time Audio)**

- **需求**：音訊資料必須以固定的採樣率（如 48kHz）送入 DAC
- **後果**：如果因為 USB 封包處理卡住了音訊中斷，DAC 的緩衝區會空掉 (Underrun)，使用者會聽到爆音或雜訊

**場景 #3：高速通訊 (Gigabit Ethernet / High-speed USB)**

- **需求**：封包以極高速度湧入，Rx FIFO 很快就會滿
- **後果**：如果不允許網卡中斷打斷低速的周邊處理，Rx FIFO 會溢出 (Overflow)，導致嚴重掉包

**場景 #4：安全關鍵系統 (Safety Critical)**

- **需求**：無論 CPU 正在做什麼（哪怕是在處理另一個中斷），E-Stop 中斷必須立刻執行以切斷電源

**真的一定要巢狀中斷嗎？**

**答案是不一定。** 絕大多數的嵌入式應用其實不需要巢狀中斷。

通常會覺得「需要」巢狀中斷，往往是因為**驅動程式寫得不好**，把太多的邏輯放在 ISR 裡面做了。

- **如果你的 ISR 都很短 (例如 < 10μs)**：那根本不需要巢狀中斷。因為高優先級中斷最多只會被延遲 10μs，這對絕大多數系統都是可接受的。
- **如果你在 ISR 裡做 `printf`、字串複製、複雜運算**：那你就會覺得你需要巢狀中斷，否則系統會卡死。

**硬體不支援時的替代方案**

如果你的 RISC-V 硬體（如標準 CLINT/PLIC）不支援巢狀中斷，而你又面臨上述的高即時性需求，業界標準的解法有以下三種：

**方案 A：延遲處理 (Deferred Interrupt Processing) —— 最推薦**

這是現代 RTOS (Linux, FreeRTOS, Zephyr) 的標準設計模式，又稱為「Top Half / Bottom Half」設計。

**原理**：

1. **Top Half (ISR)**：保持極短。只做最緊急的事（如：從硬體 FIFO 讀一個 Byte、清除中斷旗標）
2. **觸發**：ISR 透過 `xSemaphoreGiveFromISR` 或 `xTaskNotifyFromISR` 喚醒一個高優先級的 Task
3. **退出**：ISR 立刻結束（中斷被重新打開）
4. **Bottom Half (Task)**：複雜的邏輯（如解析封包、複製陣列）在 Task 裡面做

**為什麼這能解決問題？**

因為複雜邏輯是在 **Task** 層級執行的，而 Task **隨時可以被新的中斷打斷**。也就是說，原本會在 ISR 裡卡住高優先級中斷的程式碼，現在變成了 Task，高優先級中斷就能自由地「搶佔」它了。

**範例**：

```c
// UART ISR (Top Half) - 極短
void UART_IRQHandler(void) {
    char data = UART_ReadByte();
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    // 只做最緊急的事：讀取資料，放入 Queue
    xQueueSendFromISR(uart_queue, &data, &xHigherPriorityTaskWoken);

    // 立刻退出，讓中斷重新開啟
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

// UART Task (Bottom Half) - 複雜邏輯
void vTaskUARTProcess(void *pvParameters) {
    char data;
    while (1) {
        // 等待資料（會阻塞）
        xQueueReceive(uart_queue, &data, portMAX_DELAY);

        // 複雜的處理（可以被中斷打斷）
        parse_protocol(data);
        update_display();
        log_to_flash();
    }
}
```

**方案 B：軟體模擬巢狀 (Software-Managed Nesting)**

如果你真的必須在 ISR 裡跑很久（例如 legacy code 改不動），你可以在 ISR 開頭手動開啟中斷。

**風險**：

- **Stack Overflow**：如果中斷來得太快，堆疊會被無限堆高直到爆掉
- **複雜度**：程式碼難以維護，容易出錯

**範例**：

```c
void UART_ISR(void) {
    // 保存 CSR
    uint32_t mstatus_save = read_csr(mstatus);
    uint32_t mepc_save = read_csr(mepc);

    // 手動開啟中斷，允許更高優先級中斷打斷
    set_csr(mstatus, MSTATUS_MIE);

    // 耗時的 UART 處理...

    // 關閉中斷，恢復 CSR
    clear_csr(mstatus, MSTATUS_MIE);
    write_csr(mstatus, mstatus_save);
    write_csr(mepc, mepc_save);
}
```

**方案 C：使用支援硬體巢狀的控制器 (RISC-V CLIC)**

標準的 RISC-V 使用 CLINT/PLIC，這不支援硬體巢狀。但 RISC-V 標準委員會定義了一個擴充標準：**CLIC (Core-Local Interrupt Controller)**。

**特點**：

- CLIC 專為嵌入式設計，硬體支援中斷搶佔級別 (Preemption Levels)
- 引入 **Level (搶佔等級)** 和 **Priority (仲裁優先權)** 的區別
- 硬體自動管理巢狀中斷，無需軟體介入

**重要警告**：

- ⚠️ CLIC 是**擴充標準**，不是所有 RISC-V CPU 都支援
- ⚠️ QEMU 預設的 `virt` machine **不支援** CLIC
- ⚠️ 需要確認硬體規格書明確支援 CLIC 或 ECLIC（廠商自定義版本）

**現狀**：許多新款的 MCU 等級 RISC-V 晶片（如 GigaDevice GD32V 部分型號、Nuclei Core）都實作了 CLIC。

**總結建議**

| 需求等級 | 建議解法 | 說明 |
|---------|---------|------|
| **一般應用** (IoT, 介面控制) | **保持 ISR 極短** | 這是最乾淨的寫法。不要在 ISR 裡做複雜運算。 |
| **複雜邏輯** (通訊協議, 數據搬運) | **延遲處理 (Task)** | 把 ISR 的工作丟給高優先級 Task 做，讓中斷能打斷 Task。 |
| **極限即時** (馬達 FOC, 音訊) | **硬體加速 (DMA) / CLIC** | 盡量用 DMA 搬資料，減少 CPU 進中斷的次數；或選用支援 CLIC 的晶片。 |
| **不得不** (Legacy Code) | **軟體模擬巢狀** | 下下策，需極度小心 Stack 溢出。 |

---

## 二、RISC-V Timer Interrupt 的設定

現在讓我們看看如何設定 RISC-V 的 Timer Interrupt。

### 2.1 QEMU virt Machine 的記憶體映射

QEMU 的 `virt` machine 提供以下硬體：

**CLINT（Core-Local Interruptor）**：

- 位址：0x0200_0000
- 功能：Timer 和 Software Interrupt
- mtime：0x0200_BFF8（64-bit，當前時間）
- mtimecmp：0x0200_4000（64-bit，比較值）

**PLIC（Platform-Level Interrupt Controller）**：

- 位址：0x0C00_0000
- 功能：管理外部中斷（UART, GPIO, etc.）

**UART0**：

- 位址：0x1000_0000
- 功能：串列通訊

**DRAM**：

- 位址：0x8000_0000
- 大小：128 MB（預設）

### 2.2 設定 Timer Interrupt

**步驟 1：定義記憶體映射位址**

```c
// CLINT 的記憶體映射位址
#define CLINT_BASE      0x02000000UL
#define MTIME_ADDR      (CLINT_BASE + 0xBFF8)  // 0x0200BFF8
#define MTIMECMP_ADDR   (CLINT_BASE + 0x4000)  // 0x02004000

// 存取 mtime 和 mtimecmp
#define MTIME           (*((volatile uint64_t *)MTIME_ADDR))
#define MTIMECMP        (*((volatile uint64_t *)MTIMECMP_ADDR))
```

**步驟 2：設定 Timer Interrupt**

```c
#define CPU_FREQ        10000000UL  // 10 MHz（QEMU virt machine）
#define TICK_RATE_HZ    1000        // 1000 Hz = 1 ms per tick

void vPortSetupTimerInterrupt(void) {
    // 1. 計算下一次 Interrupt 的時間
    uint64_t next_tick = MTIME + (CPU_FREQ / TICK_RATE_HZ);

    // 2. 設定 mtimecmp
    MTIMECMP = next_tick;

    // 3. 啟用 Timer Interrupt（設定 mie.MTIE）
    set_csr(mie, MIE_MTIE);  // MIE_MTIE = (1 << 7)

    // 4. 啟用全域中斷（設定 mstatus.MIE）
    set_csr(mstatus, MSTATUS_MIE);  // MSTATUS_MIE = (1 << 3)
}
```

**步驟 3：Timer Interrupt Handler**

```c
void handle_m_timer_interrupt(void) {
    // 1. 清除 Interrupt Pending（設定下一次 Interrupt）
    uint64_t next_tick = MTIME + (CPU_FREQ / TICK_RATE_HZ);
    MTIMECMP = next_tick;

    // 2. 呼叫 FreeRTOS 的 Tick Handler
    xPortSysTickHandler();
}
```

**步驟 4：中斷向量表**

```c
// 中斷向量表（簡化版）
void trap_handler(void) {
    uint64_t cause = read_csr(mcause);

    // 檢查是否是 Interrupt（bit 63 = 1）
    if (cause & (1UL << 63)) {
        // 取得 Exception Code（bits 0-62）
        uint64_t code = cause & 0x7FFFFFFFFFFFFFFF;

        switch (code) {
            case 7:  // Machine Timer Interrupt
                handle_m_timer_interrupt();
                break;
            case 11:  // Machine External Interrupt
                handle_m_external_interrupt();
                break;
            default:
                // 未知的中斷
                break;
        }
    } else {
        // Exception（非中斷）
        handle_exception(cause);
    }
}
```

### 2.3 CSR 操作的巨集

**讀取 CSR**：

```c
#define read_csr(reg) ({ \
    unsigned long __tmp; \
    asm volatile ("csrr %0, " #reg : "=r"(__tmp)); \
    __tmp; \
})
```

**寫入 CSR**：

```c
#define write_csr(reg, val) ({ \
    asm volatile ("csrw " #reg ", %0" :: "rK"(val)); \
})
```

**設定 CSR 的 Bit**：

```c
#define set_csr(reg, bit) ({ \
    unsigned long __tmp; \
    asm volatile ("csrrs %0, " #reg ", %1" : "=r"(__tmp) : "rK"(bit)); \
    __tmp; \
})
```

**清除 CSR 的 Bit**：

```c
#define clear_csr(reg, bit) ({ \
    unsigned long __tmp; \
    asm volatile ("csrrc %0, " #reg ", %1" : "=r"(__tmp) : "rK"(bit)); \
    __tmp; \
})
```

---

## 三、FromISR API 的三大差異

現在讓我們看看為什麼 ISR 中必須使用 FromISR API。

### 3.1 差異 #1：不能阻塞（No Blocking）

**一般 API**：

- 可以阻塞（Block）
- 例如：`xQueueSend(queue, &data, portMAX_DELAY)`
- 如果 Queue 滿了，Task 會進入 Blocked 狀態，等待空間

**FromISR API**：

- **絕對不能阻塞**
- 例如：`xQueueSendFromISR(queue, &data, &xHigherPriorityTaskWoken)`
- 如果 Queue 滿了，立即返回 `pdFAIL`
- ISR 必須盡快返回，不能等待

**原因**：

- ISR 在中斷上下文中執行，不是 Task
- 沒有 TCB（Task Control Block）
- 無法進入 Blocked 狀態

### 3.2 差異 #2：不能呼叫 Scheduler（No Scheduler Calls）

**一般 API**：

- 可能會觸發 Context Switch
- 例如：`xQueueSend()` 可能會喚醒等待的 Task
- 如果喚醒的 Task 優先權更高，立即切換

**FromISR API**：

- **不會立即觸發 Context Switch**
- 只會設定 `xHigherPriorityTaskWoken` 旗標
- ISR 返回前，手動檢查並觸發 Context Switch

**範例**：

```c
void UART_IRQHandler(void) {
    char data = UART_ReadByte();
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    // 1. 呼叫 FromISR API
    xQueueSendFromISR(uart_queue, &data, &xHigherPriorityTaskWoken);

    // 2. 檢查是否需要 Context Switch
    if (xHigherPriorityTaskWoken != pdFALSE) {
        // 3. 觸發 Context Switch（在 ISR 返回後執行）
        portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
    }
}
```

**portYIELD_FROM_ISR() 的實作**：

```c
// RISC-V 版本
#define portYIELD_FROM_ISR(x) \
{ \
    if (x != pdFALSE) { \
        vTaskSwitchContext(); \
    } \
}
```

### 3.3 差異 #3：Critical Section 的實作不同

**一般 API**：

- 使用 `taskENTER_CRITICAL()` / `taskEXIT_CRITICAL()`
- 禁用中斷（設定 mstatus.MIE = 0）
- 可以 Nested（巢狀）

**FromISR API**：

- 使用 `taskENTER_CRITICAL_FROM_ISR()` / `taskEXIT_CRITICAL_FROM_ISR()`
- **不禁用中斷**（已經在 ISR 中，中斷已被禁用）
- 只保存/恢復中斷狀態

**範例**：

```c
// 一般 API 的 Critical Section
void vTaskFunction(void) {
    taskENTER_CRITICAL();  // 禁用中斷

    // Critical Section
    shared_variable++;

    taskEXIT_CRITICAL();  // 恢復中斷
}

// FromISR API 的 Critical Section
void ISR_Handler(void) {
    UBaseType_t uxSavedInterruptStatus;

    uxSavedInterruptStatus = taskENTER_CRITICAL_FROM_ISR();

    // Critical Section
    shared_variable++;

    taskEXIT_CRITICAL_FROM_ISR(uxSavedInterruptStatus);
}
```

### 3.4 FromISR API 列表

**常用的 FromISR API**：

| 一般 API | FromISR API |
|----------|-------------|
| xQueueSend() | xQueueSendFromISR() |
| xQueueReceive() | xQueueReceiveFromISR() |
| xSemaphoreGive() | xSemaphoreGiveFromISR() |
| xSemaphoreTake() | xSemaphoreTakeFromISR() |
| xTaskNotifyGive() | vTaskNotifyGiveFromISR() |
| xTaskNotifyWait() | ❌ 不能在 ISR 中呼叫 |
| vTaskDelay() | ❌ 不能在 ISR 中呼叫 |

**記住**：

- 所有在 ISR 中呼叫的 FreeRTOS API 都必須是 FromISR 版本
- 如果沒有 FromISR 版本，就不能在 ISR 中呼叫

---

## 四、實際的 ISR 實作範例

現在讓我們看幾個實際的 ISR 範例。

### 4.1 範例 #1：UART 接收中斷

**場景**：UART 接收到資料，需要放入 Queue 給 Task 處理。

**實作**：

```c
// UART Queue
QueueHandle_t uart_rx_queue;

// UART 初始化
void uart_init(void) {
    // 1. 建立 Queue（可存放 128 個 char）
    uart_rx_queue = xQueueCreate(128, sizeof(char));

    // 2. 設定 UART 硬體
    UART0_CTRL = UART_RX_ENABLE | UART_RX_INT_ENABLE;

    // 3. 啟用 UART 中斷（透過 PLIC）
    plic_enable_interrupt(UART0_IRQ);
}

// UART ISR
void uart0_irq_handler(void) {
    // 1. 讀取資料
    char data = UART0_DATA;

    // 2. 清除中斷旗標
    UART0_STATUS = UART_RX_INT_CLEAR;

    // 3. 放入 Queue
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    xQueueSendFromISR(uart_rx_queue, &data, &xHigherPriorityTaskWoken);

    // 4. 檢查是否需要 Context Switch
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

// UART 接收 Task
void vTaskUARTReceive(void *pvParameters) {
    char data;

    while (1) {
        // 從 Queue 讀取資料（會阻塞）
        if (xQueueReceive(uart_rx_queue, &data, portMAX_DELAY) == pdTRUE) {
            // 處理資料
            printf("Received: %c\n", data);
        }
    }
}
```

### 4.2 範例 #2：GPIO 按鍵中斷

**場景**：按鍵按下，需要通知 Task 處理。

**實作**：

```c
// Binary Semaphore
SemaphoreHandle_t button_semaphore;

// GPIO 初始化
void gpio_init(void) {
    // 1. 建立 Binary Semaphore
    button_semaphore = xSemaphoreCreateBinary();

    // 2. 設定 GPIO 為輸入，啟用中斷
    GPIO_BUTTON_DIR = GPIO_INPUT;
    GPIO_BUTTON_INT_ENABLE = 1;
    GPIO_BUTTON_INT_EDGE = GPIO_FALLING_EDGE;

    // 3. 啟用 GPIO 中斷（透過 PLIC）
    plic_enable_interrupt(GPIO_BUTTON_IRQ);
}

// GPIO ISR
void gpio_button_irq_handler(void) {
    // 1. 清除中斷旗標
    GPIO_BUTTON_INT_CLEAR = 1;

    // 2. 釋放 Semaphore
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    xSemaphoreGiveFromISR(button_semaphore, &xHigherPriorityTaskWoken);

    // 3. 檢查是否需要 Context Switch
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

// 按鍵處理 Task
void vTaskButtonHandler(void *pvParameters) {
    while (1) {
        // 等待 Semaphore（會阻塞）
        if (xSemaphoreTake(button_semaphore, portMAX_DELAY) == pdTRUE) {
            // 處理按鍵事件
            printf("Button pressed!\n");

            // Debounce（防彈跳）
            vTaskDelay(pdMS_TO_TICKS(50));
        }
    }
}
```

### 4.3 範例 #3：Timer 中斷（PWM 控制）

**場景**：Timer 中斷，更新 PWM 訊號。

**實作**：

```c
// PWM 參數
volatile uint32_t pwm_duty_cycle = 50;  // 50%

// Timer 初始化
void timer_init(void) {
    // 設定 Timer（假設使用 CLINT 的 Timer）
    // 已在 vPortSetupTimerInterrupt() 中設定
}

// Timer ISR（在 FreeRTOS Tick Handler 中呼叫）
void handle_m_timer_interrupt(void) {
    static uint32_t tick_count = 0;

    // 1. 更新 PWM（每個 Tick 更新一次）
    if (tick_count % 100 < pwm_duty_cycle) {
        GPIO_PWM_PIN = 1;  // High
    } else {
        GPIO_PWM_PIN = 0;  // Low
    }

    tick_count++;

    // 2. 設定下一次 Interrupt
    uint64_t next_tick = MTIME + (CPU_FREQ / TICK_RATE_HZ);
    MTIMECMP = next_tick;

    // 3. 呼叫 FreeRTOS 的 Tick Handler
    xPortSysTickHandler();
}
```

---

## 五、QEMU 虛擬硬體

讓我們快速了解 QEMU virt machine 提供的虛擬硬體。

### 5.1 CLINT（Core-Local Interruptor）

**功能**：

- Timer Interrupt（mtime, mtimecmp）
- Software Interrupt（msip）

**記憶體映射**：

```
0x0200_0000: msip (Software Interrupt Pending)
0x0200_4000: mtimecmp (Timer Compare)
0x0200_BFF8: mtime (Timer Counter)
```

### 5.2 PLIC（Platform-Level Interrupt Controller）

**功能**：

- 管理外部中斷（UART, GPIO, etc.）
- 支援優先權
- 支援多個 CPU Core

**記憶體映射**：

```
0x0C00_0000: Priority registers
0x0C00_1000: Pending registers
0x0C00_2000: Enable registers
0x0C20_0000: Claim/Complete registers
```

### 5.3 UART0

**功能**：

- 串列通訊
- 支援 TX/RX 中斷

**記憶體映射**：

```
0x1000_0000: UART0 base address
  +0x00: TX/RX data
  +0x04: Interrupt enable
  +0x08: Interrupt status
  +0x0C: Line control
```

---

## 六、總結

本文深入探討了 RTOS 的中斷處理機制。讓我們回顧重點：

### 核心概念

1. **RISC-V 中斷架構**：
   - Exception vs. Interrupt
   - Software, Timer, External Interrupt
   - 關鍵 CSRs（mstatus, mie, mip, mcause, mepc, mtvec）
   - **重要**：RISC-V 預設不支援巢狀中斷（進入 ISR 後 mstatus.MIE = 0）

2. **任務搶佔 vs. 巢狀中斷**：
   - 任務搶佔：中斷打斷 Task（RISC-V 完全支援）
   - 巢狀中斷：中斷打斷 ISR（RISC-V 需軟體實作）
   - **FreeRTOS 只需要任務搶佔，不需要巢狀中斷**

3. **Timer Interrupt 設定**：
   - mtime 和 mtimecmp
   - 設定 mie.MTIE 和 mstatus.MIE
   - 中斷處理流程

4. **FromISR API 的三大差異**：
   - 不能阻塞（No Blocking）
   - 不能呼叫 Scheduler（No Scheduler Calls）
   - Critical Section 實作不同

5. **實際 ISR 範例**：
   - UART 接收中斷
   - GPIO 按鍵中斷
   - Timer 中斷（PWM 控制）

6. **QEMU 虛擬硬體**：
   - CLINT（Timer, Software Interrupt）
   - PLIC（External Interrupt）
   - UART0（串列通訊）

### 實務建議

1. **永遠使用 FromISR API**：
   - 在 ISR 中只能呼叫 FromISR 版本的 API
   - 檢查 `xHigherPriorityTaskWoken` 並呼叫 `portYIELD_FROM_ISR()`

2. **ISR 要盡快返回**：
   - 只做最少的工作（讀取資料、清除旗標）
   - 複雜的處理交給 Task

3. **使用 Queue 或 Semaphore 通訊**：
   - ISR 和 Task 之間用 Queue 傳遞資料
   - 用 Semaphore 通知事件

4. **注意中斷優先權**：
   - RISC-V 的中斷優先權由 PLIC 管理
   - 確保關鍵中斷有更高優先權

### 下一篇預告

在下一篇文章中，我們將探討 **RTOS 記憶體管理：5 種 Heap 方案的選擇與權衡**：

- heap_1 到 heap_5 的詳細比較
- 碎片化問題與解決方案
- 如何根據專案需求選擇合適的 Heap 方案
- 記憶體使用分析與除錯

敬請期待！

---

## 參考資料

**官方文檔**：

- [RISC-V Privileged Specification](https://riscv.org/technical/specifications/)
- [FreeRTOS Interrupt Management](https://www.freertos.org/RTOS-interrupt-management.html)
- [QEMU RISC-V virt Machine](https://www.qemu.org/docs/master/system/riscv/virt.html)

**原始碼**：

- [FreeRTOS RISC-V Port](https://github.com/FreeRTOS/FreeRTOS-Kernel/tree/main/portable/GCC/RISC-V)
- [QEMU RISC-V](https://github.com/qemu/qemu/tree/master/hw/riscv)

**技術文章**：

- [RISC-V Interrupt Handling](https://five-embeddev.com/riscv-isa-manual/latest/machine.html)
- [FreeRTOS FromISR APIs](https://www.freertos.org/a00106.html)

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
