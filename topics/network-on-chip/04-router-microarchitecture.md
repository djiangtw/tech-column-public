# Router 微架構設計：從 Pipeline 到硬體實作

**作者**: Danny Jiang  
**日期**: 2025-12-07  

---

## 前言：Router 的複雜度

NoC Router 的設計遠比表面看起來複雜。

一個看似簡單的功能：**"把封包從 Input Port 轉送到 Output Port"**

卻需要 **4 個 Pipeline Stage**：RC (Route Compute)、VA (Virtual Channel Allocation)、SA (Switch Allocation)、ST (Switch Traversal)。

每個 Stage 都有自己的邏輯、Arbiter、State Machine。

當工程師第一次在 Waveform 中看到一個 Flit 經過這 4 個 Stage 時，往往會驚訝於其複雜度：

**Router 不只是一個 Crossbar，而是一個精密的 Pipeline 處理器。**

---

## 一、Router 的基本架構

### 1.1 Router 的組成

**核心組件**：

```
Router 由 5 個主要組件組成：

1. Input Buffer:
   - 儲存進來的 Flit
   - 每個 Input Port 有多個 Virtual Channel (VC)
   - 每個 VC 有自己的 Buffer

2. Route Compute (RC):
   - 計算封包的輸出 Port
   - 實作路由演算法（如 XY Routing）

3. Virtual Channel Allocator (VA):
   - 為封包分配下游 Router 的 VC
   - 避免 Head-of-Line Blocking

4. Switch Allocator (SA):
   - 為封包分配 Crossbar 的連接
   - 解決多個封包競爭同一個 Output Port 的衝突

5. Crossbar:
   - 實體的連接矩陣
   - 將 Input Port 連接到 Output Port
```

**圖示**：

```
                    Router
┌─────────────────────────────────────────┐
│                                         │
│  Input Buffers                          │
│  ┌──────┐  ┌──────┐  ┌──────┐          │
│  │ VC0  │  │ VC1  │  │ VC2  │          │
│  │Buffer│  │Buffer│  │Buffer│          │
│  └───┬──┘  └───┬──┘  └───┬──┘          │
│      │         │         │              │
│      └─────────┴─────────┘              │
│              │                          │
│         ┌────▼────┐                     │
│         │   RC    │ Route Compute       │
│         └────┬────┘                     │
│              │                          │
│         ┌────▼────┐                     │
│         │   VA    │ VC Allocator        │
│         └────┬────┘                     │
│              │                          │
│         ┌────▼────┐                     │
│         │   SA    │ Switch Allocator    │
│         └────┬────┘                     │
│              │                          │
│         ┌────▼────┐                     │
│         │Crossbar │                     │
│         └────┬────┘                     │
│              │                          │
│         Output Ports                    │
│         ┌────┴────┐                     │
│         │  N S E W│                     │
│         └─────────┘                     │
└─────────────────────────────────────────┘
```

---

### 1.2 Router Pipeline

**4-Stage Pipeline**：

```
Stage 1: RC (Route Compute)
  - 讀取封包的目的地地址
  - 計算輸出 Port
  - 延遲：1 cycle

Stage 2: VA (Virtual Channel Allocation)
  - 為封包分配下游 Router 的 VC
  - 檢查下游 VC 是否有空位
  - 延遲：1 cycle

Stage 3: SA (Switch Allocation)
  - 為封包分配 Crossbar 的連接
  - 解決多個封包競爭同一個 Output Port
  - 延遲：1 cycle

Stage 4: ST (Switch Traversal)
  - Flit 通過 Crossbar
  - 傳輸到下游 Router
  - 延遲：1 cycle

總延遲：4 cycles (Hop Latency)
```

**Pipeline 圖示**：

```
Cycle:  1    2    3    4    5    6    7    8
        ┌────┬────┬────┬────┬────┬────┬────┐
Flit 0: │ RC │ VA │ SA │ ST │    │    │    │
        ├────┼────┼────┼────┼────┼────┼────┤
Flit 1: │    │ RC │ VA │ SA │ ST │    │    │
        ├────┼────┼────┼────┼────┼────┼────┤
Flit 2: │    │    │ RC │ VA │ SA │ ST │    │
        ├────┼────┼────┼────┼────┼────┼────┤
Flit 3: │    │    │    │ RC │ VA │ SA │ ST │
        └────┴────┴────┴────┴────┴────┴────┘

吞吐量：1 Flit/cycle (Pipeline 滿載時)
延遲：4 cycles (單個 Flit)
```

---

## 二、Input Buffer 的設計

### 2.1 Buffer 的結構

**FIFO Buffer**：

```
每個 Virtual Channel 有一個 FIFO Buffer:

┌─────────────────────────────────────┐
│  VC0 Buffer (8 Flits)               │
│  ┌────┬────┬────┬────┬────┬────┐   │
│  │ F0 │ F1 │ F2 │ F3 │    │    │   │
│  └────┴────┴────┴────┴────┴────┘   │
│   ↑                          ↑      │
│  Head                       Tail    │
│  (Read Pointer)      (Write Pointer)│
└─────────────────────────────────────┘

操作：
  Enqueue (寫入):
    buffer[tail] = flit
    tail = (tail + 1) % 8
  
  Dequeue (讀取):
    flit = buffer[head]
    head = (head + 1) % 8
  
  Empty:
    head == tail
  
  Full:
    (tail + 1) % 8 == head
```

---

### 2.2 Buffer 的大小選擇

**權衡**：

```
Buffer 太小：
  ❌ 容易塞滿
  ❌ Credit Return 頻繁
  ❌ 吞吐量降低

Buffer 太大：
  ❌ 面積增加
  ❌ 功耗增加
  ❌ 延遲增加（Flit 在 Buffer 中等待）

業界常見配置：
  - 每個 VC：4-8 Flits
  - 每個 Input Port：4 個 VC
  - 總 Buffer：16-32 Flits per Input Port
```

**面積估算**：

```
假設：
  Flit 寬度：128 bits (16 bytes)
  每個 VC：8 Flits
  每個 Input Port：4 個 VC
  
Buffer 面積：
  128 bits × 8 Flits × 4 VCs = 4,096 bits = 512 bytes
  
SRAM 面積 (7nm)：
  ~0.02 mm² per 512 bytes
  
5 個 Input Port (N, S, E, W, Local):
  5 × 0.02 = 0.1 mm²
  
結論：
  Buffer 佔 Router 面積的 ~30-40%
```

> 註：以上 Buffer 面積估算為基於代表性 SRAM 配置的估計值，
> 用來說明 Buffer 在 Router 面積中的占比，
> 實際數字會依製程節點、SRAM 編譯器與 VC 配置而異。

---

### 2.3 Chisel 實作範例

**FIFO Buffer**：

```scala
class FIFOBuffer(depth: Int, width: Int) extends Module {
  val io = IO(new Bundle {
    val enq = Flipped(Decoupled(UInt(width.W)))  // 寫入介面
    val deq = Decoupled(UInt(width.W))           // 讀取介面
    val count = Output(UInt(log2Ceil(depth+1).W)) // 當前數量
  })
  
  // Buffer 儲存
  val buffer = Reg(Vec(depth, UInt(width.W)))
  val head = RegInit(0.U(log2Ceil(depth).W))
  val tail = RegInit(0.U(log2Ceil(depth).W))
  val empty = head === tail
  val full = (tail + 1.U) % depth.U === head
  
  // Enqueue (寫入)
  io.enq.ready := !full
  when (io.enq.fire) {
    buffer(tail) := io.enq.bits
    tail := (tail + 1.U) % depth.U
  }
  
  // Dequeue (讀取)
  io.deq.valid := !empty
  io.deq.bits := buffer(head)
  when (io.deq.fire) {
    head := (head + 1.U) % depth.U
  }
  
  // Count
  io.count := Mux(tail >= head, tail - head, depth.U + tail - head)
}
```

---

## 三、Route Compute (RC)

### 3.1 RC 的功能

**輸入**：

```
- Flit Header (包含目的地地址)
- 當前 Router 的座標 (x, y)

輸出：
- Output Port (N, S, E, W, Local)
```

**XY Routing 實作**：

```scala
class RouteCompute(val x: Int, val y: Int) extends Module {
  val io = IO(new Bundle {
    val dest_x = Input(UInt(4.W))
    val dest_y = Input(UInt(4.W))
    val output_port = Output(UInt(3.W))  // 0=Local, 1=N, 2=S, 3=E, 4=W
  })
  
  // XY Routing Logic
  when (x.U < io.dest_x) {
    io.output_port := 3.U  // East
  } .elsewhen (x.U > io.dest_x) {
    io.output_port := 4.U  // West
  } .elsewhen (y.U < io.dest_y) {
    io.output_port := 1.U  // North
  } .elsewhen (y.U > io.dest_y) {
    io.output_port := 2.U  // South
  } .otherwise {
    io.output_port := 0.U  // Local
  }
}
```

---

### 3.2 RC 的優化

**Speculative RC**：

```
問題：
  RC 需要等待 Head Flit 到達才能計算
  → 浪費 1 cycle

優化：
  在 Head Flit 還在前一個 Router 時，就開始計算
  → 提前 1 cycle
  
實作：
  前一個 Router 在發送 Head Flit 時，同時發送目的地地址
  → 下游 Router 可以提前計算
  
延遲降低：
  4 cycles → 3 cycles
```

**Look-Ahead Routing**：

```
概念：
  在當前 Router 計算下下一跳的路由
  → 下一個 Router 不需要 RC Stage
  
範例：
  Router A 計算：
    - 下一跳：Router B (East)
    - 下下一跳：Router C (North)
  
  Router A 發送給 Router B：
    - Flit + "下一跳是 North"
  
  Router B 直接使用 "North"，跳過 RC Stage
  
延遲降低：
  4 cycles → 3 cycles
```

---

## 四、Virtual Channel Allocator (VA)

### 4.1 VA 的功能

**目的**：

```
為封包分配下游 Router 的 Virtual Channel

為什麼需要 VA？
  - 避免 Head-of-Line Blocking
  - 不同類型的訊息使用不同的 VC（避免 Protocol Deadlock）
```

**輸入**：

```
- 當前 VC 的 Head Flit
- Output Port (來自 RC)
- 下游 Router 的 VC 狀態（哪些 VC 有空位）

輸出：
- 分配的下游 VC ID
```

---

### 4.2 VA 的 Arbiter

**問題**：

```
多個 Input VC 競爭同一個 Output VC

範例：
  Input VC0, VC1, VC2 都想使用 Output VC0
  → 需要 Arbiter 決定誰獲勝
```

**Round-Robin Arbiter**：

```
原理：
  輪流給每個 Input VC 機會
  
範例：
  Cycle 1: VC0 獲勝
  Cycle 2: VC1 獲勝
  Cycle 3: VC2 獲勝
  Cycle 4: VC0 獲勝
  ...

優點：
  ✅ 公平
  ✅ 實作簡單

缺點：
  ❌ 延遲不可預測
```

**Priority Arbiter**：

```
原理：
  給每個 Input VC 一個優先級
  優先級高的先獲勝
  
範例：
  Priority: VC2 > VC1 > VC0
  → VC2 總是優先

優點：
  ✅ 延遲可預測（高優先級）

缺點：
  ❌ 不公平（低優先級可能餓死）
```

---

### 4.3 Chisel 實作範例

**Round-Robin Arbiter**：

```scala
class RoundRobinArbiter(n: Int) extends Module {
  val io = IO(new Bundle {
    val request = Input(Vec(n, Bool()))   // n 個請求
    val grant = Output(Vec(n, Bool()))    // n 個授權
  })
  
  val priority = RegInit(0.U(log2Ceil(n).W))  // 當前優先級
  
  // 找到第一個請求
  val grant_oh = Wire(Vec(n, Bool()))
  grant_oh := VecInit(Seq.fill(n)(false.B))
  
  // Round-Robin 邏輯
  val found = Wire(Bool())
  found := false.B
  
  for (i <- 0 until n) {
    val idx = (priority + i.U) % n.U
    when (!found && io.request(idx)) {
      grant_oh(idx) := true.B
      found := true.B
      priority := (idx + 1.U) % n.U  // 更新優先級
    }
  }
  
  io.grant := grant_oh
}
```

---

## 五、Switch Allocator (SA)

### 5.1 SA 的功能

**目的**：

```
為封包分配 Crossbar 的連接

問題：
  多個 Input Port 的封包想要使用同一個 Output Port
  → Crossbar 一次只能建立一個連接
  → 需要 Arbiter 決定誰獲勝
```

**輸入**：

```
- 所有 Input VC 的請求
- 每個請求的 Output Port

輸出：
- 每個 Input VC 是否獲得 Crossbar 連接
```

---

### 5.2 SA 的 Arbiter 架構

**Separable Allocator**：

```
兩階段 Arbiter:

階段 1：Input Arbiter
  - 每個 Input Port 有多個 VC
  - 選擇一個 VC 來競爭 Crossbar
  
階段 2：Output Arbiter
  - 每個 Output Port 有多個 Input Port 競爭
  - 選擇一個 Input Port 獲勝

範例：
  Input Port 0 有 VC0, VC1, VC2
  → Input Arbiter 選擇 VC1
  
  Output Port East 有 Input Port 0, 1, 2 競爭
  → Output Arbiter 選擇 Input Port 0
  
  → Input Port 0 的 VC1 獲得連接到 Output Port East
```

**圖示**：

```
Input Arbiter (每個 Input Port)
┌─────────────┐
│  VC0  VC1  VC2  │
│   │    │    │   │
│   └────┴────┘   │
│        │        │
│     Winner      │
└────────┬────────┘
         │
         ▼
Output Arbiter (每個 Output Port)
┌─────────────┐
│ In0 In1 In2 │
│  │   │   │  │
│  └───┴───┘  │
│      │      │
│   Winner    │
└─────────────┘
```

---

### 5.3 Wavefront Allocator（進階）

**概念**：

```
Wavefront Allocator:
  一種高效的 Allocator，可以在 1 cycle 內完成分配
  
原理：
  使用對角線掃描，避免衝突
  
範例（4×4 Crossbar）：
  Cycle 1:
    Input 0 → Output 0
    Input 1 → Output 1
    Input 2 → Output 2
    Input 3 → Output 3
  
  Cycle 2:
    Input 0 → Output 1
    Input 1 → Output 2
    Input 2 → Output 3
    Input 3 → Output 0
  
  ...
```

**優點**：

```
✅ 延遲低（1 cycle）
✅ 吞吐量高
✅ 公平性好
```

**缺點**：

```
❌ 硬體複雜度高
❌ 面積較大
```

---

## 六、Crossbar 的設計

### 6.1 Crossbar 的結構

**Full Crossbar**：

```
N×N Crossbar:
  N 個 Input Port
  N 個 Output Port
  任意 Input 可以連接到任意 Output

圖示（4×4 Crossbar）：
        Output
         0  1  2  3
       ┌──┬──┬──┬──┐
    0  │  │  │  │  │
Input ├──┼──┼──┼──┤
    1  │  │  │  │  │
     ├──┼──┼──┼──┤
    2  │  │  │  │  │
     ├──┼──┼──┼──┤
    3  │  │  │  │  │
       └──┴──┴──┴──┘

每個交叉點是一個 Mux
```

---

### 6.2 Crossbar 的實作

**Chisel 實作**：

```scala
class Crossbar(n: Int, width: Int) extends Module {
  val io = IO(new Bundle {
    val input = Input(Vec(n, UInt(width.W)))
    val select = Input(Vec(n, UInt(log2Ceil(n).W)))  // 每個 Output 選擇哪個 Input
    val output = Output(Vec(n, UInt(width.W)))
  })
  
  // 每個 Output Port 是一個 Mux
  for (i <- 0 until n) {
    io.output(i) := io.input(io.select(i))
  }
}
```

**硬體成本**：

```
N×N Crossbar:
  - N 個 N-to-1 Mux
  - 每個 Mux：N × width bits
  
範例（5×5 Crossbar, 128-bit Flit）：
  - 5 個 5-to-1 Mux
  - 每個 Mux：5 × 128 = 640 bits
  - 總面積：~0.05 mm² (7nm)
  
結論：
  Crossbar 佔 Router 面積的 ~15-20%
```

---

### 6.3 Crossbar 的優化

**Segmented Crossbar**：

```
概念：
  將 Crossbar 分成多個小的 Crossbar
  → 降低複雜度
  
範例（8×8 → 2 個 4×4）：
  Input 0-3 → Crossbar 0 → Output 0-3
  Input 4-7 → Crossbar 1 → Output 4-7
  
優點：
  ✅ 面積降低（O(N²) → O(N²/2)）
  ✅ 延遲降低（Mux 較小）

缺點：
  ❌ 靈活性降低（Input 0 無法連接到 Output 4）
```

---

## 七、Pipeline 優化技術

### 7.1 Speculative Switch Allocation

**問題**：

```
傳統 Pipeline:
  Cycle 1: RC
  Cycle 2: VA
  Cycle 3: SA
  Cycle 4: ST

  → 4 cycles Hop Latency
```

**優化：Speculative SA**

```
概念：
  在 VA 的同時，投機性地執行 SA
  如果 VA 成功，SA 的結果可以直接使用
  如果 VA 失敗，SA 的結果丟棄

Pipeline:
  Cycle 1: RC
  Cycle 2: VA + SA (並行)
  Cycle 3: ST

  → 3 cycles Hop Latency (降低 25%)
```

**風險**：

```
如果 VA 失敗：
  - SA 的結果無效
  - 需要重新執行 VA 和 SA
  - 浪費功耗

適用場景：
  - VA 成功率高（> 90%）
  - 延遲敏感的應用
```

---

### 7.2 Bypass Path

**概念**：

```
如果 Router 的 Buffer 是空的，且 Crossbar 可用
→ Flit 可以直接通過，不需要進入 Buffer
→ 降低延遲
```

**實作**：

```
正常路徑：
  Input Port → Buffer → RC → VA → SA → Crossbar → Output Port
  延遲：4 cycles

Bypass 路徑：
  Input Port → RC → Crossbar → Output Port
  延遲：1 cycle

條件：
  - Buffer 是空的
  - Crossbar 可用（沒有其他 Flit 使用）
  - 下游 Router 有空位
```

**性能提升**：

```
Zero-load Latency:
  正常：4 cycles
  Bypass：1 cycle

  → 降低 75%

實際場景（50% 負載）：
  平均延遲：~2.5 cycles

  → 降低 37.5%
```

---

### 7.3 Virtual Channel Regrouping

**概念**：

```
動態調整 VC 的分配
→ 提高 Buffer 利用率
```

**問題**：

```
固定 VC 分配：
  VC0: Request (8 Flits)
  VC1: Response (8 Flits)
  VC2: Snoop (8 Flits)

如果 Request 很多，Response 很少：
  VC0 塞滿（8/8）
  VC1 幾乎空（1/8）
  VC2 幾乎空（0/8）

  → VC1 和 VC2 的 Buffer 浪費
```

**解決方案：Regrouping**

```
允許 VC 共享 Buffer:
  Total Buffer: 24 Flits

  動態分配：
    Request: 18 Flits
    Response: 4 Flits
    Snoop: 2 Flits

  → 提高 Buffer 利用率
```

**限制**：

```
需要確保不會產生 Protocol Deadlock:
  - 每個 VC 至少保留最小 Buffer（如 2 Flits）
  - Response 的優先級高於 Request
```

---

## 八、真實案例：Intel 和 ARM 的 Router 設計

### 8.1 Intel Skylake-SP Mesh Router

**規格**：

```
拓撲：7×4 Mesh
Router 數量：28 個

每個 Router:
  - 5 個 Port (N, S, E, W, Local)
  - 3 個 Virtual Channels (Request, Snoop, Response)
  - Buffer：每個 VC 8 Flits
  - Flit 寬度：64 bytes
  - 頻率：2.5 GHz
```

**Pipeline**：

```
4-Stage Pipeline:
  RC: 1 cycle
  VA: 1 cycle
  SA: 1 cycle
  ST: 1 cycle

Hop Latency: 4 cycles = 1.6 ns @ 2.5 GHz
```

**優化**：

```
1. Speculative SA:
   - VA 和 SA 並行執行
   - Hop Latency 降低到 3 cycles

2. Adaptive Routing:
   - 根據擁塞情況動態選擇路徑
   - 提高吞吐量

3. Credit-Based Flow Control:
   - 精確控制 Buffer 使用
   - 避免溢出
```

**性能**：

```
延遲：
  平均 Hop Latency: ~2 cycles (包含 Bypass)
  平均距離：4.5 跳
  平均延遲：~9 cycles = 3.6 ns

頻寬：
  每條 Link：64 bytes/cycle @ 2.5 GHz = 160 GB/s
  Bisection Bandwidth：7 × 160 = 1,120 GB/s
```

---

### 8.2 ARM CMN-700 Cross Point (XP)

**規格**：

```
拓撲：最多 8×8 Mesh
XP (Cross Point) 數量：最多 64 個

每個 XP:
  - 4 個 Mesh Port (N, S, E, W)
  - 1-4 個 Device Port (CPU, GPU, Cache)
  - 2 個 Virtual Channels (Request, Response)
  - Buffer：每個 VC 16 Flits
  - Flit 寬度：256 bits (32 bytes)
  - 頻率：2.0 GHz
```

**Pipeline**：

```
3-Stage Pipeline:
  RC + VA: 1 cycle (合併)
  SA: 1 cycle
  ST: 1 cycle

Hop Latency: 3 cycles = 1.5 ns @ 2.0 GHz
```

**特殊設計**：

```
1. Heterogeneous Devices:
   - 可以連接不同類型的 Device（CPU, GPU, NPU）
   - 每個 Device 有不同的頻寬需求

2. QoS (Quality of Service):
   - 不同的 Device 有不同的優先級
   - 高優先級的 Device 優先獲得 Crossbar

3. Power Gating:
   - 閒置的 XP 可以關閉
   - 降低功耗
```

**性能**：

```
延遲：
  平均 Hop Latency: ~2 cycles (包含 Bypass)
  平均距離：~6 跳 (8×8 Mesh)
  平均延遲：~12 cycles = 6 ns

頻寬：
  每條 Link：32 bytes/cycle @ 2.0 GHz = 64 GB/s
  Bisection Bandwidth：8 × 64 = 512 GB/s
```

---

## 九、Router 的面積和功耗分析

### 9.1 面積分解

**5-Port Router (4 VCs, 8 Flits per VC, 128-bit Flit)**：

```
組件                面積 (mm², 7nm)    佔比
─────────────────────────────────────────
Input Buffer        0.10              33%
  (5 ports × 4 VCs × 8 Flits × 128 bits)

Crossbar            0.05              17%
  (5×5 Crossbar, 128-bit)

Route Compute       0.01               3%
  (簡單的比較器和 Mux)

VC Allocator        0.05              17%
  (20 個 Input VC → 20 個 Output VC)

Switch Allocator    0.06              20%
  (20 個 Input VC → 5 個 Output Port)

Control Logic       0.03              10%
  (Pipeline 控制, Credit 管理)

─────────────────────────────────────────
總計                0.30             100%
```

**結論**：

```
Buffer 是最大的組件（33%）
→ 優化 Buffer 可以顯著降低面積
```

> 註：以上 Router 面積分解為基於代表性 7nm 製程與 SRAM 編譯器的估計值，
> 用來說明各組件在 Router 面積中的占比，
> 實際數字會依製程節點、VC 配置與 Flit 寬度而異。

---

### 9.2 功耗分解

**5-Port Router @ 2.5 GHz, 50% 負載**：

```
組件                功耗 (mW)         佔比
─────────────────────────────────────────
Input Buffer        12               40%
  (SRAM 讀寫功耗)

Crossbar            6                20%
  (Mux 切換功耗)

Route Compute       1                 3%
  (組合邏輯功耗)

VC Allocator        4                13%
  (Arbiter 功耗)

Switch Allocator    5                17%
  (Arbiter 功耗)

Control Logic       2                 7%
  (Pipeline 暫存器功耗)

─────────────────────────────────────────
總計                30              100%
```

**結論**：

```
Buffer 是最大的功耗來源（40%）
→ 優化 Buffer 可以顯著降低功耗
```

> 註：以上 Router 功耗分解為基於代表性 7nm 製程與 2.5 GHz 頻率的估計值，
> 用來說明各組件在 Router 功耗中的占比，
> 實際數字會依製程節點、頻率、負載與電壓而異。

---

### 9.3 優化策略

**降低面積**：

```
1. 減少 Buffer 大小:
   - 從 8 Flits → 4 Flits
   - 面積降低 ~15%
   - 但吞吐量可能降低

2. 減少 VC 數量:
   - 從 4 VCs → 2 VCs
   - 面積降低 ~20%
   - 但可能產生 Protocol Deadlock

3. 簡化 Allocator:
   - 使用 Round-Robin 而非 Wavefront
   - 面積降低 ~10%
   - 但吞吐量可能降低
```

**降低功耗**：

```
1. Clock Gating:
   - 閒置的 Buffer 不切換時鐘
   - 功耗降低 ~20%

2. Power Gating:
   - 閒置的 Router 完全關閉
   - 功耗降低 ~50%（低負載時）

3. Voltage Scaling:
   - 根據負載動態調整電壓
   - 功耗降低 ~30%
```

---

## 十、Router 設計的常見陷阱

### 10.1 Pipeline Hazard

**問題場景**：

```
某 Router 設計在高負載下出現錯誤：
  某些 Flit 會消失

原因：
  VA 和 SA 之間的 Pipeline Hazard

場景：
  Cycle 1: Flit A 在 VA Stage，請求 Output VC0
  Cycle 2: Flit A 在 SA Stage，但 VC0 已被 Flit B 佔用
  → Flit A 的 VA 結果無效
  → 需要重新執行 VA

  如果設計沒有處理這種情況
  → Flit A 可能被丟棄
```

**解決方案**：

```
增加 Retry 機制：
  if (VA 成功 && SA 失敗):
      重新執行 VA
      Flit 留在 Buffer 中

  → Flit 不會被丟棄
```

---

### 10.2 Credit Overflow

**問題場景**：

```
某 Router 在長時間運行後出現死鎖：
  Credit Counter 溢出

原因：
  Credit Return 訊號在傳輸過程中遺失
  → Sender 的 Credit Counter 一直減少
  → 最終變成負數（溢出）
  → Sender 停止發送
```

**解決方案**：

```
增加 Credit Refresh 機制：
  定期（例如每 1000 cycles），Receiver 發送完整的 Credit 狀態
  → Sender 重新同步 Credit Counter

  → 即使 Credit Return 遺失，也能恢復
```

---

### 10.3 Arbiter Starvation

**問題場景**：

```
某 Router 在特定場景下性能極差：
  某些 VC 的延遲非常高（> 100 cycles）

原因：
  使用 Priority Arbiter
  → 低優先級的 VC 被餓死

場景：
  VC0 (高優先級) 一直有封包
  → VC1 (低優先級) 永遠無法獲得 Crossbar
```

**解決方案**：

```
方案 1：改用 Round-Robin Arbiter
  每個 VC 輪流獲得機會
  → 公平性提高

方案 2：使用 Aging 機制
  等待時間越長，優先級越高
  → 避免餓死
```

---

## 十一、總結

Router 微架構是 NoC 的核心：

1. **基本架構**：
   - Input Buffer, RC, VA, SA, Crossbar
   - 4-Stage Pipeline (RC, VA, SA, ST)
   - Hop Latency: 4 cycles

2. **關鍵組件**：
   - Input Buffer：FIFO, 4-8 Flits per VC
   - Route Compute：XY Routing, 1 cycle
   - VC Allocator：Round-Robin Arbiter
   - Switch Allocator：Separable Allocator
   - Crossbar：N×N Mux

3. **優化技術**：
   - Speculative SA：降低延遲 25%
   - Bypass Path：降低 Zero-load Latency 75%
   - VC Regrouping：提高 Buffer 利用率

4. **真實案例**：
   - Intel Mesh：4-Stage Pipeline, 3 VCs, Speculative SA
   - ARM CMN-700：3-Stage Pipeline, QoS, Power Gating

5. **面積和功耗**：
   - Buffer 佔 33% 面積, 40% 功耗
   - 優化策略：Clock Gating, Power Gating, Voltage Scaling

6. **設計經驗**：
   - Pipeline Hazard：需要 Retry 機制
   - Credit Overflow：需要 Refresh 機制
   - Arbiter Starvation：使用 Round-Robin 或 Aging

理解 Router 微架構，可以幫助我們：
- 設計高效的 NoC
- 優化延遲和吞吐量
- 除錯複雜的硬體問題

---

## 下一篇預告

在下一篇文章中，我們會深入探討：

**NoC 與 Cache Coherency 整合**
- MESI 協議在 NoC 上的實作
- Directory-Based Coherency
- Snoop Filter 的設計
- Coherency 訊息的路由
- 如何避免 Protocol Deadlock

敬請期待！

---

## 參考資料

**公開文檔**：
- *Principles and Practices of Interconnection Networks* - Dally & Towles
- *Microarchitecture of Network-on-Chip Routers* - Jerger & Peh (2009)
- Intel Xeon Scalable Processor Architecture Specification
- ARM CoreLink CMN-700 Technical Reference Manual
- *A Survey of Techniques for Architecting and Managing Asymmetric Multicore Processors* - Kumar et al. (2013)

**個人筆記**：
- Network-on-Chip Technical Notes (基於公開領域知識整理)


---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: https://github.com/djiangtw/tech-column-public
