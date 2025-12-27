# 多核心擴展：從雙核心到八核心的驗證之旅

**作者**: Danny Jiang
**日期**: 2025-12-26

---

## 前言：理論與實踐的差距

> 「紙上得來終覺淺，絕知此事要躬行。」—— 陸游《冬夜讀書示子聿》

經過 Ch 31-39 的設計與實作，danieRTOS v3.x 在理論上已經支援 SMP + User Mode。但軟體工程有個殘酷的事實：**沒跑過的 code 都是假的**。

這一章，我們要做兩件事：
1. **驗證 v3p2 Fault Isolation**：User Task crash 時，系統是否真的保持穩定？
2. **驗證 v3p3 多核心擴展**：從 2 核心擴展到 4 核心、8 核心，系統還能正常運作嗎？

讓我們用實際的 demo 來證明一切。

---

## 一、Fault Isolation Demo 設計

### 1.1 測試情境

我們設計一個經典的「好人壞人」測試情境：

| Task | 模式 | 行為 | 預期結果 |
|------|------|------|----------|
| BadTask | U-mode | 嘗試寫入 kernel memory | 被終止 |
| GoodTask | U-mode | 正常計數 0-14 | 完成 |
| AnotherTask | U-mode | 正常計數 0-9 | 完成 |
| Monitor | M-mode | 監控系統狀態 | 報告系統穩定 |

### 1.2 BadTask 的惡意行為

```c
USER_FUNC static void bad_task(void *arg)
{
    (void)arg;

    sys_puts("[BadTask] Started in U-mode\n");
    sys_puts("[BadTask] Counting down to crash...\n");

    for (int i = 3; i > 0; i--) {
        sys_printf("[BadTask] %d...\n", i);
        sys_delay(100);
    }

    sys_puts("[BadTask] Attempting to access kernel memory...\n");

    /* 嘗試寫入 kernel memory (0x80000000) - 應該觸發 access fault */
    volatile int *kernel_ptr = (int *)0x80000000;
    *kernel_ptr = 42;  /* 這會觸發 Store Access Fault！ */

    /* 不應該執行到這裡 */
    sys_puts("[BadTask] ERROR: This should not print!\n");
}
```

關鍵點：
- `0x80000000` 是 Kernel 區域的起始地址
- PMP 設定為 `M:RWX, U:None`，User mode 沒有任何權限
- 寫入操作會觸發 **Store Access Fault (mcause=7)**

### 1.3 GoodTask 的正常行為

```c
USER_FUNC static void good_task(void *arg)
{
    (void)arg;
    int count = 0;

    sys_puts("[GoodTask] Started in U-mode\n");

    while (count < 15) {
        sys_printf("[GoodTask] Still running... %d\n", count);
        sys_delay(150);
        count++;
    }

    sys_puts("[GoodTask] Done! (System remained stable)\n");
    sys_exit();
}
```

GoodTask 會持續運行約 2250 ticks（15 × 150ms）。如果系統在 BadTask crash 後仍能讓 GoodTask 完成，就證明 Fault Isolation 成功。

---

## 二、運行 Fault Isolation Demo

### 2.1 編譯與執行

```bash
cd code.v3
make clean
make DEMO=fault SMP=2
make run
```

### 2.2 實際輸出

```
========================================
  danieRTOS v3.0 - SMP + User Mode
========================================

[Core 0] Initializing kernel...
[Core 0] SMP BSP initialized, cpu=0x0000000080018900
[Core 0] Created 2 idle tasks
[Core 0] Kernel initialized

=== danieRTOS v3p2 Fault Isolation Demo ===
BadTask will crash, but GoodTask should keep running.

All tasks created. Starting scheduler...

[Core 0] Signaling APs to start
[Core 0] Starting scheduler
[Core 1] SMP AP initialized, cpu=0x0000000080018940
[Core 0] Starting scheduler, first task: BadTask
[Core 1] Starting scheduler, first task: Monitor
[Monitor] Kernel monitor started
[Monitor] tick=0, system OK
[BadTask] Started in U-mode
[BadTask] Counting down to crash...
[GoodTask] Started in U-mode
[BadTask] 3...
[GoodTask] Still running... 0
[BadTask] 2...
[GoodTask] Still running... 1
[BadTask] 1...
[Monitor] tick=251, system OK
[BadTask] Attempting to access kernel memory...
[Fault][Core 0] mcause=7, mtval=0x80000000, mepc=0x800200ea (mode=U)
[Fault] Terminating user task 'BadTask'
[GoodTask] Still running... 2
[AnotherTask] Working... 2
...
[GoodTask] Done! (System remained stable)
[AnotherTask] Done!
[Monitor] All done - fault isolation verified!
```

### 2.3 關鍵事件分析

讓我們解讀 Fault 訊息：

```
[Fault][Core 0] mcause=7, mtval=0x80000000, mepc=0x800200ea (mode=U)
```

| 欄位 | 值 | 意義 |
|------|-----|------|
| Core | 0 | Fault 發生在 Core 0 |
| mcause | 7 | Store/AMO Access Fault |
| mtval | 0x80000000 | 嘗試存取的地址（Kernel 區域） |
| mepc | 0x800200ea | 發生 fault 的指令地址 |
| mode | U | 確認是 User mode 觸發的 fault |

**這正是 PMP 保護機制運作的證據！**

---

## 三、4-Core SMP 驗證

### 3.1 為什麼要測試 4 核心？

2 核心的 SMP 其實隱藏了很多問題：
- **競爭條件**：只有 2 個執行緒，發生衝突的機率較低
- **調度複雜度**：任務分配相對簡單
- **資源競爭**：同時存取 shared data 的情況較少

當核心數增加到 4 個，這些問題的發生機率會顯著上升。

### 3.2 編譯與執行

```bash
make clean
make DEMO=fault SMP=4
qemu-system-riscv64 -machine virt -cpu rv64 -smp 4 -m 128M \
    -nographic -bios none -kernel build/daniertos.elf
```

### 3.3 4-Core 輸出

```
========================================
  danieRTOS v3.0 - SMP + User Mode
========================================

[Core 0] Initializing kernel...
[Core 0] SMP BSP initialized, cpu=0x0000000080018900
[Core 0] Created 4 idle tasks
[Core 0] Kernel initialized

=== danieRTOS v3p2 Fault Isolation Demo ===
...

[Core 0] Starting scheduler, first task: BadTask
[Core 1] Starting scheduler, first task: Monitor
[Core 2] Starting scheduler, first task: GoodTask
[Core 3] Starting scheduler, first task: AnotherTask
...
[Fault][Core 0] mcause=7, mtval=0x80000000, mepc=0x800200ea (mode=U)
[Fault] Terminating user task 'BadTask'
...
[GoodTask] Done! (System remained stable)
[AnotherTask] Done!
[Monitor] All done - fault isolation verified!
```

**觀察重點**：
1. 4 個核心都正確初始化
2. 每個核心都被分配到一個任務
3. Core 0 的 BadTask fault 不影響其他核心
4. 所有任務正常完成

---

## 四、8-Core SMP 驗證

### 4.1 測試極限

既然 4 核心可以，我們再挑戰一下：**8 核心**。

```bash
make clean
make DEMO=fault SMP=8
qemu-system-riscv64 -machine virt -cpu rv64 -smp 8 -m 128M \
    -nographic -bios none -kernel build/daniertos.elf
```

### 4.2 8-Core 輸出

```
[Core 0] Initializing kernel...
[Core 0] SMP BSP initialized, cpu=0x0000000080018900
[Core 0] Created 8 idle tasks
[Core 0] Kernel initialized
...
[Core 0] Starting scheduler, first task: BadTask
[Core 1] Starting scheduler, first task: Monitor
[Core 2] Starting scheduler, first task: idle2
[Core 3] Starting scheduler, first task: idle3
[Core 4] Starting scheduler, first task: AnotherTask
[Core 5] Starting scheduler, first task: idle5
[Core 6] Starting scheduler, first task: GoodTask
[Core 7] Starting scheduler, first task: idle7
```

**觀察重點**：
1. 8 個核心全部成功初始化
2. 沒有足夠的任務給每個核心，所以有些核心運行 idle task
3. Fault isolation 依然正常運作
4. 系統在 8 核心下保持穩定

---

## 五、多核心擴展的技術細節

### 5.1 關鍵配置

在 `config.h` 中：

```c
/* 編譯時期決定的核心數量 */
#define CONFIG_NUM_CORES  SMP  /* 由 Makefile 傳入 */

/* 每核心的資料結構 */
cpu_t cpus[CONFIG_NUM_CORES];
```

### 5.2 Idle Task 自動創建

在 `smp.c` 的初始化中：

```c
void smp_init(void)
{
    /* 為每個核心創建 idle task */
    for (int i = 0; i < CONFIG_NUM_CORES; i++) {
        char name[16];
        snprintf(name, sizeof(name), "idle%d", i);

        task_create(name, idle_task, (void*)(uintptr_t)i, 0,
                    idle_stacks[i], IDLE_STACK_SIZE, (1U << i));

        cpus[i].idle_task = /* ... */;
    }

    uart_printf("[Core 0] Created %d idle tasks\n", CONFIG_NUM_CORES);
}
```

### 5.3 PMP 在多核心下的同步

每個核心都有獨立的 PMP 暫存器，但我們使用相同的配置：

```c
void pmp_init(void)
{
    /* 這個函數在每個核心的 startup 中被呼叫 */

    /* Entry 0: Kernel - M:RWX, U:None */
    pmp_set_entry(0, KERNEL_BASE, KERNEL_SIZE, PMP_NAPOT | PMP_R | PMP_W | PMP_X);

    /* Entry 1: User - M:RW, U:RWX */
    pmp_set_entry(1, USER_BASE, USER_SIZE, PMP_NAPOT | PMP_R | PMP_W | PMP_X | PMP_U);

    /* Entry 2: Default deny */
    pmp_set_entry(2, 0, -1, PMP_TOR);
}
```

**關鍵點**：所有核心使用相同的記憶體保護策略，確保一致的安全邊界。

---

## 六、驗證結果總結

### 6.1 測試矩陣

| 核心數 | 編譯 | 啟動 | Fault Isolation | 任務調度 | 系統穩定 |
|--------|------|------|-----------------|----------|----------|
| 2-Core | ✅ | ✅ | ✅ | ✅ | ✅ |
| 4-Core | ✅ | ✅ | ✅ | ✅ | ✅ |
| 8-Core | ✅ | ✅ | ✅ | ✅ | ✅ |

### 6.2 關鍵驗證點

1. **PMP 保護有效**
   - mcause=7 正確觸發
   - Kernel memory 不可被 User mode 存取

2. **Fault Isolation 成功**
   - BadTask 被終止，不影響其他任務
   - 系統保持穩定運行

3. **多核心調度正確**
   - 所有核心都正確參與調度
   - 任務可以在多核心間正確分配

4. **擴展性良好**
   - 從 2 核心到 8 核心，無需修改 kernel code
   - 只需調整編譯參數 `SMP=N`

---

## 總結

### 本章回顧

這一章，我們完成了 danieRTOS v3.x 的最終驗證：

1. **Fault Isolation Demo**
   - 設計「好人壞人」測試情境
   - 驗證 BadTask crash 不影響 GoodTask
   - 確認 PMP 保護機制正確運作

2. **多核心擴展測試**
   - 2-Core → 4-Core → 8-Core
   - 所有核心都正確初始化
   - Fault isolation 在多核心環境下正常運作

3. **關鍵發現**
   - danieRTOS v3.x 支援 2-8 核心的 SMP
   - 記憶體保護在所有核心上一致
   - 系統在極端情況下保持穩定

### 系列總結

從 Ch 01 的 "Hello, RISC-V!" 開始，到現在的 8-Core SMP + User Mode + Fault Isolation，我們走過了漫長的旅程：

| 版本 | 別名 | 章節 | 核心功能 |
|------|------|------|----------|
| v0.x | Nano | 01-12 | 基礎 RTOS：Task, Scheduler, Semaphore, Mutex, Queue |
| v1.x | Secure | 13-19 | User Mode：PMP, Syscall, Fault Handling |
| v2.x | MSMP | 20-30 | SMP：Spinlock, IPI, Multi-core Scheduler |
| v3.x | SMP | 31-40 | 整合：SMP + User Mode + Fault Isolation |

**40 篇文章，從零開始，打造一個教育級的 RISC-V RTOS。**

希望這個系列對你有所幫助。如果你有任何問題或建議，歡迎聯繫我！

---

## 參考資料

1. RISC-V Privileged Specification v1.12
2. RISC-V Instruction Set Manual
3. danieRTOS v3.x 原始碼：`code.v3/`
4. v3-spec.md：設計文檔

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)

