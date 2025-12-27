# 血淚教訓：NAPOT 對齊的五個陷阱

**作者**: Danny Jiang
**日期**: 2025-12-26

---

## 前言：為什麼 User Task 還是 fault？

> 「我已經設定了 PMP，為什麼 User Task 還是 fault？」

這是我在開發 danieRTOS v3p2 時，對著螢幕說的最多的一句話。

PMP（Physical Memory Protection）的概念很簡單：設定幾個記憶體區域，給每個區域不同的權限。但魔鬼藏在細節裡——特別是 **NAPOT（Naturally Aligned Power-Of-Two）** 模式的對齊要求。

這篇文章記錄了我在實作 Fault Isolation 時踩過的五個坑，希望能幫你少走一些彎路。

---

## 場景：一個看似正確的 Demo

我寫了一個簡單的 fault isolation demo：

```c
USER_FUNC static void good_task(void *arg) {
    for (int i = 0; i < 10; i++) {
        user_printf("[GoodTask] Still running... %d\n", i);
        user_delay(100);
    }
    user_printf("[GoodTask] Done!\n");
}

USER_FUNC static void bad_task(void *arg) {
    user_printf("[BadTask] Started\n");
    
    // 嘗試存取 kernel 記憶體
    volatile uint64_t *kernel = (uint64_t *)0x80000000;
    *kernel = 0xDEAD;  // 這應該觸發 fault
    
    user_printf("[BadTask] This should never print\n");
}
```

預期結果：BadTask 觸發 fault 被終止，GoodTask 繼續運行。

實際結果：

```
[BadTask] Started
[Fault] mcause=5, mtval=0x80003170, mepc=0x800201be
```

等等，`mcause=5` 是 **Load Access Fault**，不是 Store。而且 `mtval=0x80003170` 根本不是 `0x80000000`！

Fault 發生在 `user_printf` 裡面，不是在存取 kernel 記憶體時。

**問題出在哪？**

---

## 陷阱 1：字串常量住在 .rodata

第一個領悟：C 語言的字串常量不在你的函數裡。

```c
user_printf("[BadTask] Started\n");
//          ^^^^^^^^^^^^^^^^^^^^^^^^
//          這個字串在哪裡？
```

讓我們用 `objdump` 看看：

```bash
$ riscv64-unknown-elf-objdump -s -j .rodata build/daniertos.elf | grep -A2 "BadTask"
80003170 5b426164 5461736b 5d205374 61727465  [BadTask] Starte
80003180 6420696e 20552d6d 6f64650a 00000000  d in U-mode.....
```

字串 `"[BadTask] Started"` 在 `0x80003170`——這是 `.rodata` section，在 **Kernel 區域**！

當 User Task 嘗試讀取這個字串時，PMP 拒絕了存取，觸發 Load Access Fault。

### 解決方案：Shared Region

我們需要讓 User Mode 可以讀取 `.rodata`：

```c
// PMP Entry 1: Shared region (Read-only)
uint64_t shared_base = (uint64_t)&_pmp_shared_start;
uint64_t shared_size = (uint64_t)&_pmp_shared_end - shared_base;
uint64_t napot_addr = pmp_napot_addr(shared_base, shared_size);
csr_write(CSR_PMPADDR1, napot_addr);

// 權限：R（只讀）
cfg |= (PMP_A_NAPOT | PMP_R) << (1 * 8);
```

---

## 陷阱 2：NAPOT 的對齊噩夢

修改後再跑一次：

```
[Fault] mcause=5, mtval=0x80003170, mepc=0x800201be
```

還是 fault！讓我看看 PMP 實際配置了什麼：

```c
void pmp_dump(void) {
    uint64_t addr0 = csr_read(CSR_PMPADDR0);
    // ... 印出所有 PMP entries
}
```

輸出：

```
PMP Entry 0: addr=0x80020000, size=128KB, perm=RWX (User region)
PMP Entry 1: addr=0x80000000, size=128KB, perm=R   (Shared region) ← 問題！
```

等等，Shared region 的 base 是 `0x80000000`？我明明設的是 `0x80003000`！

這就是 **NAPOT 對齊的陷阱**。

### NAPOT 的對齊規則

NAPOT 要求：
1. **Size 必須是 2 的冪次方**（如 4KB, 8KB, 16KB...）
2. **Base 必須對齊到 Size**

如果你的區域是：
- base = 0x80003000
- size = 0x1000 (4KB)

這是合法的，因為 0x80003000 % 0x1000 == 0。

但如果：
- base = 0x80003000  
- size = 0x11000 (68KB，不是 2 的冪次方)

NAPOT 會 **round up** size 到下一個 2 的冪次方：128KB (0x20000)。

然後問題來了：0x80003000 % 0x20000 ≠ 0！

NAPOT 會把 base **向下對齊**到 0x80000000，結果：

```
預期覆蓋：0x80003000 - 0x80014000
實際覆蓋：0x80000000 - 0x80020000  ← 包含了 kernel code！
```

### 解決方案：在 Linker Script 中強制對齊

```ld
/* 確保 User section 開始於 128KB 對齊的地址 */
. = ALIGN(128K);
_user_section_start = .;

.user_text : {
    *(.user_text .user_text.*)
} > RAM
```

這樣 base 就會自動對齊到可能的 NAPOT size。

---

## 陷阱 3：Linker Script 符號定義位置

我在 linker script 中這樣定義符號：

```ld
. = ALIGN(4096);
_rodata_start = .;
.rodata : {
    *(.rodata .rodata.*)
} > RAM
_rodata_end = .;
```

然後用 `nm` 檢查：

```bash
$ riscv64-unknown-elf-nm build/daniertos.elf | grep rodata
0000000080003000 R _rodata_end    ← 等等，end 和 start 一樣？
0000000080003000 R _rodata_start
```

`_rodata_start` 和 `_rodata_end` 都是 `0x80003000`！Size 是 0！

問題在於：**符號定義在 section 外面時，它的值是當前位置計數器的值**，而不是 section 結束後的值。

### 解決方案：在 Section 內部定義符號

```ld
.rodata ALIGN(4096) : {
    _rodata_start = .;      /* 在 section 開始處 */
    *(.rodata .rodata.*)
    _rodata_end = .;        /* 在 section 結束處 */
} > RAM
```

或者，如果需要跨越多個 sections：

```ld
.rodata ALIGN(4096) : {
    _shared_start = .;
    *(.rodata .rodata.*)
} > RAM

.data : {
    *(.data .data.*)
    . = ALIGN(4096);
    _shared_end = .;
} > RAM
```

---

## 陷阱 4：User Stack 放錯地方

PMP 設定正確了，字串也可以讀取了，但現在換成另一個 fault：

```
[Fault] mcause=7, mtval=0x80006040, mepc=0x80020102
```

`mcause=7` 是 Store Access Fault。地址 `0x80006040` 是什麼？

```bash
$ riscv64-unknown-elf-nm build/daniertos.elf | grep 80006
0000000080006040 b g_user_ustacks
```

這是 User Stack 的陣列！它被放在 `.bss` section——Kernel 區域！

原本的定義：

```c
// task.c
static uint8_t g_user_ustacks[MAX_USER_TASKS][USER_USTACK_SIZE];
```

編譯器把它放在 `.bss`，這是 Kernel 區域，User Mode 無法存取。

### 解決方案：Section Attribute

```c
// 使用 section attribute 強制放到 .user_data
static uint8_t g_user_ustacks[MAX_USER_TASKS][USER_USTACK_SIZE]
    __attribute__((section(".user_data"), aligned(16)));
```

並在 linker script 中確保 `.user_data` 在 User Region：

```ld
.user_data : {
    _user_data_start = .;
    *(.user_data .user_data.*)
    _user_data_end = .;
} > RAM
```

---

## 陷阱 5：函數本體也需要在 User Region

User Stack 修好了，再跑一次：

```
[Fault] mcause=1, mtval=0x80001234, mepc=0x80001234
```

`mcause=1` 是 **Instruction Access Fault**！User Mode 嘗試執行 `0x80001234` 的程式碼，但那是 Kernel 區域。

問題：`good_task` 和 `bad_task` 函數在哪裡？

```bash
$ riscv64-unknown-elf-nm build/daniertos.elf | grep task
0000000080001234 T good_task    ← 在 .text (Kernel)
0000000080001300 T bad_task     ← 在 .text (Kernel)
```

函數被放在 `.text` section，不是 `.user_text`！

### 解決方案：USER_FUNC Macro

定義一個 macro 來標記 User 函數：

```c
// user_syscall.h
#define USER_FUNC __attribute__((section(".user_text")))
#define USER_DATA __attribute__((section(".user_data")))
```

使用：

```c
USER_FUNC static void good_task(void *arg) {
    // 這個函數會被放在 .user_text
    user_printf("[GoodTask] Running...\n");
}
```

**重要提醒**：所有 User Task 呼叫的函數都需要 `USER_FUNC`！包括：
- Task 主函數
- User-mode 的 helper 函數
- 但 **不包括** syscall wrapper（它們會透過 `ecall` 進入 Kernel）

---

## 最終的記憶體布局

經過五個陷阱的洗禮，最終正確的記憶體布局是：

```
┌─────────────────────────────────────────────────────────────┐
│                    RAM: 0x80000000 - 0x88000000             │
├─────────────────────────────────────────────────────────────┤
│ 0x80000000 │ .text (Kernel Code)        │ M-mode only      │
│            │ ~9KB                        │                  │
├────────────┼────────────────────────────┼──────────────────┤
│ 0x80003000 │ .rodata + .data (Shared)   │ U-mode R         │
│            │ 4KB NAPOT                   │ (字串常量)       │
├────────────┼────────────────────────────┼──────────────────┤
│ 0x80004000 │ .bss (Kernel Globals)      │ M-mode only      │
│            │ ~84KB                       │                  │
├────────────┼────────────────────────────┼──────────────────┤
│ 0x80020000 │ .user_text + .user_data    │ U-mode RWX       │
│            │ 128KB NAPOT                 │ (User 程式+Stack)│
├────────────┼────────────────────────────┼──────────────────┤
│ 0x87F80000 │ Kernel Stacks              │ U-mode RW        │
│            │ 512KB NAPOT                 │ (任務執行時)     │
└────────────┴────────────────────────────┴──────────────────┘
```

---

## 除錯工具箱

當你遇到 PMP 問題時，這些工具很有用：

### 1. 理解 mcause

| mcause | 意義 | 可能原因 |
|--------|------|----------|
| 1 | Instruction access fault | User 嘗試執行 Kernel code |
| 5 | Load access fault | User 嘗試讀取 Kernel data |
| 7 | Store access fault | User 嘗試寫入 Kernel data |

### 2. 用 objdump 檢查 sections

```bash
$ riscv64-unknown-elf-objdump -h build/daniertos.elf
  0 .text         000023fc  0000000080000000
  1 .rodata       00000924  0000000080003000
  2 .data         000006dc  0000000080003924
  3 .bss          00014b30  0000000080004000
  4 .user_text    00000404  0000000080020000
  5 .user_data    00010000  0000000080020410
```

### 3. 用 nm 找符號地址

```bash
$ riscv64-unknown-elf-nm build/daniertos.elf | grep _pmp
0000000080003000 R _pmp_shared_start
0000000080004000 D _pmp_shared_end
0000000080020000 A _pmp_user_start
0000000080040000 A _pmp_user_end
```

### 4. 驗證 NAPOT 對齊

```python
def check_napot(base, size):
    # Round up size to power of 2
    napot_size = 1 << (size - 1).bit_length()

    if base % napot_size != 0:
        print(f"ERROR: 0x{base:x} not aligned to 0x{napot_size:x}")
        actual_base = base & ~(napot_size - 1)
        print(f"Actual: 0x{actual_base:x} - 0x{actual_base + napot_size:x}")
    else:
        print(f"OK: 0x{base:x}, size=0x{napot_size:x}")

check_napot(0x80020000, 68 * 1024)  # 68KB user region
# OK: 0x80020000, size=0x20000 (128KB)
```

### 5. Dump PMP 狀態

```c
void pmp_dump(void) {
    log_printf("PMP Configuration:\n");

    uint64_t cfg0 = csr_read(CSR_PMPCFG0);
    uint64_t addr0 = csr_read(CSR_PMPADDR0);

    for (int i = 0; i < 8; i++) {
        uint8_t cfg = (cfg0 >> (i * 8)) & 0xFF;
        uint64_t addr = /* read pmpaddr[i] */;

        if ((cfg & PMP_A_MASK) == 0) continue;  // Entry disabled

        // Decode NAPOT
        uint64_t base, size;
        napot_decode(addr, &base, &size);

        log_printf("  Entry %d: base=0x%08lx, size=0x%lx, perm=%c%c%c\n",
            i, base, size,
            (cfg & PMP_R) ? 'R' : '-',
            (cfg & PMP_W) ? 'W' : '-',
            (cfg & PMP_X) ? 'X' : '-');
    }
}
```

---

## 經驗法則

經過這些踩坑，我總結出幾個經驗法則：

### 1. 先畫圖，再寫 Code

在修改 linker script 或 PMP 設定之前，先畫出記憶體地圖。包括：
- 每個 section 的預期地址範圍
- 每個 PMP entry 的覆蓋範圍
- 對齊要求

### 2. NAPOT Size 決定一切

記住公式：**NAPOT size = next_power_of_2(actual_size)**

如果你的區域是 68KB，NAPOT 會用 128KB。確保 base 對齊到 128KB！

### 3. 檢查每個 Section 的位置

用 `objdump -h` 確認：
- `.text` 在哪裡？（Kernel code）
- `.rodata` 在哪裡？（字串常量）
- `.user_text` 在哪裡？（User code）
- `.user_data` 在哪裡？（User stack）

### 4. 使用 Section Attributes

明確標記每個 User 函數和資料：

```c
USER_FUNC void my_task(void *arg);
USER_DATA static int my_data;
```

### 5. 在 Linker Script 中定義 PMP 符號

讓 C code 可以讀取 section 邊界：

```ld
_pmp_user_start = _user_section_start;
_pmp_user_end   = _user_section_end;
```

---

## 總結

PMP 的概念很簡單，但實作時有很多細節容易出錯。這篇文章記錄的五個陷阱：

| 陷阱 | 症狀 | 解決方案 |
|------|------|----------|
| 字串在 .rodata | mcause=5, 讀取字串時 fault | 建立 Shared Region |
| NAPOT 對齊錯誤 | PMP 覆蓋範圍不對 | Linker script 對齊 |
| 符號定義位置 | Start/End 相同 | 在 section 內部定義 |
| Stack 在 Kernel | mcause=7, 寫入 stack 時 | Section attribute |
| 函數在 Kernel | mcause=1, 執行時 fault | USER_FUNC macro |

下一篇，我們會從設計角度來看這個問題：如何規劃一個正確的記憶體布局，讓 PMP 和 Linker Script 協同工作。

---

## 參考資料

**RISC-V 規格**

- [RISC-V Privileged Specification v1.12](https://riscv.org/specifications/privileged-isa/)
  Chapter 3.7: Physical Memory Protection

**danieRTOS 系列**

- Ch 16: PMP 基礎概念
- Ch 34: SMP 下的 PMP 配置

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)

