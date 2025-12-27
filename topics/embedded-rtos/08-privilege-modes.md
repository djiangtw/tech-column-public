# RISC-V 特權模式：M-mode, S-mode, U-mode 的設計與應用

**作者**: Danny Jiang
**日期**: 2025-12-09

---

## 前言：當你需要隔離不可信的程式碼

2022 年，我在一家物聯網安全公司擔任安全架構師。有一天，產品經理找到我，說：

**「我們的智慧家居閘道需要支援第三方應用，但我們不能信任這些程式碼。如果第三方應用崩潰或被駭，不能影響核心系統。」**

我們的產品是一個智慧家居閘道，運行 FreeRTOS。核心系統包含網路協定棧、安全模組、OTA 更新等關鍵功能。第三方應用則包含智慧燈泡控制、溫度監控、語音助理等。

安全需求很明確：第三方應用不能存取核心系統的記憶體，不能直接存取硬體（例如 Flash、網路介面），而且崩潰時不能影響核心系統。

但傳統的 FreeRTOS 運行在 M-mode（最高特權），所有 Tasks 都有相同的權限。這意味著任何 Task 都可以存取任何記憶體、任何硬體。

我開始研究解決方案。

第一個想法是使用 MPU (Memory Protection Unit)。一些 MCU 提供 MPU，可以設定記憶體保護區域。但 MPU 的區域數量有限（通常只有 8-16 個），配置複雜，而且效能 Overhead 高。

第二個想法是使用虛擬化。使用 Hypervisor 來隔離不同的應用。但這需要硬體支援（RISC-V H Extension），而且 Overhead 太高，不適合嵌入式系統。

最後，我發現了 RISC-V 特權模式的設計。我們可以讓核心系統運行在 M-mode（最高特權），第三方應用運行在 U-mode（最低特權），並使用 PMP (Physical Memory Protection) 來隔離記憶體。

**解決方案**：

1. **核心系統運行在 M-mode**（最高特權，可以存取所有硬體）
2. **第三方應用運行在 U-mode**（最低特權，受限制）
3. **使用 PMP 設定記憶體保護**（第三方應用只能存取特定區域）

**結果**：

- 第三方應用無法存取核心系統的記憶體
- 第三方應用無法直接存取硬體
- 第三方應用崩潰時，觸發 Exception，核心系統可以處理

---

## 一、RISC-V 特權模式概述

### 1.1 三種特權模式

RISC-V 定義了 3 種特權模式：

| 模式 | 名稱 | 編碼 | 說明 |
|------|------|------|------|
| **M-mode** | Machine Mode | 11 | 最高特權，可以存取所有硬體 |
| **S-mode** | Supervisor Mode | 01 | 作業系統核心（例如 Linux Kernel） |
| **U-mode** | User Mode | 00 | 使用者應用程式 |

**特權等級**：M-mode > S-mode > U-mode

### 1.2 為什麼需要特權模式？

**原因 #1：安全性**

隔離不可信的程式碼，防止惡意程式存取敏感資料或硬體。

**原因 #2：穩定性**

防止應用程式崩潰影響整個系統。

**原因 #3：資源管理**

作業系統可以控制應用程式對硬體資源的存取。

### 1.3 不同模式的權限

**M-mode（Machine Mode）**：

- ✅ 可以存取所有 CSRs
- ✅ 可以存取所有記憶體
- ✅ 可以存取所有硬體
- ✅ 可以處理所有 Interrupt 和 Exception

**S-mode（Supervisor Mode）**：

- ✅ 可以存取部分 CSRs（`s*` 開頭的 CSRs）
- ⚠️ 記憶體存取受 PMP 限制
- ❌ 不能直接存取硬體（需要透過 M-mode）
- ✅ 可以處理部分 Interrupt 和 Exception

**U-mode（User Mode）**：

- ❌ 不能存取 CSRs（除了少數例外，例如 `cycle`, `time`）
- ⚠️ 記憶體存取受 PMP 限制
- ❌ 不能直接存取硬體
- ❌ 不能處理 Interrupt 和 Exception

### 1.4 特權模式的切換

**從低特權到高特權**：使用 `ecall` 指令

```asm
# U-mode 呼叫 M-mode 的服務
ecall
```

**從高特權到低特權**：使用 `mret` 或 `sret` 指令

```asm
# M-mode 返回到 U-mode
mret
```

**硬體自動切換**：當 Interrupt 或 Exception 發生時

```
U-mode → (Interrupt) → M-mode
M-mode → (mret) → U-mode
```

---

## 二、M-mode（Machine Mode）

### 2.1 M-mode 的特性

**M-mode 是 RISC-V 的最高特權模式**：

- 所有 RISC-V 實作都必須支援 M-mode
- M-mode 可以存取所有硬體資源
- M-mode 是系統啟動時的預設模式

**M-mode 的使用場景**：

- **Bootloader**：系統啟動時的初始化程式碼
- **Bare-metal 應用**：不需要作業系統的簡單應用
- **RTOS**：FreeRTOS、Zephyr 等嵌入式 RTOS
- **Firmware**：硬體抽象層（HAL）

### 2.2 M-mode 的 CSRs

**M-mode 專用的 CSRs**：

| CSR | 名稱 | 說明 |
|-----|------|------|
| `mstatus` | Machine Status | 包含 MIE, MPIE, MPP 等狀態位元 |
| `misa` | Machine ISA | CPU 支援的 ISA 擴充 |
| `mie` | Machine Interrupt Enable | 中斷啟用遮罩 |
| `mtvec` | Machine Trap Vector | Trap Handler 的地址 |
| `mscratch` | Machine Scratch | 臨時暫存器（用於保存 Context） |
| `mepc` | Machine Exception PC | Exception 發生時的 PC |
| `mcause` | Machine Cause | Exception 的原因 |
| `mtval` | Machine Trap Value | Exception 的額外資訊 |
| `mip` | Machine Interrupt Pending | 中斷 Pending 狀態 |
| `mvendorid` | Machine Vendor ID | CPU 廠商 ID |
| `marchid` | Machine Architecture ID | CPU 架構 ID |
| `mimpid` | Machine Implementation ID | CPU 實作 ID |
| `mhartid` | Machine Hardware Thread ID | CPU 核心 ID |

### 2.3 FreeRTOS 在 M-mode 運行

**為什麼 FreeRTOS 運行在 M-mode？**

**原因 #1：簡單**

M-mode 不需要配置 PMP 或虛擬記憶體，實作簡單。

**原因 #2：效能**

M-mode 沒有額外的權限檢查，效能最高。

**原因 #3：硬體存取**

嵌入式系統通常需要直接存取硬體，M-mode 可以直接存取所有硬體。

**原因 #4：相容性**

大多數嵌入式 RISC-V CPU 只實作 M-mode（不支援 S-mode 或 U-mode）。

**FreeRTOS 的 Trap Handler**：

```c
void handle_trap(void) {
    uintptr_t mcause_val = read_csr(mcause);
    
    if (mcause_val & MCAUSE_INTERRUPT) {
        // Interrupt
        uintptr_t code = mcause_val & MCAUSE_CODE_MASK;
        
        if (code == IRQ_M_TIMER) {
            handle_m_timer_interrupt();
        } else if (code == IRQ_M_EXT) {
            handle_m_external_interrupt();
        }
    } else {
        // Exception
        uintptr_t code = mcause_val & MCAUSE_CODE_MASK;
        
        if (code == CAUSE_ILLEGAL_INSTRUCTION) {
            handle_illegal_instruction();
        } else {
            // 未處理的 Exception
            while (1);
        }
    }
}
```

---

## 三、S-mode（Supervisor Mode）

### 3.1 S-mode 的特性

**S-mode 是為作業系統設計的特權模式**：

- S-mode 可以管理虛擬記憶體（使用 MMU）
- S-mode 可以處理部分 Interrupt 和 Exception
- S-mode 受 M-mode 的 PMP 限制

**S-mode 的使用場景**：

- **Linux Kernel**：運行在 S-mode
- **其他 OS Kernel**：例如 FreeBSD、Zephyr（支援 MMU 的版本）

### 3.2 S-mode 的 CSRs

**S-mode 專用的 CSRs**：

| CSR | 名稱 | 說明 |
|-----|------|------|
| `sstatus` | Supervisor Status | S-mode 的狀態（MIE, MPIE, SPP 等） |
| `sie` | Supervisor Interrupt Enable | S-mode 的中斷啟用遮罩 |
| `stvec` | Supervisor Trap Vector | S-mode 的 Trap Handler 地址 |
| `sscratch` | Supervisor Scratch | S-mode 的臨時暫存器 |
| `sepc` | Supervisor Exception PC | S-mode 的 Exception PC |
| `scause` | Supervisor Cause | S-mode 的 Exception 原因 |
| `stval` | Supervisor Trap Value | S-mode 的 Exception 額外資訊 |
| `sip` | Supervisor Interrupt Pending | S-mode 的中斷 Pending 狀態 |
| `satp` | Supervisor Address Translation | 虛擬記憶體的頁表基址 |

### 3.3 Linux 在 S-mode 運行

**Linux 的特權模式架構**：

```
┌─────────────────────────────────────┐
│         User Applications           │ ← U-mode
├─────────────────────────────────────┤
│         Linux Kernel                │ ← S-mode
├─────────────────────────────────────┤
│         OpenSBI (Firmware)          │ ← M-mode
└─────────────────────────────────────┘
```

**OpenSBI (Open Source Supervisor Binary Interface)**：

- 運行在 M-mode
- 提供 SBI 服務給 S-mode（例如：Timer, IPI, Console）
- 處理 M-mode 的 Interrupt 和 Exception

**Linux Kernel**：

- 運行在 S-mode
- 使用 MMU 管理虛擬記憶體
- 透過 `ecall` 呼叫 OpenSBI 的服務

**User Applications**：

- 運行在 U-mode
- 透過 System Call（`ecall`）呼叫 Linux Kernel

### 3.4 S-mode 的 Trap Delegation

**問題**：如果所有 Trap 都由 M-mode 處理，效能會很差。

**解決方案**：Trap Delegation

M-mode 可以將部分 Trap 委派給 S-mode 處理。

**配置 Trap Delegation**：

```c
// 將 Timer Interrupt 委派給 S-mode
write_csr(mideleg, (1 << IRQ_S_TIMER));

// 將 Page Fault 委派給 S-mode
write_csr(medeleg, (1 << CAUSE_LOAD_PAGE_FAULT) |
                   (1 << CAUSE_STORE_PAGE_FAULT) |
                   (1 << CAUSE_FETCH_PAGE_FAULT));
```

**效果**：

- 當 Timer Interrupt 發生時，硬體會直接跳轉到 S-mode 的 Trap Handler
- 當 Page Fault 發生時，硬體會直接跳轉到 S-mode 的 Trap Handler

---

## 四、U-mode（User Mode）

### 4.1 U-mode 的特性

**U-mode 是最低特權模式**：

- U-mode 不能存取 CSRs（除了少數例外）
- U-mode 的記憶體存取受 PMP 或 MMU 限制
- U-mode 不能直接存取硬體

**U-mode 的使用場景**：

- **使用者應用程式**：在 Linux 上運行的應用程式
- **第三方程式碼**：不可信的程式碼（例如：插件、腳本）

### 4.2 U-mode 可以存取的 CSRs

**U-mode 可以讀取的 CSRs**：

| CSR | 名稱 | 說明 |
|-----|------|------|
| `cycle` | Cycle Counter | CPU Cycle 計數器 |
| `time` | Timer | 當前時間（從 `mtime` 映射） |
| `instret` | Instructions Retired | 已執行的指令數量 |

**範例**：

```c
// U-mode 程式碼
uint64_t start = read_csr(cycle);
do_work();
uint64_t end = read_csr(cycle);
printf("Elapsed cycles: %llu\n", end - start);
```

### 4.3 從 U-mode 呼叫 M-mode 服務

**使用 `ecall` 指令**：

```asm
# U-mode 程式碼
li      a0, 1           # System Call Number
li      a1, 42          # Argument
ecall                   # 呼叫 M-mode
# 返回後，a0 包含返回值
```

**M-mode 的 Trap Handler**：

```c
void handle_trap(void) {
    uintptr_t mcause_val = read_csr(mcause);
    
    if (mcause_val == CAUSE_USER_ECALL) {
        // 處理 U-mode 的 ecall
        uintptr_t syscall_num = read_csr(a0);
        uintptr_t arg = read_csr(a1);
        
        uintptr_t result = handle_syscall(syscall_num, arg);
        
        // 將返回值寫入 a0
        write_csr(a0, result);
        
        // 更新 mepc（跳過 ecall 指令）
        uintptr_t mepc_val = read_csr(mepc);
        write_csr(mepc, mepc_val + 4);
    }
}
```

### 4.4 FreeRTOS 支援 U-mode

**FreeRTOS 可以讓部分 Tasks 運行在 U-mode**：

```c
// 建立 U-mode Task
xTaskCreateRestricted(&xTaskParameters, &xHandle);

// Task Parameters
TaskParameters_t xTaskParameters = {
    .pvTaskCode = vUserTask,
    .pcName = "UserTask",
    .usStackDepth = 512,
    .pvParameters = NULL,
    .uxPriority = 2,
    .puxStackBuffer = user_stack,
    .xRegions = {
        // 設定記憶體區域（使用 PMP）
        {0x20000000, 0x1000, portMPU_REGION_READ_WRITE},
        {0x30000000, 0x1000, portMPU_REGION_READ_ONLY},
    }
};
```

**當 Context Switch 到 U-mode Task 時**：

```asm
# 設定 mstatus.MPP = U-mode
li      t0, MSTATUS_MPP_U
csrc    mstatus, MSTATUS_MPP_MASK
csrs    mstatus, t0

# 返回到 U-mode
mret
```

---

## 五、PMP (Physical Memory Protection)

### 5.1 什麼是 PMP？

**PMP (Physical Memory Protection)**：RISC-V 的記憶體保護機制。

**核心功能**：

- 限制 S-mode 和 U-mode 對記憶體的存取
- M-mode 可以配置 PMP
- M-mode 不受 PMP 限制

**PMP 的使用場景**：

- 隔離不可信的程式碼
- 保護敏感資料（例如：密鑰、憑證）
- 防止 Stack Overflow 覆蓋其他記憶體

### 5.2 PMP 的配置

**PMP 有 16 個 Entry（pmp0-pmp15）**：

每個 Entry 包含：

- **pmpaddr[i]**：記憶體區域的地址
- **pmpcfg[i]**：記憶體區域的配置（權限、模式）

**pmpcfg 的格式**：

| Bit | 名稱 | 說明 |
|-----|------|------|
| 7 | L | Lock（鎖定，M-mode 也無法修改） |
| 4:3 | A | Address Matching Mode |
| 2 | X | Executable（可執行） |
| 1 | W | Writable（可寫入） |
| 0 | R | Readable（可讀取） |

**Address Matching Mode**：

| A | 模式 | 說明 |
|---|------|------|
| 00 | OFF | 停用 |
| 01 | TOR | Top of Range（範圍：pmpaddr[i-1] ~ pmpaddr[i]） |
| 10 | NA4 | Naturally Aligned 4-byte |
| 11 | NAPOT | Naturally Aligned Power-of-Two |

### 5.3 PMP 配置範例

**範例 #1：保護 Flash（只讀）**

```c
// Flash: 0x20000000 - 0x20100000 (1 MB)
// 設定為只讀（R-X）

// pmpaddr0 = 0x20000000 >> 2
write_csr(pmpaddr0, 0x20000000 >> 2);

// pmpaddr1 = 0x20100000 >> 2
write_csr(pmpaddr1, 0x20100000 >> 2);

// pmpcfg0 = TOR | R | X
write_csr(pmpcfg0, (PMP_TOR << 3) | PMP_R | PMP_X);
```

**範例 #2：保護 RAM（可讀寫）**

```c
// RAM: 0x80000000 - 0x80010000 (64 KB)
// 設定為可讀寫（RW-）

// pmpaddr2 = 0x80000000 >> 2
write_csr(pmpaddr2, 0x80000000 >> 2);

// pmpaddr3 = 0x80010000 >> 2
write_csr(pmpaddr3, 0x80010000 >> 2);

// pmpcfg0 的 Entry 1（Bits 15:8）
uint32_t pmpcfg0_val = read_csr(pmpcfg0);
pmpcfg0_val |= ((PMP_TOR << 3) | PMP_R | PMP_W) << 8;
write_csr(pmpcfg0, pmpcfg0_val);
```

**範例 #3：使用 NAPOT 模式**

```c
// 保護 4 KB 區域：0x80000000 - 0x80001000
// 使用 NAPOT 模式（更簡單）

// NAPOT 的地址計算：
// pmpaddr = (base >> 2) | ((size - 1) >> 3)
// size 必須是 2 的冪次

uintptr_t base = 0x80000000;
uintptr_t size = 0x1000;  // 4 KB

uintptr_t pmpaddr_val = (base >> 2) | ((size - 1) >> 3);
write_csr(pmpaddr4, pmpaddr_val);

// pmpcfg1 的 Entry 0（Bits 7:0）
write_csr(pmpcfg1, (PMP_NAPOT << 3) | PMP_R | PMP_W);
```

### 5.4 PMP 的優先權

**當多個 PMP Entry 匹配時，使用最低編號的 Entry**。

**範例**：

```c
// Entry 0: 0x80000000 - 0x80010000 (RW-)
// Entry 1: 0x80000000 - 0x80001000 (R--)

// 存取 0x80000500：
// - 匹配 Entry 0 和 Entry 1
// - 使用 Entry 0（最低編號）
// - 權限：RW-
```

**最佳實踐**：將最嚴格的規則放在最前面。

### 5.5 PMP 的 Lock 位元

**Lock 位元（L）**：一旦設定，M-mode 也無法修改這個 Entry。

**使用場景**：保護 Bootloader 或安全模組。

**範例**：

```c
// 保護 Bootloader（0x20000000 - 0x20010000）
write_csr(pmpaddr0, 0x20000000 >> 2);
write_csr(pmpaddr1, 0x20010000 >> 2);

// 設定為只讀，並鎖定
write_csr(pmpcfg0, (PMP_TOR << 3) | PMP_R | PMP_X | PMP_L);

// 之後無法修改這個 Entry（即使在 M-mode）
```

---

## 六、特權模式切換

### 6.1 從 U-mode 切換到 M-mode（ecall）

**U-mode 程式碼**：

```c
// U-mode 呼叫 System Call
int result = syscall(SYS_WRITE, fd, buffer, size);
```

**組合語言**：

```asm
# syscall(int num, ...)
syscall:
    # a0 = num, a1 = arg1, a2 = arg2, ...
    ecall           # 觸發 Environment Call Exception
    ret             # 返回（a0 包含返回值）
```

**M-mode Trap Handler**：

```c
void handle_trap(void) {
    uintptr_t mcause_val = read_csr(mcause);

    if (mcause_val == CAUSE_USER_ECALL) {
        // 讀取參數
        uintptr_t num = read_csr(a0);
        uintptr_t arg1 = read_csr(a1);
        uintptr_t arg2 = read_csr(a2);

        // 處理 System Call
        uintptr_t result;
        switch (num) {
            case SYS_WRITE:
                result = sys_write(arg1, arg2, read_csr(a3));
                break;
            // ...
        }

        // 寫入返回值
        write_csr(a0, result);

        // 更新 mepc（跳過 ecall 指令）
        uintptr_t mepc_val = read_csr(mepc);
        write_csr(mepc, mepc_val + 4);
    }
}
```

### 6.2 從 M-mode 切換到 U-mode（mret）

**M-mode 程式碼**：

```c
void start_user_task(void (*entry)(void), void *stack) {
    // 1. 設定 mepc（返回地址）
    write_csr(mepc, (uintptr_t)entry);

    // 2. 設定 mstatus.MPP = U-mode
    uintptr_t mstatus_val = read_csr(mstatus);
    mstatus_val &= ~MSTATUS_MPP_MASK;
    mstatus_val |= MSTATUS_MPP_U;
    write_csr(mstatus, mstatus_val);

    // 3. 設定 Stack Pointer
    asm volatile ("mv sp, %0" :: "r"(stack));

    // 4. 返回到 U-mode
    asm volatile ("mret");
}
```

**組合語言**：

```asm
start_user_task:
    # a0 = entry, a1 = stack

    # 1. 設定 mepc
    csrw    mepc, a0

    # 2. 設定 mstatus.MPP = U-mode (00)
    li      t0, MSTATUS_MPP_MASK
    csrc    mstatus, t0         # 清除 MPP

    # 3. 設定 Stack Pointer
    mv      sp, a1

    # 4. 返回到 U-mode
    mret
```

### 6.3 mstatus 的關鍵位元

**mstatus 的格式**（RV32）：

| Bit | 名稱 | 說明 |
|-----|------|------|
| 31 | SD | Dirty（FPU/Vector 狀態） |
| 12:11 | MPP | Previous Privilege Mode（M-mode 進入 Trap 前的模式） |
| 7 | MPIE | Previous Interrupt Enable（M-mode 進入 Trap 前的 MIE） |
| 3 | MIE | Machine Interrupt Enable（M-mode 的中斷啟用） |

**MPP 的值**：

| MPP | 模式 |
|-----|------|
| 00 | U-mode |
| 01 | S-mode |
| 11 | M-mode |

**mret 的行為**：

1. 將 `mstatus.MPIE` 複製到 `mstatus.MIE`
2. 將 `mstatus.MPP` 設定為 U-mode（00）
3. 跳轉到 `mepc`

### 6.4 完整的特權模式切換流程

**流程圖**：

```
U-mode (應用程式)
    |
    | ecall (System Call)
    v
M-mode (Trap Handler)
    |
    | 處理 System Call
    |
    | mret
    v
U-mode (應用程式)
```

**時序圖**：

```
Time →

U-mode:  [執行] → [ecall] → [等待] → [繼續執行]
                      ↓         ↑
M-mode:              [Trap] → [處理] → [mret]
```

---

## 七、實戰案例：隔離第三方程式碼

### 7.1 系統架構

**目標**：

- 核心系統運行在 M-mode
- 第三方應用運行在 U-mode
- 使用 PMP 隔離記憶體

**記憶體佈局**：

```
0x00000000 - 0x00010000: Bootloader (M-mode, Locked)
0x20000000 - 0x20100000: Flash (M-mode, R-X)
0x80000000 - 0x80010000: Core RAM (M-mode, RW-)
0x80010000 - 0x80020000: App RAM (U-mode, RW-)
0x40000000 - 0x40001000: Peripherals (M-mode only)
```

### 7.2 PMP 配置

```c
void setup_pmp(void) {
    // Entry 0: Bootloader (Locked, R-X)
    write_csr(pmpaddr0, 0x00000000 >> 2);
    write_csr(pmpaddr1, 0x00010000 >> 2);
    write_csr(pmpcfg0, (PMP_TOR << 3) | PMP_R | PMP_X | PMP_L);

    // Entry 1: Flash (R-X)
    write_csr(pmpaddr2, 0x20000000 >> 2);
    write_csr(pmpaddr3, 0x20100000 >> 2);
    uint32_t pmpcfg0_val = read_csr(pmpcfg0);
    pmpcfg0_val |= ((PMP_TOR << 3) | PMP_R | PMP_X) << 8;
    write_csr(pmpcfg0, pmpcfg0_val);

    // Entry 2: Core RAM (RW-)
    write_csr(pmpaddr4, 0x80000000 >> 2);
    write_csr(pmpaddr5, 0x80010000 >> 2);
    pmpcfg0_val |= ((PMP_TOR << 3) | PMP_R | PMP_W) << 16;
    write_csr(pmpcfg0, pmpcfg0_val);

    // Entry 3: App RAM (RW-, U-mode 可存取)
    write_csr(pmpaddr6, 0x80010000 >> 2);
    write_csr(pmpaddr7, 0x80020000 >> 2);
    pmpcfg0_val |= ((PMP_TOR << 3) | PMP_R | PMP_W) << 24;
    write_csr(pmpcfg0, pmpcfg0_val);

    // Entry 4: Peripherals (M-mode only, 不設定 R/W/X)
    write_csr(pmpaddr8, 0x40000000 >> 2);
    write_csr(pmpaddr9, 0x40001000 >> 2);
    write_csr(pmpcfg1, (PMP_TOR << 3));  // 沒有 R/W/X，U-mode 無法存取
}
```

### 7.3 啟動第三方應用

```c
// 第三方應用的入口點
extern void third_party_app_main(void);

// 第三方應用的 Stack
uint8_t app_stack[4096] __attribute__((aligned(16)));

void start_third_party_app(void) {
    // 1. 配置 PMP
    setup_pmp();

    // 2. 啟動應用（切換到 U-mode）
    start_user_task(third_party_app_main, app_stack + sizeof(app_stack));
}
```

### 7.4 第三方應用的 System Call

**第三方應用**：

```c
// U-mode 程式碼
void third_party_app_main(void) {
    // 呼叫 System Call 來存取硬體
    int result = syscall(SYS_GPIO_WRITE, 5, 1);  // GPIO 5 = HIGH

    if (result < 0) {
        // 錯誤處理
    }
}
```

**M-mode System Call Handler**：

```c
int sys_gpio_write(int pin, int value) {
    // 檢查權限（只允許特定的 GPIO）
    if (pin < 0 || pin > 10) {
        return -EINVAL;
    }

    // 存取硬體（M-mode 可以直接存取）
    volatile uint32_t *gpio = (uint32_t *)0x40000000;
    if (value) {
        gpio[pin / 32] |= (1 << (pin % 32));
    } else {
        gpio[pin / 32] &= ~(1 << (pin % 32));
    }

    return 0;
}
```

### 7.5 處理第三方應用的 Exception

**當第三方應用崩潰時**：

```c
void handle_trap(void) {
    uintptr_t mcause_val = read_csr(mcause);
    uintptr_t mepc_val = read_csr(mepc);
    uintptr_t mtval_val = read_csr(mtval);

    if (mcause_val == CAUSE_LOAD_ACCESS_FAULT) {
        // 第三方應用嘗試存取不允許的記憶體
        printf("App crashed: Load Access Fault at 0x%lx (address: 0x%lx)\n",
               mepc_val, mtval_val);

        // 終止應用，返回到核心系統
        terminate_app();

        // 切換回 M-mode 的主程式
        start_core_system();
    }
}
```

---

## 八、總結

### 8.1 核心要點

1. **RISC-V 的三種特權模式**：
   - M-mode（Machine Mode）：最高特權，可以存取所有硬體
   - S-mode（Supervisor Mode）：作業系統核心（例如 Linux）
   - U-mode（User Mode）：使用者應用程式

2. **特權模式的切換**：
   - 從低特權到高特權：`ecall`（Environment Call）
   - 從高特權到低特權：`mret` 或 `sret`
   - 硬體自動切換：Interrupt 或 Exception

3. **M-mode 的特性**：
   - 所有 RISC-V 實作都必須支援 M-mode
   - FreeRTOS 運行在 M-mode（簡單、效能高、硬體存取）
   - M-mode 可以配置 PMP

4. **S-mode 的特性**：
   - 為作業系統設計（Linux Kernel）
   - 支援虛擬記憶體（MMU）
   - 受 M-mode 的 PMP 限制
   - 可以使用 Trap Delegation

5. **U-mode 的特性**：
   - 最低特權模式
   - 不能存取 CSRs（除了 `cycle`, `time`, `instret`）
   - 記憶體存取受 PMP 或 MMU 限制
   - 透過 `ecall` 呼叫 M-mode 服務

6. **PMP (Physical Memory Protection)**：
   - 限制 S-mode 和 U-mode 對記憶體的存取
   - 16 個 Entry（pmp0-pmp15）
   - 支援 TOR, NA4, NAPOT 模式
   - Lock 位元（L）可以鎖定 Entry

7. **實戰應用**：
   - 隔離不可信的第三方程式碼
   - 保護敏感資料（密鑰、憑證）
   - 防止 Stack Overflow 覆蓋其他記憶體

### 8.2 實務建議

**建議 #1：選擇正確的特權模式**

| 應用場景 | 推薦模式 |
|---------|---------|
| Bare-metal 應用 | M-mode |
| 嵌入式 RTOS | M-mode |
| Linux Kernel | S-mode |
| 使用者應用程式 | U-mode |
| 第三方程式碼 | U-mode + PMP |

**建議 #2：使用 PMP 保護關鍵區域**

即使在 M-mode 運行，也可以使用 PMP 來保護 Bootloader 或安全模組（使用 Lock 位元）。

**建議 #3：最小化特權模式切換**

特權模式切換有 Overhead（~100 cycles），應該最小化切換次數。

**建議 #4：使用 Trap Delegation**

如果使用 S-mode，應該配置 Trap Delegation 來減少 M-mode 的介入。

**建議 #5：測試 PMP 配置**

PMP 配置錯誤可能導致系統無法啟動，應該在開發階段充分測試。

### 8.3 回到 Mock Scenario

**問題**：嵌入式系統需要隔離不可信的第三方程式碼。

**解決方案**：

1. 核心系統運行在 M-mode
2. 第三方應用運行在 U-mode
3. 使用 PMP 設定記憶體保護

**結果**：

- 第三方應用無法存取核心系統的記憶體
- 第三方應用無法直接存取硬體
- 第三方應用崩潰時，觸發 Exception，核心系統可以處理

### 8.4 系列總結

**恭喜！你已經完成了整個 Embedded RTOS 系列！**

**我們學到了什麼？**

1. **文章 #1：RTOS 入門**
   - RTOS 的核心概念（Task, Scheduler, Interrupt）
   - FreeRTOS 的基本使用
   - QEMU + RISC-V 環境搭建

2. **文章 #2：Scheduler Deep Dive**
   - Ready List 的實作（Array of Linked Lists）
   - Bitmap 的優化（O(1) 優先權查找）
   - vTaskSwitchContext 的流程

3. **文章 #3：Interrupt Handling**
   - RISC-V 的中斷機制（CLINT, PLIC）
   - Tick Interrupt 的實作
   - 巢狀中斷的討論（CLIC）

4. **文章 #4：Memory Management**
   - 5 種 Heap 方案（heap_1 到 heap_5）
   - 記憶體碎片化問題
   - Lock-Free Memory Pool

5. **文章 #5：GDB + QEMU 除錯**
   - QEMU 啟動參數（-s -S）
   - GDB 基本指令
   - 除錯 FreeRTOS Tasks
   - 常見問題除錯（Stack Overflow, Deadlock）

6. **文章 #6：RTOS SMP**
   - 多核心環境下的挑戰（Race Condition, Cache Coherency）
   - Spinlock 的實作（RISC-V AMO 指令）
   - IPI (Inter-Processor Interrupt)
   - Lock-Free 資料結構

7. **文章 #7：Context Switch 組語**
   - RISC-V 暫存器約定（Caller-Saved vs. Callee-Saved）
   - Context Switch 的完整流程
   - portASM.S 逐行解析
   - 效能優化（Compressed 指令）

8. **文章 #8：特權模式**
   - RISC-V 的三種特權模式（M-mode, S-mode, U-mode）
   - 特權模式切換（ecall, mret）
   - PMP (Physical Memory Protection)
   - 隔離第三方程式碼

**下一步？**

- **實作專案**：使用 FreeRTOS + QEMU + RISC-V 實作一個完整的嵌入式系統
- **深入研究**：閱讀 FreeRTOS 的原始碼，理解更多細節
- **效能優化**：測量和優化系統的效能
- **安全性**：使用 PMP 和特權模式來提升系統的安全性

**感謝閱讀！**

---

## 參考資料

**官方文檔**：

- [RISC-V Privileged Specification](https://riscv.org/specifications/privileged-isa/)
- [RISC-V PMP Specification](https://github.com/riscv/riscv-isa-manual)
- [OpenSBI Documentation](https://github.com/riscv-software-src/opensbi)

**延伸閱讀**：

- [Linux on RISC-V](https://wiki.debian.org/RISC-V)
- [FreeRTOS MPU Support](https://www.freertos.org/FreeRTOS-MPU-memory-protection-unit.html)
- [RISC-V Security](https://riscv.org/technical/security/)

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
