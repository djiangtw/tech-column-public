# Cloud Storage - 雲端環境中的 SSD 管理

**作者**: Danny Jiang  
**日期**: 2025-12-09  

---

## 前言：一次雲端儲存的慘痛教訓

2019 年，我負責將一個資料庫密集型應用遷移到 AWS。為了節省成本，我選擇了 `t3.medium` 實例搭配 `gp2` EBS 卷（500GB）。

**初期測試一切正常**：

```bash
# 壓力測試
$ sysbench oltp_read_write --mysql-host=localhost run

Transactions: 1,250 TPS
P95 Latency: 15ms
```

**上線第一天，災難發生**：

```
時間：上午 10:00（業務高峰期）

監控警報：
CRITICAL: Database Response Time Spike
P95 Latency: 15ms -> 850ms (56x increase)
TPS: 1,250 -> 180 (85% drop)
```

**緊急診斷**：

```bash
# 檢查 CloudWatch Metrics
$ aws cloudwatch get-metric-statistics \
    --namespace AWS/EBS \
    --metric-name BurstBalance \
    --dimensions Name=VolumeId,Value=vol-xxxxx

# 結果：
BurstBalance: 0% (耗盡！)
```

**根本原因分析**：

我犯了一個新手錯誤：**不理解 gp2 的 Burst Bucket 機制**。

```
gp2 的效能模型：
- 基準線 (Baseline)：3 IOPS/GB
- 500GB 卷：500 × 3 = 1,500 IOPS (基準線)
- Burst 能力：最高 3,000 IOPS
- Burst Bucket：5.4 million I/O credits

問題：
- 測試時：Burst Bucket 是滿的，可以跑 3,000 IOPS
- 上線後：持續高負載，30 分鐘內耗盡 Burst Credits
- 效能掉到基準線：1,500 IOPS（遠低於需求）

結果：
- 資料庫延遲飆升
- 用戶體驗極差
- 業務損失：每小時 $5,000
```

**緊急修復**：

```bash
# 方案 1：增加容量（治標不治本）
$ aws ec2 modify-volume --volume-id vol-xxxxx --size 1000
# 1000GB × 3 IOPS/GB = 3,000 IOPS (基準線)
# 但成本翻倍！

# 方案 2：遷移到 gp3（正確方案）
$ aws ec2 create-volume \
    --volume-type gp3 \
    --size 500 \
    --iops 5000 \
    --throughput 250

# gp3 優勢：
# - IOPS 與容量解耦
# - 無 Burst Bucket 限制
# - 成本比 gp2 便宜 20%
```

**最終結果**：

| 指標 | gp2 (500GB) | gp3 (500GB, 5000 IOPS) | 改善 |
|------|-------------|------------------------|------|
| 基準線 IOPS | 1,500 | 5,000 | +233% |
| Burst IOPS | 3,000 (有時間限制) | 5,000 (持續) | +67% |
| P95 Latency | 850ms (Burst 耗盡後) | 12ms | -98.6% |
| 月成本 | $50 | $42 | -16% |

**深刻的教訓**：

在雲端環境中，**「硬碟」不再是實體硬體，而是一種可程式化的服務 (SLA)**。

不理解雲端儲存的 QoS 機制，會導致嚴重的效能問題和成本浪費。

本文將深入探討雲端環境中的 SSD 管理，涵蓋 AWS、Azure、GCP 三大雲端平台。

---

## 一、AWS EBS 類型選擇

### 1.1 EBS 類型對比

AWS Elastic Block Store (EBS) 提供多種類型，選擇正確的類型是效能與成本平衡的關鍵：

| 類型 | 代號 | 效能特性 | 最佳使用場景 | 成本 (us-east-1) |
|------|------|----------|--------------|------------------|
| **General Purpose SSD** | gp3 | 3,000 IOPS / 125 MB/s (基準線)<br>最高 16,000 IOPS / 1,000 MB/s | 大多數生產環境、Web 伺服器、中小型資料庫 | $0.08/GB-month |
| **General Purpose SSD** | gp2 | 3 IOPS/GB (基準線)<br>Burst 最高 3,000 IOPS | **已過時，不建議使用** | $0.10/GB-month |
| **Provisioned IOPS SSD** | io2 | 最高 64,000 IOPS / 1,000 MB/s<br>99.999% 耐用度 | 關鍵任務資料庫 (Oracle, SAP HANA) | $0.125/GB-month<br>+ $0.065/IOPS-month |
| **Provisioned IOPS SSD** | io2 Block Express | 最高 256,000 IOPS / 4,000 MB/s<br>延遲 < 1ms | 超大型資料庫、高頻交易系統 | $0.125/GB-month<br>+ $0.065/IOPS-month |
| **Throughput Optimized HDD** | st1 | 最高 500 MB/s<br>Burst 最高 500 IOPS | 大數據、日誌處理、資料倉儲 | $0.045/GB-month |
| **Cold HDD** | sc1 | 最高 250 MB/s<br>Burst 最高 250 IOPS | 冷資料歸檔 | $0.015/GB-month |

---

### 1.2 gp3 vs gp2 深度對比

**gp2 的致命缺陷**：

```
gp2 的效能模型（Burst Bucket）：

1. 基準線效能：3 IOPS/GB
   - 100GB 卷：300 IOPS
   - 500GB 卷：1,500 IOPS
   - 1TB 卷：3,000 IOPS

2. Burst 能力：
   - 最高 3,000 IOPS（無論容量多大）
   - Burst Bucket 容量：5.4 million I/O credits
   - 賺取速度：3 IOPS/GB/sec
   - 消耗速度：實際 IOPS - 基準線 IOPS

3. 問題：
   - 為了獲得高 IOPS，必須購買大容量
   - 例如：需要 10,000 IOPS，必須買 3.3TB（浪費容量）
   - Burst 有時間限制，持續高負載會耗盡
```

**gp3 的優勢**：

```
gp3 的效能模型（Provisioned）：

1. 基準線效能（免費）：
   - 3,000 IOPS
   - 125 MB/s
   - 與容量無關

2. 可擴展效能（付費）：
   - IOPS：最高 16,000 ($0.005/IOPS-month)
   - 吞吐量：最高 1,000 MB/s ($0.04/MB/s-month)

3. 優勢：
   - IOPS 與容量解耦
   - 無 Burst Bucket 限制（效能持續）
   - 成本比 gp2 便宜 20%
```

**實際案例對比**：

```
需求：500GB 容量，5,000 IOPS

方案 A：gp2
- 容量：需要 1,667GB (5000 / 3 = 1667)
- 成本：1,667GB × $0.10 = $166.7/month
- 問題：浪費 1,167GB 容量

方案 B：gp3
- 容量：500GB
- 基準線：3,000 IOPS (免費)
- 額外 IOPS：2,000 IOPS × $0.005 = $10/month
- 總成本：500GB × $0.08 + $10 = $50/month

節省：$166.7 - $50 = $116.7/month (70% 成本節省)
```

---

### 1.3 io2 Block Express - 極致效能

**何時需要 io2？**

```
適用場景：
1. 關鍵任務資料庫（Oracle RAC, SAP HANA）
2. 對延遲極度敏感的應用（< 1ms）
3. 需要超高 IOPS（> 16,000）

技術特性：
- 運行在 AWS Nitro 系統上
- SR-IOV (Single Root I/O Virtualization)
- 幾乎消除了虛擬化開銷
- 99.999% 耐用度（gp3 是 99.8%~99.9%）
```

**效能對比**：

| 指標 | gp3 | io2 | io2 Block Express |
|------|-----|-----|-------------------|
| 最大 IOPS | 16,000 | 64,000 | 256,000 |
| 最大吞吐量 | 1,000 MB/s | 1,000 MB/s | 4,000 MB/s |
| 延遲 (P99) | 1-2ms | < 1ms | < 0.5ms |
| 最大容量 | 16TB | 16TB | 64TB |

**成本分析**：

```
範例：1TB 卷，20,000 IOPS

io2：
- 容量：1TB × $0.125 = $125/month
- IOPS：20,000 × $0.065 = $1,300/month
- 總計：$1,425/month

gp3（無法達到 20,000 IOPS）：
- 最高只能 16,000 IOPS
- 如果 16,000 IOPS 足夠：
  - 容量：1TB × $0.08 = $80/month
  - IOPS：13,000 × $0.005 = $65/month
  - 總計：$145/month

結論：io2 成本是 gp3 的 10x，只在真正需要時使用
```

---

## 二、Azure 和 GCP 的儲存選項

### 2.1 Azure Managed Disks

Azure 的儲存分級與 AWS 類似，但有一些獨特功能：

| 類型 | 效能特性 | 特色功能 | 適用場景 |
|------|----------|----------|----------|
| **Standard HDD** | 最高 500 IOPS / 60 MB/s | 成本最低 | 開發/測試環境 |
| **Standard SSD** | 最高 6,000 IOPS / 750 MB/s | Disk Bursting | 一般工作負載 |
| **Premium SSD** | 最高 20,000 IOPS / 900 MB/s | 低延遲 (< 1ms) | 生產資料庫 |
| **Premium SSD v2** | 最高 80,000 IOPS / 1,200 MB/s | IOPS 與容量解耦 | 高效能資料庫 |
| **Ultra Disk** | 最高 160,000 IOPS / 4,000 MB/s | 動態調整 IOPS（不需重啟） | 關鍵任務應用 |

**Azure 的獨特功能**：

**1. Disk Bursting (磁碟突發)**

```
適用於：Standard SSD, Premium SSD

機制：
- 類似 AWS gp2 的 Burst Bucket
- 允許小容量磁碟在短時間內超越基準線

範例：
- Premium SSD P10 (128GB)
- 基準線：500 IOPS / 100 MB/s
- Burst：3,500 IOPS / 170 MB/s (最多 30 分鐘)

適用場景：
- 間歇性高負載（如每日備份）
- 開發/測試環境
```

**2. Shared Disks (共享磁碟)**

```
功能：
- 允許多個 VM 同時掛載同一個磁碟
- 適用於 Windows Server Failover Cluster (WSFC)

限制：
- 只支援 Premium SSD 和 Ultra Disk
- 需要應用程式層的鎖機制（如 SCSI-3 PR）

適用場景：
- SQL Server Always On FCI
- Oracle RAC
```

---

### 2.2 GCP Persistent Disks

GCP 的儲存設計哲學強調**靈活性和自動化**：

| 類型 | 效能特性 | 特色功能 | 適用場景 |
|------|----------|----------|----------|
| **Standard PD** | 最高 7,500 IOPS / 1,200 MB/s | 成本最低 | 大數據、歸檔 |
| **Balanced PD** | 最高 80,000 IOPS / 1,200 MB/s | **性價比甜蜜點** | 大多數生產環境 |
| **SSD PD** | 最高 100,000 IOPS / 1,200 MB/s | 高效能 | 資料庫、分析 |
| **Extreme PD** | 最高 120,000 IOPS / 2,400 MB/s | 超高效能 | 關鍵任務應用 |
| **Hyperdisk** | 最高 350,000 IOPS / 4,800 MB/s | 新一代架構 | 超大型資料庫 |

**GCP 的獨特功能**：

**1. Live Resizing (線上擴容)**

```
優勢：
- 可以線上調整容量、IOPS、類型
- 不需要 Detach 磁碟
- 不需要重啟 VM

範例：
$ gcloud compute disks resize my-disk --size 1000GB
$ gcloud compute disks update my-disk --provisioned-iops 10000

# 立即生效，無需重啟
```

**2. Regional Persistent Disks (區域持久磁碟)**

```
功能：
- 資料同步複製到兩個 Zone
- 自動容錯移轉 (Failover)
- 99.99% SLA（Zonal PD 是 99.9%）

成本：
- 價格是 Zonal PD 的 2x
- 但提供了內建的 DR (Disaster Recovery)

適用場景：
- 高可用性資料庫
- 關鍵業務應用
```

## 三、Instance Store (NVMe) - 本地 SSD

### 3.1 Instance Store 的特性

Instance Store（又稱 Ephemeral Storage）是**直接物理連接**在宿主機上的 NVMe SSD：

**效能數據**：

```
AWS i3.8xlarge (4 × 1.9TB NVMe SSD)：
- 隨機讀取 IOPS：3,300,000 (4k)
- 順序讀取：16 GB/s
- 延遲：30-50 微秒 (µs)

對比 EBS io2 Block Express：
- IOPS：256,000
- 吞吐量：4 GB/s
- 延遲：< 1ms (1,000 µs)

Instance Store 快 10-100 倍！
```

**致命限制**：

```
資料易失性 (Ephemeral)：

會導致資料丟失的操作：
1. Stop Instance（停止實例）
2. Terminate Instance（終止實例）
3. 硬體故障
4. 宿主機維護

不會丟失的操作：
1. Reboot（重啟）
2. OS 層級的重啟

結論：
- Instance Store 的資料是暫時的
- 必須假設資料隨時會消失
```

**其他限制**：

```
1. 容量固定：
   - 與 Instance Type 綁定
   - 無法調整大小
   - 例如：i3.large 固定 475GB

2. 無快照功能：
   - 無法使用 EBS Snapshot
   - 需要自行實作備份

3. 無法 Detach/Attach：
   - 無法在 Instance 之間移動
```

---

### 3.2 Instance Store 的最佳使用場景

**場景 1：NoSQL 資料庫的 Cache 層**

```
範例：Redis / Memcached

架構：
- 使用 Instance Store 作為快取儲存
- 資料來源在 EBS 或 S3
- 即使 Instance Store 資料丟失，可以從來源重建

優勢：
- 極低延遲（微秒級）
- 高 IOPS（數百萬）
- 成本低（Instance Store 免費）
```

**場景 2：分散式資料庫（已實作應用層複製）**

```
範例：Cassandra, MongoDB, Elasticsearch

架構：
- 每個節點使用 Instance Store
- 資料複製因子 (Replication Factor) ≥ 3
- 即使單一節點故障，資料仍在其他節點

優勢：
- 極致效能
- 靠軟體層保證資料不丟失
- 成本效益高
```

**場景 3：大數據計算的中間暫存**

```
範例：Spark Shuffle, MapReduce Intermediate Data

特性：
- 中間資料是暫時的
- 即使丟失，可以重新計算
- 不需要持久化

優勢：
- 高速讀寫
- 不佔用 EBS IOPS 配額
```

## 四、雲端環境的 I/O QoS 和 Noisy Neighbor 問題

### 4.1 I/O QoS 機制

雲端硬碟本質上是**網路儲存 (NAS/SAN)**，為了公平分配資源，雲端供應商實施嚴格的 QoS：

**IOPS Credits 和 Burst Balance**：

```
適用於：AWS gp2, Azure Standard/Premium SSD

權杖桶演算法 (Token Bucket Algorithm)：
1. 每個磁碟有一個「桶子」(Bucket)
2. 閒置時賺取 Credits（充電）
3. 讀寫時消耗 Credits（放電）
4. Credits 耗盡時，效能降到基準線

範例（AWS gp2, 500GB）：
- Bucket 容量：5.4 million I/O credits
- 賺取速度：1,500 credits/sec (3 IOPS/GB)
- Burst 能力：3,000 IOPS

情境：
- 閒置 1 小時：賺取 5.4M credits（桶子滿）
- 突發負載 3,000 IOPS：消耗 3,000 credits/sec
- 持續時間：5.4M / (3000 - 1500) = 3,600 秒 = 1 小時
- 1 小時後：Credits 耗盡，效能降到 1,500 IOPS
```

**Instance Level Limit vs. EBS Level Limit**：

```
常見陷阱：
- 購買了 20,000 IOPS 的 io2 EBS
- 掛載在 t3.small 上
- 實際只能跑 2,000 IOPS

原因：
- EC2 Instance 本身也有 I/O 頻寬限制
- t3.small 的 EBS 頻寬上限較低

結論：
- EBS 效能受兩個限制：
  1. EBS 本身的 Provisioned IOPS
  2. Instance 的 I/O 頻寬上限
- 取兩者的最小值
```

---

### 4.2 Noisy Neighbor 問題

**吵雜鄰居 (Noisy Neighbor)** 是指同一台實體機上的其他 VM 搶佔了過多的 I/O 或網路資源：

**現象**：

```
症狀：
- 負載沒變，但延遲忽高忽低
- 特定時間點效能下降（如鄰居的備份時間）
- CloudWatch Metrics 顯示 I/O 延遲抖動

範例：
- 平時 P95 延遲：5ms
- 每天凌晨 2:00：P95 延遲飆升到 50ms
- 原因：鄰居 VM 在執行大量 I/O（備份、批次處理）
```

**解決方案**：

**1. 使用 Dedicated Host**

```
優勢：
- 包下整台實體機
- 物理隔離，無鄰居干擾

缺點：
- 成本高（約 3-5x）
- 需要自行管理容量

適用場景：
- 關鍵任務應用
- 需要穩定效能的資料庫
```

**2. 選擇 QoS 保證型儲存**

```
AWS：
- io2 / io2 Block Express
- 更嚴格的 SLA 和底層隔離

Azure：
- Ultra Disk
- Premium SSD v2

GCP：
- Extreme PD
- Hyperdisk
```

**3. 應用程式層的重試機制**

```python
import time
import random

def exponential_backoff_retry(func, max_retries=5):
    for attempt in range(max_retries):
        try:
            return func()
        except IOError as e:
            if attempt == max_retries - 1:
                raise
            # 指數退避：2^attempt × 100ms + 隨機抖動
            wait_time = (2 ** attempt) * 0.1 + random.uniform(0, 0.1)
            time.sleep(wait_time)
```

**4. 現代雲端架構的改善**

```
AWS Nitro System：
- 將 I/O 虛擬化卸載到專用硬體卡
- 減少鄰居間的干擾
- 提供更可預測的效能

GCP Titanium：
- 類似 Nitro 的架構
- 硬體加速的網路和儲存虛擬化
```

---

## 五、成本優化策略

### 5.1 gp2 遷移至 gp3

**最立竿見影的優化**：

```
效益：
- gp3 每 GB 價格比 gp2 便宜 20%
- IOPS 與容量解耦，避免過度配置

範例：
需求：500GB, 5,000 IOPS

gp2：
- 需要 1,667GB (5000 / 3)
- 成本：$166.7/month

gp3：
- 500GB + 2,000 額外 IOPS
- 成本：$50/month

節省：70%
```

**遷移步驟**：

```bash
# 1. 創建 Snapshot
$ aws ec2 create-snapshot --volume-id vol-xxxxx

# 2. 從 Snapshot 創建 gp3 卷
$ aws ec2 create-volume \
    --snapshot-id snap-xxxxx \
    --volume-type gp3 \
    --size 500 \
    --iops 5000 \
    --availability-zone us-east-1a

# 3. Detach 舊卷，Attach 新卷
$ aws ec2 detach-volume --volume-id vol-old
$ aws ec2 attach-volume --volume-id vol-new --instance-id i-xxxxx --device /dev/sdf

# 4. 刪除舊卷
$ aws ec2 delete-volume --volume-id vol-old
```

---

### 5.2 刪除未掛載磁碟 (Orphaned Volumes)

**問題**：

```
當 EC2 被 Terminate 時：
- Root Volume：預設刪除
- Data Volume：預設保留（"Available" 狀態）

結果：
- 這些 "Available" 狀態的磁碟會持續計費
- 很多公司有數百個被遺忘的磁碟
```

**自動化清理**：

```python
import boto3
from datetime import datetime, timedelta

ec2 = boto3.client('ec2')

# 找出所有 Available 狀態的卷
volumes = ec2.describe_volumes(
    Filters=[{'Name': 'status', 'Values': ['available']}]
)

for volume in volumes['Volumes']:
    volume_id = volume['VolumeId']
    create_time = volume['CreateTime']

    # 如果超過 7 天未使用
    if datetime.now(create_time.tzinfo) - create_time > timedelta(days=7):
        # 創建 Snapshot（保險起見）
        snapshot = ec2.create_snapshot(
            VolumeId=volume_id,
            Description=f'Auto-backup before deletion: {volume_id}'
        )

        # 刪除卷
        ec2.delete_volume(VolumeId=volume_id)
        print(f'Deleted {volume_id}, snapshot: {snapshot["SnapshotId"]}')
```

---

### 5.3 快照生命週期管理 (DLM)

**問題**：

```
快照是增量的，但長期累積也很貴：
- 每天一個快照
- 保留 365 天
- 成本：$0.05/GB-month × 365 snapshots

範例：
- 1TB 卷
- 每日變化：10GB
- 一年快照成本：10GB × 365 × $0.05 = $182.5/month
```

**優化策略**：

```bash
# 使用 Data Lifecycle Manager (DLM)
$ aws dlm create-lifecycle-policy \
    --description "Daily snapshots with 30-day retention" \
    --state ENABLED \
    --execution-role-arn arn:aws:iam::123456789012:role/AWSDataLifecycleManagerDefaultRole \
    --policy-details '{
        "ResourceTypes": ["VOLUME"],
        "TargetTags": [{"Key": "Backup", "Value": "true"}],
        "Schedules": [{
            "Name": "Daily snapshots",
            "CreateRule": {"Interval": 24, "IntervalUnit": "HOURS"},
            "RetainRule": {"Count": 30},
            "CopyTags": true
        }]
    }'

# 或使用 EBS Snapshots Archive（便宜 75%）
$ aws ebs put-snapshot-archive --snapshot-id snap-xxxxx
```

---

## 六、監控和最佳實踐

### 6.1 關鍵指標

**CloudWatch Metrics**：

```
最重要的指標：

1. VolumeQueueLength（佇列長度）：
   - 定義：等待處理的 I/O 請求數量
   - Rule of Thumb：持續 > 1 (每 1000 IOPS) 代表磁碟處理不過來
   - 警報設定：> 2 持續 5 分鐘

2. BurstBalance（Burst 餘額）：
   - 適用於 gp2, st1
   - 警報設定：< 10% 持續 10 分鐘

3. VolumeThroughputPercentage：
   - 實際吞吐量 / Provisioned 吞吐量
   - 持續 100% 代表達到上限

4. VolumeReadOps / VolumeWriteOps：
   - 實際 IOPS
   - 與 Provisioned IOPS 對比
```

**設定警報**：

```bash
$ aws cloudwatch put-metric-alarm \
    --alarm-name ebs-queue-length-high \
    --metric-name VolumeQueueLength \
    --namespace AWS/EBS \
    --statistic Average \
    --period 300 \
    --threshold 2 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --dimensions Name=VolumeId,Value=vol-xxxxx \
    --alarm-actions arn:aws:sns:us-east-1:123456789012:my-topic
```

---

### 6.2 使用 Datadog / Prometheus 進行深度監控

**Datadog 整合**：

```yaml
# datadog.yaml
init_config:

instances:
  - aws_region: us-east-1
    collect_custom_metrics: true
    metrics:
      - VolumeQueueLength
      - BurstBalance
      - VolumeThroughputPercentage
```

**Prometheus + node_exporter**：

```bash
# 安裝 node_exporter
$ wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
$ tar xvfz node_exporter-1.6.1.linux-amd64.tar.gz
$ ./node_exporter &

# 採集 OS 層級的 iostat 數據
# 繪製 Heatmap，觀察所有機器的 I/O 延遲分佈
```

---

## 七、總結

### 7.1 核心要點回顧

**AWS EBS 選擇**：

1. **預設使用 gp3** - 性價比最佳
2. **避免 gp2** - 已過時，成本高且有 Burst 限制
3. **關鍵任務用 io2** - 但成本是 gp3 的 10x

**Azure / GCP**：

1. **Azure**: Balanced PD 是性價比甜蜜點
2. **GCP**: Live Resizing 和 Regional PD 提供更好的運維體驗

**Instance Store**：

1. **極致效能** - 比 EBS 快 10-100 倍
2. **資料易失** - 只用於可重建的資料
3. **最佳場景** - Cache、分散式資料庫、中間暫存

**QoS 和 Noisy Neighbor**：

1. **監控 Burst Balance** - 避免效能突然下降
2. **注意 Instance Level Limit** - 不要浪費 Provisioned IOPS
3. **使用 QoS 保證型儲存** - 關鍵應用避免鄰居干擾

**成本優化**：

1. **gp2 -> gp3 遷移** - 立即節省 20-70%
2. **清理 Orphaned Volumes** - 自動化掃描和刪除
3. **快照生命週期管理** - 使用 DLM 或 Archive

**監控**：

1. **VolumeQueueLength** - 最重要的指標
2. **BurstBalance** - 預防效能下降
3. **設定警報** - 及早發現問題

---

## 參考資料

1. AWS EBS Documentation: <https://docs.aws.amazon.com/ebs/>
2. Azure Managed Disks: <https://learn.microsoft.com/en-us/azure/virtual-machines/managed-disks-overview>
3. GCP Persistent Disks: <https://cloud.google.com/compute/docs/disks>
4. AWS Nitro System: <https://aws.amazon.com/ec2/nitro/>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
