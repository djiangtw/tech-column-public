# Heterogeneous System Architecture: Design & Performance

**Author**: Danny Jiang
**Date**: 2026-02-10

---

## Introduction: When IPC Optimization Hits Physical Limits

In the previous article, *All Roads Lead to IPC*, we established a core insight: **every CPU microarchitecture feature exists to maximize one metric—IPC**. Superscalar execution, out-of-order processing, branch prediction, caching, reorder buffers—these aren't independent features. They are compensatory mechanisms invented to rescue IPC from the harsh realities of memory latency, data dependencies, and control hazards.

But that story isn't complete.

During an AI accelerator project, our team faced a brutal reality: even with a 512-entry ROB, 99% branch prediction accuracy, and 64MB of L3 cache, IPC remained abysmal for deep learning workloads.

What went wrong?

After weeks of analysis, we discovered two insurmountable walls:

**The Memory Wall**: Neural network weights span gigabytes. No L3 cache, however large, can contain them. Every cache miss incurs 200+ cycles of DRAM latency—a gap that even the largest ROB cannot hide.

**The Power Wall**: Maintaining high IPC requires complex out-of-order logic, massive ROBs, and sophisticated branch predictors. These mechanisms consume enormous power. When we pushed clock frequencies beyond 4GHz, chip power exceeded thermal limits.

This experience taught me something profound: **maximizing single-processor IPC can no longer meet modern computational demands**.

The industry's solution? Stop chasing "one supremely powerful core" and embrace "collaboration among specialized cores"—this is the essence of **heterogeneous computing**.

This article extends the architect's perspective from the previous piece, expanding from single-CPU IPC optimization to heterogeneous system design spanning CPU + GPU + NPU + DPU. We'll use **six performance laws** as our analytical framework to understand the mathematics behind these design decisions.

---

## Six Performance Laws: The System Thinker's Compass

Before diving into heterogeneous architectures, we need to establish our analytical toolkit. These six laws serve as a compass for system thinkers, helping us navigate complex design spaces.

### 1. Amdahl's Law: Serial Portions Determine the Speedup Ceiling

In 1967, Gene Amdahl presented a two-page paper at the AFIPS conference that forever changed our understanding of parallel computing.

**The Formula**:

$$
S = \frac{1}{(1-p) + \frac{p}{N}}
$$

Where $p$ is the parallelizable fraction and $N$ is the number of processors.

**Intuition**: If 10% of your program is serial (cannot be parallelized), then no matter how many GPU cores you add, speedup will never exceed 10×.

**Real-World Data**: Consider deep learning training. Even when matrix operations parallelize perfectly, these operations remain serial:
- **Gradient synchronization**: AllReduce operations must wait for all GPUs to complete
- **Data loading**: Reading training data from SSDs is often a single-threaded bottleneck
- **Model updates**: Optimizer parameter update steps

In practice, even training ResNet-50 on 8× A100 GPUs, serial portions (data loading, gradient sync) still consume 15-20% of total time. This means theoretical speedup caps at 5-6.7×, not 8×.

**Implications for Heterogeneous Systems**: This explains why we can't rely solely on GPUs. Even with thousands of GPU cores, serial logic (control flow, data dependency handling) still requires CPUs. **The CPU is the guardian of Amdahl's Law**—it handles what cannot be parallelized.

**Design Pitfall**: Many assume "more GPUs = linear speedup." This misunderstands Amdahl's Law. True optimization means **reducing the serial fraction**, not blindly adding parallel resources.

> **Citation**: Amdahl, G. M. (1967). "Validity of the Single Processor Approach to Achieving Large Scale Computing Capabilities." *AFIPS Conference Proceedings*, 30, 483-485.

### 2. Gustafson's Law: Scaling Problem Size Breaks the Limit

In 1988, John Gustafson observed something interesting at Sandia National Laboratories: when scientists gained access to more processors, they didn't solve the same problems faster—they chose to solve *bigger* problems. This observation led to a reinterpretation of Amdahl's Law.

**The Formula**:

$$
S = s + p \times N
$$

Where $s$ is the serial time fraction, $p$ is the parallel time fraction ($s + p = 1$), and $N$ is the number of processors.

**Intuition**: Amdahl assumed fixed problem size. Gustafson asked: "If we have more cores, why not solve larger problems?"

**Real-World Data**: AI model evolution perfectly demonstrates this law:

| Model | Year | Parameters | Training Compute (PF-days) |
|-------|------|------------|---------------------------|
| GPT-2 | 2019 | 1.5B | ~40 |
| GPT-3 | 2020 | 175B | ~3,640 |
| GPT-4 | 2023 | ~1.8T | ~21,000+ |
| Llama 3.1 405B | 2024 | 405B | ~30,000+ |

Problem size grew 1000×, but the parallel fraction also increased (matrix operations rose from 85% to 95%+). This is Gustafson's Law in action.

**Implications for Heterogeneous Systems**: This is the theoretical foundation for GPUs and NPUs. **The GPU embodies Gustafson's Law**—it enables us to tackle previously impossible problems.

**Design Insight**: The true value of heterogeneous systems isn't "making the same tasks faster"—it's "enabling tasks that were previously impossible."

> **Citation**: Gustafson, J. L. (1988). "Reevaluating Amdahl's Law." *Communications of the ACM*, 31(5), 532-533.

### 3. USL (Universal Scalability Law): Coherence Overhead Causes Performance Retrograde

In 2007, Neil Gunther proposed USL, extending Amdahl's Law further. He observed that in real systems, performance doesn't just "saturate"—it can actually "retrograde." Adding more nodes can make things slower.

**The Formula**:

$$
C(N) = \frac{N}{1 + \sigma(N-1) + \kappa N(N-1)}
$$

Where:
- $\sigma$ = serialization coefficient (contention)
- $\kappa$ = coherence coefficient (communication overhead)

**The Meeting Room Analogy—Understanding κ's Destructive Power**:

Imagine a team making decisions in a meeting:

- **1 person** (single core): Think alone, decide alone. 100% efficiency.
- **10 people** (multi-core SMP): Every new idea requires confirming "Did everyone hear that?" Meeting efficiency drops to 80%.
- **100 people** (large-scale NUMA): Just confirming "Everyone agrees this data is current" consumes most of the meeting. Actual work time falls below 50%.
- **1000 people** (distributed cluster): Before the actual meeting can even start, the sheer overhead of roll call and syncing information consumes 100% of the time slot. The meeting ends before it begins. No work gets done. This is **Performance Retrograde**—adding more people actually produces *less* output than a smaller team.

This analogy precisely maps to USL's mathematics: the $\kappa N(N-1)$ term represents the cost of "ensuring everyone is synchronized," growing with the **square** of participant count.

**Measured κ Coefficients**:

| System Type | κ Coefficient | Peak Node Count | Typical Scenario |
|-------------|---------------|-----------------|------------------|
| Tightly-coupled SMP | 0.00001 | ~300 | Single-socket multi-core |
| NUMA system | 0.0001 | ~100 | Dual/quad-socket servers |
| Loosely-coupled cluster | 0.001 | ~30 | Distributed training |
| No coherence | 0.01 | ~10 | Early GPU clusters |

**Example**: With $\sigma = 0.02$ (2% serialization) and $\kappa = 0.0001$:
- N=8: $C(8) = 8/(1 + 0.02×7 + 0.0001×8×7) = 6.9$ (86% efficiency)
- N=64: $C(64) = 64/(1 + 0.02×63 + 0.0001×64×63) = 28.5$ (45% efficiency)
- N=128: $C(128) = 128/(1 + 0.02×127 + 0.0001×128×127) = 32.1$ (25% efficiency, **retrograde begins**)

**Implications for Heterogeneous Systems**: This explains why cache coherence protocols are so critical. **Coherence is the κ coefficient's battlefield**—poor protocol design leads to performance retrograde.

> **Citation**: Gunther, N. J. (2007). "A General Theory of Computational Scalability Based on Rational Functions." *arXiv preprint arXiv:0808.1431*.

### 4. Roofline Model: Identifying the Bottleneck Type

In 2009, Samuel Williams and David Patterson proposed the Roofline Model, providing a visual framework for understanding performance bottlenecks.

**Core Concept**: Every workload has an **Operational Intensity (OI)**—the ratio of compute operations to memory bytes accessed:

$$
OI = \frac{FLOPs}{Bytes}
$$

**The Roofline Diagram**:

```
Performance (FLOPS)
        │
        │                    ┌─────────── Compute Bound (Peak FLOPS)
        │                   /│
        │                  / │
        │                 /  │
        │                /   │
        │               /    │
        │              /     │
        │             /      │
        │            /       │
        │           /        │
        │          /         │
        │         /          │
        │        /           │
        │       / Memory Bound (Bandwidth × OI)
        │      /
        │     /
        │    /
        │   /
        │  /
        │ /
        │/
        └─────────────────────────────────────────→ Operational Intensity
                          Ridge Point
```

**Two Regimes**:
- **Memory Bound** (left of ridge): Performance limited by memory bandwidth. Adding compute units won't help.
- **Compute Bound** (right of ridge): Performance limited by compute capacity. Adding bandwidth won't help.

**Real-World Data** (NVIDIA A100):

| Workload | OI (FLOPs/Byte) | Bottleneck | Actual Performance |
|----------|-----------------|------------|-------------------|
| Vector Add | 0.25 | Memory | 500 GFLOPS (25% peak) |
| GEMM (naive) | 1.0 | Memory | 2 TFLOPS (10% peak) |
| GEMM (tiled) | 64 | Compute | 19.5 TFLOPS (100% peak) |
| Transformer Attention | 4-8 | Transitional | 8-12 TFLOPS (40-60% peak) |

**Implications for Heterogeneous Systems**: Different processors target different OI ranges:
- **CPU**: Optimized for low-OI, latency-sensitive workloads
- **GPU**: Optimized for medium-to-high OI through massive parallelism
- **NPU**: Designed to maximize OI through systolic arrays and data reuse

**Design Insight**: Before optimizing, first determine whether you're memory-bound or compute-bound. The prescription differs entirely.

> **Citation**: Williams, S., Waterman, A., & Patterson, D. (2009). "Roofline: An Insightful Visual Performance Model for Multicore Architectures." *Communications of the ACM*, 52(4), 65-76.

### 5. Little's Law: Quantifying Buffer Requirements

In 1961, John Little proved a deceptively simple formula that applies to any stable queuing system:

$$
L = \lambda \times W
$$

Where:
- $L$ = average number of items in the system
- $\lambda$ = arrival rate (throughput)
- $W$ = average time in the system (latency)

**Intuition**: To maintain throughput λ with latency W, you need L items "in flight" at any moment.

**CPU Application**: Why do modern CPUs need 600+ entry ROBs?

```
Target IPC: 6 instructions/cycle
Average memory latency: 100 cycles
Required in-flight instructions: 6 × 100 = 600 entries
```

This is Little's Law in action. The ROB must be large enough to hide memory latency while maintaining target throughput.

**GPU Application**: Why do GPUs need thousands of threads?

```
Memory latency: ~400 cycles
Target bandwidth: 2 TB/s
Cache line: 128 bytes
Required in-flight requests: (2 TB/s × 400 cycles) / 128 bytes ≈ 6,250 requests
Threads per request: 1
Required threads: 6,250+
```

This explains why A100 supports 2048 threads per SM × 108 SMs = 221,184 concurrent threads.

**Implications for Heterogeneous Systems**: When designing buffers, queues, or prefetchers, Little's Law provides the sizing formula. **Buffer size = Throughput × Latency**.

> **Citation**: Little, J. D. C. (1961). "A Proof for the Queuing Formula: L = λW." *Operations Research*, 9(3), 383-387.

### 6. Queuing Theory: Why 100% Utilization Is Catastrophic

The final law comes from queuing theory. When system utilization approaches 100%, latency explodes:

$$
W = \frac{1}{\mu - \lambda}
$$

Where $\mu$ is service rate and $\lambda$ is arrival rate. As $\lambda \to \mu$, $W \to \infty$.

**The Hockey Stick Curve**:

```
Latency
    │
    │                                    *
    │                                   *
    │                                  *
    │                                 *
    │                               *
    │                             *
    │                          *
    │                      *
    │                 *
    │           *
    │      *
    │  *
    │*
    └────────────────────────────────────→ Utilization
    0%                              100%
```

**Real-World Data**: AWS Lambda cold start latency vs. utilization:

| Utilization | P50 Latency | P99 Latency |
|-------------|-------------|-------------|
| 50% | 5ms | 15ms |
| 70% | 8ms | 35ms |
| 85% | 15ms | 120ms |
| 95% | 50ms | 800ms |

**Implications for Heterogeneous Systems**: This is why we need **slack**. Systems designed for 100% utilization will have unpredictable latency. **E-cores handle background tasks so P-cores maintain slack for latency-critical work**.

### Mid-Point Summary: How the Six Laws Map to Hardware Design

Before diving into hardware details, let's establish a mental map—how these six laws guide our subsequent discussion:

```
┌─────────────────────────────────────────────────────────────────┐
│              Six Laws → Hardware Design Mapping                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Amdahl's Law ──────────→ CPU Design (handle serial bottlenecks)│
│  "Serial portions cap speedup"  • Wide decode, large ROB, high IPC│
│                                                                  │
│  Gustafson's Law ───────→ GPU/NPU Design (scale parallel work)  │
│  "Scale problem size"          • Massive cores, high throughput  │
│                                                                  │
│  USL (κ coefficient) ───→ Coherence Protocol Design             │
│  "Coherence overhead retrogrades" • Directory-based, scoped     │
│                                                                  │
│  Roofline Model ────────→ Memory Interconnect Design            │
│  "Identify bottleneck type"    • Bandwidth vs latency tradeoffs │
│                                                                  │
│  Little's Law ──────────→ Buffer Design (ROB/Queue/Buffer)      │
│  "L = λ × W"                   • DPU packet buffers, CXL prefetch│
│                                                                  │
│  Queuing Theory ────────→ Scheduler Design (preserve slack)     │
│  "Avoid 100% utilization"      • E-cores for background, P-cores│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Reading Suggestion**: The following content is information-dense. If you're feeling fatigued, consider jumping to the "Case Studies" section to see how these laws apply concretely in Apple M-series and NVIDIA Grace Hopper, then return for details.

---

## Four Processors, Four Parallel Philosophies

Modern heterogeneous systems typically include four processor types: CPU, GPU, NPU, and DPU. Each embodies a different design philosophy and parallelization strategy—like different specialists on a high-performance team.

### CPU: The Master of Instruction-Level Parallelism

**Design Philosophy**: Few but powerful cores, optimized for latency

The CPU's core mission is maximizing **IPC (Instructions Per Cycle)**. As discussed in the previous article, this requires:

- **Wide decode** (6-8 instructions/cycle)
- **Deep out-of-order execution** (ROB 500+ entries)
- **Sophisticated branch prediction** (>97% accuracy)
- **Large caches** (L3 reaching 64MB+)

**Parallelization Strategy: ILP (Instruction-Level Parallelism)**

CPUs extract parallelism from *within* a single instruction stream. Even sequential code contains independent operations that can execute simultaneously.

**Modern CPU Microarchitecture Specifications**:

| Processor | Decode Width | ROB Size | L3 Cache | Branch Predictor |
|-----------|--------------|----------|----------|------------------|
| Intel Raptor Lake (P-core) | 6-wide | 512 | 36MB | TAGE-like |
| AMD Zen 4 | 6-wide | 320 | 96MB | Perceptron |
| Apple M3 (P-core) | 8-wide | 600+ | 36MB | Custom |
| ARM Cortex-X4 | 8-wide | 384 | 12MB | TAGE |

**Six Laws Perspective**:

- **Amdahl's Law**: CPUs handle the serial portions that limit overall speedup. Wide decode and large ROBs maximize serial code performance.
- **Little's Law**: A 600-entry ROB with 100-cycle memory latency supports 6 IPC—this is Little's Law in silicon.

**Best For**: Control-flow-heavy code, low-latency requirements, serial bottlenecks

**Analogy**: The CPU is like a Michelin-starred chef—supremely skilled, handling complex dishes with finesse, but there are only a few of them.

### GPU: The Master of Data-Level Parallelism

**Design Philosophy**: Many simple cores, optimized for throughput

GPUs take the opposite approach: instead of a few powerful cores, they deploy thousands of simple ones. The goal isn't low latency for individual operations—it's maximum throughput across massive datasets.

**Parallelization Strategy: DLP (Data-Level Parallelism) / SIMT**

GPUs use **SIMT (Single Instruction, Multiple Threads)**—one instruction controls thousands of threads executing in lockstep. This works brilliantly when the same operation applies to many data elements.

**Modern GPU Specifications**:

| GPU | CUDA Cores | Tensor Cores | Memory BW | TDP |
|-----|------------|--------------|-----------|-----|
| NVIDIA A100 | 6,912 | 432 | 2.0 TB/s (HBM2e) | 400W |
| NVIDIA H100 | 16,896 | 528 | 3.35 TB/s (HBM3) | 700W |
| NVIDIA B200 | 18,432 | 576 | 8.0 TB/s (HBM3e) | 1000W |
| AMD MI300X | 19,456 | 1,216 | 5.3 TB/s (HBM3) | 750W |

**Latency Hiding Through Warp Scheduling**:

GPUs don't try to reduce memory latency—they hide it. When one warp (32 threads) stalls on memory, the scheduler instantly switches to another ready warp:

```
Warp 0: Execute → Memory Request → [Waiting...]
Warp 1: [Ready] → Execute → Memory Request → [Waiting...]
Warp 2: [Ready] → Execute → ...
...
Warp 0: [...Waiting...] → Data Returns → Execute
```

With enough warps in flight, the GPU maintains full throughput despite high memory latency. This is Little's Law applied at the thread level.

**Memory Coalescing—Why Data Layout Matters**:

GPU memory controllers merge memory requests from threads in the same warp into single large transactions. This only works when threads access contiguous addresses:

```
AoS Access Pattern (32 threads reading x coordinates):
Thread 0: particles[0].x  → Address 0
Thread 1: particles[1].x  → Address 24  (skipping y, z, vx, vy, vz)
Result: 32 scattered requests, cannot coalesce → ~17% bandwidth utilization

SoA Access Pattern (32 threads reading x coordinates):
Thread 0: particles.x[0]  → Address 0
Thread 1: particles.x[1]  → Address 4
Result: Contiguous 128 bytes, coalesced into 1 request → 100% bandwidth utilization
```

**Quantified Impact**:

| Layout | Bandwidth Utilization | Actual Throughput (A100) | Performance Gap |
|--------|----------------------|--------------------------|-----------------|
| AoS | ~17% | 340 GB/s | 1× |
| SoA | ~100% | 2,000 GB/s | **5.9×** |

**For Software Engineers—The Code Difference**:

```cpp
// AoS (Array of Structures): Bad for GPU Coalescing
struct Particle {
    float x, y, z;      // position
    float vx, vy, vz;   // velocity
};
Particle particles[N];  // accessing particles[i].x strides by 24 bytes

// SoA (Structure of Arrays): Good for GPU Coalescing
struct Particles {
    float x[N], y[N], z[N];     // positions grouped
    float vx[N], vy[N], vz[N];  // velocities grouped
};
Particles p;  // accessing p.x[i] is contiguous—perfect for coalescing
```

This isn't just a GPU optimization. CPU SIMD (AVX/Neon), NPU systolic arrays, and cache prefetchers all benefit from SoA's predictable, contiguous memory access patterns.

**Six Laws Perspective**:

- **Gustafson's Law**: GPUs embody Gustafson—they enable problems of unprecedented scale.
- **Roofline Model**: GPUs push the memory bandwidth slope higher, allowing more workloads to reach compute-bound territory.

**Best For**: Matrix operations, image processing, parallel algorithms with regular access patterns

**Analogy**: The GPU is like a fast-food kitchen—hundreds of workers each doing simple tasks, but together producing enormous throughput.

### NPU: The Master of Operational Intensity

**Design Philosophy**: Domain-specific, maximize data reuse

NPUs (Neural Processing Units) are designed specifically for neural network inference. Their secret weapon is the **Systolic Array**—a grid of processing elements that pass data to neighbors, maximizing reuse.

**Parallelization Strategy: Systolic Array + Data Reuse**

```
┌─────────────────────────────────────────────────────────────┐
│                    Systolic Array (8×8)                      │
│                                                              │
│    Weight →  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐│
│    Flow      │PE │→│PE │→│PE │→│PE │→│PE │→│PE │→│PE │→│PE ││
│              └─↓─┘ └─↓─┘ └─↓─┘ └─↓─┘ └─↓─┘ └─↓─┘ └─↓─┘ └─↓─┘│
│              ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐│
│    Activation│PE │→│PE │→│PE │→│PE │→│PE │→│PE │→│PE │→│PE ││
│    Flow ↓    └─↓─┘ └─↓─┘ └─↓─┘ └─↓─┘ └─↓─┘ └─↓─┘ └─↓─┘ └─↓─┘│
│              ...                                             │
│                                                              │
│    Each PE: Multiply-Accumulate (MAC)                        │
│    Data flows through array, reused N times                  │
└─────────────────────────────────────────────────────────────┘
```

**The Mathematics of Data Reuse**:

For an $N \times N$ matrix multiplication $C = A \times B$:
- **Naive implementation**: Each output element reads $2N$ inputs → $2N^3$ total memory accesses
- **Systolic Array**: Each input reused $N$ times → only $3N^2$ memory accesses

Operational Intensity jumps from $O(1)$ to $O(N)$. For $N = 1024$, that's a 1000× difference!

**Why This Saves Power—Addressing the Power Wall**:

This isn't just about performance—it's about **energy efficiency**. Memory access energy dwarfs computation:

| Operation | Energy (pJ) | Relative to FP32 Multiply |
|-----------|-------------|---------------------------|
| FP32 Multiply | ~4 pJ | 1× |
| Register Read | ~1 pJ | 0.25× |
| L1 Cache Read | ~5 pJ | 1.25× |
| DRAM Read | ~640 pJ | **160×** |
| HBM Read | ~20 pJ | 5× |

**Key Insight**: Reading DRAM once costs as much energy as 160 FP32 multiplies!

Systolic Arrays pass data between PEs (register-to-register, ~1 pJ), avoiding repeated DRAM access (~640 pJ). This is why TPUs achieve **2-4 TOPS/W** efficiency while GPUs typically reach only **0.5-1 TOPS/W**.

**Key Insight**: This massive energy gap is exactly why general-purpose CPUs hit the **Power Wall** mentioned in our introduction. Moving data costs more than computing it. When we can't infinitely increase power, the only path forward is **reducing data movement**.

**Modern NPU Specifications**:

| NPU | Cores | Peak Performance | Memory | TDP | TOPS/W |
|-----|-------|------------------|--------|-----|--------|
| Google TPU v4 | 4 TensorCores | 275 TFLOPS (BF16) | 32GB HBM2e | 175W | 1.6 |
| Google TPU v5e | 8 TensorCores | 393 TFLOPS (BF16) | 16GB HBM2e | 200W | 2.0 |
| Apple Neural Engine (M3) | 16 cores | 18 TOPS | Unified | ~15W | 1.2 |

**Six Laws Perspective**:

- **Roofline Model**: NPUs maximize Operational Intensity through data reuse, pushing workloads into compute-bound territory.
- **Queuing Theory**: NPUs for inference have strict latency requirements. Good NPU design preserves slack for predictable latency.

**Best For**: AI inference, matrix multiplication, convolutions

**Analogy**: The NPU is like an automotive assembly line—each worker does one thing, but parts flow between workers, gradually assembled into finished products. Extremely efficient, but only for specific product types.

> **Citation**: Jouppi, N. P., et al. (2017). "In-Datacenter Performance Analysis of a Tensor Processing Unit." *ISCA '17*.

### DPU: The Master of Data Movement

**Design Philosophy**: Offload infrastructure, free the CPU

**Why Now?** Network bandwidth growth has far outpaced CPU core count growth. Over the past decade, datacenter networks jumped from 10 Gbps to 400/800 Gbps (40-80×), while CPU cores grew from 8 to 128 (16×), with Moore's Law visibly slowing. This means: **if CPUs handle 100 Gbps network packets themselves, protocol stack processing alone would saturate the CPU at 100%, leaving no capacity for business logic.** This is precisely the "offloading" necessity from Amdahl's Law—moving serial bottlenecks off the CPU.

DPUs (Data Processing Units) are the "logistics corps" of heterogeneous systems. They handle network packets, storage requests, encryption/decryption—"infrastructure" tasks that would otherwise tax the CPU.

**Parallelization Strategy: PLP (Packet-Level Parallelism) + Pipeline**

DPU parallelization occurs at the **packet level**. Each network packet or I/O request is independent, processable by different units simultaneously. Internally, DPUs use pipeline designs where packets flow through processing stages:

```
┌─────────────────────────────────────────────────────┐
│                       DPU                            │
│  ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐ │
│  │ Parse  │ → │ Lookup │ → │ Modify │ → │ Crypto │ │
│  └────────┘   └────────┘   └────────┘   └────────┘ │
│       ↑                                       ↓     │
│   Packet In                              Packet Out │
│                                                     │
│  + Multiple ARM/RISC-V cores for complex logic      │
└─────────────────────────────────────────────────────┘
```

**Modern DPU Specifications**:

| DPU | CPU Cores | Network | Crypto Throughput | NVMe | TDP |
|-----|-----------|---------|-------------------|------|-----|
| NVIDIA BlueField-3 | 16× Arm A78 | 400 Gbps | 400 Gbps AES-GCM | 32 NVMe-oF | 75W |
| AMD Pensando Elba | 16× Arm A72 | 200 Gbps | 200 Gbps | 24 NVMe-oF | 45W |
| Intel IPU E2100 | 16× Xeon cores | 200 Gbps | 200 Gbps | 16 NVMe | 100W |

**Quantified Offload Benefits**:

According to NVIDIA data, "infrastructure tax" can consume 30% of CPU resources in cloud environments:

| Function | CPU Load (No DPU) | CPU Load (With DPU) | Offload Rate |
|----------|-------------------|---------------------|--------------|
| Network virtualization (OVS) | 12-15% | <1% | 95%+ |
| Storage acceleration (NVMe-oF) | 8-10% | <1% | 90%+ |
| Encryption/Decryption | 5-8% | <1% | 95%+ |
| **Total** | **25-33%** | **<3%** | **90%+** |

**Six Laws Perspective**:

- **Amdahl's Law**: DPUs reduce the CPU's "infrastructure tax," effectively shrinking the serial portion.
- **Queuing Theory**: DPUs provide deterministic latency for network/storage operations, improving P99 predictability.

**Best For**: Network virtualization, storage offload, security acceleration

**Analogy**: The DPU is like a CEO's executive assistant—handling scheduling, correspondence, and logistics so the CEO (CPU) can focus on strategic decisions.

---

## The Memory Wall Showdown: UMA, CXL, and NVLink

Memory bandwidth and latency are the ultimate battleground in heterogeneous systems. Different interconnect technologies represent different design philosophies.

### Apple UMA: Unified Memory for Extreme Integration

Apple's Unified Memory Architecture (UMA) takes a radical approach: **all processors share a single memory pool with zero-copy semantics**.

**Design Philosophy**: Instead of copying data between CPU and GPU memory, both access the same physical addresses. This eliminates:
- PCIe transfer latency (typically 10-20μs)
- Memory copy overhead (limited by bus bandwidth)
- Double memory allocation (no separate GPU memory)

**Apple Silicon Memory Specifications**:

| SoC | Memory | Bandwidth | Latency | Capacity |
|-----|--------|-----------|---------|----------|
| M1 | LPDDR4X | 68 GB/s | ~80ns | 8-16GB |
| M2 | LPDDR5 | 100 GB/s | ~90ns | 8-24GB |
| M3 Max | LPDDR5 | 400 GB/s | ~90ns | 36-128GB |
| M2 Ultra | LPDDR5 | 800 GB/s | ~90ns | 64-192GB |

**The Zero-Copy Advantage**:

Traditional discrete GPU workflow:
```
CPU Memory → PCIe (16 GB/s) → GPU Memory → Compute → PCIe → CPU Memory
Total latency: ~50-100μs per transfer
```

Apple UMA workflow:
```
Unified Memory → Compute (GPU accesses same addresses)
Total latency: ~0 (no transfer needed)
```

For interactive applications (video editing, 3D modeling), this latency difference is perceptible.

**Limitation**: Capacity. Apple's maximum is 192GB (M2 Ultra). Datacenter workloads requiring terabytes of memory need different solutions.

### CXL: Memory Disaggregation for the Datacenter

CXL (Compute Express Link) takes the opposite approach from UMA: **separate memory pools connected via high-speed fabric**.

**Design Philosophy**: Instead of putting all memory on-chip (capacity limited), connect to external memory pools via CXL fabric. Trade latency for capacity.

**CXL Version Evolution**:

| Version | Features | Latency | Use Case |
|---------|----------|---------|----------|
| CXL 1.1 | Type 3 memory expander | 150-200ns | Memory capacity expansion |
| CXL 2.0 | Memory pooling, switching | 200-350ns | Shared memory pools |
| CXL 3.0 | Fabric, back-to-back switch | 300-500ns | Datacenter-scale memory |

**Memory Tiering Strategy**:

CXL enables **tiered memory** where hot data stays in local DRAM while cold data moves to CXL-attached memory:

```
┌─────────────────────────────────────────────────────────────┐
│                    Memory Tiering                            │
├─────────────────────────────────────────────────────────────┤
│  Tier 0: CPU Cache        ~1ns     Hot data (working set)   │
│  Tier 1: Local DDR5      ~80ns     Warm data (active pages) │
│  Tier 2: CXL Memory     ~200ns     Cold data (infrequent)   │
│  Tier 3: CXL Pooled     ~500ns     Archive (shared pool)    │
└─────────────────────────────────────────────────────────────┘
```

**Six Laws Perspective**:
- **Little's Law**: CXL prefetchers must maintain more in-flight requests to hide higher latency
- **Queuing Theory**: CXL fabric congestion can cause latency spikes; careful traffic shaping required

### NVLink: High-Bandwidth GPU Interconnect

NVIDIA's NVLink provides **ultra-high bandwidth** between GPUs and between GPU and CPU (in Grace Hopper).

**NVLink Evolution**:

| Version | Bandwidth (Bidirectional) | Links | Use Case |
|---------|---------------------------|-------|----------|
| NVLink 1.0 | 80 GB/s | 4 | P100 GPU-GPU |
| NVLink 2.0 | 150 GB/s | 6 | V100 GPU-GPU |
| NVLink 3.0 | 600 GB/s | 12 | A100 GPU-GPU |
| NVLink 4.0 | 900 GB/s | 18 | H100 GPU-GPU |
| NVLink-C2C | 900 GB/s | - | Grace Hopper CPU-GPU |

**NVLink-C2C in Grace Hopper**:

The Grace Hopper superchip uses NVLink-C2C to connect CPU and GPU on the same module:

```
┌───────────────────────────────────────────────────────────────┐
│                     Grace Hopper GH200                        │
│  ┌─────────────────────┐     ┌─────────────────────────────┐ │
│  │   Grace CPU         │     │       Hopper GPU            │ │
│  │   72 Arm Cores      │     │       80GB HBM3             │ │
│  │   512GB LPDDR5X     │◄───►│       3.35 TB/s BW          │ │
│  │                     │ 900 │                             │ │
│  │                     │GB/s │                             │ │
│  └─────────────────────┘     └─────────────────────────────┘ │
│                    NVLink-C2C                                 │
└───────────────────────────────────────────────────────────────┘
```

**Unified Memory Address Space**: CPU and GPU share a unified virtual address space. The GPU can directly access CPU memory and vice versa, similar to Apple UMA but at datacenter scale.

### The Latency-Capacity Tradeoff: A 2D Quadrant View

These three technologies occupy different positions in the latency-capacity design space:

```
                          Latency (log scale)
                    Low ◄────────────────────► High

              ┌─────────────────────────────────────────┐
         High │                                         │
              │                        ┌──────────────┐ │
              │                        │   CXL 3.0   │ │
              │                        │  Memory Pool │ │
              │                        └──────────────┘ │
     Capacity │          ┌─────────────┐                │
              │          │  NVLink-C2C │                │
              │          │ Grace Hopper│                │
              │          └─────────────┘                │
              │  ┌─────────────┐                        │
          Low │  │  Apple UMA  │                        │
              │  │  M3 Ultra   │                        │
              │  └─────────────┘                        │
              └─────────────────────────────────────────┘

Interpretation:
• Apple UMA: Extreme low latency, limited capacity (192GB max)
• NVLink-C2C: High bandwidth / Medium capacity (900 GB/s, 592GB total, ~100ns)
• CXL Memory Pool: Extreme capacity, higher latency (TB-scale, 200-500ns)
```

**Design Guidance**:
- **Interactive/edge applications** → Apple UMA (latency-critical, smaller models)
- **AI training/inference servers** → NVLink (balance of bandwidth and capacity)
- **Memory-intensive analytics** → CXL pooling (capacity trumps latency)

---

## Coherence Protocols: The Hidden Tax

Cache coherence ensures all processors see consistent data. But this consistency has a cost—the κ coefficient from USL.

### Why Coherence Matters

When CPU and GPU share memory, what happens when both modify the same cache line?

Without coherence:
```
Time 0: Memory[X] = 0
Time 1: CPU writes Memory[X] = 1 (CPU cache updated)
Time 2: GPU reads Memory[X] → sees 0 (stale!)
Time 3: GPU writes Memory[X] = 2 (overwrites CPU's update!)
```

With coherence:
```
Time 0: Memory[X] = 0
Time 1: CPU writes Memory[X] = 1
        → Coherence protocol invalidates GPU's cached copy
Time 2: GPU reads Memory[X] → cache miss → fetches 1 ✓
Time 3: GPU writes Memory[X] = 2
        → Coherence protocol invalidates CPU's cached copy
```

Coherence ensures correctness, but every invalidation/synchronization message adds latency.

### TileLink: RISC-V's Modular Coherence

TileLink is the coherence protocol used in RISC-V SoCs (SiFive, various academic chips). It uses five channels:

| Channel | Direction | Purpose |
|---------|-----------|---------|
| A | Master → Slave | Acquire (request data/permissions) |
| B | Slave → Master | Probe (invalidate/downgrade request) |
| C | Master → Slave | Release (voluntary writeback) |
| D | Slave → Master | Grant (data/permissions response) |
| E | Master → Slave | GrantAck (final acknowledgment) |

**Example: CPU Read Miss**:
```
CPU                     Directory                  Memory
 │                          │                         │
 │──── A: Acquire ─────────►│                         │
 │                          │──── Read ──────────────►│
 │                          │◄─── Data ───────────────│
 │◄─── D: Grant ────────────│                         │
 │──── E: GrantAck ────────►│                         │
```

### ARM CHI: Enterprise-Grade Coherence

ARM's CHI (Coherent Hub Interface) powers enterprise SoCs like AWS Graviton and Ampere Altra. Key features:

- **Distributed Directory**: No single bottleneck; coherence state distributed across nodes
- **Speculative Reads**: Reduce latency by initiating reads before knowing if data is shared
- **Partial Cache Line Transfers**: Save bandwidth when only part of line is needed

**CHI vs. TileLink**:

| Aspect | TileLink | ARM CHI |
|--------|----------|---------|
| Target scale | 1-16 cores | 64+ cores |
| Directory | Centralized | Distributed |
| Message complexity | 5 channels | 12+ message types |
| Silicon area | Smaller | Larger |

### IOMMU: Device Coherence

When accelerators (GPU, NPU, DPU) participate in coherence, IOMMU translates virtual addresses and enforces access permissions:

```
┌───────────────────────────────────────────────────────────────┐
│                    Coherent System                            │
│                                                               │
│  ┌──────┐   ┌──────┐   ┌──────────┐   ┌──────────────────┐  │
│  │ CPU  │   │ GPU  │   │   DMA    │   │   Memory         │  │
│  │ +TLB │   │+IOMMU│   │ +IOMMU   │   │  Controller      │  │
│  └───┬──┘   └───┬──┘   └────┬─────┘   └────────┬─────────┘  │
│      │          │           │                   │            │
│      └──────────┴───────────┴───────────────────┘            │
│                   Coherent Interconnect                       │
│                (TileLink / CHI / CXL.cache)                   │
└───────────────────────────────────────────────────────────────┘
```

**CXL.cache Protocol**: CXL defines a coherence protocol allowing accelerators to participate in host CPU's coherence domain. This enables accelerators to cache host memory coherently.

---

## Software-Hardware Collaboration: From MLIR to Scheduling

Hardware without software is useless. This section explores how software extracts performance from heterogeneous hardware.

### Heterogeneous Scheduling: Global vs. Local

Modern heterogeneous systems employ a **hierarchical scheduling** architecture:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Hierarchical Scheduling                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Global Scheduler (OS/Runtime)                │   │
│  │  • Runs on CPU                                            │   │
│  │  • Decomposes Task Graph                                  │   │
│  │  • Assigns tasks to processors (CPU/GPU/NPU/DPU)          │   │
│  │  • Manages cross-processor dependencies                   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│              ┌───────────────┼───────────────┐                  │
│              ▼               ▼               ▼                  │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐      │
│  │ CPU Scheduler  │ │ GPU Command    │ │ NPU Firmware   │      │
│  │ (OS Thread     │ │ Processor      │ │ Scheduler      │      │
│  │  Scheduler)    │ │ (Local Sched)  │ │                │      │
│  └────────────────┘ └────────────────┘ └────────────────┘      │
│                                                                  │
│  Local Schedulers: Micro-level dispatching based on             │
│  execution unit availability and cache states                   │
└─────────────────────────────────────────────────────────────────┘
```

**Six Laws Perspective**:

- **Amdahl's Law**: The Global Scheduler itself is a serial bottleneck. Its efficiency caps total system throughput. This is why GPU drivers batch commands—reducing scheduler overhead.
- **Little's Law**: To hide scheduling latency, the system needs enough "in-flight tasks." GPU drivers maintain deep Command Queues (hundreds of commands) to ensure the GPU never starves.
- **Queuing Theory**: Scheduler queues must be sized to handle burst arrivals without overflow, while maintaining acceptable latency.

**Example: CUDA Stream Scheduling**:
```
CPU Thread                    GPU Command Processor
    │                                │
    │── Launch Kernel A ────────────►│ Queue: [A]
    │── Launch Kernel B ────────────►│ Queue: [A, B]
    │── Launch Kernel C ────────────►│ Queue: [A, B, C]
    │                                │
    │   (CPU continues other work)   │── Execute A
    │                                │── Execute B
    │                                │── Execute C
```

The CPU doesn't wait for each kernel to complete—it queues work and continues. This is Little's Law in action: maintaining enough in-flight work to hide latency.

### Compiler Infrastructure: MLIR and Multi-Level Optimization

**MLIR (Multi-Level Intermediate Representation)** is a compiler framework designed for heterogeneous systems. Unlike traditional compilers with 2-3 IRs, MLIR supports arbitrary **Dialects**—domain-specific IRs that capture different abstraction levels.

**Dialect Hierarchy**:

```
High-Level               Mid-Level                Low-Level
┌────────────┐          ┌────────────┐           ┌────────────┐
│ TensorFlow │          │   Linalg   │           │   Vector   │
│   Dialect  │    →     │   Dialect  │     →     │   Dialect  │
│            │          │            │           │            │
│ tf.MatMul  │          │ linalg.    │           │ vector.    │
│            │          │ matmul     │           │ contract   │
└────────────┘          └────────────┘           └────────────┘
      │                       │                        │
      │                       │                        │
      ▼                       ▼                        ▼
┌────────────┐          ┌────────────┐           ┌────────────┐
│   Affine   │          │    SCF     │           │    LLVM    │
│   Dialect  │    →     │   Dialect  │     →     │   Dialect  │
│            │          │            │           │            │
│ affine.for │          │  scf.for   │           │ llvm.call  │
└────────────┘          └────────────┘           └────────────┘
```

**Concrete Example: Matrix Multiplication Transformation**

Let's trace a 4×4 MatMul through MLIR's dialect levels:

**Level 1: TensorFlow Dialect (Semantic Layer)**
```mlir
// What operation? MatMul. What data types? Float32.
%result = "tf.MatMul"(%A, %B) : (tensor<4x4xf32>, tensor<4x4xf32>) -> tensor<4x4xf32>
```

**Level 2: Linalg Dialect (Algorithm Layer)**
```mlir
// MatMul = C[i,j] += A[i,k] * B[k,j]
linalg.matmul ins(%A, %B : tensor<4x4xf32>, tensor<4x4xf32>)
              outs(%C : tensor<4x4xf32>)
```

**Level 3: Affine Dialect (Loop Structure Layer)**
```mlir
// Explicit nested loops with affine indexing
affine.for %i = 0 to 4 {
  affine.for %j = 0 to 4 {
    affine.for %k = 0 to 4 {
      %a = affine.load %A[%i, %k] : memref<4x4xf32>
      %b = affine.load %B[%k, %j] : memref<4x4xf32>
      %c = affine.load %C[%i, %j] : memref<4x4xf32>
      %prod = arith.mulf %a, %b : f32
      %sum = arith.addf %c, %prod : f32
      affine.store %sum, %C[%i, %j] : memref<4x4xf32>
    }
  }
}
```

**Level 4: Vector Dialect (SIMD Layer)**
```mlir
// Vectorized operations using SIMD
%va = vector.load %A[%i, %k] : memref<4x4xf32>, vector<4xf32>
%vb = vector.load %B[%k, %j] : memref<4x4xf32>, vector<4xf32>
%vc = vector.contract %va, %vb, %acc : vector<4xf32>, vector<4xf32> into f32
```

**Level 5: LLVM Dialect (Machine Layer)**
```mlir
// Target-specific intrinsics
%result = llvm.call @llvm.x86.avx.dp.ps.256(%va, %vb, %mask)
```

**Key Insight**: This level transformation (Lowering) is precisely where the compiler performs **"architecture-aware optimization"**. For example, only at the Linalg level does the compiler know it's dealing with matrix multiplication, allowing it to choose optimal Tiling strategies based on target cache sizes. Once lowered to LLVM IR, this high-level semantic information is lost, and optimization opportunities vanish with it.

### Programming Model Comparison

Different programming models offer different tradeoffs:

| Model | Portability | Performance | Ease of Use | Target |
|-------|-------------|-------------|-------------|--------|
| CUDA | Low (NVIDIA only) | Highest | Medium | GPU |
| OpenCL | High | Medium | Low | Any accelerator |
| SYCL | High | High | Medium | Any accelerator |
| OpenMP Offload | High | Medium | High | CPU + accelerators |
| HIP | Medium (AMD+NVIDIA) | High | Medium | GPU |

**SYCL Example**:
```cpp
sycl::queue q;
q.parallel_for(sycl::range<1>(N), [=](sycl::id<1> i) {
    C[i] = A[i] + B[i];
}).wait();
```

This code runs on CPUs, GPUs, or FPGAs with appropriate backend selection.

---

## PPA Tradeoffs: Power, Performance, and Area

Every design decision involves tradeoffs among Power, Performance, and Area (PPA). Understanding these tradeoffs is essential for architects.

### DVFS: Dynamic Voltage Frequency Scaling

Power scales with voltage and frequency:

$$
P = C \cdot V^2 \cdot f
$$

Where $C$ is switched capacitance, $V$ is voltage, and $f$ is frequency.

**The Cubic Relationship**: Raising frequency requires raising voltage for stability. If doubling frequency requires 1.2× voltage:

$$
P_{new} = C \cdot (1.2V)^2 \cdot (2f) = 2.88P
$$

Nearly 3× power for 2× frequency! This is why frequency scaling has diminishing returns.

**Intel i9-13900K DVFS Example**:

| P-State | Frequency | Voltage | TDP | Performance |
|---------|-----------|---------|-----|-------------|
| Base | 3.0 GHz | 0.95V | 125W | 1.0× |
| Boost | 5.4 GHz | 1.35V | 253W | ~1.6× |
| Max | 5.8 GHz | 1.45V | 280W+ | ~1.7× |

**Race-to-Sleep**: Sometimes the most efficient approach is running at maximum frequency briefly, then sleeping. But the math is nuanced:

**Real-World Example: A 50ms Computational Task**

Consider a task that takes 50ms at peak frequency:

| Strategy | Frequency | Power | Time | Energy |
|----------|-----------|-------|------|--------|
| High Frequency | 5.8 GHz | 253W | 50ms | **12.65 J** |
| Low Frequency | 3.0 GHz | 45W | 97ms | **4.37 J** |

In this specific case, running slower saves **65% energy**! So why ever run fast?

**The Answer: System-Level Idle Power**

```
Scenario: Background task keeping CPU awake

Strategy A (Slow): 3.0 GHz for 97ms + system idle power during task
Strategy B (Fast): 5.8 GHz for 50ms + return to deep C-state (near-zero power)

If system idle power is 20W:
- Strategy A: 4.37J + (20W × 0ms saved) = 4.37J
- Strategy B: 12.65J + (0W × 47ms in C-state) = 12.65J

Strategy A wins for isolated tasks.

But if the task blocks other work:
- Strategy A: 4.37J + 47ms of blocked latency
- Strategy B: 12.65J + 0ms blocked latency

For interactive/latency-sensitive work, Strategy B wins.
```

**Decision Framework**:
- **Deadline pressure** → Race-to-Sleep (high frequency)
- **Interactive latency requirements** → Race-to-Sleep
- **Pure batch processing** → Low frequency
- **Thermal-constrained** → Low frequency

### Big.LITTLE: Heterogeneous CPU Cores

ARM's big.LITTLE (and Intel's P/E cores) uses different core types for different workloads:

| Aspect | P-Core (Performance) | E-Core (Efficiency) |
|--------|---------------------|---------------------|
| Design goal | Maximum single-thread | Maximum perf/watt |
| Decode width | 6-8 wide | 4 wide |
| Out-of-order | Deep (300+ ROB) | Shallow (<100 ROB) |
| Power | 15-30W/core | 3-5W/core |
| Use case | Latency-critical | Background tasks |

**Scheduling Strategy**:
```
┌──────────────────────────────────────────────────────────┐
│                  Thread Scheduler                         │
│                                                          │
│    High Priority + Interactive ────────► P-Cores         │
│    Background + Batch          ────────► E-Cores         │
│    Thermal Throttling          ────────► Migrate to E    │
└──────────────────────────────────────────────────────────┘
```

**Six Laws Connection**: This is Queuing Theory in action—P-cores maintain slack for latency-critical work while E-cores handle background load.

---

## Case Studies: Theory Meets Reality

### Apple M-Series: UMA Mastery

Apple's M-series chips demonstrate what's possible with tight hardware-software integration.

**Evolution**:

| SoC | CPU Cores | GPU Cores | Neural Engine | Memory BW | Process |
|-----|-----------|-----------|---------------|-----------|---------|
| M1 | 4P+4E | 7-8 | 16 | 68 GB/s | 5nm |
| M2 | 4P+4E | 8-10 | 16 | 100 GB/s | 5nm |
| M3 | 4P+4E | 10 | 16 | 100 GB/s | 3nm |
| M3 Max | 12P+4E | 40 | 16 | 400 GB/s | 3nm |
| M2 Ultra | 16P+8E | 76 | 32 | 800 GB/s | 5nm |

**Six Laws Analysis of M3 Max**:

1. **Amdahl's Law**: 12 P-cores with 8-wide decode handle serial bottlenecks efficiently
2. **Gustafson's Law**: 40 GPU cores enable larger creative workloads (8K video, 3D rendering)
3. **Roofline**: 400 GB/s unified bandwidth keeps GPU cores fed
4. **Little's Law**: Unified memory means CPU can prefetch data GPU will use—reduced in-flight requirements
5. **Queuing Theory**: 4 E-cores handle background tasks, preserving P-core slack

**Key Innovation**: Zero-copy unified memory eliminates the PCIe bottleneck that plagues discrete GPU systems.

### NVIDIA Grace Hopper: Datacenter Superchip

Grace Hopper combines ARM CPU and Hopper GPU on one module.

**Specifications**:

| Component | Grace CPU | Hopper GPU |
|-----------|-----------|------------|
| Cores | 72 Arm Neoverse V2 | 132 SMs, 16896 CUDA cores |
| Memory | 512GB LPDDR5X | 80GB HBM3 |
| Bandwidth | 546 GB/s | 3.35 TB/s |
| Interconnect | NVLink-C2C: 900 GB/s bidirectional |

**Unified Memory Advantages**:

Unlike traditional GPU servers where CPU and GPU have separate memory spaces:
- No explicit data copies between CPU and GPU memory
- CPU can access GPU HBM directly (and vice versa)
- Enables larger-than-GPU-memory models with automatic page migration

**Example: LLM Inference**:
```
Traditional: Load 175B parameter model
→ 350GB weights won't fit in 80GB GPU memory
→ Complex model parallelism, frequent PCIe transfers

Grace Hopper: 512GB CPU + 80GB GPU unified space
→ Hot layers in HBM, cold layers in LPDDR5X
→ Automatic page migration based on access patterns
→ Simpler programming, better performance
```

### AI Inference Server: Latency Breakdown

Let's analyze a real AI inference request flow:

```
┌──────────────────────────────────────────────────────────────┐
│           Inference Request Timeline (P50)                   │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Network Receive      ████ 0.5ms                            │
│  Deserialization      ██ 0.2ms                              │
│  Tokenization         ████ 0.5ms                            │
│  GPU Transfer         ██ 0.3ms                              │
│  GPU Compute          ████████████████████ 3.0ms            │
│  GPU→CPU Transfer     ██ 0.2ms                              │
│  Postprocessing       ██ 0.2ms                              │
│  Serialization        ██ 0.2ms                              │
│  Network Send         ████ 0.4ms                            │
│                                              ─────────────── │
│                                   Total:     5.5ms           │
└──────────────────────────────────────────────────────────────┘
```

**Observations**:
- GPU compute is only 55% of total latency
- Data movement (transfers, serialization) consumes 45%
- DPU can offload network/serialization, reducing non-compute portion

**Batch Size Tradeoff**:

| Batch Size | Latency (P50) | Throughput | GPU Utilization |
|------------|---------------|------------|-----------------|
| 1 | 5.5ms | 180 req/s | 35% |
| 8 | 12ms | 660 req/s | 65% |
| 32 | 35ms | 915 req/s | 82% |
| 128 | 120ms | 1070 req/s | 89% |

**Tradeoff**: Larger batches improve throughput and utilization but hurt latency. This is Queuing Theory in action—batch queuing adds latency.

---

## Future Trends: Where Are We Heading?

### Processing-In-Memory (PIM)

The ultimate solution to the memory wall: put compute *inside* memory.

**Concept**: Instead of moving data to compute units, embed compute units in memory chips.

**Current Products**:
- **Samsung HBM-PIM**: DRAM dies with embedded MAC units
- **UPMEM**: Standard DDR4 DIMMs with processing cores

**Six Laws Perspective**: PIM maximizes Operational Intensity by eliminating data movement entirely.

### Chiplets and Advanced Packaging

**Monolithic Limitations**: Larger dies have lower yields and hit reticle limits (~800mm²).

**Chiplet Solution**: Build smaller "chiplets" and interconnect them via advanced packaging:

| Technology | Bandwidth | Pitch | Use Case |
|------------|-----------|-------|----------|
| MCM | 1-10 GB/s | ~1mm | Traditional multi-chip |
| 2.5D (CoWoS) | 100+ GB/s | ~50μm | GPU + HBM |
| 3D stacking | 1+ TB/s | ~10μm | L3 cache stacking |

**Examples**:
- AMD MI300: GPU + CPU chiplets on same package
- Intel Ponte Vecchio: 47 tiles, 5 process nodes

### CXL 3.0 and Memory Pooling

**CXL 3.0 Features**:
- **Back-to-back switching**: Multi-hop fabric support
- **Memory sharing**: Multiple hosts access same memory pool
- **Enhanced coherence**: Peer-to-peer device communication

**Vision**: Datacenter memory becomes a shared resource, allocated dynamically to workloads.

```
┌────────────────────────────────────────────────────────────┐
│                    CXL 3.0 Fabric                          │
│                                                            │
│   ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐      │
│   │Server 1│   │Server 2│   │Server 3│   │Server 4│      │
│   └───┬────┘   └───┬────┘   └───┬────┘   └───┬────┘      │
│       │            │            │            │            │
│       └────────────┴────────────┴────────────┘            │
│                         │                                  │
│              ┌──────────┴──────────┐                      │
│              │   CXL Switch Fabric  │                      │
│              └──────────┬──────────┘                      │
│                         │                                  │
│    ┌────────────────────┼────────────────────┐            │
│    │                    │                    │            │
│ ┌──┴───┐            ┌──┴───┐            ┌──┴───┐         │
│ │Memory│            │Memory│            │Memory│         │
│ │Pool 1│            │Pool 2│            │Pool 3│         │
│ │ 1TB  │            │ 2TB  │            │ 1TB  │         │
│ └──────┘            └──────┘            └──────┘         │
└────────────────────────────────────────────────────────────┘
```

---

## Conclusion: From Processor-Centric to Data-Centric

We've covered substantial ground. Let's synthesize.

### The Six Laws: A Unified Perspective

These six laws aren't independent—they form an interconnected framework:

```
┌─────────────────────────────────────────────────────────────┐
│                   Six Laws Interconnection                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Amdahl ←──────── Tension ────────► Gustafson             │
│   "Serial limits"              "Scale problem size"         │
│         │                              │                    │
│         │                              │                    │
│         ▼                              ▼                    │
│   CPU Design                    GPU/NPU Design              │
│   (handle serial)              (handle parallel)            │
│         │                              │                    │
│         └──────────────┬───────────────┘                    │
│                        │                                    │
│                        ▼                                    │
│                      USL                                    │
│            "Coherence overhead limits scaling"              │
│                        │                                    │
│         ┌──────────────┼──────────────┐                    │
│         │              │              │                    │
│         ▼              ▼              ▼                    │
│     Roofline       Little's       Queuing                  │
│   "Know your       "Size your     "Maintain               │
│    bottleneck"      buffers"       slack"                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### The Paradigm Shift

> **"The era of processor-centric design is over. The era of data-centric design has begun."**

For decades, we optimized by making processors faster. The Memory Wall and Power Wall have ended that era.

**The Complete Picture—Processor-Centric vs. Data-Centric**:

| Dimension | The Past (Processor-Centric) | The Future (Data-Centric) |
|-----------|------------------------------|---------------------------|
| **Hardware Design** | Moving data to the processor | Building processors around data layout |
| **Software Design** | Object-Oriented Design (OOD) | **Data-Oriented Design (DOD)** |
| **Data Structure** | AoS (Array of Structures) - Intuitive | **SoA (Structure of Arrays)** - Bandwidth efficient |
| **Optimization Goal** | Minimizing CPU cycles | Minimizing data movement distance |
| **Key Metric** | IPC, Clock Frequency | Bandwidth Utilization, Efficiency (TOPS/W) |

Modern systems are designed around **data movement**, not computation:
- **UMA**: Eliminate copies through shared memory
- **NVLink/CXL**: Provide high-bandwidth data paths
- **PIM**: Put compute where data lives
- **Systolic Arrays**: Maximize data reuse

**For Software Engineers—Data-Oriented Design**:

This hardware shift demands a software mindset shift:

| Object-Oriented Design | Data-Oriented Design |
|------------------------|----------------------|
| Organize by entities (Person, Account) | Organize by data layout (PersonPositions, PersonVelocities) |
| AoS: Array of Structures | SoA: Structure of Arrays |
| Polymorphism via virtual functions | Polymorphism via data tables |
| Cache-oblivious | Cache-conscious |

**Rethink Data Layout**: As demonstrated in the GPU section, the choice between AoS and SoA can mean a **5.9× performance difference**. But this isn't just about GPUs—CPU SIMD (AVX/Neon), NPU systolic arrays, and even cache prefetchers all benefit from contiguous, predictable memory access patterns. In a data-centric world, how you arrange bytes in memory matters more than how you arrange classes in code.

**The Takeaway**: Understanding data movement costs is now as important as understanding algorithmic complexity.

### Looking Forward

The trends are clear:
1. **More specialization**: Domain-specific accelerators for AI, video, crypto, networking
2. **Higher integration**: UMA, NVLink-C2C, 3D stacking bringing compute closer to memory
3. **Memory disaggregation**: CXL enabling flexible memory allocation
4. **Software-hardware co-design**: MLIR and specialized compilers bridging the abstraction gap

The heterogeneous era isn't coming—it's here. Architects and engineers who master these six laws, understand the tradeoffs, and think "data-first" will build the systems that define the next decade of computing.

---

## References

### Foundational Papers

1. Amdahl, G. M. (1967). "Validity of the Single Processor Approach to Achieving Large Scale Computing Capabilities." *AFIPS Conference Proceedings*, 30, 483-485.

2. Gustafson, J. L. (1988). "Reevaluating Amdahl's Law." *Communications of the ACM*, 31(5), 532-533.

3. Gunther, N. J. (2007). "A General Theory of Computational Scalability Based on Rational Functions." *arXiv preprint arXiv:0808.1431*.

4. Williams, S., Waterman, A., & Patterson, D. (2009). "Roofline: An Insightful Visual Performance Model for Multicore Architectures." *Communications of the ACM*, 52(4), 65-76.

5. Little, J. D. C. (1961). "A Proof for the Queuing Formula: L = λW." *Operations Research*, 9(3), 383-387.

6. Jouppi, N. P., et al. (2017). "In-Datacenter Performance Analysis of a Tensor Processing Unit." *ISCA '17*.

### Textbooks

7. Hennessy, J. L., & Patterson, D. A. (2019). *Computer Architecture: A Quantitative Approach* (6th ed.). Morgan Kaufmann.

8. Patterson, D. A., & Hennessy, J. L. (2021). *Computer Organization and Design RISC-V Edition* (2nd ed.). Morgan Kaufmann.

### Technical Documentation

9. Apple Inc. (2023). "Apple Silicon Architecture Overview." Developer Documentation.

10. NVIDIA Corporation. (2024). "NVIDIA Grace Hopper Superchip Architecture Whitepaper."

11. CXL Consortium. (2023). "Compute Express Link Specification 3.0."

12. SiFive. (2023). "TileLink Specification 1.8."

13. ARM Ltd. (2022). "AMBA CHI Protocol Specification."

### Related Books (Same Author)

14. Jiang, D. (2025). *Data Structures in Practice: A Hardware-Aware Approach for System Software Engineers*. Covers AoS vs SoA, cache behavior, and memory hierarchy in depth.

15. Jiang, D. (2026). *Performance and Benchmarking: Beyond the Bottleneck—From Classic Systems to Modern AI and HPC*. Comprehensive guide to performance analysis, Roofline Model, and AI/HPC benchmarking.

16. Jiang, D. (2026). *System Design: An Architecture-Aware Approach for Unstable Hardware Assumptions*. Framework for heterogeneous system design, covering NUMA, coherence protocols, and offload decisions.

---

*This article is the second in the Computer Architecture series. The first article, "All Roads Lead to IPC: Rethinking CPU Performance Design," explored single-CPU microarchitecture. Future articles will explore specific topics in greater depth.*
