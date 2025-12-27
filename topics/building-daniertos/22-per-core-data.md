# 各有所屬：Per-Core Data

**作者**: Danny Jiang
**日期**: 2025-12-14

---

## 前言：誰是「當前 Task」？

在單核心的 v0.x 世界裡，有一個全域變數：

```c
tcb_t *g_current_task;  // 全域唯一
```

Scheduler 把它指向正在執行的 Task，中斷處理時用它來保存 Context，一切都很簡單。

但在多核心的世界裡，這個設計立刻崩潰。

想像 Core 0 正在執行 Task A，Core 1 正在執行 Task B：

```
Core 0: g_current_task = TaskA
Core 1: g_current_task = TaskB  // 覆蓋了！
```

兩個核心同時寫入同一個變數，結果是未定義的混亂。

**解決方案**：每個核心需要「自己的」current_task。這就是 **Per-Core Data** 的概念。

---

## 一、Per-Core Data 設計

### 1.1 什麼需要 Per-Core？

讓我們分類所有的系統資料：

| 類別 | 資料 | 原因 |
|------|------|------|
| **Per-Core** | `current_task` | 每個核心執行不同的 Task |
| **Per-Core** | `idle_task` | 每個核心需要自己的 Idle Task |
| **Per-Core** | `irq_nest_depth` | 中斷巢狀深度是核心獨立的 |
| **Per-Core** | `scheduler_lock` | (可選) Per-Core 的排程鎖 |
| **Shared** | `ready_queue` | 所有核心共享同一個 Ready Queue |
| **Shared** | `semaphore_list` | Semaphore 是全域資源 |
| **Shared** | `mutex_list` | Mutex 是全域資源 |

### 1.2 Per-Core 資料結構

```c
#include <stdint.h>

#define MAX_CORES 4
#define CACHE_LINE_SIZE 64

// Per-Core 資料結構
typedef struct {
    // 身份識別
    uint64_t hartid;

    // 排程器狀態
    tcb_t *current_task;    // 當前執行的 Task
    tcb_t *idle_task;       // 這個核心的 Idle Task

    // 中斷狀態
    int32_t irq_nest_depth; // 0 = Thread mode, >0 = Interrupt mode

    // Trap 用的暫存空間
    uint64_t scratch[8];    // 保存臨時暫存器

    // Padding: 避免 False Sharing
    uint8_t padding[CACHE_LINE_SIZE];

} __attribute__((aligned(CACHE_LINE_SIZE))) cpu_t;

// 全域陣列
cpu_t g_cpus[MAX_CORES];
```

### 1.3 為什麼要 Cache Line 對齊？

這是為了避免 **False Sharing**。

現代 CPU 的 Cache 以「Cache Line」為單位（通常 64 bytes）。如果兩個核心修改同一個 Cache Line 中的不同變數，硬體會認為它們在「競爭」，導致 Cache 不斷失效：

```
不對齊的情況：
┌────────────────────────────────────────────────────────────────┐
│ Cache Line (64 bytes)                                          │
├────────────────────────────┬───────────────────────────────────┤
│ cpu[0].current_task        │ cpu[1].current_task               │
│ (Core 0 寫入)              │ (Core 1 寫入)                     │
└────────────────────────────┴───────────────────────────────────┘
          ↑                              ↑
          Core 0 寫入                    Core 1 寫入
          → 整個 Cache Line 失效！       → 整個 Cache Line 失效！
```

對齊後：

```
┌────────────────────────────────────────────────────────────────┐
│ Cache Line 0 (64 bytes) - cpu[0]                               │
├────────────────────────────────────────────────────────────────┤
│ current_task, idle_task, irq_nest_depth, padding...           │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐

### 2.1 方法 A：每次讀取 mhartid（慢）

最直覺的方法是每次需要 Per-Core 資料時，讀取 `mhartid` 並計算偏移：

```c
cpu_t *get_cpu_slow(void) {
    uint64_t hartid;
    asm volatile("csrr %0, mhartid" : "=r"(hartid));
    return &g_cpus[hartid];
}

tcb_t *get_current_task_slow(void) {
    return get_cpu_slow()->current_task;
}
```

**優點**：簡單，不需要額外設定

**缺點**：

- 每次都要執行 `csrr` 指令（~2-5 cycles）
- 每次都要計算陣列偏移
- 在高頻呼叫的路徑上（如中斷處理），overhead 明顯

### 2.2 方法 B：使用 tp 暫存器（快）

RISC-V 保留 `x4` 暫存器作為 **Thread Pointer (tp)**。在 OS 核心中，我們可以用它來指向當前核心的 `cpu_t`。

```c
// 使用 GCC 的 register variable 語法
register cpu_t *my_cpu asm("tp");

static inline cpu_t *get_cpu(void) {
    return my_cpu;
}

static inline tcb_t *get_current_task(void) {
    return my_cpu->current_task;
}
```

**優點**：

- 零計算！`tp` 已經指向正確的位址
- 單一指令存取：`ld a0, offset(tp)`
- Linux Kernel 也使用這個方法

**缺點**：

- 編譯器少了一個可用暫存器（影響極小）
- 需要在啟動時正確設定 `tp`

### 2.3 效能比較

| 方法 | 指令數 | 預估 Cycles |
|------|--------|-------------|
| mhartid 方法 | 5-7 | 10-15 |
| tp 方法 | 1 | 1-2 |

在中斷處理路徑上，每個 tick 都會呼叫 `get_current_task()`，使用 tp 方法可以節省大量 cycles。

**結論**：我們使用 **tp 暫存器方法**。

---

## 三、設定 tp 暫存器

### 3.1 在 startup.S 中設定

`tp` 必須在跳入 C 程式碼之前設定好：

```asm
# ========================================
# 設定 tp 暫存器指向 Per-Core 資料
# tp = &g_cpus[mhartid]
# ========================================

setup_tp:
    # 1. 取得 g_cpus 的基底位址
    la   t0, g_cpus

    # 2. 計算 offset = mhartid × sizeof(cpu_t)
    csrr t1, mhartid
    li   t2, 128            # sizeof(cpu_t) = 128 bytes
    mul  t2, t2, t1         # offset = hartid × 128

    # 3. tp = base + offset
    add  tp, t0, t2

    # 4. 順便把 hartid 存到結構體裡
    sd   t1, 0(tp)          # cpu->hartid = mhartid
```

**注意**：`sizeof(cpu_t)` 必須與 C 語言定義一致！如果結構改變，Assembly 也要更新。

### 3.2 完整的 startup.S（含 tp 設定）

```asm
.section .text
.global _start

.equ STACK_SIZE, 4096
.equ CPU_SIZE, 128          # sizeof(cpu_t)

_start:
    csrw mie, zero
    csrr t0, mhartid
    bnez t0, secondary_hart_entry

primary_hart_entry:
    # Stack
    la   sp, _stack_top

    # tp = &g_cpus[0]
    la   tp, g_cpus
    csrr t0, mhartid
    sd   t0, 0(tp)          # cpu->hartid = 0

    # BSS, trap vector, etc.
    # ...

    # Boot flag
    fence w, w
    li   t0, 1
    la   t1, smp_boot_flag
    sw   t0, 0(t1)
    fence w, r

    tail main

secondary_hart_entry:
    # Wait for boot flag
    la   t1, smp_boot_flag
wait_loop:
    lw   t2, 0(t1)
    bnez t2, secondary_setup
    wfi
    j    wait_loop

secondary_setup:
    fence r, rw

    # Stack: sp = _stack_top - (hartid × STACK_SIZE)
    la   t0, _stack_top
    csrr t1, mhartid
    li   t2, STACK_SIZE
    mul  t2, t2, t1
    sub  sp, t0, t2

    # tp = &g_cpus[hartid]
    la   t0, g_cpus
    csrr t1, mhartid
    li   t2, CPU_SIZE
    mul  t2, t2, t1
    add  tp, t0, t2
    sd   t1, 0(tp)          # cpu->hartid = hartid

    # Trap vector
    la   t0, trap_vector
    csrw mtvec, t0

    tail secondary_main
```

---

## 四、C 語言存取

### 4.1 標頭檔定義

```c
// cpu.h
#ifndef CPU_H
#define CPU_H

#include <stdint.h>
#include "task.h"

#define MAX_CORES 4
#define CACHE_LINE_SIZE 64

typedef struct {
    uint64_t hartid;
    tcb_t *current_task;
    tcb_t *idle_task;
    int32_t irq_nest_depth;
    uint64_t scratch[8];
    uint8_t padding[CACHE_LINE_SIZE - 24];
} __attribute__((aligned(CACHE_LINE_SIZE))) cpu_t;

extern cpu_t g_cpus[MAX_CORES];

// tp 暫存器綁定
register cpu_t *my_cpu asm("tp");

// 快速存取巨集
#define get_cpu()           (my_cpu)
#define get_current_task()  (my_cpu->current_task)
#define get_hartid()        (my_cpu->hartid)
#define get_irq_depth()     (my_cpu->irq_nest_depth)

// 設定巨集
#define set_current_task(t) (my_cpu->current_task = (t))

#endif // CPU_H
```

### 4.2 使用範例

```c
#include "cpu.h"

void scheduler_tick(void) {
    cpu_t *cpu = get_cpu();

    // 檢查是否在中斷中
    if (cpu->irq_nest_depth > 1) {
        return;  // 巢狀中斷，不排程
    }

    tcb_t *current = cpu->current_task;

    // 減少時間片
    if (current->time_slice > 0) {
        current->time_slice--;
    }

    // 時間片用完，觸發排程
    if (current->time_slice == 0) {
        sched_yield();
    }
}
```

### 4.3 中斷處理中的使用

```c
void trap_handler(void) {
    cpu_t *cpu = get_cpu();

    // 進入中斷：增加巢狀計數
    cpu->irq_nest_depth++;

    // 處理中斷...
    uint64_t mcause = read_csr(mcause);
    handle_interrupt(mcause);

    // 離開中斷：減少巢狀計數
    cpu->irq_nest_depth--;

    // 如果回到 Thread mode，檢查是否需要排程
    if (cpu->irq_nest_depth == 0) {
        check_reschedule();
    }
}
```

---

## 五、Per-Core Idle Task

### 5.1 為什麼每個核心需要 Idle Task？

當一個核心沒有 Ready Task 可以執行時，它不能停下來——CPU 必須執行「某些東西」。

選項：

1. **Busy Loop**：`while (1) {}`——浪費電力
2. **WFI 指令**：進入低功耗等待，直到中斷喚醒
3. **Idle Task**：專門的 Task，執行 WFI 並在需要時被搶占

我們選擇 **Idle Task** 方案，因為它與 RTOS 的 Task 模型一致。

### 5.2 建立 Idle Task

```c
void idle_task_func(void *param) {
    uint64_t hartid = (uint64_t)param;

    while (1) {
        // 進入低功耗模式
        asm volatile("wfi");

        // 被中斷喚醒後，檢查是否有工作
        // (Scheduler 會自動處理)
    }
}

void idle_task_create(uint64_t hartid) {
    tcb_t *idle = task_create_internal(
        idle_task_func,
        (void *)hartid,
        "Idle",
        0,  // 最低優先權
        hartid  // CPU Affinity
    );

    // 設定為這個核心的 Idle Task
    g_cpus[hartid].idle_task = idle;
}
```

### 5.3 Scheduler 中的使用

```c
tcb_t *pick_next_task(void) {
    cpu_t *cpu = get_cpu();

    // 從 Ready Queue 中找最高優先權的 Task
    tcb_t *next = ready_queue_pop();

    // 如果沒有 Ready Task，返回 Idle Task
    if (next == NULL) {
        next = cpu->idle_task;
    }

    return next;
}
```

---

## 六、本章回顧

在這一章，我們設計了 Per-Core 資料結構：

1. **cpu_t 結構**：包含 current_task, idle_task, irq_nest_depth
2. **tp 暫存器**：在啟動時設定，指向當前核心的 cpu_t
3. **Cache Line 對齊**：避免 False Sharing
4. **Per-Core Idle Task**：每個核心有自己的 Idle Task

關鍵程式碼：

```c
// 定義 tp 綁定
register cpu_t *my_cpu asm("tp");

// 快速存取
#define get_current_task() (my_cpu->current_task)
```

```asm
# 在 startup.S 中設定 tp
la   t0, g_cpus
csrr t1, mhartid
li   t2, 128              # sizeof(cpu_t)
mul  t2, t2, t1
add  tp, t0, t2
```

下一章，我們將實作 **Spinlock**——保護共享資料的關鍵工具。

---

## 參考資料

- **See RISC-V Run**, Ch 11 - Per-CPU Data and tp Register
- **Data Structures in Practice**, Ch 13 - False Sharing
- [Linux Kernel percpu](https://www.kernel.org/doc/html/latest/core-api/this_cpu_ops.html)

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。
