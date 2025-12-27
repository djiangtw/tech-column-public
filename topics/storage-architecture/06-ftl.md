# Flash Translation Layer (FTL) - 地址轉換的藝術

**作者**: Danny Jiang  
**日期**: 2025-12-08  

---

## 前言：一次神秘的效能衰退

2019 年，我在一家儲存系統公司負責 SSD 性能測試。有一天，測試工程師拿著一份報告來找我：

**問題現象**：

```
測試場景：循序寫入 100GB 資料到 1TB SSD

第一次測試（全新 SSD）：
- 寫入速度：3,500 MB/s（穩定）
- 完成時間：29 秒

第二次測試（同一顆 SSD，已使用 50%）：
- 前 10GB：3,500 MB/s
- 10-50GB：2,000 MB/s  ← 突然掉速
- 50-100GB：800 MB/s   ← 繼續惡化
- 完成時間：95 秒（慢了 3 倍）
```

測試工程師困惑地問：「這顆 SSD 是不是壞了？為什麼越用越慢？」

我當時的第一反應是檢查 SMART 資訊，但所有指標都正常。後來我意識到，這不是硬體故障，而是 **Flash Translation Layer (FTL)** 的正常行為。

**真相**：

1. **第一次測試**：SSD 內部全是空白 Block，FTL 直接寫入，速度飛快
2. **第二次測試**：SSD 內部有 50% 的資料，FTL 必須先做 **Garbage Collection (GC)**，搬運有效資料，才能釋放空間

這次經驗讓我深刻理解：**FTL 是 SSD 的靈魂，它決定了 SSD 的性能、壽命和可靠性**。

本文將深入探討 FTL 的設計原理、核心機制，以及它如何在幕後默默地管理 NAND Flash。

---

## 一、為什麼需要 FTL？

### 1.1 NAND Flash 的物理限制

**作業系統的期望**：硬碟是可以隨意覆蓋寫入的線性空間

```
OS 的視角：
LBA 0: [Data A]  → 更新 → LBA 0: [Data A']  ← 直接覆蓋
LBA 1: [Data B]  → 更新 → LBA 1: [Data B']
LBA 2: [Data C]  → 更新 → LBA 2: [Data C']

期望：像 HDD 一樣，想寫哪裡就寫哪裡
```

**NAND Flash 的現實**：完全不同的物理特性

```
NAND Flash 的限制：

1. 寫入前必須先擦除 (Erase-before-Write)
   - 不能直接覆蓋已有資料的 Page
   - 必須先擦除整個 Block，才能重新寫入

2. 讀寫單位 vs 擦除單位不同
   - 讀取單位：Page (4KB, 8KB, 16KB)
   - 寫入單位：Page (4KB, 8KB, 16KB)
   - 擦除單位：Block (256KB, 512KB, 1MB, 2MB)
   - 一個 Block 包含 64-256 個 Pages

3. 有限的擦除次數 (P/E Cycle)
   - SLC：~100,000 次
   - MLC：~10,000 次
   - TLC：~3,000 次
   - QLC：~1,000 次
```

**矛盾的核心**：

```
OS 說：「我要更新 LBA 100 的 4KB 資料」
NAND Flash 說：「我不能直接覆蓋，我必須：
  1. 找一個空白的 Page 寫入新資料
  2. 標記舊的 Page 為無效 (Invalid)
  3. 等到整個 Block 都是無效 Page 時，才能擦除」
```

**FTL 的使命**：掩蓋這個矛盾，讓 NAND Flash 看起來像 HDD 一樣可以隨意覆蓋。

---

### 1.2 FTL 的核心任務

**FTL (Flash Translation Layer)** 是運行在 SSD Controller 上的韌體核心，負責：

1. **地址映射 (Address Mapping)**
   - 將 OS 的邏輯地址 (LBA) 轉換為 NAND Flash 的實體地址 (PBA)
   - 維護 L2P (Logical-to-Physical) Mapping Table

2. **垃圾回收 (Garbage Collection, GC)**
   - 回收無效 Page，釋放空間
   - 搬運有效 Page 到新 Block
   - 擦除舊 Block，加入 Free Block List

3. **磨損平衡 (Wear Leveling, WL)**
   - 平均分配擦除次數到所有 Block
   - 避免某些 Block 過早損壞

4. **壞塊管理 (Bad Block Management)**
   - 偵測並標記壞塊
   - 使用備用 Block 替換

**FTL 的位置**：

```
┌─────────────────────────────────────────┐
│          Host (OS / Application)        │
└─────────────────┬───────────────────────┘
                  │ LBA (Logical Block Address)
                  ↓
┌─────────────────────────────────────────┐
│         NVMe Driver (PCIe)              │
└─────────────────┬───────────────────────┘
                  │
                  ↓
┌─────────────────────────────────────────┐
│         SSD Controller                  │
│  ┌───────────────────────────────────┐  │
│  │  Flash Translation Layer (FTL)    │  │ ← 這裡！
│  │  - L2P Mapping                    │  │
│  │  - Garbage Collection             │  │
│  │  - Wear Leveling                  │  │
│  └───────────────────────────────────┘  │
└─────────────────┬───────────────────────┘
                  │ PBA (Physical Block Address)
                  ↓
┌─────────────────────────────────────────┐
│         NAND Flash (Physical Media)     │
└─────────────────────────────────────────┘
```

---

## 二、L2P Mapping - 地址轉換的核心

### 2.1 Out-of-Place Update

**傳統 HDD 的 In-Place Update**：

```
HDD 的更新方式：
LBA 100: [Data A]  → 更新 → LBA 100: [Data A']
         ↑                           ↑
         同一個物理位置，直接覆蓋
```

**SSD 的 Out-of-Place Update**：

```
SSD 的更新方式：
LBA 100: [Data A] (PBA 500)  → 更新 → LBA 100: [Data A'] (PBA 800)
         ↑                                      ↑
         舊位置標記為 Invalid                   寫入新位置

L2P Mapping Table 更新：
Before: LBA 100 → PBA 500
After:  LBA 100 → PBA 800

PBA 500 的 Page 變成 Invalid（垃圾）
```

**為什麼要 Out-of-Place Update？**

因為 NAND Flash 不能直接覆蓋，必須先擦除。但擦除的單位是 Block（包含數百個 Pages），如果每次更新 4KB 就擦除整個 Block（可能 2MB），效率極低且會快速耗盡 P/E Cycle。

---

### 2.2 L2P Mapping Table 的實作

**Page-Level Mapping（頁級映射）**：

```
最細粒度的映射：每個 4KB Page 都有獨立的映射

L2P Table 結構：
LBA 0    → PBA 1000
LBA 1    → PBA 1001
LBA 2    → PBA 1005  ← 不連續，因為 Out-of-Place Update
LBA 3    → PBA 1002
...
LBA N    → PBA XXXX

優點：
- 靈活性最高
- 可以任意映射

缺點：
- 記憶體需求巨大
```

**記憶體需求計算**：

```
假設：
- SSD 容量：1TB
- Page 大小：4KB
- 每個映射項：4 Bytes (32-bit PBA)

計算：
- 總 Page 數：1TB / 4KB = 256M (2^28) Pages
- L2P Table 大小：256M × 4 Bytes = 1GB

結論：1TB SSD 需要 1GB DRAM 來存放 L2P Table！
```

**實際的優化策略**：

```
1. Hybrid Mapping（混合映射）
   - 熱資料：Page-Level Mapping（快速）
   - 冷資料：Block-Level Mapping（省記憶體）

2. Demand Paging（按需載入）
   - L2P Table 的一部分存在 DRAM
   - 其餘部分存在 NAND Flash
   - 需要時才載入（類似 OS 的 Virtual Memory）

3. Caching（快取）
   - 使用 LRU/LFU 演算法
   - 只快取最常用的映射項

4. Compression（壓縮）
   - 利用映射的局部性
   - 壓縮連續的映射項
```

---

### 2.3 L2P Table 的持久化

**問題**：L2P Table 存在 DRAM（揮發性記憶體），斷電會消失

**解決方案**：

```
1. 定期保存 (Periodic Flush)
   - 每隔一段時間（如 10 秒）將 L2P Table 寫入 NAND
   - 缺點：斷電可能丟失最近 10 秒的映射

2. 斷電保護 (Power Loss Protection, PLP)
   - 使用電容儲能
   - 偵測到斷電時，立即將 L2P Table 寫入 NAND
   - 企業級 SSD 的標配

3. Journaling（日誌）
   - 記錄映射的變更（類似資料庫的 WAL）
   - 重啟後重放日誌，重建 L2P Table
```

---

## 三、Garbage Collection - 垃圾回收的藝術

### 3.1 為什麼需要 GC？

**問題場景**：

```
初始狀態：SSD 有 1000 個 Blocks，全部空白

使用者寫入 500GB 資料：
- 使用了 500 個 Blocks
- 剩餘 500 個 Free Blocks

使用者更新其中 200GB 資料（Out-of-Place Update）：
- 新資料寫入 200 個新的 Free Blocks
- 舊資料的 200 個 Blocks 變成「部分無效」
  （有些 Pages 是 Invalid，有些還是 Valid）

現在的狀態：
- 完全有效的 Blocks：300 個
- 部分無效的 Blocks：200 個（混合 Valid 和 Invalid Pages）
- 完全空白的 Blocks：300 個

問題：
- 使用者實際資料：500GB
- 但 SSD 內部已經沒有完整的 Free Block 可以直接使用
- 必須先「整理」那 200 個部分無效的 Blocks
```

**GC 的目標**：回收 Invalid Pages，釋放完整的 Free Blocks

---

### 3.2 GC 的運作流程

**Step 1: Victim Block Selection（選擇受害者 Block）**

```
選擇策略：
1. Greedy（貪婪）
   - 選擇 Invalid Pages 最多的 Block
   - 優點：回收效率高
   - 缺點：可能忽略磨損平衡

2. Cost-Benefit（成本效益）
   - 考慮 Block 的年齡（P/E Cycle）
   - 平衡回收效率和磨損平衡

範例：
Block A: 90% Invalid, 10% Valid, P/E = 100
Block B: 70% Invalid, 30% Valid, P/E = 1000

Greedy 選 Block A（Invalid 多）
Cost-Benefit 可能選 Block B（避免 Block A 過度磨損）
```

**Step 2: Valid Page Migration（有效 Page 搬運）**

```
假設選中 Block #100 進行 GC：

Block #100 的狀態：
Page 0: [Valid Data]    ← 需要搬運
Page 1: [Invalid]       ← 可以丟棄
Page 2: [Valid Data]    ← 需要搬運
Page 3: [Invalid]       ← 可以丟棄
...

GC 動作：
1. 讀取 Page 0 的資料
2. 寫入到新的 Free Block（如 Block #500, Page 0）
3. 更新 L2P Table：LBA X → PBA (500, 0)
4. 讀取 Page 2 的資料
5. 寫入到 Block #500, Page 1
6. 更新 L2P Table：LBA Y → PBA (500, 1)
```

**Step 3: Block Erase（擦除 Block）**

```
所有 Valid Pages 搬運完成後：

1. 擦除 Block #100
   - 所有 Pages 變成空白
   - P/E Cycle 計數器 +1

2. 加入 Free Block List
   - Block #100 可以重新使用
```

---

### 3.3 Write Amplification Factor (WAF)

**定義**：

```
WAF = NAND 實際寫入量 / Host 寫入量

理想情況：WAF = 1（Host 寫 1GB，NAND 也寫 1GB）
實際情況：WAF = 2-5（Host 寫 1GB，NAND 寫 2-5GB）
```

**WAF 的來源**：

```
1. GC 的 Valid Page 搬運
   - Host 寫入 1GB 新資料
   - GC 搬運 1GB Valid Pages
   - 總共寫入 2GB → WAF = 2

2. Over-Provisioning (OP)
   - SSD 保留 7-28% 的空間不給使用者
   - 用於 GC 和 Wear Leveling
   - 降低 WAF

3. 資料更新模式
   - 隨機更新：WAF 高（3-5）
   - 循序更新：WAF 低（1-2）
```

**降低 WAF 的策略**：

```
1. 增加 Over-Provisioning
   - 更多 Free Blocks → GC 壓力小 → WAF 低

2. 優化 GC 演算法
   - 選擇 Invalid 比例高的 Block
   - 減少 Valid Page 搬運

3. 資料分類（Hot/Cold Separation）
   - 熱資料（頻繁更新）放在一起
   - 冷資料（很少更新）放在一起
   - 避免冷資料被 GC 反覆搬運

4. Host-Aware（讓 Host 協助）
   - TRIM 命令：告訴 SSD 哪些資料已刪除
   - ZNS (Zoned Namespace)：Host 管理資料放置
```

---

## 四、實際案例：FTL 如何影響效能

### 4.1 案例一：SLC Cache 與 Folding

**背景**：現代消費級 SSD 使用 SLC Cache 技術

```
SLC Cache 機制：
- 將 TLC/QLC Block 以 SLC 模式使用（1 bit/cell）
- 寫入速度：7,000 MB/s (SLC) vs 1,000 MB/s (TLC)
- Cache 大小：通常 10-50GB（動態調整）

Folding（摺疊）：
- 閒置時，FTL 將 SLC Cache 的資料搬回 TLC
- 釋放 SLC Cache 空間
```

**效能表現**：

```
測試：寫入 100GB 資料到 1TB TLC SSD

階段 1：前 30GB（SLC Cache 內）
- 寫入速度：3,500 MB/s
- 使用者體驗：極快

階段 2：30-100GB（SLC Cache 滿）
- FTL 必須邊寫入、邊 Folding
- 寫入速度：800 MB/s  ← 懸崖式掉速
- 使用者體驗：突然變慢

原因：
- SLC Cache 滿了
- FTL 必須直接寫入 TLC（慢）
- 同時執行 Folding（佔用 NAND 頻寬）
```

---

### 4.2 案例二：TRIM 的重要性

**沒有 TRIM 的情況**：

```
使用者刪除 100GB 檔案：

OS 的視角：
- 檔案系統標記這些 LBA 為「未使用」
- 但不會通知 SSD

SSD 的視角：
- L2P Table 仍然指向這些 PBA
- FTL 認為這些 Pages 是 Valid
- GC 時會浪費時間搬運這些「已刪除」的資料

結果：
- WAF 增加
- GC 效率降低
- 效能衰退
```

**有 TRIM 的情況**：

```
使用者刪除 100GB 檔案：

OS 發送 TRIM 命令：
- 告訴 SSD 這些 LBA 已經不需要了

SSD 的動作：
- 標記對應的 PBA 為 Invalid
- L2P Table 移除這些映射
- GC 時可以直接跳過這些 Pages

結果：
- WAF 降低
- GC 效率提升
- 效能維持穩定
```

**實測數據**：

```
測試環境：
- 1TB SSD，已使用 80%
- 刪除 200GB 檔案
- 執行 fstrim（Linux TRIM 命令）

Before TRIM:
- 隨機寫入 IOPS：50,000
- 平均延遲：20 μs

After TRIM:
- 隨機寫入 IOPS：120,000  ← 提升 2.4 倍
- 平均延遲：8 μs          ← 降低 60%

原因：
- FTL 知道哪些 Blocks 可以直接使用
- 減少 GC 的觸發頻率
- 降低 WAF
```

---

### 4.3 案例三：Over-Provisioning 的影響

**測試**：比較不同 OP 比例的效能

```
測試 SSD：1TB 容量，TLC NAND

配置 1：OP = 7%（標準消費級）
- 使用者可用容量：930GB
- 穩態隨機寫入 IOPS：80,000
- WAF：3.5

配置 2：OP = 28%（企業級）
- 使用者可用容量：720GB
- 穩態隨機寫入 IOPS：200,000  ← 提升 2.5 倍
- WAF：1.8                      ← 降低 49%

原因：
- 更多 Free Blocks → GC 壓力小
- FTL 有更多空間選擇最佳的 Block
- Valid Page 搬運量減少
```

---

## 五、FTL 的設計挑戰與權衡

### 5.1 記憶體 vs 效能

**挑戰**：L2P Table 需要大量 DRAM

```
權衡選項：

1. 大 DRAM（企業級）
   - 優點：全部 L2P Table 在記憶體，速度快
   - 缺點：成本高（1TB SSD 需要 1-2GB DRAM）

2. 小 DRAM + Demand Paging（消費級）
   - 優點：成本低
   - 缺點：L2P Table 缺頁時需要從 NAND 讀取，延遲增加

3. DRAM-less + HMB（低成本）
   - 優點：成本最低
   - 缺點：依賴 Host DRAM，相容性問題
```

---

### 5.2 效能 vs 壽命

**挑戰**：GC 會增加 NAND 寫入量，縮短壽命

```
權衡選項：

1. 激進 GC（效能優先）
   - 頻繁執行 GC，保持大量 Free Blocks
   - 優點：效能穩定
   - 缺點：WAF 高，壽命短

2. 保守 GC（壽命優先）
   - 只在必要時執行 GC
   - 優點：WAF 低，壽命長
   - 缺點：效能波動大

3. 自適應 GC（平衡）
   - 根據負載動態調整 GC 策略
   - 閒置時積極 GC
   - 忙碌時延遲 GC
```

---

### 5.3 一致性 vs 延遲

**挑戰**：FTL 的背景作業會影響前台 I/O

```
權衡選項：

1. 消費級 FTL
   - 背景 GC 優先級低
   - 前台 I/O 可能被 GC 阻塞
   - 延遲波動大（P99 可能 > 100ms）

2. 企業級 FTL
   - 支援 Preemption（搶佔）
   - GC 可以被前台 I/O 中斷
   - 延遲穩定（P99 < 10ms）
   - 需要更複雜的排程器
```

---

## 六、總結：FTL 是 SSD 的靈魂

### 6.1 核心要點回顧

**FTL 的三大核心機制**：

1. **L2P Mapping（地址映射）**
   - 將 LBA 轉換為 PBA
   - 實現 Out-of-Place Update
   - 需要大量 DRAM

2. **Garbage Collection（垃圾回收）**
   - 回收 Invalid Pages
   - 釋放 Free Blocks
   - 影響 WAF 和效能

3. **Wear Leveling（磨損平衡）**
   - 平均分配擦除次數
   - 延長 SSD 壽命
   - 下一篇文章詳細探討

---

### 6.2 FTL 對效能的影響

**關鍵因素**：

```
1. SLC Cache 大小
   - 影響爆發效能
   - Cache 滿後的掉速程度

2. Over-Provisioning 比例
   - 影響穩態效能
   - 影響 WAF 和壽命

3. GC 演算法
   - 影響延遲穩定性
   - 影響 NAND 寫入量

4. TRIM 支援
   - 影響長期效能衰退
   - 影響 GC 效率
```

---

### 6.3 給系統軟體工程師的建議

**如何與 FTL 和平共處**：

1. **定期執行 TRIM**

   ```bash
   # Linux
   sudo fstrim -av

   # 或設定定時任務
   sudo systemctl enable fstrim.timer
   ```

2. **避免填滿 SSD**
   - 保留至少 10-20% 空間
   - 給 FTL 足夠的 Free Blocks

3. **理解效能特性**
   - 循序寫入 > 隨機寫入
   - 大 I/O > 小 I/O
   - 讀取 > 寫入

4. **監控 SMART 資訊**

   ```bash
   # 查看 SSD 健康度
   sudo smartctl -a /dev/nvme0n1

   # 關注指標：
   # - Percentage Used（已使用壽命百分比）
   # - Data Units Written（總寫入量）
   # - Media and Data Integrity Errors（錯誤計數）
   ```

---

### 6.4 下一篇預告

在下一篇文章中，我們將深入探討：

- **Garbage Collection 的進階演算法**
- **Wear Leveling 的實作細節**
- **Hot/Cold Data Separation**
- **ZNS (Zoned Namespace) 如何消滅 FTL**

FTL 是 SSD 最複雜也最迷人的部分，理解它的運作原理，能幫助我們更好地使用和優化 SSD。

---

## 參考資料

1. "Design Tradeoffs for SSD Performance", USENIX ATC 2008
2. "The Multi-streamed Solid-State Drive", HotStorage 2014
3. NVMe Specification 1.4, Section 6 (Dataset Management)
4. "Understanding the Robustness of SSDs under Power Fault", FAST 2013

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
