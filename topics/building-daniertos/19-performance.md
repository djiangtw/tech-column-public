# 19. 安全的代價：效能分析與優化

> 「天下沒有白吃的午餐。」—— Robert A. Heinlein

2018 年 1 月 3 日，整個科技界都在討論一件事。

這一天，Google Project Zero 和幾個大學的研究團隊公開了兩個驚人的 CPU 漏洞：**Meltdown** 和 **Spectre**。這些漏洞影響了過去 20 年間生產的幾乎所有 Intel CPU、大部分 AMD 和 ARM 處理器。

漏洞的威力令人毛骨悚然：攻擊者可以從一個普通的 User 程式，**讀取 Kernel 的機密資料**——包括密碼、加密金鑰、其他程式的記憶體。

這不是軟體 Bug，這是**硬體設計缺陷**。

### 為什麼會這樣？

根本原因是 Intel 為了追求效能，在 CPU 的「推測執行」（Speculative Execution）中繞過了安全檢查。

現代 CPU 不會等一條指令完成後才執行下一條——它會「猜測」接下來的分支走向，提前執行可能需要的指令。如果猜對了，省下等待時間；如果猜錯了，丟棄結果，回滾狀態。

問題是，即使結果被丟棄，**Cache 的狀態已經改變了**。

攻擊者可以這樣做：

1. 設計一個會被「推測執行」的程式碼路徑
2. 讓推測執行讀取 Kernel 記憶體
3. 根據讀到的資料，存取不同的 Cache line
4. 推測執行回滾，但 Cache 狀態保留
5. 透過時間差測量，推斷出 Kernel 資料

這就是所謂的 **Side Channel Attack**。

### 修復的代價

修復這些漏洞需要在軟體層面加入額外的保護：

- **KPTI (Kernel Page Table Isolation)**：每次 syscall 都切換 Page Table
- **Retpoline**：改變間接跳轉的實現方式

代價是 **效能下降 5-30%**，取決於工作負載。

這就是安全與效能的權衡：Intel 為了快幾個百分點，最終付出了巨大的代價——不只是效能損失，還有信譽損失和法律訴訟。

---

## 機場安檢模型

想像兩種飛機場：

**v0.x：私人機場**
- 你開車直接到飛機旁
- 上機就飛
- 快，但沒有任何安全檢查

**v1.x：商業機場**
- 排隊、過安檢、驗護照
- 等候、登機
- 慢，但確保沒有危險物品

我們在 danieRTOS v1.x 加入了「安檢」——User Mode、PMP、Syscall。問題是：這些安檢要花多少時間？

---

## 測量工具：mcycle 和 minstret

RISC-V 提供了兩個效能計數器：

| CSR | 說明 |
|-----|------|
| mcycle | CPU 週期數（時脈計數） |
| minstret | 已執行指令數 |

```c
static inline uint64_t read_cycle(void) {
    uint64_t cycle;
    asm volatile("rdcycle %0" : "=r"(cycle));
    return cycle;
}

static inline uint64_t read_instret(void) {
    uint64_t instret;
    asm volatile("rdinstret %0" : "=r"(instret));
    return instret;
}
```

### 測量框架

```c
#define BENCHMARK(name, code) do { \
    uint64_t start = read_cycle(); \
    code; \
    uint64_t end = read_cycle(); \
    kprintf(#name ": %lu cycles\n", end - start); \
} while(0)
```

---

## v0.x vs v1.x：開銷對比

### 測試 1：直接呼叫 vs Syscall

**v0.x：直接呼叫**
```c
void test_v0(void) {
    BENCHMARK(direct_yield, {
        for (int i = 0; i < 1000; i++) {
            task_yield();  // 直接呼叫
        }
    });
}
// 結果：約 50,000 cycles（每次 ~50 cycles）
```

**v1.x：透過 Syscall**
```c
void test_v1(void) {
    BENCHMARK(syscall_yield, {
        for (int i = 0; i < 1000; i++) {
            sys_yield();  // ecall → trap → handler → mret
        }
    });
}
// 結果：約 200,000 cycles（每次 ~200 cycles）
```

**開銷**：約 4 倍

### Syscall 開銷拆解

| 步驟 | 預估 Cycles |
|------|-------------|
| ecall 觸發 Trap | ~10 |
| csrrw sp, mscratch, sp | ~2 |
| 儲存 31 個暫存器 | ~35 |
| 讀取 mepc, mstatus 等 | ~10 |
| call trap_handler | ~5 |
| Syscall 分發 | ~10 |
| 實際工作（yield） | ~50 |
| 恢復暫存器 | ~35 |
| 寫入 mepc, mstatus | ~10 |
| csrrw sp, mscratch, sp | ~2 |
| mret | ~10 |
| **總計** | **~180** |

---

### 測試 2：Context Switch

**v0.x**：只儲存 callee-saved registers（13 個）
```c
// 約 80 cycles
```

**v1.x**：儲存全部暫存器 + CSRs（40+ 個），更新 mscratch
```c
// 約 250 cycles
```

**開銷**：約 3 倍

---

### 測試 3：PMP 更新（如果使用 Per-Task PMP）

```c
void update_pmp(tcb_t *task) {
    csr_write(pmpaddr0, task->region_start >> 2);
    csr_write(pmpaddr1, task->region_end >> 2);
    csr_write(pmpcfg0, PMP_TOR | PMP_R | PMP_W | PMP_X);
}
// 約 20 cycles
```

如果每次 Context Switch 都更新 PMP，會增加約 20 cycles。

---

## 總體開銷

| 操作 | v0.x | v1.x | 開銷 |
|------|------|------|------|
| yield/syscall | ~50 | ~180 | 3.6x |
| Context Switch | ~80 | ~250 | 3.1x |
| 中斷處理 | ~60 | ~200 | 3.3x |

**結論**：v1.x 比 v0.x 慢約 3-4 倍。

---

## 這值得嗎？

讓我們換個角度思考。

### 每秒處理量

假設：
- CPU 時脈：100 MHz（每秒 1 億個 cycles）
- 每次 syscall：200 cycles
- 純 syscall 處理量：500,000 次/秒

對於大多數嵌入式應用，這綽綽有餘。

### 安全的價值

沒有安全保護的代價：

| 事件 | 代價 |
|------|------|
| Morris Worm (1988) | 6,000 台電腦感染，估計損失數百萬美元 |
| Intel Meltdown (2018) | 數十億設備需要 patch，效能下降 5-30% |
| Therac-25 (1985-87) | 6 人死亡 |

**200 cycles 的「保險費」換來的是系統不會因為一個 Bug 全體崩潰。**

---

## 優化技巧

如果效能真的是瓶頸，有幾個優化方向：

### 1. Fast Syscall Path

對於常用的 syscall（如 yield），可以跳過部分檢查：

```c
void handle_syscall(trap_frame_t *tf) {
    // Fast path for yield
    if (tf->a7 == SYS_YIELD) {
        // 直接切換，不需要參數處理
        scheduler_yield();
        tf->mepc += 4;
        return;
    }

    // Normal path for other syscalls
    // ...
}
```

### 2. Lazy 暫存器儲存

只有在 Context Switch 時才儲存全部暫存器。如果 syscall 不切換 Task，只需要儲存 caller-saved：

```c
void trap_vector_fast:
    # 只儲存 caller-saved
    sd ra, ...
    sd t0-t6, ...
    sd a0-a7, ...

    call handle_syscall

    # 如果需要 context switch，跳到 full save
    bnez a0, trap_vector_full_save

    # 否則直接返回
    ld ra, ...
    ld t0-t6, ...
    ld a0-a7, ...
    mret
```

### 3. Inline Syscall Handler

對於簡單的 syscall，可以直接在 assembly 處理：

```asm
trap_vector:
    # 快速檢查是否是 yield
    csrr t0, mcause
    li t1, 8  # User ecall
    bne t0, t1, slow_path

    csrr t0, mscratch
    ld t1, A7_OFFSET(t0)  # syscall number
    bne t1, zero, slow_path  # SYS_YIELD = 0

    # Inline yield
    # ...

    # mepc += 4
    csrr t0, mepc
    addi t0, t0, 4
    csrw mepc, t0

    mret
```

### 4. 減少 PMP 更新

如果使用全域二分策略（所有 User Task 共享同一區域），就不需要每次 Context Switch 更新 PMP。

---

## Benchmark Task

完整的 Benchmark 程式碼：

```c
void benchmark_task(void *arg) {
    kprintf("=== danieRTOS v1.x Benchmark ===\n\n");

    // Test 1: Syscall overhead
    {
        uint64_t start = read_cycle();
        for (int i = 0; i < 10000; i++) {
            sys_yield();
        }
        uint64_t end = read_cycle();
        uint64_t per_call = (end - start) / 10000;
        kprintf("Syscall (yield): %lu cycles/call\n", per_call);
    }

    // Test 2: Semaphore syscall
    {
        static sem_t sem = { .count = 10000 };
        uint64_t start = read_cycle();
        for (int i = 0; i < 10000; i++) {
            sys_sem_wait(&sem);
        }
        uint64_t end = read_cycle();
        uint64_t per_call = (end - start) / 10000;
        kprintf("Syscall (sem_wait): %lu cycles/call\n", per_call);
    }

    // Test 3: Context Switch
    // 需要另一個高優先權 Task 來觸發切換

    kprintf("\n=== Benchmark Complete ===\n");
}
```

預期輸出：

```
=== danieRTOS v1.x Benchmark ===

Syscall (yield): 185 cycles/call
Syscall (sem_wait): 210 cycles/call

=== Benchmark Complete ===
```

---

## 效能 vs 安全：設計決策

| 決策 | 選擇 | 理由 |
|------|------|------|
| User Mode | ✅ 使用 | 隔離是必要的 |
| PMP | ✅ 使用 | 記憶體保護是必要的 |
| 全域二分 PMP | ✅ 使用 | 簡單，避免頻繁更新 |
| Per-Task PMP | ❌ 不使用 | 開銷太大，複雜度高 |
| Fast Syscall Path | ⚠️ 可選 | 如果效能是瓶頸 |

---

## 小結

這一章我們分析了 **v1.x 的效能開銷**：

1. **測量工具**：mcycle, minstret
2. **開銷來源**：Trap、Context Save/Restore、PMP 更新
3. **數據**：v1.x 比 v0.x 慢約 3-4 倍
4. **價值**：200 cycles 換來系統穩定性和安全性
5. **優化**：Fast Path、Lazy Save、Inline Handler

Intel Meltdown 事件告訴我們：**為了效能繞過安全，最終會付出更大的代價**。我們選擇接受 3-4 倍的開銷，換來一個「鐵桶江山」——即使有惡意或有 Bug 的 Task，系統也能繼續運行。

這就是 danieRTOS v1.x 的完成。從 v0.x 的「大通鋪」到 v1.x 的「公寓大樓」，我們建立了一個有隔離、有保護、有秩序的小王國。

---

## 本章重點

| 概念 | 說明 |
|------|------|
| Meltdown/Spectre | 2018 Intel 漏洞，為效能犧牲安全 |
| mcycle | CPU 週期計數器 |
| minstret | 已執行指令計數器 |
| Syscall 開銷 | 約 180 cycles（對比直接呼叫 50 cycles） |
| 總體開銷 | v1.x 比 v0.x 慢約 3-4 倍 |
| 權衡 | 200 cycles 換來系統穩定性和安全性 |

---

## v1.x 系列總結

經過 7 篇文章，我們完成了 danieRTOS v1.x：

| 章節 | 主題 | 成果 |
|------|------|------|
| 13 | 權力的遊戲 | M + U mode 架構 |
| 14 | 穿越邊境的密道 | Dual Stack + mscratch |
| 15 | 國王的信箱 | System Call 機制 |
| 16 | 隱形的力場 | PMP 記憶體保護 |
| 17 | 鐵桶江山 | 錯誤處理與隔離 |
| 18 | Debug 的藝術 | M+U 除錯策略 |
| 19 | 安全的代價 | 效能分析與優化 |

**v0.x → v1.x 的轉變**：
- 從「大通鋪」到「公寓大樓」
- 從「上帝視角」到「君主立憲」
- 從「不安全但快」到「安全且足夠快」

下一站：**v2.x SMP 多核心支援**。

我們將面對新的挑戰：當有多個國王（多個 Core）時，如何協調他們的統治？如何確保他們不會互相衝突？

敬請期待！

---

## 參考資料

**安全事件**

- **Meltdown and Spectre** - meltdownattack.com
  <https://meltdownattack.com/>
  2018 年 CPU 漏洞的官方網站。

- **Therac-25** - Wikipedia
  <https://en.wikipedia.org/wiki/Therac-25>
  放射治療機事故的詳細記錄。

**RISC-V 效能計數器**

- **RISC-V Privileged Specification - Chapter 3.1.10 Hardware Performance Monitor**
  <https://github.com/riscv/riscv-isa-manual>
  mcycle、minstret 等計數器的規格。

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)
