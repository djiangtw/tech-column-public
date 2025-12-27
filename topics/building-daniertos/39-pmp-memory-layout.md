# 記憶體地圖：從規劃到實作的設計之路

**作者**: Danny Jiang
**日期**: 2025-12-26

---

## 前言：先磨斧頭

> 「給我六小時砍一棵樹，我會花前四小時磨斧頭。」—— Abraham Lincoln

上一篇我們踩了五個坑，每次都是改了一個地方、跑一次、又 fault、再改。這種「trial and error」的方式效率太低。

這一篇，我們要換個角度：**先設計，後實作**。

從需求分析開始，畫出記憶體地圖，計算 NAPOT 對齊，最後才寫 code。這樣可以一次把事情做對（好吧，至少減少踩坑次數）。

---

## 一、需求分析

在開始畫圖之前，先釐清我們需要什麼：

### 1.1 區域類型

| 區域 | 內容 | M-mode 權限 | U-mode 權限 |
|------|------|-------------|-------------|
| Kernel Code | .text | RWX | - |
| Kernel Data | .bss, globals | RW | - |
| Shared Data | .rodata（字串常量）| R | R |
| User Code | .user_text | RW | RX |
| User Data | .user_data（stacks）| RW | RW |
| Kernel Stacks | per-core stacks | RW | RW |

### 1.2 平台限制

- **RAM**：0x80000000 - 0x88000000（128MB，QEMU virt）
- **PMP Entries**：8 個（QEMU 實作）
- **NAPOT 最小單位**：4KB（實際上）

### 1.3 大小估算

| 區域 | 預估大小 | NAPOT Size | 對齊要求 |
|------|----------|------------|----------|
| Kernel Code | ~10KB | 16KB | 16KB |
| Kernel Data | ~100KB | 128KB | 128KB |
| Shared Data | ~4KB | 4KB | 4KB |
| User Code | ~2KB | 4KB | 4KB |
| User Data | 64KB (8 tasks × 8KB) | 64KB | 64KB |
| User Region (合併) | ~68KB | 128KB | 128KB |
| Kernel Stacks | 512KB (8 cores × 64KB) | 512KB | 512KB |

---

## 二、畫出記憶體地圖

### 2.1 初步佈局

根據 NAPOT 對齊要求，我們需要把相近的區域合併，減少碎片：

```
0x80000000 ┌─────────────────────────────────────┐
           │                                     │
           │  Kernel Region                      │
           │  (.text + gap + .rodata + .data     │
           │   + .bss)                           │
           │                                     │
           │  不需要 U-mode 存取                  │
           │  （除了 .rodata）                    │
           │                                     │
0x80020000 ├─────────────────────────────────────┤ ← 128KB 對齊
           │                                     │
           │  User Region                        │
           │  (.user_text + .user_data)          │
           │                                     │
           │  U-mode RWX                         │
           │                                     │
0x80040000 ├─────────────────────────────────────┤ ← 128KB 對齊
           │                                     │
           │  （未使用空間）                       │
           │                                     │
0x87F80000 ├─────────────────────────────────────┤ ← 512KB 對齊
           │                                     │
           │  Kernel Stacks                      │
           │  8 cores × 64KB                     │
           │                                     │
           │  U-mode RW（任務執行時使用）          │
           │                                     │
0x88000000 └─────────────────────────────────────┘
```

### 2.2 處理 Shared Region

問題：`.rodata` 在 Kernel Region 裡面，但 User 需要讀取字串常量。

解決方案：在 Kernel Region 內部劃出一個 Shared 子區域：

```
0x80000000 ┌─────────────────────────────────────┐
           │  .text (Kernel Code)                │ ~9KB
0x800023FC ├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┤
           │  Gap (padding to 4KB)               │
0x80003000 ├─────────────────────────────────────┤ ← 4KB 對齊
           │  .rodata + .data (Shared)           │ ← U-mode R
           │  4KB NAPOT                          │
0x80004000 ├─────────────────────────────────────┤
           │  .bss (Kernel Globals)              │ ~84KB
0x80018B30 ├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┤
           │  Gap (padding to 128KB)             │
0x80020000 └─────────────────────────────────────┘
```

**關鍵洞察**：PMP entries 會按順序檢查，第一個匹配的 entry 決定權限。我們可以：

1. Entry 0：User Region（128KB NAPOT，RWX）
2. Entry 1：Shared Region（4KB NAPOT，R）
3. Entry 2：Stack Region（512KB NAPOT，RW）

Kernel 區域**不需要 PMP entry**！因為 PMP 的預設規則是：
- M-mode：如果沒有匹配的 entry，允許存取
- U-mode：如果沒有匹配的 entry，拒絕存取

所以 Kernel 區域自動受保護。

---

## 三、計算 NAPOT 地址

### 3.1 NAPOT 編碼公式

NAPOT 的 pmpaddr 編碼有點反直覺：

```
pmpaddr = (base + (size/2 - 1)) >> 2
```

這個編碼方式讓硬體可以用簡單的位元運算來檢查地址是否在範圍內。

### 3.2 計算各區域

**User Region (128KB at 0x80020000)**：
```
base = 0x80020000
size = 0x20000 (128KB)
pmpaddr = (0x80020000 + 0x10000 - 1) >> 2
        = 0x8002FFFF >> 2
        = 0x2000BFFF
```

**Shared Region (4KB at 0x80003000)**：
```
base = 0x80003000
size = 0x1000 (4KB)
pmpaddr = (0x80003000 + 0x800 - 1) >> 2
        = 0x800037FF >> 2
        = 0x20000DFF
```

**Stack Region (512KB at 0x87F80000)**：
```
base = 0x87F80000
size = 0x80000 (512KB)
pmpaddr = (0x87F80000 + 0x40000 - 1) >> 2
        = 0x87FBFFFF >> 2
        = 0x21FEFFFF
```

### 3.3 C Code 實作

把公式封裝成函數：

```c
/**
 * 計算 NAPOT 模式的 pmpaddr 值
 * @param base  區域起始地址（必須對齊到 size）
 * @param size  區域大小（必須是 2 的冪次方）
 */
static inline uint64_t pmp_napot_addr(uint64_t base, uint64_t size)
{
    // 驗證對齊
    assert((base & (size - 1)) == 0);
    // 驗證 size 是 2 的冪次方
    assert((size & (size - 1)) == 0);
    assert(size >= 8);

    return (base + (size / 2 - 1)) >> 2;
}
```

---

## 四、Linker Script 設計

有了記憶體地圖，現在來寫 linker script。

### 4.1 記憶體區域定義

```ld
MEMORY
{
    RAM (rwx) : ORIGIN = 0x80000000, LENGTH = 128M
}
```

我們用一個大的 RAM 區域，然後用 section 配置來控制佈局。

### 4.2 Kernel Sections

```ld
SECTIONS
{
    /* ══════════════════════════════════════════
     * Kernel Region: 0x80000000 - 0x80020000
     * ══════════════════════════════════════════ */

    /* Kernel Code */
    .text ORIGIN(RAM) : {
        _text_start = .;
        KEEP(*(.text.entry))    /* Entry point first */
        *(.text .text.*)
        _text_end = .;
    } > RAM

    /* Shared Region: 4KB NAPOT for U-mode read access */
    . = ALIGN(4096);
    .rodata : {
        _shared_start = .;
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    } > RAM

    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
        . = ALIGN(4096);
        _shared_end = .;
    } > RAM

    /* BSS (Kernel Globals) */
    .bss (NOLOAD) : {
        _bss_start = .;
        *(.bss .bss.*)
        *(.sbss .sbss.*)
        _bss_end = .;
    } > RAM
}
```

### 4.3 User Sections

關鍵：使用 `ALIGN(128K)` 確保 NAPOT 對齊：

```ld
    /* ══════════════════════════════════════════
     * User Region: 128KB NAPOT at 0x80020000
     * ══════════════════════════════════════════ */

    . = ALIGN(128K);
    _user_section_start = .;

    .user_text : {
        _user_text_start = .;
        *(.user_text .user_text.*)
        _user_text_end = .;
    } > RAM

    .user_data : {
        _user_data_start = .;
        *(.user_data .user_data.*)
        _user_data_end = .;
    } > RAM

    . = ALIGN(4096);
    _user_section_end = .;
```

### 4.4 Stack Region

Stack 放在 RAM 尾端，對齊到 512KB：

```ld
    /* ══════════════════════════════════════════
     * Stack Region: 512KB NAPOT at RAM end
     * ══════════════════════════════════════════ */

    . = ORIGIN(RAM) + LENGTH(RAM) - (8 * 64K);  /* 8 cores × 64KB */
    . = ALIGN(512K);  /* NAPOT alignment */

    _stack_bottom = .;
    . += 8 * 64K;
    _stack_top = .;
```

### 4.5 導出 PMP 符號

讓 C code 可以讀取這些邊界：

```ld
    /* ══════════════════════════════════════════
     * PMP Symbols for C code
     * ══════════════════════════════════════════ */

    _pmp_user_start   = _user_section_start;
    _pmp_user_end     = _user_section_end;
    _pmp_shared_start = _shared_start;
    _pmp_shared_end   = _shared_end;
    _pmp_stacks_start = _stack_bottom;
    _pmp_stacks_end   = _stack_top;
```

---

## 五、PMP 初始化程式碼

### 5.1 計算 NAPOT Size

Linker 只給我們 start/end，需要計算 size 並 round up：

```c
static uint64_t next_power_of_2(uint64_t n)
{
    if (n == 0) return 1;
    n--;
    n |= n >> 1;
    n |= n >> 2;
    n |= n >> 4;
    n |= n >> 8;
    n |= n >> 16;
    n |= n >> 32;
    return n + 1;
}
```

### 5.2 完整的 pmp_init

```c
void pmp_init(void)
{
    extern char _pmp_user_start[], _pmp_user_end[];
    extern char _pmp_shared_start[], _pmp_shared_end[];
    extern char _pmp_stacks_start[], _pmp_stacks_end[];

    /* ── Entry 0: User Region (RWX) ── */
    uint64_t user_base = (uint64_t)_pmp_user_start;
    uint64_t user_size = (uint64_t)_pmp_user_end - user_base;
    uint64_t user_napot = next_power_of_2(user_size);

    // 驗證對齊
    if (user_base % user_napot != 0) {
        panic("User region not aligned to NAPOT size!");
    }

    csr_write(CSR_PMPADDR0, pmp_napot_addr(user_base, user_napot));

    /* ── Entry 1: Shared Region (R only) ── */
    uint64_t shared_base = (uint64_t)_pmp_shared_start;
    uint64_t shared_size = (uint64_t)_pmp_shared_end - shared_base;
    uint64_t shared_napot = next_power_of_2(shared_size);

    csr_write(CSR_PMPADDR1, pmp_napot_addr(shared_base, shared_napot));

    /* ── Entry 2: Stack Region (RW) ── */
    uint64_t stacks_base = (uint64_t)_pmp_stacks_start;
    uint64_t stacks_size = (uint64_t)_pmp_stacks_end - stacks_base;
    uint64_t stacks_napot = next_power_of_2(stacks_size);

    csr_write(CSR_PMPADDR2, pmp_napot_addr(stacks_base, stacks_napot));

    /* ── 設定權限 ── */
    uint64_t cfg = 0;
    cfg |= (PMP_A_NAPOT | PMP_R | PMP_W | PMP_X) << (0 * 8);  // User: RWX
    cfg |= (PMP_A_NAPOT | PMP_R) << (1 * 8);                   // Shared: R
    cfg |= (PMP_A_NAPOT | PMP_R | PMP_W) << (2 * 8);           // Stacks: RW

    csr_write(CSR_PMPCFG0, cfg);

    log_printf("[PMP] Initialized:\n");
    log_printf("  User:   0x%08lx - 0x%08lx (NAPOT %ldKB)\n",
               user_base, user_base + user_napot, user_napot / 1024);
    log_printf("  Shared: 0x%08lx - 0x%08lx (NAPOT %ldKB)\n",
               shared_base, shared_base + shared_napot, shared_napot / 1024);
    log_printf("  Stacks: 0x%08lx - 0x%08lx (NAPOT %ldKB)\n",
               stacks_base, stacks_base + stacks_napot, stacks_napot / 1024);
}
```

---

## 六、驗證設計

### 6.1 編譯後檢查

```bash
$ riscv64-unknown-elf-objdump -h build/daniertos.elf

Sections:
Idx Name          Size      VMA
  0 .text         000023fc  0000000080000000
  1 .rodata       00000924  0000000080003000  ← 4KB 對齊 ✓
  2 .data         000006dc  0000000080003924
  3 .bss          00014b30  0000000080004000
  4 .user_text    00000404  0000000080020000  ← 128KB 對齊 ✓
  5 .user_data    00010000  0000000080020410
```

### 6.2 符號檢查

```bash
$ riscv64-unknown-elf-nm build/daniertos.elf | grep _pmp

0000000080003000 R _pmp_shared_start
0000000080004000 D _pmp_shared_end     ← size = 4KB ✓
0000000080020000 A _pmp_user_start
0000000080040000 A _pmp_user_end       ← 範圍足夠 ✓
0000000087f80000 A _pmp_stacks_start   ← 512KB 對齊 ✓
0000000088000000 A _pmp_stacks_end
```

### 6.3 執行時驗證

```
[PMP] Initialized:
  User:   0x80020000 - 0x80040000 (NAPOT 128KB)
  Shared: 0x80003000 - 0x80004000 (NAPOT 4KB)
  Stacks: 0x87f80000 - 0x88000000 (NAPOT 512KB)
```

### 6.4 功能測試

```
[BadTask] Started in U-mode
[BadTask] Counting down to crash...
[BadTask] 3...
[BadTask] 2...
[BadTask] 1...
[BadTask] Attempting to access kernel memory...
[Fault] mcause=7, mtval=0x80000000  ← Store fault, 正確！
[Fault] Terminating user task 'BadTask'
[GoodTask] Still running... 5
[GoodTask] Still running... 6
...
[GoodTask] Done! (System remained stable)  ← Fault isolation 成功！
```

---

## 七、移植檢查清單

當你要移植到不同平台時，檢查這些項目：

### 7.1 平台參數

| 參數 | QEMU virt | 你的平台 |
|------|-----------|----------|
| RAM 起始 | 0x80000000 | ? |
| RAM 大小 | 128MB | ? |
| PMP entries | 8 | ? |
| 核心數 | 2-8 | ? |

### 7.2 NAPOT 對齊計算

```python
def plan_layout(ram_start, ram_size, num_cores, max_user_tasks):
    """計算記憶體布局"""

    # Stack region
    stack_per_core = 64 * 1024  # 64KB
    total_stacks = num_cores * stack_per_core
    stack_napot = next_power_of_2(total_stacks)
    stack_base = (ram_start + ram_size - stack_napot) & ~(stack_napot - 1)

    # User region (假設每 task 8KB stack)
    user_stack_total = max_user_tasks * 8 * 1024
    user_code_estimate = 4 * 1024  # 4KB
    user_total = user_stack_total + user_code_estimate
    user_napot = next_power_of_2(user_total)

    # 找一個對齊的位置
    # 必須在 kernel 之後，stack 之前
    # 假設 kernel 大約 256KB
    user_base = (ram_start + 256 * 1024)
    user_base = (user_base + user_napot - 1) & ~(user_napot - 1)  # 對齊

    print(f"Kernel:  0x{ram_start:08x} - 0x{user_base:08x}")
    print(f"User:    0x{user_base:08x} - 0x{user_base + user_napot:08x} ({user_napot // 1024}KB NAPOT)")
    print(f"Stacks:  0x{stack_base:08x} - 0x{stack_base + stack_napot:08x} ({stack_napot // 1024}KB NAPOT)")

    return {
        'user_base': user_base,
        'user_napot': user_napot,
        'stack_base': stack_base,
        'stack_napot': stack_napot,
    }

# 範例
plan_layout(0x80000000, 128 * 1024 * 1024, 4, 8)
```

### 7.3 Linker Script 修改

1. 調整 `MEMORY` 區塊的 ORIGIN 和 LENGTH
2. 調整 ALIGN() 參數符合你的 NAPOT 需求
3. 調整 stack 區域大小

### 7.4 Config 修改

```c
// config.h
#define CONFIG_SMP_MAX_CORES    4       // 你的核心數
#define MAX_USER_TASKS          8       // User tasks 數量
#define USER_USTACK_SIZE        (8*1024) // 每 task stack 大小
```

---

## 八、常見問題

### Q1: 為什麼不用 TOR mode？

TOR (Top of Range) 模式更靈活，不需要 2^n 對齊。但：
- 每個區域需要 2 個 PMP entries（start 和 end）
- PMP entries 數量有限（通常 8-16 個）
- NAPOT 更省 entries

### Q2: 可以讓每個 Task 有獨立的 PMP 嗎？

可以，但需要：
- 每次 context switch 時更新 PMP
- 每個 task 需要 1-2 個 PMP entries
- 8 個 entries 大約支援 4-8 個獨立 task

### Q3: 如果 User code 超過 NAPOT size 怎麼辦？

兩個選擇：
1. 增加 NAPOT size（例如 128KB → 256KB）
2. 分成多個 PMP entries

---

## 總結

這一篇我們用「設計優先」的方法來規劃 PMP 記憶體保護：

| 步驟 | 重點 |
|------|------|
| 需求分析 | 列出所有區域類型和權限需求 |
| 記憶體地圖 | 畫出視覺化的布局圖 |
| NAPOT 計算 | 計算每個區域的 size 和對齊 |
| Linker Script | 用 ALIGN() 確保對齊 |
| 驗證 | objdump, nm, 執行測試 |

**經驗法則**：

1. **先畫圖**：在寫任何 code 之前，畫出記憶體地圖
2. **計算 NAPOT**：知道 round up 到多大，確保 base 對齊
3. **Linker Script 控制一切**：用 ALIGN() 和 section attribute
4. **導出符號**：讓 C code 可以讀取邊界
5. **驗證、驗證、驗證**：objdump, nm, runtime log

希望這兩篇文章能幫你在實作 PMP 時少踩一些坑。記住：**花時間在設計上，比花時間在除錯上更值得**。

---

## 參考資料

**RISC-V 規格**

- [RISC-V Privileged Specification v1.12](https://riscv.org/specifications/privileged-isa/)
  Chapter 3.7: Physical Memory Protection

**danieRTOS 相關**

- Ch 16: PMP 基礎概念
- Ch 34: SMP 下的 PMP 配置
- Ch 38: NAPOT 對齊的五個陷阱
- [docs/PMP_DESIGN.md](code.v3/docs/PMP_DESIGN.md): 完整技術文件

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)

