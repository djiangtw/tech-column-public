# Context Switch 實作

**作者**: Danny Jiang
**日期**: 2025-12-13

---

## 前言：RTOS 最核心、最容易出錯的地方

如果你問我「RTOS 中最難實作的部分是什麼？」，我會毫不猶豫地回答：**Context Switch**。

這不是因為 Context Switch 的概念難以理解——概念其實很簡單：保存當前 Task 的暫存器，恢復下一個 Task 的暫存器。但魔鬼藏在細節裡：

- 少存一個暫存器？Crash。
- 恢復的順序錯了？Crash。
- Stack 對齊錯了？Crash。
- `mret` 之前忘了設 `mepc`？跳到奇怪的地方，然後 Crash。

更糟糕的是，這些 bug 通常不會立刻表現出來。系統可能會運行幾秒、幾分鐘、甚至幾小時，然後在某個特定的 Task 切換時突然死機。這種隨機性讓除錯變成一場噩夢。

我曾經花了整整三天追蹤一個 Context Switch 的 bug。最後發現原因是：**我把 `x2`（Stack Pointer）也存到了 Stack 上，然後又從 Stack 恢復它**。聽起來很合理對吧？但問題是，恢復 `sp` 之後，其他暫存器的 offset 就全錯了。

這種經驗讓我深刻體會到：Context Switch 是那種「看起來簡單，但一個小錯誤就會讓整個系統崩潰」的程式碼。

本文將詳細說明如何在 RISC-V 上實作 Context Switch。讀完這篇文章，你將能夠：

- 理解為什麼要保存「所有」暫存器（而不只是 Callee-saved）
- 設計一個清晰的 Stack Frame Layout
- 實作 `SAVE_CONTEXT` 和 `RESTORE_CONTEXT` 巨集
- 處理「第一次 Context Switch」的特殊情況

> 💡 **相關閱讀**：如果你對 RISC-V 的暫存器和 ABI 不熟悉，建議先閱讀《See RISC-V Run》系列。

---

## 一、Context 的完整定義

### 1.1 什麼是 Context？

**Context（上下文）**是一個 Task 在某個時間點的「完整狀態」。

想像你正在玩一個 RPG 遊戲。當你存檔時，遊戲會記錄：

- 角色的位置（X, Y 座標）
- 角色的狀態（HP, MP, 等級）
- 背包裡的道具
- 目前的任務進度

讀檔時，遊戲恢復所有這些資訊，你可以繼續玩，就像從未離開過一樣。

**對 CPU 來說，Context 就是「遊戲存檔」。**

### 1.2 RISC-V 的 Context 包含什麼？

在 RISC-V RV64 上，一個 Task 的 Context 包含：

**1. 通用暫存器（x0-x31）**

| 暫存器 | ABI 名稱 | 用途 |
|--------|----------|------|
| x0 | zero | 硬體接線為 0，不需要保存 |
| x1 | ra | Return Address |
| x2 | sp | Stack Pointer |
| x3 | gp | Global Pointer |
| x4 | tp | Thread Pointer |
| x5-x7 | t0-t2 | Temporaries（Caller-saved） |
| x8 | s0/fp | Saved register / Frame pointer |
| x9 | s1 | Saved register |
| x10-x17 | a0-a7 | Arguments / Return values |
| x18-x27 | s2-s11 | Saved registers（Callee-saved） |
| x28-x31 | t3-t6 | Temporaries（Caller-saved） |

**2. 關鍵 CSR**

| CSR | 用途 | 為什麼要保存？ |
|-----|------|---------------|
| mepc | 中斷發生時的 PC | 沒有它，Task 不知道回到哪裡繼續執行 |
| mstatus | 中斷狀態、特權模式 | 沒有它，`mret` 後中斷可能是關閉的 |

**3. 浮點暫存器（Optional）**

如果 Task 使用浮點運算，還需要保存 `f0-f31`。但這會讓 Context 大小翻倍，所以 Phase 1 我們**不支援浮點**。

### 1.3 為什麼要保存「所有」暫存器？

你可能會問：「RISC-V ABI 不是規定 Callee-saved 暫存器由被呼叫的函數負責保存嗎？那我們只需要保存 Caller-saved 不就好了？」

這個問題問得很好。讓我解釋為什麼不行。

**一般函數呼叫**：

```
Task A 呼叫 foo()
  → foo() 保存 s0-s11（如果它會用到）
  → foo() 執行
  → foo() 恢復 s0-s11
  → 返回 Task A
```

在這種情況下，Task A 知道自己正在呼叫 foo()，所以它會假設 `t0-t6`（Caller-saved）可能會被改變，但 `s0-s11`（Callee-saved）會被保留。

**Preemptive Context Switch**：

```
Task A 正在執行
  → Timer Interrupt 發生
  → Task A 被強制中斷（它完全不知道！）
  → 切換到 Task B
  → ... 一段時間後 ...
  → 切換回 Task A
  → Task A 繼續執行
```

**關鍵差異**：Task A 不知道自己被中斷了。它可能正在使用 `t0` 存放一個重要的中間結果，如果我們沒有保存 `t0`，當 Task A 恢復執行時，`t0` 的值已經是 Task B 留下的垃圾。

**結論：Preemptive Context Switch 必須保存所有暫存器，因為 Task 可能在任何時刻被中斷。**

---

## 二、Stack Frame Layout

### 2.1 設計原則

一個好的 Stack Frame Layout 應該：

1. **對齊**：RISC-V 要求 Stack 16-byte 對齊
2. **一致性**：每次保存和恢復的順序必須完全相同
3. **可除錯**：Layout 要夠清晰，方便用 GDB 檢查

### 2.2 danieRTOS 的 Stack Frame

我們需要保存 31 個 GPR（x1-x31）和 2 個 CSR（mepc、mstatus），總共 33 個 64-bit 值。

為了 16-byte 對齊，我們分配 34 個 slots（272 bytes）：

```
┌────────────────────────────────────────┐ ← 原本的 SP (High Address)
│ mstatus                                │ offset: 33 × 8 = 264
│ mepc                                   │ offset: 32 × 8 = 256
├────────────────────────────────────────┤
│ x31 (t6)                               │ offset: 31 × 8 = 248
│ x30 (t5)                               │ offset: 30 × 8 = 240
│ ...                                    │
│ x3  (gp)                               │ offset: 3 × 8 = 24
│ x2  (sp) [not actually used]           │ offset: 2 × 8 = 16
│ x1  (ra)                               │ offset: 1 × 8 = 8
│ [reserved / padding]                   │ offset: 0 × 8 = 0
└────────────────────────────────────────┘ ← 新的 SP (Low Address)
                                           tcb->sp 指向這裡
```

**為什麼 x2 (sp) 的 slot 標記為 "not actually used"？**

因為我們用 `sp` 本身來定位整個 Stack Frame。當我們執行 `addi sp, sp, -272` 時，新的 `sp` 就是 Stack Frame 的起點。恢復時，我們先恢復其他暫存器，最後才 `addi sp, sp, 272`。

所以 x2 的 slot 只是一個佔位符，保持 Layout 的一致性。

### 2.3 Offset 常數定義

為了讓 Assembly 程式碼更清晰，我們定義常數：

```asm
# context.h (可以用 C 預處理器產生)
.equ CTX_RA,      8       # x1
.equ CTX_SP,      16      # x2 (placeholder)
.equ CTX_GP,      24      # x3
.equ CTX_TP,      32      # x4
.equ CTX_T0,      40      # x5
.equ CTX_T1,      48      # x6
.equ CTX_T2,      56      # x7
.equ CTX_S0,      64      # x8
.equ CTX_S1,      72      # x9
.equ CTX_A0,      80      # x10
.equ CTX_A1,      88      # x11
# ... 以此類推 ...
.equ CTX_T6,      248     # x31
.equ CTX_MEPC,    256
.equ CTX_MSTATUS, 264
.equ CTX_SIZE,    272     # 總大小
```

---

## 三、Assembly 實作

這是 danieRTOS 中唯一必須用 Assembly 寫的部分。我們使用 `.S` 檔案（大寫 S 表示會經過 C 預處理器）。

### 3.1 SAVE_CONTEXT 巨集

```asm
# portasm.S

.macro SAVE_CONTEXT
    # 1. 在 Stack 上分配空間
    addi sp, sp, -272

    # 2. 保存通用暫存器 (x1-x31)
    sd x1,   8(sp)      # ra
    # 注意：x2 (sp) 我們不在這裡存，因為它的值就是 sp+272
    sd x3,  24(sp)      # gp
    sd x4,  32(sp)      # tp
    sd x5,  40(sp)      # t0
    sd x6,  48(sp)      # t1
    sd x7,  56(sp)      # t2
    sd x8,  64(sp)      # s0
    sd x9,  72(sp)      # s1
    sd x10, 80(sp)      # a0
    sd x11, 88(sp)      # a1
    sd x12, 96(sp)      # a2
    sd x13, 104(sp)     # a3
    sd x14, 112(sp)     # a4
    sd x15, 120(sp)     # a5
    sd x16, 128(sp)     # a6
    sd x17, 136(sp)     # a7
    sd x18, 144(sp)     # s2
    sd x19, 152(sp)     # s3
    sd x20, 160(sp)     # s4
    sd x21, 168(sp)     # s5
    sd x22, 176(sp)     # s6
    sd x23, 184(sp)     # s7
    sd x24, 192(sp)     # s8
    sd x25, 200(sp)     # s9
    sd x26, 208(sp)     # s10
    sd x27, 216(sp)     # s11
    sd x28, 224(sp)     # t3
    sd x29, 232(sp)     # t4
    sd x30, 240(sp)     # t5
    sd x31, 248(sp)     # t6

    # 3. 保存 CSR
    csrr t0, mepc
    csrr t1, mstatus
    sd t0, 256(sp)      # mepc
    sd t1, 264(sp)      # mstatus

    # 4. 把新的 SP 存到 current_tcb->sp
    la t0, current_tcb  # t0 = &current_tcb
    ld t0, 0(t0)        # t0 = current_tcb (指標的值)
    sd sp, 0(t0)        # current_tcb->sp = sp
.endm
```

**重點解釋**：

- 第 4 步很關鍵：我們把 Stack Pointer 存到 TCB 的第一個欄位（`sp`）。這樣 Scheduler 可以透過 `current_tcb` 找到這個 Task 的完整 Context。

### 3.2 RESTORE_CONTEXT 巨集

```asm
.macro RESTORE_CONTEXT
    # 1. 從 current_tcb 讀取 SP
    la t0, current_tcb
    ld t0, 0(t0)        # t0 = current_tcb
    ld sp, 0(t0)        # sp = current_tcb->sp

    # 2. 恢復 CSR
    ld t0, 256(sp)      # t0 = saved mepc
    ld t1, 264(sp)      # t1 = saved mstatus
    csrw mepc, t0
    csrw mstatus, t1

    # 3. 恢復通用暫存器 (x1-x31)
    ld x1,   8(sp)      # ra
    # x2 (sp) 最後才處理
    ld x3,  24(sp)      # gp
    ld x4,  32(sp)      # tp
    ld x5,  40(sp)      # t0
    ld x6,  48(sp)      # t1
    ld x7,  56(sp)      # t2
    ld x8,  64(sp)      # s0
    ld x9,  72(sp)      # s1
    ld x10, 80(sp)      # a0
    ld x11, 88(sp)      # a1
    ld x12, 96(sp)      # a2
    ld x13, 104(sp)     # a3
    ld x14, 112(sp)     # a4
    ld x15, 120(sp)     # a5
    ld x16, 128(sp)     # a6
    ld x17, 136(sp)     # a7
    ld x18, 144(sp)     # s2
    ld x19, 152(sp)     # s3
    ld x20, 160(sp)     # s4
    ld x21, 168(sp)     # s5
    ld x22, 176(sp)     # s6
    ld x23, 184(sp)     # s7
    ld x24, 192(sp)     # s8
    ld x25, 200(sp)     # s9
    ld x26, 208(sp)     # s10
    ld x27, 216(sp)     # s11
    ld x28, 224(sp)     # t3
    ld x29, 232(sp)     # t4
    ld x30, 240(sp)     # t5
    ld x31, 248(sp)     # t6

    # 4. 釋放 Stack 空間
    addi sp, sp, 272

    # 5. 返回 Task (Magic Jump!)
    mret
.endm
```

**`mret` 的魔法**：

當 CPU 執行 `mret` 時，它會：

1. 跳轉到 `mepc` 指向的地址
2. 恢復 `mstatus.MPIE` 到 `mstatus.MIE`（開啟中斷）
3. 恢復 `mstatus.MPP` 到當前特權模式

這就是為什麼我們在偽造 Stack 時，要把 `mstatus` 設為 `0x1880`（MPIE=1, MPP=11）。

### 3.3 Trap Handler 整合

把 `SAVE_CONTEXT` 和 `RESTORE_CONTEXT` 整合到 Trap Handler：

```asm
.section .text
.global trap_handler
.align 4

trap_handler:
    # 保存 Context
    SAVE_CONTEXT

    # 呼叫 C 語言的 Trap 處理函數
    # 參數：a0 = mcause, a1 = mepc
    csrr a0, mcause
    csrr a1, mepc
    call handle_trap    # C function

    # C 函數可能呼叫了 schedule() 並改變了 current_tcb

    # 恢復 Context（可能是不同的 Task！）
    RESTORE_CONTEXT
```

**關鍵觀察**：`RESTORE_CONTEXT` 讀取的是 `current_tcb->sp`。如果 `handle_trap()` 裡面的 Scheduler 改變了 `current_tcb`，那麼 `RESTORE_CONTEXT` 就會恢復到新的 Task。

**這就是 Context Switch 發生的地方。**

---

## 四、第一次 Context Switch

### 4.1 問題：系統啟動時沒有 current_tcb

當 `main()` 呼叫 `scheduler_start()` 時，系統正在使用啟動時的 Stack（Boot Stack）。此時 `current_tcb` 是 NULL。

我們不能用正常的 `SAVE_CONTEXT` → `RESTORE_CONTEXT` 流程，因為沒有「舊的 Task」需要保存。

### 4.2 解法：直接 RESTORE 第一個 Task

`start_first_task()` 的邏輯：

1. 從 Ready List 選出第一個 Task
2. 設定 `current_tcb` 指向它
3. 直接執行 `RESTORE_CONTEXT`（跳過 `SAVE_CONTEXT`）

```c
// scheduler.c
void scheduler_start(void) {
    // 1. 選擇第一個 Task
    current_tcb = find_highest_priority_ready_task();

    if (current_tcb == NULL) {
        danie_panic("No task to run!");
    }

    current_tcb->state = TASK_RUNNING;

    // 2. 開啟 Timer
    timer_init();

    // 3. 跳轉到第一個 Task（永不返回）
    start_first_task();  // Assembly function
}
```

```asm
# portasm.S
.global start_first_task
.align 4

start_first_task:
    # 設定 Trap Handler
    la t0, trap_handler
    csrw mtvec, t0

    # 直接恢復第一個 Task 的 Context
    RESTORE_CONTEXT

    # 永遠不會到這裡
```

**注意**：`start_first_task()` 永遠不會返回。它執行 `mret` 後，CPU 就開始執行第一個 Task 了。

### 4.3 為什麼這樣可以運作？

回想一下 Ch3 的 Stack 初始化：我們「偽造」了一個 Stack Frame，讓新 Task 看起來像被中斷過。

當 `RESTORE_CONTEXT` 執行時：

1. 從 `current_tcb->sp` 讀取 Stack Pointer
2. 從 Stack 恢復所有暫存器
3. 從 Stack 恢復 `mepc`（指向 Task 入口函數）
4. 從 Stack 恢復 `mstatus`（MPIE=1, MPP=11）
5. 執行 `mret` → 跳到 Task 入口函數，開始執行

**CPU 完全不知道這是一個「新」Task。它以為自己只是從一個普通的中斷返回。**

---

## 五、除錯技巧

Context Switch 的 bug 很難追蹤。以下是一些實用的除錯技巧：

### 5.1 在 Stack Frame 加入 Magic Number

在 Stack 的特定位置放一個辨識碼：

```c
#define CONTEXT_MAGIC 0xDEADBEEFCAFEBABE

void task_stack_init(tcb_t *tcb, ...) {
    // ...
    sp[0] = CONTEXT_MAGIC;  // 放在 offset 0
    // ...
}
```

在 `RESTORE_CONTEXT` 之前檢查：

```asm
restore_context:
    la t0, current_tcb
    ld t0, 0(t0)
    ld sp, 0(t0)

    # 檢查 Magic Number
    ld t1, 0(sp)
    li t2, 0xDEADBEEFCAFEBABE
    bne t1, t2, context_corrupted

    # 繼續正常流程...
```

### 5.2 用 GDB 檢查 Stack

```gdb
# 查看當前 Task 的 TCB
(gdb) p *current_tcb

# 查看 Stack 內容
(gdb) x/34gx current_tcb->sp

# 查看特定暫存器
(gdb) p/x *(uint64_t*)(current_tcb->sp + 256)  # mepc
```

### 5.3 在 UART 輸出 Context Switch 資訊

```c
void handle_trap(uint64_t mcause, uint64_t mepc) {
    if (is_timer_interrupt(mcause)) {
        tcb_t *old = current_tcb;
        schedule();
        tcb_t *new = current_tcb;

        if (old != new) {
            uart_puts("Switch: ");
            uart_puts(old->name);
            uart_puts(" -> ");
            uart_puts(new->name);
            uart_puts("\n");
        }
    }
}
```

---

## 總結

本文實作了 danieRTOS 的 Context Switch：

1. **Context 定義**：31 個 GPR + 2 個 CSR（mepc、mstatus）
2. **為什麼保存所有暫存器**：Preemptive Context Switch 中，Task 可能在任何時刻被中斷
3. **Stack Frame Layout**：272 bytes，16-byte 對齊
4. **Assembly 實作**：`SAVE_CONTEXT` 和 `RESTORE_CONTEXT` 巨集
5. **第一次 Context Switch**：跳過 `SAVE_CONTEXT`，直接 `RESTORE_CONTEXT`

Context Switch 是 RTOS 的心臟。現在我們有了「保存」和「恢復」的能力，下一步就是決定「什麼時候切換」以及「切換給誰」。

---

## 參考資料

**RISC-V 規格**

- **RISC-V ELF psABI Specification**
  RISC-V International
  https://github.com/riscv-non-isa/riscv-elf-psabi-doc
  Calling Convention、Caller/Callee-saved 暫存器的官方定義。

- **RISC-V Instruction Set Manual, Volume II: Privileged Architecture**
  RISC-V International
  https://github.com/riscv/riscv-isa-manual
  mepc、mstatus 等 CSR 的說明。

**延伸閱讀**

- **See RISC-V Run**
  Danny Jiang
  RISC-V 架構深入解析，涵蓋 Trap 機制、CSR、Privilege Modes。

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)
