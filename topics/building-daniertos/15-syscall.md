# 15. 國王的信箱：System Calls

> 「當你需要國王辦事，你不會直接闖進皇宮。你會寫一封信，投進信箱，然後等待回覆。」

1969 年，Ken Thompson 和 Dennis Ritchie 在貝爾實驗室的一間辦公室裡，正在設計 Unix。

他們面臨一個根本性的問題：User 程式需要做一些「危險」的事情——讀寫檔案、分配記憶體、傳送網路封包。但如果讓 User 程式直接操作硬體，一個 Bug 就能毀掉整個系統。

解決方案很優雅：**User 程式不能直接操作硬體，但可以「請求」Kernel 代為處理。**

這個請求機制，就是 System Call。

---

## 銀行櫃檯模型

想像你去銀行辦事：

1. **填寫表格**：寫下你要做的事（轉帳？提款？）和相關資訊（金額、帳號）
2. **按鈴**：通知櫃員你準備好了
3. **等待處理**：櫃員驗證你的身份，檢查表格，執行操作
4. **收到結果**：櫃員告訴你成功或失敗，可能還有餘額資訊

System Call 就是這個流程的數位版：

| 銀行 | System Call |
|------|-------------|
| 表格 | 暫存器 a0-a6（參數） |
| 表格編號 | 暫存器 a7（syscall number） |
| 按鈴 | `ecall` 指令 |
| 櫃員處理 | Trap Handler + Syscall Handler |
| 結果 | 暫存器 a0（返回值） |

---

## RISC-V 的 ecall

RISC-V 提供了 `ecall`（Environment Call）指令來觸發 System Call：

```asm
ecall
```

這條指令會：
1. 觸發「Environment call from U-mode」異常（mcause = 8）
2. 跳轉到 `mtvec` 指定的 Trap Handler
3. `mepc` 指向 `ecall` 指令本身

注意第三點：`mepc` 指向 `ecall`，不是下一條指令。這意味著如果我們直接 `mret`，會再次執行 `ecall`，形成無限迴圈。

**正確的做法**：Syscall Handler 在返回前，必須把 `mepc += 4`。

---

## Syscall ABI

我們需要定義一個「合約」：User 如何告訴 Kernel 想做什麼，Kernel 如何回覆結果。

參考 Linux RISC-V ABI：

| 暫存器 | 用途 |
|--------|------|
| a7 | Syscall Number（要做什麼） |
| a0 | 參數 1 / 返回值 |
| a1 | 參數 2 |
| a2 | 參數 3 |
| a3 | 參數 4 |
| a4 | 參數 5 |
| a5 | 參數 6 |
| a6 | 參數 7 |

### danieRTOS Syscall Numbers

```c
#define SYS_YIELD       0
#define SYS_DELAY       1
#define SYS_SEM_WAIT    2
#define SYS_SEM_POST    3
#define SYS_MUTEX_LOCK  4
#define SYS_MUTEX_UNLOCK 5
#define SYS_QUEUE_SEND  6
#define SYS_QUEUE_RECV  7
#define SYS_EXIT        8
```

---

## User Space Wrapper

User Task 不會直接寫 `ecall`，而是呼叫一個 wrapper 函式：

```c
// syscall.h - User Space

static inline long syscall0(long n) {
    register long a7 asm("a7") = n;
    register long a0 asm("a0");
    asm volatile("ecall" : "=r"(a0) : "r"(a7) : "memory");
    return a0;
}

static inline long syscall1(long n, long arg0) {
    register long a7 asm("a7") = n;
    register long a0 asm("a0") = arg0;
    asm volatile("ecall" : "+r"(a0) : "r"(a7) : "memory");
    return a0;
}

static inline long syscall2(long n, long arg0, long arg1) {
    register long a7 asm("a7") = n;
    register long a0 asm("a0") = arg0;
    register long a1 asm("a1") = arg1;
    asm volatile("ecall" : "+r"(a0) : "r"(a7), "r"(a1) : "memory");
    return a0;
}

// 更高階的 wrapper
void sys_yield(void) {
    syscall0(SYS_YIELD);
}

void sys_delay(uint32_t ticks) {
    syscall1(SYS_DELAY, ticks);
}

int sys_sem_wait(sem_t *sem) {
    return syscall1(SYS_SEM_WAIT, (long)sem);
}

int sys_sem_post(sem_t *sem) {
    return syscall1(SYS_SEM_POST, (long)sem);
}
```

---

## Trap Handler 的分流

上一章的 Trap Handler 需要擴展，區分「中斷」和「Syscall」：

```c
void trap_handler(trap_frame_t *tf) {
    uint64_t cause = csr_read(mcause);
    
    if (cause & MCAUSE_INTERRUPT) {
        // 中斷處理
        handle_interrupt(cause & ~MCAUSE_INTERRUPT, tf);
    } else {
        // 異常處理
        handle_exception(cause, tf);
    }
}

void handle_exception(uint64_t cause, trap_frame_t *tf) {
    switch (cause) {
        case CAUSE_USER_ECALL:  // 8: Environment call from U-mode
            handle_syscall(tf);
            break;
        
        case CAUSE_LOAD_ACCESS:
        case CAUSE_STORE_ACCESS:
            handle_access_fault(tf);
            break;
        
        case CAUSE_ILLEGAL_INST:
            handle_illegal_instruction(tf);
            break;
        
        default:
            panic("Unknown exception: %d", cause);
    }
}
```

---

## Syscall Handler

```c
void handle_syscall(trap_frame_t *tf) {
    // 讀取 syscall number 和參數
    long syscall_num = tf->a7;
    long arg0 = tf->a0;
    long arg1 = tf->a1;
    long arg2 = tf->a2;
    
    long ret = 0;
    
    switch (syscall_num) {
        case SYS_YIELD:
            scheduler_yield();
            break;
        
        case SYS_DELAY:
            task_delay(arg0);
            break;
        
        case SYS_SEM_WAIT:
            ret = sem_wait((sem_t *)arg0);
            break;
        
        case SYS_SEM_POST:
            ret = sem_post((sem_t *)arg0);
            break;
        
        case SYS_MUTEX_LOCK:
            ret = mutex_lock((mutex_t *)arg0);
            break;
        
        case SYS_MUTEX_UNLOCK:
            ret = mutex_unlock((mutex_t *)arg0);
            break;

        case SYS_QUEUE_SEND:
            ret = queue_send((queue_t *)arg0, (void *)arg1);
            break;

        case SYS_QUEUE_RECV:
            ret = queue_recv((queue_t *)arg0, (void *)arg1);
            break;

        case SYS_EXIT:
            task_exit();
            break;

        default:
            ret = -1;  // Unknown syscall
            break;
    }

    // 設定返回值
    tf->a0 = ret;

    // 關鍵：mepc += 4，跳過 ecall 指令
    tf->mepc += 4;
}
```

---

## mepc += 4 的重要性

這是個容易犯的錯誤，值得強調：

```
ecall 執行前
        │
        ▼
mepc → [ecall]     ← PC 在這裡
       [下一條指令]

ecall 觸發 Trap
        │
        ▼
mepc 仍指向 [ecall]  ← 異常位置

如果直接 mret...
        │
        ▼
回到 [ecall]
        │
        ▼
又觸發 Trap → 無限迴圈！
```

**正確做法**：

```c
// 在 syscall handler 結束時
tf->mepc += 4;  // 指向下一條指令

// mret 後
// PC = mepc = 原本 ecall 的下一條指令
```

對比其他異常：

| 異常類型 | mepc 處理 |
|----------|-----------|
| ecall (Syscall) | mepc += 4（跳過） |
| Page Fault | 修復後不變（重試） |
| Illegal Instruction | 通常終止 Task |

---

## 指標參數的驗證

我們的 Syscall 有些接受指標參數（如 `sem_t *sem`）。這帶來安全問題：

**問題**：User 傳入的指標可能指向 Kernel 記憶體！

```c
// 惡意 User 程式
sys_sem_wait((sem_t *)0x80000000);  // 指向 Kernel 區域
```

如果 Kernel 盲目地對這個指標操作，可能會：
1. 洩漏 Kernel 資訊
2. 讓 Kernel 寫入錯誤位置
3. 繞過 PMP 保護（因為 Kernel 在 M-mode）

**解法**：驗證所有 User 指標

```c
// 檢查指標是否在 User 區域
static int validate_user_ptr(void *ptr, size_t size) {
    uintptr_t start = (uintptr_t)ptr;
    uintptr_t end = start + size;

    // 檢查是否在 User 記憶體範圍內
    if (start < USER_MEM_START || end > USER_MEM_END) {
        return -1;  // 無效指標
    }

    // 檢查是否溢出
    if (end < start) {
        return -1;
    }

    return 0;  // 有效
}

void handle_syscall(trap_frame_t *tf) {
    // ...

    case SYS_SEM_WAIT:
        if (validate_user_ptr((void *)arg0, sizeof(sem_t)) < 0) {
            ret = -EFAULT;
            break;
        }
        ret = sem_wait((sem_t *)arg0);
        break;

    // ...
}
```

---

## 從 v0.x 到 v1.x：API 變化

### v0.x（直接呼叫）

```c
// User 程式直接呼叫 Kernel 函式
void task_a(void *arg) {
    while (1) {
        sem_wait(&shared_sem);    // 直接呼叫
        // ... 工作 ...
        sem_post(&shared_sem);    // 直接呼叫
        task_delay(100);          // 直接呼叫
    }
}
```

### v1.x（透過 Syscall）

```c
// User 程式透過 Syscall wrapper
void task_a(void *arg) {
    while (1) {
        sys_sem_wait(&shared_sem);   // ecall → Kernel → sem_wait
        // ... 工作 ...
        sys_sem_post(&shared_sem);   // ecall → Kernel → sem_post
        sys_delay(100);              // ecall → Kernel → task_delay
    }
}
```

表面上只是改了函式名稱，但背後發生的事情完全不同：

```
v0.x: task_a() → sem_wait() → return
      ↑ 全部在 M-mode

v1.x: task_a() → sys_sem_wait() → ecall → trap_handler()
                                         → handle_syscall()
                                         → sem_wait()
                                         → mret
      ↑ U-mode    ↑ U-mode        ↑ M-mode ─────────────────┘
```

---

## 完整流程

```
Task A (U-mode)
     │
     ├─ 設定參數：a7 = SYS_DELAY, a0 = 100
     │
     ├─ ecall
     │
     ▼
trap_vector (M-mode)
     │
     ├─ csrrw sp, mscratch, sp  ← 切換 Stack
     ├─ 儲存 Trap Frame
     ├─ call trap_handler
     │
     ▼
trap_handler
     │
     ├─ cause = 8 (User ecall)
     ├─ call handle_syscall
     │
     ▼
handle_syscall
     │
     ├─ syscall_num = a7 = 1 (SYS_DELAY)
     ├─ arg0 = a0 = 100
     ├─ task_delay(100)
     │       │
     │       └─ 設定 delay_ticks = 100
     │          移到 BLOCKED 狀態
     │          trigger scheduler
     │
     ├─ a0 = 返回值
     ├─ mepc += 4  ← 關鍵！
     │
     ▼
trap_return
     │
     ├─ 恢復 Trap Frame
     ├─ csrrw sp, mscratch, sp  ← 切換回 User Stack
     ├─ mret
     │
     ▼
Task B (U-mode)  ← Scheduler 選了別的 Task

... 100 ticks 後 ...

     │
     ▼
Task A (U-mode)
     │
     └─ 從 ecall 的下一條指令繼續
```

---

## 小結

這一章我們建立了 **System Call** 機制：

1. **概念**：User 透過「信箱」向 Kernel 提出請求
2. **觸發**：`ecall` 指令產生異常，跳入 Trap Handler
3. **ABI**：a7 = syscall number, a0-a6 = 參數, a0 = 返回值
4. **關鍵**：Handler 返回前必須 `mepc += 4`
5. **安全**：驗證所有 User 指標

現在 User Task 有了正式的請願管道。他們不能直接闖進皇宮，但可以投遞信件，等待國王處理。

下一章，我們要建立「隱形的力場」——PMP 記憶體保護。讓即使是最惡意的 User Task，也無法越界。

---

## 本章重點

| 概念 | 說明 |
|------|------|
| System Call | User 向 Kernel 請求服務的機制 |
| ecall | RISC-V 指令，觸發 Environment Call 異常 |
| mcause = 8 | Environment call from U-mode |
| ABI | a7 = number, a0-a6 = args, a0 = return |
| mepc += 4 | 跳過 ecall，避免無限迴圈 |
| 指標驗證 | 檢查 User 指標不能指向 Kernel 區域 |
