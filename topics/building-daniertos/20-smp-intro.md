# 分身有術：SMP 多核心的挑戰

**作者**: Danny Jiang
**日期**: 2025-12-14

---

## 前言：當一個核心不夠用

在 v0.x 和 v1.x 的旅程中，我們的 danieRTOS 一直是「單打獨鬥」——一個 CPU 核心負責所有工作。Scheduler 挑選最高優先權的 Task，執行它，處理中斷，再繼續。簡單、優雅、可預測。

但現實世界的需求從不簡單。

想像你正在開發一個 **SSD Controller**。這個小小的晶片需要同時處理：

- **Front-End**：NVMe 命令解析，Host I/O 佇列管理
- **Back-End**：Flash 讀寫，ECC 編解碼，FTL 邏輯
- **Garbage Collection**：Block 回收，Wear Leveling
- **Misc**：溫度監控，電源管理，錯誤日誌

每個任務都有嚴格的時間要求。Front-End 需要在微秒內回應 Host；Back-End 需要持續推送命令給 Flash；GC 需要在背景默默執行，不能干擾前台性能。

單核心？忙不過來。

```
單核心的困境：
┌────────────────────────────────────────────────┐
│ Core 0                                         │
├────────────────────────────────────────────────┤
│ FE → BE → GC → FE → BE → FE → GC → ...        │
│     ↑                                          │
│     每次切換都有 overhead                       │
│     GC 被 starve，Flash 空間不足               │
│     Front-End 延遲上升，Host timeout            │
└────────────────────────────────────────────────┘
```

解決方案？**多核心 (Multi-Core)**。

但多核心不是「複製貼上」這麼簡單。當兩個核心同時存取同一個資料結構時，一切都變了。

---

## 一、多核心的恐怖故事

### 1.1 Race Condition：看不見的殺手

讓我們看一個看似無害的程式碼：

```c
// 全域計數器
volatile int counter = 0;

void task_increment(void) {
    for (int i = 0; i < 1000000; i++) {
        counter++;  // 這一行有問題嗎？
    }
}
```

在單核心上，兩個 Task 各跑一次 `task_increment`，最後 `counter` 會是 2,000,000。完美。

在雙核心上？可能是 1,500,000、1,200,000、甚至更少。

**為什麼？**

`counter++` 看起來是一行程式碼，但 CPU 實際執行三個步驟：

```asm
# counter++ 的真實面目
lw   t0, 0(a0)    # 1. 讀取 counter 到暫存器
addi t0, t0, 1    # 2. 暫存器 +1
sw   t0, 0(a0)    # 3. 寫回記憶體
```

當兩個核心同時執行時：

```
時間  Core 0                Core 1
────  ────────────────────  ────────────────────
 T0   lw t0, counter (=5)
 T1                         lw t0, counter (=5)
 T2   addi t0, t0, 1 (=6)
 T3                         addi t0, t0, 1 (=6)
 T4   sw t0, counter (=6)
 T5                         sw t0, counter (=6)
────  ────────────────────  ────────────────────
      兩次 increment，結果卻只有 +1！
```

這就是 **Race Condition**——結果取決於誰跑得快，而不是程式邏輯。

### 1.2 Lost Update：消失的資料

更糟的情況是資料結構被破壞。想像一個 Linked List：

```c
void list_insert(node_t *new_node) {
    new_node->next = head;  // 步驟 1
    head = new_node;         // 步驟 2
}
```

兩個核心同時插入：

```
時間  Core 0                    Core 1
────  ────────────────────────  ────────────────────────
 T0   new_node_A->next = head
 T1                             new_node_B->next = head
 T2   head = new_node_A
 T3                             head = new_node_B
────  ────────────────────────  ────────────────────────
      Node A 消失了！head 只指向 B
```

這不是理論問題——這是無數 SMP 系統的 Bug 來源。

---

## 二、SMP vs AMP：兩條路線

面對多核心，有兩種架構選擇：

### 2.1 AMP (Asymmetric Multi-Processing)

**AMP** 讓每個核心運行獨立的程式（甚至不同的 OS）。

```
AMP 架構：
┌─────────────┐    ┌─────────────┐
│   Core 0    │    │   Core 1    │
│  FreeRTOS   │    │  Bare-metal │
│  Front-End  │    │  Back-End   │
├─────────────┤    ├─────────────┤
│ 私有記憶體  │    │ 私有記憶體  │
└──────┬──────┘    └──────┬──────┘
       │                  │
       └──────┬───────────┘
              ▼
       ┌─────────────┐
       │ 共享記憶體  │ ← 需要特殊 IPC
       │ (Mailbox)   │
       └─────────────┘
```

**優點**：
- 簡單！每個核心獨立運作
- 不需要複雜的同步機制
- 適合異構系統（如 ARM big.LITTLE）

**缺點**：
- 負載不平衡：Core 0 忙死，Core 1 閒置
- 資源浪費：每個核心需要完整的 OS 副本
- 通訊複雜：需要額外的 IPC 機制

### 2.2 SMP (Symmetric Multi-Processing)

**SMP** 讓所有核心共享記憶體和 OS，任何 Task 可以在任何核心執行。

```
SMP 架構：
┌─────────────────────────────────────────┐
│              danieRTOS (SMP)            │
├──────────────┬──────────────────────────┤
│   Core 0     │        Core 1            │
│ Current Task │     Current Task         │
├──────────────┴──────────────────────────┤
│           共享資料結構                   │
│  ┌────────┐ ┌────────┐ ┌────────────┐   │
│  │Ready   │ │Mutex   │ │Semaphore   │   │
│  │Queue   │ │List    │ │List        │   │
│  └────────┘ └────────┘ └────────────┘   │
│                 ↑                        │
│           Spinlock 保護                  │
└─────────────────────────────────────────┘
```

**優點**：
- 自動負載平衡：Task 自動分配到閒置核心
- 資源共享：一份 OS，一份資料結構
- 簡化程式設計：Task 不需要知道自己在哪個核心

**缺點**：
- 同步複雜：所有共享資料需要保護
- Cache 問題：Cache Coherency 開銷
- Debug 困難：Race Condition 難以重現

### 2.3 danieRTOS v2.x 的選擇

對於 SSD Controller 這樣的應用，我們選擇 **SMP**：

1. **Task 需要動態調度**：GC 在背景執行，但 Host 忙時要讓出 CPU
2. **資源有限**：嵌入式系統記憶體寶貴，不能浪費
3. **學習價值**：SMP 的同步問題是系統程式設計的核心知識

---

## 三、SMP 的五大挑戰

### 3.1 挑戰 #1：多核心啟動

QEMU virt 有個特性：**所有核心同時開機**。

不像 x86 有 BSP/AP 的概念（Bootstrap Processor 先啟動，再喚醒 Application Processors），RISC-V QEMU 在 `-bios none` 模式下，所有 hart 同時從 `0x80000000` 開始執行。

這意味著我們的 `startup.S` 必須處理這個情況——否則多個核心會同時清除 BSS、同時設定 Stack、同時呼叫 `main()`。

**解決方案**：使用 `mhartid` 判斷核心 ID，讓 Hart 0 先初始化，其他核心 spin-wait。

### 3.2 挑戰 #2：共享資料保護

在 v0.x，我們用 **中斷禁用** 保護 Critical Section：

```c
void critical_section(void) {
    uint32_t mstatus = disable_interrupts();
    // 安全的操作共享資料
    restore_interrupts(mstatus);
}
```

這在單核心有效，但多核心無效——你禁用了 Core 0 的中斷，Core 1 仍然可以存取共享資料！

**解決方案**：**Spinlock**——一種使用原子指令實作的鎖。

### 3.3 挑戰 #3：Cache Coherency

現代 CPU 有多層 Cache。當 Core 0 寫入資料時，Core 1 的 Cache 可能還是舊值：

```
Core 0 Cache: data = 42  (剛寫入)
Core 1 Cache: data = 0   (舊值)
Memory:       data = 42  (已更新)
```

如果沒有 **Cache Coherency Protocol**（如 MESI），Core 1 會讀到錯誤的值。

**解決方案**：RISC-V 使用 **fence** 指令和 **Acquire/Release** 語意確保記憶體順序。

### 3.4 挑戰 #4：核心間通訊

當 Core 0 喚醒了一個高優先權 Task，但 Core 1 正在執行低優先權 Task，我們需要通知 Core 1「該 reschedule 了！」

**解決方案**：**IPI (Inter-Processor Interrupt)**——一個核心可以向另一個核心發送中斷。

### 3.5 挑戰 #5：Per-Core 資料

有些資料是每個核心獨有的：

- `current_task`：目前執行的 Task
- `irq_nest_depth`：中斷巢狀深度
- `idle_task`：該核心的 Idle Task

**解決方案**：使用 **tp 暫存器** 指向 Per-Core 資料結構。

---

## 四、v2.x 架構預覽

讓我們預覽 danieRTOS v2.x 的架構：

```
┌─────────────────────────────────────────────────────────────┐
│                    danieRTOS v2.x (SMP)                     │
├──────────────────────────┬──────────────────────────────────┤
│         Core 0           │           Core 1                 │
│  ┌────────────────────┐  │  ┌────────────────────────────┐  │
│  │ Per-Core Data      │  │  │ Per-Core Data              │  │
│  │ • current_task     │  │  │ • current_task             │  │
│  │ • idle_task        │  │  │ • idle_task                │  │
│  │ • irq_nest_depth   │  │  │ • irq_nest_depth           │  │
│  └────────────────────┘  │  └────────────────────────────┘  │
│           │              │              │                   │
│           ▼              │              ▼                   │
│  ┌────────────────────┐  │  ┌────────────────────────────┐  │
│  │ Local Timer (CLINT)│  │  │ Local Timer (CLINT)        │  │
│  └────────────────────┘  │  └────────────────────────────┘  │
├──────────────────────────┴──────────────────────────────────┤
│                     IPI (CLINT MSIP)                        │
├─────────────────────────────────────────────────────────────┤
│                   Shared Data (Spinlock Protected)          │
│  ┌──────────────┐ ┌──────────────┐ ┌────────────────────┐   │
│  │ Global Ready │ │  Semaphore   │ │      Queue         │   │
│  │    Queue     │ │    List      │ │      Pool          │   │
│  └──────────────┘ └──────────────┘ └────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 4.1 新增元件

| 元件 | 用途 |
|------|------|
| **Per-Core Data** | 每個核心的私有資料 (tp 暫存器指向) |
| **Spinlock** | 保護共享資料結構 |
| **IPI** | 核心間通訊 (使用 CLINT MSIP) |
| **SMP Scheduler** | 支援多核心排程 |
| **SMP-Safe IPC** | Semaphore/Mutex/Queue 的 SMP 版本 |

### 4.2 修改元件

| 元件 | v0.x | v2.x |
|------|------|------|
| **startup.S** | 單核心初始化 | Per-Core 啟動流程 |
| **scheduler.c** | 單一 current_task | Per-Core current_task |
| **critical section** | 中斷禁用 | Spinlock + 中斷禁用 |
| **tick.c** | 單一 Timer | Per-Core Timer |

---

## 五、本章回顧

在這一章，我們認識了多核心程式設計的挑戰：

1. **Race Condition**：多核心同時存取共享資料會導致不可預期的結果
2. **SMP vs AMP**：SMP 共享 OS 和記憶體，AMP 各自獨立
3. **五大挑戰**：Boot、Spinlock、Cache、IPI、Per-Core Data

接下來的章節，我們將逐一攻克這些挑戰：

- **Ch 21**：多核心啟動流程
- **Ch 22**：Per-Core 資料結構設計
- **Ch 23**：Spinlock 實作
- **Ch 24**：Cache Coherency
- **Ch 25**：IPI 實作
- **Ch 26**：SMP Scheduler
- **Ch 27**：SMP-Safe IPC
- **Ch 28**：Lock-Free Queue
- **Ch 29**：整合與驗證

準備好了嗎？讓我們開始這趟 SMP 之旅！

---

## 參考資料

- **See RISC-V Run**, Ch 10 - Multi-core Boot
- **Data Structures in Practice**, Ch 13 - Concurrent Data Structures
- [FreeRTOS SMP](https://www.freertos.org/symmetric-multiprocessing-introduction.html)
- [Linux Kernel SMP](https://www.kernel.org/doc/html/latest/core-api/smp.html)

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。
