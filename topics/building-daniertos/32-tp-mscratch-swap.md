# 靈魂交換：tp 與 mscratch 的雙重人格

**作者**: Danny Jiang
**日期**: 2025-12-26

---

## 前言：兩個世界的鑰匙

在上一章，我們用「臥底警察」的比喻來理解 v3.x 的核心挑戰。今天，我們要深入這個比喻的核心：**如何在毫秒之間完成身份切換？**

想像臥底警察隨身攜帶兩把鑰匙：
- **便服口袋裡的假身份證** → 混混世界的通行證
- **秘密夾層裡的警證** → 警察世界的通行證

當電話響起（trap 發生），他需要：
1. 把假身份證藏到秘密夾層
2. 從秘密夾層取出警證
3. 瞬間從「阿明」變成「警員 0037」

在 RISC-V 中，這兩把「鑰匙」就是：
- **tp 暫存器** → 當前使用的身份
- **mscratch 暫存器** → 秘密夾層，藏著另一個身份

而那個「電話響起」的瞬間，只需要一條指令：

```asm
csrrw tp, mscratch, tp
```

**就這麼簡單。但魔鬼在細節裡。**

---

## 一、csrrw：原子交換的魔法

### 1.1 指令解析

`csrrw` 是 RISC-V 的 CSR (Control and Status Register) 操作指令。它的全名是 **CSR Read & Write**，執行的是「原子交換」操作：

```asm
csrrw rd, csr, rs
# 等價於（但是原子的）：
#   t = csr
#   csr = rs
#   rd = t
```

在我們的場景中：

```asm
csrrw tp, mscratch, tp
# 1. 讀取 mscratch 到臨時變數 t
# 2. 將舊的 tp 寫入 mscratch
# 3. 將 t 寫入 tp
# 結果：tp 和 mscratch 的值互換
```

**關鍵特性**：
- **原子性**：整個操作不會被中斷打斷
- **無額外暫存器**：不需要額外的暫存器做暫存
- **一條指令**：最小化 trap entry 的 overhead

### 1.2 為什麼不用兩條指令？

你可能會問：為什麼不用這樣的方式？

```asm
# 錯誤示範（非原子）
mv t0, tp           # 保存 tp
csrr tp, mscratch   # 讀取 mscratch 到 tp
csrw mscratch, t0   # 寫入舊 tp 到 mscratch
```

**問題**：這需要一個臨時暫存器 `t0`。但在 trap entry 的第一刻，我們**不能破壞任何暫存器**，因為所有暫存器都可能包含 User 程式的重要資料！

`csrrw` 的原子交換解決了這個問題：它不需要任何臨時暫存器。

---

## 二、cpu_t 結構：Kernel 的身份證

當 `csrrw` 執行後，`tp` 指向的是 `cpu_t` 結構。這個結構是 Kernel 在每個核心上的「身份證」。

### 2.1 結構定義

```c
typedef struct cpu {
    /* ─── Trap Scratch Area (offsets used in assembly) ─── */
    uint64_t kernel_trap_sp; /* +0:  當前 Task 的 Kernel Stack Top */
    uint64_t user_sp_save;   /* +8:  臨時保存 User SP */
    uint64_t user_tp_save;   /* +16: 保存 User TP */

    /* ─── Standard SMP Data ─── */
    uint64_t hartid;         /* +24: 硬體執行緒 ID */
    tcb_t   *current_task;   /* +32: 當前執行的 Task */
    tcb_t   *idle_task;      /* +40: 這個核心的 Idle Task */
    uint32_t irq_nesting;    /* +48: 中斷巢狀計數 */
    uint32_t need_reschedule;/* +52: 需要重新排程的旗標 */

    /* ─── Cache Line Padding ─── */
    uint8_t  padding[64 - 56];
} __attribute__((aligned(64))) cpu_t;
```

**關鍵設計**：前三個欄位（kernel_trap_sp、user_sp_save、user_tp_save）構成「Trap Scratch Area」。這些欄位的 offset 是**硬編碼**在組合語言中的！

### 2.2 為什麼 offset 這麼重要？

看看 trap.S 中的程式碼：

```asm
#define CPU_KERNEL_TRAP_SP  0
#define CPU_USER_SP_SAVE    8
#define CPU_USER_TP_SAVE    16

trap_entry:
    csrrw   tp, mscratch, tp    # tp = &cpu_t
    sd      sp, CPU_USER_SP_SAVE(tp)    # 保存 user sp 到 offset 8
    ld      sp, CPU_KERNEL_TRAP_SP(tp)  # 載入 kernel sp from offset 0
```

這些數字（0、8、16）必須與 C 結構的記憶體佈局**完全一致**。

**常見錯誤**：
- 在 cpu_t 前面加一個 `char debug_flag` → offset 全部錯位 → 系統崩潰
- 改變欄位順序 → 同上

**教訓**：Trap Scratch Area 的欄位順序是**不可修改的契約**。

---

## 三、Trap Entry 完整流程

現在讓我們看看完整的 trap_entry 程式碼，理解每一步的意義：

### 3.1 從 User Mode 進入 Trap

```asm
trap_entry:
    # ══════════════════════════════════════════════════════════
    # Step 1: 身份切換
    # ══════════════════════════════════════════════════════════
    csrrw   tp, mscratch, tp    # tp = &cpu_t, mscratch = user_tp

    # 如果 tp = 0，表示我們本來就在 M-mode（kernel task）
    beqz    tp, .from_m_mode

    # ══════════════════════════════════════════════════════════
    # Step 2: 切換到 Kernel Stack
    # ══════════════════════════════════════════════════════════
    sd      sp, 8(tp)           # 保存 user_sp 到 cpu_t.user_sp_save
    ld      sp, 0(tp)           # 載入 kernel_sp from cpu_t.kernel_trap_sp

    # 現在我們有了安全的 Kernel Stack！
    addi    sp, sp, -CONTEXT_SIZE   # 分配 context frame (34 * 8 = 272 bytes)

    # ══════════════════════════════════════════════════════════
    # Step 3: 保存暫存器
    # ══════════════════════════════════════════════════════════
    sd      t0, REG_T0(sp)      # 先保存 t0（我們需要用它）
    ld      t0, 8(tp)           # 取回 user_sp
    sd      t0, REG_SP(sp)      # 保存 user_sp 到 context
    csrr    t0, mscratch        # 取得 user_tp（現在在 mscratch）
    sd      t0, REG_TP(sp)      # 保存 user_tp 到 context
    sd      t0, 16(tp)          # 也保存到 cpu_t.user_tp_save
    j       .save_context

.from_m_mode:
    # 本來就在 M-mode，恢復 tp 並用傳統方式處理
    csrrw   tp, mscratch, tp    # 恢復 tp = &cpu_t
    addi    sp, sp, -CONTEXT_SIZE
    sd      t0, REG_T0(sp)
    addi    t0, sp, CONTEXT_SIZE
    sd      t0, REG_SP(sp)      # 保存原本的 sp
    sd      tp, REG_TP(sp)      # 保存 tp (= &cpu_t)

.save_context:
    # 保存所有其他暫存器
    sd      ra,  REG_RA(sp)
    sd      gp,  REG_GP(sp)
    # ... (t1, t2, s0-s11, a0-a7, t3-t6) ...
    csrr    t0, mepc
    sd      t0, REG_MEPC(sp)
    csrr    t0, mstatus
    sd      t0, REG_MSTATUS(sp)

    # ══════════════════════════════════════════════════════════
    # Step 4: 呼叫 C trap handler
    # ══════════════════════════════════════════════════════════
    mv      a0, sp              # a0 = pointer to saved context
    call    trap_handler        # 返回值 a0 = new sp (可能是不同 task)
    mv      sp, a0              # 切換到新 task 的 stack
```

### 3.2 流程圖

```
User Mode                                    Kernel Mode
    │                                            │
    │ Trap (Timer/Ecall/...)                     │
    ▼                                            │
┌─────────────────────────────────────────────────────────────┐
│ csrrw tp, mscratch, tp                                      │
│   tp: User TLS → &cpu_t                                     │
│   mscratch: &cpu_t → User TLS                               │
└─────────────────────────────────────────────────────────────┘
    │                                            │
    ▼                                            │
┌─────────────────────────────────────────────────────────────┐
│ Save user sp → cpu_t.user_sp_save                           │
│ Load kernel sp ← cpu_t.kernel_trap_sp                       │
└─────────────────────────────────────────────────────────────┘
    │                                            │
    ▼                                            │
┌─────────────────────────────────────────────────────────────┐
│ Allocate context frame on kernel stack                      │
│ Save all registers                                          │
│ Call trap_handler(sp)                                       │
└─────────────────────────────────────────────────────────────┘
                                                 │
                                                 ▼
                                          Kernel 處理中
                                          (tp = &cpu_t)
```

---

## 四、Trap Exit：返回的藝術

Trap Exit 比 Entry 更複雜，因為我們需要處理**兩種不同的返回目標**：

### 4.1 檢查返回目標

```asm
trap_exit:
    # 恢復 mepc 和 mstatus
    ld      t0, REG_MEPC(sp)
    csrw    mepc, t0
    ld      t0, REG_MSTATUS(sp)
    csrw    mstatus, t0

    # ── 檢查將返回的特權級 ──
    # mstatus.MPP (bits 12:11) 決定 mret 返回的特權級
    # MPP = 0: User Mode
    # MPP = 3: Machine Mode
    srli    t1, t0, 11
    andi    t1, t1, 3
    bnez    t1, .restore_m_mode

.restore_u_mode:
    # ══════════════════════════════════════════════════════════
    # 返回 U-mode：設置 mscratch = &cpu_t
    # ══════════════════════════════════════════════════════════
    # 下次 User Mode 發生 trap 時，trap_entry 會執行
    # csrrw tp, mscratch, tp，需要 mscratch = &cpu_t
    csrw    mscratch, tp        # mscratch = &cpu_t

    # 恢復 User tp
    ld      tp, REG_TP(sp)
    j       .restore_common

.restore_m_mode:
    # ══════════════════════════════════════════════════════════
    # 返回 M-mode：設置 mscratch = 0 作為標記
    # ══════════════════════════════════════════════════════════
    # M-mode 任務發生 trap 時，trap_entry 會檢測到 mscratch = 0
    # 知道不需要做 tp 交換（tp 已經是 &cpu_t）
    csrw    mscratch, zero      # mscratch = 0
    # 注意：不恢復 tp！保持 tp = &cpu_t

.restore_common:
    # 恢復所有其他暫存器
    ld      ra,  REG_RA(sp)
    ld      t0,  REG_T0(sp)
    # ... (恢復其他暫存器) ...
    ld      sp,  REG_SP(sp)     # 恢復 sp (user sp 或 kernel sp)
    mret                         # 返回
```

### 4.2 為什麼 M-mode 任務需要特殊處理？

這是我在開發 v3p0 時踩過的坑。

**場景**：Monitor task（M-mode kernel 任務）在 Core 1 執行，被 Timer 中斷。

**問題**：
1. trap_entry 執行 `csrrw tp, mscratch, tp`
2. 因為這是 M-mode 任務，mscratch 原本是 0
3. 交換後 tp = 0，mscratch = &cpu_t
4. `beqz tp, .from_m_mode` 跳轉，恢復正確狀態
5. trap handler 處理完成
6. **trap_exit 錯誤地恢復 tp**
7. 從 context 載入的 tp = 0（因為 M-mode 任務不使用 User TLS）
8. **任務返回後 tp = 0，任何存取 cpu_t 的操作都崩潰！**

**解決方案**：trap_exit 根據 MPP 位決定是否恢復 tp。

---

## 五、初始化：設置舞台

在任務開始執行之前，我們需要正確設置 mscratch。

### 5.1 SMP 初始化

```c
void smp_init_bsp(void)
{
    cpu_t *cpu = &g_cpus[0];
    cpu->hartid = 0;
    cpu->current_task = NULL;
    cpu->idle_task = NULL;

    /* 設置 tp 指向這個核心的 cpu_t */
    asm volatile("mv tp, %0" :: "r"(cpu) : "memory");

    /* mscratch 初始為 0（表示 M-mode） */
    /* 當第一個 U-mode 任務開始執行時，
       trap_exit 會設置 mscratch = &cpu_t */
}
```

### 5.2 Context Switch 時更新 kernel_trap_sp

每次 Context Switch 到新 Task 時，需要更新 cpu_t.kernel_trap_sp：

```c
void switch_to(tcb_t *prev, tcb_t *next)
{
    cpu_t *cpu = smp_get_cpu();

    /* 更新當前任務指標 */
    cpu->current_task = next;

    /* 更新 kernel trap sp（如果是 U-mode 任務） */
    if (next->kstack_top != NULL) {
        cpu->kernel_trap_sp = (uint64_t)next->kstack_top;
    }

    /* 執行底層 context switch */
    __switch_context(&prev->sp, &next->sp);
}
```

**這確保了下次 trap 時，sp 會被正確載入為這個任務的 Kernel Stack。**

---

## 六、驗證：確保一切正常

### 6.1 測試案例

```c
/* 測試 M-mode + U-mode 混合任務 */
void test_mixed_tasks(void)
{
    /* M-mode 任務：可以直接存取硬體 */
    task_create("Monitor", monitor_func, NULL, 3,
                monitor_stack, sizeof(monitor_stack), CORE_1);

    /* U-mode 任務：只能透過 syscall */
    task_create_user("UserA", user_func_a, NULL, 2, CORE_ANY);
    task_create_user("UserB", user_func_b, NULL, 2, CORE_ANY);
}
```

### 6.2 GDB 除錯技巧

```gdb
# 在 trap_entry 設斷點
(gdb) break trap_entry

# 查看 tp 和 mscratch
(gdb) info registers tp
(gdb) x/gx $mscratch

# 確認 cpu_t 結構
(gdb) print *(cpu_t*)$tp

# 追蹤 context switch
(gdb) break switch_to
(gdb) commands
> print next->name
> print next->kstack_top
> continue
> end
```

---

## 七、本章總結

tp/mscratch 交換機制的核心要點：

| 概念 | 說明 |
|------|------|
| **csrrw 原子交換** | 一條指令完成身份切換，不需要臨時暫存器 |
| **cpu_t.Trap Scratch Area** | 前三個欄位 offset 固定，組合語言硬編碼 |
| **U-mode → Kernel** | mscratch 存 &cpu_t，交換後 tp = &cpu_t |
| **M-mode → Kernel** | mscratch = 0，trap_entry 檢測並跳過交換 |
| **返回 U-mode** | mscratch = &cpu_t，恢復 User tp |
| **返回 M-mode** | mscratch = 0，不恢復 tp |

**經驗法則**：
- Trap Scratch Area 的欄位順序**不可修改**
- 每次 context switch 需要更新 kernel_trap_sp
- trap_exit 必須根據 MPP 分歧處理

下一章，我們將探討 **Per-Task Kernel Stack** 的設計，解釋為什麼每個任務需要自己的 Kernel Stack。

---

## 參考資料

**RISC-V 規格**

- [RISC-V Privileged Specification](https://riscv.org/specifications/privileged-isa/)
  Chapter 3.1: Machine-Level CSRs (mscratch)

**danieRTOS 系列**

- Ch 22: Per-Core Data (v2.x tp 用法)
- Ch 13-15: User Mode 基礎 (v1.x mscratch 用法)

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: [tech-column-public](https://github.com/djiangtw/tech-column-public)

