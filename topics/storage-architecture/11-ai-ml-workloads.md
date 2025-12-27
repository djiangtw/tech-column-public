# AI/ML Workloads - 訓練資料的儲存策略

**作者**: Danny Jiang  
**日期**: 2025-12-09  

---

## 前言：GPU 在等待資料的那些日子

2020 年，我參與了一個計算機視覺專案，目標是訓練一個物件偵測模型（類似 YOLO）。公司剛採購了 8 張 NVIDIA V100 GPU（每張價值 $10,000），團隊對訓練速度充滿期待。

**第一次訓練的慘痛經驗**：

```bash
# 啟動訓練
$ python train.py --dataset imagenet --batch-size 256

Epoch 1/100:
[████░░░░░░░░░░░░░░░░] 5% | ETA: 18 hours
```

**問題出現**：

```bash
# 監控 GPU 使用率
$ nvidia-smi dmon -s u

# gpu   sm   mem   enc   dec
# Idx    %     %     %     %
    0   35    15     0     0
    1   32    12     0     0
    2   38    18     0     0
    ...

# GPU 使用率只有 30-40%！
```

**深入診斷**：

```bash
# 檢查 CPU
$ htop
# CPU 0: 100% (主進程)
# CPU 1-15: 5-10% (閒置)

# 檢查磁碟 I/O
$ iostat -xz 1
# Device: nvme0n1
# %util: 100%
# await: 125ms (嚴重延遲)
# r/s: 8,500 (大量小檔案讀取)
```

**根本原因分析**：

我們的資料集結構是：

```
dataset/
├── train/
│   ├── 0001.jpg (50KB)
│   ├── 0002.jpg (48KB)
│   ├── 0003.jpg (52KB)
│   ...
│   └── 1281167.jpg (ImageNet 訓練集)
```

**問題**：

1. **小檔案災難 (Small Files Problem)**：
   - 128 萬個小 JPEG 檔案
   - 每次讀取需要：Open → Seek → Read → Close
   - 檔案系統 Metadata 查詢成為瓶頸
   - 即使 SSD 的 IOPS 很高，仍然跟不上

2. **單執行緒讀取**：
   - PyTorch DataLoader 預設 `num_workers=0`
   - 主進程負責讀取、解碼、資料增強
   - CPU 0 跑滿 100%，其他核心閒置

3. **沒有預取 (Prefetch)**：
   - GPU 訓練完一個 Batch 後，才開始讀取下一個 Batch
   - GPU 大部分時間在等待資料

**結果**：

- **8 張 V100 GPU 的算力浪費了 60%+**
- **訓練一個 Epoch 需要 18 小時**（理論上應該 < 2 小時）
- **每小時成本：$80（GPU 租用費）**
- **浪費的成本：$80 × 16 小時 × 100 Epochs = $128,000**

**優化後的驚人改善**：

經過一系列優化（本文將詳細介紹），我們達成：

| 指標 | 優化前 | 優化後 | 改善幅度 |
|------|--------|--------|----------|
| GPU 使用率 | 35% | 98% | +180% |
| 每 Epoch 訓練時間 | 18 小時 | 1.8 小時 | -90% |
| 訓練吞吐量 | 450 samples/sec | 3,200 samples/sec | +611% |
| 總訓練時間 (100 Epochs) | 75 天 | 7.5 天 | -90% |
| 成本節省 | - | $115,200 | - |

**深刻的教訓**：

在深度學習訓練中，**I/O 往往是最容易被忽視但影響最大的瓶頸**。

本文將深入探討如何優化 AI/ML 訓練的儲存策略，讓 GPU 不再等待資料。

---

## 一、理解 AI/ML 訓練的 I/O 特性

### 1.1 訓練流程的 I/O 瓶頸

深度學習訓練的典型流程：

```
訓練迴圈：
for epoch in range(100):
    for batch in dataloader:
        # 1. 資料載入 (I/O 密集)
        images, labels = batch
        
        # 2. 資料傳輸 (PCIe 頻寬)
        images = images.to('cuda')
        
        # 3. 前向傳播 (GPU 計算)
        outputs = model(images)
        
        # 4. 反向傳播 (GPU 計算)
        loss.backward()
        
        # 5. 權重更新 (GPU 計算)
        optimizer.step()
```

**時間分配分析**（未優化）：

```
單一 Batch 的時間分解：
- 資料載入：120ms (I/O + CPU 解碼)
- 資料傳輸：5ms (CPU -> GPU)
- GPU 計算：25ms (前向 + 反向 + 更新)

總時間：150ms
GPU 利用率：25ms / 150ms = 16.7%

問題：GPU 有 83.3% 的時間在等待資料！
```

**優化目標**：

```
理想狀態：
- 當 GPU 在計算 Batch N 時，CPU 已經準備好 Batch N+1
- I/O 時間被 GPU 計算時間完全掩蓋 (Overlap)

實現方式：
- 多進程並行讀取 (num_workers)
- 預取 (prefetch)
- 記憶體鎖頁 (pin_memory)
```

---

### 1.2 資料集的 I/O 模式

不同類型的資料集有不同的 I/O 特性：

**計算機視覺 (CV)**：

```
特性：
- 大量小檔案（JPEG/PNG）
- 每個檔案：10KB ~ 500KB
- 需要 CPU 解碼（JPEG 解壓縮）
- 資料增強（Resize, Crop, Flip）

I/O 瓶頸：
- 檔案系統 Metadata 查詢
- 隨機讀取小檔案
- CPU 解碼和增強

典型資料集：
- ImageNet: 128 萬張圖片
- COCO: 33 萬張圖片
```

**自然語言處理 (NLP)**：

```
特性：
- 中等大小檔案（文字檔、Tokenized 資料）
- 每個檔案：1KB ~ 10MB
- 需要 Tokenization（CPU 密集）

I/O 瓶頸：
- 文字解析和 Tokenization
- 動態 Padding

典型資料集：
- Wikipedia Dump: 20GB+
- Common Crawl: TB 級
```

**大型語言模型 (LLM)**：

```
特性：
- 超大檔案（預處理後的 Token 序列）
- 單一檔案：數 GB ~ 數 TB
- 順序讀取為主

I/O 瓶頸：
- 磁碟吞吐量
- 記憶體映射 (mmap) 效率

典型資料集：
- The Pile: 825GB
- RedPajama: 1.2TB
```

## 二、PyTorch DataLoader 優化

### 2.1 關鍵參數詳解

PyTorch 的 `DataLoader` 是訓練流程的 I/O 門戶，正確配置可以帶來數倍的效能提升：

**1. `num_workers` - 並行讀取進程數**

```python
from torch.utils.data import DataLoader

# 錯誤：單執行緒讀取
dataloader = DataLoader(dataset, batch_size=256, num_workers=0)

# 正確：多進程並行讀取
dataloader = DataLoader(dataset, batch_size=256, num_workers=8)
```

**如何選擇 `num_workers`？**

```python
import multiprocessing

# 方法 1：CPU 核心數的 1/2
num_workers = multiprocessing.cpu_count() // 2

# 方法 2：根據 Batch Size 調整
# 經驗法則：每個 worker 處理 32-64 個樣本
num_workers = batch_size // 32

# 方法 3：實驗測試（最準確）
for num_workers in [0, 2, 4, 8, 16]:
    # 測試訓練速度
    # 選擇速度最快且 CPU 使用率合理的值
```

**注意事項**：

```
num_workers 並非越大越好：

過小 (0-2)：
- I/O 成為瓶頸
- GPU 閒置

適中 (4-8)：
- I/O 和計算平衡
- 最佳效能

過大 (16+)：
- CPU 切換開銷 (Context Switch)
- RAM 佔用過高（每個 worker 複製一份資料集）
- 可能比適中值更慢
```

**2. `pin_memory` - 記憶體鎖頁**

```python
dataloader = DataLoader(
    dataset,
    batch_size=256,
    num_workers=8,
    pin_memory=True  # 關鍵優化
)
```

**原理**：

```
預設行為 (pin_memory=False)：
CPU RAM (Pageable Memory) -> GPU VRAM
- OS 可能會將記憶體 Swap 到磁碟
- 傳輸時需要先鎖定記憶體
- 傳輸速度：~3 GB/s

啟用 pin_memory=True：
CPU RAM (Page-Locked Memory) -> GPU VRAM
- 記憶體保證在實體 RAM 中
- 可以使用 DMA (Direct Memory Access)
- 傳輸速度：~12 GB/s (PCIe 3.0 x16)

效能提升：
- 資料傳輸時間減少 75%
- 對大 Batch Size 特別有效
```

**代價**：

```
Page-Locked Memory 是稀缺資源：
- 佔用的記憶體無法被 OS Swap
- 過度使用會導致系統記憶體不足

建議：
- 訓練時：pin_memory=True
- 推論時：pin_memory=False（節省記憶體）
```

**3. `prefetch_factor` - 預取倍數**

```python
dataloader = DataLoader(
    dataset,
    batch_size=256,
    num_workers=8,
    pin_memory=True,
    prefetch_factor=2  # 每個 worker 預取 2 個 Batch
)
```

**作用**：

```
prefetch_factor=2 (預設)：
- 每個 worker 預先載入 2 個 Batch
- 總預取量：num_workers × prefetch_factor = 8 × 2 = 16 Batches

記憶體佔用計算：
- Batch Size: 256
- 圖片大小：224×224×3 (ImageNet)
- 單一 Batch：256 × 224 × 224 × 3 × 4 bytes = 154 MB
- 總預取記憶體：16 × 154 MB = 2.5 GB

調整策略：
- RAM 充足：prefetch_factor=4（更激進的預取）
- RAM 受限：prefetch_factor=1（節省記憶體）
```

**4. `persistent_workers` - 保持 Workers 存活**

```python
dataloader = DataLoader(
    dataset,
    batch_size=256,
    num_workers=8,
    pin_memory=True,
    prefetch_factor=2,
    persistent_workers=True  # PyTorch 1.7+
)
```

**為什麼重要？**

```
預設行為 (persistent_workers=False)：
每個 Epoch 結束：
1. 關閉所有 worker 進程
2. 下個 Epoch 開始時重新創建 workers
3. 重新載入 Dataset 到每個 worker

開銷：
- 進程創建：~2-5 秒
- Dataset 複製：~5-10 秒（取決於 Dataset 大小）
- 每個 Epoch 浪費 10-15 秒

啟用 persistent_workers=True：
- Workers 在整個訓練過程中保持存活
- 只在第一個 Epoch 創建一次
- 100 Epochs 節省：(10 秒 × 99) = 16.5 分鐘
```

---

### 2.2 完整優化範例

**優化前**：

```python
# 基礎配置
train_loader = DataLoader(
    train_dataset,
    batch_size=256,
    shuffle=True
)

# 訓練速度：450 samples/sec
# GPU 使用率：35%
```

**優化後**：

```python
train_loader = DataLoader(
    train_dataset,
    batch_size=256,
    shuffle=True,
    num_workers=8,           # 並行讀取
    pin_memory=True,         # 加速 CPU->GPU 傳輸
    prefetch_factor=2,       # 預取
    persistent_workers=True  # 保持 workers 存活
)

# 訓練速度：2,800 samples/sec (+522%)
# GPU 使用率：92%
```

## 三、TensorFlow tf.data API 優化

### 3.1 Pipeline 設計原則

TensorFlow 的 `tf.data` API 採用 **Pipeline (管線化)** 設計，核心思想是讓 I/O 和計算重疊：

**基礎 Pipeline**：

```python
import tensorflow as tf

# 錯誤：未優化的 Pipeline
dataset = tf.data.TFRecordDataset(filenames)
dataset = dataset.map(parse_function)
dataset = dataset.batch(256)

# 問題：
# - 單執行緒讀取
# - 沒有預取
# - I/O 和計算串行執行
```

**優化後的 Pipeline**：

```python
dataset = tf.data.TFRecordDataset(
    filenames,
    num_parallel_reads=tf.data.AUTOTUNE  # 並行讀取多個檔案
)

dataset = dataset.interleave(
    lambda x: tf.data.TFRecordDataset(x),
    cycle_length=8,  # 同時讀取 8 個檔案
    num_parallel_calls=tf.data.AUTOTUNE
)

dataset = dataset.map(
    parse_function,
    num_parallel_calls=tf.data.AUTOTUNE  # 並行解析
)

dataset = dataset.cache()  # 快取到記憶體（若資料集夠小）

dataset = dataset.batch(256)

dataset = dataset.prefetch(
    buffer_size=tf.data.AUTOTUNE  # 自動調整預取量
)
```

---

### 3.2 關鍵操作詳解

**1. `interleave()` - 並行讀取多個檔案**

```python
dataset = dataset.interleave(
    map_func=lambda x: tf.data.TFRecordDataset(x),
    cycle_length=8,  # 同時處理 8 個檔案
    num_parallel_calls=tf.data.AUTOTUNE
)
```

**為什麼需要？**

```
問題：
- 資料集通常分成多個 TFRecord 檔案（如 1,000 個）
- 預設是順序讀取：File 1 -> File 2 -> File 3 ...
- 單一檔案讀取無法充分利用 SSD 的並行能力

interleave 的作用：
- 同時開啟 8 個檔案
- 交錯讀取（File 1 的 1 筆 -> File 2 的 1 筆 -> ...）
- 充分利用 SSD 的多佇列特性

效能提升：
- 單檔案讀取：500 MB/s
- 8 檔案並行：3,200 MB/s (接近 NVMe 上限)
```

**2. `map()` - 並行資料轉換**

```python
def parse_and_augment(example_proto):
    # 解析 TFRecord
    features = tf.io.parse_single_example(example_proto, feature_description)

    # 解碼 JPEG
    image = tf.io.decode_jpeg(features['image'], channels=3)

    # 資料增強
    image = tf.image.random_flip_left_right(image)
    image = tf.image.random_crop(image, [224, 224, 3])

    return image, features['label']

dataset = dataset.map(
    parse_and_augment,
    num_parallel_calls=tf.data.AUTOTUNE  # 多核 CPU 並行處理
)
```

**`AUTOTUNE` 的魔力**：

```
tf.data.AUTOTUNE 會動態調整並行度：
- 監控 CPU 使用率
- 監控 I/O 等待時間
- 自動增加或減少並行執行緒數

優勢：
- 不需要手動調參
- 適應不同硬體環境
```

**3. `cache()` - 快取資料**

```python
# 策略 1：快取到記憶體（資料集小於 RAM）
dataset = dataset.cache()

# 策略 2：快取到磁碟（資料集大於 RAM）
dataset = dataset.cache('/tmp/cache')
```

**何時使用？**

```
適用場景：
- 資料集小於 RAM（如 CIFAR-10: 170MB）
- 多個 Epoch 訓練
- 資料增強在 cache() 之後

效果：
- 第一個 Epoch：從磁碟讀取（慢）
- 後續 Epochs：從記憶體讀取（快 100x+）

注意：
- cache() 的位置很重要
- 放在資料增強之前：快取原始資料
- 放在資料增強之後：快取增強後的資料（佔用更多記憶體）
```

**4. `prefetch()` - 預取（最重要）**

```python
dataset = dataset.prefetch(buffer_size=tf.data.AUTOTUNE)
```

**必須放在 Pipeline 的最後！**

```
作用：
- 當 GPU 在訓練 Batch N 時
- CPU 已經在準備 Batch N+1, N+2, ...

視覺化：
時間軸：
GPU: [Batch 1] [Batch 2] [Batch 3] ...
CPU:     [Batch 2] [Batch 3] [Batch 4] ...

結果：
- GPU 永遠不需要等待資料
- I/O 時間被完全掩蓋
```

---

## 四、檔案格式的選擇

### 4.1 格式對比

不同檔案格式對 I/O 效能有巨大影響：

| 格式 | I/O 模式 | 讀取速度 | 適用場景 | 缺點 |
|------|----------|----------|----------|------|
| **Raw Images** (JPG/PNG) | 隨機小 I/O | ★☆☆☆☆ | 小規模實驗、除錯 | 檔案系統 Metadata 負擔重，Inode 耗盡 |
| **TFRecord** | 順序 I/O | ★★★★☆ | TensorFlow 生態、雲端訓練 | 格式封閉，無法隨機存取 |
| **HDF5** | 隨機/順序 | ★★★☆☆ | 科學計算、矩陣數據 | 並行讀取有鎖競爭 |
| **Parquet** | 列式 I/O | ★★★★☆ | NLP、Tabular Data | 讀取圖像需要解碼 |
| **WebDataset** (Tar) | 順序 I/O | ★★★★★ | PyTorch 大規模訓練 | 基於 Tar，易於管理 |

---

### 4.2 實際效能測試

**測試環境**：

- 資料集：ImageNet (128 萬張圖片)
- 硬體：NVMe SSD, 16-core CPU, NVIDIA V100

**場景 A：Raw JPEG 檔案**

```python
# 資料集結構
dataset/
├── train/
│   ├── 0001.jpg
│   ├── 0002.jpg
│   ...
│   └── 1281167.jpg
```

**結果**：

```
讀取速度：450 samples/sec
磁碟 IOPS：8,500 (大量隨機讀取)
磁碟吞吐量：220 MB/s (遠低於 SSD 能力)
GPU 使用率：35%

問題：
- 檔案系統 Metadata 查詢成為瓶頸
- 即使 SSD IOPS 很高，仍然跟不上
```

**場景 B：TFRecord 格式**

```bash
# 轉換為 TFRecord
$ python convert_to_tfrecord.py \
    --input_dir dataset/train \
    --output_dir dataset/tfrecords \
    --num_shards 128

# 結果：128 個 TFRecord 檔案，每個約 1GB
```

```python
# 讀取 TFRecord
filenames = tf.io.gfile.glob('dataset/tfrecords/*.tfrecord')
dataset = tf.data.TFRecordDataset(filenames, num_parallel_reads=8)
```

**結果**：

```
讀取速度：2,500 samples/sec (+456%)
磁碟 IOPS：250 (順序讀取)
磁碟吞吐量：3,200 MB/s (接近 NVMe 上限)
GPU 使用率：90%

改善：
- 順序讀取消除了 Seek Time
- 磁碟吞吐量跑滿
```

**場景 C：WebDataset (PyTorch)**

```bash
# 轉換為 WebDataset (Tar 格式)
$ python convert_to_webdataset.py \
    --input_dir dataset/train \
    --output_dir dataset/webdataset \
    --shard_size 1000

# 結果：1,281 個 Tar 檔案
```

```python
import webdataset as wds

dataset = wds.WebDataset('dataset/webdataset/shard-{000000..001280}.tar')
dataset = dataset.decode('pil').to_tuple('jpg', 'cls')
dataset = dataset.batched(256)
```

**結果**：

```
讀取速度：3,200 samples/sec (+611%)
磁碟 IOPS：180 (順序讀取)
磁碟吞吐量：3,500 MB/s
GPU 使用率：98%

優勢：
- 基於標準 Tar 格式，易於管理
- 支援 Streaming（無需下載完整資料集）
- PyTorch 生態系的最佳選擇
```

## 五、GPU Direct Storage (GDS)

### 5.1 傳統 I/O 路徑的瓶頸

**傳統資料載入流程**：

```
Storage (NVMe SSD)
    ↓ (PCIe DMA)
System RAM
    ↓ (CPU Copy)
GPU VRAM

問題：
1. 資料需要經過 System RAM（Bounce Buffer）
2. CPU 負責搬運資料（CPU Overhead）
3. 佔用 PCIe 頻寬兩次（SSD->RAM, RAM->GPU）
```

**瓶頸分析**：

```
假設：
- NVMe SSD 頻寬：7 GB/s (PCIe 4.0 x4)
- PCIe 3.0 x16 頻寬：16 GB/s
- CPU 記憶體複製速度：20 GB/s

實際吞吐量：
- SSD -> RAM：7 GB/s
- RAM -> GPU：12 GB/s (受 PCIe 限制)
- CPU 使用率：30% (搬運資料)

問題：
- CPU 資源浪費在資料搬運
- 延遲增加（多一次複製）
```

---

### 5.2 GPU Direct Storage 原理

**GDS (NVIDIA Magnum IO) 架構**：

```
Storage (NVMe SSD)
    ↓ (Direct PCIe DMA)
GPU VRAM

優勢：
1. 繞過 System RAM
2. 繞過 CPU
3. 直接從 SSD 到 GPU VRAM
```

**技術細節**：

```
實現方式：
1. cuFile API (NVIDIA 提供)
2. 需要支援的 NVMe 驅動程式
3. 需要支援的檔案系統（ext4, XFS）

硬體需求：
- NVIDIA GPU：A100, H100, RTX 3090+
- NVMe SSD：支援 PCIe P2P (Peer-to-Peer)
- CPU/主機板：支援 PCIe ACS (Access Control Services)
```

---

### 5.3 效能提升

**測試場景**：載入高解析度醫療影像（4K × 4K × 16-bit）

**傳統方式**：

```python
import numpy as np
import torch

# 從 SSD 讀取到 RAM
data = np.load('medical_image.npy')  # 128 MB

# 複製到 GPU
data_gpu = torch.from_numpy(data).cuda()

# 時間：
# - SSD -> RAM：18ms
# - RAM -> GPU：11ms
# - 總計：29ms
```

**使用 GDS**：

```python
import cupy as cp
from cufile import CuFile

# 直接從 SSD 讀取到 GPU VRAM
cf = CuFile('medical_image.npy', 'r')
data_gpu = cp.empty((4096, 4096), dtype=cp.uint16)
cf.read(data_gpu)

# 時間：
# - SSD -> GPU：8ms
# - 總計：8ms

# 提升：29ms -> 8ms (3.6x 加速)
```

**大規模訓練的效益**：

```
假設：
- 訓練 100 Epochs
- 每 Epoch 10,000 Batches
- 每 Batch 載入時間節省：21ms

總節省時間：
21ms × 10,000 × 100 = 21,000 秒 ≈ 5.8 小時

成本節省：
5.8 小時 × $3/hour (A100 租用費) = $17.4 per training run
```

## 六、Checkpoint 策略

### 6.1 Checkpoint 的 I/O 挑戰

大型模型的 Checkpoint 是一個巨大的 I/O 突發：

**問題規模**：

```
GPT-3 (175B 參數)：
- FP32：175B × 4 bytes = 700 GB
- FP16：175B × 2 bytes = 350 GB
- 加上 Optimizer State (Adam)：350 GB × 3 = 1.05 TB

寫入時間（傳統方式）：
- NVMe SSD 寫入速度：3 GB/s
- 時間：350 GB / 3 GB/s = 117 秒 ≈ 2 分鐘

影響：
- 訓練暫停 2 分鐘
- GPU 閒置
- 如果每 30 分鐘存一次，浪費 6.7% 的訓練時間
```

---

### 6.2 優化策略

**1. 異步 Checkpoint (Async Checkpointing)**

```python
import torch
import threading

def async_save_checkpoint(model, optimizer, path):
    # 將模型權重複製到 CPU RAM（快速）
    state_dict_cpu = {k: v.cpu() for k, v in model.state_dict().items()}

    # 在背景執行緒中寫入磁碟（慢速）
    def save_thread():
        torch.save({
            'model': state_dict_cpu,
            'optimizer': optimizer.state_dict()
        }, path)

    thread = threading.Thread(target=save_thread)
    thread.start()

    # 訓練立即繼續，不等待寫入完成
    return thread

# 使用
checkpoint_thread = async_save_checkpoint(model, optimizer, 'checkpoint.pt')
# 訓練繼續...
# 在下次 checkpoint 前確保上次寫入完成
checkpoint_thread.join()
```

**效果**：

```
同步 Checkpoint：
- GPU -> CPU：2 秒
- CPU -> Disk：120 秒
- 訓練暫停：122 秒

異步 Checkpoint：
- GPU -> CPU：2 秒（訓練暫停）
- CPU -> Disk：120 秒（背景執行，訓練繼續）
- 訓練暫停：2 秒

改善：122 秒 -> 2 秒 (61x 加速)
```

**2. 使用 `safetensors` 格式**

```python
from safetensors.torch import save_file, load_file

# 傳統方式（Pickle）
torch.save(model.state_dict(), 'model.pt')
# 問題：
# - 序列化慢
# - 載入時需要執行 Python 程式碼（安全風險）
# - 無法部分載入

# safetensors 方式
save_file(model.state_dict(), 'model.safetensors')
# 優勢：
# - Zero-copy（記憶體映射）
# - 載入速度快 3-5x
# - 安全（純資料，無程式碼）
# - 支援部分載入
```

**效能對比**：

```
模型：GPT-2 (1.5B 參數)

torch.save (Pickle)：
- 寫入時間：45 秒
- 載入時間：38 秒

safetensors：
- 寫入時間：12 秒 (3.75x 加速)
- 載入時間：8 秒 (4.75x 加速)
```

**3. Sharded Checkpoints (分片存檔)**

```python
# DeepSpeed / FSDP (Fully Sharded Data Parallel)
# 每個 GPU 只存自己的權重分片

# GPU 0 存：model_shard_0.pt
# GPU 1 存：model_shard_1.pt
# ...
# GPU 7 存：model_shard_7.pt

# 優勢：
# - 並行寫入（8 個 GPU 同時寫入）
# - 避免主節點 I/O 塞車
# - 寫入時間：350 GB / 8 / 3 GB/s = 14.6 秒
```

---

### 6.3 Checkpoint 頻率策略

**如何決定 Checkpoint 頻率？**

```
公式（基於故障恢復成本）：
Optimal Interval ≈ √(2 × Checkpoint Cost × MTBF)

其中：
- Checkpoint Cost：存檔時間
- MTBF：平均故障間隔時間

範例：
- Checkpoint Cost：2 分鐘
- MTBF：48 小時（雲端 GPU 實例）

Optimal Interval ≈ √(2 × 2 min × 2880 min) ≈ 107 分鐘

實務建議：
- 每 30 分鐘或 1 小時存一次
- 保留最近 3-5 個 Checkpoints
- 定期存長期備份（每 10 Epochs）
```

---

## 七、總結

### 7.1 核心要點回顧

**PyTorch 優化**：

1. **`num_workers=8`** - 並行讀取
2. **`pin_memory=True`** - 加速 CPU->GPU 傳輸
3. **`prefetch_factor=2`** - 預取
4. **`persistent_workers=True`** - 保持 workers 存活

**TensorFlow 優化**：

1. **`interleave()`** - 並行讀取多個檔案
2. **`map(num_parallel_calls=AUTOTUNE)`** - 並行資料轉換
3. **`cache()`** - 快取資料（若資料集夠小）
4. **`prefetch(AUTOTUNE)`** - 預取（必須放最後）

**檔案格式選擇**：

- **小規模實驗**：Raw Images（方便除錯）
- **TensorFlow 生態**：TFRecord
- **PyTorch 大規模訓練**：WebDataset (Tar)
- **NLP/Tabular**：Parquet

**進階技術**：

- **GPU Direct Storage**：繞過 CPU，直接從 SSD 到 GPU（3-8x 加速）
- **異步 Checkpoint**：訓練不暫停（61x 加速）
- **safetensors 格式**：更快、更安全（3-5x 加速）
- **Sharded Checkpoints**：分散式並行寫入

**通用原則**：

- **I/O 和計算重疊**：讓 GPU 永遠不等待資料
- **減少小檔案**：使用打包格式（TFRecord, WebDataset）
- **充分利用並行**：多進程、多檔案、多佇列

---

### 7.2 下一篇預告

下一篇文章將探討 **Cloud Storage - 雲端環境中的 SSD 管理**：

- AWS EBS 類型選擇（gp3 vs io2 vs st1）
- Instance Store (NVMe) 的效能與限制
- 雲端環境的 I/O QoS 機制
- 多租戶環境的 Noisy Neighbor 問題
- 成本優化策略

---

## 參考資料

1. PyTorch DataLoader Documentation: <https://pytorch.org/docs/stable/data.html>
2. TensorFlow tf.data Performance Guide: <https://www.tensorflow.org/guide/data_performance>
3. NVIDIA GPU Direct Storage: <https://developer.nvidia.com/gpudirect-storage>
4. WebDataset Documentation: <https://github.com/webdataset/webdataset>
5. safetensors: <https://github.com/huggingface/safetensors>

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: <https://github.com/djiangtw/tech-column-public>
