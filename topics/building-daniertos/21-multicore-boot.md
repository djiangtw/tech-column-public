# 眾人皆醒：Multi-Core Boot

**作者**: Danny Jiang
**日期**: 2025-12-14

---

## 前言：同時醒來的災難

想像這個場景：一個宿舍有四張床，四個室友。早上鬧鐘響了，四個人「同時」醒來，同時衝向唯一的浴室、同時搶唯一的熱水壺、同時翻冰箱找早餐。

混亂。

這就是 RISC-V QEMU 多核心開機的情況——在 `-bios none` 模式下，**所有 hart 同時從 `0x80000000` 開始執行**。

不像 x86 的 BSP/AP 架構（Bootstrap Processor 先啟動，完成初始化後再用 IPI 喚醒 Application Processors），RISC-V QEMU 沒有這種「禮讓」。所有核心一起醒來，一起跑 `_start`，一起嘗試初始化 BSS、設定 Stack、呼叫 `main()`。

如果不處理這個情況，結果是：

- 多個核心同時清除 BSS → 沒問題（但浪費）
- 多個核心用同一個 Stack → **Stack 互相覆蓋，程式崩潰**
- 多個核心同時呼叫 `main()` → **全域變數被初始化多次**

今天，我們要解決這個問題。

---

## 一、辨識自己：mhartid CSR

### 1.1 什麼是 Hart？

在 RISC-V 術語中，**Hart** (Hardware Thread) 就是一個硬體執行緒，可以簡單理解為「核心」。

每個 hart 都有一個唯一的 ID，存在 `mhartid` CSR (Control and Status Register) 中。

```c
// 讀取當前核心的 hart ID
static inline uint64_t get_hartid(void) {
    uint64_t hartid;
    asm volatile("csrr %0, mhartid" : "=r"(hartid));
    return hartid;
}
```

在 QEMU virt 機器上：

- 2 核心：hartid = 0, 1
- 4 核心：hartid = 0, 1, 2, 3

### 1.2 誰是老大？

我們選擇 **Hart 0** 作為 Primary Core（也叫 Boot Core、BSP）。Hart 0 負責：

1. 清除 BSS
2. 初始化全域資料結構
3. 設定 Trap Vector
4. 喚醒其他核心

其他核心（Hart 1, 2, ...）是 Secondary Core（也叫 AP），它們的責任是：

1. 等待 Primary Core 完成初始化
2. 設定自己的 Stack
3. 設定自己的 Trap Vector
4. 開始執行

---

## 二、啟動流程設計

### 2.1 流程圖

```
                    ┌─────────────────┐
                    │     _start      │
                    │  (所有核心同時) │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ csrr t0, mhartid │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
     ┌────────▼────────┐          ┌────────▼────────┐
     │   Hart 0?       │          │   Hart 1+?      │
     │   (Primary)     │          │   (Secondary)   │
     └────────┬────────┘          └────────┬────────┘
              │                             │
     ┌────────▼────────┐          ┌────────▼────────┐
     │ • 清除 BSS      │          │ • Spin-wait     │
     │ • 設定 Stack    │          │   等待 flag     │
     │ • 設定 mtvec    │          │                 │
     │ • 初始化 CLINT  │          │                 │
     └────────┬────────┘          └────────┬────────┘
              │                             │
     ┌────────▼────────┐                    │
     │ smp_boot_flag=1 │────────────────────┘
     │ (Memory Fence)  │                    │
     └────────┬────────┘          ┌────────▼────────┐
              │                   │ • 設定 Stack    │
     ┌────────▼────────┐          │ • 設定 mtvec    │
     │   main()        │          └────────┬────────┘
     └─────────────────┘                    │
                                  ┌────────▼────────┐
                                  │ secondary_main()│
                                  └─────────────────┘
```

### 2.2 關鍵：Boot Flag

Secondary Core 需要等待 Primary Core 完成初始化。我們使用一個全域變數 `smp_boot_flag`：

- `smp_boot_flag = 0`：尚未就緒，Secondary Core 繼續等待
- `smp_boot_flag = 1`：初始化完成，Secondary Core 可以開始

```c
volatile int smp_boot_flag = 0;  // 在 .data 段
```

### 2.3 關鍵：Per-Core Stack

每個核心需要自己的 Stack，否則會互相覆蓋。我們的配置：

```
記憶體配置：
┌─────────────────────────────────────────────┐
│                                             │
│              Stack 區域                      │
│                                             │
├─────────────────────────────────────────────┤ ← _stack_top
│ Hart 0 Stack  (4KB)                         │

### 3.1 完整的 startup.S

```asm
.section .text
.global _start

# 常數定義
.equ STACK_SIZE, 4096    # 每個核心 4KB Stack

_start:
    # ========================================
    # Step 1: 關閉中斷
    # ========================================
    csrw mie, zero

    # ========================================
    # Step 2: 讀取 Hart ID
    # ========================================
    csrr t0, mhartid

    # ========================================
    # Step 3: 分流 - Primary vs Secondary
    # ========================================
    bnez t0, secondary_hart_entry

# ========================================
# PRIMARY HART (Hart 0)
# ========================================
primary_hart_entry:
    # ----------------------------------------
    # Step 4: 設定 Primary Core 的 Stack
    # sp = _stack_top (Hart 0 用最高位置)
    # ----------------------------------------
    la   sp, _stack_top

    # ----------------------------------------
    # Step 5: 清除 BSS 段
    # ----------------------------------------
    la   t0, _bss_start
    la   t1, _bss_end
    bge  t0, t1, bss_done

bss_clear_loop:
    sd   zero, 0(t0)
    addi t0, t0, 8
    blt  t0, t1, bss_clear_loop

bss_done:
    # ----------------------------------------
    # Step 6: 設定 Trap Vector
    # ----------------------------------------
    la   t0, trap_vector
    csrw mtvec, t0

    # ----------------------------------------
    # Step 7: 設定 Boot Flag，喚醒其他核心
    # ----------------------------------------
    li   t0, 1
    la   t1, smp_boot_flag

    # Memory Fence: 確保 BSS 清除和其他初始化
    # 在設定 flag 之前完成
    fence w, w

    # 設定 flag
    sw   t0, 0(t1)

    # Memory Fence: 確保 flag 寫入對其他核心可見
    fence w, r

    # ----------------------------------------
    # Step 8: 跳到 main()
    # ----------------------------------------
    tail main

# ========================================
# SECONDARY HARTS (Hart 1, 2, ...)
# ========================================
secondary_hart_entry:
    # ----------------------------------------
    # Step 1: Spin-wait 等待 Primary Core
    # ----------------------------------------
    la   t1, smp_boot_flag

wait_loop:
    lw   t2, 0(t1)
    bnez t2, secondary_setup

    # WFI: 省電等待（可選）
    # 注意：QEMU 可能忽略 WFI
    wfi
    j    wait_loop

secondary_setup:
    # ----------------------------------------
    # Step 2: Memory Fence
    # 確保看到 Primary Core 初始化的所有資料
    # ----------------------------------------
    fence r, rw

    # ----------------------------------------
    # Step 3: 計算並設定 Stack
    # sp = _stack_top - (hartid × STACK_SIZE)
    # ----------------------------------------
    la   t0, _stack_top
    csrr t1, mhartid
    li   t2, STACK_SIZE
    mul  t2, t2, t1       # t2 = hartid × STACK_SIZE
    sub  sp, t0, t2       # sp = _stack_top - offset

    # ----------------------------------------
    # Step 4: 設定 Trap Vector
    # ----------------------------------------
    la   t0, trap_vector
    csrw mtvec, t0

    # ----------------------------------------
    # Step 5: 跳到 secondary_main()
    # ----------------------------------------
    tail secondary_main

# ========================================
# DATA SECTION
# ========================================
.section .data
.align 4
.global smp_boot_flag
smp_boot_flag:
    .word 0    # 0 = Not Ready, 1 = Ready
```

### 3.2 逐行解析

**Primary Hart 關鍵步驟：**

| 步驟 | 程式碼 | 說明 |
|------|--------|------|
| 關閉中斷 | `csrw mie, zero` | 初始化期間不要被中斷打斷 |
| 讀取 ID | `csrr t0, mhartid` | 判斷自己是哪個核心 |
| 分流 | `bnez t0, secondary` | 非 0 就跳到 Secondary 流程 |
| 設定 Stack | `la sp, _stack_top` | Hart 0 用最高位置 |
| 清除 BSS | `sd zero, 0(t0)` | 迴圈清除未初始化全域變數 |
| 設定 Trap | `csrw mtvec, t0` | 設定例外處理入口 |
| Fence | `fence w, w` | 確保寫入順序 |
| 設定 Flag | `sw t0, 0(t1)` | 通知 Secondary Cores |

**Secondary Hart 關鍵步驟：**

| 步驟 | 程式碼 | 說明 |
|------|--------|------|
| 等待 Flag | `lw t2, 0(t1)` | 讀取 `smp_boot_flag` |
| Spin | `bnez t2, setup` | Flag 為 0 就繼續等 |
| Fence | `fence r, rw` | 確保看到最新資料 |
| 計算 Stack | `mul t2, t2, t1` | offset = hartid × STACK_SIZE |
| 設定 Stack | `sub sp, t0, t2` | sp = _stack_top - offset |

---

## 四、Memory Fence 深入

### 4.1 為什麼需要 Fence？

RISC-V 使用 **Weak Memory Model** (RVWMO)。這意味著 CPU 和編譯器可能重排記憶體操作的順序。

沒有 Fence 的問題：

```c
// Primary Core
data = 42;           // (A)
smp_boot_flag = 1;   // (B)

// Secondary Core
while (smp_boot_flag == 0);  // (C)
use(data);                    // (D)
```

問題：編譯器可能把 (B) 排在 (A) 之前！Secondary Core 可能看到 flag=1 但 data 還沒更新。

### 4.2 Fence 指令格式

```asm
fence [predecessor], [successor]
```

參數：

- `r` = Read
- `w` = Write
- `rw` = Read + Write
- `i` = Input (Device)
- `o` = Output (Device)

常用組合：

| Fence | 效果 |
|-------|------|
| `fence w, w` | 前面的 Write 必須在後面的 Write 之前完成 |
| `fence r, r` | 前面的 Read 必須在後面的 Read 之前完成 |
| `fence rw, rw` | 完整的 Memory Barrier |
| `fence w, r` | Release 語意 |
| `fence r, rw` | Acquire 語意 |

### 4.3 在 Boot 中的使用

**Primary Core：**

```asm
# 確保所有初始化寫入完成
fence w, w

# 設定 flag
sw t0, 0(t1)

# 確保 flag 對其他核心可見
fence w, r
```

**Secondary Core：**

```asm
# 看到 flag 後
fence r, rw

# 確保後續讀取看到最新值
```

---

## 五、C 語言配合

### 5.1 main() 函式

Primary Core 的 `main()` 負責初始化 RTOS 並開始排程：

```c
#include <stdint.h>
#include "rtos.h"

extern volatile int smp_boot_flag;

void main(void) {
    // 1. 初始化 Per-Core 資料結構
    per_core_init(0);  // Hart 0

    // 2. 初始化 RTOS 核心
    scheduler_init();
    tick_init();

    // 3. 建立 Tasks
    task_create(task_a, "TaskA", 2);
    task_create(task_b, "TaskB", 1);

    // 4. 建立 Idle Task (Hart 0)
    idle_task_create(0);

    // 5. 開始排程
    uart_puts("[Hart 0] Starting scheduler\n");
    scheduler_start();

    // 不會到這裡
    while (1) {}
}
```

### 5.2 secondary_main() 函式

Secondary Core 的 `secondary_main()` 較簡單：

```c
void secondary_main(void) {
    // 1. 取得自己的 Hart ID
    uint64_t hartid = get_hartid();

    // 2. 初始化 Per-Core 資料結構
    per_core_init(hartid);

    // 3. 建立 Idle Task (這個 Hart)
    idle_task_create(hartid);

    // 4. 設定 Timer
    tick_init_secondary(hartid);

    // 5. 開始排程
    uart_printf("[Hart %d] Ready\n", hartid);
    scheduler_start_secondary();

    // 不會到這裡
    while (1) {}
}
```

### 5.3 Linker Script 配合

Linker Script 需要定義 Stack 區域：

```ld
/* linker.ld */
MEMORY {
    RAM (rwx) : ORIGIN = 0x80000000, LENGTH = 128M
}

SECTIONS {
    .text : { *(.text*) } > RAM
    .rodata : { *(.rodata*) } > RAM
    .data : { *(.data*) } > RAM

    .bss : {
        _bss_start = .;
        *(.bss*)
        _bss_end = .;
    } > RAM

    /* Stack 區域 - 從高位址向下增長 */
    . = ALIGN(16);
    . = . + 16K;  /* 4 cores × 4KB */
    _stack_top = .;
}
```

---

## 六、QEMU 驗證

### 6.1 編譯和執行

```bash
# 編譯
riscv64-unknown-elf-gcc -nostdlib -nostartfiles \
    -T linker.ld -o kernel.elf startup.S main.c

# 執行 (2 核心)
qemu-system-riscv64 -M virt -smp 2 -nographic \
    -bios none -kernel kernel.elf

# 執行 (4 核心)
qemu-system-riscv64 -M virt -smp 4 -nographic \
    -bios none -kernel kernel.elf
```

### 6.2 預期輸出

```
[Hart 0] Starting scheduler
[Hart 1] Ready
[Hart 0] TaskA running
[Hart 1] TaskB running
...
```

### 6.3 GDB 除錯

```bash
# 終端 1：啟動 QEMU 等待 GDB
qemu-system-riscv64 -M virt -smp 2 -nographic \
    -bios none -kernel kernel.elf -S -s

# 終端 2：連接 GDB
riscv64-unknown-elf-gdb kernel.elf
(gdb) target remote :1234
(gdb) info threads           # 查看所有 Harts
(gdb) thread 2               # 切換到 Hart 1
(gdb) break secondary_main   # 在 secondary_main 設中斷點
(gdb) continue
```

---

## 七、常見問題

### 7.1 Secondary Core 沒有醒來

**症狀**：只有 Hart 0 輸出訊息

**可能原因**：

1. `smp_boot_flag` 沒有被正確設定
2. 缺少 Memory Fence
3. QEMU 沒有用 `-smp N` 參數

**除錯方法**：

```c
// 在 secondary_main 開頭加上
uart_printf("[Hart %d] Woke up!\n", get_hartid());
```

### 7.2 Stack 互相覆蓋

**症狀**：程式執行一段時間後崩潰，變數被莫名修改

**可能原因**：

1. Stack 大小計算錯誤
2. hartid 讀取錯誤
3. `_stack_top` 位址不正確

**除錯方法**：

```c
// 檢查 Stack Pointer
uart_printf("[Hart %d] SP = %p\n", get_hartid(), get_sp());
```

### 7.3 Race Condition

**症狀**：每次執行結果不同

**可能原因**：

1. 共享資料沒有保護
2. 缺少 Spinlock

**解決方案**：見下一章 (Ch 23: Spinlock)

---

## 八、本章回顧

在這一章，我們實作了多核心啟動流程：

1. **mhartid**：讀取 CSR 判斷核心 ID
2. **Boot Flag**：Primary Core 設定 flag 通知 Secondary Cores
3. **Per-Core Stack**：每個核心有獨立的 Stack
4. **Memory Fence**：確保記憶體操作順序

關鍵程式碼：

```asm
# Primary: 設定 flag
fence w, w
sw t0, 0(t1)

# Secondary: 等待並讀取
wait_loop:
    lw t2, 0(t1)
    bnez t2, setup
    j wait_loop
setup:
    fence r, rw
```

下一章，我們將探討 **Per-Core Data**——如何設計每個核心的私有資料結構。

---

## 參考資料

- **See RISC-V Run**, Ch 10 - Multi-core Boot and IPI
- [RISC-V Weak Memory Ordering](https://riscv.org/specifications/isa-spec-pdf/)
- [QEMU RISC-V virt Machine](https://www.qemu.org/docs/master/system/riscv/virt.html)

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。
