# 16. 隱形的力場：PMP 記憶體保護

> 「好籬笆造就好鄰居。」—— Robert Frost

1988 年 11 月 2 日，一個程式開始在網路上蔓延。

這個程式叫做 **Morris Worm**，由康乃爾大學研究生 Robert Tappan Morris 編寫。他本意是測量網路的規模，但程式中的一個 Bug 讓它失控了——它會重複感染同一台機器，最終把系統資源耗盡。

24 小時內，約 6,000 台電腦被感染——當時網路上大約 10% 的機器。這是史上第一次大規模網路攻擊。

Morris Worm 利用了幾個漏洞，其中最關鍵的是 **Buffer Overflow**。當程式寫入超出邊界的資料時，可以覆蓋堆疊上的返回地址，讓程式跳到攻擊者指定的位置執行惡意程式碼。

當時的 Unix 系統雖然有 Privilege Mode 的概念，但記憶體保護不夠完善。一個漏洞就能讓攻擊者從 User 程式跳進 Kernel，進而控制整個系統。

今天，我們要在 danieRTOS 中建立一道「隱形的力場」——讓 User Task 無論怎麼嘗試，都無法碰觸 Kernel 的記憶體。即使有 Buffer Overflow，攻擊者也無法突破這道防線。

---

## 飯店房卡模型

想像你住進一家飯店：

- 整棟大樓有 100 間房間
- 每個房客都有一張房卡
- 你的房卡只能開你自己的房間
- 就算你知道隔壁房間的門在哪，刷卡也沒用

這就是記憶體保護的概念：

| 飯店 | 記憶體保護 |
|------|------------|
| 大樓 | 整個記憶體空間 |
| 房間 | PMP 區域 |
| 房卡 | 權限設定 |
| 刷卡失敗 | Access Fault |

---

## PMP：Physical Memory Protection

RISC-V 提供兩種記憶體保護機制：

1. **MMU (Memory Management Unit)**：虛擬記憶體，需要 S-mode
2. **PMP (Physical Memory Protection)**：實體記憶體，只需要 M-mode

由於 danieRTOS 使用 M + U mode（沒有 S-mode），我們使用 PMP。

### PMP 的組成

PMP 由一組 CSR 控制：

| CSR | 說明 |
|-----|------|
| pmpaddr0-15 | 區域邊界地址 |
| pmpcfg0-3 | 區域配置（每個 8 bits） |

每個 PMP entry 控制一個記憶體區域的權限：

```
pmpcfg entry (8 bits):
┌───┬───┬───┬───┬───────┐
│ L │ 0 │ 0 │ A │ X W R │
└───┴───┴───┴───┴───────┘
  │           │   │ │ │
  │           │   │ │ └── Read
  │           │   │ └──── Write  
  │           │   └────── Execute
  │           └────────── Address Matching Mode
  └────────────────────── Lock（鎖定後無法修改）
```

### Address Matching Modes

| A | Mode | 說明 |
|---|------|------|
| 00 | OFF | 區域關閉 |
| 01 | TOR | Top of Range（使用前一個 pmpaddr 為起點） |
| 10 | NA4 | 自然對齊 4 bytes |
| 11 | NAPOT | 自然對齊 2^n bytes |

---

## 簡單二分策略

對於教育用途，我們採用最簡單的策略：把記憶體分成兩塊。

```
0x80000000 ┌─────────────────────────┐
           │                         │
           │      Kernel Region      │  ← M-mode 專屬
           │      (1 MB)             │
           │                         │
0x80100000 ├─────────────────────────┤
           │                         │
           │      User Region        │  ← U-mode 可存取
           │      (1 MB)             │
           │                         │
0x80200000 └─────────────────────────┘
```

### PMP 設定

我們需要兩個 PMP entries：

```c
void pmp_init(void) {
    // Entry 0: Kernel region (0x80000000 - 0x80100000)
    // 使用 TOR mode，M-mode 可存取，U-mode 不可
    
    // Entry 1: User region (0x80100000 - 0x80200000)
    // 使用 TOR mode，M-mode 和 U-mode 都可存取
    
    // pmpaddr 存的是 address >> 2
    uint64_t kernel_end = 0x80100000 >> 2;
    uint64_t user_end = 0x80200000 >> 2;
    
    // 設定地址
    csr_write(pmpaddr0, kernel_end);
    csr_write(pmpaddr1, user_end);
    
    // 設定權限
    // Entry 0: TOR, no R/W/X for U-mode (由 M-mode 存取)
    // Entry 1: TOR, R/W/X for all
    uint64_t cfg = 0;
    cfg |= (PMP_TOR) << 0;                      // Entry 0: TOR, no permissions
    cfg |= (PMP_TOR | PMP_R | PMP_W | PMP_X) << 8;  // Entry 1: TOR, RWX
    
    csr_write(pmpcfg0, cfg);
}
```

### TOR Mode 解釋

TOR (Top of Range) 使用 **前一個 pmpaddr** 作為起點：

```
pmpaddr[-1] = 0（隱含的起點）
pmpaddr[0] = 0x80100000 >> 2

Entry 0 範圍：0x00000000 ~ 0x80100000

pmpaddr[0] = 0x80100000 >> 2
pmpaddr[1] = 0x80200000 >> 2

Entry 1 範圍：0x80100000 ~ 0x80200000
```

---

## PMP 的權限檢查

當 CPU 存取記憶體時，PMP 會進行檢查：

```
CPU 發出記憶體存取請求
        │
        ▼
┌───────────────────────────────┐
│  掃描 PMP entries (0 → 15)   │
│  找到第一個匹配的區域        │
└───────────────────────────────┘
        │
        ▼
   找到匹配？
        │
   ┌────┴────┐
   No        Yes
   │          │
   ▼          ▼
[預設規則]  [檢查權限]
   │          │
   ▼          ▼
M-mode 允許  權限符合？
U-mode 拒絕  ├── Yes → 允許存取
             └── No → Access Fault
```

### 預設規則

如果沒有任何 PMP entry 匹配：
- **M-mode**：允許存取（向後相容）
- **U-mode**：拒絕存取

這意味著只要設定 PMP，U-mode 就只能存取明確允許的區域。

---

## 存取違規處理

當 U-mode 嘗試存取 Kernel 區域，會觸發異常：

| 違規類型 | mcause |
|----------|--------|
| Load access fault | 5 |
| Store access fault | 7 |
| Instruction access fault | 1 |

`mtval` 會儲存觸發違規的地址。

### Fault Handler

```c
void handle_access_fault(trap_frame_t *tf) {
    uint64_t cause = csr_read(mcause);
    uint64_t addr = csr_read(mtval);
    uint64_t pc = tf->mepc;

    kprintf("Access Fault!\n");
    kprintf("  Cause: %s\n", cause == 5 ? "Load" : "Store");
    kprintf("  Address: 0x%lx\n", addr);
    kprintf("  PC: 0x%lx\n", pc);
    kprintf("  Task: %s\n", current_task->name);

    // 終止違規的 Task
    task_kill(current_task);

    // 切換到其他 Task
    scheduler_reschedule();
}
```

---

## 測試 PMP

讓我們寫一個「惡意」Task 來測試 PMP：

```c
// 這個 Task 會嘗試讀取 Kernel 記憶體
void evil_task(void *arg) {
    kprintf("Evil Task: I'm going to read kernel memory!\n");

    // 嘗試讀取 Kernel 區域
    volatile uint64_t *kernel_ptr = (uint64_t *)0x80000000;
    uint64_t secret = *kernel_ptr;  // 這會觸發 Access Fault

    // 這行永遠不會執行
    kprintf("Secret: 0x%lx\n", secret);
}
```

預期輸出：

```
Evil Task: I'm going to read kernel memory!
Access Fault!
  Cause: Load
  Address: 0x80000000
  PC: 0x80100xxx
  Task: evil_task
[evil_task terminated]
[Other tasks continue running...]
```

---

## 進階：Per-Task PMP

簡單二分策略讓所有 User Task 共享同一塊記憶體。如果我們想讓每個 Task 有自己的獨立空間呢？

### 動態 PMP 更新

每次 Context Switch 時，更新 PMP 設定：

```c
void switch_to(tcb_t *next) {
    // ... 原有的 context switch 邏輯 ...

    // 更新 PMP 為新 Task 的區域
    uint64_t task_start = (uint64_t)next->user_stack_base >> 2;
    uint64_t task_end = (uint64_t)(next->user_stack_base + next->user_stack_size) >> 2;

    csr_write(pmpaddr2, task_start);
    csr_write(pmpaddr3, task_end);

    // 設定 Entry 2-3：只有這個 Task 的區域可存取
    // ...
}
```

### 限制

RISC-V PMP 通常只有 **16 個 entries**。如果每個 Task 需要多個 entries（Code、Stack、Heap），能支援的 Task 數量會受限。

這也是為什麼複雜系統使用 **MMU**（虛擬記憶體）——每個 Task 有自己完整的地址空間。

---

## PMP 設計決策

對於 danieRTOS v1.x，我們選擇：

| 決策 | 選擇 | 理由 |
|------|------|------|
| 策略 | 全域二分 | 簡單，足夠示範保護概念 |
| User 區域權限 | RWX | 簡化，允許 User Code 在 User 區域 |
| Kernel 區域權限 | 無（對 U-mode）| 嚴格隔離 |
| Per-Task 隔離 | 不實作 | 避免複雜性，留給讀者作練習 |

---

## 記憶體佈局總結

```
0x80000000  ┌─────────────────────────┐
            │  Kernel Code (.text)    │  PMP Entry 0
            │  Kernel Data (.data)    │  M-mode only
            │  Kernel BSS (.bss)      │
            │  Kernel Stacks          │
0x80100000  ├═════════════════════════┤  ← 邊界
            │  User Code (.text)      │  PMP Entry 1
            │  User Data (.data)      │  M+U mode
            │  User BSS (.bss)        │  RWX
            │  User Stacks            │
0x80200000  └─────────────────────────┘
```

### Linker Script 修改

```ld
MEMORY
{
    KERNEL (rwx) : ORIGIN = 0x80000000, LENGTH = 1M
    USER   (rwx) : ORIGIN = 0x80100000, LENGTH = 1M
}

SECTIONS
{
    /* Kernel sections */
    .text : {
        *(.text.entry)
        *(.text.kernel*)
    } > KERNEL

    .data : {
        *(.data.kernel*)
    } > KERNEL

    /* User sections */
    .user.text : {
        *(.text.user*)
    } > USER

    .user.data : {
        *(.data.user*)
    } > USER
}
```

---

## 小結

這一章我們建立了 **PMP 記憶體保護**：

1. **PMP**：RISC-V 的實體記憶體保護機制
2. **CSRs**：pmpaddr0-15, pmpcfg0-3
3. **TOR Mode**：使用前一個地址作為起點
4. **簡單二分**：Kernel (M-only) + User (M+U)
5. **違規處理**：Access Fault → 終止 Task

現在我們的 User Task 被困在自己的區域裡。無論他們多麼惡意，都無法碰觸 Kernel 的記憶體。就像飯店房卡一樣——你可以知道別的房間在哪，但就是進不去。

下一章，我們要處理「例外情況」——當 Task 犯規時，如何像裁判一樣做出公正的判決。

---

## 本章重點

| 概念 | 說明 |
|------|------|
| PMP | Physical Memory Protection，RISC-V 記憶體保護 |
| pmpaddr | 區域邊界地址（實際地址 >> 2） |
| pmpcfg | 權限設定（L, A, X, W, R） |
| TOR Mode | 使用前一個 pmpaddr 作為起點 |
| Access Fault | 違規存取觸發的異常（mcause 1, 5, 7） |
| mtval | 儲存違規的地址 |

---

## 常見錯誤與陷阱

### 錯誤 1：pmpaddr 沒有右移 2 位

```c
// 錯誤！pmpaddr 存的是 address >> 2
csr_write(pmpaddr0, 0x80100000);  // 錯誤

// 正確
csr_write(pmpaddr0, 0x80100000 >> 2);
```

PMP 地址暫存器的最低 2 位被「借用」來表示更多資訊，所以實際地址需要右移 2 位。

### 錯誤 2：忘記 TOR 的隱含起點

```c
// TOR mode 中，Entry 0 的起點是 pmpaddr[-1]，也就是 0x00000000
// 如果你只設定 pmpaddr0 = 0x80100000 >> 2
// 那麼 Entry 0 的範圍是 0x00000000 ~ 0x80100000
// 這可能不是你想要的！
```

### 錯誤 3：權限設定順序

PMP 檢查是從 Entry 0 開始，找到第一個匹配就停止。順序很重要！

```c
// 錯誤：Entry 0 給了全部權限，Entry 1 永遠不會被檢查
cfg |= (PMP_TOR | PMP_R | PMP_W | PMP_X) << 0;   // 0 ~ 0x80100000 全開
cfg |= (0) << 8;                                  // 這永遠不會執行
```

### 錯誤 4：Lock 位一旦設定就無法清除

```c
// PMP 的 L 位是「熔斷」的——設定後只有 Reset 才能清除
cfg |= PMP_L;  // 設定 Lock
csr_write(pmpcfg0, cfg);

// 之後這行會失敗（靜默忽略）
cfg &= ~PMP_L;
csr_write(pmpcfg0, cfg);  // Lock 仍然設定！
```

---

## PMP Debug 技巧

### 用 GDB 查看 PMP 設定

```gdb
(gdb) p /x $pmpaddr0
$1 = 0x20040000

(gdb) p /x $pmpaddr0 << 2
$2 = 0x80100000

(gdb) p /x $pmpcfg0
$3 = 0x1f08

# 解析 pmpcfg0：
# Entry 0: 0x08 = 0b00001000 = TOR, no RWX
# Entry 1: 0x1f = 0b00011111 = TOR, RWX
```

### 追蹤 Access Fault

當你看到 Access Fault 時：

```gdb
(gdb) p /x $mcause
$1 = 5  # Load access fault

(gdb) p /x $mtval
$2 = 0x80000000  # 嘗試存取的地址

(gdb) p /x $mepc
$3 = 0x80100abc  # 觸發 Fault 的指令位置
```

---

## 參考資料

**RISC-V 規格**

- **RISC-V Privileged Specification - Chapter 3.7 PMP**
  <https://github.com/riscv/riscv-isa-manual>
  PMP 的官方規格。

**歷史事件**

- **Morris Worm** - Wikipedia
  <https://en.wikipedia.org/wiki/Morris_worm>
  1988 年第一個大規模網路蠕蟲。

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)
