# RTOS 除錯實戰：GDB + QEMU 的完整工作流程

**作者**: Danny Jiang
**日期**: 2025-12-09

---

## 前言：一個讓人抓狂的 Heisenbug

2019 年，我在一家工業自動化公司擔任韌體工程師。有一天，測試團隊報告了一個詭異的問題：

**工業控制器在運行 2-3 小時後會隨機 Hard Fault，導致系統重啟。但只要我們連接 JTAG 除錯器，問題就消失了。**

我們的產品是一個工業控制器，運行 FreeRTOS，需要同時處理溫度、壓力、流量三個感測器的讀取，執行 PID 控制演算法，透過 Modbus RTU 與上位機通訊，還要更新 LCD 顯示。總共 6 個 Tasks 在系統中運行。

我花了整整一週時間在實體硬體上除錯，但問題就是無法重現。連接 JTAG 後，系統運行正常；加入 `printf` 除錯訊息後，問題消失；問題只在「正常運行」時出現。

這是典型的 **Heisenbug（觀察者效應）**：一旦你試圖觀察它，它就消失了。

後來我的同事建議：「為什麼不試試 QEMU？在虛擬環境中，你可以隨時暫停、檢查、回溯，而不會改變系統的時序。」

這個建議改變了一切。

最後，我們使用 **QEMU + GDB** 在虛擬環境中重現問題：

1. **在 QEMU 中運行相同的程式碼**
2. **使用 GDB 設定條件中斷點**（當 Stack 使用超過 80% 時中斷）
3. **單步執行並檢視 Task 狀態**

結果：我們發現 **控制 Task 的 Stack 不夠大**，在某些情況下會 Stack Overflow，覆蓋其他 Task 的記憶體。

**修正**：將控制 Task 的 Stack 從 256 words 增加到 512 words。

**結果**：系統穩定運行超過 6 個月，沒有任何 Hard Fault。

---

## 一、為什麼使用 QEMU + GDB？

### 1.1 實體硬體除錯的挑戰

**挑戰 #1：觀察者效應（Heisenbug）**

- 連接 JTAG 會改變時序
- 加入 `printf` 會改變記憶體佈局
- 問題無法重現

**挑戰 #2：硬體限制**

- 有些 MCU 的 JTAG 功能有限
- 有些 MCU 沒有 Trace 功能
- 硬體中斷點數量有限（通常只有 2-4 個）

**挑戰 #3：開發效率**

- 每次修改都需要燒錄
- 燒錄時間長（10-30 秒）
- 無法快速迭代

### 1.2 QEMU + GDB 的優勢

**優勢 #1：完全可控的環境**

- 確定性執行（每次運行結果相同）
- 可以暫停、單步執行、回溯
- 不會改變時序

**優勢 #2：強大的除錯功能**

- 無限的中斷點
- 條件中斷點（例如：當變數 > 100 時中斷）
- Watchpoint（當記憶體被修改時中斷）
- 可以檢視所有暫存器、記憶體、CSRs

**優勢 #3：快速迭代**

- 編譯後立即執行（< 1 秒）
- 不需要燒錄
- 可以快速測試不同的修正方案

**優勢 #4：學習工具**

- 可以深入理解 RTOS 的運作機制
- 可以看到 Context Switch 的完整流程
- 可以分析效能（Context Switch Overhead）

### 1.3 QEMU 的限制

**限制 #1：不是 Cycle-Accurate**

- QEMU 的時序與實體硬體不同
- 無法精確測量效能

**限制 #2：簡化的硬體模型**

- Cache 模型簡化
- 沒有真實的 I/O 設備

**限制 #3：不能完全取代實體硬體**

- 最終還是需要在實體硬體上驗證

**最佳實踐**：

- **功能除錯**：使用 QEMU + GDB（快速、方便）
- **效能驗證**：使用實體硬體（精確、真實）

---

## 二、QEMU 啟動參數與 GDB Server

### 2.1 基本的 QEMU 啟動

**最簡單的啟動方式**：

```bash
qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -bios none \
    -kernel build/RTOSDemo.elf
```

**參數說明**：

- `-machine virt`：使用 virt machine（通用虛擬機）
- `-nographic`：不使用圖形介面（使用終端機）
- `-bios none`：不載入 BIOS（直接執行 kernel）
- `-kernel build/RTOSDemo.elf`：載入 ELF 檔案

**退出 QEMU**：按 `Ctrl-A` 然後按 `X`

### 2.2 啟動 GDB Server

**啟動 QEMU 並等待 GDB 連接**：

```bash
qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -bios none \
    -kernel build/RTOSDemo.elf \
    -s \
    -S
```

**新增參數說明**：

- `-s`：啟動 GDB Server（監聽 port 1234）
- `-S`：啟動時暫停 CPU（等待 GDB 連接）

**這兩個參數的組合非常重要**：

- `-s`：讓 GDB 可以連接
- `-S`：讓你可以在程式開始執行前設定中斷點

### 2.3 連接 GDB

**啟動 GDB**：

```bash
riscv64-unknown-elf-gdb build/RTOSDemo.elf
```

**在 GDB 中連接到 QEMU**：

```gdb
(gdb) target remote :1234
Remote debugging using :1234
0x0000000000001000 in ?? ()
```

**設定中斷點並開始執行**：

```gdb
(gdb) break main
Breakpoint 1 at 0x80000234: file main.c, line 42.

(gdb) continue
Continuing.

Breakpoint 1, main () at main.c:42
42      int main(void) {
```

**恭喜！你已經成功連接 GDB 並在 main 函式設定中斷點。**

### 2.4 常用的 QEMU 參數

**參數 #1：指定 CPU 核心數量**

```bash
-smp 2  # 2 個 CPU 核心（用於測試 SMP）
```

**參數 #2：指定記憶體大小**

```bash
-m 128M  # 128 MB RAM
```

**參數 #3：啟用 Trace**

```bash
-d in_asm -D trace.log  # 記錄所有執行的指令到 trace.log
```

**參數 #4：重定向 UART 輸出**

```bash
-serial stdio  # UART 輸出到終端機
-serial file:uart.log  # UART 輸出到檔案
```

**完整範例**：

```bash
qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -bios none \
    -kernel build/RTOSDemo.elf \
    -m 128M \
    -smp 1 \
    -serial stdio \
    -s -S
```

---

## 三、GDB 基本指令

### 3.1 中斷點（Breakpoints）

**設定中斷點**：

```gdb
# 在函式設定中斷點
(gdb) break main
(gdb) break vTaskStartScheduler
(gdb) break vTaskSwitchContext

# 在特定行設定中斷點
(gdb) break main.c:42

# 在特定地址設定中斷點
(gdb) break *0x80000234
```

**列出所有中斷點**：

```gdb
(gdb) info breakpoints
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x0000000080000234 in main at main.c:42
2       breakpoint     keep y   0x0000000080001000 in vTaskStartScheduler
```

**刪除中斷點**：

```gdb
(gdb) delete 1  # 刪除中斷點 #1
(gdb) delete    # 刪除所有中斷點
```

**暫時停用中斷點**：

```gdb
(gdb) disable 1  # 停用中斷點 #1
(gdb) enable 1   # 啟用中斷點 #1
```

### 3.2 執行控制

**繼續執行**：

```gdb
(gdb) continue  # 繼續執行直到下一個中斷點
(gdb) c         # 簡寫
```

**單步執行（Step Over）**：

```gdb
(gdb) next  # 執行下一行（不進入函式）
(gdb) n     # 簡寫
```

**單步執行（Step Into）**：

```gdb
(gdb) step  # 執行下一行（進入函式）
(gdb) s     # 簡寫
```

**執行到函式返回**：

```gdb
(gdb) finish  # 執行到當前函式返回
```

**執行到特定行**：

```gdb
(gdb) until 50  # 執行到第 50 行
```

### 3.3 檢視變數

**列印變數**：

```gdb
(gdb) print count
$1 = 42

(gdb) print pxCurrentTCB
$2 = (TCB_t *) 0x80010000

(gdb) print *pxCurrentTCB
$3 = {
  pxTopOfStack = 0x80011000,
  xStateListItem = {...},
  uxPriority = 2,
  ...
}
```

**列印陣列**：

```gdb
(gdb) print pxReadyTasksLists[2]
(gdb) print pxReadyTasksLists[2]@5  # 列印 5 個元素
```

**列印字串**：

```gdb
(gdb) print (char *)pcTaskName
$4 = 0x80012000 "Task1"
```

**以不同格式列印**：

```gdb
(gdb) print/x count  # 十六進位
(gdb) print/d count  # 十進位
(gdb) print/t count  # 二進位
(gdb) print/c count  # 字元
```

### 3.4 檢視記憶體

**檢視記憶體內容**：

```gdb
# x/[數量][格式][大小] [地址]
# 格式：x=十六進位, d=十進位, s=字串, i=指令
# 大小：b=byte, h=halfword, w=word, g=giant(8 bytes)

(gdb) x/10x 0x80000000  # 檢視 10 個 word（十六進位）
(gdb) x/10i $pc         # 檢視 10 條指令（從 PC 開始）
(gdb) x/s 0x80012000    # 檢視字串
```

**檢視 Stack**：

```gdb
(gdb) x/32x $sp  # 檢視 Stack 的前 32 個 word
```

**範例**：

```gdb
(gdb) x/10i $pc
=> 0x80000234 <main+0>:     addi    sp,sp,-32
   0x80000236 <main+2>:     sd      ra,24(sp)
   0x80000238 <main+4>:     sd      s0,16(sp)
   0x8000023a <main+6>:     addi    s0,sp,32
   0x8000023c <main+8>:     li      a0,256
   0x80000240 <main+12>:    li      a1,1
   0x80000244 <main+16>:    call    0x80001000 <xTaskCreate>
   ...
```

### 3.5 檢視暫存器

**檢視所有通用暫存器**：

```gdb
(gdb) info registers
ra             0x80000100       0x80000100
sp             0x80010000       0x80010000
gp             0x80020000       0x80020000
tp             0x0              0x0
t0             0x1              1
t1             0x2              2
...
```

**檢視特定暫存器**：

```gdb
(gdb) print $pc
$1 = (void (*)()) 0x80000234 <main>

(gdb) print $sp
$2 = (void *) 0x80010000

(gdb) print $ra
$3 = (void (*)()) 0x80000100
```

**檢視 RISC-V CSRs**：

```gdb
(gdb) info all-registers  # 檢視所有暫存器（包含 CSRs）

(gdb) print $mstatus
$4 = 0x1800

(gdb) print $mepc
$5 = 0x80000234

(gdb) print $mcause
$6 = 0x8000000000000007  # Timer interrupt
```

### 3.6 檢視 Call Stack

**檢視 Call Stack**：

```gdb
(gdb) backtrace
#0  vTaskDelay (xTicksToDelay=1000) at tasks.c:1234
#1  0x80000500 in vTask1 (pvParameters=0x0) at main.c:42
#2  0x80001000 in pxPortInitialiseStack () at port.c:123
```

**檢視特定 Frame 的變數**：

```gdb
(gdb) frame 1  # 切換到 Frame #1
(gdb) info locals  # 檢視區域變數
count = 42
data = 0x80012000
```

---

## 四、除錯 FreeRTOS Tasks

### 4.1 檢視所有 Tasks

FreeRTOS 沒有內建的 GDB 指令來列出所有 Tasks，但我們可以手動檢視。

**方法 #1：檢視 Ready List**

```gdb
# 檢視優先權 2 的 Ready List
(gdb) print pxReadyTasksLists[2]
$1 = {
  uxNumberOfItems = 2,
  pxIndex = 0x80010100,
  xListEnd = {
    xItemValue = 0xffffffff,
    pxNext = 0x80010200,
    pxPrevious = 0x80010300,
    pvOwner = 0x0,
    pvContainer = 0x80010000
  }
}

# 遍歷 List 中的所有 Tasks
(gdb) print *(TCB_t *)pxReadyTasksLists[2].xListEnd.pxNext->pvOwner
$2 = {
  pxTopOfStack = 0x80011000,
  xStateListItem = {...},
  uxPriority = 2,
  pcTaskName = "Task1",
  ...
}
```

**方法 #2：使用 FreeRTOS 的 Run Time Stats**

如果你啟用了 `configGENERATE_RUN_TIME_STATS`，可以在程式中呼叫 `vTaskGetRunTimeStats()` 來列出所有 Tasks。

```c
// 在程式中加入
char stats_buffer[1024];
vTaskGetRunTimeStats(stats_buffer);
printf("%s\n", stats_buffer);
```

**輸出範例**：

```
Task            Abs Time        % Time
Task1           12345           45%
Task2           8901            32%
Idle            6234            23%
```

### 4.2 檢視當前 Task

**檢視 pxCurrentTCB**：

```gdb
(gdb) print pxCurrentTCB
$1 = (TCB_t *) 0x80010000

(gdb) print *pxCurrentTCB
$2 = {
  pxTopOfStack = 0x80011000,
  xStateListItem = {
    xItemValue = 0,
    pxNext = 0x80010100,
    pxPrevious = 0x80010200,
    pvOwner = 0x80010000,
    pvContainer = 0x80010300
  },
  uxPriority = 2,
  pxStack = 0x80010800,
  pcTaskName = "Task1",
  ...
}
```

**檢視 Task 名稱**：

```gdb
(gdb) print pxCurrentTCB->pcTaskName
$3 = "Task1"
```

**檢視 Task 優先權**：

```gdb
(gdb) print pxCurrentTCB->uxPriority
$4 = 2
```

**檢視 Task Stack Pointer**：

```gdb
(gdb) print pxCurrentTCB->pxTopOfStack
$5 = (StackType_t *) 0x80011000
```

### 4.3 檢視 Task Stack

**檢視 Stack 內容**：

```gdb
(gdb) x/32x pxCurrentTCB->pxTopOfStack
0x80011000:     0x80000100      0x00000001      0x00000002      0x00000003
0x80011010:     0x00000004      0x00000005      0x00000006      0x00000007
...
```

**檢視 Stack 使用量**：

```gdb
# 計算 Stack 使用量
(gdb) print pxCurrentTCB->pxStack
$6 = (StackType_t *) 0x80010800

(gdb) print pxCurrentTCB->pxTopOfStack
$7 = (StackType_t *) 0x80011000

# Stack 使用量 = pxTopOfStack - pxStack
(gdb) print (int)($7 - $6)
$8 = 512  # 使用了 512 words
```

**使用 FreeRTOS API 檢視 Stack High Water Mark**：

在程式中加入：

```c
UBaseType_t watermark = uxTaskGetStackHighWaterMark(NULL);
printf("Stack high water mark: %u words\n", watermark);
```

### 4.4 切換到不同的 Task

**問題**：GDB 只能看到當前執行的 Task 的 Stack，如何檢視其他 Task 的 Stack？

**解決方案**：手動切換 Stack Pointer

```gdb
# 1. 保存當前的 SP
(gdb) set $saved_sp = $sp

# 2. 切換到 Task1 的 Stack
(gdb) set $sp = pxReadyTasksLists[2].xListEnd.pxNext->pvOwner->pxTopOfStack

# 3. 檢視 Task1 的 Call Stack
(gdb) backtrace

# 4. 恢復原本的 SP
(gdb) set $sp = $saved_sp
```

**注意**：這個方法只能檢視 Stack，無法真正切換 Task 的執行。

---

## 五、除錯 Context Switch

### 5.1 在 Context Switch 設定中斷點

**設定中斷點**：

```gdb
(gdb) break vTaskSwitchContext
(gdb) break vPortYield
(gdb) break xPortStartScheduler
```

**執行並觀察**：

```gdb
(gdb) continue
Breakpoint 1, vTaskSwitchContext () at tasks.c:2345

(gdb) print pxCurrentTCB->pcTaskName
$1 = "Task1"

(gdb) next
(gdb) next
...

(gdb) print pxCurrentTCB->pcTaskName
$2 = "Task2"  # Task 已經切換了！
```

### 5.2 檢視 Context Switch 的完整流程

**在組合語言層級設定中斷點**：

```gdb
(gdb) break vPortYield
(gdb) continue

Breakpoint 1, vPortYield () at portASM.S:42
42      addi sp, sp, -32*4

(gdb) disassemble
Dump of assembler code for function vPortYield:
=> 0x80001000 <+0>:     addi    sp,sp,-128
   0x80001004 <+4>:     sd      ra,0(sp)
   0x80001008 <+8>:     sd      t0,8(sp)
   ...
```

**單步執行組合語言**：

```gdb
(gdb) stepi  # 執行一條指令
(gdb) si     # 簡寫

(gdb) info registers  # 檢視暫存器變化
```

**觀察 Stack 的變化**：

```gdb
# 在保存暫存器前
(gdb) x/10x $sp
0x80011000:     0x00000000      0x00000000      ...

# 執行 sd ra, 0(sp)
(gdb) stepi

# 在保存暫存器後
(gdb) x/10x $sp
0x80011000:     0x80000100      0x00000000      ...  # ra 已保存
```

### 5.3 測量 Context Switch Overhead

**方法 #1：使用 QEMU 的 Cycle Counter**

QEMU 可以提供指令計數（不是真正的 Cycle，但可以作為參考）。

```bash
# 啟動 QEMU 並啟用 icount
qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -bios none \
    -kernel build/RTOSDemo.elf \
    -icount shift=0 \
    -s -S
```

**在 GDB 中測量**：

```gdb
# 在 vPortYield 開始設定中斷點
(gdb) break vPortYield
(gdb) continue

# 記錄開始的指令計數
(gdb) monitor info jit
...

# 單步執行到 vPortYield 結束
(gdb) finish

# 記錄結束的指令計數
(gdb) monitor info jit
...

# 計算差值
```

**方法 #2：使用 FreeRTOS 的 Run Time Stats**

在 `FreeRTOSConfig.h` 中啟用：

```c
#define configGENERATE_RUN_TIME_STATS  1
#define portCONFIGURE_TIMER_FOR_RUN_TIME_STATS()  /* 配置 Timer */
#define portGET_RUN_TIME_COUNTER_VALUE()  /* 讀取 Timer */
```

然後在程式中呼叫：

```c
char stats_buffer[1024];
vTaskGetRunTimeStats(stats_buffer);
printf("%s\n", stats_buffer);
```

---

## 六、除錯中斷處理

### 6.1 在 ISR 設定中斷點

**設定中斷點**：

```gdb
(gdb) break handle_m_timer_interrupt
(gdb) break UART_IRQHandler
```

**執行並觀察**：

```gdb
(gdb) continue
Breakpoint 1, handle_m_timer_interrupt () at interrupt.c:42

(gdb) print xTickCount
$1 = 1000

(gdb) next
(gdb) next
...

(gdb) print xTickCount
$2 = 1001  # Tick 已經增加了
```

### 6.2 檢視中斷相關的 CSRs

**檢視 mstatus**：

```gdb
(gdb) print/x $mstatus
$1 = 0x1800

# 解析 mstatus
# Bit 3 (MIE): 0 = 中斷關閉, 1 = 中斷開啟
# Bit 7 (MPIE): 進入 Trap 前的 MIE 狀態
```

**檢視 mcause**：

```gdb
(gdb) print/x $mcause
$2 = 0x8000000000000007

# 解析 mcause
# Bit 63: 1 = Interrupt, 0 = Exception
# Bits 0-62: Exception Code
#   7 = Machine Timer Interrupt
#   11 = Machine External Interrupt
```

**檢視 mepc**：

```gdb
(gdb) print/x $mepc
$3 = 0x80000234  # 中斷發生時的 PC
```

**檢視 mie 和 mip**：

```gdb
(gdb) print/x $mie
$4 = 0x888  # 哪些中斷被啟用

(gdb) print/x $mip
$5 = 0x080  # 哪些中斷正在 Pending
```

### 6.3 模擬中斷觸發

**在 QEMU 中，你可以手動觸發中斷**：

```gdb
# 觸發 Timer Interrupt
(gdb) monitor system_reset
(gdb) monitor sendkey ctrl-alt-f1
```

**或者，修改 mtime 和 mtimecmp**：

```gdb
# 讀取 mtime
(gdb) x/1xg 0x0200BFF8
0x0200bff8:     0x0000000000001000

# 讀取 mtimecmp
(gdb) x/1xg 0x02004000
0x02004000:     0x0000000000002000

# 修改 mtime 使其超過 mtimecmp（觸發中斷）
(gdb) set {long}0x0200BFF8 = 0x2001

# 繼續執行，中斷會被觸發
(gdb) continue
```

---

## 七、常見問題除錯

### 7.1 Stack Overflow

**症狀**：

- 系統隨機 Hard Fault
- 變數值被覆蓋
- Task 行為異常

**除錯方法**：

**方法 #1：啟用 FreeRTOS 的 Stack Overflow 檢測**

```c
// FreeRTOSConfig.h
#define configCHECK_FOR_STACK_OVERFLOW  2

// main.c
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName) {
    printf("ERROR: Stack overflow in task %s!\n", pcTaskName);
    while (1);  // 在這裡設定中斷點
}
```

**方法 #2：使用 GDB 檢查 Stack 使用量**

```gdb
# 檢視 Task 的 Stack
(gdb) print pxCurrentTCB->pxStack
$1 = (StackType_t *) 0x80010800

(gdb) print pxCurrentTCB->pxTopOfStack
$2 = (StackType_t *) 0x80011000

# 計算使用量
(gdb) print (int)($2 - $1)
$3 = 512  # 使用了 512 words

# 檢查是否接近 Stack 底部
(gdb) print pxCurrentTCB->pxStack[0]
$4 = 0xa5a5a5a5  # 如果不是這個值，表示 Stack 已經 Overflow
```

**方法 #3：設定 Watchpoint**

```gdb
# 監視 Stack 底部的記憶體
(gdb) watch *(int *)0x80010800

# 當這個記憶體被修改時，GDB 會中斷
```

### 7.2 Deadlock

**症狀**：

- 系統停止回應
- 所有 Task 都在等待

**除錯方法**：

**方法 #1：檢視所有 Task 的狀態**

```gdb
# 暫停執行
(gdb) Ctrl-C

# 檢視當前 Task
(gdb) print pxCurrentTCB->pcTaskName
$1 = "Idle"  # 如果是 Idle Task，表示所有 Task 都在等待

# 檢視 Blocked List
(gdb) print xSuspendedTaskList
(gdb) print xDelayedTaskList1
(gdb) print xDelayedTaskList2
```

**方法 #2：檢視 Mutex/Semaphore 的狀態**

```gdb
# 檢視 Mutex
(gdb) print my_mutex
$2 = {
  uxMessagesWaiting = 0,  # 0 = Locked
  xTasksWaitingToReceive = {
    uxNumberOfItems = 2,  # 有 2 個 Task 在等待
    ...
  }
}

# 檢視誰持有 Mutex
(gdb) print my_mutex.pxMutexHolder
$3 = (TCB_t *) 0x80010000

(gdb) print my_mutex.pxMutexHolder->pcTaskName
$4 = "Task1"  # Task1 持有 Mutex
```

**方法 #3：設定條件中斷點**

```gdb
# 當 Task1 嘗試取得 Mutex 時中斷
(gdb) break xQueueSemaphoreTake if strcmp(pxCurrentTCB->pcTaskName, "Task1") == 0
```

### 7.3 Priority Inversion

**症狀**：

- 高優先權 Task 被低優先權 Task 阻塞
- 系統回應變慢

**除錯方法**：

**方法 #1：檢視 Mutex 的優先權繼承**

```c
// 確保使用 Mutex（支援優先權繼承）而不是 Binary Semaphore
QueueHandle_t my_mutex = xSemaphoreCreateMutex();
```

**方法 #2：使用 GDB 追蹤優先權變化**

```gdb
# 設定 Watchpoint 監視 Task 的優先權
(gdb) watch pxCurrentTCB->uxPriority

# 當優先權改變時，GDB 會中斷
```

### 7.4 記憶體洩漏

**症狀**：

- 可用記憶體持續減少
- 最終系統 Out of Memory

**除錯方法**：

**方法 #1：監控 Heap 使用量**

```gdb
# 定期檢查 Heap 使用量
(gdb) print xPortGetFreeHeapSize()
$1 = 32768

# 繼續執行一段時間
(gdb) continue
(gdb) Ctrl-C

# 再次檢查
(gdb) print xPortGetFreeHeapSize()
$2 = 28672  # 減少了 4096 bytes
```

**方法 #2：設定條件中斷點**

```gdb
# 當 Heap 使用量超過 80% 時中斷
(gdb) break pvPortMalloc if xPortGetFreeHeapSize() < 6553
```

**方法 #3：追蹤所有的 malloc/free**

在程式中加入追蹤程式碼（參考文章 #4 的記憶體洩漏檢測）。

---

## 八、進階 GDB 技巧

### 8.1 條件中斷點

**語法**：

```gdb
break [位置] if [條件]
```

**範例 #1：當變數超過某個值時中斷**

```gdb
(gdb) break main.c:42 if count > 100
```

**範例 #2：當特定 Task 執行時中斷**

```gdb
(gdb) break vTaskDelay if strcmp(pxCurrentTCB->pcTaskName, "Task1") == 0
```

**範例 #3：當 Stack 使用量超過 80% 時中斷**

```gdb
(gdb) break vTaskSwitchContext if (pxCurrentTCB->pxTopOfStack - pxCurrentTCB->pxStack) > 409
```

### 8.2 Watchpoint（監視點）

**Watchpoint** 會在記憶體被修改時中斷。

**設定 Watchpoint**：

```gdb
(gdb) watch count  # 監視變數 count
(gdb) watch *(int *)0x80010000  # 監視特定地址
```

**範例**：

```gdb
(gdb) watch pxCurrentTCB->uxPriority
Hardware watchpoint 1: pxCurrentTCB->uxPriority

(gdb) continue
Hardware watchpoint 1: pxCurrentTCB->uxPriority

Old value = 2
New value = 3
vTaskPrioritySet () at tasks.c:1234
```

### 8.3 自動化指令（Commands）

**在中斷點觸發時自動執行指令**：

```gdb
(gdb) break vTaskSwitchContext
(gdb) commands
> silent
> print pxCurrentTCB->pcTaskName
> continue
> end
```

**這樣每次 Context Switch 時，GDB 會自動列印 Task 名稱並繼續執行。**

**範例輸出**：

```
$1 = "Task1"
$2 = "Task2"
$3 = "Idle"
$4 = "Task1"
...
```

### 8.4 GDB 腳本

**建立 GDB 腳本檔案**（`debug.gdb`）：

```gdb
# 連接到 QEMU
target remote :1234

# 設定中斷點
break main
break vTaskSwitchContext

# 自動化指令
commands 2
  silent
  print pxCurrentTCB->pcTaskName
  continue
end

# 開始執行
continue
```

**使用腳本**：

```bash
riscv64-unknown-elf-gdb build/RTOSDemo.elf -x debug.gdb
```

### 8.5 Python 擴充

GDB 支援 Python 擴充，可以撰寫自訂指令。

**範例：列出所有 FreeRTOS Tasks**

建立 `freertos_gdb.py`：

```python
import gdb

class ListTasksCommand(gdb.Command):
    """List all FreeRTOS tasks"""

    def __init__(self):
        super(ListTasksCommand, self).__init__("list-tasks", gdb.COMMAND_USER)

    def invoke(self, arg, from_tty):
        # 讀取 pxReadyTasksLists
        for priority in range(0, 32):
            try:
                list_ptr = gdb.parse_and_eval(f"pxReadyTasksLists[{priority}]")
                num_items = int(list_ptr['uxNumberOfItems'])

                if num_items > 0:
                    print(f"Priority {priority}: {num_items} task(s)")

                    # 遍歷 List
                    # (這裡需要更複雜的邏輯來遍歷 FreeRTOS 的 List)
            except:
                pass

ListTasksCommand()
```

**載入 Python 腳本**：

```gdb
(gdb) source freertos_gdb.py
(gdb) list-tasks
Priority 0: 1 task(s)
Priority 1: 2 task(s)
Priority 2: 1 task(s)
```

---

## 九、實戰案例：除錯 Stack Overflow

讓我們回到 Mock Scenario，使用 GDB 除錯 Stack Overflow 問題。

### 9.1 重現問題

**步驟 1：啟動 QEMU + GDB**

```bash
# Terminal 1
qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -bios none \
    -kernel build/RTOSDemo.elf \
    -s -S

# Terminal 2
riscv64-unknown-elf-gdb build/RTOSDemo.elf
```

**步驟 2：連接並設定中斷點**

```gdb
(gdb) target remote :1234
(gdb) break main
(gdb) continue
```

**步驟 3：啟用 Stack Overflow 檢測**

在 `FreeRTOSConfig.h` 中：

```c
#define configCHECK_FOR_STACK_OVERFLOW  2
```

在 `main.c` 中：

```c
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName) {
    printf("ERROR: Stack overflow in task %s!\n", pcTaskName);
    while (1);  // 在這裡設定中斷點
}
```

**步驟 4：設定中斷點並執行**

```gdb
(gdb) break vApplicationStackOverflowHook
(gdb) continue
```

### 9.2 分析問題

**當中斷點觸發時**：

```gdb
Breakpoint 1, vApplicationStackOverflowHook (xTask=0x80010000, pcTaskName=0x80012000 "ControlTask") at main.c:123

(gdb) print pcTaskName
$1 = "ControlTask"  # 問題出在 ControlTask

(gdb) print xTask
$2 = (TaskHandle_t) 0x80010000

(gdb) print *(TCB_t *)xTask
$3 = {
  pxTopOfStack = 0x80010900,
  pxStack = 0x80010800,
  uxPriority = 2,
  pcTaskName = "ControlTask",
  ...
}

# 計算 Stack 使用量
(gdb) print (int)(0x80010900 - 0x80010800)
$4 = 256  # Stack 大小是 256 words

# 檢查 Stack 是否真的 Overflow
(gdb) x/10x 0x80010800
0x80010800:     0x12345678  0x9abcdef0  ...  # Stack 底部被覆蓋了！
```

### 9.3 找出原因

**檢視 ControlTask 的程式碼**：

```c
void vTaskControl(void *pvParameters) {
    float pid_buffer[100];  // 400 bytes（100 * 4）
    sensor_data_t data;     // 256 bytes
    // ...

    while (1) {
        // PID 控制
        calculate_pid(pid_buffer, &data);
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

**問題**：

- `pid_buffer` 需要 400 bytes
- `sensor_data_t` 需要 256 bytes
- 總共需要 656 bytes
- 但 Stack 只有 256 words（1024 bytes），扣除 Context 保存的空間，不夠用！

### 9.4 修正問題

**解決方案 #1：增加 Stack 大小**

```c
// 從 256 words 增加到 512 words
xTaskCreate(vTaskControl, "ControlTask", 512, NULL, 2, NULL);
```

**解決方案 #2：減少 Stack 使用**

```c
// 使用靜態分配或動態分配
static float pid_buffer[100];  // 移到全域變數

void vTaskControl(void *pvParameters) {
    sensor_data_t *data = pvPortMalloc(sizeof(sensor_data_t));
    // ...
}
```

**驗證修正**：

```gdb
(gdb) continue
# 系統正常運行，沒有 Stack Overflow
```

---

## 十、總結

### 10.1 核心要點

1. **QEMU + GDB 的優勢**：
   - 確定性執行、快速迭代
   - 強大的除錯功能（無限中斷點、條件中斷點、Watchpoint）
   - 可以深入理解 RTOS 的運作機制

2. **QEMU 啟動參數**：
   - `-s`：啟動 GDB Server（port 1234）
   - `-S`：啟動時暫停 CPU
   - `-m`：指定記憶體大小
   - `-smp`：指定 CPU 核心數量

3. **GDB 基本指令**：
   - 中斷點：`break`, `delete`, `disable`, `enable`
   - 執行控制：`continue`, `next`, `step`, `finish`
   - 檢視變數：`print`, `x`, `info registers`, `backtrace`

4. **除錯 FreeRTOS Tasks**：
   - 檢視 `pxCurrentTCB`
   - 檢視 Ready List 和 Blocked List
   - 檢視 Stack 使用量
   - 使用 `uxTaskGetStackHighWaterMark()`

5. **除錯 Context Switch**：
   - 在 `vTaskSwitchContext` 設定中斷點
   - 單步執行組合語言
   - 觀察 Stack 和暫存器的變化

6. **除錯中斷處理**：
   - 檢視 CSRs（mstatus, mcause, mepc, mie, mip）
   - 在 ISR 設定中斷點
   - 手動觸發中斷（修改 mtime）

7. **常見問題除錯**：
   - Stack Overflow：啟用檢測、檢查 Stack 使用量、設定 Watchpoint
   - Deadlock：檢視 Task 狀態、檢視 Mutex/Semaphore 狀態
   - Priority Inversion：使用 Mutex（支援優先權繼承）
   - 記憶體洩漏：監控 Heap 使用量、設定條件中斷點

8. **進階技巧**：
   - 條件中斷點：`break [位置] if [條件]`
   - Watchpoint：`watch [變數]`
   - 自動化指令：`commands`
   - GDB 腳本：`-x script.gdb`
   - Python 擴充：自訂 GDB 指令

### 10.2 實務建議

**建議 #1：善用條件中斷點**

條件中斷點可以大幅提升除錯效率，避免手動檢查每個中斷點。

**建議 #2：建立 GDB 腳本**

將常用的 GDB 指令寫成腳本，可以快速重現除錯環境。

**建議 #3：啟用所有檢查**

在開發階段，啟用所有 FreeRTOS 的檢查功能：

```c
#define configCHECK_FOR_STACK_OVERFLOW  2
#define configUSE_MALLOC_FAILED_HOOK  1
#define configASSERT(x)  if (!(x)) { while(1); }
```

**建議 #4：使用 QEMU 進行功能除錯，實體硬體進行效能驗證**

QEMU 適合快速迭代和功能除錯，但最終還是需要在實體硬體上驗證效能。

**建議 #5：學習組合語言**

理解組合語言可以幫助你更深入地理解 RTOS 的運作機制，特別是 Context Switch 和中斷處理。

### 10.3 回到 Mock Scenario

回到我們的 Mock Scenario：

**問題**：系統在運行 2-3 小時後會隨機 Hard Fault。

**除錯過程**：

1. 在 QEMU 中重現問題
2. 啟用 Stack Overflow 檢測
3. 使用 GDB 設定中斷點在 `vApplicationStackOverflowHook`
4. 分析 Stack 使用量，發現 ControlTask 的 Stack 不夠大

**解決方案**：將 ControlTask 的 Stack 從 256 words 增加到 512 words。

**結果**：系統穩定運行超過 6 個月，沒有任何 Hard Fault。

### 10.4 下一篇預告

在下一篇文章中，我們將探討 **RTOS SMP：多核心環境下的排程與同步**：

- 多核心環境下的 Task 排程
- Spinlock 的實作（RISC-V AMO 指令）
- IPI (Inter-Processor Interrupt) 機制
- Cache Coherency 和 Memory Ordering
- QEMU SMP 驗證方法

敬請期待！

---

## 參考資料

**官方文檔**：

- [GDB User Manual](https://sourceware.org/gdb/current/onlinedocs/gdb/)
- [QEMU Documentation](https://www.qemu.org/docs/master/)
- [FreeRTOS Debugging](https://www.freertos.org/Debugging-Hard-Faults-On-Cortex-M-Microcontrollers.html)

**工具**：

- [RISC-V GNU Toolchain](https://github.com/riscv-collab/riscv-gnu-toolchain)
- [QEMU RISC-V](https://www.qemu.org/docs/master/system/target-riscv.html)

**延伸閱讀**：

- [GDB Tips and Tricks](https://interrupt.memfault.com/blog/gdb-for-firmware-1)
- [Debugging FreeRTOS with GDB](https://mcuoneclipse.com/2016/07/06/debugging-freertos-with-gdb/)
- [QEMU Debugging Tutorial](https://qemu.readthedocs.io/en/latest/system/gdb.html)

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
