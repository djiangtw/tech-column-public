# 14. 穿越邊境的密道：Dual Stack 與 Trap 重構

> 「你永遠不應該相信街上撿來的東西。」

1988 年 11 月 2 日，一個叫 Robert Morris 的康奈爾大學研究生釋放了一隻蠕蟲。

這不是生物學的蠕蟲，而是一段能夠自我複製的程式碼。它利用了 Unix `fingerd` 服務的一個弱點：Stack Buffer Overflow。攻擊者能夠覆蓋函式的 Return Address，讓程式跳轉到注入的惡意程式碼。

在短短幾小時內，這隻蠕蟲感染了約 6,000 台電腦——當時網際網路約有 60,000 台電腦連線。這是人類歷史上第一次大規模的網路攻擊。

Morris Worm 給世人上了一課：**如果攻擊者能夠控制程式的 Stack，他就能控制整個程式。**

而如果那個程式是 Kernel？

---

## 問題：為什麼 Kernel 不能用 User Stack？

上一章我們成功地將 Task 關進了 U-mode 的籠子。但有個細節被我們跳過了：當 Trap 發生時，CPU 切換到 M-mode，但 **Stack Pointer 沒有改變**。

這意味著什麼？

```
User Task 執行中
     │
     ▼ (ecall 或中斷)
[進入 M-mode]
     │
SP 仍指向 User Stack！
     │
Kernel 開始使用 User Stack...
```

這是災難性的。

### 問題 1：安全問題

User 可以事先在 Stack 上佈置惡意資料。當 Kernel 使用這個 Stack 時，可能會讀取到偽造的 Return Address，跳轉到 User 控制的程式碼——即使已經在 M-mode。

這就是 Morris Worm 的攻擊模式，只是規模更大：不只是控制一個程式，而是控制整個 Kernel。

### 問題 2：可靠性問題

User 的 SP 可能指向無效記憶體。如果 User Task 有 Bug 導致 SP 亂掉，Kernel 一使用這個 Stack 就會觸發 Page Fault 或 Access Fault——在 Trap Handler 裡面觸發 Trap，形成無限迴圈。

### 問題 3：隔離問題

即使 User Stack 是有效的，Kernel 在上面留下的資料也可能洩漏給 User。Kernel 的內部狀態、指標、甚至密碼，都可能被 User 從 Stack 上讀到。

結論很明確：**Kernel 需要自己的 Stack。**

---

## 更衣室理論

想像你要進入一間無塵室——那種製造晶片的工廠。

你不能穿著街上的衣服直接走進去。灰塵、細菌、各種污染物會毀掉精密的製程。所以工廠有個規定：

1. 進門前，先把街服脫掉，放進置物櫃
2. 換上工廠提供的無塵服
3. 進入無塵室工作
4. 離開時，脫掉無塵服
5. 從置物櫃取回街服

**街服就是 User Stack，無塵服就是 Kernel Stack，置物櫃就是 `mscratch`。**

---

## ARM 的解法：雙 Stack Pointer

ARM Cortex-M 系列處理器直接在硬體層面解決了這個問題。它有兩個 Stack Pointer：

- **MSP (Main Stack Pointer)**：用於 Handler Mode（類似 M-mode）
- **PSP (Process Stack Pointer)**：用於 Thread Mode（類似 U-mode）

當中斷發生時，硬體自動切換到 MSP。Handler 永遠使用 MSP，不會被 User 影響。

這是個優雅的設計——硬體自動處理一切。

---

## RISC-V 的解法：mscratch

RISC-V 選擇了不同的路線。它沒有雙 SP，但提供了一個特殊的 CSR：`mscratch`。

`mscratch` 是個通用暫存器，專門給 M-mode Trap Handler 使用。它的用途完全由軟體定義，但最常見的用法是：

- **在 U-mode 時**：存放 Kernel Stack 的位址
- **在 M-mode 時**：存放 User Stack 的位址（暫存）

關鍵在於 `csrrw` 指令——**原子交換**：

```asm
csrrw sp, mscratch, sp
```

這條指令做的事情是：
1. 讀取 `mscratch` 的值
2. 把 `sp` 的值寫入 `mscratch`
3. 把讀取的值寫入 `sp`

一條指令，完成交換。不需要額外的暫存器，不會有中間狀態。

---

## Dual Stack 架構

現在我們可以設計完整的 Dual Stack 架構了。

### TCB 結構變化

```c
typedef struct {
    // 原有欄位
    void *sp;              // 現在專指 Kernel SP
    uint32_t state;
    uint32_t priority;
    
    // v1.x 新增
    void *user_sp;         // User Stack Pointer
    void *user_stack_base; // User Stack 底部（用於 PMP）
    size_t user_stack_size;
    
    trap_frame_t *trap_frame; // 指向 Kernel Stack 上的 Trap Frame
} tcb_t;
```

### 記憶體佈局

```
0x80000000  ┌─────────────────────────┐
            │      Kernel Code        │
            ├─────────────────────────┤
            │      Kernel Data        │
            ├─────────────────────────┤
            │   Kernel Stack (Task 0) │
            │   Kernel Stack (Task 1) │
            │   ...                   │
0x80100000  ├═════════════════════════┤  ← Kernel/User 邊界
            │    User Stack (Task 0)  │
            │    User Stack (Task 1)  │
            │    ...                  │
            │      User Code          │
0x80200000  └─────────────────────────┘
```

---

## Trap Handler 重構

### 進入 M-mode（U → M）

```asm
.align 4
.global trap_vector
trap_vector:
    # 第一件事：切換到 Kernel Stack
    csrrw sp, mscratch, sp
    # 現在：sp = Kernel Stack, mscratch = User SP
    
    # 分配 Trap Frame 空間
    addi sp, sp, -TRAP_FRAME_SIZE
    
    # 儲存所有暫存器
    sd ra,  0(sp)
    sd t0,  8(sp)
    sd t1, 16(sp)
    # ... 儲存 x1-x31（除了 sp）
    
    # 儲存 User SP（從 mscratch 讀取）
    csrr t0, mscratch
    sd t0, SP_OFFSET(sp)
    
    # 儲存 CSRs
    csrr t0, mepc
    sd t0, MEPC_OFFSET(sp)
    csrr t0, mstatus
    sd t0, MSTATUS_OFFSET(sp)
    
    # 呼叫 C handler
    mv a0, sp           # 傳遞 trap_frame 指標
    call trap_handler

    # 從 C handler 返回，準備返回 User
    j trap_return

trap_return:
    # 恢復 CSRs
    ld t0, MEPC_OFFSET(sp)
    csrw mepc, t0
    ld t0, MSTATUS_OFFSET(sp)
    csrw mstatus, t0

    # 恢復 User SP 到 mscratch
    ld t0, SP_OFFSET(sp)
    csrw mscratch, t0

    # 恢復所有暫存器
    ld ra,  0(sp)
    ld t0,  8(sp)
    ld t1, 16(sp)
    # ... 恢復 x1-x31（除了 sp）

    # 釋放 Trap Frame 空間
    addi sp, sp, TRAP_FRAME_SIZE

    # 切換回 User Stack
    csrrw sp, mscratch, sp
    # 現在：sp = User SP, mscratch = Kernel SP

    mret
```

### Trap Frame 結構

```c
typedef struct {
    // 通用暫存器 x0-x31（x0 不存，x2=sp 特殊處理）
    uint64_t ra;    // x1
    uint64_t sp;    // x2 (User SP)
    uint64_t gp;    // x3
    uint64_t tp;    // x4
    uint64_t t0;    // x5
    // ... 其他暫存器
    uint64_t t6;    // x31

    // CSRs
    uint64_t mepc;
    uint64_t mstatus;
} trap_frame_t;

#define TRAP_FRAME_SIZE sizeof(trap_frame_t)
```

---

## Context Switch 變化

v0.x 的 Context Switch 假設所有 Task 都在 M-mode，只需要儲存 callee-saved 暫存器。

v1.x 需要考慮更多：

### 情境：Timer Interrupt 觸發 Context Switch

```
Task A 在 U-mode 執行
        │
        ▼ (Timer Interrupt)
[進入 trap_vector]
        │
csrrw sp, mscratch, sp  ← 切換到 Kernel Stack
儲存完整 Trap Frame
        │
        ▼
[trap_handler]
  判斷是 Timer Interrupt
  呼叫 scheduler_tick()
        │
        ▼
[scheduler 決定切換到 Task B]
        │
switch_to(task_b)
        │
        ▼
[恢復 Task B 的 Trap Frame]
從 Task B 的 Kernel Stack 恢復暫存器
設定 mscratch = Task B 的 Kernel SP
        │
        ▼
csrrw sp, mscratch, sp  ← 切換到 Task B 的 User Stack
mret
        │
        ▼
Task B 在 U-mode 繼續執行
```

### switch_to 實作

```c
void switch_to(tcb_t *next) {
    tcb_t *prev = current_task;

    // 儲存當前 Task 的 Kernel SP
    // （Trap Frame 已經在 Kernel Stack 上了）
    prev->sp = get_current_sp();

    // 切換 current_task
    current_task = next;

    // 設定新 Task 的 mscratch（下次 Trap 用）
    // mscratch 應該指向 Kernel Stack 頂部
    csr_write(mscratch, next->kernel_stack_top);

    // 切換到新 Task 的 Kernel Stack
    set_sp(next->sp);

    // 返回後會執行 trap_return，恢復 User context
}
```

---

## 第一次啟動 User Task

新建立的 Task 沒有 Trap Frame，因為它從來沒被中斷過。我們需要「偽造」一個：

```c
void task_init_user(tcb_t *task, void (*entry)(void *), void *arg) {
    // 分配 Kernel Stack
    task->kernel_stack_base = kmalloc(KERNEL_STACK_SIZE);
    task->kernel_stack_top = task->kernel_stack_base + KERNEL_STACK_SIZE;

    // 分配 User Stack
    task->user_stack_base = umalloc(USER_STACK_SIZE);
    task->user_stack_top = task->user_stack_base + USER_STACK_SIZE;

    // 在 Kernel Stack 上偽造 Trap Frame
    trap_frame_t *tf = (trap_frame_t *)(
        task->kernel_stack_top - sizeof(trap_frame_t)
    );

    memset(tf, 0, sizeof(trap_frame_t));

    // 設定 User Stack
    tf->sp = (uint64_t)task->user_stack_top;

    // 設定入口點
    tf->mepc = (uint64_t)entry;

    // 設定參數
    tf->a0 = (uint64_t)arg;

    // 設定 mstatus：MPP = 00 (U-mode), MPIE = 1 (返回後開中斷)
    tf->mstatus = MSTATUS_MPIE;  // MPP = 0 已經是 U-mode

    // Task 的 Kernel SP 指向 Trap Frame
    task->sp = (void *)tf;

    task->state = TASK_READY;
}
```

當這個 Task 第一次被 switch_to 選中時：
1. 載入它的 Kernel SP（指向偽造的 Trap Frame）
2. 執行 trap_return
3. 從 Trap Frame 恢復暫存器
4. mret 到 entry 函式
5. 開始在 U-mode 執行

---

## 置物櫃的管理

`mscratch` 是我們的「置物櫃」，需要仔細管理：

### 規則

| 狀態 | mscratch 內容 |
|------|---------------|
| U-mode 執行中 | Kernel Stack Top |
| M-mode 執行中（剛進入）| User SP |
| M-mode 執行中（處理完畢）| User SP |
| 返回 U-mode 後 | Kernel Stack Top |

### Context Switch 時

切換 Task 時，必須同時更新 `mscratch`：

```c
void switch_context(tcb_t *prev, tcb_t *next) {
    // ... 儲存/恢復暫存器 ...

    // 更新 mscratch 為新 Task 的 Kernel Stack
    csr_write(mscratch, next->kernel_stack_top);
}
```

如果忘記這一步，下次 Trap 時會切換到「錯誤的 Kernel Stack」——可能是已經被釋放的記憶體，或是別的 Task 的 Stack。

---

## 完整流程圖

```
                    Task A (U-mode)
                         │
                    [Interrupt]
                         ▼
              ┌──────────────────────┐
              │   trap_vector        │
              │   ─────────────────  │
              │   csrrw sp, mscratch │ ← 穿上無塵服
              │   save registers     │
              │   call handler       │
              └──────────┬───────────┘
                         │
              ┌──────────▼───────────┐
              │   trap_handler (C)   │
              │   ─────────────────  │
              │   判斷中斷類型       │
              │   scheduler_tick()   │
              │   選擇 Task B        │
              └──────────┬───────────┘
                         │
              ┌──────────▼───────────┐
              │   switch_to(B)       │
              │   ─────────────────  │
              │   save A's context   │
              │   load B's context   │
              │   update mscratch    │
              └──────────┬───────────┘
                         │
              ┌──────────▼───────────┐
              │   trap_return        │
              │   ─────────────────  │
              │   restore registers  │
              │   csrrw sp, mscratch │ ← 脫下無塵服
              │   mret               │
              └──────────┬───────────┘
                         │
                         ▼
                    Task B (U-mode)
```

---

## 小結

這一章我們建立了 **Dual Stack** 機制：

1. **問題**：Kernel 使用 User Stack 會帶來安全、可靠性、隔離問題
2. **解法**：每個 Task 有兩個 Stack——Kernel Stack 和 User Stack
3. **機制**：使用 `mscratch` CSR 和 `csrrw` 指令進行原子交換
4. **流程**：進入 M-mode 時切換到 Kernel Stack，返回 U-mode 時切換回 User Stack

現在我們的 Task 有了自己的「更衣室」。進入皇宮前換上防塵服，離開時換回街服。無論 User Stack 上有什麼垃圾，都不會影響 Kernel 的運作。

下一章，我們要建立「請願系統」——System Call。讓 User Task 能夠正式地向 Kernel 提出請求。

---

## 本章重點

| 概念 | 說明 |
|------|------|
| Morris Worm | 1988 年第一個大規模網路攻擊，利用 Stack Buffer Overflow |
| Dual Stack | 每個 Task 有 Kernel Stack 和 User Stack |
| mscratch | RISC-V CSR，用於暫存 Stack Pointer |
| csrrw | 原子交換指令，一條指令完成 SP 切換 |
| Trap Frame | 儲存在 Kernel Stack 上的完整 CPU 狀態 |
| 更衣室理論 | 進無塵室要換防塵服，進 M-mode 要換 Kernel Stack |
