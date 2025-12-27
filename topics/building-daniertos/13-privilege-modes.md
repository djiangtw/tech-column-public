# 權力的遊戲

**作者**: Danny Jiang
**日期**: 2025-12-14

---

## 前言：大通鋪的悲劇

在 v0.x 的世界裡，我們的 danieRTOS 像是一間大通鋪。

所有 Task 住在同一個大房間，共用同一把鑰匙（M-mode），可以隨意翻動任何人的抽屜。Sensor Task 可以直接修改 Logger Task 的變數；Logger Task 可以覆寫 LED Task 的 Stack。

這在我們精心編寫的 Demo 裡運作良好。但現實世界呢？

想像這個場景：

```c
void buggy_task(void) {
    char buffer[64];
    strcpy(buffer, very_long_string);  // Stack Overflow!
}
```

一個 Task 的 Stack Overflow，覆蓋了隔壁 Task 的資料。隔壁 Task 莫名其妙崩潰，而你花了三天 Debug，最後發現兇手是另一個看似無關的 Task。

更糟的是，惡意程式碼可以做任何事：

```c
void evil_task(void) {
    // 直接關閉中斷，獨佔 CPU
    asm volatile("csrc mstatus, 8");
    while (1) {}  // 系統凍結
}
```

v0.x 沒有任何防禦機制。一個 Bug、一段惡意程式碼，就能癱瘓整個系統。

這不是理論上的問題——歷史上無數的災難都源於此。而解決方案，早在 1960 年代就被發明了。

---

## 一、歷史：Multics 的八層城堡 (1964)

讓我們回到 1964 年的 MIT。

**Project MAC** (Multiple Access Computer) 正在開發一個野心勃勃的作業系統：**Multics** (Multiplexed Information and Computing Service)。這是人類第一次嘗試建造一個「多使用者、分時共享、安全隔離」的系統。

Multics 的設計者面對一個根本問題：**如何讓不信任的程式共存？**

他們的答案是 **Protection Rings**——保護環。

```
        ┌─────────────────┐
        │     Ring 7      │ ← 最低權限（普通程式）
        ├─────────────────┤
        │     Ring 6      │
        ├─────────────────┤
        │      ....       │
        ├─────────────────┤
        │     Ring 1      │
        ├─────────────────┤
        │     Ring 0      │ ← 最高權限（核心）
        └─────────────────┘
```

Multics 有 **8 層保護環**（Ring 0 到 Ring 7）。每層有不同的權限：

- **Ring 0**：核心最敏感的部分（中斷處理、記憶體管理）
- **Ring 1-3**：作業系統服務
- **Ring 4-7**：使用者程式

每個程式碼片段都有一個「ring number」，決定它能存取什麼資源。低編號的 ring 可以存取高編號 ring 的資料，但反過來不行——就像城堡的同心圓城牆，內層可以看到外層，外層無法進入內層。

更精妙的是，Multics 為每個記憶體區段定義了三個 ring numbers：`[R1, R2, R3]`

- **R1**：Write Bracket（誰能寫入）
- **R2**：Read Bracket（誰能讀取）
- **R3**：Call Bracket（誰能呼叫）

這是一個精美的、數學上完備的安全模型。

### 為什麼 Multics 失敗了？

1972 年，Honeywell 6180 終於實現了硬體支援的 8 層保護環。然而，Multics 最終只賣出約 80 套系統。

問題出在哪？

**太複雜了。**

程式設計師需要為每個 segment 設定三個 ring numbers，需要理解 ring crossing 的規則，需要小心處理權限提升⋯⋯ 這些額外的心智負擔，讓開發變得痛苦。

1969 年，兩位 Bell Labs 的研究員受夠了這種複雜性。他們決定離開 Multics 專案，自己寫一個作業系統。

這兩個人叫 **Ken Thompson** 和 **Dennis Ritchie**。他們寫的系統叫做 **Unix**。

---

## 二、Unix 的反思：簡單就是美

Ken Thompson 後來這樣評價 Multics：

> "It was overdesigned and overbuilt and over everything."
> （設計過度、建造過度、一切都過度。）

Unix 的設計哲學是 Multics 的反面：**Keep It Simple, Stupid (KISS)**。

8 層保護環？Unix 只要 2 層：

| Multics | Unix |
|---------|------|
| Ring 0-7 | Kernel Mode / User Mode |
| 8 種權限等級 | 2 種：可信任 / 不可信任 |
| 複雜的 Bracket 系統 | 簡單的「能不能」 |

Ken Thompson 和 Dennis Ritchie 意識到一個關鍵洞察：

> **大部分情況下，你只需要區分「可信任」和「不可信任」。**

Kernel 是可信任的——它經過仔細審查，負責管理一切。使用者程式是不可信任的——它可能有 Bug，可能是惡意的，不能讓它直接操作硬體。

這就夠了。

有趣的是，Unix 這個名字本身就是對 Multics 的諷刺：

- **Multics** = Multiplexed（多工）
- **Unix** = Uniplexed（單一）

還有另一個版本的故事：Unix 是「castrated Multics」（閹割版 Multics）。無論哪個版本，都傳達了同樣的訊息：**簡單，才是王道。**

---

## 三、RISC-V 的現代設計

50 年過去了。Multics 的 8 層變成了歷史，Unix 的 2 層成為主流。現代處理器如何處理特權模式？

| 架構 | 層數 | 名稱 |
|------|------|------|
| Multics | 8 | Ring 0-7 |
| x86 | 4 | Ring 0-3（實際只用 0 和 3） |
| ARM | 4 | EL0-EL3 |
| RISC-V | 3 | M / S / U mode |

RISC-V 選擇了 **3 層**：

```
┌─────────────────────────────────────┐
│  M-mode (Machine)                   │ ← 最高權限
│  完全控制硬體、CSRs、中斷           │
├─────────────────────────────────────┤
│  S-mode (Supervisor)                │ ← 作業系統
│  虛擬記憶體、部分 CSRs              │
├─────────────────────────────────────┤
│  U-mode (User)                      │ ← 最低權限
│  只能執行普通指令                   │
└─────────────────────────────────────┘
```

### 為什麼是 3 層？

1. **M-mode 是必要的**：嵌入式系統需要直接控制硬體，韌體和 Bootloader 需要最高權限。

2. **S-mode 是可選的**：如果你要運行 Linux 這種完整的作業系統，需要 S-mode 來管理虛擬記憶體。但 MCU 通常不需要。

3. **U-mode 是保護邊界**：不信任的程式碼運行在這裡。

這種設計讓 RISC-V 非常靈活：

- **嵌入式 MCU**：只實作 M-mode（最簡單）
- **帶保護的 MCU**：M + U mode（danieRTOS v1.x）
- **完整系統**：M + S + U mode（Linux）
- **虛擬化**：M + H + S + U mode（Hypervisor）

### 權限編號的秘密

注意 RISC-V 的 mode 編號：

| Mode | 編號 |
|------|------|
| U-mode | 0 |
| S-mode | 1 |
| M-mode | 3 |

**編號 2 去哪了？**

這是故意留空的。RISC-V 規格保留編號 2 給未來可能的擴展。更重要的是，這種編號方式讓權限比較變得簡單：編號越大，權限越高。

---

## 四、danieRTOS v1.x 的選擇

了解了歷史和現代設計，我們來看 danieRTOS v1.x 的選擇：

### 我們只需要 M + U

| Mode | 用途 |
|------|------|
| M-mode | Kernel（排程器、中斷處理、IPC） |
| U-mode | User Tasks（應用程式邏輯） |

**為什麼不需要 S-mode？**

S-mode 主要用於管理虛擬記憶體（MMU、Page Table）。我們的 MCU 沒有 MMU，所以不需要 S-mode。

我們用 **PMP (Physical Memory Protection)** 代替 MMU——這是 M-mode 的功能，可以直接限制 U-mode 能存取的實體記憶體範圍。

### 從大通鋪到公寓大樓

讓我用一個比喻來總結這個轉變：

**v0.x：大通鋪**
```
┌──────────────────────────────┐
│  Task A  Task B  Task C  ... │  ← 所有人住一起
│  （M-mode）                  │
│  可以互相存取、沒有隔離      │
└──────────────────────────────┘
```

**v1.x：公寓大樓**
```
┌─────────────────────────────────────┐
│         管理員辦公室               │ ← Kernel (M-mode)
│  （排程、IPC、中斷處理）           │
├──────┬──────┬──────┬───────────────┤
│ 房間A │ 房間B │ 房間C │ ...          │ ← User Tasks (U-mode)
│(TaskA)│(TaskB)│(TaskC)│              │
│  PMP保護各自的 Stack               │
└──────┴──────┴──────┴───────────────┘

規則：
1. 每個房間有門鎖（PMP）
2. 要找管理員辦事？去櫃檯填單（Syscall）
3. 闖入別人房間？保全會來（Trap Handler）
```

### 系統架構變化

```
v0.x                           v1.x
────────────────────────────   ────────────────────────────
                               ┌────────────────────────┐
                               │  User Tasks (U-mode)   │
                               ├────────────────────────┤
                               │  Syscall API (ecall)   │
                               ╞════════════════════════╡
┌────────────────────────┐     │                        │
│  All Code (M-mode)     │  →  │  Kernel (M-mode)       │
│  Task, Scheduler, IPC  │     │  Task, Scheduler, IPC  │
├────────────────────────┤     │  + Syscall Handler     │
│  HAL, Trap Handler     │     │  + PMP Configuration   │
└────────────────────────┘     └────────────────────────┘
```

---

## 五、mstatus.MPP：追蹤當前模式

最後，讓我們看一個技術細節：CPU 如何知道「現在是什麼模式」？

在 RISC-V 中，`mstatus` 暫存器的 **MPP** 欄位記錄了這個資訊：

```
mstatus 暫存器（部分）：

┌─────┬─────┬─────────────────────┬─────┬─────┐
│ ... │ MPP │ ...                 │ MIE │ ... │
│     │[12:11]                    │ [3] │     │
└─────┴─────┴─────────────────────┴─────┴─────┘

MPP (Machine Previous Privilege):
  00 = U-mode
  01 = S-mode
  11 = M-mode
```

**MPP 的作用**：

1. 當 Trap 發生時（中斷或異常），CPU 會把「trap 前的模式」存入 MPP
2. 當執行 `mret` 時，CPU 會根據 MPP 的值，返回對應的模式

這就是模式切換的核心機制：

```c
// U-mode → M-mode（Trap 發生時，自動由硬體處理）
// CPU 自動：mstatus.MPP = 00 (記錄來自 U-mode)

// M-mode → U-mode（mret 指令）
// 設定 mstatus.MPP = 00
// 設定 mepc = user_task_entry
// mret  ← CPU 根據 MPP 切換到 U-mode
```

我們將在下一章詳細實作這個機制。

---

## 六、常見錯誤與陷阱

在實作 Privilege Mode 時，新手常犯以下錯誤：

### 錯誤 1：忘記設定 MPP 就執行 mret

```c
// 錯誤！沒設定 MPP
void jump_to_user(void) {
    csr_write(mepc, user_entry);
    asm volatile("mret");  // MPP 可能還是 M-mode！
}
```

結果：系統以為要回到 M-mode，User Task 仍然擁有完整權限。

**正確做法**：

```c
void jump_to_user(void) {
    uint64_t mstatus = csr_read(mstatus);
    mstatus &= ~(3UL << 11);  // 清除 MPP
    mstatus |= (0UL << 11);   // 設定 MPP = U-mode
    csr_write(mstatus, mstatus);
    csr_write(mepc, user_entry);
    asm volatile("mret");
}
```

### 錯誤 2：在 U-mode 執行特權指令

```c
// User Task 嘗試直接操作 CSR
void user_task(void) {
    csr_write(mstatus, 0);  // 會觸發 Illegal Instruction！
}
```

U-mode 無法存取 `m*` 開頭的 CSR（如 mstatus、mepc、mcause）。嘗試這樣做會觸發 Illegal Instruction Exception。

### 錯誤 3：Trap Handler 沒有正確返回

```c
// 錯誤：用 ret 而不是 mret
trap_handler:
    // ... 處理 trap ...
    ret  // 錯誤！這只是普通的函數返回
```

`ret` 只是跳到 `ra` 暫存器指向的位置，不會切換 Privilege Mode。必須用 `mret` 才能根據 MPP 切換回正確的模式。

---

## 七、實作路線圖

了解了理論，接下來 v1.x 系列的實作順序是：

```
Ch 14: Dual Stack（每個 Task 兩個 Stack）
    └── Trap 時 Stack 切換機制

Ch 15: Syscall（ecall 指令）
    └── User Task 請求 Kernel 服務

Ch 16: PMP（記憶體保護）
    └── 阻止 User 存取 Kernel 記憶體

Ch 17: Fault Handling（錯誤處理）
    └── Task 犯規時的處理機制

Ch 18: Debugging（除錯技巧）
    └── 在雙重世界中 Debug

Ch 19: Performance（效能分析）
    └── 安全的代價有多大？
```

### 程式碼變化預覽

| 模組 | v0.x | v1.x |
|------|------|------|
| TCB | 1 個 stack_ptr | 2 個：user_sp, kernel_sp |
| Trap Handler | 只處理 Timer | 處理 Timer + Syscall + Fault |
| Task 啟動 | 直接 call | 透過 mret 進入 U-mode |
| IPC API | 直接呼叫 | 透過 syscall |

---

## 總結

| 概念 | 說明 |
|------|------|
| 問題 | v0.x 沒有隔離，一個 Bug 全系統崩潰 |
| 歷史 | Multics 8 層太複雜，Unix 簡化為 2 層 |
| RISC-V | 3 層設計（M/S/U），可按需選配 |
| v1.x 選擇 | M + U mode，用 PMP 保護記憶體 |
| 核心機制 | mstatus.MPP 追蹤模式，mret 切換模式 |

從 Multics 的 8 層城堡，到 Unix 的 2 層簡約，再到 RISC-V 的 3 層靈活設計——歷史告訴我們：**簡單，才是可行的安全。**

下一章，我們將進入實作：如何在 Kernel 和 User Mode 之間切換？如何讓每個 Task 有兩個 Stack？

---

## 本章重點

1. **大通鋪問題**：v0.x 沒有隔離，任何 Task 都可以破壞其他 Task
2. **Multics 保護環**：1960 年代的創新，但 8 層太複雜
3. **Unix 簡化**：只需 Kernel/User 兩層，簡單有效
4. **RISC-V 三層**：M/S/U，按需選配
5. **v1.x 架構**：M-mode (Kernel) + U-mode (User Tasks) + PMP

---

## 參考資料

**歷史**

- **Multics** - Wikipedia
  https://en.wikipedia.org/wiki/Multics

- **Protection Ring** - Wikipedia
  https://en.wikipedia.org/wiki/Protection_ring

- **Multics Data Security** - multicians.org
  https://multicians.org/muldat.html

**RISC-V 規格**

- **RISC-V Privileged Specification**
  https://github.com/riscv/riscv-isa-manual

**延伸閱讀**

- **See RISC-V Run** - Danny Jiang
  Chapter 2: Privilege Levels, Chapter 10: Machine Mode

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)

