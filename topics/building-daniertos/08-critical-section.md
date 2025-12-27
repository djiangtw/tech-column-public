# Critical Section 與中斷管理

**作者**: Danny Jiang
**日期**: 2025-12-13

---

## 前言：銀行轉帳的災難

1991 年，一家銀行發現了一個令人困惑的問題。

客戶抱怨轉帳金額不對——有人轉了 $1000，但對方只收到 $500；有人轉帳失敗，但錢已經從帳戶扣除。

工程師檢查程式碼，看起來完全正確：

```c
void transfer(Account *from, Account *to, int amount) {
    if (from->balance >= amount) {
        from->balance -= amount;
        to->balance += amount;
    }
}
```

問題在於，這個系統是多執行緒的。當兩個轉帳同時發生，它們會互相干擾。這就是 **Race Condition**——一個在併發程式設計中最難發現、最難除錯的問題。

---

## 一個讓我除錯三天的 Bug

我自己也踩過這個坑。

有一次，我的程式莫名其妙地 crash，而且每次 crash 的地點都不一樣。

有時候是在 Task A，有時候是在 Task B，有時候根本不 crash 但輸出的數字是錯的。更詭異的是，當我加上 `printf()` 試圖除錯時，bug 就消失了。

「這一定是靈異事件，」我絕望地想。

後來我才知道，這是 **Race Condition** 的典型症狀。

想像兩個人同時編輯同一份 Google Doc。第一個人改了標題，第二個人改了內文，然後兩人同時按下「儲存」。結果可能是：只有一個人的修改被保留，另一個人的心血白費了。

在 RTOS 中，這種情況更加隱蔽。Task A 正在更新一個全域變數，執行到一半時，Timer Interrupt 發生，Scheduler 切換到 Task B。Task B 也去讀取同一個變數——但它看到的是「半成品」資料。

程式行為變得不可預測。而且因為 Context Switch 的時機是隨機的，bug 也是隨機的。

**這就是為什麼我加上 `printf()` 後 bug 消失了**——`printf()` 改變了時序，讓 Race Condition 剛好沒發生。這種 bug 被稱為「Heisenbug」（測不準 bug），因為你一觀察它，它就消失。

**Critical Section（臨界區段）** 是解決這個問題的最基本工具。它確保一段程式碼在執行時不會被打斷。

> 💡 **警告**：Race Condition 是多任務程式設計中最難除錯的問題之一。預防永遠比事後除錯容易。

---

## 一、Race Condition 範例

### 1.1 經典案例：Counter++

```c
volatile uint32_t counter = 0;

void task_a(void) {
    while (1) {
        counter++;  // 看起來很簡單...
        danie_delay(10);
    }
}

void task_b(void) {
    while (1) {
        counter++;  // 兩個 Task 都在加
        danie_delay(10);
    }
}
```

問題在於 `counter++` 不是原子操作。在 RISC-V 上，它被編譯成：

```asm
lw   t0, counter    # 1. 讀取 counter 到 t0
addi t0, t0, 1      # 2. t0 = t0 + 1
sw   t0, counter    # 3. 寫回 counter
```

如果在第 1 步和第 3 步之間發生 Context Switch...

### 1.2 災難發生

```
初始狀態：counter = 100

Task A:                         Task B:
  lw t0, counter  (t0 = 100)
  addi t0, t0, 1  (t0 = 101)
                                  lw t0, counter  (t0 = 100) ← 還沒更新！
                                  addi t0, t0, 1  (t0 = 101)
                                  sw t0, counter  (counter = 101)
  sw t0, counter  (counter = 101) ← 覆蓋了 Task B 的結果！

最終：counter = 101（應該是 102）
```

這就是 Race Condition。兩個 Task 各加了一次，但結果只增加了 1。

---

## 二、Critical Section 實作

### 2.1 最簡單的方式：關閉中斷

在 danieRTOS 中，Context Switch 只會在中斷處理中發生。如果我們**關閉中斷**，就不會有 Context Switch，也就不會有 Race Condition。

```c
void critical_enter(void) {
    // 關閉 Machine Interrupt Enable
    asm volatile("csrc mstatus, %0" : : "r"(1 << 3));
}

void critical_exit(void) {
    // 開啟 Machine Interrupt Enable
    asm volatile("csrs mstatus, %0" : : "r"(1 << 3));
}
```

### 2.2 使用範例

```c
void task_a(void) {
    while (1) {
        critical_enter();
        counter++;  // 受保護的操作
        critical_exit();
        
        danie_delay(10);
    }
}
```

### 2.3 問題：Nested Critical Section

如果 Critical Section 裡面又呼叫了另一個函數，而那個函數也使用 Critical Section 呢？

```c
void foo(void) {
    critical_enter();
    // ...
    critical_exit();  // 開啟中斷！
}

void bar(void) {
    critical_enter();
    foo();  // foo() 裡面會 exit，開啟中斷
    // 這裡中斷已經開啟了，但我們還在 bar() 的 Critical Section！
    critical_exit();
}
```

這會破壞保護。

### 2.4 解法：儲存並恢復中斷狀態

```c
static volatile uint32_t critical_nesting = 0;

void critical_enter(void) {
    // 關閉中斷
    asm volatile("csrc mstatus, %0" : : "r"(1 << 3));
    critical_nesting++;
}

void critical_exit(void) {
    critical_nesting--;
    if (critical_nesting == 0) {
        // 只有最外層才開啟中斷
        asm volatile("csrs mstatus, %0" : : "r"(1 << 3));
    }
}
```

---

## 三、更安全的實作

### 3.1 保存原本的中斷狀態

更安全的做法是保存進入時的中斷狀態，離開時恢復：

```c
typedef uint64_t critical_state_t;

critical_state_t critical_enter_save(void) {
    critical_state_t state;
    asm volatile("csrr %0, mstatus" : "=r"(state));
    asm volatile("csrc mstatus, %0" : : "r"(1 << 3));
    return state;
}

void critical_exit_restore(critical_state_t state) {
    asm volatile("csrw mstatus, %0" : : "r"(state));
}
```

### 3.2 使用巨集簡化

```c
#define CRITICAL_SECTION_BEGIN() \
    critical_state_t __saved_state = critical_enter_save()

#define CRITICAL_SECTION_END() \
    critical_exit_restore(__saved_state)

// 使用
void some_function(void) {
    CRITICAL_SECTION_BEGIN();
    // 受保護的程式碼
    counter++;
    CRITICAL_SECTION_END();
}
```

---

## 四、ISR 安全的設計模式

### 4.1 問題：從 ISR 呼叫 API

有些 RTOS API 不能從 ISR（Interrupt Service Routine）中呼叫。例如：

```c
void timer_isr(void) {
    danie_delay(10);  // 錯誤！ISR 不能 delay
}
```

為什麼？因為 `delay()` 會把當前 Task 放入 Blocked 狀態，但 ISR 不是 Task！

### 4.2 解法：提供 ISR 專用的 API

FreeRTOS 的做法是提供兩套 API：

| Task API | ISR API |
|----------|---------|
| `xSemaphoreTake()` | `xSemaphoreTakeFromISR()` |
| `xQueueSend()` | `xQueueSendFromISR()` |

ISR 版本的特點：

1. **不會 Block**：如果資源不可用，立即返回失敗
2. **可能觸發 Context Switch**：返回一個 flag 表示是否需要切換

### 4.3 danieRTOS 的設計

```c
// 檢查是否在 ISR 中
bool in_interrupt_context(void) {
    // 可以用一個全域變數來追蹤
    return interrupt_nesting > 0;
}

// 進入 ISR 時呼叫
void isr_enter(void) {
    interrupt_nesting++;
}

// 離開 ISR 時呼叫
void isr_exit(void) {
    interrupt_nesting--;
}
```

在 Trap Handler 中：

```c
void handle_trap(uint64_t mcause, uint64_t mepc) {
    isr_enter();

    if (is_timer_interrupt(mcause)) {
        handle_timer_interrupt();
    }
    // ...

    isr_exit();
}
```

### 4.4 API 中的檢查

```c
void danie_delay(uint32_t ticks) {
    if (in_interrupt_context()) {
        danie_panic("delay() called from ISR!");
    }
    // ...
}
```

---

## 五、Critical Section 的代價

### 5.1 關閉中斷的影響

關閉中斷會導致：

1. **延遲中斷響應**：Timer Interrupt 可能延遲
2. **影響即時性**：高優先級 Task 無法立即搶佔

### 5.2 最佳實踐

**Critical Section 要盡可能短**：

```c
// 好的做法
void update_data(int new_value) {
    int processed = expensive_calculation(new_value);  // 在外面算

    critical_enter();
    shared_data = processed;  // 只保護最小範圍
    critical_exit();
}

// 壞的做法
void update_data(int new_value) {
    critical_enter();
    int processed = expensive_calculation(new_value);  // 在裡面算
    shared_data = processed;
    critical_exit();  // 中斷關閉太久了！
}
```

### 5.3 替代方案

當需要保護的時間較長時，可以考慮：

1. **Mutex**：只阻止其他 Task，不影響中斷
2. **Atomic 操作**：使用 RISC-V 的原子指令（如 `amoadd`）
3. **Lock-free 資料結構**：進階技術，避免鎖定

---

## 六、RISC-V 原子指令（進階）

### 6.1 AMO 指令

RISC-V 的 A 擴展提供了 Atomic Memory Operations：

```asm
# 原子加法：counter += 1
li    t1, 1
amoadd.w t0, t1, (counter)  # t0 = 舊值，counter = 舊值 + 1
```

這個指令是**不可分割的**，不會被中斷打斷。

### 6.2 在 C 中使用

```c
// 使用 GCC 內建函數
__atomic_add_fetch(&counter, 1, __ATOMIC_SEQ_CST);

// 或者用 inline assembly
static inline uint32_t atomic_add(volatile uint32_t *ptr, uint32_t val) {
    uint32_t old;
    asm volatile("amoadd.w %0, %1, (%2)"
                 : "=r"(old)
                 : "r"(val), "r"(ptr)
                 : "memory");
    return old;
}
```

### 6.3 適用場景

原子指令適合：

- 簡單的計數器
- Flag 設定
- 單一變數的更新

不適合：

- 需要同時更新多個變數
- 複雜的資料結構操作

---

## 總結

本文實作了 danieRTOS 的 Critical Section 機制：

1. **Race Condition**：多 Task 同時存取共享資源的危險
2. **關閉中斷**：最簡單的 Critical Section 實作
3. **Nested 支援**：使用 nesting counter 或保存狀態
4. **ISR 安全**：區分 Task API 和 ISR API
5. **最佳實踐**：Critical Section 要盡可能短
6. **原子指令**：RISC-V A 擴展的進階用法

Critical Section 解決了「互斥存取」的問題，但如果 Task 需要「等待資源」呢？這就是 Semaphore 的用武之地。

---

## 參考資料

**RISC-V 規格**

- **RISC-V Instruction Set Manual, Volume I: Unprivileged ISA - A Extension**
  RISC-V International
  https://github.com/riscv/riscv-isa-manual
  Atomic Instructions（AMO）的官方規格。

**RTOS 參考實作**

- **FreeRTOS Kernel**
  <https://github.com/FreeRTOS/FreeRTOS-Kernel>
  taskENTER_CRITICAL() 和 taskEXIT_CRITICAL() 的參考實作。

**歷史案例**

- **Therac-25 事故**
  <https://en.wikipedia.org/wiki/Therac-25>
  Race Condition 導致的放射治療事故。

---

## 常見錯誤與 Debug 技巧

### 錯誤 1：忘記退出 Critical Section

```c
void buggy_function(void) {
    critical_enter();
    if (error_condition) {
        return;  // 錯誤！沒有 critical_exit()
    }
    critical_exit();
}
```

**後果**：中斷永遠被關閉，系統卡死。

**解法**：使用 goto 或確保所有路徑都有 exit。

### 錯誤 2：在 Critical Section 中呼叫可能 Block 的函數

```c
void buggy_function(void) {
    critical_enter();
    uart_write_blocking("Hello");  // 如果 UART 忙碌會等待
    critical_exit();
}
```

**後果**：中斷關閉期間等待 I/O，其他中斷無法處理。

### 錯誤 3：Deadlock

```c
void task_a(void) {
    critical_enter();
    // ... 等待 Task B 設定的 flag ...
    critical_exit();
}

void task_b(void) {
    // 因為中斷被關閉，Task B 永遠不會執行
    flag = 1;
}
```

**後果**：Task A 在等 Task B，但 Task B 無法執行。

### Debug 技巧

1. **加入 Timeout**：Critical Section 超過一定時間就 Panic
2. **記錄 Nesting**：追蹤誰進入了 Critical Section
3. **使用 Watchdog**：系統卡死時自動 Reset

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)
