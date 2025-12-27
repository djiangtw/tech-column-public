# 18. Debug 的藝術：M+U 模式除錯策略

> 「Debug 的困難不在於找到 Bug，而在於理解你看到的是什麼。」

1969 年 7 月 20 日，阿波羅 11 號正在降落月球表面。

突然，太空人 Neil Armstrong 聽到了警報聲。顯示器上出現了「1202」和「1201」錯誤代碼。整個任務指揮中心陷入緊張。

這是 AGC（Apollo Guidance Computer）的過載警報。電腦正在處理太多任務，無法完成所有工作。

但 MIT 的軟體工程師 Margaret Hamilton 早就預見了這個情況。她設計的系統會在過載時放棄低優先權任務，確保最關鍵的降落計算能夠繼續。警報只是告訴太空人：「我很忙，但核心功能沒問題。」

任務繼續。人類登上了月球。

這個故事告訴我們兩件事：

1. **系統需要告訴你它的狀態**——AGC 有顯示器（DSKY）讓太空人看到錯誤代碼
2. **你需要理解那些狀態代表什麼**——如果沒人知道 1202 是什麼意思，任務可能被終止

Debug 就是這兩件事的結合：**觀測**和**理解**。

---

## 雙重世界的挑戰

v0.x 時代，所有程式碼都在 M-mode。GDB 可以自由設定 breakpoint，查看任何記憶體，一切都很直觀。

v1.x 時代，我們有兩個世界：

```
┌─────────────────────────────────────┐
│         U-mode (User Tasks)         │
│  - 受 PMP 限制                      │
│  - 只能用 syscall 存取 Kernel       │
└─────────────────────────────────────┘
                  │
                ecall
                  │
                  ▼
┌─────────────────────────────────────┐
│         M-mode (Kernel)             │
│  - 完整權限                         │
│  - 處理 Trap、Syscall、中斷        │
└─────────────────────────────────────┘
```

Debug 變得複雜：
- 「現在 CPU 在哪個 mode？」
- 「這個記憶體存取會成功嗎？」
- 「Trap Handler 正在處理什麼？」

---

## GDB 的「透視眼」

這是最重要的觀念：**GDB 不受 PMP 限制**。

GDB 透過 JTAG 或 Debug 通道與 CPU 溝通。這個通道在所有保護機制之外——它可以讀寫任何記憶體，無論 PMP 設定如何。

這是雙面刃：

**好處**：你可以在 Kernel 區域設 breakpoint，即使你正在 Debug User Task
**陷阱**：GDB 能讀到的資料，不代表 CPU（在 U-mode）能讀到

### PMP 騙局

```c
// User Task
void user_task(void *arg) {
    uint64_t *kernel_data = (uint64_t *)0x80000000;
    uint64_t x = *kernel_data;  // 這會觸發 Access Fault
}
```

在 GDB 中：
```
(gdb) p *kernel_data
$1 = 0x12345678      # GDB 能讀到！
(gdb) c
# CPU 執行時觸發 Access Fault
```

**解法**：永遠記住 GDB 的視角 ≠ CPU 的視角。

---

## 關鍵 CSR 追蹤

Debug 時，這些 CSR 是你的「儀表板」：

```gdb
# .gdbinit 設定
define hook-stop
  printf "--- CPU State ---\n"
  printf "mstatus: 0x%lx  ", $mstatus
  
  # 解析 MPP（Previous Privilege Mode）
  set $mpp = ($mstatus >> 11) & 3
  if $mpp == 0
    printf "(MPP=User)"
  else if $mpp == 3
    printf "(MPP=Machine)"
  end
  printf "\n"
  
  printf "mepc:    0x%lx\n", $mepc
  printf "mcause:  %ld\n", $mcause
  printf "mtval:   0x%lx\n", $mtval
  printf "mscratch:0x%lx\n", $mscratch
  printf "-----------------\n"
end
```

### CSR 意義

| CSR | 意義 |
|-----|------|
| mstatus.MPP | Trap 前的 mode（00=U, 11=M） |
| mepc | Trap 發生時的 PC（要返回的位置） |
| mcause | 異常/中斷原因 |
| mtval | 觸發異常的地址或指令 |
| mscratch | 另一個 Stack Pointer（取決於當前 mode）|

---

## 薛丁格的 Stack

有個特別危險的時刻：`csrrw sp, mscratch, sp` 執行的瞬間。

在這條指令執行前後，sp 的含義完全不同：

```
執行前：sp = User Stack, mscratch = Kernel Stack
執行後：sp = Kernel Stack, mscratch = User Stack
```

如果你用 GDB 的 `next` 跨過這條指令，GDB 可能會困惑——它假設 Stack 是連續的，但你剛剛換了一個完全不同的 Stack。

**解法**：在 Trap Handler 裡，永遠用 `stepi`（單步指令），不要用 `next` 或 `step`。

```gdb
# 進入 trap_vector 後
(gdb) stepi
(gdb) stepi  # csrrw sp, mscratch, sp
(gdb) stepi  # 現在 sp 是 Kernel Stack 了
```

---

## Trap Handler Debug 技巧

### 技巧 1：在 Trap Entry 設 Breakpoint

```gdb
(gdb) b trap_vector
(gdb) c
# 任何 Trap 都會停在這裡
```

### 技巧 2：在 Syscall Handler 設 Breakpoint

```gdb
(gdb) b handle_syscall
(gdb) c
# 只在 syscall 時停下
```

### 技巧 3：條件 Breakpoint

```gdb
# 只在特定 syscall 停下
(gdb) b handle_syscall if tf->a7 == 1  # SYS_DELAY
```

### 技巧 4：觀察 Context Switch

```gdb
(gdb) b switch_to
(gdb) commands
  printf "Switching from %s to %s\n", prev->name, next->name
  continue
end
```

---

## 常見陷阱

### 陷阱 1：Trap Loop

**症狀**：系統卡死，不斷觸發同一個 Trap

**原因**：mepc 沒有更新，mret 回到同一條指令，再次觸發 Trap

**Debug**：
```gdb
(gdb) b trap_vector
(gdb) c
# 觀察 mepc 是否每次都一樣
(gdb) p /x $mepc
```

**修復**：確保 ecall handler 有 `mepc += 4`

### 陷阱 2：Stack 混亂

**症狀**：暫存器值莫名其妙，函式返回到錯誤位置

**原因**：mscratch 沒有正確更新，Context Switch 後用了錯誤的 Stack

**Debug**：
```gdb
(gdb) p /x $mscratch
(gdb) p /x $sp
# 確認它們指向合理的位置
```

### 陷阱 3：Breakpoint 在 U-mode 失效

**症狀**：設了 breakpoint 但從未觸發

**原因**：ebreak 在 U-mode 會觸發異常，但我們的 Trap Handler 可能沒處理

**修復**：在 Trap Handler 加上 ebreak 處理：
```c
case CAUSE_BREAKPOINT:  // cause = 3
    // 不做任何事，讓 GDB 接管
    break;
```

---

## kprintf：你的隨身 Debug 工具

有時候 GDB 不夠用——你需要看到程式執行的軌跡，而不只是單點狀態。

### 基本 kprintf

```c
void kprintf(const char *fmt, ...) {
    va_list args;
    va_start(args, fmt);

    // 輸出到 UART
    vsnprintf(buffer, sizeof(buffer), fmt, args);
    uart_puts(buffer);

    va_end(args);
}
```

### Debug Level

```c
#define DEBUG_LEVEL_ERROR   0
#define DEBUG_LEVEL_WARN    1
#define DEBUG_LEVEL_INFO    2
#define DEBUG_LEVEL_DEBUG   3
#define DEBUG_LEVEL_TRACE   4

static int debug_level = DEBUG_LEVEL_INFO;

#define TRACE(...) do { \
    if (debug_level >= DEBUG_LEVEL_TRACE) \
        kprintf("[TRACE] " __VA_ARGS__); \
} while(0)

#define DEBUG(...) do { \
    if (debug_level >= DEBUG_LEVEL_DEBUG) \
        kprintf("[DEBUG] " __VA_ARGS__); \
} while(0)
```

### 使用範例

```c
void handle_syscall(trap_frame_t *tf) {
    TRACE("syscall entry: num=%ld\n", tf->a7);

    switch (tf->a7) {
        case SYS_DELAY:
            DEBUG("sys_delay(%ld)\n", tf->a0);
            // ...
    }

    TRACE("syscall exit: ret=%ld\n", tf->a0);
}
```

---

## GDB Helper Script

建立一個 `.gdbinit` 檔案，放常用的 Debug 指令：

```gdb
# danieRTOS Debug Helpers

# 連接 QEMU
target remote :1234

# 顯示當前 mode
define mode
  set $mpp = ($mstatus >> 11) & 3
  if $mpp == 0
    printf "Previous mode: User\n"
  else
    printf "Previous mode: Machine\n"
  end
end

# 顯示當前 Task
define task
  printf "Current task: %s (id=%d, state=%d)\n", \
    current_task->name, current_task->id, current_task->state
end

# 顯示所有 Task
define tasks
  set $t = task_list
  while $t != 0
    printf "%s: id=%d state=%d\n", $t->name, $t->id, $t->state
    set $t = $t->next
  end
end

# 顯示 Trap Frame
define tf
  set $tf = (trap_frame_t *)$a0
  printf "mepc:  0x%016lx\n", $tf->mepc
  printf "ra:    0x%016lx\n", $tf->ra
  printf "sp:    0x%016lx\n", $tf->sp
  printf "a0-a7: %lx %lx %lx %lx %lx %lx %lx %lx\n", \
    $tf->a0, $tf->a1, $tf->a2, $tf->a3, \
    $tf->a4, $tf->a5, $tf->a6, $tf->a7
end

# 在 Trap Handler 單步
define trap_step
  stepi
  mode
  printf "pc: 0x%lx\n", $pc
end
```

---

## Debug 流程

當遇到問題時，遵循這個流程：

```
1. 確認問題
   └── 系統卡死？Crash？輸出錯誤？

2. 收集資訊
   ├── 最後一個 kprintf 輸出是什麼？
   ├── 用 GDB attach，看 $pc 在哪？
   └── 檢查 mcause, mepc, mtval

3. 縮小範圍
   ├── 在可疑位置設 breakpoint
   ├── 用 TRACE 追蹤執行路徑
   └── 二分法：註解掉一半程式碼

4. 形成假設
   └── 根據資訊推測原因

5. 驗證假設
   ├── 加入驗證程式碼
   └── 用 GDB 觀察特定變數

6. 修復並驗證
   └── 修改程式碼，確認問題解決
```

---

## 小結

這一章我們學會了在 **雙重世界** 中 Debug 的技巧：

1. **GDB 透視眼**：GDB 不受 PMP 限制，但這也是陷阱
2. **CSR 追蹤**：mstatus、mepc、mcause、mtval、mscratch
3. **薛丁格 Stack**：csrrw 時要用 stepi
4. **常見陷阱**：Trap Loop、Stack 混亂、Breakpoint 失效
5. **kprintf**：你的隨身 Debug 工具
6. **GDB Helper**：自動化常用 Debug 指令

就像阿波羅 11 號的太空人需要理解 1202 警報的意義，我們也需要理解系統告訴我們的每一個訊息。Debug 不只是找 Bug，更是理解系統如何運作。

下一章，我們來計算「安全的代價」——v1.x 比 v0.x 慢了多少？這值得嗎？

---

## 本章重點

| 概念 | 說明 |
|------|------|
| Apollo 1202 | 1969 登月時的過載警報，系統繼續工作 |
| GDB 透視眼 | GDB 不受 PMP 限制，可讀寫任何記憶體 |
| PMP 騙局 | GDB 能讀 ≠ CPU 在 U-mode 能讀 |
| 薛丁格 Stack | csrrw 時 sp 含義改變，用 stepi |
| kprintf | 輸出到 UART，追蹤執行軌跡 |
| .gdbinit | GDB 啟動時執行的腳本 |

---

## Debug 案例：真實問題分析

### 案例 1：神秘的卡死

**現象**：系統啟動後執行幾秒鐘就卡死，沒有任何輸出。

**除錯過程**：

1. 用 GDB attach，查看 `$pc`：

```gdb
(gdb) p /x $pc
$1 = 0x80000abc  # 在 trap_vector 附近
```

2. 檢查 `mcause`：

```gdb
(gdb) p $mcause
$2 = 8  # User ecall
```

3. 觀察 `mepc`：

```gdb
(gdb) p /x $mepc
$3 = 0x80100100  # 總是同一個值
```

4. 連續按 `c` 再 Ctrl-C，發現 `mepc` 從未改變。

**診斷**：ecall handler 沒有 `mepc += 4`，所以 `mret` 回到同一條 `ecall`，無限循環。

**修復**：

```c
void handle_syscall(trap_frame_t *tf) {
    // ... 處理 syscall ...
    tf->mepc += 4;  // 跳過 ecall 指令！
}
```

### 案例 2：暫存器莫名其妙變成 0

**現象**：User Task 的變數計算結果莫名其妙變成 0。

**除錯過程**：

1. 在 Task 中加入 kprintf，發現問題出現在 syscall 返回後
2. 在 trap_vector 設 breakpoint，觀察暫存器儲存/恢復

**診斷**：Context Save 時儲存了 31 個暫存器，但恢復時只恢復了 30 個——漏了 `t0`。

**修復**：仔細對齊 save/restore 的暫存器列表。

---

## 參考資料

**Apollo Guidance Computer**

- **Apollo 11 Source Code** - GitHub
  <https://github.com/chrislgarry/Apollo-11>
  AGC 的原始程式碼，包括 1202 警報的處理邏輯。

- **Margaret Hamilton** - Wikipedia
  <https://en.wikipedia.org/wiki/Margaret_Hamilton_(software_engineer)>
  AGC 軟體總監，提出「軟體工程」一詞。

**GDB 參考**

- **GDB Manual - Remote Debugging**
  <https://sourceware.org/gdb/current/onlinedocs/gdb/Remote-Debugging.html>
  使用 GDB 遠端除錯的完整指南。

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)
