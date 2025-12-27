# 雙重保險：Per-Task Kernel Stack 的必要性

**作者**: Danny Jiang
**日期**: 2025-12-26

---

## 前言：一個 Stack 不夠用

在 v2.x 的純 M-mode 世界裡，每個任務只需要一個 Stack。簡單、直接、沒有問題。

但當我們加入 User Mode 後，事情變得複雜了。

想像一個場景：User Task A 正在執行，突然 Timer 中斷發生。Kernel 需要保存 A 的狀態，但問題來了：

**我們應該把狀態保存在哪裡？**

- **User Stack？** 不行！User 程式可能故意把 sp 設成無效值，或者 Stack 已經溢出。
- **Per-Core Kernel Stack？** 看起來可以，但有隱藏的問題...

這一章，我們要探討為什麼 danieRTOS v3.x 選擇了 **Per-Task Kernel Stack** 設計。

---

## 一、Per-Core vs Per-Task：兩種設計

### 1.1 Per-Core Kernel Stack

這是最直覺的設計：每個 CPU 核心有一個共用的 Kernel Stack。

```
Core 0                          Core 1
┌─────────────────┐             ┌─────────────────┐
│ Kernel Stack 0  │             │ Kernel Stack 1  │
│ (所有 Task 共用) │             │ (所有 Task 共用) │
└─────────────────┘             └─────────────────┘
        ▲                               ▲
        │                               │
   Task A, B, C                    Task D, E, F
   (輪流使用)                       (輪流使用)
```

**優點**：
- 記憶體效率高（只需要 N 個 Stack，N = 核心數）
- 實作簡單

**缺點**：
- **Kernel 內不能阻塞**：如果 Task A 在 Kernel 中阻塞，它的 context 還在 Stack 上，Task B 無法使用這個 Stack
- **Task 遷移困難**：如果 Task 從 Core 0 遷移到 Core 1，它的 Kernel context 在哪？

### 1.2 Per-Task Kernel Stack

每個 Task 有自己專屬的 Kernel Stack：

```
Task A                Task B                Task C
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│ User Stack A │      │ User Stack B │      │ User Stack C │
├──────────────┤      ├──────────────┤      ├──────────────┤
│Kernel Stack A│      │Kernel Stack B│      │Kernel Stack C│
└──────────────┘      └──────────────┘      └──────────────┘
```

**優點**：
- **Kernel 內可以阻塞**：每個 Task 的 Kernel context 獨立保存
- **Task 遷移自然**：Kernel Stack 跟著 Task 走
- **更安全**：一個 Task 的 Kernel Stack 溢出不會影響其他 Task

**缺點**：
- 記憶體使用較多（每個 Task 多一個 Stack）

---

## 二、為什麼 v3.x 選擇 Per-Task？

### 2.1 Kernel 內阻塞的需求

考慮這個場景：

```c
void user_task_a(void *arg) {
    /* User Mode 執行 */
    sys_mutex_lock(&shared_mutex);  /* Syscall 進入 Kernel */
    /* ... 如果 mutex 被佔用，需要阻塞 ... */
}
```

當 `sys_mutex_lock` 發現 mutex 被佔用時，Kernel 需要：
1. 保存 Task A 的狀態
2. 切換到另一個 Task
3. 等 mutex 釋放後，恢復 Task A

**如果使用 Per-Core Stack**：
- Task A 的 Kernel context 在 Core 0 的 Stack 上
- 切換到 Task B 時，Task B 也要用 Core 0 的 Stack
- **衝突！** Task A 的 context 會被覆蓋

**如果使用 Per-Task Stack**：
- Task A 的 Kernel context 在 Task A 的 Kernel Stack 上
- 切換到 Task B 時，使用 Task B 的 Kernel Stack
- **沒有衝突！**

### 2.2 SMP 下的 Task 遷移

在 SMP 系統中，Task 可能在不同核心間遷移：

```
Time T1: Task A 在 Core 0 執行
Time T2: Task A 被 Timer 中斷，context 保存
Time T3: Core 1 的 Scheduler 選中 Task A
Time T4: Task A 在 Core 1 恢復執行
```

**如果使用 Per-Core Stack**：
- T2 時，Task A 的 context 在 Core 0 的 Stack
- T4 時，Core 1 需要存取 Core 0 的 Stack
- **跨核心存取，複雜且低效**

**如果使用 Per-Task Stack**：
- Task A 的 context 永遠在 Task A 的 Stack
- 無論在哪個核心執行，都能正確存取
- **自然且高效**

---

## 三、實作細節

### 3.1 TCB 結構擴展

```c
typedef struct tcb {
    /* ... 基本欄位 ... */
    
    /* Stack 相關 */
    reg_t   *stack_base;    /* 相容性：指向主要 stack */
    size_t   stack_size;
    
    /* v3.x 新增：雙 Stack 支援 */
    reg_t   *ustack_base;   /* User Stack 基址 */
    size_t   ustack_size;   /* User Stack 大小 */
    reg_t   *kstack_base;   /* Kernel Stack 基址 */
    reg_t   *kstack_top;    /* Kernel Stack 頂端（用於 trap entry） */
    
    /* ... 其他欄位 ... */
} tcb_t;
```

### 3.2 Task 創建時分配雙 Stack

```c
tcb_t *task_create_user(const char *name, task_func_t func, 
                        void *arg, priority_t priority, uint32_t affinity)
{
    /* 分配 Kernel Stack */
    void *kstack = alloc_kstack();  /* 4KB */
    reg_t *kstack_top = (reg_t *)((reg_t)kstack + KSTACK_SIZE);
    kstack_top = (reg_t *)((reg_t)kstack_top & ~0xF);  /* 16-byte align */
    
    /* 分配 User Stack */
    void *ustack = alloc_ustack();  /* 8KB */
    reg_t *ustack_top = (reg_t *)((reg_t)ustack + USTACK_SIZE);
    ustack_top = (reg_t *)((reg_t)ustack_top & ~0xF);
    
    /* 設置 TCB */
    task->kstack_base = kstack;
    task->kstack_top = kstack_top;
    task->ustack_base = ustack;
    task->ustack_size = USTACK_SIZE;
    
    /* 初始 context 放在 Kernel Stack 上 */
    task->sp = stack_init(kstack_top, func, arg, TASK_PRIV_U);
    /* 設置 context 中的 sp 為 User Stack */
    task->sp[REG_SP_IDX] = (reg_t)ustack_top;

    return task;
}
```

### 3.3 Context Switch 時更新 cpu_t

每次切換到新 Task 時，需要更新 `cpu_t.kernel_trap_sp`：

```c
void switch_to(tcb_t *prev, tcb_t *next)
{
    cpu_t *cpu = smp_get_cpu();

    /* 更新當前任務 */
    cpu->current_task = next;

    /* 關鍵：更新 kernel_trap_sp */
    if (next->kstack_top != NULL) {
        /* U-mode 任務：使用它的 Kernel Stack */
        cpu->kernel_trap_sp = (uint64_t)next->kstack_top;
    } else {
        /* M-mode 任務：不需要特別設置 */
        /* trap 時會直接使用當前 sp */
    }

    /* 執行底層 context switch */
    __switch_context(&prev->sp, &next->sp);
}
```

**這確保了下次 trap 時，`trap_entry` 會載入正確的 Kernel Stack。**

---

## 四、記憶體佈局

### 4.1 Stack 分配策略

```
┌─────────────────────────────────────────────────────────────┐
│                    Memory Layout                             │
├─────────────────────────────────────────────────────────────┤
│ 0x80000000 - 0x8001FFFF │ Kernel Code & Data (128 KB)       │
├─────────────────────────────────────────────────────────────┤
│ 0x80020000 - 0x8003FFFF │ User Code & Data (128 KB)         │
├─────────────────────────────────────────────────────────────┤
│ 0x80040000+             │ Stack Pool                         │
│                         │                                    │
│   Task 0: ┌─────────────────────────────────────┐           │
│           │ Kernel Stack (4 KB)                 │           │
│           ├─────────────────────────────────────┤           │
│           │ User Stack (8 KB)                   │           │
│           └─────────────────────────────────────┘           │
│                                                              │
│   Task 1: ┌─────────────────────────────────────┐           │
│           │ Kernel Stack (4 KB)                 │           │
│           ├─────────────────────────────────────┤           │
│           │ User Stack (8 KB)                   │           │
│           └─────────────────────────────────────┘           │
│                                                              │
│   ...                                                        │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Stack 大小考量

| Stack 類型 | 大小 | 用途 |
|------------|------|------|
| **Kernel Stack** | 4 KB | Trap 處理、Syscall、Kernel 函數呼叫 |
| **User Stack** | 8 KB | User 程式執行、函數呼叫、局部變數 |
| **Idle Stack** | 1 KB | 只執行 `wfi`，幾乎不需要 Stack |

**為什麼 Kernel Stack 較小？**
- Kernel 程式碼經過審查，不會有深度遞迴
- Syscall 處理通常很淺
- 如果需要更多空間，可以動態調整

**為什麼 User Stack 較大？**
- User 程式行為不可預測
- 可能有深度函數呼叫
- 需要空間給局部變數和陣列

---

## 五、Trap 流程中的 Stack 切換

### 5.1 完整流程圖

```
User Mode                                    Kernel Mode
    │                                            │
    │ Task A 執行中                               │
    │ sp = User Stack A                          │
    │                                            │
    │ ◄─── Timer Interrupt ───                   │
    ▼                                            │
┌─────────────────────────────────────────────────────────────┐
│ trap_entry:                                                 │
│   csrrw tp, mscratch, tp    # tp = &cpu_t                   │
│   sd sp, 8(tp)              # 保存 user sp                  │
│   ld sp, 0(tp)              # sp = cpu_t.kernel_trap_sp     │
│                             # = Task A 的 Kernel Stack Top  │
└─────────────────────────────────────────────────────────────┘
    │                                            │
    ▼                                            │
┌─────────────────────────────────────────────────────────────┐
│ 在 Task A 的 Kernel Stack 上：                              │
│   - 保存所有暫存器                                          │
│   - 呼叫 trap_handler()                                     │
│   - Scheduler 決定切換到 Task B                             │
└─────────────────────────────────────────────────────────────┘
    │                                            │
    ▼                                            │
┌─────────────────────────────────────────────────────────────┐
│ switch_to(Task A, Task B):                                  │
│   - 保存 Task A 的 callee-saved regs                        │
│   - cpu->kernel_trap_sp = Task B 的 Kernel Stack Top        │
│   - 恢復 Task B 的 callee-saved regs                        │
│   - sp 現在指向 Task B 的 Kernel Stack                      │
└─────────────────────────────────────────────────────────────┘
    │                                            │
    ▼                                            │
┌─────────────────────────────────────────────────────────────┐
│ trap_exit:                                                  │
│   - 從 Task B 的 Kernel Stack 恢復暫存器                    │
│   - sp = Task B 的 User Stack                               │
│   - mret 返回 User Mode                                     │
└─────────────────────────────────────────────────────────────┘
    │                                            │
    ▼                                            │
    │ Task B 執行中                               │
    │ sp = User Stack B                          │
```

### 5.2 關鍵觀察

1. **Trap Entry 時**：sp 從 User Stack 切換到 Kernel Stack
2. **Context Switch 時**：更新 `cpu_t.kernel_trap_sp` 為新 Task 的 Kernel Stack
3. **Trap Exit 時**：sp 從 Kernel Stack 切換回 User Stack

**每個 Task 的 Kernel context 永遠保存在自己的 Kernel Stack 上。**

---

## 六、M-mode Task 的特殊處理

M-mode Task（如 Monitor）不需要雙 Stack：

```c
tcb_t *task_create(const char *name, task_func_t func,
                   void *arg, priority_t priority,
                   void *stack, size_t stack_size, uint32_t affinity)
{
    /* M-mode 任務：單一 Stack */
    task->stack_base = stack;
    task->stack_size = stack_size;
    task->ustack_base = stack;   /* 相容性 */
    task->ustack_size = stack_size;
    task->kstack_base = NULL;    /* 無獨立 Kernel Stack */
    task->kstack_top = NULL;

    /* 初始 context 放在這個 Stack 上 */
    task->sp = stack_init(stack_top, func, arg, TASK_PRIV_M);

    return task;
}
```

**當 M-mode Task 被 trap 時**：
- `trap_entry` 檢測到 `mscratch = 0`
- 知道這是 M-mode Task，不做 Stack 切換
- 直接在當前 Stack 上保存 context

---

## 七、除錯技巧

### 7.1 常見問題

| 症狀 | 可能原因 | 檢查方法 |
|------|----------|----------|
| Trap 後 sp 無效 | kernel_trap_sp 未更新 | 檢查 switch_to 是否正確設置 |
| Context 被覆蓋 | 使用了 Per-Core Stack | 確認每個 Task 有獨立 Kernel Stack |
| Stack Overflow | Kernel Stack 太小 | 增加 KSTACK_SIZE |
| 遷移後崩潰 | Kernel Stack 地址錯誤 | 確認 kstack_top 正確設置 |

### 7.2 GDB 除錯

```gdb
# 檢查 Task 的 Stack 設置
(gdb) print task->kstack_base
(gdb) print task->kstack_top
(gdb) print task->ustack_base

# 檢查 cpu_t.kernel_trap_sp
(gdb) print ((cpu_t*)$tp)->kernel_trap_sp

# 在 switch_to 設斷點
(gdb) break switch_to
(gdb) commands
> print next->name
> print next->kstack_top
> print ((cpu_t*)$tp)->kernel_trap_sp
> continue
> end
```

---

## 八、本章總結

Per-Task Kernel Stack 的設計要點：

| 設計決策 | 理由 |
|----------|------|
| **每個 U-mode Task 有獨立 Kernel Stack** | 支援 Kernel 內阻塞、Task 遷移 |
| **M-mode Task 使用單一 Stack** | 簡化設計，節省記憶體 |
| **Context Switch 時更新 kernel_trap_sp** | 確保下次 trap 使用正確的 Stack |
| **Kernel Stack 較小 (4KB)** | Kernel 程式碼可控，不需要大 Stack |

**經驗法則**：
- 如果你的 RTOS 需要支援 Kernel 內阻塞（如 mutex、semaphore），使用 Per-Task Kernel Stack
- 如果只是簡單的 Syscall 處理，Per-Core Stack 可能足夠

下一章，我們將探討 **PMP 在 SMP 下的配置**，解決 Task 遷移時的記憶體保護問題。

---

## 參考資料

**作業系統設計**

- Linux Kernel: Per-Thread Kernel Stack
- FreeRTOS: Task Stack Management

**danieRTOS 系列**

- Ch 14: User Mode Stack 設計 (v1.x)
- Ch 22: Per-Core Data (v2.x)

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)
```

