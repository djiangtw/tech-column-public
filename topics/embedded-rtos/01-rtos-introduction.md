# RTOS 入門：用 FreeRTOS 理解即時作業系統

**作者**: Danny Jiang
**日期**: 2025-12-09

---

## 前言：一個讓人徹夜難眠的 Bug

2016 年，我在一家工業控制設備公司擔任韌體工程師。有一天，測試團隊報告了一個詭異的問題：

**馬達控制系統偶爾會「抖動」，導致產品不良率從 0.1% 飆升到 5%。**

這個問題只在高負載時出現，而且無法穩定重現。我們花了兩週時間，用示波器、邏輯分析儀、GDB 除錯器輪番上陣，終於找到了根本原因：

**我們的韌體使用 Linux 作為作業系統，但 Linux 是 GPOS (General-Purpose OS)，不是 RTOS (Real-Time OS)。**

當系統負載高時，Linux Scheduler 可能會延遲馬達控制 Task 的執行時間，導致 PWM 訊號產生時間不穩定。即使只延遲 5 毫秒，馬達就會抖動，產品就報廢了。

最後，我們將系統改用 **FreeRTOS**，問題完全消失。不良率回到 0.1%，系統穩定運行至今。

這個慘痛的教訓讓我深刻體會到：**選錯作業系統，再好的硬體也救不了你。**

本文將用 FreeRTOS 作為範例，帶你理解 RTOS 的核心概念。讀完這篇文章，你將能夠：

- 理解 RTOS 與 GPOS 的核心差異
- 知道 FreeRTOS 的核心組件（Task, Queue, Semaphore）
- 了解為什麼選擇 QEMU + RISC-V 作為學習平台
- 掌握基本的環境設定

---

## 一、RTOS vs. GPOS：核心差異在哪裡？

### 1.1 什麼是「即時」(Real-Time)？

**「即時」不是「很快」，而是「可預測」。**

想像一下兩種送貨服務：

**快遞 A（GPOS）**：

- 平均送達時間：1 天
- 最快：4 小時
- 最慢：3 天
- **不保證**什麼時候送到

**快遞 B（RTOS）**：

- 保證送達時間：24 小時內
- 最快：12 小時
- 最慢：23 小時
- **保證**在 24 小時內送到

**哪個是「即時」？** 答案是快遞 B。

即使快遞 A 平均更快，但它無法保證最壞情況下的送達時間。對於需要「準時」的應用（如馬達控制、飛行控制、醫療設備），**可預測性比平均速度更重要**。

### 1.2 RTOS 的三大特性

**1. Deterministic Behavior（確定性行為）**

RTOS 保證：

- Task 切換時間是固定的（例如：5 微秒）
- 中斷回應時間有上限（例如：10 微秒）
- 最高優先權的 Task 一定會在 Deadline 前執行

**2. Priority-Based Preemptive Scheduling（基於優先權的搶佔式排程）**

```
高優先權 Task 可以「插隊」，立即搶佔 CPU
低優先權 Task 必須等待
```

**3. Minimal Latency（最小延遲）**

RTOS 的核心設計目標：

- 中斷延遲 (Interrupt Latency) < 10 微秒
- Context Switch 時間 < 5 微秒
- 系統呼叫 (System Call) 時間 < 1 微秒

### 1.3 GPOS 的設計目標

相比之下，GPOS（如 Linux, Windows）的設計目標是：

**1. Throughput（吞吐量）**

- 最大化整體系統效能
- 平均回應時間最佳化

**2. Fairness（公平性）**

- 每個 Process 都能分到 CPU 時間
- 避免 Starvation（飢餓）

**3. Flexibility（彈性）**

- 支援各種應用場景
- 豐富的功能和驅動程式

**GPOS 不保證最壞情況下的回應時間。**

### 1.4 重要觀念：並發 (Concurrency) vs. 並行 (Parallelism)

在討論 RTOS 時，經常會混淆這兩個概念。讓我們澄清：

**並發 (Concurrency)**：

- 定義：透過時間切片 (Time Slicing) 快速切換，讓多個任務「看起來」像是一起執行
- 實際：同一瞬間只有一個 Task 在執行
- 適用：單核心 (Single Core) 環境
- 範例：FreeRTOS 在單核心 RISC-V 上執行多個 Task

**並行 (Parallelism)**：

- 定義：物理上「真正同時」執行多個任務
- 實際：多個 Task 在不同的 CPU Core 上同時執行
- 適用：多核心 (Multi-Core) 環境
- 範例：FreeRTOS SMP 在 4 核心 RISC-V 上執行 4 個 Task

**本系列的重點**：

- 文章 #1-#5 聚焦於**單核心環境**，討論的是**並發 (Concurrency)**
- 文章 #6 才會討論**多核心 SMP**，涉及**並行 (Parallelism)**

**記住**：在單核心環境下，RTOS 透過「快速切換」實現並發，而非真正的並行。

### 1.5 真實案例：為什麼 Linux 不適合馬達控制？

讓我們看看我遇到的問題：

**需求**：

- 馬達控制 Task 必須每 1 毫秒更新一次 PWM 訊號
- 如果延遲超過 5 毫秒，馬達會抖動

**Linux 的問題**：

```
正常情況：
  Task 每 1 ms 執行一次 ✅

高負載情況（CPU 100%）：
  Task 可能被延遲 10-50 ms ❌
  原因：Linux Scheduler 優先處理其他 Process
  結果：馬達抖動，產品報廢
```

**FreeRTOS 的解決方案**：

```
無論負載多高：
  最高優先權的 Task 保證在 Deadline 前執行 ✅
  Context Switch 時間 < 5 μs
  中斷延遲 < 10 μs
  結果：馬達穩定運行
```

---

## 二、FreeRTOS 核心組件

FreeRTOS 是一個輕量級的 RTOS，專為嵌入式系統設計。讓我們看看它的核心組件。

### 2.1 Task（任務）

**Task** 是 FreeRTOS 的基本執行單位，類似於 Linux 的 Thread。

**比喻**：想像一個工廠，每個 Task 是一條生產線。

**Task 的特性**：

- 每個 Task 有自己的 Stack（堆疊）
- 每個 Task 有自己的優先權（Priority）
- Task 可以是無窮迴圈（大多數情況）

**範例**：

```c
void vTaskLED(void *pvParameters) {
    while (1) {
        GPIO_Toggle(LED_PIN);
        vTaskDelay(pdMS_TO_TICKS(500));  // 延遲 500 ms
    }
}

// 建立 Task
xTaskCreate(
    vTaskLED,           // Task 函式
    "LED Task",         // Task 名稱
    128,                // Stack 大小（words）
    NULL,               // 參數
    1,                  // 優先權
    NULL                // Task Handle
);
```

### 2.2 Scheduler（排程器）

**Scheduler** 決定哪個 Task 應該執行。

**FreeRTOS 的排程策略**：

1. **Priority-Based Preemptive Scheduling**
   - 最高優先權的 Ready Task 會執行
   - 高優先權 Task 可以搶佔低優先權 Task

2. **Round Robin（同優先權）**
   - 同優先權的 Task 輪流執行
   - 每個 Task 執行一個 Time Slice（時間片）

**比喻**：Scheduler 就像工廠的生產排程員，決定哪條生產線應該開工。

### 2.3 Queue（佇列）

**Queue** 用於 Task 之間的資料傳遞。

**特性**：

- FIFO（First In First Out）
- Thread-Safe（執行緒安全）
- 可以設定 Timeout

**範例**：

```c
QueueHandle_t xQueue;

// 建立 Queue（可存放 10 個 int）
xQueue = xQueueCreate(10, sizeof(int));

// Task A：發送資料
void vTaskSender(void *pvParameters) {
    int data = 42;
    xQueueSend(xQueue, &data, portMAX_DELAY);
}

// Task B：接收資料
void vTaskReceiver(void *pvParameters) {
    int received;
    xQueueReceive(xQueue, &received, portMAX_DELAY);
    printf("Received: %d\n", received);
}
```

**比喻**：Queue 就像工廠的傳送帶，生產線 A 把零件放上去，生產線 B 從傳送帶取下來。

### 2.4 Semaphore（號誌）

**Semaphore** 用於同步和互斥。

**兩種類型**：

**1. Binary Semaphore（二元號誌）**

- 用於同步（Synchronization）
- 類似於「通知」機制

**2. Counting Semaphore（計數號誌）**

- 用於資源管理
- 例如：管理 5 個可用的 Buffer

**範例**：

```c
SemaphoreHandle_t xSemaphore;

// 建立 Binary Semaphore
xSemaphore = xSemaphoreCreateBinary();

// Task A：等待訊號
void vTaskWait(void *pvParameters) {
    xSemaphoreTake(xSemaphore, portMAX_DELAY);
    printf("Semaphore received!\n");
}

// ISR：發送訊號
void vISR(void) {
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;
    xSemaphoreGiveFromISR(xSemaphore, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

**比喻**：Semaphore 就像工廠的「開工鈴」，鈴響了，生產線才能開始工作。

### 2.5 Mutex（互斥鎖）

**Mutex** 用於保護共享資源。

**特性**：

- 只有「拿到鎖」的 Task 可以存取資源
- 支援 Priority Inheritance（優先權繼承）

**範例**：

```c
SemaphoreHandle_t xMutex;

// 建立 Mutex
xMutex = xSemaphoreCreateMutex();

// Task A 和 Task B 都要存取共享資源
void vTaskShared(void *pvParameters) {
    xSemaphoreTake(xMutex, portMAX_DELAY);  // 取得鎖

    // 存取共享資源（Critical Section）
    shared_variable++;

    xSemaphoreGive(xMutex);  // 釋放鎖
}
```

**比喻**：Mutex 就像工廠的「工具間鑰匙」，同一時間只有一個人可以進去拿工具。

---

## 三、為什麼選擇 QEMU + RISC-V？

在學習 RTOS 時，我們需要一個實驗平台。為什麼選擇 QEMU + RISC-V？

### 3.1 QEMU 的優勢

**QEMU** 是一個開源的虛擬機，可以模擬各種硬體平台。

**優勢**：

1. **免費且開源**：不需要購買開發板
2. **完整的除錯支援**：內建 GDB Server
3. **可重現性**：環境完全一致，不受硬體差異影響
4. **快速迭代**：不需要燒錄韌體，直接執行

**劣勢**：

- 無法測試真實硬體的時序特性
- 性能分析數據僅供參考

### 3.2 RISC-V 的優勢

**RISC-V** 是一個開源的指令集架構（ISA）。

**優勢**：

1. **開源且免費**：沒有授權費用
2. **簡潔的設計**：指令集精簡，易於學習
3. **模組化**：可以根據需求選擇擴展（M, A, F, D, C）
4. **產業支援**：SiFive, Andes, Alibaba 等公司支持

**與 ARM Cortex-M 的比較**：

| 特性 | RISC-V | ARM Cortex-M |
|------|--------|--------------|
| 授權 | 開源免費 | 需要授權費 |
| 指令集 | 精簡（~50 條基本指令） | 較複雜 |
| Context Switch | 軟體主導 | 硬體輔助 |
| 學習曲線 | 較平緩 | 較陡峭 |
| 產業應用 | 快速成長 | 成熟穩定 |

### 3.3 FreeRTOS + QEMU + RISC-V 的組合

**為什麼這個組合適合學習？**

1. **完全開源**：所有工具和原始碼都可以免費取得
2. **社群支援**：豐富的文檔和範例
3. **深入理解**：可以看到從 C 程式碼到組合語言的完整流程
4. **實務價值**：RISC-V 在嵌入式領域快速成長

---

## 四、環境設定

讓我們快速設定 FreeRTOS + QEMU + RISC-V 的開發環境。

### 4.1 安裝 RISC-V Toolchain

**Ubuntu/Debian**：

```bash
sudo apt-get install gcc-riscv64-unknown-elf
sudo apt-get install gdb-multiarch
```

**macOS**：

```bash
brew tap riscv/riscv
brew install riscv-tools
```

**驗證安裝**：

```bash
riscv64-unknown-elf-gcc --version
```

### 4.2 安裝 QEMU

**Ubuntu/Debian**：

```bash
sudo apt-get install qemu-system-riscv64
```

**macOS**：

```bash
brew install qemu
```

**驗證安裝**：

```bash
qemu-system-riscv64 --version
```

### 4.3 下載 FreeRTOS

```bash
git clone https://github.com/FreeRTOS/FreeRTOS.git
cd FreeRTOS/FreeRTOS/Demo/RISC-V-Qemu-virt_GCC
```

### 4.4 編譯和執行

**編譯**：

```bash
make
```

**執行**：

```bash
qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -bios none \
    -kernel build/RTOSDemo.elf
```

**預期輸出**：

```
FreeRTOS V10.x.x
Starting scheduler...
Task 1 running
Task 2 running
...
```

**退出 QEMU**：按 `Ctrl-A` 然後按 `X`

### 4.5 使用 GDB 除錯

**啟動 QEMU（GDB Server 模式）**：

```bash
qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -bios none \
    -kernel build/RTOSDemo.elf \
    -s -S
```

參數說明：

- `-s`：啟動 GDB Server（port 1234）
- `-S`：啟動時暫停，等待 GDB 連接

**啟動 GDB**：

```bash
riscv64-unknown-elf-gdb build/RTOSDemo.elf
```

**GDB 指令**：

```gdb
(gdb) target remote :1234
(gdb) break main
(gdb) continue
(gdb) info threads
(gdb) backtrace
```

---

## 五、第一個 FreeRTOS 程式

讓我們寫一個簡單的 FreeRTOS 程式，建立兩個 Task。

**程式碼**：

```c
#include <FreeRTOS.h>
#include <task.h>
#include <stdio.h>

// Task 1：每秒印出訊息
void vTask1(void *pvParameters) {
    int count = 0;
    while (1) {
        printf("Task 1: count = %d\n", count++);
        vTaskDelay(pdMS_TO_TICKS(1000));  // 延遲 1 秒
    }
}

// Task 2：每 500 毫秒印出訊息
void vTask2(void *pvParameters) {
    int count = 0;
    while (1) {
        printf("Task 2: count = %d\n", count++);
        vTaskDelay(pdMS_TO_TICKS(500));  // 延遲 500 毫秒
    }
}

int main(void) {
    // 建立 Task 1（優先權 1）
    xTaskCreate(vTask1, "Task 1", 256, NULL, 1, NULL);

    // 建立 Task 2（優先權 1）
    xTaskCreate(vTask2, "Task 2", 256, NULL, 1, NULL);

    // 啟動 Scheduler
    vTaskStartScheduler();

    // 不應該執行到這裡
    while (1);

    return 0;
}
```

**執行結果**：

```
Task 1: count = 0
Task 2: count = 0
Task 2: count = 1
Task 1: count = 1
Task 2: count = 2
Task 2: count = 3
Task 1: count = 2
...
```

**觀察**：

- Task 2 執行頻率是 Task 1 的兩倍（500 ms vs. 1000 ms）
- 兩個 Task 的優先權相同，所以使用 Round Robin 排程
- `vTaskDelay()` 會讓 Task 進入 Blocked 狀態，讓其他 Task 執行

---

## 六、RTOS 的應用場景

RTOS 適合哪些應用？讓我們看幾個真實案例。

### 6.1 馬達控制

**需求**：

- PWM 訊號必須每 1 毫秒更新
- 延遲超過 5 毫秒會導致馬達抖動

**RTOS 的優勢**：

- 保證最高優先權 Task 在 Deadline 前執行
- Context Switch 時間 < 5 微秒
- 中斷延遲 < 10 微秒

### 6.2 飛行控制

**需求**：

- 姿態控制迴路必須每 10 毫秒執行
- 感測器資料必須在 1 毫秒內處理

**RTOS 的優勢**：

- 確定性行為，保證回應時間
- Priority-Based Scheduling，關鍵 Task 優先執行

### 6.3 醫療設備

**需求**：

- 心跳監測必須每 100 毫秒更新
- 警報必須在 50 毫秒內觸發

**RTOS 的優勢**：

- 符合醫療設備的即時性要求
- 可預測的行為，通過安全認證

### 6.4 工業自動化

**需求**：

- PLC 控制迴路必須每 10 毫秒執行
- 多個感測器同時讀取

**RTOS 的優勢**：

- 多 Task 並發處理（透過優先權排程）
- 確定性排程，保證控制精度

---

## 七、RTOS 的挑戰

RTOS 雖然強大，但也有挑戰：

### 7.1 記憶體限制

**問題**：

- 每個 Task 需要自己的 Stack
- 嵌入式系統的 RAM 通常很小（幾 KB 到幾百 KB）

**解決方案**：

- 仔細設計 Stack 大小
- 使用 Static Allocation 而非 Dynamic Allocation
- 監控 Stack Usage

### 7.2 優先權反轉 (Priority Inversion)

**問題**：

- 高優先權 Task 等待低優先權 Task 釋放資源
- 中優先權 Task 搶佔低優先權 Task
- 結果：高優先權 Task 被中優先權 Task 間接阻塞

**解決方案**：

- 使用 Mutex 的 Priority Inheritance 機制
- 避免在 Critical Section 中執行耗時操作

### 7.3 Deadlock（死鎖）

**問題**：

- Task A 等待 Task B 釋放資源
- Task B 等待 Task A 釋放資源
- 結果：兩個 Task 都卡住

**解決方案**：

- 統一資源取得順序
- 使用 Timeout 機制
- 仔細設計同步邏輯

### 7.4 除錯困難

**問題**：

- 多 Task 並發執行（透過時間切片快速切換），問題難以重現
- Race Condition 難以偵測

**解決方案**：

- 使用 QEMU + GDB 進行除錯
- 使用 FreeRTOS 的 Trace 功能
- 仔細設計測試案例

---

## 八、總結

本文介紹了 RTOS 的核心概念，並用 FreeRTOS 作為範例。讓我們回顧重點：

### 核心概念

1. **RTOS vs. GPOS**：
   - RTOS 強調「可預測性」，而非「平均速度」
   - RTOS 保證最壞情況下的回應時間
   - GPOS 優化平均性能和吞吐量

2. **FreeRTOS 核心組件**：
   - **Task**：基本執行單位
   - **Scheduler**：Priority-Based Preemptive Scheduling
   - **Queue**：Task 之間的資料傳遞
   - **Semaphore**：同步機制
   - **Mutex**：互斥鎖，保護共享資源

3. **QEMU + RISC-V**：
   - 完全開源的學習平台
   - 支援 GDB 除錯
   - 適合深入理解 RTOS 內部機制

### 實務建議

1. **選擇合適的作業系統**：
   - 需要即時性？選 RTOS
   - 需要豐富功能？選 GPOS
   - 需要極致性能？考慮 Bare-Metal

2. **仔細設計 Task 優先權**：
   - 關鍵 Task 給高優先權
   - 避免優先權反轉
   - 使用 Mutex 的 Priority Inheritance

3. **監控系統資源**：
   - Stack Usage
   - CPU Utilization
   - Memory Fragmentation

4. **充分測試**：
   - 高負載測試
   - 長時間運行測試
   - 邊界條件測試

### 下一篇預告

在下一篇文章中，我們將深入探討 **FreeRTOS Scheduler 的實作細節**：

- Ready List 的資料結構設計
- Priority-Based Scheduling 的演算法
- Context Switch 的完整流程
- Tick Interrupt 如何觸發排程

敬請期待！

---

## 參考資料

**官方文檔**：

- [FreeRTOS Official Documentation](https://www.freertos.org/Documentation/RTOS_book.html)
- [RISC-V ISA Specification](https://riscv.org/technical/specifications/)
- [QEMU Documentation](https://www.qemu.org/docs/master/)

**開源專案**：

- [FreeRTOS GitHub Repository](https://github.com/FreeRTOS/FreeRTOS)
- [FreeRTOS RISC-V Port](https://github.com/FreeRTOS/FreeRTOS-Kernel/tree/main/portable/GCC/RISC-V)

**學術論文**：

- "Real-Time Systems: Design Principles for Distributed Embedded Applications" by Hermann Kopetz
- "The Art of Multiprocessor Programming" by Maurice Herlihy and Nir Shavit

**技術文章**：

- [Understanding RTOS](https://www.embedded.com/understanding-rtos/)
- [FreeRTOS Tutorial](https://www.digikey.com/en/maker/projects/introduction-to-rtos)

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
