# 17. 鐵桶江山：錯誤處理與整合驗證

> 「系統不會因為你希望它不出錯，就真的不出錯。」

1985 年到 1987 年間，一台叫做 Therac-25 的放射治療機造成了 6 人死亡。

這不是機械故障，是軟體 Bug。治療機的軟體有一個 Race Condition：當操作員快速修改設定時，軟體會進入錯誤狀態，導致病人接受到致命劑量的輻射。

更糟糕的是，系統沒有任何保護機制。軟體相信自己的狀態總是正確的，完全沒有考慮「萬一出錯怎麼辦」。

Therac-25 事件教會了整個軟體產業一個血的教訓：**系統必須假設錯誤會發生，並且優雅地處理它們。**

---

## 裁判與紅牌

想像一場足球比賽：

- 球員（Task）在場上踢球
- 裁判（Fault Handler）負責維持秩序
- 當球員犯規時，裁判有幾種選擇：
  - **黃牌**：警告，比賽繼續
  - **紅牌**：驅逐出場，但比賽繼續
  - **終止比賽**：極端情況，例如球場起火

在 danieRTOS 中：

| 足球 | RTOS |
|------|------|
| 犯規 | Exception |
| 黃牌 | 記錄 Log，繼續執行 |
| 紅牌 | 終止 Task，其他 Task 繼續 |
| 終止比賽 | Kernel Panic |

---

## Exception 類型總覽

RISC-V 定義了多種異常，我們關心的主要有：

| mcause | 名稱 | 來源 |
|--------|------|------|
| 0 | Instruction address misaligned | 跳轉到未對齊地址 |
| 1 | Instruction access fault | PMP 阻擋執行 |
| 2 | Illegal instruction | 無效指令 |
| 4 | Load address misaligned | 讀取未對齊 |
| 5 | Load access fault | PMP 阻擋讀取 |
| 6 | Store address misaligned | 寫入未對齊 |
| 7 | Store access fault | PMP 阻擋寫入 |
| 8 | Environment call from U-mode | ecall（這是正常的） |

### 分類

```
Exception
    │
    ├── Recoverable（可恢復）
    │       └── ecall (syscall)
    │
    ├── Fault（可處理）
    │       ├── Access Fault（PMP 違規）
    │       ├── Misaligned（未對齊存取）
    │       └── Illegal Instruction
    │
    └── Fatal（致命）
            └── Kernel 自己觸發的異常
```

---

## Trap Handler 架構

```c
void trap_handler(trap_frame_t *tf) {
    uint64_t cause = csr_read(mcause);
    
    if (cause & MCAUSE_INTERRUPT) {
        handle_interrupt(cause & ~MCAUSE_INTERRUPT, tf);
        return;
    }
    
    // Exception handling
    switch (cause) {
        case CAUSE_USER_ECALL:
            handle_syscall(tf);
            break;
        
        case CAUSE_INST_ACCESS:
        case CAUSE_LOAD_ACCESS:
        case CAUSE_STORE_ACCESS:
            handle_access_fault(cause, tf);
            break;
        
        case CAUSE_INST_MISALIGN:
        case CAUSE_LOAD_MISALIGN:
        case CAUSE_STORE_MISALIGN:
            handle_misalign(cause, tf);
            break;
        
        case CAUSE_ILLEGAL_INST:
            handle_illegal_instruction(tf);
            break;
        
        default:
            handle_unknown_exception(cause, tf);
            break;
    }
}
```

---

## 診斷資訊

當異常發生時，我們需要儘可能多的資訊來診斷問題：

### 驗屍報告

```c
void print_exception_info(uint64_t cause, trap_frame_t *tf) {
    uint64_t addr = csr_read(mtval);
    uint64_t mpp = (tf->mstatus >> 11) & 0x3;
    
    kprintf("=== EXCEPTION REPORT ===\n");
    kprintf("Cause:   %s (%ld)\n", exception_name(cause), cause);
    kprintf("PC:      0x%016lx\n", tf->mepc);
    kprintf("Address: 0x%016lx\n", addr);
    kprintf("Mode:    %s\n", mpp == 0 ? "User" : "Machine");
    kprintf("Task:    %s (id=%d)\n", current_task->name, current_task->id);
    kprintf("\n");
    
    kprintf("Registers:\n");
    kprintf("  ra:  0x%016lx  sp:  0x%016lx\n", tf->ra, tf->sp);
    kprintf("  a0:  0x%016lx  a1:  0x%016lx\n", tf->a0, tf->a1);
    kprintf("  a2:  0x%016lx  a3:  0x%016lx\n", tf->a2, tf->a3);
    // ... 更多暫存器 ...
    
    kprintf("========================\n");
}

static const char *exception_name(uint64_t cause) {
    static const char *names[] = {
        [0] = "Instruction address misaligned",
        [1] = "Instruction access fault",
        [2] = "Illegal instruction",
        [4] = "Load address misaligned",
        [5] = "Load access fault",
        [6] = "Store address misaligned",
        [7] = "Store access fault",
        [8] = "Environment call from U-mode",
    };
    if (cause < sizeof(names)/sizeof(names[0]) && names[cause]) {
        return names[cause];
    }
    return "Unknown";
}
```

---

## Access Fault 處理

這是最常見的 Fault——User Task 嘗試存取 Kernel 記憶體：

```c
void handle_access_fault(uint64_t cause, trap_frame_t *tf) {
    print_exception_info(cause, tf);
    
    // 檢查是不是 User Task 觸發的
    uint64_t mpp = (tf->mstatus >> 11) & 0x3;
    if (mpp == 0) {
        // User Task 犯規 → 紅牌
        kprintf("[FAULT] User task '%s' terminated\n", current_task->name);
        task_kill(current_task);
        scheduler_reschedule();
        // 不會返回
    } else {
        // Kernel 自己犯規 → 致命錯誤
        kprintf("[PANIC] Kernel access fault!\n");
        panic("Kernel memory protection violation");
    }
}
```

---

## 終止 Task

當 Task 犯規時，我們需要安全地終止它：

```c
void task_kill(tcb_t *task) {
    // 1. 從任何等待列表中移除
    if (task->state == TASK_BLOCKED) {
        // 如果在等待 semaphore/mutex/queue，移除
        remove_from_wait_list(task);
    }
    
    // 2. 從 ready list 移除
    remove_from_ready_list(task);
    
    // 3. 設定狀態為 DEAD
    task->state = TASK_DEAD;
    
    // 4. 釋放資源（如果持有 mutex）
    release_held_mutexes(task);
    
    // 5. 標記 stack 可回收
    // （實際回收可以延後到 idle task）
    task->flags |= TASK_FLAG_ZOMBIE;
}
```

### Mutex 持有者死亡

這是個棘手的問題：如果 Task 持有 Mutex 時被終止，其他等待的 Task 會永遠等下去。

```c
void release_held_mutexes(tcb_t *task) {
    for (int i = 0; i < task->held_mutex_count; i++) {
        mutex_t *mtx = task->held_mutexes[i];
        
        // 強制釋放
        mtx->owner = NULL;
        mtx->locked = 0;
        
        // 喚醒一個等待者
        if (!list_empty(&mtx->wait_list)) {
            tcb_t *waiter = list_first_entry(&mtx->wait_list, tcb_t, list_node);
            list_del(&waiter->list_node);
            waiter->state = TASK_READY;
            add_to_ready_list(waiter);
        }
    }
}
```

---

## Illegal Instruction 處理

User 執行了無效指令：

```c
void handle_illegal_instruction(trap_frame_t *tf) {
    print_exception_info(CAUSE_ILLEGAL_INST, tf);

    // 讀取觸發異常的指令
    uint32_t inst = *(uint32_t *)tf->mepc;
    kprintf("Faulting instruction: 0x%08x\n", inst);

    // 終止 Task
    kprintf("[FAULT] Illegal instruction, task '%s' terminated\n",
            current_task->name);
    task_kill(current_task);
    scheduler_reschedule();
}
```

### 常見原因

1. **跳轉到資料區**：把資料當成程式碼執行
2. **Stack 溢出**：Return address 被覆蓋成無效值
3. **特權指令**：User 嘗試執行 M-mode 專屬指令（如 `mret`）

---

## Kernel Panic

當 Kernel 自己出錯時，沒有人能拯救它了：

```c
void panic(const char *msg) {
    // 關閉中斷，避免進一步破壞
    csr_clear(mstatus, MSTATUS_MIE);

    kprintf("\n");
    kprintf("========================================\n");
    kprintf("            KERNEL PANIC                \n");
    kprintf("========================================\n");
    kprintf("%s\n", msg);
    kprintf("========================================\n");
    kprintf("\n");

    // 印出所有 Task 狀態（Debug 用）
    print_all_tasks();

    // 停止系統
    while (1) {
        asm volatile("wfi");
    }
}
```

---

## 整合驗證

現在讓我們驗證整個 v1.x 系統的正確性。

### 測試 1：正常 Syscall

```c
void test_syscall(void *arg) {
    kprintf("Test: Normal syscall\n");

    sys_delay(100);
    kprintf("  [PASS] sys_delay returned\n");

    sys_yield();
    kprintf("  [PASS] sys_yield returned\n");

    kprintf("Test PASSED\n");
}
```

### 測試 2：PMP 保護

```c
void test_pmp(void *arg) {
    kprintf("Test: PMP protection\n");

    // 這應該會觸發 Access Fault
    volatile uint64_t *kernel_ptr = (uint64_t *)0x80000000;
    uint64_t x = *kernel_ptr;

    // 這行不應該執行
    kprintf("  [FAIL] Read succeeded: 0x%lx\n", x);
}
```

### 測試 3：Fault 隔離

```c
void test_isolation(void *arg) {
    kprintf("Test: Fault isolation\n");
    kprintf("  Starting bad_task...\n");

    // 等待 bad_task 被終止
    sys_delay(500);

    kprintf("  I'm still alive!\n");
    kprintf("Test PASSED\n");
}

void bad_task(void *arg) {
    // 故意觸發 Access Fault
    *(volatile uint64_t *)0x80000000 = 0xDEADBEEF;
    // 永遠不會到這裡
}
```

### 測試 4：Semaphore + Syscall

```c
static sem_t sem;

void producer_task(void *arg) {
    for (int i = 0; i < 5; i++) {
        kprintf("Producer: posting %d\n", i);
        sys_sem_post(&sem);
        sys_delay(100);
    }
}

void consumer_task(void *arg) {
    for (int i = 0; i < 5; i++) {
        sys_sem_wait(&sem);
        kprintf("Consumer: got %d\n", i);
    }
    kprintf("Consumer: done\n");
}
```

---

## 預期輸出

```
=== danieRTOS v1.x Integration Test ===

Test: Normal syscall
  [PASS] sys_delay returned
  [PASS] sys_yield returned
Test PASSED

Test: PMP protection
=== EXCEPTION REPORT ===
Cause:   Load access fault (5)
PC:      0x00000000801001a4
Address: 0x0000000080000000
Mode:    User
Task:    test_pmp (id=2)
========================
[FAULT] User task 'test_pmp' terminated

Test: Fault isolation
  Starting bad_task...
=== EXCEPTION REPORT ===
Cause:   Store access fault (7)
PC:      0x0000000080100200
Address: 0x0000000080000000
Mode:    User
Task:    bad_task (id=3)
========================
[FAULT] User task 'bad_task' terminated
  I'm still alive!
Test PASSED

Test: Semaphore + Syscall
Producer: posting 0
Consumer: got 0
Producer: posting 1
Consumer: got 1
...
Consumer: done

=== All tests passed ===
```

---

## 小結

這一章我們建立了完整的 **錯誤處理機制**：

1. **分類**：Recoverable、Fault、Fatal
2. **診斷**：印出詳細的異常報告
3. **處理**：User Fault → 終止 Task，Kernel Fault → Panic
4. **隔離**：一個 Task 的錯誤不影響其他 Task
5. **驗證**：整合測試確認系統行為

現在 danieRTOS v1.x 是個「鐵桶江山」——即使有惡意或有 Bug 的 Task，系統也能繼續運行。這正是 Therac-25 事件教給我們的教訓：**永遠假設會出錯，並優雅地處理它們。**

下一章，我們進入「偵探辦案」——如何在雙重世界（M-mode + U-mode）中 Debug。

---

## 本章重點

| 概念 | 說明 |
|------|------|
| Therac-25 | 1985-87 放射治療機事故，軟體 Bug 致死 |
| mcause | 異常原因代碼 |
| mtval | 觸發異常的地址或指令 |
| task_kill | 終止 Task，清理資源 |
| Kernel Panic | Kernel 自身錯誤，系統停止 |
| Fault 隔離 | 一個 Task 的錯誤不影響其他 Task |
