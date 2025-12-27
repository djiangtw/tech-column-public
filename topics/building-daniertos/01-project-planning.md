# 專案規劃：為什麼要自己寫 RTOS

**作者**: Danny Jiang
**日期**: 2025-12-13

---

## 前言：從學習者到實作者

你有沒有過這種經驗？看完一本技術書籍，覺得自己全懂了，但當別人問你「那你能不能自己寫一個？」時，突然語塞。

2024 年底，我就遇到了這種尷尬。

當時我剛完成 FreeRTOS 系列的八篇技術文章，涵蓋了 Scheduler、Context Switch、Interrupt Handling、Memory Management 等核心主題。我用 QEMU + RISC-V 搭建了學習環境，用 GDB 一步步追蹤 FreeRTOS 的執行流程，甚至深入到組合語言層級分析 Context Switch 的每一條指令。

我以為我懂了。

然後有一天，我問自己：**我可以解釋 FreeRTOS 的每一個細節，但我真的能從零開始寫一個 RTOS 嗎？**

這個問題困擾了我好幾天。最後，我決定用最直接的方式來回答：**動手寫一個。**

不是為了取代 FreeRTOS，也不是為了商業用途。純粹是為了證明一件事：**讀過文章是理解，親手實作才是真正掌握。**

於是，**danieRTOS** 誕生了。

> 💡 **給讀者**：如果你也有同樣的疑問，這個系列就是為你而寫的。我們會從第一行組合語言開始，一步步打造一個可以運行的 RTOS。

本文是 Building danieRTOS 系列的第一篇，我將說明專案的目標、設計哲學、技術選擇，以及開發環境的設定。讀完這篇文章，你將能夠：

- 理解 danieRTOS 的設計目標和取捨
- 了解為什麼選擇 RISC-V + QEMU 作為開發平台
- 掌握 Minimal RTOS 的核心功能清單
- 設定完整的開發環境

---

## 一、為什麼要自己寫 RTOS？

### 1.1 學習的最佳方式

學習 RTOS 有很多方式：

- **讀書**：理解概念，但容易流於表面
- **讀 Code**：看到實作細節，但容易迷失在龐大的 codebase
- **寫文章**：整理思路，但仍是「解讀」別人的作品
- **自己寫**：從零開始，每一行 code 都是自己的決策

自己寫 RTOS 的價值在於：**你必須回答所有「為什麼」。**

為什麼 TCB 要這樣設計？為什麼 Context Switch 要保存這些暫存器？為什麼 Scheduler 要用 Priority-based 而不是 Round-Robin？

當你是使用者，這些問題可以跳過。當你是實作者，每一個問題都必須有答案。

### 1.2 教育用途 vs 商業用途

**danieRTOS 的定位是教育用途**，這意味著設計原則與商業 RTOS（如 FreeRTOS、Zephyr）截然不同：

| 面向 | 商業 RTOS | danieRTOS |
|------|-----------|-----------|
| **首要目標** | 效能、可移植性 | 可讀性、易理解 |
| **程式碼風格** | 大量巨集、匈牙利命名法 | 清晰的函數、現代命名法 |
| **資料結構** | Opaque pointers（隱藏實作） | 透明結構（方便學習） |
| **平台支援** | 數十種 MCU | 只支援 RISC-V (QEMU) |
| **功能完整性** | 完整的 IPC、Timer、Hook | Minimal 核心功能 |

**簡單來說：danieRTOS 犧牲效能和移植性，換取可讀性和易理解性。**

### 1.3 danieRTOS 的名字由來

**danieRTOS = Daniel + RTOS**

這是一個帶有個人品牌的專案名稱。就像 Linus 有 Linux，Richard 有 GNU，我有 danieRTOS。

名字不重要，重要的是：**這是我的作品，我為它負責。**

---

## 二、核心功能：定義 MVP

對於教育用途的 Minimal RTOS，我們需要區分「核心必要功能」與「擴充功能」。

### 2.1 Phase 1：核心功能（Must-Have）

要讓 RTOS 能動起來並證明其價值，至少需要以下模組：

**1. Task 管理**

- **TCB (Task Control Block)**：定義 Task 的資料結構（Stack Pointer、Priority、State、Name）
- **Task 創建**：分配記憶體、初始化 Stack（偽造 Context）
- **Task 切換**：從一個 Task 切換到另一個 Task

**2. Scheduler**

- 決定「下一個執行誰」
- 實作 **Fixed Priority Preemptive Scheduling**（固定優先級搶佔式排程）
- 支援 **Round-Robin** 對相同優先級的 Task 輪轉

**3. Context Switch**

- **Port Layer（Assembly）**：這是 RTOS 的心臟
- 在 RISC-V 上，負責保存 `x1`-`x31`、`mstatus`、`mepc` 到 Stack
- 切換 Stack Pointer (`sp`)，恢復另一個 Task 的 Context

**4. System Tick**

- 設定 RISC-V Timer（`mtime` / `mtimecmp`）產生週期性中斷
- 在中斷處理常式中觸發 Scheduler，實現 Time Slicing

**5. Critical Section**

- 最簡單的實作：全域中斷開關（`mstatus.MIE`）
- 防止資料競爭

### 2.2 Phase 2：擴充功能（Nice-to-Have）

在第一階段可以先跳過，保持核心乾淨：

- **IPC**：Queue、Semaphore、Mutex
- **動態記憶體管理**：Heap Allocator
- **軟體計時器**：Software Timer
- **Hook Functions**：Idle Hook、Tick Hook
- **SMP 支援**：多核心排程

### 2.3 主流 RTOS 核心模組參考

| RTOS | 核心模組 | 對 danieRTOS 的啟示 |
|------|----------|---------------------|
| **FreeRTOS** | `tasks.c`、`list.c`、`port.c`、`queue.c` | **List 是靈魂**。FreeRTOS 用 `list.c` 管理 Ready/Blocked 佇列 |
| **RT-Thread** | Object Container、IPC、Scheduler | 架構較大，但「物件容器」的概念有助於管理資源 |
| **Zephyr** | Microkernel、Device Tree、Kconfig | 過度複雜，不適合「手刻」教育專案 |

**結論**：danieRTOS 參考 FreeRTOS 的設計，但簡化實作、改善可讀性。

---

## 三、設計哲學：教育優先

### 3.1 拒絕 Macro Hell

FreeRTOS 為了跨平台和效能，使用了大量的巨集：

```c
// FreeRTOS 風格（Macro Hell）
#define portENTER_CRITICAL()    vPortEnterCritical()
#define portEXIT_CRITICAL()     vPortExitCritical()
#define configASSERT( x )       if( ( x ) == 0 ) { taskDISABLE_INTERRUPTS(); for( ;; ); }
```

**danieRTOS 的做法**：儘量使用 `static inline` 函數，C 語言的可讀性遠高於巨集。

```c
// danieRTOS 風格（清晰的函數）
static inline void enter_critical(void) {
    disable_interrupts();
    critical_nesting++;
}

static inline void exit_critical(void) {
    critical_nesting--;
    if (critical_nesting == 0) {
        enable_interrupts();
    }
}
```

### 3.2 現代命名法

FreeRTOS 使用匈牙利命名法（Hungarian Notation），這在現代已被視為過時：

```c
// FreeRTOS 風格（匈牙利命名法）
TCB_t * pxCurrentTCB;
BaseType_t xTaskCreate(...);
TickType_t xTickCount;
```

**danieRTOS 的做法**：使用 **Snake Case**，保持語意清晰。

```c
// danieRTOS 風格（Snake Case）
tcb_t *current_tcb;
danie_err_t task_create(...);
uint64_t tick_count;
```

### 3.3 資料結構透明化

商業 RTOS 傾向隱藏結構體內容（Opaque Pointers），防止使用者直接存取內部資料。

**danieRTOS 的做法**：為了教學，讓使用者能直接看到 TCB 裡的 `sp`、`priority` 是如何變化的。

```c
// danieRTOS：透明的 TCB 結構
typedef struct tcb {
    uint64_t *stack_ptr;        // 當前 Stack Pointer
    uint32_t priority;          // Task 優先級
    uint32_t state;             // Task 狀態
    char name[16];              // Task 名稱（除錯用）
    struct tcb *next;           // Ready List 鏈結
} tcb_t;
```

### 3.4 架構選擇：Monolithic

在 M-mode 下，沒有 MMU 隔離，Microkernel 和 Monolithic 的界線是模糊的。

**danieRTOS 採用 Monolithic 架構**：

- 所有服務（Scheduler、IPC）在同一個位址空間
- 直接函數呼叫，簡單高效
- 本質上是一個靜態連結庫（`libdaniertos.a`）
- 應用程式與核心編譯在一起

這是 FreeRTOS 的模式，也是最適合教學的模式。

---

## 四、技術選擇

### 4.1 為什麼選擇 RISC-V

**1. 開放架構**

RISC-V 是開源的 ISA，所有規格都可以免費取得。不像 ARM 需要授權，RISC-V 對學習者完全開放。

**2. 簡潔設計**

RISC-V 的設計哲學是「簡潔」。基礎指令集只有 47 條指令，比 x86 的數千條指令簡單得多。

**3. 學習資源豐富**

我對 RISC-V 的 Privilege Modes、Exception Handling、CSR 都有深入了解（參見本書末「延伸閱讀」）。

**4. 業界趨勢**

RISC-V 正在快速成長，特別是在嵌入式和 AIoT 領域。學習 RISC-V RTOS 開發是有價值的技能。

### 4.2 為什麼選擇 QEMU

**1. 免費且開源**

不需要購買開發板，任何人都可以在自己的電腦上運行。

**2. GDB 整合**

QEMU 支援 GDB Remote Debug，可以單步執行、設定斷點、查看暫存器和記憶體。這對於除錯 Context Switch 這種低階操作至關重要。

**3. 可重現性**

QEMU 的執行是確定性的，同樣的程式會產生同樣的結果。這讓除錯變得容易，也讓讀者可以重現我的所有實驗。

**4. virt Machine**

QEMU 的 `virt` machine 是一個標準化的虛擬平台，包含：

- CLINT（Core Local Interrupter）：提供 Timer Interrupt
- PLIC（Platform-Level Interrupt Controller）：處理外部中斷
- UART：串口輸出
- RAM：可配置的記憶體大小

### 4.3 開發平台規格

| 項目 | 規格 |
|------|------|
| **架構** | RISC-V RV64GC |
| **特權模式** | M-mode（Machine Mode） |
| **模擬器** | QEMU virt machine |
| **工具鏈** | riscv64-unknown-elf-gcc（Newlib） |
| **除錯器** | riscv64-unknown-elf-gdb |
| **核心數** | 單核心（Phase 1） |

---

## 五、API 設計原則

### 5.1 前綴一致性

所有 API 使用 `danie_` 前綴，防止命名衝突：

```c
danie_err_t danie_task_create(...);
void danie_sched_start(void);
void danie_delay(uint32_t ticks);
```

### 5.2 型別安全

使用 `<stdint.h>` 的標準型別，不自己定義 `BaseType_t`：

```c
#include <stdint.h>

typedef struct tcb tcb_t;
typedef uint64_t tick_t;
typedef int32_t priority_t;
```

### 5.3 錯誤處理

定義清晰的錯誤碼：

```c
typedef enum {
    DANIE_OK = 0,           // 成功
    DANIE_ERR_NOMEM,        // 記憶體不足
    DANIE_ERR_INVALID,      // 參數錯誤
    DANIE_ERR_TIMEOUT       // 等待超時
} danie_err_t;
```

對於邏輯錯誤（例如：在中斷中呼叫會 Block 的 API），直接 Panic：

```c
void danie_panic(const char *msg) {
    uart_puts("PANIC: ");
    uart_puts(msg);
    uart_puts("\n");
    while (1);  // 停止執行
}
```

---

## 六、開發環境設定

### 6.1 工具鏈安裝

**Ubuntu/Debian**：

```bash
# 安裝 RISC-V 工具鏈
sudo apt install gcc-riscv64-unknown-elf

# 安裝 QEMU
sudo apt install qemu-system-misc

# 驗證安裝
riscv64-unknown-elf-gcc --version
qemu-system-riscv64 --version
```

**macOS（使用 Homebrew）**：

```bash
brew tap riscv-software-src/riscv
brew install riscv-gnu-toolchain
brew install qemu
```

### 6.2 專案結構

```
daniertos/
├── Makefile                # 建置腳本
├── linker.ld               # Linker Script
├── src/
│   ├── main.c              # 入口點
│   ├── start.S             # 啟動程式碼（Assembly）
│   ├── task.c              # Task 管理
│   ├── scheduler.c         # Scheduler
│   ├── port.c              # RISC-V Port（C 部分）
│   └── portasm.S           # RISC-V Port（Assembly 部分）
├── include/
│   ├── daniertos.h         # 主要標頭檔
│   ├── task.h              # Task API
│   ├── scheduler.h         # Scheduler API
│   └── port.h              # Port 定義
└── demo/
    └── demo.c              # Demo 應用程式
```

### 6.3 QEMU 運行指令

```bash
# 編譯
make

# 運行
qemu-system-riscv64 -machine virt -nographic -kernel daniertos.elf

# 除錯模式（等待 GDB 連接）
qemu-system-riscv64 -machine virt -nographic -kernel daniertos.elf -S -gdb tcp::1234

# GDB 連接
riscv64-unknown-elf-gdb daniertos.elf
(gdb) target remote :1234
(gdb) break main
(gdb) continue
```

---

## 總結

本文介紹了 danieRTOS 的專案規劃：

1. **動機**：從學習者到實作者，親手寫 RTOS 是理解 RTOS 的最佳方式
2. **定位**：教育用途優先，可讀性 > 效能
3. **核心功能**：Task 管理、Scheduler、Context Switch、System Tick、Critical Section
4. **設計哲學**：拒絕 Macro Hell、現代命名法、資料結構透明化
5. **技術選擇**：RISC-V RV64GC + QEMU virt + M-mode
6. **API 設計**：前綴一致、型別安全、清晰的錯誤處理

從下一篇開始，我們將進入實作階段，從 RISC-V HAL 開始，一步步建構 danieRTOS。

---

## 參考資料

**RTOS 參考實作**

- **FreeRTOS Kernel**
  https://github.com/FreeRTOS/FreeRTOS-Kernel
  業界廣泛使用的開源 RTOS，本專案的主要參考對象。

**RISC-V 規格**

- **RISC-V Instruction Set Manual, Volume II: Privileged Architecture**
  RISC-V International
  https://github.com/riscv/riscv-isa-manual
  M-mode、CSR、Trap 機制的官方規格。

**開發工具**

- **QEMU RISC-V Emulator**
  https://www.qemu.org/docs/master/system/target-riscv.html
  本專案使用的模擬器，支援 virt machine 和 GDB 除錯。

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)
