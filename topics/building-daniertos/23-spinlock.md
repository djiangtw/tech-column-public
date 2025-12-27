# 自旋的藝術：Spinlock

**作者**: Danny Jiang
**日期**: 2025-12-14

---

## 前言：當關閉中斷不夠用

在單核心的 v0.x 和 v1.x 世界裡，我們用 `critical_enter()` 關閉中斷來保護共享資源：

```c
void critical_enter(void) {
    asm volatile("csrc mstatus, 0x8");  // 關閉 MIE
}
```

這很有效——關閉中斷後，沒有其他程式碼會執行，自然不會有競爭條件。

但在多核心世界裡，這招失效了。

Core 0 關閉自己的中斷，只是阻止了自己被打斷。Core 1、Core 2、Core 3 仍然在全速運行，隨時可能存取同一個資料結構。

我們需要一個新工具：**Spinlock**。

---

## 一、Spinlock 基本概念

### 1.1 什麼是 Spinlock？

Spinlock 是一種**忙碌等待鎖 (Busy-Waiting Lock)**。

當一個核心想要獲取鎖但鎖已被佔用時，它不會睡眠（因為沒有 OS 可以切換），而是不斷地「自旋」檢查鎖是否釋放：

```c
while (lock == LOCKED) {
    // 自旋等待
}
lock = LOCKED;  // 獲取鎖
```

### 1.2 Spinlock vs Mutex

| 特性 | Spinlock | Mutex |
|------|----------|-------|
| 等待方式 | 忙碌等待 (自旋) | 睡眠等待 |
| CPU 使用 | 100%（等待時浪費 CPU） | 0%（等待時不佔用 CPU） |
| Context Switch | 無 | 有 |
| 適用場景 | 短時間持有、中斷處理 | 長時間持有、一般任務 |
| SMP 支援 | 是（設計目的） | 需要額外處理 |

### 1.3 為什麼 Spinlock 比 Mutex 更適合 SMP？

在 SMP 環境中，Scheduler 本身也是共享資源。如果我們用 Mutex 保護 Scheduler，當一個核心等待 Mutex 時，它會嘗試呼叫 Scheduler 來切換 Task——但 Scheduler 正是被 Mutex 保護的東西！

這就是「先有雞還是先有蛋」的問題。Spinlock 不需要 Scheduler，因此可以用來保護 Scheduler 本身。

---

## 二、原子操作基礎

### 2.1 為什麼需要原子操作？

考慮這個「天真」的 Spinlock 實作：

```c
// 錯誤的實作！
void spinlock_acquire(spinlock_t *lock) {
    while (lock->locked == 1) {}  // (1) 讀取
    lock->locked = 1;              // (2) 寫入
}
```

問題：步驟 (1) 和 (2) 不是原子的！

```
時間 →
Core 0: 讀取 locked=0 ─────────────────────── 寫入 locked=1
Core 1: ─────────────── 讀取 locked=0 ─────── 寫入 locked=1
                                              ↑
                                           兩個核心都認為自己獲得了鎖！
```

我們需要把「讀取-檢查-寫入」變成單一的原子操作。

### 2.2 RISC-V 原子指令

RISC-V 提供兩類原子操作：

**1. AMO (Atomic Memory Operations)**

單一指令完成讀-改-寫：

| 指令 | 操作 |
|------|------|
| `amoswap.w rd, rs2, (rs1)` | rd = *rs1; *rs1 = rs2 |
| `amoadd.w rd, rs2, (rs1)` | rd = *rs1; *rs1 = *rs1 + rs2 |
| `amoand.w rd, rs2, (rs1)` | rd = *rs1; *rs1 = *rs1 & rs2 |
| `amoor.w rd, rs2, (rs1)` | rd = *rs1; *rs1 = *rs1 \| rs2 |
| `amoxor.w rd, rs2, (rs1)` | rd = *rs1; *rs1 = *rs1 ^ rs2 |

**2. LR/SC (Load-Reserved / Store-Conditional)**

兩個指令配對使用，可以實作任意複雜的原子操作：

| 指令 | 說明 |
|------|------|
| `lr.w rd, (rs1)` | 載入並設定 Reservation |
| `sc.w rd, rs2, (rs1)` | 如果 Reservation 有效則寫入，rd=0 成功，rd=1 失敗 |

### 2.3 Acquire/Release 語意

RISC-V 原子指令可以加上後綴來控制記憶體順序：

- `.aq` (Acquire)：後續操作不會被重排到這之前
- `.rl` (Release)：前面操作不會被重排到這之後
- `.aqrl`：同時具有兩者

```asm
amoswap.w.aq   # Acquire 語意
amoswap.w.rl   # Release 語意

### 3.1 方法 A：使用 AMOSWAP

這是最簡潔的實作方式：

```asm
.section .text
.global spinlock_acquire
.global spinlock_release

# ========================================
# void spinlock_acquire(spinlock_t *lock)
# a0 = lock 的位址
# ========================================
spinlock_acquire:
    li      t0, 1                   # t0 = 1 (locked)
1:
    amoswap.w.aq t1, t0, (a0)       # t1 = old_value, *lock = 1 (原子)
    bnez    t1, 1b                  # if (old_value != 0) 重試
    ret

# ========================================
# void spinlock_release(spinlock_t *lock)
# a0 = lock 的位址
# ========================================
spinlock_release:
    amoswap.w.rl zero, zero, (a0)   # *lock = 0 (帶 Release 語意)
    ret
```

**執行流程**：

```
┌────────────────────────────────────────────────────────┐
│ amoswap.w.aq t1, t0, (a0)                              │
│                                                        │
│   原子操作：                                           │
│   1. t1 = *lock     ← 讀取舊值                         │
│   2. *lock = t0     ← 寫入新值 (1)                     │
│   3. (Acquire fence)                                   │
│                                                        │
│   如果 t1 == 0：我們成功獲得了鎖                       │
│   如果 t1 == 1：鎖已被佔用，重試                       │
└────────────────────────────────────────────────────────┘
```

### 3.2 方法 B：使用 LR/SC

這個方法更靈活，可以擴展為更複雜的操作：

```asm
.section .text
.global spinlock_acquire
.global spinlock_release

# ========================================
# void spinlock_acquire(spinlock_t *lock)
# ========================================
spinlock_acquire:
    li      t0, 1               # t0 = 1 (locked)
1:
    lr.w    t1, (a0)            # t1 = *lock (Load-Reserved)
    bnez    t1, 1b              # if (t1 != 0) 自旋等待
    sc.w    t2, t0, (a0)        # *lock = 1 (Store-Conditional)
    bnez    t2, 1b              # if (sc failed) 重試
    fence   rw, rw              # Memory Barrier
    ret

# ========================================
# void spinlock_release(spinlock_t *lock)
# ========================================
spinlock_release:
    fence   rw, rw              # Memory Barrier
    sw      zero, (a0)          # *lock = 0
    ret
```

**LR/SC 流程**：

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: lr.w t1, (a0)                                       │
│         讀取 lock，設定 Reservation                         │
│                                                             │
│ Step 2: bnez t1, 1b                                         │
│         如果 lock != 0，跳回去重試                          │
│                                                             │
│ Step 3: sc.w t2, t0, (a0)                                   │
│         嘗試寫入 1 到 lock                                  │
│         如果 Reservation 還有效 → t2=0 (成功)               │
│         如果被其他核心打斷 → t2=1 (失敗)                    │
│                                                             │
│ Step 4: bnez t2, 1b                                         │
│         如果 SC 失敗，跳回去重試                            │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 AMOSWAP vs LR/SC

| 比較 | AMOSWAP | LR/SC |
|------|---------|-------|
| 指令數 | 2 | 4 |
| 實作複雜度 | 簡單 | 較複雜 |
| 擴展性 | 有限（只能做預定義操作）| 可以實作任意原子操作 |
| 公平性 | 無保證 | 無保證 |
| 效能 | 稍好 | 稍差（可能多次重試）|

**結論**：對於簡單的 Spinlock，使用 AMOSWAP。對於需要複雜判斷的操作（如 CAS），使用 LR/SC。

---

## 四、Test-and-Test-and-Set 優化

### 4.1 問題：Cache Line Bouncing

當多個核心在 Spinlock 上自旋時，每個 `amoswap` 指令都會導致 Cache Line 在核心之間「彈跳」：

```
Core 0: amoswap → 獲取 Cache Line → 修改
Core 1: amoswap → 等待 Cache Line → 獲取 → 修改
Core 2: amoswap → 等待 Cache Line → 獲取 → 修改
Core 3: amoswap → 等待 Cache Line → 獲取 → 修改
```

每次 `amoswap` 都會使其他核心的 Cache 失效，造成大量的記憶體流量。

### 4.2 解決：先讀後交換

在嘗試 `amoswap` 之前，先用普通的 `lw` 讀取。如果鎖已被佔用，就持續讀取等待，不執行寫入：

```asm
spinlock_acquire:
    li      t0, 1
1:
    lw      t1, (a0)            # 普通讀取（不會造成 Cache 失效）
    bnez    t1, 1b              # if (locked) 自旋等待
    amoswap.w.aq t1, t0, (a0)   # 嘗試獲取
    bnez    t1, 1b              # if (失敗) 重試
    ret
```

**改進效果**：

```
Core 0 持有鎖
Core 1: lw → lw → lw → lw...  ← 只讀，不寫，不造成 Cache 失效
Core 2: lw → lw → lw → lw...
Core 3: lw → lw → lw → lw...

Core 0 釋放鎖
Core 1: lw=0 → amoswap → 獲取鎖
Core 2: lw=0 → amoswap → 失敗 → 回到 lw 自旋
Core 3: lw=0 → amoswap → 失敗 → 回到 lw 自旋
```

### 4.3 加入 WFI 省電

在等待時使用 `wfi` (Wait For Interrupt) 可以減少電力消耗：

```asm
spinlock_acquire:
    li      t0, 1
1:
    lw      t1, (a0)
    bnez    t1, 2f              # if (locked) 跳到等待
    amoswap.w.aq t1, t0, (a0)
    bnez    t1, 1b
    ret
2:
    wfi                         # 等待中斷（通常是 IPI）
    j       1b
```

**注意**：QEMU 可能忽略 `wfi`，在真實硬體上效果更明顯。

---

## 五、Ticket Lock：公平的 Spinlock

### 5.1 問題：Spinlock 不公平

基本的 Spinlock 沒有公平性保證。一個核心可能一直獲得鎖，而其他核心一直等待：

```
Core 0: 獲取 → 釋放 → 獲取 → 釋放 → 獲取 → ...
Core 1: 等待 → 等待 → 等待 → 等待 → ...
```

### 5.2 Ticket Lock 原理

Ticket Lock 像是銀行叫號系統：

1. 每個核心進入時領取一張「號碼牌」（next_ticket）
2. 等待「當前叫號」（now_serving）等於自己的號碼
3. 離開時，叫下一號

```c
typedef struct {
    volatile uint32_t now_serving;  // 目前服務的號碼
    volatile uint32_t next_ticket;  // 下一張號碼牌
} ticket_lock_t;
```

### 5.3 Assembly 實作

```asm
.section .text
.global ticket_lock_acquire
.global ticket_lock_release

# ========================================
# void ticket_lock_acquire(ticket_lock_t *lock)
# lock->now_serving at offset 0
# lock->next_ticket at offset 4
# ========================================
ticket_lock_acquire:
    # 1. 領取號碼牌：my_ticket = atomic_fetch_add(next_ticket, 1)
    li      t0, 1
    amoadd.w t1, t0, 4(a0)      # t1 = old next_ticket, next_ticket++

    # 2. 等待叫號
1:
    lw      t2, 0(a0)           # t2 = now_serving
    bne     t2, t1, 1b          # if (now_serving != my_ticket) 等待

    # 3. 輪到我了
    fence   rw, rw              # Memory Barrier
    ret

# ========================================
# void ticket_lock_release(ticket_lock_t *lock)
# ========================================
ticket_lock_release:
    fence   rw, rw              # Memory Barrier

    # now_serving++
    lw      t0, 0(a0)
    addi    t0, t0, 1
    sw      t0, 0(a0)
    ret
```

### 5.4 Ticket Lock 運作範例

```
初始狀態：now_serving=0, next_ticket=0

Core 0 進入：
  my_ticket = 0, next_ticket 變成 1
  now_serving == my_ticket → 獲得鎖

Core 1 進入：
  my_ticket = 1, next_ticket 變成 2
  now_serving (0) != my_ticket (1) → 等待

Core 2 進入：
  my_ticket = 2, next_ticket 變成 3
  now_serving (0) != my_ticket (2) → 等待

Core 0 離開：
  now_serving 變成 1

Core 1 看到 now_serving == 1 → 獲得鎖
Core 2 繼續等待

Core 1 離開：
  now_serving 變成 2

Core 2 看到 now_serving == 2 → 獲得鎖
```

---

## 六、C 語言整合

### 6.1 標頭檔定義

```c
// spinlock.h
#ifndef SPINLOCK_H
#define SPINLOCK_H

#include <stdint.h>

typedef struct {
    volatile int32_t locked;
} spinlock_t;

typedef struct {
    volatile uint32_t now_serving;
    volatile uint32_t next_ticket;
} ticket_lock_t;

#define SPINLOCK_INIT    {0}
#define TICKET_LOCK_INIT {0, 0}

// Assembly 函式宣告
extern void spinlock_acquire(spinlock_t *lock);
extern void spinlock_release(spinlock_t *lock);

extern void ticket_lock_acquire(ticket_lock_t *lock);
extern void ticket_lock_release(ticket_lock_t *lock);

// Inline 版本（使用 GCC built-in）
static inline void spin_lock(spinlock_t *lock) {
    while (__atomic_exchange_n(&lock->locked, 1, __ATOMIC_ACQUIRE) != 0) {
        while (__atomic_load_n(&lock->locked, __ATOMIC_RELAXED) != 0) {
            // Spin
        }
    }
}

static inline void spin_unlock(spinlock_t *lock) {
    __atomic_store_n(&lock->locked, 0, __ATOMIC_RELEASE);
}

#endif // SPINLOCK_H
```

### 6.2 使用範例

```c
#include "spinlock.h"
#include "cpu.h"

// 全域 Spinlock 保護 Ready Queue
spinlock_t ready_queue_lock = SPINLOCK_INIT;

void sched_add_ready(tcb_t *task) {
    spinlock_acquire(&ready_queue_lock);

    // 加入 Ready Queue
    list_add_tail(&task->node, &ready_queue);

    spinlock_release(&ready_queue_lock);
}

tcb_t *sched_pop_ready(void) {
    spinlock_acquire(&ready_queue_lock);

    tcb_t *task = NULL;
    if (!list_empty(&ready_queue)) {
        task = list_first_entry(&ready_queue, tcb_t, node);
        list_del(&task->node);
    }

    spinlock_release(&ready_queue_lock);
    return task;
}
```

### 6.3 IRQ-Safe Spinlock

在中斷處理中使用 Spinlock 需要特別小心。如果一個核心持有 Spinlock 時被中斷，而中斷處理程式也嘗試獲取同一個 Spinlock，就會造成死鎖：

```c
// 危險！可能死鎖
spinlock_acquire(&lock);
// ... 被中斷 ...
// 中斷處理程式：spinlock_acquire(&lock); ← 永遠等不到

spinlock_release(&lock);
```

解決方案：獲取 Spinlock 時關閉中斷

```c
typedef struct {
    spinlock_t lock;
    reg_t saved_status;
} irq_spinlock_t;

static inline void irq_spin_lock(irq_spinlock_t *lock) {
    // 1. 關閉中斷
    reg_t status;
    asm volatile("csrrc %0, mstatus, 0x8" : "=r"(status));
    lock->saved_status = status;

    // 2. 獲取 Spinlock
    spinlock_acquire(&lock->lock);
}

static inline void irq_spin_unlock(irq_spinlock_t *lock) {
    // 1. 釋放 Spinlock
    spinlock_release(&lock->lock);

    // 2. 恢復中斷狀態
    if (lock->saved_status & 0x8) {
        asm volatile("csrs mstatus, 0x8");
    }
}
```

---

## 七、本章回顧

在這一章，我們實作了 Spinlock：

1. **AMOSWAP**：最簡潔的 Spinlock 實作
2. **LR/SC**：更靈活，可擴展
3. **Test-and-Test-and-Set**：減少 Cache 競爭
4. **Ticket Lock**：公平的 Spinlock
5. **IRQ-Safe Spinlock**：適用於中斷處理

關鍵程式碼：

```asm
# 簡單 Spinlock
spinlock_acquire:
    li      t0, 1
1:
    amoswap.w.aq t1, t0, (a0)
    bnez    t1, 1b
    ret
```

下一章，我們將探討 **Cache Coherency**——多核心系統中 Cache 一致性的挑戰與解決方案。

---

## 參考資料

- **See RISC-V Run**, Ch 6 - Atomic Instructions
- **Data Structures in Practice**, Ch 14 - Spinlock Optimizations
- [RISC-V Atomics Extension](https://five-embeddev.com/riscv-isa-manual/latest/a.html)

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。
