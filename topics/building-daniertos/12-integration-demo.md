# 整合與 Demo

**作者**: Danny Jiang
**日期**: 2025-12-13

---

## 前言：旅程的終點，也是起點

還記得這個系列的第一篇文章嗎？那時我說：「與其用現成的 RTOS，不如自己寫一個。」

現在，我們真的做到了。

如果你跟著走到這裡，你已經經歷了一段不平凡的旅程：

- 你見證了 CPU 從 Reset 到第一行 C 程式碼的過程
- 你理解了為什麼「少存一個暫存器就會 Crash」
- 你設計了一個 O(1) 的 Scheduler
- 你解決了 Mars Pathfinder 同樣遇到的 Priority Inversion 問題

這些不是課本上的知識，而是你親手實作、親眼見證的經驗。

我記得第一次看到 danieRTOS 在 QEMU 上成功運行多個 Task 時的感覺——那種「我真的理解這個系統的每一個細節」的成就感，是用任何現成框架都無法獲得的。

現在，讓我們把所有元件整合起來，在 QEMU 上運行一個完整的 Demo。這是我們的畢業典禮。

> 💡 **回顧**：如果你在任何環節卡住，可以回去複習對應的文章。每一篇都是後續的基礎。

---

## 一、專案結構

### 1.1 檔案組織

```
danieRTOS/
├── src/
│   ├── main.c           # 主程式與 Task 定義
│   ├── kernel/
│   │   ├── task.c       # Task 管理
│   │   ├── scheduler.c  # 排程器
│   │   ├── tick.c       # Timer Tick
│   │   ├── delay.c      # Delay 機制
│   │   ├── critical.c   # Critical Section
│   │   ├── semaphore.c  # Semaphore
│   │   ├── mutex.c      # Mutex
│   │   └── queue.c      # Queue
│   └── hal/
│       ├── uart.c       # UART 驅動
│       ├── timer.c      # Timer 驅動
│       └── trap.S       # Trap Handler
├── include/
│   ├── daniertos.h      # 主要 API
│   ├── config.h         # 設定
│   └── hal.h            # HAL 介面
├── linker.ld            # Linker Script
└── Makefile
```

### 1.2 設定檔

```c
// config.h
#ifndef DANIERTOS_CONFIG_H
#define DANIERTOS_CONFIG_H

#define CONFIG_MAX_TASKS        8
#define CONFIG_MAX_PRIORITY     4
#define CONFIG_TICK_RATE_HZ     1000    // 1ms per tick
#define CONFIG_MINIMAL_STACK    512
#define CONFIG_IDLE_STACK       256

#endif
```

---

## 二、完整 Demo：多 Task 協作

### 2.1 Demo 情境

我們設計一個模擬「感測器資料處理」的 Demo：

```
┌─────────────┐     Queue     ┌─────────────┐
│ Sensor Task │ ──────────→ │ Process Task│
│ (Priority 2)│    data      │ (Priority 2)│
└─────────────┘               └──────┬──────┘
                                     │
                              Semaphore (notify)
                                     │
                                     ▼
                              ┌─────────────┐
                              │ Logger Task │
                              │ (Priority 1)│
                              └─────────────┘

┌─────────────┐
│  LED Task   │  ← 每 500ms 切換 LED
│ (Priority 1)│
└─────────────┘

┌─────────────┐
│  Idle Task  │  ← 沒事做時執行
│ (Priority 0)│
└─────────────┘
```

### 2.2 主程式

```c
// main.c
#include "daniertos.h"
#include "hal.h"

// 資料結構
typedef struct {
    uint32_t timestamp;
    int32_t value;
} sensor_data_t;

// IPC 物件
queue_t sensor_queue;
sensor_data_t sensor_buffer[8];

semaphore_t log_semaphore;
volatile int32_t last_processed_value;

// Task TCBs
tcb_t sensor_tcb, process_tcb, logger_tcb, led_tcb, idle_tcb;

// Task Stacks
uint8_t sensor_stack[1024];
uint8_t process_stack[1024];
uint8_t logger_stack[512];
uint8_t led_stack[512];
uint8_t idle_stack[256];

// ─────────────────────────────────────
// Sensor Task：模擬感測器讀取
// ─────────────────────────────────────
void sensor_task(void *arg) {
    (void)arg;
    int32_t fake_value = 0;

    uart_puts("Sensor Task started\n");

    while (1) {
        sensor_data_t data = {
            .timestamp = get_tick_count(),
            .value = fake_value++
        };

        if (queue_send(&sensor_queue, &data, 100)) {
            // 成功送出
        } else {
            uart_puts("Sensor: Queue full!\n");
        }

        danie_delay(200);  // 每 200ms 讀取一次
    }
}

// ─────────────────────────────────────
// Process Task：處理感測器資料
// ─────────────────────────────────────
void process_task(void *arg) {
    (void)arg;

    uart_puts("Process Task started\n");

    while (1) {
        sensor_data_t data;

        if (queue_receive(&sensor_queue, &data, WAIT_FOREVER)) {
            // 模擬處理
            int32_t result = data.value * 2;
            last_processed_value = result;

            uart_printf("Processed: %d -> %d (t=%u)\n",
                       data.value, result, data.timestamp);

            // 通知 Logger
            semaphore_give(&log_semaphore);
        }
    }
}

// ─────────────────────────────────────
// Logger Task：記錄處理結果
// ─────────────────────────────────────
void logger_task(void *arg) {
    (void)arg;
    uint32_t log_count = 0;

    uart_puts("Logger Task started\n");

    while (1) {
        if (semaphore_take(&log_semaphore, WAIT_FOREVER)) {
            log_count++;
            uart_printf("[LOG #%u] Value = %d\n",
                       log_count, last_processed_value);
        }
    }
}

// ─────────────────────────────────────
// LED Task：閃爍指示燈
// ─────────────────────────────────────
void led_task(void *arg) {
    (void)arg;
    int led_state = 0;

    uart_puts("LED Task started\n");

    while (1) {
        led_state = !led_state;
        uart_printf("LED: %s\n", led_state ? "ON" : "OFF");
        danie_delay(500);
    }
}

// ─────────────────────────────────────
// Idle Task：背景任務
// ─────────────────────────────────────
void idle_task(void *arg) {
    (void)arg;

    while (1) {
        // 可以放 WFI 指令節省電力
        asm volatile("wfi");
    }
}

// ─────────────────────────────────────
// Main
// ─────────────────────────────────────
int main(void) {
    // 初始化硬體
    uart_init();
    timer_init();

    uart_puts("\n=== danieRTOS Demo ===\n\n");

    // 初始化 IPC
    queue_init(&sensor_queue, sensor_buffer, sizeof(sensor_data_t), 8);
    binary_semaphore_init(&log_semaphore);

    // 初始化 Kernel
    kernel_init();

    // 創建 Tasks
    task_create(&sensor_tcb, sensor_task, NULL,
                sensor_stack, sizeof(sensor_stack), 2);

    task_create(&process_tcb, process_task, NULL,
                process_stack, sizeof(process_stack), 2);

    task_create(&logger_tcb, logger_task, NULL,
                logger_stack, sizeof(logger_stack), 1);

    task_create(&led_tcb, led_task, NULL,
                led_stack, sizeof(led_stack), 1);

    task_create(&idle_tcb, idle_task, NULL,
                idle_stack, sizeof(idle_stack), 0);

    // 啟動排程器（不會返回）
    scheduler_start();

    // 不應該到這裡
    while (1);
}
```

---

## 三、在 QEMU 上運行

### 3.1 編譯

```bash
# Makefile 示意
CROSS = riscv64-unknown-elf-
CC = $(CROSS)gcc
CFLAGS = -march=rv64imac -mabi=lp64 -mcmodel=medany \
         -ffreestanding -nostdlib -O2 -g
LDFLAGS = -T linker.ld

all: daniertos.elf

daniertos.elf: $(OBJS)
    $(CC) $(LDFLAGS) -o $@ $^

clean:
    rm -f *.o *.elf
```

```bash
make clean && make
```

### 3.2 運行 QEMU

```bash
qemu-system-riscv64 \
    -machine virt \
    -cpu rv64 \
    -smp 1 \
    -m 128M \
    -nographic \
    -bios none \
    -kernel daniertos.elf
```

### 3.3 預期輸出

```
=== danieRTOS Demo ===

Sensor Task started
Process Task started
Logger Task started
LED Task started
LED: ON
Processed: 0 -> 0 (t=0)
[LOG #1] Value = 0
Processed: 1 -> 2 (t=200)
[LOG #2] Value = 2
LED: OFF
Processed: 2 -> 4 (t=400)
[LOG #3] Value = 4
LED: ON
Processed: 3 -> 6 (t=600)
...
```

---

## 四、GDB 除錯

### 4.1 啟動 QEMU（等待 GDB）

```bash
qemu-system-riscv64 \
    -machine virt \
    -cpu rv64 \
    -smp 1 \
    -m 128M \
    -nographic \
    -bios none \
    -kernel daniertos.elf \
    -S -s  # -S: 啟動時暫停, -s: 開啟 GDB server (port 1234)
```

### 4.2 連接 GDB

```bash
riscv64-unknown-elf-gdb daniertos.elf

(gdb) target remote :1234
(gdb) break main
(gdb) continue
```

### 4.3 實用 GDB 指令

```gdb
# 查看所有 Task
(gdb) print sensor_tcb
(gdb) print process_tcb

# 查看當前執行的 Task
(gdb) print *current_tcb

# 查看 Ready List
(gdb) print ready_list[0]
(gdb) print ready_list[1]
(gdb) print ready_list[2]

# 在 Context Switch 設中斷點
(gdb) break context_switch

# 在 Timer Interrupt 設中斷點
(gdb) break timer_interrupt_handler

# 查看暫存器
(gdb) info registers

# 單步執行
(gdb) stepi

# 查看 Queue 狀態
(gdb) print sensor_queue
```

### 4.4 驗證排程行為

```gdb
# 觀察 Task 切換順序
(gdb) break schedule
(gdb) commands
> print current_tcb->name
> continue
> end
(gdb) continue
```

---

## 五、驗證重點

### 5.1 Priority Preemption

測試：高優先級 Task 喚醒時，立即搶佔低優先級 Task

```c
void high_priority_test(void) {
    while (1) {
        semaphore_take(&test_sem, WAIT_FOREVER);
        uart_puts("HIGH: Got semaphore!\n");
    }
}

void low_priority_test(void) {
    while (1) {
        uart_puts("LOW: Working...\n");
        semaphore_give(&test_sem);  // 應立即被搶佔
        uart_puts("LOW: Back!\n");  // 高優先級 Task 執行完才會到這
        danie_delay(1000);
    }
}
```

### 5.2 Delay 精確度

```c
void timing_test(void) {
    while (1) {
        uint32_t start = get_tick_count();
        danie_delay(1000);  // 1 秒
        uint32_t elapsed = get_tick_count() - start;
        uart_printf("Delay 1000 ticks, actual: %u\n", elapsed);
    }
}
```

### 5.3 Queue 流量控制

```c
void fast_producer(void) {
    int i = 0;
    while (1) {
        if (!queue_send(&q, &i, 0)) {  // 不等待
            uart_puts("Queue full, dropping\n");
        }
        i++;
        danie_delay(10);  // 快速產生
    }
}

void slow_consumer(void) {
    while (1) {
        int data;
        queue_receive(&q, &data, WAIT_FOREVER);
        uart_printf("Consumed: %d\n", data);
        danie_delay(100);  // 慢速消費
    }
}
```

---

## 總結

恭喜！經過這個系列，你已經從零開始實作了一個完整的 RTOS：

| 功能 | 狀態 |
|------|------|
| Task 創建與管理 | ✅ |
| Priority-based Scheduling | ✅ |
| Preemptive Multitasking | ✅ |
| Timer Tick | ✅ |
| Delay / Sleep | ✅ |
| Critical Section | ✅ |
| Semaphore | ✅ |
| Mutex with Priority Inheritance | ✅ |
| Queue | ✅ |

### 學到的不只是程式碼

這個專案的價值不在於程式碼本身，而在於：

1. **理解 RTOS 的核心機制**：Context Switch 如何運作、為什麼需要 Semaphore 等
2. **掌握 RISC-V 底層**：CSR、Trap、Timer 等硬體互動
3. **系統程式設計思維**：中斷安全、Race Condition、資源管理

### 進階方向

如果你想繼續深入，可以考慮：

- **Tickless Mode**：低功耗設計，不需要時關閉 Timer
- **Memory Pool**：動態記憶體管理
- **Event Groups**：更靈活的同步機制
- **Software Timer**：不佔用硬體 Timer 的延遲執行
- **Stack Overflow Detection**：MPU 或 Canary 保護
- **Multi-core Support**：SMP 排程

---

## 參考資料

**RTOS 參考實作**

- **FreeRTOS Kernel**
  https://github.com/FreeRTOS/FreeRTOS-Kernel
  danieRTOS 的主要參考對象。

**RISC-V 規格**

- **RISC-V Instruction Set Manual**
  RISC-V International
  https://github.com/riscv/riscv-isa-manual
  Unprivileged 和 Privileged 規格。

**開發工具**

- **QEMU RISC-V Emulator**
  https://www.qemu.org/docs/master/system/target-riscv.html
  本系列使用的模擬環境。

**延伸閱讀**

- **See RISC-V Run**
  Danny Jiang
  RISC-V 架構深入解析。

- **Data Structures in Practice**
  Danny Jiang
  資料結構實作，包含 Linked List、Ring Buffer。

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)
