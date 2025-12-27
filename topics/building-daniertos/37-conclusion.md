# 大功告成：danieRTOS v3.x 的完整驗證

**作者**: Danny Jiang
**日期**: 2025-12-26

---

## 前言：旅程的終點

從 v0.x 的第一個 Context Switch，到 v3.x 的 SMP + User Mode 整合，我們走過了漫長的旅程。

現在，是時候驗證我們的成果了。

這一章，我們將：
1. 進行整合測試，確保所有功能正常運作
2. 展示 Demo，展現 v3.x 的完整能力
3. 比較各版本的效能
4. 總結這個系列，展望未來

---

## 一、整合測試

### 1.1 測試矩陣

| 測試項目 | v0.x | v1.x | v2.x | v3.x |
|----------|------|------|------|------|
| 基本 Context Switch | ✅ | ✅ | ✅ | ✅ |
| Timer 中斷 | ✅ | ✅ | ✅ | ✅ |
| Semaphore | ✅ | ✅ | ✅ | ✅ |
| Mutex | ✅ | ✅ | ✅ | ✅ |
| Queue | ✅ | ✅ | ✅ | ✅ |
| User Mode | ❌ | ✅ | ❌ | ✅ |
| PMP 保護 | ❌ | ✅ | ❌ | ✅ |
| Syscall | ❌ | ✅ | ❌ | ✅ |
| SMP (多核) | ❌ | ❌ | ✅ | ✅ |
| Spinlock | ❌ | ❌ | ✅ | ✅ |
| IPI | ❌ | ❌ | ✅ | ✅ |
| Task 遷移 | ❌ | ❌ | ✅ | ✅ |
| 混合 M/U 任務 | ❌ | ❌ | ❌ | ✅ |
| Per-Task Kernel Stack | ❌ | ❌ | ❌ | ✅ |
| 資源清理 | ❌ | ❌ | ❌ | ✅ |

### 1.2 自動化測試

```c
/* test_v3.c */

void test_basic_context_switch(void)
{
    volatile int counter = 0;
    
    task_create_user("TestA", test_task_a, &counter, 2, CORE_ANY);
    task_create_user("TestB", test_task_b, &counter, 2, CORE_ANY);
    
    /* 等待測試完成 */
    task_delay(1000);
    
    ASSERT(counter == 20);  /* 每個 task 增加 10 次 */
    uart_puts("[PASS] Basic context switch\n");
}

void test_smp_ipi(void)
{
    volatile int flag = 0;
    
    /* 在 Core 1 創建任務 */
    task_create("Core1Task", core1_task, &flag, 3, 
                stack, sizeof(stack), (1U << 1));
    
    /* 從 Core 0 發送 IPI */
    smp_send_ipi(1);
    
    /* 等待 Core 1 處理 */
    task_delay(100);
    
    ASSERT(flag == 1);
    uart_puts("[PASS] SMP IPI\n");
}

void test_pmp_protection(void)
{
    /* 創建 User Task，嘗試存取 Kernel 記憶體 */
    task_create_user("BadTask", bad_task, NULL, 2, CORE_ANY);
    
    /* 等待 fault 發生 */
    task_delay(100);
    
    /* 檢查 BadTask 是否被終止 */
    ASSERT(bad_task_terminated);
    uart_puts("[PASS] PMP protection\n");
}

void test_resource_cleanup(void)
{
    mutex_t mutex;
    mutex_init(&mutex);
    
    /* 創建持有 mutex 的任務，然後讓它 fault */
    task_create_user("FaultTask", fault_task, &mutex, 2, CORE_ANY);
    
    /* 等待 fault 發生 */
    task_delay(100);
    
    /* 檢查 mutex 是否被釋放 */
    ASSERT(mutex.owner == NULL);
    uart_puts("[PASS] Resource cleanup\n");
}

void run_all_tests(void)
{
    uart_puts("\n=== danieRTOS v3.x Test Suite ===\n\n");
    
    test_basic_context_switch();
    test_smp_ipi();
    test_pmp_protection();
    test_resource_cleanup();
    /* ... 更多測試 ... */
    
    uart_puts("\n=== All tests passed! ===\n");
}
```

---

## 二、Demo：多核心安全隔離

### 2.1 場景描述

```
┌─────────────────────────────────────────────────────────────┐
│                    Demo: Secure IoT Gateway                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Core 0                           Core 1                     │
│  ┌─────────────────────┐          ┌─────────────────────┐   │
│  │ Sensor Task (U-mode)│          │ Network Task (U-mode)│   │
│  │ - 讀取感測器資料     │          │ - 發送資料到雲端     │   │
│  │ - 寫入 Queue        │          │ - 從 Queue 讀取      │   │
│  └─────────┬───────────┘          └─────────┬───────────┘   │
│            │                                │                │
│            │◄──────── Queue ───────────────►│                │
│            │      (Kernel 管理)              │                │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    Kernel (M-mode)                       ││
│  │  - PMP 保護：Task 不能直接存取對方記憶體                  ││
│  │  - Syscall：唯一的通訊方式                               ││
│  │  - 資源清理：Task fault 不影響其他 Task                  ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Demo 程式碼

```c
/* demo_secure_gateway.c */

static queue_t sensor_queue;
static uint8_t queue_buffer[10 * sizeof(sensor_data_t)];

/* Sensor Task (Core 0, U-mode) */
static void sensor_task(void *arg)
{
    sys_puts("[Sensor] Started on Core 0\n");
    
    for (int i = 0; i < 100; i++) {
        sensor_data_t data;
        data.timestamp = sys_get_tick();
        data.temperature = read_temperature_sensor();
        data.humidity = read_humidity_sensor();
        
        /* 透過 syscall 發送到 Queue */
        int ret = sys_queue_send(&sensor_queue, &data, 100);
        if (ret < 0) {
            sys_puts("[Sensor] Queue full, dropping data\n");
        }
        
        sys_delay(50);  /* 50ms 間隔 */
    }
    
    sys_puts("[Sensor] Done!\n");
    sys_exit();
}

/* Network Task (Core 1, U-mode) */
static void network_task(void *arg)
{
    sys_puts("[Network] Started on Core 1\n");
    
    while (1) {
        sensor_data_t data;
        
        /* 透過 syscall 從 Queue 接收 */
        int ret = sys_queue_recv(&sensor_queue, &data, 1000);
        if (ret < 0) {
            sys_puts("[Network] Timeout, no data\n");
            continue;
        }
        
        /* 模擬發送到雲端 */
        sys_printf("[Network] Sending: temp=%d, humidity=%d\n",
                   data.temperature, data.humidity);
        
        /* 模擬網路延遲 */
        sys_delay(20);
    }
}

/* Monitor Task (Core 1, M-mode) */
static void monitor_task(void *arg)
{
    uart_puts("[Monitor] Kernel monitor started\n");

    while (1) {
        /* 直接存取 Kernel 資料結構 */
        uart_printf("[Monitor] Queue count: %d\n", sensor_queue.count);
        uart_printf("[Monitor] Core 0 task: %s\n",
                    g_cpus[0].current_task->name);
        uart_printf("[Monitor] Core 1 task: %s\n",
                    g_cpus[1].current_task->name);

        task_delay(500);
    }
}

void demo_init(void)
{
    uart_puts("\n=== danieRTOS v3.x Secure Gateway Demo ===\n\n");

    /* 初始化 Queue */
    queue_init(&sensor_queue, queue_buffer, sizeof(sensor_data_t), 10);

    /* 創建 User Tasks */
    task_create_user("Sensor", sensor_task, NULL, 2, (1U << 0));
    task_create_user("Network", network_task, NULL, 2, (1U << 1));

    /* 創建 Kernel Monitor */
    task_create("Monitor", monitor_task, NULL, 1,
                monitor_stack, sizeof(monitor_stack), (1U << 1));

    uart_puts("All tasks created. Starting scheduler...\n\n");
}
```

### 2.3 運行結果

```
=== danieRTOS v3.x Secure Gateway Demo ===

All tasks created. Starting scheduler...

[Sensor] Started on Core 0
[Network] Started on Core 1
[Monitor] Kernel monitor started
[Network] Sending: temp=25, humidity=60
[Network] Sending: temp=26, humidity=61
[Monitor] Queue count: 3
[Monitor] Core 0 task: Sensor
[Monitor] Core 1 task: Network
[Network] Sending: temp=25, humidity=59
...
```

---

## 三、Demo：Fault Isolation

### 3.1 場景描述

展示當一個 User Task fault 時，不會影響其他 Task。

```c
/* demo_fault_isolation.c */

/* 故意 fault 的任務 */
static void bad_task(void *arg)
{
    sys_puts("[BadTask] I'm going to crash...\n");

    /* 嘗試存取 Kernel 記憶體 */
    volatile int *kernel_ptr = (int *)0x80000000;
    *kernel_ptr = 42;  /* Access Fault! */

    /* 不會執行到這裡 */
    sys_puts("[BadTask] This should not print\n");
}

/* 正常的任務 */
static void good_task(void *arg)
{
    sys_puts("[GoodTask] Started\n");

    for (int i = 0; i < 10; i++) {
        sys_printf("[GoodTask] Still running... %d\n", i);
        sys_delay(100);
    }

    sys_puts("[GoodTask] Done!\n");
    sys_exit();
}

void demo_fault_isolation(void)
{
    uart_puts("\n=== Fault Isolation Demo ===\n\n");

    task_create_user("BadTask", bad_task, NULL, 2, CORE_ANY);
    task_create_user("GoodTask", good_task, NULL, 2, CORE_ANY);
}
```

### 3.2 運行結果

```
=== Fault Isolation Demo ===

[BadTask] I'm going to crash...
[Fault] Task 'BadTask' faulted: mcause=7, mtval=0x80000000
[Fault] Cleaning up resources and terminating task...
[GoodTask] Started
[GoodTask] Still running... 0
[GoodTask] Still running... 1
[GoodTask] Still running... 2
...
[GoodTask] Done!
```

**觀察**：BadTask 被終止，但 GoodTask 繼續正常執行。

---

## 四、效能比較

### 4.1 Context Switch 時間

| 版本 | Context Switch 時間 | 說明 |
|------|---------------------|------|
| v0.x | ~500 cycles | 基本 M-mode 切換 |
| v1.x | ~800 cycles | 加入 PMP 切換 |
| v2.x | ~600 cycles | SMP，無 PMP |
| v3.x | ~1000 cycles | SMP + PMP + 雙 Stack |

### 4.2 Syscall 時間

| 版本 | Syscall 時間 | 說明 |
|------|--------------|------|
| v0.x | N/A | 無 Syscall |
| v1.x | ~300 cycles | 單核 Syscall |
| v2.x | N/A | 無 Syscall |
| v3.x | ~400 cycles | SMP Syscall + Spinlock |

### 4.3 記憶體使用

| 版本 | 每 Task 記憶體 | 說明 |
|------|----------------|------|
| v0.x | 4 KB | 單一 Stack |
| v1.x | 8 KB | User Stack + 共用 Kernel Stack |
| v2.x | 4 KB | 單一 Stack |
| v3.x | 12 KB | User Stack (8KB) + Kernel Stack (4KB) |

---

## 五、系列總結

### 5.1 我們學到了什麼？

| 章節 | 主題 | 核心概念 |
|------|------|----------|
| Ch 1-5 | 基礎建設 | RISC-V 啟動、UART、Timer |
| Ch 6-12 | v0.x Nano | Context Switch、Scheduler、同步原語 |
| Ch 13-19 | v1.x Secure | User Mode、PMP、Syscall |
| Ch 20-30 | v2.x MSMP | SMP、Spinlock、IPI、Cache Coherency |
| Ch 31-37 | v3.x | SMP + User Mode 整合 |

### 5.2 設計原則

在這個系列中，我們遵循了幾個重要的設計原則：

1. **漸進式開發**：從簡單開始，逐步增加複雜度
2. **先讓它工作，再讓它正確，最後讓它快**
3. **每個設計決策都有理由**：不是因為「別人這樣做」
4. **測試驅動**：每個功能都有對應的測試
5. **文件即程式碼**：程式碼和文件同步更新

### 5.3 未來展望

danieRTOS 還有很多可以改進的地方：

| 功能 | 描述 | 難度 |
|------|------|------|
| **S-mode 支援** | 加入 Supervisor Mode | ⭐⭐⭐ |
| **虛擬記憶體** | 使用 MMU 而非 PMP | ⭐⭐⭐⭐ |
| **動態記憶體** | malloc/free 實作 | ⭐⭐ |
| **檔案系統** | 簡單的 FAT 或 LittleFS | ⭐⭐⭐ |
| **網路堆疊** | lwIP 整合 | ⭐⭐⭐ |
| **更多核心** | 支援 4+ 核心 | ⭐⭐ |

---

## 六、致謝

感謝所有閱讀這個系列的讀者。

寫這個系列的過程中，我學到了很多。希望你也是。

如果你有任何問題或建議，歡迎在 GitHub 上開 Issue 或 PR。

---

## 七、附錄：完整程式碼

danieRTOS v3.x 的完整程式碼可以在以下位置找到：

```
code.v3/
├── src/
│   ├── boot/
│   │   ├── start.S          # 啟動程式碼
│   │   └── trap.S           # Trap 處理
│   ├── kernel/
│   │   ├── task.c           # Task 管理
│   │   ├── scheduler.c      # 排程器
│   │   ├── smp.c            # SMP 支援
│   │   ├── syscall.c        # Syscall 處理
│   │   ├── mutex.c          # Mutex 實作
│   │   ├── semaphore.c      # Semaphore 實作
│   │   └── queue.c          # Queue 實作
│   ├── drivers/
│   │   ├── uart.c           # UART 驅動
│   │   └── timer.c          # Timer 驅動
│   └── user/
│       └── demo.c           # Demo 程式
├── include/
│   ├── daniertos.h          # 主要標頭檔
│   ├── smp.h                # SMP 定義
│   └── pmp.h                # PMP 定義
├── Makefile
└── linker.ld
```

---

## 八、最後的話

> "The best way to learn is to build."

這個系列的目的不是教你如何使用現成的 RTOS，而是教你如何**從零開始建造一個**。

當你理解了底層的原理，你就能：
- 更好地使用現成的 RTOS
- 在遇到問題時知道從哪裡開始除錯
- 設計更好的系統架構

希望這個系列對你有幫助。

**Happy hacking!**

---

## 參考資料

**RISC-V 規格**

- [RISC-V Unprivileged Specification](https://riscv.org/specifications/)
- [RISC-V Privileged Specification](https://riscv.org/specifications/privileged-isa/)

**作業系統設計**

- "Operating Systems: Three Easy Pieces" by Remzi H. Arpaci-Dusseau
- "The Design and Implementation of the FreeBSD Operating System"
- Linux Kernel Source Code

**RTOS 參考**

- FreeRTOS
- Zephyr
- RT-Thread

**danieRTOS 系列**

- [See RISC-V Run](https://github.com/djiangtw/see-riscv-run)
- [danieRTOS GitHub](https://github.com/djiangtw/daniertos)

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)

---

**The End.**
```

