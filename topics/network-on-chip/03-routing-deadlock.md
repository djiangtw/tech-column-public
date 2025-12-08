# NoC 路由演算法與死鎖避免：從理論到實作

**作者**: Danny Jiang  
**日期**: 2025-12-07  

---

## 前言：NoC 死鎖的挑戰

在多核心 SoC 的設計中，NoC 死鎖是最難除錯的問題之一。

業界曾有這樣的案例：一個 16 核心的 SoC 在壓力測試下，**偶爾會完全卡死**。

所有核心都停止回應，沒有任何錯誤訊息，就像時間靜止了一樣。

工程師花了數天時間，用 Waveform Viewer 一個 cycle 一個 cycle 地追蹤，終於發現：

**4 個封包在 NoC 中形成了一個環，互相等待對方釋放 Buffer，導致死鎖。**

這個案例說明：**路由演算法的設計，不只是找到最短路徑，更重要的是避免死鎖。**

---

## 一、路由演算法的基礎

### 1.1 什麼是路由 (Routing)？

**定義**：

```
路由 = 決定封包從來源到目的地的路徑

在 NoC 中：
  每個 Router 收到封包後，需要決定：
  "這個封包應該往哪個 Port 轉送？"
```

**路由的兩種方式**：

```
1. Source Routing (來源路由):
   - 來源節點計算完整路徑
   - 路徑資訊編碼在封包 Header 中
   - Router 只需要讀取 Header，不需要計算
   
   優點：Router 簡單
   缺點：Header 較大，無法動態調整

2. Distributed Routing (分散式路由):
   - 每個 Router 獨立計算下一跳
   - 封包 Header 只包含目的地地址
   - Router 需要路由邏輯
   
   優點：Header 小，可以動態調整
   缺點：Router 較複雜
```

**NoC 通常使用 Distributed Routing**，因為：
- Header 小，節省頻寬
- 可以根據擁塞情況動態調整路徑

---

### 1.2 路由演算法的分類

**1. Deterministic Routing (確定性路由)**

```
定義：
  給定來源和目的地，路徑唯一且固定

範例：XY Routing
  從 (0, 0) 到 (3, 2)
  路徑：(0,0) → (1,0) → (2,0) → (3,0) → (3,1) → (3,2)
  
優點：
  ✅ 實作簡單
  ✅ 容易證明無死鎖
  ✅ 延遲可預測

缺點：
  ❌ 無法避開擁塞
  ❌ 負載不均衡
```

---

**2. Adaptive Routing (自適應路由)**

```
定義：
  可以根據網路狀態動態選擇路徑

範例：West-First Routing
  從 (0, 0) 到 (3, 2)
  可能路徑 1：(0,0) → (1,0) → (2,0) → (3,0) → (3,1) → (3,2)
  可能路徑 2：(0,0) → (0,1) → (0,2) → (1,2) → (2,2) → (3,2)
  
  根據擁塞情況選擇
  
優點：
  ✅ 可以避開擁塞
  ✅ 負載均衡
  ✅ 吞吐量較高

缺點：
  ❌ 實作複雜
  ❌ 需要擁塞偵測邏輯
  ❌ 延遲不可預測
  ❌ 容易產生死鎖（需要特殊設計）
```

---

## 二、XY Routing：最經典的路由演算法

### 2.1 XY Routing 的原理

**演算法**：

```
XY Routing (Dimension-Order Routing):

1. 先沿著 X 軸移動，直到 X 座標相符
2. 再沿著 Y 軸移動，直到 Y 座標相符

虛擬碼：
  if (current_x < dest_x):
      output_port = East
  elif (current_x > dest_x):
      output_port = West
  elif (current_y < dest_y):
      output_port = North
  elif (current_y > dest_y):
      output_port = South
  else:
      output_port = Local  # 已到達目的地
```

**範例**：

```
從 (0, 0) 到 (3, 2) 的路徑：

[0,0] →E [1,0] →E [2,0] →E [3,0]
                              ↓N
                            [3,1]
                              ↓N
                            [3,2] ✓

步驟：
  1. X: 0 → 1 → 2 → 3 (往東 3 步)
  2. Y: 0 → 1 → 2 (往北 2 步)
  
總共 5 跳
```

---

### 2.2 XY Routing 的硬體實作

**Chisel 實作範例**：

```scala
class XYRouter(val x: Int, val y: Int) extends Module {
  val io = IO(new Bundle {
    val packet_in = Input(new Packet)
    val output_port = Output(UInt(3.W))  // 0=Local, 1=N, 2=S, 3=E, 4=W
  })
  
  val dest_x = io.packet_in.header.dest_x
  val dest_y = io.packet_in.header.dest_y
  
  // Route Compute Logic
  when (x.U < dest_x) {
    io.output_port := 3.U  // East
  } .elsewhen (x.U > dest_x) {
    io.output_port := 4.U  // West
  } .elsewhen (y.U < dest_y) {
    io.output_port := 1.U  // North
  } .elsewhen (y.U > dest_y) {
    io.output_port := 2.U  // South
  } .otherwise {
    io.output_port := 0.U  // Local
  }
}
```

**硬體成本**：

```
邏輯：
  - 2 個比較器 (x 和 y)
  - 1 個 5-to-1 Mux
  
延遲：
  - 比較器：~0.5 ns (7nm)
  - Mux：~0.2 ns
  - 總延遲：~0.7 ns (< 1 cycle @ 2.5 GHz)
  
面積：
  - ~100 個邏輯閘
  - 可以忽略不計（相比 Router 的 Buffer 和 Crossbar）
```

---

### 2.3 為什麼 XY Routing 不會死鎖？

**死鎖的必要條件**（Coffman Conditions）：

```
1. Mutual Exclusion (互斥)：資源一次只能被一個 Process 使用
2. Hold and Wait (持有並等待)：Process 持有資源，同時等待其他資源
3. No Preemption (不可搶占)：資源不能被強制釋放
4. Circular Wait (循環等待)：存在一個 Process 的循環等待鏈

在 NoC 中：
  資源 = Buffer
  Process = Packet
  
  前 3 個條件通常無法避免
  → 關鍵是避免 Circular Wait (循環等待)
```

**XY Routing 的無環證明**：

```
定義 Channel Dependency Graph (CDG):
  - Vertex = Channel (Router 之間的連結)
  - Edge = 封包可能從 Channel A 轉到 Channel B
  
XY Routing 的 CDG:
  East → East  (X 軸方向可以繼續往東)
  East → North (X 軸完成後，可以往北)
  East → South (X 軸完成後，可以往南)
  West → West  (X 軸方向可以繼續往西)
  West → North (X 軸完成後，可以往北)
  West → South (X 軸完成後，可以往南)
  North → North (Y 軸方向可以繼續往北)
  South → South (Y 軸方向可以繼續往南)
  
關鍵觀察：
  - 沒有 North → East/West
  - 沒有 South → East/West
  
  → 一旦進入 Y 軸方向，就不會再回到 X 軸方向
  → CDG 是 DAG (Directed Acyclic Graph)
  → 無環 → 無死鎖
```

**圖示**：

```
CDG (Channel Dependency Graph):

East → East → North → North
  ↓      ↓      ↑
  ↓      ↓      ↑
  ↓      ↓    South → South
  ↓      ↓
West → West

沒有環！
```

---

## 三、死鎖的類型與解決方案

### 3.1 Routing Deadlock (路由死鎖)

**定義**：

```
封包在路由過程中形成循環等待

範例（如果沒有 XY Routing）：
  Packet A: (0,0) → (2,0) → (2,2)
  Packet B: (2,2) → (2,0) → (0,0)
  Packet C: (0,2) → (0,0) → (2,0)
  Packet D: (2,0) → (0,0) → (0,2)
  
  如果 A, B, C, D 同時在 (0,0), (2,0), (0,2), (2,2)
  → 形成環 → 死鎖
```

**解決方案 1：XY Routing**

```
使用 Dimension-Order Routing
→ CDG 無環
→ 無死鎖
```

**解決方案 2：Turn Model**

```
Turn Model:
  限制某些轉彎方向，破壞環

West-First Routing:
  - 如果需要往西，必須先往西
  - 往西之後，可以往任何方向
  - 其他方向之後，不能再往西
  
  禁止的轉彎：
    North → West
    South → West
    East → West
  
  → 破壞環 → 無死鎖
```

---

### 3.2 Protocol Deadlock (協議死鎖)

**定義**：

```
不同類型的訊息之間形成循環依賴

範例（MESI 協議）：
  Core 0 發出 Read Request → Cache Slice 5
  Cache Slice 5 需要發出 Invalidate → All Cores
  但 NoC 的 Buffer 已滿，Invalidate 無法送出
  → Core 0 一直等待 Data Response
  → Cache Slice 5 一直等待 Buffer 空位
  → 死鎖
```

**為什麼會發生？**

```
Request 和 Response 有依賴關係：
  Request → Response (Response 依賴 Request)
  
如果 Request 和 Response 共用同一個 Buffer：
  Request 塞滿 Buffer
  → Response 無法送出
  → Request 無法完成
  → 死鎖
```

**解決方案：Virtual Channels (虛擬通道)**

```
將 Buffer 分成多個獨立的 Virtual Channels：

VC0: Request Channel
VC1: Response Channel
VC2: Snoop Channel (Invalidate, Forward)

規則：
  - Request 只能使用 VC0
  - Response 只能使用 VC1
  - Snoop 只能使用 VC2
  
即使 VC0 (Request) 塞滿了，VC1 (Response) 依然暢通
→ Response 可以送出
→ Request 可以完成
→ 無死鎖
```

---

## 四、Virtual Channels：NoC 的殺手級功能

### 4.1 Virtual Channels 的原理

**概念**：

```
Physical Channel (實體通道):
  - 實體的連線（導線）
  - 一次只能傳輸一個 Flit (Flow Control Unit)

Virtual Channel (虛擬通道):
  - 邏輯上的通道
  - 共享同一條實體連線
  - 每個 VC 有自己的 Buffer
```

**圖示**：

```
Physical Channel:
  Router A ────────────────→ Router B
            (64-bit 導線)

Virtual Channels:
  Router A                   Router B
  ┌──────┐                   ┌──────┐
  │ VC0  │ ─────────────────→│ VC0  │
  │Buffer│                   │Buffer│
  ├──────┤                   ├──────┤
  │ VC1  │ ─────────────────→│ VC1  │
  │Buffer│                   │Buffer│
  ├──────┤                   ├──────┤
  │ VC2  │ ─────────────────→│ VC2  │
  │Buffer│                   │Buffer│
  └──────┘                   └──────┘
     ↓                          ↑
     └──────────────────────────┘
        共享同一條實體連線
```

---

### 4.2 Virtual Channels 的優勢

**優勢 1：避免 Protocol Deadlock**

```
不同類型的訊息使用不同的 VC
→ 即使某個 VC 塞滿，其他 VC 依然暢通
→ 避免死鎖
```

**優勢 2：提高頻寬利用率**

```
沒有 VC:
  如果一個封包被 Block（等待下游 Buffer）
  → 整條 Physical Channel 閒置
  → 浪費頻寬

有 VC:
  如果 VC0 的封包被 Block
  → VC1, VC2 的封包依然可以傳輸
  → 提高頻寬利用率
```

**優勢 3：減少 Head-of-Line Blocking**

```
Head-of-Line Blocking:
  Queue 前面的封包被 Block
  → 後面的封包也被 Block
  → 即使後面的封包可以走不同路徑

有 VC:
  不同路徑的封包使用不同的 VC
  → 減少 Head-of-Line Blocking
```

---

### 4.3 Virtual Channels 的硬體成本

**Buffer 成本**：

```
沒有 VC:
  每個 Input Port：1 個 Buffer (例如 8 Flits)
  
有 4 個 VC:
  每個 Input Port：4 個 Buffer (每個 8 Flits)
  
Buffer 面積增加 4 倍
```

**Arbiter 成本**：

```
沒有 VC:
  每個 Output Port：1 個 Arbiter (5 個 Input Port)
  
有 4 個 VC:
  每個 Output Port：1 個 Arbiter (5 × 4 = 20 個 Input VC)
  
Arbiter 複雜度增加 4 倍
```

**總成本**：

```
Router 面積：
  沒有 VC：~0.3 mm² (7nm)
  有 4 個 VC：~0.6 mm² (7nm)
  
功耗：
  沒有 VC：~30 mW
  有 4 個 VC：~50 mW
  
結論：
  VC 增加 ~2 倍的面積和功耗
  但可以避免死鎖，提高性能
  → 值得
```

---

## 五、流量控制 (Flow Control)

### 5.1 什麼是流量控制？

**定義**：

```
流量控制 = 管理封包的傳輸速率，避免 Buffer 溢出

問題：
  如果 Sender 發送速度 > Receiver 接收速度
  → Receiver 的 Buffer 會溢出
  → 封包遺失
  
解決方案：
  Receiver 告訴 Sender："我的 Buffer 還有多少空位"
  Sender 根據這個資訊調整發送速率
```

---

### 5.2 Credit-Based Flow Control

**原理**：

```
Credit = Buffer 的空位數量

初始狀態：
  Receiver 有 8 個 Buffer 空位
  → Sender 有 8 個 Credit

Sender 發送 1 個 Flit:
  Credit -= 1
  → Sender 剩下 7 個 Credit

Receiver 處理 1 個 Flit (釋放 Buffer):
  發送 Credit Return 給 Sender
  → Sender 的 Credit += 1
```

**硬體實作**：

```
Sender:
  credit_counter = 8  // 初始 Credit
  
  when (send_flit && credit_counter > 0):
      credit_counter -= 1
      send_flit_to_receiver()
  
  when (receive_credit_return):
      credit_counter += 1

Receiver:
  buffer_free_slots = 8  // 初始空位
  
  when (receive_flit):
      buffer_free_slots -= 1
      store_flit_in_buffer()
  
  when (process_flit):
      buffer_free_slots += 1
      send_credit_return_to_sender()
```

---

### 5.3 On/Off Flow Control

**原理**：

```
Receiver 發送 On/Off 訊號給 Sender

On:  "我的 Buffer 還有空位，可以繼續發送"
Off: "我的 Buffer 快滿了，請停止發送"

閾值：
  Buffer 使用率 > 75% → 發送 Off
  Buffer 使用率 < 25% → 發送 On
```

**優點**：

```
✅ 實作簡單（只需要 1-bit 訊號）
✅ 延遲低（不需要計數器）
```

**缺點**：

```
❌ 精確度低（無法知道確切的空位數量）
❌ 可能浪費頻寬（提前停止發送）
```

---

### 5.4 Wormhole Routing

**概念**：

```
Wormhole Routing:
  封包被切成多個 Flit (Flow Control Unit)
  Flit 像蟲一樣，一個接一個地穿過網路
  
Flit 類型：
  - Head Flit: 包含路由資訊
  - Body Flit: 包含資料
  - Tail Flit: 標記封包結束
```

**優勢**：

```
✅ Buffer 需求小（只需要存幾個 Flit，不需要存整個封包）
✅ 延遲低（Head Flit 可以先走，不需要等整個封包到達）
```

**問題**：

```
❌ 如果 Head Flit 被 Block，整個封包都被 Block
   → 佔用多個 Router 的 Buffer
   → 降低頻寬利用率
```

**解決方案：Virtual Cut-Through**

```
Virtual Cut-Through:
  - 如果下游有足夠的 Buffer，使用 Wormhole Routing
  - 如果下游 Buffer 不足，將整個封包存在當前 Router
  
  → 結合 Wormhole 的低延遲和 Store-and-Forward 的高頻寬利用率
```

---

## 六、真實案例：Intel Mesh 的路由與死鎖避免

### 6.1 Intel Skylake-SP 的路由設計

**拓撲**：

```
Intel Skylake-SP (28 核心):
  7×4 Mesh

每個 Router 連接：
  - 1 個 CPU Core
  - 1 個 Cache Slice (LLC)
  - 4 個鄰居 Router (N, S, E, W)
```

**路由演算法**：

```
Intel 使用 Dimension-Order Routing (類似 XY Routing)

但有一個特殊設計：
  - 支援 Adaptive Routing（在某些情況下）
  - 可以根據擁塞情況選擇不同路徑

如何避免死鎖？
  - 使用 Virtual Channels
  - 不同類型的訊息使用不同的 VC
```

---

### 6.2 Intel Mesh 的 Virtual Channels

**VC 配置**：

```
Intel Mesh 使用 3 個 Virtual Channels:

VC0: Request (Read Request, Write Request)
VC1: Snoop (Invalidate, Forward)
VC2: Response (Data Response, Acknowledgement)

為什麼需要 3 個 VC？
  - Request 和 Response 有依賴關係
  - Snoop 可能在 Request 處理過程中產生
  - 需要避免 Protocol Deadlock
```

**依賴關係**：

```
Request → Snoop → Response

範例：
  1. Core 0 發出 Read Request (VC0)
  2. Cache Slice 5 發出 Invalidate 給其他 Core (VC1)
  3. 其他 Core 回應 Acknowledgement (VC2)
  4. Cache Slice 5 發出 Data Response 給 Core 0 (VC2)

如果只有 1 個 VC：
  Request 塞滿 Buffer
  → Snoop 無法送出
  → Response 無法產生
  → 死鎖

有 3 個 VC：
  即使 VC0 塞滿，VC1 和 VC2 依然暢通
  → 無死鎖
```

---

### 6.3 Intel Mesh 的性能數據

**延遲**：

```
Zero-load Latency (28 核心, 7×4 Mesh):
  平均距離：~4.5 跳
  Hop Latency：~2 cycles
  平均延遲：~9 cycles @ 2.5 GHz = ~3.6 ns

比較 Ring (28 核心):
  平均距離：~7 跳
  平均延遲：~14 cycles = ~5.6 ns

Mesh 延遲降低 36%
```

**頻寬**：

```
每條 Link：64 bytes/cycle @ 2.5 GHz = 160 GB/s

Bisection Bandwidth (7×4 Mesh):
  垂直切：7 條 Link = 7 × 160 = 1,120 GB/s
  水平切：4 條 Link = 4 × 160 = 640 GB/s

比較 Ring (28 核心):
  Bisection Bandwidth = 2 × 160 = 320 GB/s

Mesh 頻寬提升 3.5 倍
```

> 註：以上 Zero-load Latency、Bisection Bandwidth 等數字為基於 Intel 公開文件的代表性估計值，
> 用來說明 Mesh 相對 Ring 的性能優勢，
> 實際數字會依 Skylake-SP 具體型號與工作負載而異。

---

## 七、進階主題：Turn Model 與 Adaptive Routing

### 7.1 Turn Model 的原理

**概念**：

```
Turn Model:
  限制某些轉彎方向，破壞 CDG 中的環

8 種可能的轉彎：
  North → East (NE)
  North → West (NW)
  South → East (SE)
  South → West (SW)
  East → North (EN)
  East → South (ES)
  West → North (WN)
  West → South (WS)
```

**West-First Routing**：

```
規則：
  如果需要往西，必須先往西
  往西之後，可以往任何方向
  其他方向之後，不能再往西

禁止的轉彎：
  NW (North → West)
  SW (South → West)
  EN → W (East → North → West)
  ES → W (East → South → West)

允許的轉彎：
  NE, SE, EN, ES, WN, WS

CDG 無環 → 無死鎖
```

**North-Last Routing**：

```
規則：
  往北必須是最後一步
  往北之後，不能再轉彎

禁止的轉彎：
  NE (North → East)
  NW (North → West)

允許的轉彎：
  SE, SW, EN, ES, WN, WS

CDG 無環 → 無死鎖
```

---

### 7.2 Adaptive Routing 的實作

**Minimal Adaptive Routing**：

```
規則：
  只選擇最短路徑中的一條
  根據擁塞情況動態選擇

範例（West-First）：
  從 (0, 0) 到 (3, 2)

  路徑 1：先往東，再往北
    (0,0) → (1,0) → (2,0) → (3,0) → (3,1) → (3,2)

  路徑 2：先往北，再往東
    (0,0) → (0,1) → (0,2) → (1,2) → (2,2) → (3,2)

  在每個 Router，檢查兩個方向的擁塞情況：
    if (east_congestion < north_congestion):
        output_port = East
    else:
        output_port = North
```

**硬體實作**：

```scala
class AdaptiveRouter(val x: Int, val y: Int) extends Module {
  val io = IO(new Bundle {
    val packet_in = Input(new Packet)
    val east_credit = Input(UInt(4.W))   // 東方的 Credit
    val north_credit = Input(UInt(4.W))  // 北方的 Credit
    val output_port = Output(UInt(3.W))
  })

  val dest_x = io.packet_in.header.dest_x
  val dest_y = io.packet_in.header.dest_y

  val need_east = x.U < dest_x
  val need_north = y.U < dest_y

  when (need_east && need_north) {
    // 兩個方向都可以，選擇 Credit 較多的
    when (io.east_credit > io.north_credit) {
      io.output_port := 3.U  // East
    } .otherwise {
      io.output_port := 1.U  // North
    }
  } .elsewhen (need_east) {
    io.output_port := 3.U  // East
  } .elsewhen (need_north) {
    io.output_port := 1.U  // North
  } .otherwise {
    io.output_port := 0.U  // Local
  }
}
```

---

### 7.3 Adaptive Routing 的性能提升

**模擬結果**（16 核心 4×4 Mesh）：

```
Uniform Random Traffic:
  XY Routing:
    平均延遲：12 cycles
    飽和吞吐量：0.45 flits/cycle/node

  West-First Adaptive Routing:
    平均延遲：10 cycles (17% 改善)
    飽和吞吐量：0.52 flits/cycle/node (16% 改善)

Hotspot Traffic (所有節點都存取 (2,2)):
  XY Routing:
    平均延遲：25 cycles
    飽和吞吐量：0.25 flits/cycle/node

  West-First Adaptive Routing:
    平均延遲：18 cycles (28% 改善)
    飽和吞吐量：0.35 flits/cycle/node (40% 改善)
```

> 註：以上 Adaptive Routing 模擬結果為基於代表性 NoC 模擬器的估計值，
> 用來說明 Adaptive Routing 相對 Deterministic Routing 的性能優勢，
> 實際數字會依流量模式、Router 微架構與 VC 配置而異。

**結論**：

```
Adaptive Routing 在 Hotspot Traffic 下效果顯著
→ 適合 Cache Coherency（某些 Cache Slice 可能成為熱點）
```

---

## 八、NoC 死鎖的除錯經驗

### 8.1 死鎖的症狀

```
1. 系統完全卡死
   - 所有核心停止回應
   - 沒有任何錯誤訊息

2. 偶發性
   - 只在高負載下出現
   - 難以重現

3. Waveform 特徵
   - 多個封包在 Router 的 Buffer 中
   - Credit Counter = 0（所有 Credit 都用完）
   - 沒有任何封包在移動
```

---

### 8.2 除錯步驟

**步驟 1：確認是死鎖，而非 Livelock**

```
Deadlock (死鎖):
  封包完全停止移動

Livelock (活鎖):
  封包一直在移動，但永遠到不了目的地

檢查方法：
  在 Waveform 中，觀察封包的位置
  如果位置不變 → Deadlock
  如果位置一直變 → Livelock
```

**步驟 2：找出循環等待鏈**

```
在 Waveform 中，追蹤每個封包：
  Packet A 在 Router 0，等待 Router 1 的 Buffer
  Packet B 在 Router 1，等待 Router 2 的 Buffer
  Packet C 在 Router 2，等待 Router 3 的 Buffer
  Packet D 在 Router 3，等待 Router 0 的 Buffer

  → 形成環：R0 → R1 → R2 → R3 → R0
```

**步驟 3：分析死鎖類型**

```
Routing Deadlock:
  封包的路由路徑形成環

  解決方案：
    - 檢查路由演算法是否正確實作
    - 確認 CDG 無環

Protocol Deadlock:
  不同類型的訊息形成依賴環

  解決方案：
    - 檢查 Virtual Channels 配置
    - 確認不同類型的訊息使用不同的 VC
```

**步驟 4：修復並驗證**

```
修復後，需要驗證：
  1. 功能正確性：跑所有測試案例
  2. 壓力測試：長時間高負載測試（24 小時+）
  3. 形式化驗證：使用工具證明無死鎖（如果可能）
```

---

### 8.3 業界案例：Virtual Channel 配置錯誤

**問題場景**：

```
某 16 核心 SoC，使用 4×4 Mesh
路由演算法：XY Routing
Virtual Channels：2 個 (VC0=Request, VC1=Response)

死鎖場景：
  Core 0 發出 Read Request → Cache Slice 5 (VC0)
  Cache Slice 5 需要發出 Invalidate → All Cores (應該用獨立 VC)
  但 Invalidate 被錯誤地放在 VC0
  → VC0 塞滿 Request 和 Invalidate
  → Response 無法送出
  → 死鎖
```

**根本原因**：

```
Invalidate 訊息被錯誤地分類為 Request
→ 使用了 VC0
→ 應該使用獨立的 VC2 (Snoop Channel)
```

**修復方案**：

```
增加第 3 個 Virtual Channel:
  VC0: Request
  VC1: Response
  VC2: Snoop (Invalidate, Forward)

修改 Coherency Protocol 的訊息分類邏輯
→ Invalidate 使用 VC2
→ 避免死鎖
```

**驗證方法**：

```
壓力測試：
  - 16 個核心同時執行多執行緒程式
  - 大量的 Cache Coherency 流量
  - 長時間運行測試（72+ 小時）
  - 確認無死鎖

結論：
  3 個 Virtual Channels 是多核心 SoC 的標準配置
```

---

## 九、總結

路由演算法與死鎖避免是 NoC 設計的核心：

1. **路由演算法**：
   - Deterministic Routing (XY Routing)：簡單、無死鎖
   - Adaptive Routing (Turn Model)：高性能、需要特殊設計

2. **死鎖類型**：
   - Routing Deadlock：路由路徑形成環
   - Protocol Deadlock：訊息依賴形成環

3. **解決方案**：
   - XY Routing：CDG 無環
   - Virtual Channels：分離不同類型的訊息
   - Turn Model：限制轉彎方向

4. **流量控制**：
   - Credit-Based Flow Control：精確、常用
   - Wormhole Routing：低延遲、低 Buffer 需求

5. **真實案例**：
   - Intel Mesh：3 個 VC，Dimension-Order Routing
   - 性能提升：延遲降低 36%，頻寬提升 3.5 倍

6. **除錯經驗**：
   - 確認死鎖類型
   - 找出循環等待鏈
   - 修復並驗證

理解這些概念，可以幫助我們：
- 設計無死鎖的 NoC
- 除錯複雜的死鎖問題
- 優化 NoC 性能

---

## 下一篇預告

在下一篇文章中，我們會深入探討：

**Router 微架構設計**
- Router Pipeline (RC, VA, SA, ST)
- Input Buffer 的設計
- Crossbar 的實作
- Arbiter 的演算法
- 如何優化 Router 的延遲和面積

敬請期待！

---

## 參考資料

**公開文檔**：
- *Principles and Practices of Interconnection Networks* - Dally & Towles
- *Deadlock-Free Message Routing in Multiprocessor Interconnection Networks* - Dally & Seitz (1987)
- *A Survey of Wormhole Routing Techniques in Direct Networks* - Ni & McKinley (1993)
- Intel Xeon Scalable Processor Architecture Specification
- ARM CoreLink CMN-700 Technical Reference Manual

**個人筆記**：
- Network-on-Chip Technical Notes (基於公開領域知識整理)


---

## 版權聲明

本文採用 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 授權。

**出處**: https://github.com/djiangtw/tech-column-public
