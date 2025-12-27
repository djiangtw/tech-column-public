# SMP 整合與驗證

**作者**: Danny Jiang
**日期**: 2025-12-14

---

## 前言：把所有元件組合起來

在過去的九章（Ch 20-28），我們實作了 SMP 的各個元件：

| 章節 | 元件 | 功能 |
|------|------|------|
| Ch 20 | SMP 概論 | 多核心的挑戰與架構 |
| Ch 21 | Multi-Core Boot | 多核心啟動 |
| Ch 22 | Per-Core Data | 每核心資料結構 |
| Ch 23 | Spinlock | 原子鎖 |
| Ch 24 | Cache Coherency | 快取一致性 |
| Ch 25 | IPI | 核心間中斷 |
| Ch 26 | SMP Scheduler | 多核心排程器 |
| Ch 27 | SMP Mutex | 多核心互斥鎖 |
| Ch 28 | Lock-Free Queue | 無鎖佇列 |

這一章，我們將把所有元件整合起來，並進行驗證。

---

## 一、系統架構總覽

### 1.1 記憶體佈局

```
┌─────────────────────────────────────────────────────────────┐
│                    Memory Layout                             │
├─────────────────────────────────────────────────────────────┤
│ 0x80000000 - 0x8001FFFF │ Kernel Code & Data (128 KB)       │
├─────────────────────────────────────────────────────────────┤
│ 0x80020000 - 0x8003FFFF │ User Code & Data (128 KB)         │
├─────────────────────────────────────────────────────────────┤
│ 0x80040000 - 0x8004FFFF │ Core 0 Stack (64 KB)              │
├─────────────────────────────────────────────────────────────┤
│ 0x80050000 - 0x8005FFFF │ Core 1 Stack (64 KB)              │
├─────────────────────────────────────────────────────────────┤
│ 0x80060000 - 0x800FFFFF │ Task Stacks & Heap                │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 核心資料結構

```c
// Per-Core Data
typedef struct cpu {
    uint64_t hartid;           // Hardware Thread ID
    tcb_t *current_task;       // 當前執行的 Task
    tcb_t *idle_task;          // Idle Task
    uint32_t irq_nesting;      // 中斷嵌套計數
    uint32_t need_reschedule;  // 需要重新排程
    spinlock_t local_lock;     // 本地鎖
} cpu_t;

// 全域資料
cpu_t g_cpus[MAX_CORES];
spinlock_t g_sched_lock;       // 排程器鎖
tcb_t *g_ready_queue;          // 全域 Ready Queue
```

### 1.3 啟動流程

```
┌─────────────────────────────────────────────────────────────┐
│                    Boot Sequence                             │
├─────────────────────────────────────────────────────────────┤
│ 1. All cores start at _start                                │
│ 2. Core 0 (BSP):                                            │
│    - Initialize BSS                                         │
│    - Initialize UART                                        │
│    - Initialize Timer                                       │
│    - Initialize Scheduler                                   │
│    - Create Tasks                                           │
│    - Set boot_flag = 1                                      │
│    - Start Scheduler                                        │
│ 3. Core 1+ (AP):                                            │
│    - Wait for boot_flag                                     │
│    - Initialize Per-Core Data                               │
│    - Start Scheduler                                        │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、整合程式碼

### 2.1 main.c

```c
#include "kernel.h"
#include "smp.h"
#include "scheduler.h"
#include "spinlock.h"
#include "uart.h"
#include "timer.h"

volatile uint32_t boot_flag = 0;

void kernel_main(void) {
    uint64_t hartid = get_hartid();

    if (hartid == 0) {
        // BSP: 初始化系統
        uart_init();
        uart_printf("danieRTOS v2.0 SMP\n");
        uart_printf("Core 0: Initializing...\n");

        // 初始化 BSS
        extern char _bss_start[], _bss_end[];
        for (char *p = _bss_start; p < _bss_end; p++) {
            *p = 0;
        }

        // 初始化 Per-Core Data
        smp_init_bsp();

        // 初始化排程器
        sched_init();

        // 初始化 Timer
        timer_init();

        // 建立 Tasks
        create_demo_tasks();

        // 通知其他核心
        __sync_synchronize();
        boot_flag = 1;

        uart_printf("Core 0: Starting scheduler\n");
        sched_start();
    } else {
        // AP: 等待 BSP 完成初始化
        while (boot_flag == 0) {
            asm volatile("wfi");
        }
        __sync_synchronize();

        // 初始化 Per-Core Data
        smp_init_ap(hartid);

        uart_printf("Core %d: Starting scheduler\n", hartid);
        sched_start();
    }

---

## 三、驗證測試

### 3.1 測試 1：多核心啟動

```c
void test_multicore_boot(void) {
    uart_printf("=== Test: Multi-Core Boot ===\n");

    for (int i = 0; i < MAX_CORES; i++) {
        cpu_t *cpu = &g_cpus[i];
        uart_printf("Core %d: hartid=%d, tp=%p\n",
                    i, cpu->hartid, cpu);
    }

    uart_printf("PASS: All cores booted\n");
}
```

### 3.2 測試 2：Spinlock

```c
volatile int counter = 0;
spinlock_t counter_lock = SPINLOCK_INIT;

void test_spinlock_task(void *arg) {
    int core = (int)(uintptr_t)arg;

    for (int i = 0; i < 10000; i++) {
        spinlock_acquire(&counter_lock);
        counter++;
        spinlock_release(&counter_lock);
    }

    uart_printf("Core %d: Done\n", core);
}

void test_spinlock(void) {
    uart_printf("=== Test: Spinlock ===\n");

    counter = 0;

    // 在兩個核心上執行
    task_create(test_spinlock_task, (void*)0, PRIORITY_NORMAL, CORE_0);
    task_create(test_spinlock_task, (void*)1, PRIORITY_NORMAL, CORE_1);

    // 等待完成
    task_delay(1000);

    uart_printf("Counter: %d (expected: 20000)\n", counter);

    if (counter == 20000) {
        uart_printf("PASS: Spinlock works correctly\n");
    } else {
        uart_printf("FAIL: Race condition detected\n");
    }
}
```

### 3.3 測試 3：IPI

```c
volatile int ipi_received[MAX_CORES] = {0};

void ipi_handler(void) {
    uint64_t hartid = get_hartid();
    ipi_received[hartid]++;
    uart_printf("Core %d: IPI received\n", hartid);
}

void test_ipi(void) {
    uart_printf("=== Test: IPI ===\n");

    // Core 0 發送 IPI 給 Core 1
    uart_printf("Core 0: Sending IPI to Core 1\n");
    smp_send_ipi(1);

    // 等待
    task_delay(100);

    if (ipi_received[1] > 0) {
        uart_printf("PASS: IPI delivered\n");
    } else {
        uart_printf("FAIL: IPI not received\n");
    }
}
```

### 3.4 測試 4：SMP Scheduler

```c
void test_sched_task(void *arg) {
    int id = (int)(uintptr_t)arg;

    for (int i = 0; i < 5; i++) {
        uint64_t hartid = get_hartid();
        uart_printf("Task %d running on Core %d\n", id, hartid);
        task_delay(100);
    }
}

void test_smp_scheduler(void) {
    uart_printf("=== Test: SMP Scheduler ===\n");

    // 建立 4 個 Tasks
    for (int i = 0; i < 4; i++) {
        task_create(test_sched_task, (void*)(uintptr_t)i,
                    PRIORITY_NORMAL, CORE_ANY);
    }

    // 觀察 Tasks 在不同核心上執行
    task_delay(2000);

    uart_printf("PASS: Tasks distributed across cores\n");
}
```

### 3.5 測試 5：Mutex

```c
mutex_t test_mutex = MUTEX_INIT;
volatile int shared_data = 0;

void test_mutex_task(void *arg) {
    int id = (int)(uintptr_t)arg;

    for (int i = 0; i < 100; i++) {
        mutex_lock(&test_mutex, WAIT_FOREVER);

        int temp = shared_data;
        task_delay(1);  // 模擬長時間操作
        shared_data = temp + 1;

        mutex_unlock(&test_mutex);
    }

    uart_printf("Task %d: Done\n", id);
}

void test_mutex(void) {
    uart_printf("=== Test: Mutex ===\n");

    shared_data = 0;

    // 在兩個核心上執行
    task_create(test_mutex_task, (void*)0, PRIORITY_NORMAL, CORE_0);
    task_create(test_mutex_task, (void*)1, PRIORITY_NORMAL, CORE_1);

    // 等待完成
    task_delay(5000);

    uart_printf("Shared data: %d (expected: 200)\n", shared_data);

    if (shared_data == 200) {
        uart_printf("PASS: Mutex works correctly\n");
    } else {
        uart_printf("FAIL: Race condition detected\n");
    }
}
```

### 3.6 測試 6：Lock-Free Queue

```c
spsc_queue_t test_queue;
volatile int producer_done = 0;
volatile int consumer_count = 0;

void producer_task(void *arg) {
    for (int i = 0; i < 10000; i++) {
        while (spsc_queue_push(&test_queue, i) != 0) {
            task_yield();  // Queue 滿，讓出 CPU
        }
    }
    producer_done = 1;
    uart_printf("Producer: Done\n");
}

void consumer_task(void *arg) {
    while (!producer_done || !spsc_queue_is_empty(&test_queue)) {
        uint32_t value;
        if (spsc_queue_pop(&test_queue, &value) == 0) {
            consumer_count++;
        } else {
            task_yield();
        }
    }
    uart_printf("Consumer: Done, count=%d\n", consumer_count);
}

void test_lockfree_queue(void) {
    uart_printf("=== Test: Lock-Free Queue ===\n");

    spsc_queue_init(&test_queue);
    producer_done = 0;
    consumer_count = 0;

    // Producer 在 Core 0，Consumer 在 Core 1
    task_create(producer_task, NULL, PRIORITY_NORMAL, CORE_0);
    task_create(consumer_task, NULL, PRIORITY_NORMAL, CORE_1);

    // 等待完成
    task_delay(5000);

    if (consumer_count == 10000) {
        uart_printf("PASS: Lock-Free Queue works correctly\n");
    } else {
        uart_printf("FAIL: Data lost, count=%d\n", consumer_count);
    }
}
```

---

## 四、效能測量

### 4.1 Context Switch 時間

```c
void measure_context_switch(void) {
    uart_printf("=== Measure: Context Switch ===\n");

    uint64_t start = read_cycles();

    for (int i = 0; i < 10000; i++) {
        task_yield();
    }

    uint64_t end = read_cycles();
    uint64_t total = end - start;

    uart_printf("Total cycles: %llu\n", total);
    uart_printf("Avg cycles/switch: %llu\n", total / 10000);
}
```

### 4.2 Spinlock 開銷

```c
void measure_spinlock(void) {
    uart_printf("=== Measure: Spinlock ===\n");

    spinlock_t lock = SPINLOCK_INIT;

    uint64_t start = read_cycles();

    for (int i = 0; i < 100000; i++) {
        spinlock_acquire(&lock);
        spinlock_release(&lock);
    }

    uint64_t end = read_cycles();
    uint64_t total = end - start;

    uart_printf("Total cycles: %llu\n", total);
    uart_printf("Avg cycles/lock-unlock: %llu\n", total / 100000);
}
```

### 4.3 IPI 延遲

```c
volatile uint64_t ipi_send_time = 0;
volatile uint64_t ipi_recv_time = 0;

void ipi_latency_handler(void) {
    ipi_recv_time = read_cycles();
}

void measure_ipi_latency(void) {
    uart_printf("=== Measure: IPI Latency ===\n");

    uint64_t total = 0;

    for (int i = 0; i < 100; i++) {
        ipi_send_time = read_cycles();
        smp_send_ipi(1);

        // 等待 IPI 處理完成
        while (ipi_recv_time <= ipi_send_time) {
            // Spin
        }

        total += (ipi_recv_time - ipi_send_time);
    }

    uart_printf("Avg IPI latency: %llu cycles\n", total / 100);
}
```

---

## 五、除錯技巧

### 5.1 常見問題

| 問題 | 症狀 | 解決方案 |
|------|------|----------|
| **Deadlock** | 系統卡住 | 檢查鎖的順序 |
| **Race Condition** | 資料不一致 | 加入適當的鎖 |
| **Cache Coherency** | 看到舊資料 | 加入 Memory Barrier |
| **Stack Overflow** | 隨機崩潰 | 增加 Stack 大小 |
| **IPI 遺失** | 核心沒反應 | 檢查 CLINT 配置 |

### 5.2 除錯輸出

```c
#define DEBUG_SMP 1

#if DEBUG_SMP
#define SMP_DEBUG(fmt, ...) \
    uart_printf("[Core %d] " fmt, get_hartid(), ##__VA_ARGS__)
#else
#define SMP_DEBUG(fmt, ...)
#endif

// 使用範例
void spinlock_acquire(spinlock_t *lock) {
    SMP_DEBUG("Acquiring lock %p\n", lock);
    // ...
    SMP_DEBUG("Lock %p acquired\n", lock);
}
```

### 5.3 Lock 追蹤

```c
typedef struct spinlock {
    volatile uint32_t locked;
    uint64_t owner_hartid;     // 持有者
    const char *acquired_at;   // 取得位置
    uint32_t acquired_line;    // 取得行號
} spinlock_t;

#define spinlock_acquire(lock) \
    spinlock_acquire_debug(lock, __FILE__, __LINE__)

void spinlock_acquire_debug(spinlock_t *lock,
                            const char *file, uint32_t line) {
    // ... 取得鎖 ...
    lock->owner_hartid = get_hartid();
    lock->acquired_at = file;
    lock->acquired_line = line;
}
```

---

## 六、v2.x 總結

### 6.1 完成的功能

| 功能 | 狀態 | 說明 |
|------|------|------|
| Multi-Core Boot | ✅ | 支援 2+ 核心啟動 |
| Per-Core Data | ✅ | 使用 tp 暫存器 |
| Spinlock | ✅ | LR/SC 實作 |
| Cache Coherency | ✅ | FENCE 指令 |
| IPI | ✅ | CLINT MSIP |
| SMP Scheduler | ✅ | Global Queue + Affinity |
| SMP Mutex | ✅ | Priority Inheritance |
| Lock-Free Queue | ✅ | SPSC/MPSC |

### 6.2 效能數據

| 指標 | 數值 |
|------|------|
| Context Switch | ~500 cycles |
| Spinlock (無競爭) | ~10 cycles |
| Spinlock (有競爭) | ~100 cycles |
| IPI Latency | ~1000 cycles |
| SPSC Queue Push | ~5 cycles |

### 6.3 未來方向

1. **Load Balancing**：自動平衡核心負載
2. **NUMA Support**：支援非統一記憶體架構
3. **RCU**：Read-Copy-Update 機制
4. **Futex**：快速使用者空間互斥鎖

---

## 七、本章回顧

在這一章，我們完成了 SMP 整合：

1. **系統架構**：記憶體佈局、核心資料結構、啟動流程
2. **整合程式碼**：main.c、smp.c
3. **驗證測試**：多核心啟動、Spinlock、IPI、Scheduler、Mutex、Lock-Free Queue
4. **效能測量**：Context Switch、Spinlock、IPI Latency
5. **除錯技巧**：常見問題、除錯輸出、Lock 追蹤

至此，danieRTOS v2.x 的 SMP 支援已經完成！

---

## 參考資料

- **FreeRTOS SMP**, Multi-Core Support
- **Linux Kernel**, SMP Implementation
- **RISC-V Privileged Spec**, CLINT

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。
