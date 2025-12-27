# 核心對話：Inter-Processor Interrupt

**作者**: Danny Jiang
**日期**: 2025-12-14

---

## 前言：當一個核心需要叫醒另一個

考慮這個場景：

1. Core 0 正在執行低優先權 Task A
2. Core 1 正在執行低優先權 Task B
3. 某個中斷發生在 Core 0，喚醒了高優先權 Task C
4. Task C 應該立即執行！

問題：Task C 應該在哪個核心執行？

- 如果 Task C 可以搶占 Task A，那就在 Core 0 執行
- 但如果 Task C 被綁定到 Core 1（CPU Affinity），怎麼辦？

這時，Core 0 需要「通知」Core 1：「嘿，有高優先權工作給你！」

這就是 **IPI (Inter-Processor Interrupt)** 的用途。

---

## 一、IPI 的基本概念

### 1.1 什麼是 IPI？

**IPI (Inter-Processor Interrupt)**：一個核心向另一個核心發送的中斷。

本質上，IPI 就是一種特殊的軟體中斷（Software Interrupt），由軟體主動觸發，而非硬體事件。

### 1.2 IPI 的使用場景

| 場景 | 說明 |
|------|------|
| **排程通知** | Core 0 喚醒 Task，需要 Core 1 重新排程 |
| **TLB Shootdown** | 修改頁表後，通知所有核心清除 TLB |
| **系統關機** | 關機前通知所有核心停止執行 |
| **性能監控** | 採樣其他核心的狀態 |
| **Debug** | 暫停其他核心以進行除錯 |

### 1.3 為什麼不用共享變數？

你可能會想：「Core 0 設一個 flag，Core 1 去輪詢 (polling) 這個 flag，不就好了嗎？」

問題：

1. **延遲高**：Core 1 必須不斷檢查 flag，浪費 CPU 時間
2. **反應慢**：取決於輪詢的頻率
3. **能耗差**：持續輪詢消耗電力

IPI 的優勢：

1. **即時性**：中斷立即打斷目標核心
2. **低延遲**：幾個 cycle 內就能反應
3. **省電**：目標核心可以用 `wfi` 等待

---

## 二、RISC-V 的 IPI 機制：CLINT

### 2.1 CLINT 簡介

**CLINT (Core Local Interruptor)** 是 RISC-V 平台的本地中斷控制器，負責：

1. **Machine Timer Interrupt (MTI)**：產生定時器中斷
2. **Machine Software Interrupt (MSI)**：實現 IPI

### 2.2 CLINT 的 Memory Map

在 QEMU virt machine 上：

| 地址 | 暫存器 | 說明 |
|------|--------|------|
| `0x2000000` | MSIP[0] | Hart 0 的 Software Interrupt Pending |
| `0x2000004` | MSIP[1] | Hart 1 的 Software Interrupt Pending |
| `0x2000008` | MSIP[2] | Hart 2 的 Software Interrupt Pending |
| ... | ... | ... |
| `0x2004000` | MTIMECMP[0] | Hart 0 的 Timer Compare |
| `0x2004008` | MTIMECMP[1] | Hart 1 的 Timer Compare |
| ... | ... | ... |
| `0x200BFF8` | MTIME | 全域計時器（所有核心共享）|

### 2.3 MSIP 暫存器

每個核心有一個 32-bit 的 MSIP 暫存器，但只使用最低位：

```
 31                                              1   0
┌────────────────────────────────────────────────┬───┐
│                  Reserved (0)                  │SIP│
└────────────────────────────────────────────────┴───┘
```

- 寫入 1：觸發該核心的 Machine Software Interrupt
- 寫入 0：清除該核心的 Machine Software Interrupt

### 2.4 發送 IPI

```c
#define CLINT_BASE      0x2000000UL
#define CLINT_MSIP(hart) (*(volatile uint32_t *)(CLINT_BASE + (hart) * 4))

void ipi_send(uint64_t target_hart) {
    CLINT_MSIP(target_hart) = 1;
}

// 發送 IPI 給多個核心

---

## 三、Trap Handler 整合

### 3.1 mcause 的值

當 Machine Software Interrupt 發生時：

```
mcause = 0x8000000000000003  (RV64)
         ↑
         最高位 = 1 表示中斷（不是例外）

mcause & 0xFF = 3 → Machine Software Interrupt
```

### 3.2 更新 Trap Handler

```c
// trap.c
void trap_handler(reg_t *sp) {
    reg_t mcause = csr_read(mcause);

    if (mcause & MCAUSE_INTERRUPT) {
        reg_t cause = mcause & 0xFF;

        switch (cause) {
        case 7:  // Machine Timer Interrupt
            timer_handler();
            break;

        case 3:  // Machine Software Interrupt (IPI)
            ipi_handler();
            break;

        case 11: // Machine External Interrupt
            plic_handler();
            break;

        default:
            uart_printf("Unknown interrupt: %ld\n", cause);
            break;
        }
    } else {
        exception_handler(mcause);
    }
}
```

### 3.3 IPI Handler

```c
void ipi_handler(void) {
    // 1. 清除 IPI（必須！）
    ipi_clear();

    // 2. 處理 IPI 訊息
    cpu_t *cpu = get_cpu();

    // 檢查 IPI 的目的
    if (cpu->ipi_flags & IPI_RESCHEDULE) {
        cpu->ipi_flags &= ~IPI_RESCHEDULE;
        sched_request_switch();
    }

    if (cpu->ipi_flags & IPI_CALL_FUNC) {
        cpu->ipi_flags &= ~IPI_CALL_FUNC;
        // 執行遠端呼叫的函式
        if (cpu->ipi_func) {
            cpu->ipi_func(cpu->ipi_arg);
        }
    }

    if (cpu->ipi_flags & IPI_HALT) {
        cpu->ipi_flags &= ~IPI_HALT;
        // 停止這個核心
        while (1) {
            asm volatile("wfi");
        }
    }
}
```

### 3.4 啟用 Software Interrupt

在初始化時，必須啟用 Machine Software Interrupt：

```c
void ipi_init(void) {
    // 啟用 Machine Software Interrupt (MSIE)
    csr_set(mie, 1 << 3);  // Bit 3 = MSIE
}
```

確認 `mie` 暫存器的結構：

```
mie 暫存器：
 11   7   3
┌────────────────────────────────────────────────┐
│...│MEIE│...│MTIE│...│MSIE│...                  │
└────────────────────────────────────────────────┘
      ↑         ↑         ↑
      外部中斷   定時器    軟體中斷 (IPI)
```

---

## 四、IPI 類型設計

### 4.1 IPI 訊息結構

我們可以定義多種 IPI 類型：

```c
// ipi.h
#ifndef IPI_H
#define IPI_H

#include <stdint.h>

// IPI 類型
#define IPI_RESCHEDULE   (1 << 0)  // 觸發重新排程
#define IPI_CALL_FUNC    (1 << 1)  // 執行遠端函式
#define IPI_HALT         (1 << 2)  // 停止核心
#define IPI_TLB_FLUSH    (1 << 3)  // 清除 TLB

// IPI 函式類型
typedef void (*ipi_func_t)(void *arg);

// 在 cpu_t 中加入 IPI 相關欄位
// uint32_t ipi_flags;
// ipi_func_t ipi_func;
// void *ipi_arg;

// API
void ipi_init(void);
void ipi_send(uint64_t target_hart);
void ipi_send_mask(uint64_t hart_mask);
void ipi_send_all_except_self(void);
void ipi_clear(void);
void ipi_handler(void);

// 高階 API
void smp_request_reschedule(uint64_t target_hart);
void smp_call_function(uint64_t target_hart, ipi_func_t func, void *arg);
void smp_halt_all(void);

#endif // IPI_H
```

### 4.2 排程通知

最常見的 IPI 用途是通知其他核心重新排程：

```c
void smp_request_reschedule(uint64_t target_hart) {
    cpu_t *target_cpu = &g_cpus[target_hart];

    // 設定 flag（原子操作）
    __atomic_or_fetch(&target_cpu->ipi_flags, IPI_RESCHEDULE, __ATOMIC_RELEASE);

    // 發送 IPI
    ipi_send(target_hart);
}
```

使用場景：

```c
void task_wake(tcb_t *task) {
    spinlock_acquire(&ready_queue_lock);

    task->state = TASK_STATE_READY;
    list_add_tail(&task->node, &ready_queue);

    // 如果這個 Task 優先權比目標核心的當前 Task 高
    for (int i = 0; i < MAX_CORES; i++) {
        cpu_t *cpu = &g_cpus[i];
        if (cpu->current_task->priority < task->priority) {
            smp_request_reschedule(i);
            break;  // 只需要通知一個核心
        }
    }

    spinlock_release(&ready_queue_lock);
}
```

### 4.3 遠端函式呼叫

有時我們需要在特定核心上執行函式：

```c
void smp_call_function(uint64_t target_hart, ipi_func_t func, void *arg) {
    cpu_t *target_cpu = &g_cpus[target_hart];

    // 設定函式和參數
    target_cpu->ipi_func = func;
    target_cpu->ipi_arg = arg;

    // 設定 flag
    __atomic_or_fetch(&target_cpu->ipi_flags, IPI_CALL_FUNC, __ATOMIC_RELEASE);

    // 發送 IPI
    ipi_send(target_hart);
}

// 使用範例：在所有核心上清除 TLB
void flush_tlb_entry(void *addr) {
    asm volatile("sfence.vma %0, zero" : : "r"(addr));
}

void smp_flush_tlb(void *addr) {
    // 清除自己的 TLB
    flush_tlb_entry(addr);

    // 通知其他核心
    for (int i = 0; i < MAX_CORES; i++) {
        if (i != get_hartid()) {
            smp_call_function(i, flush_tlb_entry, addr);
        }
    }
}
```

---

## 五、完整實作

### 5.1 ipi.c

```c
// ipi.c
#include "ipi.h"
#include "cpu.h"
#include "riscv.h"

#define CLINT_BASE      0x2000000UL
#define CLINT_MSIP(hart) (*(volatile uint32_t *)(CLINT_BASE + (hart) * 4))

void ipi_init(void) {
    // 啟用 Machine Software Interrupt
    csr_set(mie, 1 << 3);

    // 清除可能殘留的 IPI
    ipi_clear();

    // 初始化 IPI flags
    get_cpu()->ipi_flags = 0;
}

void ipi_send(uint64_t target_hart) {
    CLINT_MSIP(target_hart) = 1;
}

void ipi_send_mask(uint64_t hart_mask) {
    for (int i = 0; i < MAX_CORES; i++) {
        if (hart_mask & (1UL << i)) {
            CLINT_MSIP(i) = 1;
        }
    }
}

void ipi_send_all_except_self(void) {
    uint64_t self = get_hartid();
    for (int i = 0; i < MAX_CORES; i++) {
        if (i != self) {
            CLINT_MSIP(i) = 1;
        }
    }
}

void ipi_clear(void) {
    uint64_t hartid = get_hartid();
    CLINT_MSIP(hartid) = 0;
}

void ipi_handler(void) {
    // 必須先清除 IPI
    ipi_clear();

    cpu_t *cpu = get_cpu();
    uint32_t flags = __atomic_exchange_n(&cpu->ipi_flags, 0, __ATOMIC_ACQUIRE);

    if (flags & IPI_RESCHEDULE) {
        sched_request_switch();
    }

    if (flags & IPI_CALL_FUNC) {
        if (cpu->ipi_func) {
            cpu->ipi_func(cpu->ipi_arg);
            cpu->ipi_func = NULL;
            cpu->ipi_arg = NULL;
        }
    }

    if (flags & IPI_HALT) {
        uart_printf("[Hart %d] Halting\n", get_hartid());
        while (1) {
            asm volatile("wfi");
        }
    }
}
```

---

## 六、常見問題

### 6.1 IPI 遺失

**問題**：發送 IPI 後，目標核心沒有反應

**可能原因**：

1. 目標核心的中斷被禁用（`mstatus.MIE = 0`）
2. `mie.MSIE` 沒有設定
3. Trap Handler 沒有處理 cause=3

**除錯方法**：

```c
// 檢查 mie 暫存器
uart_printf("mie = 0x%lx\n", csr_read(mie));

// 檢查 mip 暫存器（是否有 pending 的 IPI）
uart_printf("mip = 0x%lx\n", csr_read(mip));
```

### 6.2 IPI 重複觸發

**問題**：IPI handler 一直被呼叫

**原因**：忘記清除 MSIP

**解決**：確保在 handler 開頭就清除：

```c
void ipi_handler(void) {
    ipi_clear();  // 第一行就清除！
    // ...
}
```

### 6.3 Race Condition

**問題**：設定 `ipi_flags` 和發送 IPI 之間的競爭

**場景**：

```c
Core 0                          Core 1
------                          ------
ipi_flags = IPI_RESCHEDULE
                                (被 Timer 中斷)
                                ipi_handler() → flags = 0
ipi_send(1)
                                (IPI 觸發)
                                ipi_handler() → flags = 0 (遺失了!)
```

**解決**：使用原子操作設定 flags

```c
__atomic_or_fetch(&target_cpu->ipi_flags, IPI_RESCHEDULE, __ATOMIC_RELEASE);
```

---

## 七、本章回顧

在這一章，我們實作了 IPI：

1. **CLINT MSIP**：RISC-V 的 IPI 機制
2. **Trap Handler**：處理 Machine Software Interrupt (cause=3)
3. **IPI 類型**：RESCHEDULE, CALL_FUNC, HALT
4. **高階 API**：`smp_request_reschedule()`, `smp_call_function()`

關鍵程式碼：

```c
// 發送 IPI
#define CLINT_MSIP(hart) (*(volatile uint32_t *)(0x2000000 + (hart) * 4))
CLINT_MSIP(target_hart) = 1;

// 接收 IPI
void ipi_handler(void) {
    CLINT_MSIP(get_hartid()) = 0;  // 清除
    // 處理...
}
```

下一章，我們將實作 **SMP Scheduler**——真正的多核心排程器。

---

## 參考資料

- **See RISC-V Run**, Ch 10 - SBI and IPI
- [SiFive CLINT Specification](https://www.sifive.com/documentation)
- [QEMU RISC-V virt Machine](https://www.qemu.org/docs/master/system/riscv/virt.html)

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。
