# Database Optimization - 資料庫的存儲優化

**作者**: Danny Jiang  
**日期**: 2025-12-09  

---

## 前言：一次驚心動魄的資料庫效能危機

2018 年，我負責維護一個電商平台的 MySQL 資料庫。某天凌晨 2 點，監控系統發出緊急警報：

**警報內容**：

```
CRITICAL: Database Response Time Spike
Service: MySQL Primary (db-prod-01)
P95 Latency: 15ms -> 850ms (56x increase)
QPS: 45,000 -> 8,500 (81% drop)
Active Connections: 2,500 (Max: 3,000)
```

**問題嚴重性**：

這是黑色星期五促銷活動的第二天，資料庫延遲飆升導致：

- 購物車結帳失敗率：5% → 65%
- 用戶投訴激增
- 預估損失：每分鐘 $50,000

**緊急診斷**：

```bash
# 檢查 MySQL 狀態
mysql> SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';
# Innodb_buffer_pool_pages_dirty: 95% (異常高)
# Innodb_buffer_pool_wait_free: 12,500 (等待空閒頁面)

# 檢查系統 I/O
$ iostat -xz 1
# %util: 100% (磁碟飽和)
# await: 850ms (平均等待時間，正常應 < 5ms)
# avgqu-sz: 45 (佇列長度，正常應 < 2)
```

**根本原因分析**：

經過深入調查，我們發現問題出在 SSD 配置：

1. **`innodb_io_capacity` 設定錯誤**：
   - 當前值：200 (HDD 時代的預設值)
   - SSD 實際能力：20,000+ IOPS
   - 結果：InnoDB 以為磁碟很慢，刷新 Dirty Pages 的速度遠低於產生速度

2. **`innodb_flush_neighbors` 未關閉**：
   - 這個參數會「順便」刷新鄰近的 Pages
   - 在 HDD 上有用（減少磁頭移動）
   - 在 SSD 上反而增加無謂的寫入

3. **使用 `O_DSYNC` 而非 `O_DIRECT`**：
   - 資料經過 OS Page Cache，造成雙重緩衝
   - 浪費記憶體，增加 CPU 開銷

**緊急修復**：

```sql
-- 動態調整參數（不需重啟）
SET GLOBAL innodb_io_capacity = 20000;
SET GLOBAL innodb_io_capacity_max = 40000;
SET GLOBAL innodb_flush_neighbors = 0;
```

```ini
# 修改配置文件（需重啟）
[mysqld]
innodb_flush_method = O_DIRECT
innodb_io_capacity = 20000
innodb_io_capacity_max = 40000
innodb_flush_neighbors = 0
```

**結果**：

- P95 延遲：850ms → 12ms（恢復正常）
- QPS：8,500 → 52,000（超越原本水準）
- Dirty Pages 比例：95% → 15%（健康範圍）
- 危機解除，促銷活動順利完成

**深刻的教訓**：

這次經驗讓我深刻理解：**資料庫在 SSD 上的配置邏輯與 HDD 時代完全不同**。

本文將深入探討如何針對 SSD 優化資料庫，涵蓋 PostgreSQL、MySQL 和 RocksDB 三大主流資料庫。

---

## 一、理解資料庫的 I/O 模式

### 1.1 OLTP vs OLAP 的 I/O 特性

資料庫的 I/O 模式取決於工作負載類型：

**OLTP (Online Transaction Processing)**：

```
特性：
- 大量小事務（INSERT, UPDATE, DELETE）
- 隨機讀寫為主
- 延遲敏感（要求 < 10ms）
- 高並發（數千 QPS）

I/O 模式：
- 4KB ~ 16KB 的隨機 I/O
- 讀寫比例：70% 讀 / 30% 寫（典型）
- 需要高 IOPS（數萬到數十萬）

典型應用：
- 電商交易系統
- 銀行核心系統
- 社交媒體動態
```

**OLAP (Online Analytical Processing)**：

```
特性：
- 少量大查詢（SELECT with JOIN/GROUP BY）
- 順序掃描為主
- 吞吐量優先（可容忍秒級延遲）
- 低並發（數十 QPS）

I/O 模式：
- 128KB ~ 1MB 的順序 I/O
- 讀為主（95%+ 讀取）
- 需要高吞吐量（GB/s）

典型應用：
- 數據倉儲
- BI 報表
- 日誌分析
```

**SSD 的優勢**：

```
對 OLTP：
- 隨機 IOPS 是 HDD 的 100x+
- 延遲從 10ms 降到 < 1ms
- 可支援更高的並發

對 OLAP：
- 順序讀取速度是 HDD 的 3-5x
- 多個查詢並行時不會互相干擾
```

## 二、PostgreSQL 在 SSD 上的優化

### 2.1 關鍵參數調優

PostgreSQL 傳統上依賴 OS Page Cache，但在 SSD 環境下需要重新思考配置：

**1. `random_page_cost` - 最關鍵的參數**

```sql
-- 查看當前值
SHOW random_page_cost;
-- 預設：4.0 (假設隨機讀取是順序讀取的 4 倍慢)

-- SSD 優化設定
ALTER SYSTEM SET random_page_cost = 1.1;
SELECT pg_reload_conf();
```

**為什麼這麼重要？**

```
HDD 時代：
- 順序讀取：100 MB/s
- 隨機讀取：1 MB/s (需要磁頭移動)
- random_page_cost = 4.0 是合理的

SSD 時代：
- 順序讀取：3,500 MB/s
- 隨機讀取：3,000 MB/s (幾乎沒有差異)
- random_page_cost 應該接近 1.0

影響：
- 查詢優化器會更傾向使用 Index Scan
- 避免不必要的 Sequential Scan
```

**實際案例**：

```sql
-- 測試查詢
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 12345;

-- random_page_cost = 4.0 (預設)
Seq Scan on orders  (cost=0.00..180000.00 rows=1 width=128)
                    (actual time=850.234..850.456 rows=1 loops=1)

-- random_page_cost = 1.1 (SSD 優化)
Index Scan using idx_user_id on orders  (cost=0.43..8.45 rows=1 width=128)
                                        (actual time=0.045..0.048 rows=1 loops=1)

結果：延遲從 850ms 降到 0.05ms (17,000x 提升)
```

**2. `effective_io_concurrency` - 釋放並行度**

```sql
-- 預設值：1 (HDD 時代，避免磁頭競爭)
SHOW effective_io_concurrency;

-- SSD 優化設定
ALTER SYSTEM SET effective_io_concurrency = 200;
```

**作用**：

```
影響 Bitmap Heap Scan 的預取 (Prefetch) 行為：
- 值越高，PostgreSQL 會發出更多並行 I/O 請求
- SSD 的多佇列特性可以充分利用

適用場景：
- 大表掃描
- JOIN 操作
- 分析查詢
```

**3. `shared_buffers` - 避免雙重緩衝**

```sql
-- 查看系統記憶體
$ free -h
# Total: 64GB

-- PostgreSQL 配置
ALTER SYSTEM SET shared_buffers = '16GB';  -- 25% of RAM
ALTER SYSTEM SET effective_cache_size = '48GB';  -- 75% of RAM
```

**配置邏輯**：

```
PostgreSQL 的記憶體架構：
1. shared_buffers (PG 自己管理)
2. OS Page Cache (OS 管理)
3. 兩者會有重疊 (雙重緩衝)

建議：
- shared_buffers = 25% ~ 40% of RAM
- 不要設太大 (> 50%)，會浪費記憶體
- 讓 OS Page Cache 處理剩餘的快取
```

**4. Checkpoint 優化 - 減少寫入放大**

```sql
-- 延長 Checkpoint 間隔
ALTER SYSTEM SET checkpoint_timeout = '30min';  -- 預設 5min
ALTER SYSTEM SET max_wal_size = '16GB';  -- 預設 1GB

-- 啟用 WAL 壓縮
ALTER SYSTEM SET wal_compression = on;
```

**效果**：

```
調優前：
- Checkpoint 每 5 分鐘一次
- 每次寫入 2GB 資料
- 寫入放大：2.5x
- SSD 壽命：3 年

調優後：
- Checkpoint 每 30 分鐘一次
- 每次寫入 12GB 資料
- 寫入放大：1.8x
- SSD 壽命：5 年+
```

---

### 2.2 監控和診斷

**使用 `pg_stat_statements` 找出 I/O 熱點**：

```sql
-- 啟用擴展
CREATE EXTENSION pg_stat_statements;

-- 找出最耗 I/O 的查詢
SELECT
    query,
    calls,
    total_time,
    mean_time,
    shared_blks_read,  -- 從磁碟讀取的 Blocks
    shared_blks_hit,   -- 從 Buffer Pool 讀取的 Blocks
    shared_blks_read * 100.0 / (shared_blks_read + shared_blks_hit) AS cache_miss_ratio
FROM pg_stat_statements
WHERE shared_blks_read > 0
ORDER BY shared_blks_read DESC
LIMIT 10;
```

**解讀結果**：

```
範例輸出：
query: SELECT * FROM logs WHERE created_at > ...
calls: 125,000
shared_blks_read: 5,000,000
shared_blks_hit: 500,000
cache_miss_ratio: 90.9%

分析：
- 這個查詢的快取命中率只有 9.1%
- 大量實體 I/O
- 需要優化：加索引或增加 shared_buffers
```

---

## 三、MySQL (InnoDB) 在 SSD 上的優化

### 3.1 關鍵參數調優

MySQL 的 InnoDB 引擎與 PostgreSQL 不同，它更傾向使用 Direct I/O，因此優化策略也不同：

**1. `innodb_io_capacity` - 告訴 InnoDB「SSD 很快」**

```ini
[mysqld]
# 預設值：200 (HDD 時代)
innodb_io_capacity = 20000
innodb_io_capacity_max = 40000
```

**這是最重要的參數！**

```
作用：
- 控制 InnoDB 刷新 Dirty Pages 的速度
- 控制 Purge 操作的速度
- 控制 Change Buffer Merge 的速度

設定邏輯：
- 測試 SSD 的 IOPS：
  $ fio --name=test --ioengine=libaio --direct=1 --bs=16k \
        --rw=randwrite --numjobs=4 --iodepth=64 --runtime=60 \
        --filename=/dev/nvme0n1
  # 結果：80,000 IOPS

- 設定為測試值的 25%：
  innodb_io_capacity = 20,000
  innodb_io_capacity_max = 40,000 (2x)
```

**影響**：

```
調優前 (innodb_io_capacity=200)：
- Dirty Pages 比例：85% (危險)
- Checkpoint Age：90% (接近上限)
- 寫入延遲：P95 = 120ms

調優後 (innodb_io_capacity=20000)：
- Dirty Pages 比例：15% (健康)
- Checkpoint Age：30%
- 寫入延遲：P95 = 5ms
```

**2. `innodb_flush_neighbors` - 關閉無謂的鄰近刷新**

```ini
[mysqld]
# HDD 時代：1 (刷新鄰近 Pages，減少磁頭移動)
# SSD 時代：0 (關閉，SSD 沒有磁頭)
innodb_flush_neighbors = 0
```

**3. `innodb_flush_method` - 使用 Direct I/O**

```ini
[mysqld]
# 強烈建議
innodb_flush_method = O_DIRECT
```

**為什麼？**

```
Buffered I/O (預設)：
資料路徑：InnoDB Buffer Pool -> OS Page Cache -> Disk
問題：雙重緩衝，浪費記憶體

Direct I/O (O_DIRECT)：
資料路徑：InnoDB Buffer Pool -> Disk
優勢：
- 節省記憶體
- 減少 CPU 複製開銷
- 延遲更可預測
```

**4. Buffer Pool 配置**

```ini
[mysqld]
# 系統記憶體：64GB
innodb_buffer_pool_size = 48GB  # 75% of RAM
innodb_buffer_pool_instances = 16  # CPU 核心數
```

**5. Redo Log 優化**

```ini
[mysqld]
# 增大 Redo Log，減少 Checkpoint 頻率
innodb_log_file_size = 4GB  # 預設 48MB
innodb_log_files_in_group = 2

# 總 Redo Log 大小 = 4GB x 2 = 8GB
```

---

### 3.2 實際效能對比

**測試環境**：

- 硬體：Intel Xeon, NVMe SSD (RAID 0)
- 工具：Sysbench OLTP Read/Write

**調優前 (HDD 時代配置)**：

```ini
innodb_io_capacity = 200
innodb_flush_neighbors = 1
innodb_flush_method = O_DSYNC
innodb_buffer_pool_size = 8GB
```

**調優後 (SSD 優化配置)**：

```ini
innodb_io_capacity = 20000
innodb_flush_neighbors = 0
innodb_flush_method = O_DIRECT
innodb_buffer_pool_size = 48GB
```

**結果對比**：

| 指標 | 調優前 | 調優後 | 改善幅度 |
|------|--------|--------|----------|
| QPS | 12,500 | 28,000 | +124% |
| TPS | 850 | 2,100 | +147% |
| P95 Latency | 120 ms | 15 ms | -87.5% |
| P99 Latency | 350 ms | 28 ms | -92% |
| CPU Utilization | 30% (I/O Wait 高) | 75% (User CPU 高) | I/O 不再是瓶頸 |

---

### 1.2 B-Tree vs LSM-Tree 的寫入模式

資料庫的儲存引擎決定了 I/O 模式：

**B-Tree (InnoDB, PostgreSQL)**：

```
寫入流程：
1. 寫入 WAL (Write-Ahead Log)
2. 更新 Buffer Pool 中的 Page
3. 後台刷新 Dirty Pages 到磁碟

特性：
- 寫入放大 (Write Amplification)：2-3x
- 原地更新 (In-place Update)
- 讀取效能優秀（B-Tree 索引）

適用場景：
- 讀多寫少
- 需要事務支援
- 延遲敏感
```

**LSM-Tree (RocksDB, LevelDB, Cassandra)**：

```
寫入流程：
1. 寫入 WAL
2. 寫入 Memtable (記憶體)
3. Memtable 滿了刷入 L0 (SSTable)
4. 後台 Compaction 合併 SSTables

特性：
- 寫入放大：10-30x（取決於 Compaction 策略）
- 順序寫入（對 SSD 友善）
- 讀取需要查詢多層（讀取放大）

適用場景：
- 寫多讀少
- 時序資料
- 日誌存儲
```

**SSD 對兩者的影響**：

```
B-Tree：
- SSD 的隨機寫入能力強，減少了 Dirty Pages 積壓
- 但仍需優化 Checkpoint 頻率

LSM-Tree：
- SSD 的高 IOPS 可以加速 Compaction
- 但寫入放大仍會影響 SSD 壽命
- 需要調優 Compaction 策略
```

## 四、RocksDB (LSM-Tree) 在 SSD 上的優化

### 4.1 LSM-Tree 的寫入放大問題

RocksDB 使用 LSM-Tree 結構，寫入放大是最大的挑戰：

**寫入放大的來源**：

```
用戶寫入 1GB 資料的實際流程：

1. 寫入 WAL：1GB
2. 寫入 Memtable -> Flush 到 L0：1GB
3. L0 -> L1 Compaction：讀 1GB (L0) + 10GB (L1) = 11GB
                        寫 11GB (新 L1)
4. L1 -> L2 Compaction：讀 11GB (L1) + 110GB (L2) = 121GB
                        寫 121GB (新 L2)

總寫入放大 = (1 + 1 + 11 + 121) / 1 = 134x (極端情況)
實際通常：10-30x
```

**對 SSD 的影響**：

```
假設：
- 用戶寫入：100 GB/day
- 寫入放大：20x
- 實際 SSD 寫入：2 TB/day

SSD 壽命計算：
- TLC NAND：3,000 P/E Cycles
- 容量：1TB
- 總寫入量：3,000 TB
- 壽命：3,000 TB / 2 TB/day = 1,500 days ≈ 4 年

如果降低寫入放大到 10x：
- 壽命：3,000 TB / 1 TB/day = 3,000 days ≈ 8 年
```

---

### 4.2 RocksDB 關鍵參數調優

**1. `write_buffer_size` - Memtable 大小**

```cpp
options.write_buffer_size = 128 * 1024 * 1024;  // 128MB
```

**調優邏輯**：

```
Memtable 越大：
優點：
- 減少 Flush 到 L0 的頻率
- 降低寫入放大

缺點：
- 記憶體佔用高
- 崩潰恢復時間長（需要重放 WAL）

建議：
- 寫入密集：128MB
- 記憶體受限：64MB
```

**2. `max_write_buffer_number` - Memtable 數量**

```cpp
options.max_write_buffer_number = 6;
```

**為什麼需要多個 Memtable？**

```
寫入流程：
1. 寫入 Active Memtable
2. Active Memtable 滿了 -> 變成 Immutable Memtable
3. 後台執行緒將 Immutable Memtable Flush 到 L0
4. 如果 Flush 太慢，需要額外的 Memtable 接收新寫入

如果 max_write_buffer_number 太小：
- 所有 Memtable 都滿了
- 寫入會被阻塞 (Write Stall)
- 延遲飆升

建議：4 ~ 6
```

**3. `level0_file_num_compaction_trigger` - L0 Compaction 觸發點**

```cpp
options.level0_file_num_compaction_trigger = 4;
```

**這是平衡讀寫的關鍵**：

```
L0 的特殊性：
- L0 的 SSTable 之間 Key Range 是重疊的
- 讀取時需要查詢所有 L0 檔案
- L0 檔案越多，讀取越慢

設定太小 (e.g., 2)：
- 頻繁觸發 L0->L1 Compaction
- 寫入放大嚴重
- 寫入效能差

設定太大 (e.g., 10)：
- L0 檔案堆積
- 讀取需要查詢 10 個檔案
- 讀取效能差

建議：4 (預設值通常是最佳平衡點)
```

**4. `max_background_jobs` - 後台執行緒數**

```cpp
options.max_background_jobs = 8;  // CPU 核心數的 1/4 ~ 1/2
```

**作用**：

```
後台任務：
- Flush (Memtable -> L0)
- Compaction (Ln -> Ln+1)

SSD 的高 IOPS 需要足夠的並行度來消化：
- 執行緒太少：Compaction 跟不上，L0 堆積
- 執行緒太多：CPU 切換開銷大，影響前台查詢

建議：
- 16 核 CPU：4 ~ 8 個後台執行緒
- 32 核 CPU：8 ~ 16 個後台執行緒
```

**5. Direct I/O 配置**

```cpp
options.use_direct_reads = true;
options.use_direct_io_for_flush_and_compaction = true;
```

**為什麼 RocksDB 特別需要 Direct I/O？**

```
Compaction 的特性：
- 大量讀寫操作
- 如果經過 OS Page Cache，會污染快取
- 把應用程式的熱資料擠出去

使用 Direct I/O：
- Compaction 不經過 Page Cache
- 隔離 Compaction 對系統的影響
- 前台查詢的快取命中率更高
```

---

### 4.3 實際效能對比

**測試場景**：寫入密集型工作負載（大量隨機寫入）

**調優前 (預設配置)**：

```cpp
write_buffer_size = 64MB
max_write_buffer_number = 2
level0_file_num_compaction_trigger = 4
max_background_jobs = 2
use_direct_reads = false
use_direct_io_for_flush_and_compaction = false
```

**調優後 (SSD 優化配置)**：

```cpp
write_buffer_size = 128MB
max_write_buffer_number = 6
level0_file_num_compaction_trigger = 4
max_background_jobs = 8
use_direct_reads = true
use_direct_io_for_flush_and_compaction = true
```

**結果對比**：

| 指標 | 調優前 | 調優後 | 分析 |
|------|--------|--------|------|
| 寫入放大 (Write Amp) | 25.4 | 12.1 | 有效減少 SSD 磨損 |
| 平均寫入吞吐量 | 220 MB/s | 450 MB/s | 減少了 Write Stall |
| 讀取 P99 延遲 | 8.5 ms | 2.1 ms | Direct I/O 避免了 Compaction 污染快取 |
| Write Stall 次數 | 125/hour | 5/hour | Memtable 配置更合理 |

---

## 五、Direct I/O vs Buffered I/O 的選擇

### 5.1 兩種模式的對比

| 特性 | Buffered I/O | Direct I/O (`O_DIRECT`) |
|------|--------------|-------------------------|
| **機制** | 資料先寫入 OS Page Cache，OS 決定何時刷入磁碟 | 繞過 OS Page Cache，DB 直接與 SSD 控制器溝通 |
| **優點** | 簡單，OS 提供 Read-Ahead 優化 | 無雙重緩衝，節省 RAM；延遲可預測；CPU Overhead 較低 |
| **缺點** | 雙重緩衝 (DB Buffer 和 OS Cache 存兩份)；寫入落盤時間不可控 | 必須自行實作對齊 (Memory Alignment) 和緩衝管理，開發複雜 |

---

### 5.2 各資料庫的建議

**MySQL (InnoDB)**：**強烈建議使用 Direct I/O**

```ini
[mysqld]
innodb_flush_method = O_DIRECT
```

**原因**：

- InnoDB 有非常成熟的 Buffer Pool
- 不需要 OS 插手
- 使用 Direct I/O 可節省記憶體並減少 CPU 複製開銷

**PostgreSQL**：**傳統上使用 Buffered I/O，但正在轉型**

```
現狀：
- PG 架構深度依賴 OS Page Cache
- 預設不使用 Direct I/O 讀寫數據檔

趨勢：
- PG 14/15+ 引入了 io_direct 參數 (實驗性)
- 或對 WAL 使用 Direct I/O

建議：
- 目前維持預設 (Buffered)
- 但需調優 shared_buffers 避免浪費
```

**RocksDB**：**建議開啟 Direct I/O**

```cpp
options.use_direct_reads = true;
options.use_direct_io_for_flush_and_compaction = true;
```

**原因**：

- LSM-Tree 的 Compaction 涉及大量讀寫
- 若經過 Page Cache 會嚴重污染 OS 快取
- 使用 Direct I/O 可隔離 Compaction 對系統的影響

---

## 六、監控和診斷工具

### 6.1 系統層級監控

**`iostat` - 最常用的工具**

```bash
$ iostat -xz 1

Device   r/s   w/s   rkB/s   wkB/s  await  %util
nvme0n1  1250  3500  50000   140000  2.5    85.0
```

**關鍵指標**：

```
%util：設備利用率
- 接近 100% 代表磁碟飽和
- 但在 NVMe SSD 上，即便 100% 可能只是單一佇列滿了

await：平均 I/O 請求等待時間 (ms)
- **核心指標**
- SSD 上寫入通常 < 1ms，讀取 < 0.5ms
- 若飆高，代表 SSD 處理不過來或寫入放大嚴重

avgqu-sz：平均佇列長度
- 如果此值遠大於 1，說明有 I/O 在排隊
```

**`blktrace` - 深度診斷**

```bash
# 開始追蹤
$ sudo blktrace -d /dev/nvme0n1 -o trace

# 停止追蹤 (Ctrl+C)

# 分析結果
$ blkparse -i trace | less
```

**用途**：

- 當 `iostat` 看不出原因時使用
- 可以追蹤 I/O 在 Linux 區塊層的每個階段的耗時
- 精準定位是 OS 軟體層慢還是硬體慢

---

### 6.2 資料庫層級監控

**PostgreSQL: `pg_stat_statements`**

```sql
-- 找出 I/O 熱點
SELECT
    query,
    calls,
    total_time,
    shared_blks_read,  -- 從磁碟讀取
    shared_blks_hit,   -- 從緩衝區讀取
    blk_read_time      -- 讀取 I/O 總耗時
FROM pg_stat_statements
WHERE shared_blks_read > 0
ORDER BY blk_read_time DESC
LIMIT 10;
```

**MySQL: `performance_schema`**

```sql
-- 監控檔案 I/O
SELECT
    file_name,
    COUNT_READ,
    COUNT_WRITE,
    SUM_TIMER_READ / 1000000000000 AS read_time_sec,
    SUM_TIMER_WRITE / 1000000000000 AS write_time_sec
FROM performance_schema.file_summary_by_instance
WHERE file_name LIKE '%ibd'
ORDER BY SUM_TIMER_READ DESC
LIMIT 10;
```

---

## 七、總結

### 7.1 核心要點回顧

**PostgreSQL 優化**：

1. **降低 `random_page_cost` 到 1.1** - 讓優化器知道 SSD 很快
2. **提高 `effective_io_concurrency` 到 200+** - 釋放並行度
3. **延長 Checkpoint 間隔** - 減少寫入放大

**MySQL (InnoDB) 優化**：

1. **提高 `innodb_io_capacity` 到 20,000+** - 告訴 InnoDB「SSD 很快」
2. **關閉 `innodb_flush_neighbors`** - SSD 不需要鄰近刷新
3. **使用 `O_DIRECT`** - 避免雙重緩衝

**RocksDB 優化**：

1. **調整 Memtable 配置** - 平衡記憶體和寫入放大
2. **控制 L0 Compaction 觸發點** - 平衡讀寫效能
3. **使用 Direct I/O** - 隔離 Compaction 影響

**通用原則**：

- **信任硬體**：SSD 的隨機 I/O 能力遠超 HDD
- **繞過 OS 干擾**：Direct I/O 可以提供更可預測的延遲
- **減少寫入放大**：延長 SSD 壽命

---

### 7.2 下一篇預告

下一篇文章將探討 **AI/ML Workloads - 訓練資料的儲存策略**：

- PyTorch/TensorFlow 的 DataLoader 優化
- 檔案格式選擇（TFRecord vs HDF5 vs WebDataset）
- GPU Direct Storage 技術
- 大規模訓練的 Checkpoint 策略

---

## 參考資料

1. PostgreSQL Documentation: <https://www.postgresql.org/docs/current/runtime-config-resource.html>
2. MySQL InnoDB Storage Engine: <https://dev.mysql.com/doc/refman/8.0/en/innodb-storage-engine.html>
3. RocksDB Tuning Guide: <https://github.com/facebook/rocksdb/wiki/RocksDB-Tuning-Guide>
4. "Understanding the Robustness of SSDs under Power Fault", FAST 2013

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
