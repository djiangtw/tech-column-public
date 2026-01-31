# All Roads Lead to IPC: Rethinking CPU Performance Design

**Author**: Danny Jiang
**Date**: 2026-01-28

---

## Introduction: The Question That Changed Everything

Years ago, while debugging a processor's performance bottleneck, I stumbled upon a seemingly simple question: "Why can we issue a new multiply instruction every cycle when the multiply unit has a 4-cycle latency?"

At the time, I conflated **Latency** with **Occupation** (also known as Initiation Interval). I assumed that a 4-cycle latency meant the multiplier could only handle one instruction every 4 cycles. But benchmarks told a different story—back-to-back independent multiplies achieved nearly one instruction per cycle.

This confusion sent me back to Hennessy & Patterson's classic textbook, where I finally understood something fundamental:

> **Every CPU microarchitecture feature exists for one purpose: to maximize IPC.**

Superscalar execution, out-of-order processing, branch prediction, caching, reorder buffers—these aren't independent features. They're **remedial mechanisms invented to rescue IPC** from the harsh realities of memory latency, data dependencies, and control hazards.

This article shares that understanding.

---

## Latency vs. Occupation: Two Concepts That Sound Similar But Aren't

Before diving into IPC, we need to untangle two frequently confused terms: **Latency** and **Occupation**.

### Latency

**Definition**: The number of cycles from when an operation begins execution until its result is available for use by subsequent instructions.

**In plain English**: How long until the answer is ready?

**Example**: An integer multiply with Latency = 4 cycles means:

```
Cycle 0: Instruction issued
Cycle 4: Result becomes available
```

If the next instruction needs this result, it **must wait 4 cycles**.

### Occupation (Initiation Interval)

**Definition**: How long a functional unit is "occupied" before it can accept another operation. Also called **Initiation Interval (II)** or **Reciprocal Throughput** in Intel documentation.

**In plain English**: How soon can this hardware accept the next job?

**Example**: The same multiply instruction with Occupation = 1 cycle means:

```
Cycle 0: Issue first mul
Cycle 1: Issue second mul
Cycle 2: Issue third mul
```

Even though the first multiply hasn't finished, the hardware **can already accept new work**.

### The Critical Difference

| Aspect | Latency | Occupation |
| ------ | ------- | ---------- |
| Measures | Time until result is ready | Time until unit accepts new work |
| Affects | Dependency stalls | Throughput |
| Matters for | Dependent instructions | Independent instructions |
| Lower is better | Yes | Yes |

### A Tale of Two Scenarios

Assume: Latency = 4 cycles, Occupation = 1 cycle

**Scenario A: With Dependencies**

```asm
mul x1, x2, x3
add x4, x1, x5   # needs x1
```

The `add` must wait 4 cycles. This is a **latency-induced stall**.

**Scenario B: No Dependencies**

```asm
mul x1, x2, x3
mul x4, x5, x6
mul x7, x8, x9
```

One multiply issues every cycle. This is **high throughput from Occupation = 1**.

### Why Modern CPUs Are Fast

Modern processors are designed with **high latency but low occupation**.

For FPU, MUL, and DIV units:

- Latency: 5–20 cycles
- Occupation: Often just 1 cycle

This enables massive throughput when instructions are independent.

### The Restaurant Analogy

If the technical explanation feels abstract, consider this analogy:

- **Latency**: The time from ordering food until it arrives at your table—say, 20 minutes. This determines **how long each customer waits**.
- **Occupation**: The time a waiter needs before taking the next order—say, 10 seconds. This determines **how many customers the restaurant can serve**.

A modern CPU is like a restaurant with dozens of waiters and chefs. Each dish (instruction) takes time to prepare (high latency), but the restaurant can accept and serve dozens of orders per minute (high IPC).

### Throughput and Occupation

The mathematical relationship is straightforward:

$$
\text{Throughput} = \frac{1}{\text{Occupation}}
$$

If a unit's occupation is 1 cycle, its peak throughput is **1 instruction/cycle**.
If occupation is 0.5 cycles (dual-issue ALU), throughput is **2 instructions/cycle**.

**Reducing occupation directly raises the throughput ceiling.**

### The One-Liner

> **Latency determines how long you wait.**
> **Occupation determines how fast you go.**

---

## IPC: The Ultimate CPU Performance Metric

### Formal Definition

**IPC (Instructions Per Cycle)** is defined as:

$$
\text{IPC} = \frac{\text{Instructions Retired}}{\text{Cycles Elapsed}}
$$

This is the formal definition from Hennessy & Patterson's *Computer Architecture: A Quantitative Approach*.

### What Determines IPC?

This is the heart of the article. IPC isn't determined by a single factor—it emerges from the interplay of **hardware capabilities and dynamic runtime behavior**.

Think of IPC formation as a **funnel model**:

**1. Frontend — The Funnel's Opening**

Determines how many instructions can enter per cycle (Fetch/Decode Width). If the frontend can only deliver 4 instructions per cycle, nothing downstream matters—IPC is capped at 4. This also involves **Fetch Bandwidth**: if the I-Cache can only deliver 16 bytes per cycle and average instruction length is 4 bytes, IPC is physically limited to 4.

**2. Scheduler — Flow Efficiency in the Middle**

The scheduler must find parallelism among in-flight instructions and keep execution units fed. ROB size determines how far ahead it can "see."

**3. Backend — Processing Capacity at the Bottom**

Execution units (ALU/FPU/LSU) must be plentiful enough to avoid bottlenecks. Throughput here is limited by functional unit count and occupation.

**4. Hazards — Leaks in the Funnel**

Branch mispredictions and cache misses are holes that drain instructions away, preventing them from becoming useful IPC.

The specific factors include:

1. **Frontend fetch/decode width**: How many instructions can be fetched per cycle?
2. **Issue width**: How many instructions can be dispatched to execution units per cycle?
3. **Functional unit count**: How many ALUs, MULs, and LSUs are available?
4. **Dependencies**: Data dependencies between instructions
5. **Memory stalls**: Memory access latency
6. **Branch misprediction**: The cost of wrong predictions
7. **Cache misses**: The penalty for missing in cache
8. **ROB size**: How many instructions can be in-flight?
9. **Scheduling capability**: Dynamic scheduling efficiency

**Occupation is just one small piece**—it only affects the throughput of individual functional units.

### Common Misconception: IPC ≈ Occupation?

Some might think: "Since instructions are pipelined, shouldn't IPC be close to Occupation?"

This is a common and critical misconception.

**The correct understanding**: Occupation limits IPC's ceiling, but IPC rarely equals Occupation—and often falls far short.

**Counterexample 1: Single Functional Unit**

Assume:

- Multiply unit: Latency = 4, Occupation = 1
- But the CPU has **only 1 multiply unit**

Program:

```asm
mul
mul
mul
mul
```

You can issue one per cycle → IPC ≈ 1

**Counterexample 2: Mixed Instructions**

But if the program is:

```asm
mul
add
load
branch
```

These use **different units**, so IPC can exceed 1.

Therefore:
> IPC reflects **all functional units working in parallel**.
> Occupation only describes **a single unit's rhythm**.

**Counterexample 3: Dependency Chains**

With a dependency chain:

```asm
add x1, x2, x3
add x4, x1, x5
add x6, x4, x7
```

Latency = 3 cycles per add.

Even with Occupation = 1, IPC drops to approximately:

$$
\text{IPC} \approx \frac{1}{\text{Latency}} = 0.33
$$

---

## Theoretical IPC Limits

Based on the above analysis, we can derive IPC's theoretical bounds:

$$
\text{IPC} \le \text{Issue Width}
$$

$$
\text{IPC} \le \sum (\text{Functional Unit Throughput})
$$

$$
\text{IPC} \le \frac{1}{\text{Dependency Latency}}
$$

Occupation only appears in the second constraint.

### Why Real IPC Falls Far Below Theory

In real workloads, IPC almost always falls well below theoretical maximums. The culprits:

1. **Branch Misprediction**: Wrong predictions cause pipeline flushes
2. **Cache Misses**: L1/L2/L3 misses cause tens to hundreds of cycles of stalls
3. **Memory Latency**: DRAM access can take 100+ cycles
4. **Data Dependencies**: Real programs are full of dependencies
5. **Instruction Mix**: Not all instructions can execute in parallel

This is why modern CPUs need so many complex mechanisms:

| Mechanism | Problem Solved |
| --------- | -------------- |
| Superscalar | Increase issue width |
| Out-of-Order | Bypass dependency stalls |
| Branch Predictor | Reduce branch misprediction |
| Cache Hierarchy | Reduce memory stalls |
| Prefetcher | Load data ahead of time |
| ROB | Support more in-flight instructions |

**These aren't features—they're remedial mechanisms invented to rescue IPC.**

---

## Deep Dive: The Nine Factors Affecting IPC

Let's examine each factor in detail to understand how they operate in real processors.

### 1. Frontend Fetch/Decode Width

The frontend is the CPU's "entrance," responsible for fetching and decoding instructions. If the frontend can only fetch 2 instructions per cycle, IPC is capped at 2 regardless of backend strength.

**Real-world examples**:

- Intel Core i9-12900K (P-Core): 6-wide decode
- AMD Zen 4: 4-wide decode
- Apple M2 (Avalanche): 8-wide decode

Apple's 8-wide decode is the widest in consumer processors—one reason M-series chips perform so well.

### 2. Issue Width

Issue width determines how many instructions can be dispatched to execution units per cycle. This is typically wider than decode width because the CPU can select ready instructions from the instruction window.

**Design tradeoffs**:

- Wider issue = higher potential IPC
- But also means more complex dependency checking and scheduling logic
- Increased power and area costs

### 3. Functional Unit Count

Even with wide issue, instructions can't execute without sufficient execution units.

**Typical configuration** (Intel Skylake):

- 4 ALUs (integer operations)
- 2 AGUs (address generation)
- 2 Load ports
- 1 Store port
- 2 FPU/SIMD units

The number and types of units determine throughput limits for different instruction types.

### 4. Dependencies

This is one of IPC's biggest killers. When instructions have RAW (Read After Write) dependencies, later instructions must wait for earlier results.

**Three types of dependencies**:

- **RAW (Read After Write)**: True dependency, cannot be eliminated
- **WAR (Write After Read)**: Can be eliminated via register renaming
- **WAW (Write After Write)**: Can be eliminated via register renaming

Out-of-order execution and register renaming were invented specifically to work around these dependencies.

### 5. Memory Stalls

When the CPU needs memory access and data isn't in cache, massive latency ensues.

**Latency comparison**:

| Level | Latency (cycles) | Latency (ns) |
| ----- | ---------------- | ------------ |
| L1 Cache | 4-5 | ~1 |
| L2 Cache | 12-14 | ~3-4 |
| L3 Cache | 40-50 | ~10-15 |
| DRAM | 200-300 | ~50-100 |

A single DRAM miss can idle the CPU for 200+ cycles—catastrophic for IPC.

### 6. Branch Misprediction

Modern CPUs use speculative execution: they guess branch outcomes and execute ahead. Wrong guesses mean all speculative work gets discarded.

**The cost**:

- Typical misprediction penalty: 15-20 cycles
- If 10% of instructions are branches with 5% misprediction rate
- That's 0.5 mispredictions per 100 instructions
- Average loss: 0.5 × 17 / 100 ≈ 0.085 cycles per instruction

This seems small, but it accumulates to significantly impact IPC.

### 7. Cache Misses

Cache miss impact was covered under Memory Stalls, but cache design deserves emphasis:

- **L1 Cache**: Optimized for lowest latency, typically 32-64 KB
- **L2 Cache**: Balances latency and capacity, typically 256 KB - 1 MB
- **L3 Cache**: Optimized for hit rate, typically 8-64 MB

Cache design directly affects memory stall frequency, which directly affects IPC.

### 8. ROB (Reorder Buffer) Size

The ROB is the core structure enabling out-of-order execution. It tracks all in-flight instructions and ensures they retire in program order.

**ROB size impact**:

- Larger ROB = more in-flight instructions
- Can "see further ahead" to find more parallelism
- But means higher power and area costs

**Real-world examples**:

- Intel Skylake: 224 entries
- Intel Golden Cove (Alder Lake P-Core): 512 entries
- AMD Zen 4: 320 entries

#### Little's Law: The Formula Connecting Latency and ROB

Queuing theory gives us a famous formula—**Little's Law**. In CPU architecture, it can be expressed as:

$$
\text{In-Flight Instructions} = \text{IPC} \times \text{Latency}
$$

This formula reveals a harsh truth: **if you want to maintain high IPC despite high latency, your only option is to linearly scale your buffer (ROB).**

For example, assume:

- Target IPC = 6
- Memory Latency = 100 cycles

Required in-flight instructions = 6 × 100 = **600 instructions**.

This perfectly explains why Apple's M-series needs 600+ entry ROBs—facing DRAM's long latency, maintaining high IPC requires a massive ROB. This isn't luxury; it's mathematical necessity.

### 9. Scheduling Capability

The scheduler selects ready instructions from the instruction window and dispatches them to execution units. A good scheduler can:

- Quickly identify ready instructions
- Balance load across execution units
- Prioritize critical-path instructions

Scheduling efficiency directly impacts IPC, but this is an NP-hard problem. Modern CPUs use various heuristics to approximate optimal solutions.

---

## Real-World Case Studies: Viewing Processor Design Through IPC

Let's analyze several real processor design decisions through the lens of IPC.

### Case 1: Why Are Apple M-Series Chips So Fast?

Apple's M1/M2/M3 chips deliver stunning single-threaded performance. The reasons:

1. **Ultra-wide Frontend**: 8-wide decode—the widest in the industry
2. **Massive ROB**: Over 600 entries
3. **Huge L2 Cache**: 16-24 MB per P-Core
4. **Aggressive Branch Predictor**: Extremely low misprediction rate

Every design choice maximizes IPC. Apple willingly pays the power and area costs for higher single-threaded performance.

### Case 2: Why Do Intel and AMD Choose Different L3 Cache Strategies?

- **Intel**: Smaller L3 (~30 MB) with lower latency
- **AMD**: Massive L3 (64-96 MB) with slightly higher latency

This reflects different design philosophies:

- Intel believes lower latency matters more—reducing memory stalls
- AMD believes higher hit rate matters more—avoiding DRAM accesses entirely

Both strategies aim to improve IPC; they just make different tradeoffs.

### Case 3: Why Do RISC-V Processors Typically Have Lower IPC?

Most current RISC-V processors have lower IPC than x86 and ARM. The reasons:

1. **Narrower Frontend**: Typically 2-4 wide
2. **Smaller ROB**: Typically 64-128 entries
3. **Simpler Branch Predictor**: Lower prediction accuracy
4. **Smaller Caches**: Limited by cost and power budgets

This isn't a RISC-V ISA problem—it's an implementation maturity issue. As more resources flow into RISC-V development, IPC will improve.

### Case 4: The Historical Evolution of IPC

Looking back at CPU history, we can see how IPC has driven the entire industry:

**1980s — Early RISC**:

- Goal: One instruction per cycle (IPC = 1)
- Simple 5-stage pipeline
- In-order execution

**1990s — Superscalar Era**:

- Goal: Multiple instructions per cycle (IPC > 1)
- Multiple execution units
- Out-of-order execution emerges

**2000s — The Frequency Wall**:

- Intel Pentium 4's lesson: High frequency, low IPC
- Industry pivots back to IPC optimization
- Multi-core becomes mainstream

**2010s — Present**:

- Extreme IPC optimization (Apple M-series)
- Heterogeneous computing (Big.LITTLE)
- Specialized accelerators (GPU, NPU)

### Case 5: GPU vs. CPU — Different IPC Philosophies

GPUs and CPUs take completely different approaches to IPC:

| Aspect | CPU | GPU |
| ------ | --- | --- |
| Single-thread IPC | High (2-6) | Low (~1) |
| Thread count | Few (8-32) | Massive (thousands) |
| Latency hiding | Hardware (OoO, speculation) | Software (thread switching) |
| Target workload | Latency-sensitive | Throughput-oriented |

GPUs don't need high single-thread IPC because they hide latency through massive parallelism. This is a fundamentally different design philosophy.

---

## Practical Application: Measuring and Optimizing IPC

### Top-Down Microarchitecture Analysis (TMAM)

Intel's Top-Down Microarchitecture Analysis Method is a systematic approach to identifying IPC bottlenecks. It categorizes pipeline slots into four categories:

1. **Retiring**: Slots that successfully retired instructions (good)
2. **Bad Speculation**: Slots wasted on mispredicted paths
3. **Frontend Bound**: Slots starved due to frontend limitations
4. **Backend Bound**: Slots stalled due to backend limitations

This framework helps pinpoint exactly where IPC is being lost.

### Performance Monitoring Counters (PMC)

Modern CPUs provide hardware counters for measuring IPC-related metrics:

| Counter | Meaning | Related Factor |
| ------- | ------- | -------------- |
| INST_RETIRED | Instructions retired | IPC numerator |
| CPU_CLK_UNHALTED | Cycles elapsed | IPC denominator |
| BR_MISP_RETIRED | Branch mispredictions | Branch misprediction |
| L1D_MISS | L1 Data Cache misses | Cache miss |
| L2_MISS | L2 Cache misses | Cache miss |
| RESOURCE_STALLS | Resource-induced stalls | Backend bound |

**Calculating IPC**:

```bash
# Using perf to calculate IPC
perf stat -e instructions,cycles ./your_program

# Example output:
# 1,234,567,890 instructions
#   987,654,321 cycles
# IPC = 1,234,567,890 / 987,654,321 = 1.25
```

### Advice for Software Engineers

While IPC is primarily a hardware metric, software engineers can improve their programs' IPC:

1. **Reduce Branch Misprediction**

   - Use likely/unlikely hints
   - Avoid unpredictable branches (data-dependent branches)
   - Consider branchless programming

   **Example: Branchless min/max (RISC-V Assembly Perspective)**

   Let's use RISC-V Assembly to show how "control dependency" transforms into "data dependency":

   ```asm
   # Branch version
   min_branch:
       blt  a0, a1, .L_return_a  # Branch if Less Than
       mv   a0, a1               # Otherwise, move b to return value
   .L_return_a:
       ret

   # Branchless version
   min_branchless:
       slt  t0, a0, a1    # Set Less Than: t0 = 1 if a0 < a1, else 0
       sub  t0, zero, t0  # t0 = -t0 (1 becomes -1/0xFFFFFFFF, 0 stays 0)
       xor  t1, a0, a1    # t1 = a ^ b
       and  t1, t1, t0    # t1 = (a ^ b) & mask
       xor  a0, a1, t1    # Result = b ^ ((a ^ b) & mask)
       ret
   ```

   **The Highway Analogy**:

   - **Branch version**: Like hitting a fork in the highway. If your GPS (Branch Predictor) picks wrong, you must backtrack—wasting significant time.
   - **Branchless version**: A straight road. Slightly longer (more instructions), but you can floor it (no pipeline interruption), arriving faster overall.

   **Key insight**: Branchless converts **control flow** to **data flow**. More instructions, but all simple ALU operations with no prediction risk. The CPU can fully fill the pipeline, achieving stable IPC.

2. **Improve Cache Hit Rate**

   - Optimize data structure memory layout
   - Use cache-friendly access patterns
   - Consider prefetching

3. **Reduce Dependency Chains**

   - Loop unrolling
   - Use SIMD instructions
   - Reorder computations

   **Example: Loop Unrolling**

   ```c
   // Original version (long dependency chain)
   int sum = 0;
   for (int i = 0; i < n; i++) {
       sum += arr[i];  // Each iteration depends on previous sum
   }

   // Unrolled version (multiple independent accumulators)
   int sum0 = 0, sum1 = 0, sum2 = 0, sum3 = 0;
   for (int i = 0; i < n; i += 4) {
       sum0 += arr[i];
       sum1 += arr[i+1];
       sum2 += arr[i+2];
       sum3 += arr[i+3];
   }
   int sum = sum0 + sum1 + sum2 + sum3;
   ```

   The second version lets the CPU execute multiple independent additions simultaneously, dramatically improving IPC.

   **Modern Perspective**: On powerful OoO processors (Apple M-series, Intel Golden Cove), hardware **Register Renaming** already handles false dependencies (WAW/WAR) within loops. Today, loop unrolling's main benefits are:

   - **Reducing branch instruction density**: Less frontend pressure
   - **Enabling compiler vectorization (SIMD)**
   - **Expanding the instruction window**: Letting the scheduler see further ahead

   ⚠️ **Warning**: Excessive unrolling increases code size, potentially hurting I-Cache and reducing IPC. This is a tradeoff requiring careful tuning.

4. **Exploit Instruction-Level Parallelism**

   - Avoid long dependency chains
   - Give the compiler optimization room
   - Use the `restrict` keyword to eliminate aliasing

### Advice for Hardware Engineers

If you're a CPU designer, here are directions for improving IPC:

1. **Frontend Optimization**

   - Increase fetch/decode width
   - Improve branch predictor (larger BTB, better algorithms)
   - Increase I-Cache capacity or reduce latency

2. **Backend Optimization**

   - Add more execution units
   - Increase ROB size
   - Improve scheduler algorithms

3. **Memory Optimization**

   - Increase cache capacity
   - Reduce cache latency
   - Improve prefetcher

4. **Tradeoff Considerations**

   - Power vs. performance
   - Area vs. performance
   - Single-thread IPC vs. multi-thread throughput

---

## Common Misconceptions

When learning about IPC, several misconceptions are worth addressing:

### Misconception 1: Higher Frequency = Better Performance

This was the lesson of Intel's Pentium 4 in the early 2000s. The P4 used a 31-stage deep pipeline to reach 3.8 GHz, but its high branch misprediction penalty meant actual performance lagged behind lower-frequency competitors.

**Correct understanding**: Performance = IPC × Frequency. High frequency with low IPC may underperform low frequency with high IPC.

### Misconception 2: More Cores = Better Performance

More cores certainly improve multi-threaded throughput, but for single-threaded applications, core count is irrelevant. Many everyday applications (web browsing, office software) depend primarily on single-thread IPC.

**Correct understanding**: Choose based on workload. CPU-bound, highly parallel tasks benefit from more cores; but many tasks remain single-thread bottlenecked.

### Misconception 3: RISC Has Higher IPC Than CISC

This is a long-standing myth. Modern x86 processors internally convert CISC instructions into RISC-like micro-ops, then execute them with efficient OoO engines. Intel and AMD's high-end processors match ARM in IPC.

**Correct understanding**: ISA's impact on IPC is far smaller than microarchitecture design. Good microarchitecture achieves high IPC on any ISA.

### Misconception 4: Higher IPC Is Always Better

While high IPC usually means high performance, other metrics may matter more in certain scenarios:

- **Embedded systems**: Power and area may matter more than IPC
- **Servers**: Total throughput may matter more than single-thread IPC
- **GPUs**: Massive threading hides latency; high single-thread IPC isn't needed

**Correct understanding**: IPC is important, but must be weighed against application requirements.

### Misconception 5: Maximizing IPC Is Always the Right Goal

This misconception requires deeper discussion of the **IPC vs. Frequency tradeoff**.

**The core formula**:

$$
\text{Performance} = \text{IPC} \times \text{Frequency}
$$

Pursuing extreme IPC often means more complex logic (wider issue, larger ROB, smarter scheduler). This complexity increases critical path length, **limiting maximum frequency**.

**Different design philosophies**:

- **Apple M-series**: Chose ultra-wide architecture (high IPC), with conservative frequency (~3.5-4.0 GHz)
- **Intel Core (Raptor Lake)**: Optimized pipeline stages and circuit design to hit 6.0 GHz, sometimes sacrificing IPC

**Correct understanding**: IPC isn't always better—**IPC × Frequency** is what matters.

### The Power Wall and Big.LITTLE

When discussing IPC, we can't ignore **power consumption** as a hidden constraint.

Complex OoO mechanisms (massive ROB, sophisticated scheduler) consume significant power. This is why **Big.LITTLE** (or P-Core + E-Core) design philosophy exists:

- **P-Core (Performance Core)**: Pursues maximum IPC, higher power consumption
- **E-Core (Efficiency Core)**: Sacrifices some IPC and latency for extreme energy efficiency

E-Cores have narrower backends and smaller ROBs, but handle background tasks at minimal power. This is part of the IPC design spectrum—not every scenario needs maximum IPC.

**A new metric**: Energy per Instruction (EPI) is becoming as important as IPC.

---

## IPC Dynamics: It's Not a Static Number

Before concluding, I want to emphasize a point many overlook: **IPC is not a constant**.

Program execution breathes. Sometimes it's in a "compute-intensive phase" with high IPC; sometimes it enters a "memory-intensive phase" where IPC plummets.

**High-IPC Phase (Compute-Intensive)**:

- Data resides in L1 Cache
- Low inter-instruction dependencies
- Accurate branch prediction
- IPC approaches theoretical maximum

**Low-IPC Phase (Memory-Intensive)**:

- Frequent cache misses
- Pointer chasing (linked list traversal)
- Irregular memory access patterns
- IPC may drop below 0.5

**The architect's goal**:

- Minimize Low-IPC time (through larger caches, better prefetching)
- Raise the High-IPC ceiling (through wider issue width, larger ROB)

Understanding this dynamism helps you analyze performance problems without fixating on a single "average IPC" number—instead, you'll understand program behavior across different phases.

---

## Conclusion

### Key Takeaways

1. **Latency** determines "how long you wait"; **Occupation** determines "how fast you go"
2. **IPC** is the ultimate CPU performance metric, emerging from multiple interacting factors
3. **Superscalar, OoO, Branch Predictor, Cache** aren't independent features—they're "remedial mechanisms invented to rescue IPC"
4. Understanding IPC elevates you from "fragmented knowledge" to "architect's mindset"

### In One Sentence

> **All roads lead to IPC.**
> Every CPU microarchitecture design ultimately serves this single metric.

---

## References

1. Hennessy, J. L., & Patterson, D. A. (2017). *Computer Architecture: A Quantitative Approach* (6th Edition). Morgan Kaufmann.
2. Patterson, D. A., & Hennessy, J. L. (2020). *Computer Organization and Design: The Hardware/Software Interface* (RISC-V Edition). Morgan Kaufmann.
3. Shen, J. P., & Lipasti, M. H. (2013). *Modern Processor Design: Fundamentals of Superscalar Processors*. Waveland Press.
4. Wall, D. W. (1991). "Limits of Instruction-Level Parallelism." *Proceedings of the Fourth International Conference on Architectural Support for Programming Languages and Operating Systems (ASPLOS IV)*.
5. Tomasulo, R. M. (1967). "An Efficient Algorithm for Exploiting Multiple Arithmetic Units." *IBM Journal of Research and Development*, 11(1), 25-33.

---

## License

This article is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

**Source**: <https://github.com/djiangtw/tech-column-public>
