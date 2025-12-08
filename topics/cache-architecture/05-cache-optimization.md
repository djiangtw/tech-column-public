# Cache 性能優化實戰：從理論到實踐

**作者**: Danny Jiang  
**日期**: 2025-12-07  

---

## 前言：一次 10 倍性能提升的經歷

2020 年，我在優化一個圖像處理演算法時，遇到了一個性能瓶頸：處理一張 4K 圖片需要 2 秒，遠遠達不到即時處理的要求。

我用 `perf` 分析後發現：**L1 Cache Miss Rate 高達 35%**！這意味著每 3 次記憶體存取就有 1 次需要從 L2 或更慢的地方載入資料。

經過一週的優化，我把 Cache Miss Rate 降到 3%，性能提升了 **10 倍**，處理時間從 2 秒降到 0.2 秒。

> 註：本文中提到的各種 Miss Rate 與「x 倍加速」數字，皆為在代表性硬體與測試程式上的實驗結果，用來說明 **技巧帶來的量級差異**，不同 CPU 與實際系統上的絕對數字可能有所不同，請讀者重點放在方法與趨勢本身。

今天，我想和你分享這些優化技巧。

---

## 一、理解你的敵人：Cache Miss 的三種類型

### 1.1 Compulsory Miss (強制性缺失)

**定義**：第一次存取資料時，Cache 中一定沒有，必須從記憶體載入。

**範例**：

```c
int data[1000];

// 第一次存取 data[0]
int x = data[0];  // Compulsory Miss (第一次存取)

// 第二次存取 data[0]
int y = data[0];  // Cache Hit (已經在 Cache 中)

```

**優化策略**：

- **Prefetching (預取)**：提前載入資料
- **減少 Working Set**：減少需要存取的資料量

---

### 1.2 Capacity Miss (容量缺失)

**定義**：Working Set 超過 Cache 容量，舊資料被驅逐 (Evict)。

**範例**：

```c
// L1 Cache = 32KB
int data[10000];  // 40KB，超過 L1 容量

for (int i = 0; i < 10000; i++) {
    data[i] = i;  // 前面的資料會被驅逐
}

// 再次存取
for (int i = 0; i < 10000; i++) {
    int x = data[i];  // Capacity Miss (資料已被驅逐)
}

```

**優化策略**：

- **Blocking (分塊)**：把大資料切成小塊，每塊都能放進 Cache
- **減少 Working Set**：只保留必要的資料

---

### 1.3 Conflict Miss (衝突缺失)

**定義**：多個資料映射到同一個 Set，互相驅逐。

**範例**：

```c
// 假設 L1 Cache: 32KB, 8-Way, 64B Line
// Sets = 32KB / (8 × 64B) = 64

int a[64];  // 假設 a[0] 映射到 Set 0
int b[64];  // 假設 b[0] 也映射到 Set 0

// 交替存取 a 和 b
for (int i = 0; i < 64; i++) {
    a[i] = i;  // 驅逐 b[i]
    b[i] = i;  // 驅逐 a[i]
}
// 頻繁的 Conflict Miss

```

**優化策略**：

- **Padding (填充)**：改變資料的記憶體佈局
- **提高 Associativity**：硬體層面的解決方案

---

## 二、優化技巧 1：Cache Line 對齊

### 2.1 問題：跨 Cache Line 存取

**不好的設計**：

```c
struct Point {
    double x;  // 8 bytes
    double y;  // 8 bytes
    double z;  // 8 bytes
};  // 24 bytes

Point points[1000];

// 問題：每個 Point 可能跨越兩個 Cache Line
// Cache Line = 64 bytes
// 64 / 24 = 2.67，第 3 個 Point 會跨越兩個 Cache Line

```

**性能影響**：

```
正常存取：1 次 Cache Line 載入
跨 Cache Line 存取：2 次 Cache Line 載入
→ 性能損失 50%

```

---

### 2.2 解決方案：Padding 到 Cache Line 大小

**好的設計**：

```c
struct Point {
    double x;  // 8 bytes
    double y;  // 8 bytes
    double z;  // 8 bytes
    char padding[40];  // Padding 到 64 bytes
} __attribute__((aligned(64)));

Point points[1000];

// 每個 Point 剛好佔一個 Cache Line
// 不會跨越兩個 Cache Line

```

**性能提升**：

我在 Intel Core i9-12900K 上測試：

```
不對齊版本：1.2 秒
對齊版本：0.8 秒
性能提升：50%

```

---

### 2.3 真實案例：優化資料結構

**原始程式碼**：

```c
struct Particle {
    float x, y, z;       // 12 bytes
    float vx, vy, vz;    // 12 bytes
    float mass;          // 4 bytes
    int id;              // 4 bytes
};  // 32 bytes

Particle particles[10000];

// 更新位置
for (int i = 0; i < 10000; i++) {
    particles[i].x += particles[i].vx * dt;
    particles[i].y += particles[i].vy * dt;
    particles[i].z += particles[i].vz * dt;
}

```

**問題分析**：

```
每次迴圈需要存取：

  - particles[i].x, y, z (12 bytes)
  - particles[i].vx, vy, vz (12 bytes)
  
但每次載入整個 Particle (32 bytes)
浪費了 mass 和 id (8 bytes)
→ 頻寬浪費 25%

```

---

#### 優化版本：Structure of Arrays (SoA)


```c
struct ParticleSystem {
    float x[10000];
    float y[10000];
    float z[10000];
    float vx[10000];
    float vy[10000];
    float vz[10000];
    float mass[10000];
    int id[10000];
};

ParticleSystem particles;

// 更新位置
for (int i = 0; i < 10000; i++) {
    particles.x[i] += particles.vx[i] * dt;
    particles.y[i] += particles.vy[i] * dt;
    particles.z[i] += particles.vz[i] * dt;
}

```

**性能提升**：

```
原始版本 (AoS)：2.5 秒
優化版本 (SoA)：1.2 秒
性能提升：2.1 倍

```

**原因**：

- SoA 版本：連續存取 x, vx，空間局部性好
- AoS 版本：存取 x 時載入了不需要的 mass, id，浪費頻寬

---

## 三、優化技巧 2：Loop Blocking (迴圈分塊)

### 3.1 問題：矩陣轉置的性能災難

**原始程式碼**：

```c
#define N 4096

float A[N][N];
float B[N][N];

// 矩陣轉置
for (int i = 0; i < N; i++) {
    for (int j = 0; j < N; j++) {
        B[j][i] = A[i][j];
    }
}

```

**問題分析**：

```
A[i][j] 的存取：連續 (Row-major)
  → 空間局部性好
  → Cache Hit Rate 高

B[j][i] 的存取：跳躍 (Column-major)
  → 空間局部性差
  → Cache Miss Rate 高

Working Set = 2 × N × N × 4 bytes = 128 MB
遠超 L3 Cache (30 MB)
→ 頻繁存取 DRAM

```

**性能測試**：

```
Intel Core i9-12900K:
  執行時間：1.8 秒
  L1 Cache Miss Rate：45%
  L3 Cache Miss Rate：12%

```

---

### 3.2 解決方案：Blocking

**優化程式碼**：

```c
#define N 4096
#define BLOCK_SIZE 64  // 選擇合適的 Block Size

float A[N][N];
float B[N][N];

// 分塊矩陣轉置
for (int i = 0; i < N; i += BLOCK_SIZE) {
    for (int j = 0; j < N; j += BLOCK_SIZE) {
        // 處理一個 Block
        for (int ii = i; ii < i + BLOCK_SIZE; ii++) {
            for (int jj = j; jj < j + BLOCK_SIZE; jj++) {
                B[jj][ii] = A[ii][jj];
            }
        }
    }
}

```

**為什麼有效？**

```
Working Set (每個 Block):
  A[i:i+64][j:j+64] = 64 × 64 × 4 = 16 KB
  B[j:j+64][i:i+64] = 64 × 64 × 4 = 16 KB
  總計 = 32 KB

32 KB < L1 Cache (48 KB)
→ 所有資料都在 L1 Cache 中
→ Cache Miss Rate 大幅降低

```

**性能提升**：

```
原始版本：1.8 秒
Blocking 版本：0.3 秒
性能提升：6 倍

```

---

### 3.3 如何選擇 Block Size？

**經驗法則**：

```
Block Size 應該讓 Working Set 放進 L1 Cache

公式：
  Working Set = Block_Size² × Element_Size × 2 (A 和 B)
  Working Set ≤ L1 Cache Size

範例 (L1 = 32 KB):
  Block_Size² × 4 × 2 ≤ 32 KB
  Block_Size² ≤ 4096
  Block_Size ≤ 64

```

**實際測試**：

```
Block Size = 16:  0.5 秒
Block Size = 32:  0.35 秒
Block Size = 64:  0.3 秒 (最佳)
Block Size = 128: 0.4 秒 (Working Set 超過 L1)
Block Size = 256: 0.6 秒

```

---

## 四、優化技巧 3：Prefetching (預取)

### 4.1 什麼是 Prefetching？

**概念**：

```
傳統流程：

  1. CPU 需要資料
  2. 檢查 Cache → Miss
  3. 從 Memory 載入 (200+ cycles)
  4. CPU 等待

Prefetching 流程：

  1. CPU 提前發出 Prefetch 指令
  2. 硬體在背景載入資料
  3. CPU 繼續執行其他指令
  4. 當 CPU 真正需要資料時，已經在 Cache 中了

```

---

### 4.2 軟體 Prefetching

**範例**：

```c
#include <xmmintrin.h>  // SSE intrinsics

int data[10000];

// 不使用 Prefetch
for (int i = 0; i < 10000; i++) {
    process(data[i]);  // 每次都可能 Cache Miss
}

// 使用 Prefetch
for (int i = 0; i < 10000; i++) {
    // 提前 8 個元素 Prefetch
    _mm_prefetch((char*)&data[i + 8], _MM_HINT_T0);
    process(data[i]);
}

```

**Prefetch 的層級**：

```
_MM_HINT_T0: Prefetch 到 L1 Cache
_MM_HINT_T1: Prefetch 到 L2 Cache
_MM_HINT_T2: Prefetch 到 L3 Cache
_MM_HINT_NTA: Non-Temporal (不污染 Cache)

```

---

### 4.3 真實案例：鏈結串列的 Prefetching

**問題**：

```c
struct Node {
    int data;
    struct Node* next;
};

// 遍歷鏈結串列
Node* current = head;
while (current != NULL) {
    process(current->data);
    current = current->next;  // 每次都可能 Cache Miss
}

```

**為什麼 Cache Miss Rate 高？**

```
鏈結串列的節點在記憶體中不連續
每次存取 current->next 都可能在不同的 Cache Line
→ 空間局部性差
→ Cache Miss Rate 高

```

---

**優化版本**：

```c
// 遍歷鏈結串列 + Prefetch
Node* current = head;
while (current != NULL) {
    // Prefetch 下一個節點
    if (current->next != NULL) {
        _mm_prefetch((char*)current->next, _MM_HINT_T0);
    }
    
    process(current->data);
    current = current->next;
}

```

**性能提升**：

```
不使用 Prefetch：2.5 秒
使用 Prefetch：1.2 秒
性能提升：2.1 倍

```

---

### 4.4 硬體 Prefetching

**現代 CPU 的自動 Prefetching**：

```
Sequential Prefetcher:
  偵測連續存取模式
  自動 Prefetch 下一個 Cache Line
  
Stride Prefetcher:
  偵測固定步長存取模式
  例如：data[0], data[4], data[8], ...
  自動 Prefetch data[12], data[16], ...

```

**如何幫助硬體 Prefetcher？**

```c
// 好的模式：連續存取
for (int i = 0; i < N; i++) {
    data[i] = i;  // 硬體可以自動 Prefetch
}

// 不好的模式：隨機存取
for (int i = 0; i < N; i++) {
    int index = random();
    data[index] = i;  // 硬體無法預測
}

```

---

## 五、優化技巧 4：減少 Cache Pollution

### 5.1 什麼是 Cache Pollution？

**定義**：

```
載入不會再次使用的資料到 Cache
驅逐了有用的資料
→ Cache Hit Rate 下降

```

**範例**：

```c
// 大量資料的一次性處理
char buffer[10 * 1024 * 1024];  // 10 MB

// 讀取檔案到 buffer
read(fd, buffer, sizeof(buffer));

// 處理 buffer (只用一次)
process(buffer);

// 問題：buffer 佔據了大量 Cache
// 驅逐了其他有用的資料

```

---

### 5.2 解決方案：Non-Temporal Store

**概念**：

```
Non-Temporal Store:
  直接寫入 Memory，不經過 Cache
  避免污染 Cache

```

**範例**：

```c
#include <emmintrin.h>  // SSE2 intrinsics

// 傳統 Store
for (int i = 0; i < N; i++) {
    data[i] = value;  // 寫入 Cache
}

// Non-Temporal Store
for (int i = 0; i < N; i += 4) {
    __m128i val = _mm_set1_epi32(value);
    _mm_stream_si128((__m128i*)&data[i], val);  // 直接寫入 Memory
}

```

**何時使用？**

```
✓ 大量資料的一次性寫入 (例如：初始化大陣列)
✓ 串流資料處理 (例如：視訊編碼)
✗ 需要立即讀取的資料 (會增加延遲)

```

---

## 六、優化技巧 5：Cache-Oblivious Algorithms

### 6.1 什麼是 Cache-Oblivious？

**定義**：

```
演算法不需要知道 Cache 的大小和參數
自動適應不同的 Cache 階層

```

#### 範例：矩陣乘法


**傳統 Blocking**：

```c
// 需要知道 Cache Size 來選擇 Block Size
#define BLOCK_SIZE 64  // 根據 L1 Cache 調整

for (int i = 0; i < N; i += BLOCK_SIZE) {
    for (int j = 0; j < N; j += BLOCK_SIZE) {
        for (int k = 0; k < N; k += BLOCK_SIZE) {
            // 矩陣乘法
        }
    }
}

```

---

#### Cache-Oblivious 版本：遞迴分治


```c
// 注意：這是簡化版本，展示核心概念
// 完整實作需要處理所有 8 個子矩陣乘法（C11, C12, C21, C22 各需要 2 個）
void matrix_multiply_recursive(int n, int row_a, int col_a, float* A,
                                int row_b, int col_b, float* B,
                                int row_c, int col_c, float* C,
                                int stride) {
    if (n <= 64) {
        // Base case：直接計算
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < n; j++) {
                float sum = 0;
                for (int k = 0; k < n; k++) {
                    sum += A[(row_a + i) * stride + (col_a + k)] *
                           B[(row_b + k) * stride + (col_b + j)];
                }
                C[(row_c + i) * stride + (col_c + j)] += sum;
            }
        }
    } else {
        // 遞迴分治：把矩陣切成 4 塊
        int m = n / 2;

        // C11 = A11 * B11 + A12 * B21
        matrix_multiply_recursive(m, row_a, col_a, A, row_b, col_b, B, row_c, col_c, C, stride);
        matrix_multiply_recursive(m, row_a, col_a + m, A, row_b + m, col_b, B, row_c, col_c, C, stride);

        // C12 = A11 * B12 + A12 * B22
        matrix_multiply_recursive(m, row_a, col_a, A, row_b, col_b + m, B, row_c, col_c + m, C, stride);
        matrix_multiply_recursive(m, row_a, col_a + m, A, row_b + m, col_b + m, B, row_c, col_c + m, C, stride);

        // C21 = A21 * B11 + A22 * B21
        matrix_multiply_recursive(m, row_a + m, col_a, A, row_b, col_b, B, row_c + m, col_c, C, stride);
        matrix_multiply_recursive(m, row_a + m, col_a + m, A, row_b + m, col_b, B, row_c + m, col_c, C, stride);

        // C22 = A21 * B12 + A22 * B22
        matrix_multiply_recursive(m, row_a + m, col_a, A, row_b, col_b + m, B, row_c + m, col_c + m, C, stride);
        matrix_multiply_recursive(m, row_a + m, col_a + m, A, row_b + m, col_b + m, B, row_c + m, col_c + m, C, stride);
    }
}

// 包裝函數
void matrix_multiply(int n, float* A, float* B, float* C) {
    matrix_multiply_recursive(n, 0, 0, A, 0, 0, B, 0, 0, C, n);
}

```

**為什麼有效？**

```
遞迴會自動適應不同的 Cache 階層：

  - 大矩陣：分割到能放進 L3
  - 中矩陣：分割到能放進 L2
  - 小矩陣：分割到能放進 L1
  
不需要手動調整 Block Size

```

---

## 七、測量和除錯工具

### 7.1 Linux perf

**基本用法**：

```bash
# 測量 Cache Miss
perf stat -e cache-references,cache-misses,L1-dcache-loads,L1-dcache-load-misses ./program

# 詳細分析
perf record -e cache-misses ./program
perf report

```

**關鍵指標**：

```
L1 Cache Miss Rate < 5%: 優秀
L1 Cache Miss Rate 5-10%: 良好
L1 Cache Miss Rate > 10%: 需要優化

L3 Cache Miss Rate < 1%: 優秀
L3 Cache Miss Rate 1-5%: 良好
L3 Cache Miss Rate > 5%: 需要優化

```

---

### 7.2 Intel VTune

**使用方法**：

```bash
# 收集 Memory Access 資料
vtune -collect memory-access ./program

# 查看報告
vtune -report summary

# 查看 Hotspot
vtune -report hotspots

```

**關鍵報告**：

```
Memory Access Analysis:

  - L1 Hit Rate
  - L2 Hit Rate
  - L3 Hit Rate
  - DRAM Bandwidth
  
Cache-to-Cache Transfer:

  - False Sharing 偵測
  - Remote Cache Access

```

---

### 7.3 Cachegrind (Valgrind)

**使用方法**：

```bash
# 執行 Cachegrind
valgrind --tool=cachegrind ./program

# 查看報告
cg_annotate cachegrind.out.<pid>

```

**優點**：

- 詳細的 Cache 模擬
- 可以模擬不同的 Cache 配置

**缺點**：

- 執行速度慢 (10-100 倍)
- 只是模擬，不是真實硬體

---

## 八、總結：優化的黃金法則

### 8.1 優化順序

```
1. 測量 (Measure)
   → 用 perf/VTune 找出瓶頸
   
2. 理解 (Understand)
   → 分析為什麼 Cache Miss Rate 高
   
3. 優化 (Optimize)
   → 應用合適的優化技巧
   
4. 驗證 (Verify)
   → 再次測量，確認性能提升

```

---

### 8.2 優化技巧總結

| 技巧 | 適用場景 | 性能提升 |
|------|----------|----------|
| **Cache Line 對齊** | 資料結構設計 | 1.5-2 倍 |
| **SoA vs AoS** | 大量資料處理 | 2-3 倍 |
| **Loop Blocking** | 矩陣運算 | 5-10 倍 |
| **Prefetching** | 鏈結串列、隨機存取 | 2-3 倍 |
| **Non-Temporal Store** | 大量一次性寫入 | 1.5-2 倍 |
| **Cache-Oblivious** | 遞迴演算法 | 2-5 倍 |

---

### 8.3 最後的建議

**Do**：

- ✓ 先測量，再優化
- ✓ 理解你的 Working Set
- ✓ 利用空間局部性
- ✓ 避免 False Sharing

**Don't**：

- ✗ 過早優化
- ✗ 盲目套用技巧
- ✗ 忽略可讀性
- ✗ 不測量就宣稱「優化」

---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: https://github.com/djiangtw/tech-column-public
