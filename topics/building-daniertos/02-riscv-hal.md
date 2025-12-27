# RISC-V HAL：M-mode 初始化與 CSR

**作者**: Danny Jiang
**日期**: 2025-12-13

---

## 前言：在 M-mode 中與硬體直接對話

第一次寫 Bare-metal 程式的人，通常會遇到一個讓人崩潰的問題：**程式明明編譯成功了，但跑起來什麼都沒有。**

沒有錯誤訊息，沒有 Segmentation Fault，就是單純的「什麼都沒發生」。

我第一次遇到這個情況時，花了整整一個下午才發現原因：**Stack Pointer 沒有設定。** 沒有 Stack，就沒有函數呼叫，`main()` 根本沒有被執行。

這就是 Bare-metal 的殘酷現實——在 Linux 上，作業系統幫你做了無數的初始化工作；在這裡，一切都要從頭開始。

第一步是建立 **Hardware Abstraction Layer (HAL)**——硬體抽象層。在 M-mode（Machine Mode）下，我們直接與硬體對話，沒有作業系統的保護網。這意味著：

- 程式的入口不是 `main()`，而是 Linker Script 指定的 `_start`
- 需要手動設定 Stack Pointer、清除 BSS
- 需要直接操作 CSR（Control and Status Registers）控制 CPU 行為
- 需要設定 Timer 產生週期性中斷

這些底層操作在 Linux 或 Windows 上是透明的，但在 Bare-metal 環境下，**每一步都需要我們親手完成**。聽起來很可怕？其實一旦理解了原理，你會發現這種「完全掌控」的感覺非常過癮。

本文將帶你從 Reset Vector 一路走到 `main()`，並設定好 Timer Interrupt。讀完這篇文章，你將能夠：

- 理解 RISC-V bare-metal 程式的完整啟動流程
- 掌握 QEMU virt machine 的 Memory Map
- 學會設定核心 CSR（mstatus、mtvec、mie）
- 實作週期性 Timer Interrupt

---

## 一、啟動流程：從 Reset 到 main()

### 1.1 Bare-metal 程式的入口

在一般的應用程式開發中，程式從 `main()` 開始執行。但這只是一個假象——在 `main()` 之前，C Runtime（CRT）已經做了大量的初始化工作。

在 Bare-metal 環境下，沒有 CRT，所有初始化工作都要我們自己來。程式的真正入口是 Linker Script 指定的 `_start` 符號，通常寫在 `start.S`（組合語言）中。

### 1.2 完整啟動流程

```
┌──────────────────────────────────────────────────────────┐
│  1. Reset Vector                                          │
│     CPU 上電，PC 指向 QEMU 內部 ROM                        │
│     ROM 跳轉到 0x80000000（我們的 Kernel 起始位置）        │
├──────────────────────────────────────────────────────────┤
│  2. _start (start.S)                                      │
│     ├── 關閉中斷 (mstatus.MIE = 0)                        │
│     ├── 設定 Global Pointer (gp)                          │
│     ├── 設定 Stack Pointer (sp)                           │
│     ├── 清除 BSS 區段                                     │
│     └── 跳轉到 main()                                     │
├──────────────────────────────────────────────────────────┤
│  3. main() (C 語言世界)                                   │
│     ├── 初始化 UART                                       │
│     ├── 初始化 Timer                                      │
│     ├── 創建 Task                                         │
│     └── 啟動 Scheduler                                    │
└──────────────────────────────────────────────────────────┘
```

### 1.3 start.S 實作

```asm
# start.S - RISC-V bare-metal 啟動程式碼

.section .text.init
.global _start

_start:
    # 1. 關閉中斷
    csrw mie, zero              # 清除所有中斷使能
    csrci mstatus, 0x8          # 清除 MIE bit

    # 2. 設定 Global Pointer
    .option push
    .option norelax
    la gp, __global_pointer$
    .option pop

    # 3. 設定 Stack Pointer
    la sp, __stack_top          # 從 Linker Script 取得 stack 頂端地址

    # 4. 清除 BSS 區段
    la t0, __bss_start
    la t1, __bss_end
clear_bss:
    bgeu t0, t1, bss_done       # 如果 t0 >= t1，跳出迴圈
    sd zero, 0(t0)              # 將 8 bytes 填 0
    addi t0, t0, 8              # t0 += 8
    j clear_bss
bss_done:

    # 5. 跳轉到 main()
    call main

    # 6. main() 不應該返回，如果返回就無限迴圈
halt:
    wfi                         # Wait For Interrupt（省電）
    j halt
```

**關鍵步驟解釋**：

1. **關閉中斷**：初始化過程中，不希望被中斷打斷
2. **設定 gp**：Global Pointer 用於快速存取全域變數（Linker 優化）
3. **設定 sp**：最重要的一步！沒有 Stack，就沒有函數呼叫
4. **清除 BSS**：C 語言標準要求未初始化的全域變數為 0
5. **跳轉到 main()**：進入 C 語言世界

---

## 二、QEMU virt Memory Map

在 QEMU 中，硬體地址是固定的。對於 danieRTOS，需要關注以下三個區域：

### 2.1 記憶體佈局

| 地址範圍 | 用途 | 說明 |
|----------|------|------|
| `0x0200_0000` | **CLINT** | Core Local Interrupter，Timer 在這裡 |
| `0x1000_0000` | **UART0** | 串口輸出，我們的 `printf` 窗口 |
| `0x8000_0000` | **DRAM** | 程式碼、資料、Stack、Heap 全部放在這裡 |

### 2.2 DRAM 內部佈局

```
0x80000000  ┌──────────────────────┐
            │ .text (程式碼)        │
            ├──────────────────────┤
            │ .rodata (唯讀資料)    │
            ├──────────────────────┤
            │ .data (已初始化資料)  │
            ├──────────────────────┤
            │ .bss (未初始化資料)   │
            ├──────────────────────┤
            │ Heap (由低向高成長)   │
            │        ↓             │
            │        ...           │
            │        ↑             │
            │ Stack (由高向低成長)  │
            ├──────────────────────┤
0x80800000  └──────────────────────┘ (假設 8MB RAM)
```

### 2.3 Linker Script

Linker Script 定義了各個區段的位置和大小：

```ld
/* linker.ld */

OUTPUT_ARCH(riscv)
ENTRY(_start)

MEMORY
{
    RAM (rwx) : ORIGIN = 0x80000000, LENGTH = 8M
}

SECTIONS
{
    .text : {
        *(.text.init)       /* 啟動程式碼放最前面 */
        *(.text*)
    } > RAM

    .rodata : {
        *(.rodata*)
    } > RAM

    .data : {
        *(.data*)
    } > RAM

    .bss : {
        __bss_start = .;
        *(.bss*)
        *(COMMON)
        __bss_end = .;
    } > RAM

    . = ALIGN(16);
    __stack_bottom = .;
    . += 0x10000;           /* 64KB Stack */
    __stack_top = .;

    __global_pointer$ = .;
}
```

---

## 三、CSR 設定

CSR（Control and Status Registers）是控制 CPU 行為的核心。我們使用以下指令操作 CSR：

- `csrr rd, csr`：讀取 CSR 到暫存器
- `csrw csr, rs`：寫入暫存器到 CSR
- `csrs csr, rs`：設定 CSR 的特定 bits
- `csrc csr, rs`：清除 CSR 的特定 bits

### 3.1 mstatus：全域狀態控制

`mstatus` 是最重要的 CSR，控制中斷和特權模式。

```
┌─────────────────────────────────────────────────────────┐
│                        mstatus                           │
├─────┬─────┬─────┬─────┬─────┬─────┬─────────────────────┤
│ ... │ MPP │MPIE │ ... │ MIE │ ... │                     │
│     │12:11│  7  │     │  3  │     │                     │
└─────┴─────┴─────┴─────┴─────┴─────┴─────────────────────┘
```

**關鍵欄位**：

| 欄位 | Bit | 說明 |
|------|-----|------|
| **MIE** | 3 | Machine Interrupt Enable，全域中斷總開關 |
| **MPIE** | 7 | Machine Previous Interrupt Enable，進入中斷前的 MIE 值 |
| **MPP** | 12:11 | Machine Previous Privilege，進入中斷前的特權模式 |

**設定方法**：

```c
// 開啟全域中斷
static inline void enable_interrupts(void) {
    asm volatile("csrsi mstatus, 0x8");  // Set MIE bit
}

// 關閉全域中斷
static inline void disable_interrupts(void) {
    asm volatile("csrci mstatus, 0x8");  // Clear MIE bit
}
```

### 3.2 mtvec：中斷向量基地址

`mtvec` 告訴 CPU 發生中斷要去哪裡執行。

```
┌─────────────────────────────────────────────────────────┐
│                         mtvec                            │
├─────────────────────────────────────────────────────┬───┤
│                    BASE (地址)                       │MOD│
│                      [63:2]                          │1:0│
└─────────────────────────────────────────────────────┴───┘
```

**Mode（最低 2 bits）**：

- `0`（Direct）：所有 Trap 跳到同一個地址。**danieRTOS 使用此模式**
- `1`（Vectored）：不同中斷跳到不同地址

**設定方法**：

```c
extern void trap_handler(void);

void setup_trap_vector(void) {
    asm volatile("csrw mtvec, %0" : : "r"((uint64_t)trap_handler));
}
```

### 3.3 mie：個別中斷使能

`mie` 控制各類中斷的開關：

| 欄位 | Bit | 說明 |
|------|-----|------|
| **MSIE** | 3 | Machine Software Interrupt Enable |
| **MTIE** | 7 | Machine Timer Interrupt Enable |
| **MEIE** | 11 | Machine External Interrupt Enable |

**設定方法**：

```c
// 開啟 Timer 中斷
static inline void enable_timer_interrupt(void) {
    asm volatile("csrs mie, %0" : : "r"(1 << 7));  // Set MTIE bit
}
```

### 3.4 mip：中斷待處理

`mip` 顯示目前有哪些中斷正在排隊（大部分情況唯讀）。

---

## 四、Timer 設定

RISC-V 的 Timer 不在 CSR 中，而是在 **CLINT**（Core Local Interrupter）的 Memory-Mapped 區域。

### 4.1 CLINT 結構

| 地址 | 暫存器 | 說明 |
|------|--------|------|
| `CLINT_BASE + 0xBFF8` | mtime | 64-bit 計數器，不停遞增 |
| `CLINT_BASE + 0x4000` | mtimecmp | 64-bit 比較器 |

**觸發條件**：當 `mtime >= mtimecmp` 時，產生 Timer Interrupt。

### 4.2 QEMU virt 的 Timer 頻率

QEMU virt machine 的時基頻率是 **10 MHz**（10,000,000 Hz）。

**計算 Tick Interval**：

假設我們要 1ms（1000 Hz）的 System Tick：

```
interval = CLOCK_FREQ / TICK_RATE
         = 10,000,000 / 1000
         = 10,000 cycles
```

### 4.3 Timer 實作

```c
// CLINT 地址定義
#define CLINT_BASE      0x2000000UL
#define CLINT_MTIME     (*(volatile uint64_t*)(CLINT_BASE + 0xBFF8))
#define CLINT_MTIMECMP  (*(volatile uint64_t*)(CLINT_BASE + 0x4000))

// Timer 參數
#define CLOCK_FREQ      10000000    // 10 MHz
#define TICK_RATE_HZ    1000        // 1000 Hz = 1ms per tick
#define TICK_INTERVAL   (CLOCK_FREQ / TICK_RATE_HZ)

void timer_init(void) {
    // 設定下一次中斷時間
    CLINT_MTIMECMP = CLINT_MTIME + TICK_INTERVAL;

    // 開啟 Timer 中斷使能
    asm volatile("csrs mie, %0" : : "r"(1 << 7));  // MTIE
}

void timer_interrupt_handler(void) {
    // 設定下一次中斷（Delta 模式，防止時間飄移）
    CLINT_MTIMECMP += TICK_INTERVAL;

    // 更新 Tick 計數
    tick_count++;

    // 觸發 Scheduler
    schedule();
}
```

**重要**：每次 Timer 中斷後，必須手動更新 `mtimecmp`，否則中斷會持續觸發。

---

## 五、Trap Handler

### 5.1 區分 Interrupt 與 Exception

RISC-V 將 Interrupt 和 Exception 統稱為 **Trap**。透過 `mcause` 暫存器的最高位（MSB）區分：

- **MSB = 1**：Interrupt（非同步事件）
- **MSB = 0**：Exception（同步事件，如非法指令）

```c
void handle_trap(uint64_t mcause, uint64_t mepc) {
    if (mcause & (1ULL << 63)) {
        // Interrupt
        uint64_t code = mcause & 0xFF;
        switch (code) {
            case 7:
                timer_interrupt_handler();
                break;
            default:
                danie_panic("Unknown interrupt");
        }
    } else {
        // Exception
        danie_panic("Exception occurred");
    }
}
```

### 5.2 Trap Handler（Assembly）

```asm
# trap.S - Trap 處理入口

.section .text
.global trap_handler
.align 4

trap_handler:
    # 保存所有暫存器（簡化版，實際需要保存完整 Context）
    addi sp, sp, -256
    sd ra, 0(sp)
    sd t0, 8(sp)
    sd t1, 16(sp)
    sd t2, 24(sp)
    # ... 保存其他暫存器 ...

    # 讀取 mcause 和 mepc
    csrr a0, mcause
    csrr a1, mepc

    # 呼叫 C 語言的 handle_trap()
    call handle_trap

    # 恢復暫存器
    ld ra, 0(sp)
    ld t0, 8(sp)
    ld t1, 16(sp)
    ld t2, 24(sp)
    # ... 恢復其他暫存器 ...
    addi sp, sp, 256

    # 返回
    mret
```

**注意**：這是簡化版，真正的 Context Switch 需要保存完整的 32 個暫存器，這將在下一篇文章詳細說明。

---

## 六、完整初始化流程

把所有東西串起來：

```c
// main.c

#include "daniertos.h"

volatile uint64_t tick_count = 0;

int main(void) {
    // 1. 初始化 UART（用於除錯輸出）
    uart_init();
    uart_puts("danieRTOS starting...\n");

    // 2. 設定 Trap Handler
    setup_trap_vector();

    // 3. 初始化 Timer
    timer_init();

    // 4. 創建 Task（下一篇實作）
    // task_create(...);

    // 5. 啟動 Scheduler（下一篇實作）
    // scheduler_start();

    // 6. 開啟全域中斷
    enable_interrupts();

    // 7. 等待中斷
    while (1) {
        asm volatile("wfi");  // Wait For Interrupt
    }

    return 0;  // 永遠不會到這裡
}
```

---

## 總結

本文介紹了 danieRTOS 的 RISC-V HAL 實作：

1. **啟動流程**：`_start` → 設定 sp/gp → 清除 BSS → `main()`
2. **Memory Map**：CLINT（Timer）、UART、DRAM 的地址
3. **CSR 設定**：mstatus（全域中斷）、mtvec（Trap 向量）、mie（個別中斷）
4. **Timer 設定**：mtime/mtimecmp 產生週期性中斷
5. **Trap Handler**：區分 Interrupt 和 Exception，處理 Timer 中斷

HAL 是 RTOS 的地基。有了這個基礎，我們才能開始實作上層的 Task 管理和 Scheduler。

---

## 參考資料

**RISC-V 規格**

- **RISC-V Instruction Set Manual, Volume II: Privileged Architecture**
  RISC-V International
  https://github.com/riscv/riscv-isa-manual
  M-mode、mstatus、mtvec、mie 等 CSR 的官方規格。

- **RISC-V Core Local Interrupter (CLINT)**
  SiFive CLINT Specification (參考實作)
  CLINT 包含 mtime 和 mtimecmp，用於 Timer Interrupt。

**QEMU 文檔**

- **QEMU RISC-V virt Machine**
  https://www.qemu.org/docs/master/system/riscv/virt.html
  QEMU virt machine 的 Memory Map 和周邊設備說明。

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)
