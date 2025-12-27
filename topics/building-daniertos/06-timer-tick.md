# Timer & Tick 機制

**作者**: Danny Jiang
**日期**: 2025-12-13

---

## 前言：RTOS 的心跳

你知道人的心臟每分鐘跳動 60-100 次嗎？這個穩定的節奏驅動著全身的血液循環。

RTOS 也有類似的機制——**Timer Tick**。

我第一次意識到 Tick 的重要性，是在一個沒有 Tick 的系統上。那時我天真地以為：「只要有 Context Switch，Task 就可以輪流執行了。」

結果呢？第一個 Task 開始執行後，就再也不會切換出去。因為沒有人去「叫」它切換。

想像一下，你在辦公室工作，沒有時鐘，也沒有人來叫你開會。你會永遠坐在座位上，直到世界末日。

**Tick 就是那個定期來敲門的人。**

每跳動一次，RTOS 就有機會：

- 檢查是否有 Task 該醒來了
- 決定是否該進行 Time Slice 輪轉
- 更新系統時間

沒有心跳，大腦再聰明也沒有用——因為它根本不會被觸發。

> 💡 **設計決策**：danieRTOS 使用 1ms 的 Tick（1000Hz）。這是嵌入式系統中最常見的配置，平衡了精確度和 CPU 負擔。

---

## 一、System Tick 的作用

### 1.1 什麼是 Tick？

**Tick** 是 RTOS 的最小時間單位。每次 Timer 中斷，系統就經過了「一個 Tick」。

```
時間軸：
    |-------|-------|-------|-------|-------|-------|
    Tick 0  Tick 1  Tick 2  Tick 3  Tick 4  Tick 5
            ↑       ↑       ↑       ↑       ↑
         中斷    中斷    中斷    中斷    中斷
```

### 1.2 Tick 中斷做什麼？

每次 Tick 中斷，RTOS 核心會：

1. **遞增 Tick 計數器**：`tick_count++`
2. **喚醒到期的 Task**：檢查 Delayed List，把時間到的 Task 移到 Ready List
3. **觸發排程**：執行 Round-Robin 輪轉（如果有同優先級的其他 Task）

### 1.3 Tick 頻率的選擇

**Tick Rate（Tick 頻率）** 決定了系統的時間解析度。

| Tick Rate | 週期 | 優點 | 缺點 |
|-----------|------|------|------|
| 100 Hz | 10 ms | 中斷開銷低 | 時間解析度差 |
| 1000 Hz | 1 ms | 時間精確 | 中斷開銷高 |

**danieRTOS 的選擇：1000 Hz（1 ms）**

原因：

- 在 QEMU 上跑，效能不是瓶頸
- 計算時間直觀（delay(100) = 100 ms）
- 教育用途，精確度優先

```c
#define TICK_RATE_HZ    1000
#define TICK_PERIOD_US  (1000000 / TICK_RATE_HZ)  // 1000 us = 1 ms
```

---

## 二、RISC-V CLINT Timer

### 2.1 CLINT 簡介

**CLINT（Core Local Interruptor）** 是 RISC-V 的標準 Timer 模組。在 QEMU virt machine 上，CLINT 位於固定的記憶體位址。

```c
// QEMU virt machine 的 CLINT 位址
#define CLINT_BASE      0x2000000UL
#define CLINT_MTIME     (CLINT_BASE + 0xBFF8)  // 64-bit 計時器（只讀）
#define CLINT_MTIMECMP  (CLINT_BASE + 0x4000)  // 64-bit 比較暫存器
```

### 2.2 Timer 運作原理

CLINT Timer 的運作非常簡單：

1. `mtime` 是一個自動遞增的 64-bit 計數器（通常以固定頻率遞增）
2. 當 `mtime >= mtimecmp` 時，產生 Timer Interrupt
3. 軟體在中斷處理中設定新的 `mtimecmp` 值，準備下一次中斷

```
mtime:     0 → 1 → 2 → 3 → ... → 999 → 1000 → 1001 → ...
                                    ↑
                              mtimecmp = 1000
                              產生中斷！
```

### 2.3 Timer 頻率

在 QEMU virt machine 上，`mtime` 的遞增頻率是 **10 MHz**（每秒 10,000,000 次）。

```c
#define TIMER_FREQ      10000000UL  // 10 MHz

// 每個 Tick 需要多少個 mtime 週期
#define TICK_INTERVAL   (TIMER_FREQ / TICK_RATE_HZ)  // 10000
```

---

## 三、Timer 初始化

### 3.1 設定第一次中斷

```c
// timer.c

volatile uint64_t *mtime    = (volatile uint64_t *)CLINT_MTIME;
volatile uint64_t *mtimecmp = (volatile uint64_t *)CLINT_MTIMECMP;

void timer_init(void) {
    // 設定第一次 Timer 中斷
    *mtimecmp = *mtime + TICK_INTERVAL;
}
```

### 3.2 開啟 Timer 中斷

Timer 中斷需要在 CSR 中開啟：

```c
void timer_interrupt_enable(void) {
    // 1. 在 mie 中開啟 Machine Timer Interrupt Enable (MTIE)
    uint64_t mie;
    asm volatile("csrr %0, mie" : "=r"(mie));
    mie |= (1 << 7);  // MTIE 在 bit 7
    asm volatile("csrw mie, %0" : : "r"(mie));

    // 2. 在 mstatus 中開啟 Machine Interrupt Enable (MIE)
    uint64_t mstatus;
    asm volatile("csrr %0, mstatus" : "=r"(mstatus));
    mstatus |= (1 << 3);  // MIE 在 bit 3
    asm volatile("csrw mstatus, %0" : : "r"(mstatus));
}
```

### 3.3 完整的系統啟動流程

```c
void scheduler_start(void) {
    // 1. 創建 Idle Task
    create_idle_task();

    // 2. 選擇第一個 Task
    current_tcb = find_highest_priority_ready_task();

    // 3. 初始化 Timer
    timer_init();
    timer_interrupt_enable();

    // 4. 跳轉到第一個 Task（永不返回）
    start_first_task();
}
```

---

## 四、Tick Handler 實作

### 4.1 中斷來源判斷

在 RISC-V 中，所有的 Trap（Exception 和 Interrupt）都會跳到 `mtvec` 指向的位址。我們需要根據 `mcause` 來判斷是什麼類型的 Trap。

```c
// mcause 的最高位元表示是否為 Interrupt
#define MCAUSE_INTERRUPT_FLAG  (1UL << 63)

// Timer Interrupt 的 Exception Code
#define MCAUSE_TIMER_INTERRUPT 7

bool is_timer_interrupt(uint64_t mcause) {
    return (mcause & MCAUSE_INTERRUPT_FLAG) &&
           ((mcause & 0xFF) == MCAUSE_TIMER_INTERRUPT);
}
```

### 4.2 Tick Handler

```c
// 系統 Tick 計數器
volatile uint64_t tick_count = 0;

void handle_timer_interrupt(void) {
    // 1. 設定下一次 Timer 中斷
    *mtimecmp = *mtime + TICK_INTERVAL;

    // 2. 遞增 Tick 計數器
    tick_count++;

    // 3. 喚醒到期的 Blocked Task（下一章詳述）
    wake_delayed_tasks();

    // 4. 執行排程（Round-Robin 輪轉）
    schedule();
}
```

### 4.3 整合到 Trap Handler

```c
void handle_trap(uint64_t mcause, uint64_t mepc) {
    if (is_timer_interrupt(mcause)) {
        handle_timer_interrupt();
    } else {
        // 其他類型的 Trap...
        handle_exception(mcause, mepc);
    }
}
```

在 Assembly 的 `trap_handler` 中：

```asm
trap_handler:
    SAVE_CONTEXT

    csrr a0, mcause
    csrr a1, mepc
    call handle_trap

    RESTORE_CONTEXT
```

---

## 五、64-bit Tick Count 的優勢

### 5.1 32-bit 的溢位問題

在許多 32-bit MCU 上，RTOS 使用 32-bit 的 Tick Count。問題是：

```
32-bit 最大值：4,294,967,295
在 1000 Hz 下：4,294,967 秒 ≈ 49.7 天
```

49.7 天後，Tick Count 會**溢位歸零**。這會導致時間判斷邏輯出錯，例如：

```c
// 危險！如果 tick_count 溢位，這個判斷會出錯
if (tick_count >= task->wake_time) {
    wake_task(task);
}
```

FreeRTOS 使用複雜的「雙 List」機制來處理溢位。

### 5.2 RV64 的紅利

我們的平台是 RV64，可以直接使用 64-bit Tick Count：

```c
volatile uint64_t tick_count = 0;  // 64-bit
```

64-bit 的最大值：

```
18,446,744,073,709,551,615
在 1000 Hz 下：18,446,744,073,709,551 秒 ≈ 5.84 億年
```

**結論**：在 danieRTOS 中，完全不需要考慮 Tick 溢位問題。這大大簡化了程式碼！

### 5.3 簡化的時間比較

因為不會溢位，時間比較變得非常簡單：

```c
// 安全！64-bit 不會溢位
if (tick_count >= task->wake_time) {
    wake_task(task);
}
```

---

## 六、Time Slice 輪轉

### 6.1 什麼時候輪轉？

當同優先級有多個 Ready Task 時，每個 Tick 中斷都會觸發 Round-Robin 輪轉。

```c
void schedule(void) {
    tcb_t *old_tcb = current_tcb;

    // 1. Round-Robin 輪轉
    if (old_tcb != NULL && old_tcb->state == TASK_READY) {
        ready_list_rotate(old_tcb->priority);
    }

    // 2. 選擇最高優先級的 Task
    current_tcb = find_highest_priority_ready_task();
}
```

### 6.2 更複雜的 Time Slice（可選）

如果想讓每個 Task 執行多個 Tick 再切換，可以加入 Time Slice Counter：

```c
#define TIME_SLICE_TICKS 10  // 每個 Task 執行 10 個 Tick

void handle_timer_interrupt(void) {
    *mtimecmp = *mtime + TICK_INTERVAL;
    tick_count++;

    // Time Slice Counter
    if (current_tcb != NULL) {
        current_tcb->time_slice_remaining--;

        if (current_tcb->time_slice_remaining == 0) {
            current_tcb->time_slice_remaining = TIME_SLICE_TICKS;
            schedule();  // 時間片用完，強制切換
        }
    }

    wake_delayed_tasks();
}
```

在 danieRTOS Phase 1，我們使用最簡單的方式：每個 Tick 都檢查是否需要輪轉。

---

## 七、Tick Hook（可選功能）

### 7.1 什麼是 Tick Hook？

Tick Hook 是一個「鉤子函數」，在每個 Tick 中斷中被呼叫。使用者可以在這裡放入需要週期執行的程式碼。

```c
// 使用者定義的 Hook 函數
__attribute__((weak))
void danie_tick_hook(void) {
    // 預設是空的，使用者可以覆寫
}

void handle_timer_interrupt(void) {
    *mtimecmp = *mtime + TICK_INTERVAL;
    tick_count++;

    // 呼叫使用者的 Hook
    danie_tick_hook();

    wake_delayed_tasks();
    schedule();
}
```

### 7.2 使用範例

```c
// 使用者程式碼
static uint32_t led_counter = 0;

void danie_tick_hook(void) {
    led_counter++;
    if (led_counter >= 500) {  // 每 500 ms
        led_toggle();
        led_counter = 0;
    }
}
```

### 7.3 注意事項

Tick Hook 在**中斷上下文**中執行，必須：

- 執行時間要短
- 不能呼叫會 Block 的 API（如 `delay()`）
- 不能呼叫 `schedule()`（已經在 Tick Handler 中了）

---

## 總結

本文實作了 danieRTOS 的 Timer & Tick 機制：

1. **Tick 的作用**：驅動時間推進和排程
2. **CLINT Timer**：使用 mtime/mtimecmp 產生週期中斷
3. **Tick Handler**：更新計數器、喚醒 Task、觸發排程
4. **64-bit 優勢**：不需要處理溢位問題
5. **Time Slice**：實現 Round-Robin 輪轉

現在我們有了「心跳」，下一步是讓 Task 可以「睡覺」——實作 Delay 機制。

---

## 參考資料

**RISC-V 規格**

- **RISC-V Instruction Set Manual, Volume II: Privileged Architecture**
  RISC-V International
  https://github.com/riscv/riscv-isa-manual
  Timer Interrupt 和 CLINT 的官方規格。

**延伸閱讀**

- **See RISC-V Run**
  Danny Jiang
  RISC-V 中斷機制的深入解析。

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)
