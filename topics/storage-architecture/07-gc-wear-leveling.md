# Garbage Collection & Wear Leveling - SSD 的生命管理

**作者**: Danny Jiang  
**日期**: 2025-12-08  

---

## 前言：一次意外的壽命耗盡

2020 年，我在一家雲端服務公司負責儲存系統維護。有一天，監控系統發出警報：

**警報內容**：

```
SSD Health Alert:
Device: /dev/nvme0n1
Model: Samsung PM983 960GB (Enterprise SSD)
Status: CRITICAL

SMART Attributes:
- Percentage Used: 98%  ← 接近壽命終點
- Available Spare: 5%   ← 備用 Block 幾乎用盡
- Media Errors: 127     ← 開始出現錯誤

Estimated Remaining Life: 2-4 weeks
```

**震驚的發現**：

這顆 SSD 才使用 **18 個月**，但 SMART 顯示已經消耗了 98% 的壽命。按照規格，這顆企業級 SSD 應該可以用 5 年。

**調查結果**：

```
檢查 SMART 詳細資訊：

Data Units Written: 12,500 TB  ← 總寫入量
Host Writes: 3,200 TB          ← Host 實際寫入量

計算 WAF (Write Amplification Factor):
WAF = 12,500 TB / 3,200 TB = 3.9

問題：
- 這顆 SSD 的 WAF 高達 3.9
- 每寫入 1GB，NAND 實際寫入 3.9GB
- 壽命被「放大」消耗了 3.9 倍
```

**根本原因**：

這台伺服器運行的是 **資料庫工作負載**，特性是：

1. **隨機小寫入**：大量 4KB 隨機更新
2. **高更新頻率**：同一筆資料反覆修改
3. **沒有 TRIM**：資料庫直接管理 raw device，不經過檔案系統

這種負載模式導致：

- **GC 頻繁觸發**：大量 Invalid Pages 分散在各個 Block
- **Valid Page 搬運量大**：GC 時需要搬運大量有效資料
- **WAF 飆高**：NAND 寫入量遠超 Host 寫入量

這次經驗讓我深刻理解：**Garbage Collection 和 Wear Leveling 不只是演算法，它們直接決定了 SSD 的壽命和效能**。

本文將深入探討 GC 和 WL 的進階機制、演算法選擇，以及如何優化它們。

---

## 一、Garbage Collection 深度解析

### 1.1 GC 的觸發時機

**問題**：什麼時候應該執行 GC？

```
選項 1：被動 GC (Reactive GC)
- 觸發條件：Free Block 數量低於閾值
- 優點：不浪費資源
- 缺點：可能阻塞前台 I/O

選項 2：主動 GC (Proactive GC)
- 觸發條件：系統閒置時
- 優點：不影響前台 I/O
- 缺點：可能做無用功

選項 3：混合策略（實際採用）
- 閒置時：主動 GC，保持充足的 Free Blocks
- 忙碌時：被動 GC，只在必要時執行
```

**實際實作**：

```
FTL 的 GC 觸發邏輯：

1. 檢查 Free Block Pool
   if (free_blocks < LOW_WATERMARK) {
       // 緊急 GC，優先級高
       trigger_gc(PRIORITY_HIGH);
   }

2. 檢查系統負載
   if (io_queue_depth < IDLE_THRESHOLD) {
       // 閒置 GC，優先級低
       trigger_gc(PRIORITY_LOW);
   }

3. 檢查 Invalid Page 比例
   for each block in used_blocks {
       if (block.invalid_ratio > GC_THRESHOLD) {
           // 這個 Block 值得回收
           add_to_gc_candidate_list(block);
       }
   }
```

---

### 1.2 Victim Block Selection 演算法

**核心問題**：選擇哪個 Block 進行 GC？

#### **演算法 1: Greedy（貪婪）**

```
策略：選擇 Invalid Pages 最多的 Block

偽代碼：
victim = NULL
max_invalid = 0

for each block in used_blocks {
    if (block.invalid_pages > max_invalid) {
        max_invalid = block.invalid_pages
        victim = block
    }
}

優點：
- 簡單高效
- 回收效率高（搬運的 Valid Pages 最少）

缺點：
- 忽略 Block 的年齡（P/E Cycle）
- 可能導致磨損不均
```

**範例**：

```
Block A: 90% Invalid, 10% Valid, P/E = 100
Block B: 80% Invalid, 20% Valid, P/E = 1000
Block C: 70% Invalid, 30% Valid, P/E = 500

Greedy 選擇：Block A
- 理由：Invalid 比例最高
- 搬運成本：10% Valid Pages
- 問題：Block A 的 P/E 已經很低，繼續使用會加速磨損
```

#### **演算法 2: Cost-Benefit（成本效益）**

```
策略：平衡回收效率和 Block 年齡

公式：
benefit = (invalid_pages / total_pages) × block_age
cost = valid_pages

score = benefit / cost

選擇 score 最高的 Block

優點：
- 考慮磨損平衡
- 避免年輕 Block 過度使用

缺點：
- 計算複雜度較高
- 需要追蹤 Block 年齡
```

**範例**：

```
Block A: 90% Invalid, P/E = 100, age = 1
Block B: 80% Invalid, P/E = 1000, age = 10
Block C: 70% Invalid, P/E = 500, age = 5

計算 score：
Block A: (0.9 × 1) / 0.1 = 9.0
Block B: (0.8 × 10) / 0.2 = 40.0  ← 最高
Block C: (0.7 × 5) / 0.3 = 11.7

Cost-Benefit 選擇：Block B
- 理由：雖然 Invalid 比例較低，但年齡高，值得回收
- 效果：平衡回收效率和磨損平衡
```

#### **演算法 3: d-Choices（多選擇）**

```
策略：隨機選擇 d 個 Block，從中選最好的

偽代碼：
candidates = randomly_select(d)  // d = 2-4
victim = NULL
best_score = 0

for each block in candidates {
    score = calculate_score(block)
    if (score > best_score) {
        best_score = score
        victim = block
    }
}

優點：
- 避免掃描所有 Blocks（O(d) vs O(n)）
- 效果接近全局最優
- 實作簡單

缺點：
- 不保證全局最優
```

---

### 1.3 Hot/Cold Data Separation

**問題**：冷資料被 GC 反覆搬運，浪費 NAND 寫入

**範例場景**：

```
Block #100 的內容：
Page 0: [熱資料 A]  ← 每秒更新
Page 1: [冷資料 B]  ← 從不更新
Page 2: [熱資料 C]  ← 每秒更新
Page 3: [冷資料 D]  ← 從不更新

問題：
- 熱資料頻繁更新，產生大量 Invalid Pages
- Block #100 被選為 GC Victim
- 冷資料 B 和 D 被無辜搬運
- 下次 GC 又搬運一次
- 冷資料被反覆搬運，浪費 NAND 寫入
```

**解決方案：Multi-Streamed SSD**

```
概念：將資料分流到不同的 Streams

Stream 0: 熱資料（頻繁更新）
Stream 1: 溫資料（偶爾更新）
Stream 2: 冷資料（很少更新）

FTL 的處理：
- 不同 Stream 的資料寫入不同的 Blocks
- GC 時，熱資料 Block 的 Invalid 比例高，容易回收
- 冷資料 Block 的 Invalid 比例低，很少被 GC

效果：
- 減少冷資料搬運
- 降低 WAF
- 延長 SSD 壽命
```

**NVMe Streams 實作**：

```c
// NVMe Streams API 範例

// 開啟 Stream
nvme_directive_send(fd, NVME_DIR_STREAMS, NVME_DIR_SEND_ID_ENDIR,
                    stream_id, ...);

// 寫入時指定 Stream
struct nvme_user_io io = {
    .opcode = nvme_cmd_write,
    .slba = lba,
    .nblocks = nblocks,
    .dsmgmt = stream_id << 16,  // 指定 Stream ID
    ...
};

ioctl(fd, NVME_IOCTL_SUBMIT_IO, &io);
```

**實測效果**：

```
測試：RocksDB 資料庫負載

Without Streams:
- WAF: 3.8
- NAND Writes: 380 TB (Host Writes: 100 TB)
- SSD Lifetime: 2.1 years

With Streams (4 Streams):
- WAF: 1.9  ← 降低 50%
- NAND Writes: 190 TB
- SSD Lifetime: 4.2 years  ← 翻倍
```

---

## 二、Wear Leveling 深度解析

### 2.1 為什麼需要 Wear Leveling？

**NAND Flash 的壽命限制**：

```
P/E Cycle (Program/Erase Cycle) 限制：

SLC (Single-Level Cell):  ~100,000 次
MLC (Multi-Level Cell):   ~10,000 次
TLC (Triple-Level Cell):  ~3,000 次
QLC (Quad-Level Cell):    ~1,000 次

超過限制後：
- Bit Error Rate (BER) 急劇上升
- ECC 無法修正
- Block 變成 Bad Block
```

**問題場景**：

```
假設沒有 Wear Leveling：

使用者頻繁更新同一個檔案（如日誌檔）：
- LBA 1000-1100 被反覆寫入
- FTL 總是使用 Block #50 來存放這些資料
- Block #50 的 P/E Cycle 快速增加

結果：
- 6 個月後，Block #50 達到 3,000 次 P/E Cycle（TLC 極限）
- Block #50 變成 Bad Block
- 其他 Blocks 的 P/E Cycle 還不到 100 次
- SSD 容量減少，效能下降
```

**Wear Leveling 的目標**：

```
讓所有 Blocks 的 P/E Cycle 盡量接近

理想狀態：
Block #0:   P/E = 1500
Block #1:   P/E = 1500
Block #2:   P/E = 1500
...
Block #999: P/E = 1500

實際狀態（有 WL）：
Block #0:   P/E = 1450
Block #1:   P/E = 1520
Block #2:   P/E = 1480
...
Block #999: P/E = 1510

差異：< 5%（可接受）
```

---

### 2.2 Dynamic Wear Leveling（動態磨損平衡）

**策略**：針對熱資料（頻繁更新的資料）

```
原理：
- 當需要分配 Free Block 時
- 選擇 P/E Cycle 最低的 Free Block
- 避免重複使用同一個 Block

實作：
free_block_pool = sorted_by_pe_cycle(free_blocks)

allocate_block() {
    // 選擇 P/E Cycle 最低的 Free Block
    return free_block_pool.pop_front();
}
```

**範例**：

```
Free Block Pool（按 P/E Cycle 排序）：
Block #10: P/E = 100
Block #20: P/E = 150
Block #30: P/E = 200
Block #40: P/E = 250

Host 寫入新資料：
1. FTL 分配 Block #10（P/E 最低）
2. 寫入資料
3. Block #10 的 P/E 變成 101
4. Block #10 用完後，加入 Free Block Pool 尾端

下次分配：
- 選擇 Block #20（現在 P/E 最低）
```

**優點**：

```
- 實作簡單
- 開銷小
- 對熱資料有效
```

**缺點**：

```
- 對冷資料無效
- 冷資料可能一直佔據年輕的 Blocks
```

---

### 2.3 Static Wear Leveling（靜態磨損平衡）

**問題**：冷資料佔據年輕 Blocks，導致磨損不均

**範例場景**：

```
Block #100:
- 存放作業系統檔案（冷資料，從不更新）
- P/E Cycle: 50（非常年輕）
- 佔據這個 Block 已經 2 年

其他 Blocks:
- 存放使用者資料（熱資料，頻繁更新）
- P/E Cycle: 2500-2800（接近 TLC 極限）

問題：
- Block #100 的 P/E 只有 50，但一直被冷資料佔據
- 其他 Blocks 快要壽終正寢
- 磨損極度不均
```

**Static WL 的策略**：

```
定期檢查：
- 找出 P/E Cycle 最低的 Block（如 Block #100）
- 找出 P/E Cycle 最高的 Free Block（如 Block #200）

如果差異過大（如 > 500）：
1. 將 Block #100 的冷資料搬運到 Block #200
2. 擦除 Block #100
3. Block #100 加入 Free Block Pool（P/E = 51）
4. Block #100 可以被熱資料使用

效果：
- 年輕的 Block 被釋放出來，給熱資料使用
- 年老的 Block 存放冷資料，減少擦除次數
- 磨損更均勻
```

**實作挑戰**：

```
1. 何時觸發 Static WL？
   - 太頻繁：浪費 NAND 寫入（搬運冷資料）
   - 太少：磨損不均

2. 如何識別冷資料？
   - 追蹤 Block 的最後更新時間
   - 超過閾值（如 1 個月）視為冷資料

3. 搬運成本
   - Static WL 會增加 WAF
   - 需要在磨損平衡和 WAF 之間權衡
```

---

### 2.4 Wear Leveling 的實測效果

**測試場景**：模擬不均勻負載

```
負載特性：
- 80% 寫入集中在 20% 的 LBA 範圍（熱資料）
- 20% 寫入分散在 80% 的 LBA 範圍（冷資料）

測試 1：無 Wear Leveling
- 6 個月後，20% 的 Blocks 達到 P/E 極限
- SSD 容量降低 20%
- 開始出現 Bad Blocks

測試 2：僅 Dynamic WL
- 12 個月後，熱資料 Blocks 達到 P/E 極限
- 冷資料 Blocks 的 P/E 還很低
- 磨損不均，但比測試 1 好

測試 3：Dynamic + Static WL
- 24 個月後，所有 Blocks 的 P/E 接近
- 差異 < 10%
- SSD 壽命最大化
```

**P/E Cycle 分佈圖**：

#### 1. 無 WL (No Wear Leveling)

**特性：** 集中抹寫特定區域，80% 的區塊幾乎處於閒置狀態。

```text
P/E Cycle
      ^
 3000 | ████████
 2500 | ████████
 2000 | ████████
 1500 | ████████
 1000 | ████████
  500 | ██████████████████████████████
    0 | ██████████████████████████████
      +------------------------------->
        |<- 20% ->|<-      80%      ->|
              常用區塊          閒置區塊 (Blocks)
```

#### 2. 動態 WL (Dynamic Wear Leveling)

**特性：** 只對「新資料」進行平衡，存放「冷資料」的區塊依然沒被抹寫。

```text
P/E Cycle
      ^
 3000 | ██████
 2500 | ████████████
 2000 | ████████████
 1500 | ████████████
 1000 | ████████████
  500 | ██████████████████
    0 | ██████████████████
      +------------------------------->
        |<- 動態區 ->|<- 靜態/冷資料區 ->|
            (變動中)      (從未抹寫)  (Blocks)
```

#### 3. 動態 + 靜態 WL (Dynamic + Static WL)

**特性：** 強制交換冷熱資料，所有區塊磨損極度均勻，壽命最大化。

```text
P/E Cycle
      ^
 3000 |
 2500 | ██████████████████████████████
 2000 | ██████████████████████████████
 1500 | ██████████████████████████████
 1000 |
  500 |
    0 |
      +------------------------------->
        |<-      全區塊均勻磨損      ->|
                                      (Blocks)
```

---

## 三、GC 與 WL 的協同優化

### 3.1 GC 和 WL 的衝突

**問題**：GC 和 WL 的目標有時會衝突

```
GC 的目標：
- 選擇 Invalid 比例最高的 Block
- 最小化 Valid Page 搬運
- 降低 WAF

WL 的目標：
- 平衡所有 Blocks 的 P/E Cycle
- 可能需要搬運冷資料（增加 WAF）

衝突場景：
Block A: 90% Invalid, P/E = 100（年輕）
Block B: 70% Invalid, P/E = 2500（年老）

GC 想選 Block A（回收效率高）
WL 想選 Block B（避免過度磨損）
```

**解決方案：統一的 Cost Function**

```
設計一個綜合評分函數：

score = α × (invalid_ratio) + β × (pe_cycle / max_pe_cycle)

其中：
- α: GC 權重（如 0.7）
- β: WL 權重（如 0.3）
- invalid_ratio: Invalid Pages 比例
- pe_cycle: 當前 Block 的 P/E Cycle
- max_pe_cycle: 所有 Blocks 中最高的 P/E Cycle

選擇 score 最高的 Block
```

**範例計算**：

```
Block A: 90% Invalid, P/E = 100
Block B: 70% Invalid, P/E = 2500
max_pe_cycle = 2500

Block A score:
= 0.7 × 0.9 + 0.3 × (100 / 2500)
= 0.63 + 0.012
= 0.642

Block B score:
= 0.7 × 0.7 + 0.3 × (2500 / 2500)
= 0.49 + 0.3
= 0.79  ← 更高

選擇 Block B：
- 雖然 Invalid 比例較低
- 但 P/E Cycle 高，需要優先處理
- 平衡 GC 效率和 WL
```

---

### 3.2 Over-Provisioning 的作用

**Over-Provisioning (OP)**：保留一部分容量不給使用者

```
範例：1TB SSD

OP = 7%（消費級）：
- 使用者可用：930GB
- 保留：70GB（用於 GC 和 WL）

OP = 28%（企業級）：
- 使用者可用：720GB
- 保留：280GB
```

**OP 對 GC 的影響**：

```
低 OP (7%)：
- Free Blocks 少
- GC 頻繁觸發
- Valid Page 搬運量大
- WAF 高（3-5）

高 OP (28%)：
- Free Blocks 多
- GC 較少觸發
- 可以選擇 Invalid 比例更高的 Block
- WAF 低（1.5-2）
```

**OP 對 WL 的影響**：

```
低 OP：
- Free Block Pool 小
- 選擇餘地少
- 難以平衡 P/E Cycle

高 OP：
- Free Block Pool 大
- 可以選擇 P/E Cycle 低的 Block
- WL 效果更好
```

**實測數據**：

```
測試：隨機寫入負載，持續 1 年

OP = 7%：
- WAF: 4.2
- P/E Cycle 標準差: 450（磨損不均）
- 預估壽命: 1.8 years

OP = 15%：
- WAF: 2.8
- P/E Cycle 標準差: 280
- 預估壽命: 2.7 years

OP = 28%：
- WAF: 1.9
- P/E Cycle 標準差: 120（磨損均勻）
- 預估壽命: 4.0 years
```

---

## 四、進階主題：ZNS 如何消滅 GC

### 4.1 傳統 SSD 的根本問題

**問題**：FTL 不知道 Host 的意圖

```
Host 的視角：
- 刪除檔案 A（100GB）
- 寫入檔案 B（100GB）
- 檔案 A 和 B 的生命週期完全不同

FTL 的視角：
- 看到一堆 LBA 被更新
- 不知道哪些是檔案 A，哪些是檔案 B
- 可能把 A 和 B 的資料混在同一個 Block
- GC 時被迫搬運不相關的資料
```

**結果**：

```
- FTL 做了很多無用功
- WAF 增加
- 效能下降
```

---

### 4.2 ZNS (Zoned Namespace) 的解決方案

**核心理念**：讓 Host 管理資料放置

```
ZNS 的規則：

1. SSD 被分成數千個 Zones（如 256MB 每個）
2. 每個 Zone 必須循序寫入
3. 要修改資料，必須先 Reset 整個 Zone

Host 的責任：
- 決定哪些資料放在哪個 Zone
- 管理資料的生命週期
- 決定何時 Reset Zone

FTL 的責任：
- 簡化到極致
- 只需要 Zone-level Mapping（不需要 Page-level）
- 幾乎沒有 GC（Host 負責）
```

**效果**：

```
傳統 SSD：
- FTL 複雜，需要 1GB DRAM
- GC 頻繁，WAF = 3-4
- 效能波動大

ZNS SSD：
- FTL 極簡，只需 2MB SRAM
- 幾乎沒有 GC，WAF ≈ 1
- 效能穩定，可預測
```

**實際應用：RocksDB on ZNS**

```
RocksDB 的特性：
- LSM-Tree 結構，本身就是循序寫入
- SSTable 檔案一旦寫入就不修改
- Compaction 時刪除舊 SSTable，寫入新 SSTable

在 ZNS SSD 上：
- 每個 SSTable 寫入一個 Zone
- SSTable 刪除時，Reset 對應的 Zone
- 完全沒有 GC
- WAF = 1.0

效果：
- SSD 壽命翻倍
- 效能提升 30%
- 成本降低（不需要 DRAM）
```

---

## 五、給系統軟體工程師的建議

### 5.1 如何降低 WAF

**1. 使用 TRIM**

```bash
# Linux: 定期執行 fstrim
sudo fstrim -av

# 或啟用自動 TRIM
sudo systemctl enable fstrim.timer
```

**2. 避免填滿 SSD**

```
建議：保留至少 20% 空間

原因：
- 更多 Free Blocks → GC 壓力小
- WAF 降低
- 效能更穩定
```

**3. 批次寫入**

```
不好的模式：
for (i = 0; i < 1000000; i++) {
    write(fd, &data[i], 4KB);  // 100 萬次 4KB 寫入
}

好的模式：
write(fd, data, 4GB);  // 一次 4GB 寫入

原因：
- 大 I/O 更容易對齊 NAND Page
- 減少 FTL 的映射開銷
- 降低 WAF
```

**4. 使用 Direct I/O**

```c
// 繞過 Page Cache，直接寫入 SSD
int fd = open("/dev/nvme0n1", O_DIRECT | O_SYNC);
```

---

### 5.2 監控 SSD 健康度

**關鍵 SMART 指標**：

```bash
# 查看 SMART 資訊
sudo smartctl -a /dev/nvme0n1

# 關鍵指標：
# 1. Percentage Used（已使用壽命百分比）
#    - < 80%: 健康
#    - 80-95%: 警告
#    - > 95%: 危險

# 2. Available Spare（可用備用空間）
#    - > 50%: 健康
#    - 10-50%: 警告
#    - < 10%: 危險

# 3. Data Units Written（總寫入量）
#    - 用於計算 WAF

# 4. Media and Data Integrity Errors（媒體錯誤）
#    - 應該為 0
#    - > 0: 開始出現 Bad Blocks
```

**計算 WAF**：

```bash
# 從 SMART 資訊中提取
host_writes=$(smartctl -a /dev/nvme0n1 | grep "Data Units Written" | awk '{print $4}')
nand_writes=$(smartctl -a /dev/nvme0n1 | grep "NAND Writes" | awk '{print $3}')

# 計算 WAF
waf=$(echo "scale=2; $nand_writes / $host_writes" | bc)
echo "WAF: $waf"

# 評估：
# WAF < 2: 優秀
# WAF 2-3: 良好
# WAF 3-5: 一般
# WAF > 5: 需要優化
```

---

## 六、總結

### 6.1 核心要點回顧

**Garbage Collection**：

1. **Victim Selection 演算法**
   - Greedy：簡單高效，但忽略磨損
   - Cost-Benefit：平衡效率和磨損
   - d-Choices：折衷方案

2. **Hot/Cold Separation**
   - Multi-Streamed SSD
   - 降低 WAF 50%

3. **觸發時機**
   - 被動 GC：Free Blocks 不足時
   - 主動 GC：系統閒置時

**Wear Leveling**：

1. **Dynamic WL**
   - 針對熱資料
   - 選擇 P/E Cycle 低的 Free Block

2. **Static WL**
   - 針對冷資料
   - 定期搬運冷資料，釋放年輕 Block

3. **協同優化**
   - 統一的 Cost Function
   - 平衡 GC 效率和 WL

---

### 6.2 下一篇預告

在下一篇文章中，我們將探討：

- **Error Correction (ECC/LDPC)**
- **Hard Decision vs Soft Decision**
- **Read Retry 機制**
- **Die-Level RAID**

理解 GC 和 WL，能幫助我們更好地使用 SSD，延長其壽命，優化效能。

---

**參考資料**：

1. "The Multi-streamed Solid-State Drive", HotStorage 2014
2. "ZNS: Avoiding the Block Interface Tax for Flash-based SSDs", USENIX ATC 2021
3. "Design Tradeoffs for SSD Performance", USENIX ATC 2008
4. "Understanding the Robustness of SSDs under Power Fault", FAST 2013

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
