# 動態邊界：SMP 下的 PMP 配置

**作者**: Danny Jiang
**日期**: 2025-12-26

---

## 前言：每個核心都是獨立王國

在 v1.x 的單核世界裡，PMP（Physical Memory Protection）配置很簡單：設定一次，全系統生效。

但在 SMP 世界裡，事情變得複雜了。

**PMP 是 Hart-Local 的。**

這意味著：
- Core 0 的 PMP 設定只影響 Core 0
- Core 1 的 PMP 設定只影響 Core 1
- 兩個核心可以有完全不同的記憶體保護規則

這帶來了一個有趣的問題：**當 Task 從 Core 0 遷移到 Core 1 時，它的記憶體保護怎麼辦？**

---

## 一、PMP 基礎回顧

### 1.1 PMP 是什麼？

PMP 是 RISC-V 的硬體記憶體保護機制。它允許 M-mode 軟體限制 U-mode 程式可以存取的記憶體區域。

```
┌─────────────────────────────────────────────────────────────┐
│                    Physical Memory                           │
├─────────────────────────────────────────────────────────────┤
│ 0x80000000 │ Kernel Code │ PMP: M:RWX, U:None               │
├─────────────────────────────────────────────────────────────┤
│ 0x80020000 │ User Code   │ PMP: M:RW, U:RWX                 │
├─────────────────────────────────────────────────────────────┤
│ 0x80040000 │ Task Stacks │ PMP: Per-Task 配置               │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 PMP 暫存器

RISC-V 提供 16 組 PMP 暫存器（可能更多，取決於實作）：

```c
/* PMP 配置暫存器 */
pmpcfg0, pmpcfg2, ...  /* 每個 64-bit，包含 8 個 8-bit 配置 */

/* PMP 地址暫存器 */
pmpaddr0, pmpaddr1, ... pmpaddr15  /* 每個定義一個區域邊界 */
```

每個 PMP entry 的配置：

```
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ L │ 0 │ 0 │ A │ A │ X │ W │ R │
└───┴───┴───┴───┴───┴───┴───┴───┘
  7   6   5   4   3   2   1   0

L: Lock (鎖定，M-mode 也受限)
A: Address Matching Mode (OFF/TOR/NA4/NAPOT)
X: Execute permission
W: Write permission
R: Read permission
```

---

## 二、SMP 下的 PMP 挑戰

### 2.1 Hart-Local 的問題

考慮這個場景：

```
Time T1: Task A 在 Core 0 執行
         Core 0 PMP: Task A 的 Stack 可存取
         Core 1 PMP: Task B 的 Stack 可存取

Time T2: Scheduler 決定將 Task A 遷移到 Core 1

Time T3: Task A 在 Core 1 執行
         問題：Core 1 的 PMP 還是 Task B 的設定！
         Task A 存取自己的 Stack → Access Fault!
```

### 2.2 兩種解決方案

**方案 A：Per-Task PMP 切換**
- 每次 Context Switch 時，重新配置 PMP
- 優點：完全隔離
- 缺點：PMP 切換有 overhead

**方案 B：Hybrid PMP 策略**
- 靜態區：所有 Task 共用（Kernel、User Code）
- 動態區：Per-Task 配置（Stack）
- 優點：減少切換 overhead
- 缺點：隔離不完全

**danieRTOS v3.x 選擇 Hybrid 策略。**

---

## 三、Hybrid PMP 策略

### 3.1 區域劃分

```
┌─────────────────────────────────────────────────────────────┐
│                    PMP 區域劃分                              │
├─────────────────────────────────────────────────────────────┤
│ PMP 0-3: 靜態區（所有 Task 共用）                            │
│   - Kernel Code & Data: M:RWX, U:None                       │
│   - User Code: M:RW, U:RX                                   │
│   - User Data: M:RW, U:RW                                   │
│   - Peripherals: M:RW, U:None                               │
├─────────────────────────────────────────────────────────────┤
│ PMP 4-7: 動態區（Per-Task 配置）                             │
│   - Task Kernel Stack: M:RW, U:None                         │
│   - Task User Stack: M:RW, U:RW                             │
│   - Task Private Data: M:RW, U:RW                           │
├─────────────────────────────────────────────────────────────┤
│ PMP 8-15: 保留（未來擴展）                                   │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 靜態區配置

靜態區在系統啟動時配置，之後不再改變：

```c
void pmp_init_static(void)
{
    /* PMP 0: Kernel Code & Data (0x80000000 - 0x8001FFFF) */
    /* NAPOT: 128KB region */
    csr_write(pmpaddr0, NAPOT_ADDR(0x80000000, 0x20000));
    
    /* PMP 1: User Code (0x80020000 - 0x8002FFFF) */
    /* NAPOT: 64KB region, U:RX */
    csr_write(pmpaddr1, NAPOT_ADDR(0x80020000, 0x10000));
    
    /* PMP 2: User Data (0x80030000 - 0x8003FFFF) */
    /* NAPOT: 64KB region, U:RW */
    csr_write(pmpaddr2, NAPOT_ADDR(0x80030000, 0x10000));
    
    /* PMP 3: Peripherals (0x10000000 - 0x1FFFFFFF) */
    /* TOR: 256MB region, M:RW only */
    csr_write(pmpaddr3, 0x20000000 >> 2);
    
    /* 配置權限 */
    uint64_t cfg =
        (PMP_NAPOT | PMP_R | PMP_W | PMP_X) << 0 |  /* PMP 0: Kernel */
        (PMP_NAPOT | PMP_R | PMP_X) << 8 |          /* PMP 1: User Code */
        (PMP_NAPOT | PMP_R | PMP_W) << 16 |         /* PMP 2: User Data */
        (PMP_TOR | PMP_R | PMP_W) << 24;            /* PMP 3: Peripherals */
    csr_write(pmpcfg0, cfg);
}
```

### 3.3 動態區配置

動態區在 Context Switch 時更新：

```c
void pmp_switch_task(tcb_t *task)
{
    if (task->kstack_base == NULL) {
        /* M-mode 任務：不需要 PMP 保護 */
        return;
    }

    /* PMP 4: Task Kernel Stack */
    csr_write(pmpaddr4, NAPOT_ADDR(task->kstack_base, KSTACK_SIZE));

    /* PMP 5: Task User Stack */
    csr_write(pmpaddr5, NAPOT_ADDR(task->ustack_base, task->ustack_size));

    /* 配置權限 */
    uint64_t cfg = csr_read(pmpcfg0);
    cfg &= 0xFFFFFFFF;  /* 保留 PMP 0-3 */
    cfg |= ((uint64_t)(PMP_NAPOT | PMP_R | PMP_W) << 32);      /* PMP 4 */
    cfg |= ((uint64_t)(PMP_NAPOT | PMP_R | PMP_W) << 40);      /* PMP 5 */
    csr_write(pmpcfg0, cfg);

    /* 刷新 TLB（如果有的話） */
    asm volatile("sfence.vma" ::: "memory");
}
```

---

## 四、Context Switch 整合

### 4.1 完整的 switch_to 流程

```c
void switch_to(tcb_t *prev, tcb_t *next)
{
    cpu_t *cpu = smp_get_cpu();

    /* 1. 更新當前任務 */
    cpu->current_task = next;

    /* 2. 更新 kernel_trap_sp */
    if (next->kstack_top != NULL) {
        cpu->kernel_trap_sp = (uint64_t)next->kstack_top;
    }

    /* 3. 更新 PMP（如果需要） */
    if (next->kstack_base != NULL) {
        pmp_switch_task(next);
    }

    /* 4. 執行底層 context switch */
    __switch_context(&prev->sp, &next->sp);
}
```

### 4.2 PMP 切換的 Overhead

PMP 切換涉及 CSR 寫入，這有一定的 overhead：

| 操作 | 週期數（估計） |
|------|----------------|
| csr_write(pmpaddr) | 5-10 cycles |
| csr_write(pmpcfg) | 5-10 cycles |
| sfence.vma | 10-50 cycles |
| **總計** | 20-70 cycles |

**這個 overhead 是可接受的**，因為：
- Context Switch 本身就需要數百個週期
- PMP 切換只佔總時間的一小部分
- 安全性的收益遠大於效能損失

---

## 五、記憶體佈局設計

### 5.1 對齊要求

PMP 的 NAPOT 模式要求區域大小是 2 的冪次，且起始地址對齊：

```
NAPOT 區域大小 = 2^(n+3) bytes
起始地址必須對齊到區域大小
```

這影響了我們的 Stack 分配策略：

```c
/* Stack 大小必須是 2 的冪次 */
#define KSTACK_SIZE     4096    /* 4 KB = 2^12 */
#define USTACK_SIZE     8192    /* 8 KB = 2^13 */

/* Stack 分配必須對齊 */
static uint8_t g_user_kstacks[MAX_USER_TASKS][KSTACK_SIZE]
    __attribute__((aligned(KSTACK_SIZE)));
static uint8_t g_user_ustacks[MAX_USER_TASKS][USTACK_SIZE]
    __attribute__((aligned(USTACK_SIZE)));
```

### 5.2 完整記憶體佈局

```
┌─────────────────────────────────────────────────────────────┐
│                    Memory Layout (v3.x)                      │
├─────────────────────────────────────────────────────────────┤
│ 0x80000000 │ Kernel Code (64 KB)  │ PMP 0: M:RWX, U:None    │
│ 0x80010000 │ Kernel Data (64 KB)  │                         │
├─────────────────────────────────────────────────────────────┤
│ 0x80020000 │ User Code (64 KB)    │ PMP 1: M:RW, U:RX       │
├─────────────────────────────────────────────────────────────┤
│ 0x80030000 │ User Data (64 KB)    │ PMP 2: M:RW, U:RW       │
├─────────────────────────────────────────────────────────────┤
│ 0x80040000 │ Stack Pool           │ PMP 4-5: Per-Task       │
│            │                      │                         │
│            │ Task 0 KStack (4KB)  │ 0x80040000-0x80040FFF   │
│            │ Task 0 UStack (8KB)  │ 0x80042000-0x80043FFF   │
│            │                      │                         │
│            │ Task 1 KStack (4KB)  │ 0x80044000-0x80044FFF   │
│            │ Task 1 UStack (8KB)  │ 0x80046000-0x80047FFF   │
│            │                      │                         │
│            │ ...                  │                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 六、Task 遷移的完整流程

### 6.1 遷移場景

```
Core 0                              Core 1
  │                                   │
  │ Task A 執行中                      │ Task B 執行中
  │ PMP: Task A Stack                 │ PMP: Task B Stack
  │                                   │
  │ ◄─── Timer Interrupt ───          │
  ▼                                   │
  │ Scheduler: Task A → Ready         │
  │ Scheduler: 選擇 Task C            │
  │ switch_to(A, C)                   │
  │   - pmp_switch_task(C)            │
  │ PMP: Task C Stack                 │
  │                                   │
  │                                   │ ◄─── Timer Interrupt ───
  │                                   ▼
  │                                   │ Scheduler: Task B → Ready
  │                                   │ Scheduler: 選擇 Task A (遷移!)
  │                                   │ switch_to(B, A)
  │                                   │   - pmp_switch_task(A)
  │                                   │ PMP: Task A Stack
  │                                   │
  │                                   │ Task A 在 Core 1 執行
  │                                   │ 可以正確存取自己的 Stack
```

### 6.2 關鍵觀察

1. **PMP 跟著 Task 走**：無論 Task 在哪個核心執行，PMP 都會正確配置
2. **遷移是透明的**：Task 不需要知道自己被遷移了
3. **安全性保證**：Task 只能存取自己的 Stack

---

## 七、v3p2 的進階功能（預告）

### 7.1 完整的 Fault Isolation

v3p2 計畫實作更完整的隔離：

```c
/* 每個 Task 有獨立的 PMP 配置 */
typedef struct tcb {
    /* ... */
    pmp_config_t pmp_config;  /* 完整的 PMP 設定 */
} tcb_t;

/* Context Switch 時完整切換 PMP */
void pmp_switch_full(tcb_t *task)
{
    /* 恢復這個 Task 的完整 PMP 配置 */
    for (int i = 0; i < 16; i++) {
        csr_write_indexed(pmpaddr, i, task->pmp_config.addr[i]);
    }
    csr_write(pmpcfg0, task->pmp_config.cfg0);
    csr_write(pmpcfg2, task->pmp_config.cfg2);
}
```

### 7.2 動態記憶體保護

```c
/* 允許 Task 動態申請受保護的記憶體區域 */
int sys_mprotect(void *addr, size_t len, int prot)
{
    tcb_t *task = task_get_current();

    /* 驗證地址範圍 */
    if (!is_valid_user_range(addr, len)) {
        return -EINVAL;
    }

    /* 更新 Task 的 PMP 配置 */
    pmp_add_region(task, addr, len, prot);

    /* 立即生效 */
    pmp_switch_task(task);

    return 0;
}
```

---

## 八、本章總結

SMP 下 PMP 配置的核心要點：

| 設計決策 | 理由 |
|----------|------|
| **Hybrid PMP 策略** | 平衡安全性和效能 |
| **靜態區 + 動態區** | 減少 Context Switch overhead |
| **Context Switch 時更新 PMP** | 確保 Task 遷移後 PMP 正確 |
| **Stack 對齊到 2^n** | 滿足 NAPOT 模式要求 |

**經驗法則**：
- PMP 是 Hart-Local 的，每個核心需要獨立配置
- Task 遷移時，PMP 必須跟著更新
- 使用 Hybrid 策略可以減少 overhead

下一章，我們將探討 **多核 Syscall** 的設計，解決多核心同時 syscall 的競爭問題。

---

## 參考資料

**RISC-V 規格**

- [RISC-V Privileged Specification](https://riscv.org/specifications/privileged-isa/)
  Chapter 3.7: Physical Memory Protection

**danieRTOS 系列**

- Ch 17-18: PMP 基礎 (v1.x)
- Ch 22: Per-Core Data (v2.x)

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)
```

