# RISC-V Microarchitecture: From Minimal In-Order Cores to High-Performance Out-of-Order Systems

**Pipelining · Superscalar Execution · Out-of-Order Execution · ILP · Microarchitectural Trade-offs · Performance/Efficiency Optimization · Execution Paradigms**

---

## 0. What This Study Is Actually About

This study is not simply a comparison of ten RISC-V processors.

The ten processors are examples used to understand a much bigger question:

> How does a processor extract more performance from the same instruction stream, and what hardware cost does that performance require?

The central microarchitectural progression is:

```
Simple sequential execution
        │
        ▼
Multi-cycle execution
        │
        ▼
Pipelining
        │
        ▼
Deeper pipelining
        │
        ▼
Superscalar execution
        │
        ▼
Out-of-order execution
        │
        ▼
Speculation + register renaming
        │
        ▼
Large-scale Instruction-Level Parallelism
        │
        ▼
High-performance OoO architectures
```

At every step, we gain performance by introducing more hardware complexity.

That creates the fundamental engineering problem:

> How much performance should we buy, and how much area, power, complexity, latency, verification effort, and design risk should we pay for it?

This is the central theme of the entire analysis.

---

## 1. The Ten RISC-V Cores

| Core | Execution Style | Pipeline | Issue Width | OoO? | Main Lesson |
|---|---|---|---|---|---|
| SERV | Bit-serial | Temporal / non-classical | 1 | No | Extreme area efficiency |
| PicoRV32 | Multi-cycle | FSM-based | 1 | No | Simplicity and configurability |
| Ibex | In-order | 2-stage | 1 | No | Small secure embedded core |
| CV32E40P | In-order | 4-stage | 1 | No | Efficient embedded pipeline |
| SCR1 | In-order | 4-stage | 1 | No | Compact MCU-class implementation |
| VexRiscv | In-order | Configurable, commonly 5-stage | 1 | No | FPGA-oriented configurability |
| Rocket | In-order | ~5-stage | 1 | No | Scalable application-class baseline |
| Shakti C-Class | In-order | ~5–6 stage | 1 | No | Production-oriented 64-bit RISC-V |
| BOOM / SonicBOOM | Superscalar OoO | Deep / configurable | 2–4 | Yes | Dynamic ILP extraction |
| XiangShan | Wide OoO | Deep | Multi-wide | Yes | High-performance RISC-V |

The progression is therefore approximately:

```
                    PERFORMANCE
                         ▲
                         │
                         │                       XiangShan
                         │                    BOOM
                         │
                         │              Rocket / Shakti
                         │
                         │         VexRiscv
                         │
                         │     CV32E40P / SCR1
                         │
                         │  Ibex
                         │
                         │ PicoRV32
                         │
                         │ SERV
                         └──────────────────────────────►
                              HARDWARE COMPLEXITY
```

This graph is conceptual rather than a benchmark ranking.

The important point is that microarchitecture is a design-space problem, not a simple "newer = better" hierarchy.

---

## 2. The Five Questions We Must Ask About Every Core

For every one of the ten cores, we should ask:

### Question 1 — How does it execute instructions?

This gives us the execution paradigm.

Examples:

- bit-serial
- multi-cycle
- in-order pipeline
- superscalar in-order
- out-of-order
- speculative execution

---

### Question 2 — How much work can happen simultaneously?

This is primarily about:

> **Instruction-Level Parallelism (ILP)**

A processor does not necessarily need to execute instructions one after another.

Consider:

```
ADD x5, x1, x2
ADD x6, x3, x4
MUL x7, x8, x9
```

These instructions are independent.

A processor capable of exploiting ILP could execute multiple operations concurrently.

---

### Question 3 — What prevents more parallelism?

Possible limitations include:

- data dependencies
- branches
- cache misses
- limited functional units
- register-file port limitations
- memory dependencies
- pipeline hazards
- insufficient instruction fetch bandwidth
- limited issue width

---

### Question 4 — What hardware was added to overcome those limitations?

For example:

```
Data dependency
      ↓
Forwarding
      ↓
Still insufficient
      ↓
Register renaming
      ↓
Out-of-order execution
      ↓
Issue queues
      ↓
ROB
      ↓
Speculation
```

---

### Question 5 — What did that hardware cost?

Every optimization has a price:

| Optimization | Benefit | Cost |
|---|---|---|
| Pipeline | Higher clock frequency | Hazards |
| Deeper pipeline | Higher Fmax | Larger branch penalty |
| Superscalar | Higher peak IPC | More hardware |
| OoO | Latency hiding | Huge complexity |
| Register renaming | Removes false dependencies | Large PRF/RAT |
| Branch prediction | Keeps pipeline full | Predictor area/power |
| Bigger cache | Fewer misses | Area + power + latency |
| Wider issue | More ILP | Huge datapath complexity |
| Speculation | Better utilization | Recovery machinery |

That is microarchitectural trade-off analysis.

---

## 3. First Principle: ISA ≠ Microarchitecture

This distinction is critical.

The ISA describes:

> What the processor must do.

Microarchitecture describes:

> How the processor actually does it.

For example:

```
ADD x5, x1, x2
```

At the ISA level:

```
x5 = x1 + x2
```

But the hardware implementation could be:

**SERV**

```
bit 0 → add
bit 1 → add
...
bit 31 → add
```

over many cycles.

**3-stage core**

```
Fetch → Execute → Writeback
```

**5-stage pipeline**

```
IF → ID → EX → MEM → WB
```

**Superscalar**

```
        ┌→ ALU0 ─┐
Decode ─┤        ├→ Commit
        └→ ALU1 ─┘
```

**Out-of-order**

```
Fetch
  ↓
Decode
  ↓
Rename
  ↓
Dispatch
  ↓
Issue Queue
  ↓
Execute when operands become ready
  ↓
Writeback
  ↓
ROB
  ↓
Commit in program order
```

Same ISA.

Completely different machine.

---

## 4. Execution Paradigms

This is one of the most important sections of the project.

An execution paradigm is essentially the fundamental strategy used by the processor to decide:

> When and where should an instruction execute?

---

### 4.1 Bit-Serial Execution — SERV

SERV takes the extreme approach.

Instead of:

```
32-bit + 32-bit
```

it effectively performs:

```
bit0 + bit0
bit1 + bit1
...
bit31 + bit31
```

The hardware is tiny.

The computation takes many cycles.

This is:

> Temporal reuse instead of spatial parallelism.

Instead of building:

```
32 adders
```

the architecture reuses:

```
1 small adder
```

over time.

**Fundamental trade-off**

```
Area ↓↓↓
Power/logic footprint ↓↓↓

Performance ↓↓↓
```

SERV therefore teaches the first major lesson:

> Parallel hardware buys performance by spending physical resources.

---

## 5. Multi-Cycle Execution — PicoRV32

PicoRV32 takes a different approach.

Instead of making the datapath extremely wide or building a pipeline, it uses a control FSM to execute instructions over multiple cycles.

Conceptually:

```
FETCH
  ↓
DECODE
  ↓
EXECUTE
  ↓
MEMORY
  ↓
WRITEBACK
  ↓
NEXT INSTRUCTION
```

But these aren't necessarily separate pipeline stages.

They are states of a controller.

This distinction matters.

**Pipelined processor**

Several instructions can occupy different stages simultaneously:

```
Cycle 1: I1 IF
Cycle 2: I1 ID | I2 IF
Cycle 3: I1 EX | I2 ID | I3 IF
```

**Multi-cycle processor**

Usually:

```
Cycle 1: I1 fetch
Cycle 2: I1 decode
Cycle 3: I1 execute
Cycle 4: I1 memory
Cycle 5: I1 writeback

Cycle 6: I2 fetch
```

The hardware is reused rather than duplicated.

Again:

> Temporal reuse reduces area but reduces instruction throughput.

---

## 6. Pipelining

Now we reach the first major topic of the project.

What is pipelining?

Suppose an instruction requires five logical operations:

```
Fetch
Decode
Execute
Memory
Writeback
```

Without pipelining:

```
I1: IF → ID → EX → MEM → WB
I2:                    IF → ID → EX → MEM → WB
```

Only one instruction is being processed at a time.

With pipelining:

```
Cycle       1    2    3    4    5    6    7
------------------------------------------------
I1         IF   ID   EX   MEM   WB
I2              IF   ID   EX   MEM   WB
I3                   IF   ID   EX   MEM   WB
I4                        IF   ID   EX   MEM   WB
```

Now multiple instructions are simultaneously inside the processor.

That is the key idea.

---

## 7. Latency vs Throughput

This distinction is absolutely essential.

**Latency**

How long does one instruction take from start to finish?

**Throughput**

How frequently can completed instructions emerge?

A pipeline does not necessarily make one instruction complete dramatically faster.

Instead, it allows many instructions to overlap.

Example:

Unpipelined:

```
I1 ────────────────
I2                  ────────────────
I3                                  ────────────────
```

Pipelined:

```
I1 ─────
I2    ─────
I3        ─────
I4            ─────
I5                ─────
```

The pipeline increases throughput.

---

## 8. Pipeline Depth

Consider:

3-stage:

```
IF → EX → WB
```

versus:

5-stage:

```
IF → ID → EX → MEM → WB
```

versus:

9-stage:

```
IF1 → IF2 → IF3 → ID → EX1 → EX2 → EX3 → EX4 → WB
```

Why create more stages?

Because each stage contains less combinational logic.

That can allow:

> Higher clock frequency.

If a critical computation takes 10 ns and is divided into five roughly equal stages:

```
~2 ns/stage
```

the theoretical clock period can become much smaller.

But deeper pipelines introduce problems.

---

## 9. The Cost of Deep Pipelines

Suppose a branch is resolved here:

```
IF → ID → EX → MEM → WB
          ↑
       branch
```

If the branch predictor was wrong, instructions already fetched after the branch must be discarded.

That is a:

> Pipeline flush.

A deeper pipeline means more instructions may be in flight behind the branch.

Therefore:

```
Pipeline depth ↑
       ↓
Branch resolution later
       ↓
Mispredict penalty ↑
```

This is why SweRV's deep pipeline and high frequency come with a larger branch penalty than a tiny 2–3 stage core.

---

## 10. Pipeline Hazards

A pipeline creates situations where instructions cannot safely proceed.

These are called hazards.

Three classical classes:

1. Structural hazards
2. Data hazards
3. Control hazards

---

### 10.1 Structural Hazard

Two instructions require the same hardware resource.

Example:

```
Instruction 1 → memory
Instruction 2 → memory
```

But there is only one memory port.

The processor must choose.

Possible solutions:

- stall
- duplicate hardware
- separate instruction/data memories
- increase ports
- cache hierarchy

---

## 11. Data Hazards

Consider:

```
ADD x5, x1, x2
SUB x6, x5, x3
```

The second instruction needs the result of the first.

This is:

> **RAW — Read After Write**

The second instruction wants to read `x5` before the first instruction has architecturally written it.

---

### 11.1 Forwarding

Instead of waiting for writeback:

```
ALU result
    │
    ├──────────────► next ALU input
    │
    └──────────────► register file
```

This is called:

> **Forwarding / bypassing.**

It reduces stalls.

This mechanism appears heavily in:

- Ibex
- CV32E40P
- SCR1
- VexRiscv
- Rocket
- Shakti
- SweRV

---

## 12. Load-Use Hazard

Forwarding cannot always solve everything.

Example:

```
LW  x5, 0(x1)
ADD x6, x5, x2
```

The load needs to access memory before the data exists.

So:

```
Load
 ↓
Memory access
 ↓
Data available
 ↓
Dependent ADD
```

The ADD may have to wait.

This produces a:

> **Load-use stall.**

This is an excellent example of why simply adding forwarding does not eliminate every hazard.

---

## 13. Control Hazards

Consider:

```
BEQ x1, x2, target
```

The processor doesn't immediately know whether the branch is taken.

Meanwhile, the front-end continues fetching.

If the prediction is wrong:

```
wrong-path instructions
        ↓
       FLUSH
        ↓
correct target
```

This leads to:

> **Branch prediction.**

---

## 14. Branch Prediction

Modern processors try to answer:

> "Where will execution go next?"

before the branch is actually resolved.

Important structures include:

**BTB**

Branch Target Buffer

Stores predicted target addresses.

**BHT**

Branch History Table

Stores information about whether branches tend to be taken or not taken.

**RAS**

Return Address Stack

Predicts function returns.

**TAGE**

A sophisticated predictor using multiple history lengths.

BOOM and high-performance cores use substantially more sophisticated prediction machinery than tiny embedded cores.

---

## 15. Superscalar Execution

Now we move beyond ordinary pipelining.

A scalar processor can theoretically issue:

```
1 instruction / cycle
```

A 2-wide superscalar processor can theoretically issue:

```
2 instructions / cycle
```

A 4-wide processor:

```
4 instructions / cycle
```

This is issue width.

---

## 16. Important: Width ≠ Actual IPC

Suppose:

```
4-wide processor
```

It does NOT mean:

```
IPC = 4
```

It means:

> The machine has the hardware capacity to issue up to four instructions per cycle.

Actual IPC depends on available ILP.

For example:

```
4-wide machine

Cycle 1:
I1 independent
I2 independent
I3 independent
I4 dependent

→ perhaps only 3 useful instructions issue
```

Real workloads may achieve much less.

Therefore:

```
Peak IPC ≠ Sustained IPC
```

---

## 17. Instruction-Level Parallelism — ILP

ILP is the amount of independent instruction work available in a program.

Consider:

```
ADD x5, x1, x2
ADD x6, x3, x4
MUL x7, x8, x9
SUB x10, x11, x12
```

These instructions are mostly independent.

There is significant ILP.

Now:

```
ADD x1, x1, x2
ADD x1, x1, x3
ADD x1, x1, x4
ADD x1, x1, x5
```

Every instruction depends on the previous result.

There is very little ILP.

So a 4-wide processor cannot magically turn this into 4 IPC.

---

## 18. The ILP Wall

As processors become wider:

```
1-wide
  ↓
2-wide
  ↓
4-wide
  ↓
6-wide
  ↓
8-wide
```

finding enough independent instructions becomes increasingly difficult.

Therefore:

> Hardware width eventually exceeds available program-level parallelism.

This is one reason simply making processors wider gives diminishing returns.

---

## 19. In-Order Superscalar — SweRV

SweRV EH1 demonstrates an important middle ground.

It is:

> 2-wide superscalar but in-order.

So it can execute:

```
I0 + I1
```

in the same cycle when allowed.

But instructions remain constrained by program ordering and dependency rules.

This is much simpler than full OoO.

The design philosophy is:

```
More parallelism
       +
Limited complexity
       =
Efficient superscalar embedded processor
```

This is extremely important for your microarchitecture trade-off analysis.

---

## 20. Out-of-Order Execution

Now we make the major architectural jump.

Imagine:

```
I1: LOAD
I2: ADD dependent on I1
I3: ADD independent
I4: MUL independent
```

An in-order processor may encounter:

```
I1 waiting for memory
I2 waiting for I1
I3 waiting
I4 waiting
```

An OoO processor asks:

> "Why wait for I2 if I3 and I4 are ready?"

So:

```
I1 → waiting for memory

I2 → waiting for I1

I3 → EXECUTE NOW
I4 → EXECUTE NOW
```

This is the essence of OoO execution.

---

## 21. Register Renaming

Out-of-order processors encounter another problem.

Suppose:

```
ADD x1, x2, x3
ADD x1, x4, x5
```

Both instructions write architectural register `x1`.

The register name itself creates an apparent dependency.

Register renaming replaces:

```
x1
```

with physical registers:

```
x1 → P17
x1 → P23
```

Now the machine can track the actual values separately.

This eliminates:

- WAR — Write After Read
- WAW — Write After Write

These are called:

> False dependencies / name dependencies.

RAW remains a true dependency because the actual value is required.

---

## 22. Physical Register File

An OoO processor therefore needs more registers internally than the ISA exposes.

RISC-V may expose:

```
32 architectural integer registers
```

But internally:

```
x1 → P12
x2 → P31
x3 → P07
...
```

There may be dozens or more physical registers.

This enables multiple versions of the same architectural register to exist simultaneously.

---

## 23. Issue Queue

After renaming, instructions wait in an:

> **Issue Queue**

An instruction can execute when:

```
Operand A ready
AND
Operand B ready
AND
Execution unit available
```

The scheduler performs:

```
Wakeup
   ↓
Select
   ↓
Issue
```

This is often called:

> Wakeup-and-select logic.

It is one of the major complexity sources in OoO processors.

---

## 24. Why "Out of Order" Doesn't Mean "Retire Out of Order"

This distinction is extremely important.

Suppose:

Program order:

```
I1
I2
I3
I4
```

Execution could happen:

```
I1
I3
I4
I2
```

But architectural commitment generally happens:

```
I1
I2
I3
I4
```

Why?

Because the processor must maintain:

> Precise architectural state.

This is the job of the:

**Reorder Buffer — ROB**

The ROB tracks instructions in program order.

---

## 25. Reorder Buffer

Conceptually:

```
                 Reorder Buffer

Head                              Tail
 │                                  │
 ▼                                  ▼
┌────┬────┬────┬────┬────┬────┬────┐
│ I1 │ I2 │ I3 │ I4 │ I5 │ I6 │ I7 │
└────┴────┴────┴────┴────┴────┴────┘
  ↑
commit next
```

An instruction may execute early.

But it cannot architecturally retire until all older instructions are ready.

This allows:

> Out-of-order execution + in-order commitment.

That combination is fundamental to modern OoO CPUs.

---

## 26. BOOM — The OoO RISC-V Case Study

BOOM is therefore not simply:

> "Rocket but faster."

It represents a different execution philosophy.

Rocket:

```
Fetch
 ↓
Decode
 ↓
Execute
 ↓
Memory
 ↓
Writeback
```

BOOM:

```
Fetch
 ↓
Decode
 ↓
Rename
 ↓
Dispatch
 ↓
Issue Queue
 ↓
Execute whenever ready
 ↓
Writeback
 ↓
ROB
 ↓
Commit
```

That extra machinery exists primarily to exploit dynamic ILP.

---

## 27. XiangShan — Scaling the Same Philosophy

XiangShan pushes the same basic ideas much further:

```
Wide front-end
       +
Sophisticated prediction
       +
Register renaming
       +
Large scheduling structures
       +
Multiple execution units
       +
Aggressive memory system
       +
Speculation
       +
Large OoO window
```

The central question becomes:

> How much useful work can we keep in flight simultaneously?

---

## 28. The Evolution of ILP Extraction

The ten cores form a useful conceptual ladder:

```
SERV
 │
 │ Temporal reuse
 ▼
PicoRV32
 │
 │ Multi-cycle control
 ▼
Ibex / SCR1 / CV32E40P
 │
 │ Pipelining
 ▼
VexRiscv / Rocket / Shakti
 │
 │ More pipeline sophistication
 ▼
SweRV
 │
 │ Superscalar issue
 ▼
BOOM
 │
 │ Dynamic scheduling + OoO
 ▼
XiangShan
 │
 │ Aggressive wide OoO
 ▼
High-performance ILP machine
```

---

## 29. Microarchitectural Trade-Offs

This is the heart of the project.

There is no universally "best" microarchitecture.

Instead:

```
Performance
     ▲
     │
     │       High-performance OoO
     │
     │     Superscalar
     │
     │   Pipeline
     │
     │ Multi-cycle
     │
     └──────────────────────►
       Area / Power / Complexity
```

---

## 30. Trade-Off #1 — Pipeline Depth

**Shallow pipeline**

Advantages:

- lower branch penalty
- simpler control
- lower speculation cost
- often better latency

Disadvantages:

- less frequency headroom
- more logic per stage

**Deep pipeline**

Advantages:

- higher potential Fmax
- smaller logic per stage

Disadvantages:

- branch penalty increases
- more pipeline registers
- more control complexity
- greater clock power
- more difficult verification

---

## 31. Trade-Off #2 — Issue Width

Increasing:

```
1-wide → 2-wide → 4-wide
```

increases potential throughput.

But it also increases:

- register-file ports
- bypass paths
- decode complexity
- instruction scheduling complexity
- execution-unit count
- power
- routing
- verification

The number of interactions can grow rapidly.

---

## 32. Trade-Off #3 — In-Order vs OoO

**In-order**

```
Simple
Predictable
Low area
Low power
Easy verification
```

But:

```
Poor latency hiding
Lower ILP extraction
Stalls can propagate
```

**Out-of-order**

```
High ILP
Excellent latency hiding
Better utilization of execution units
```

But:

```
Large ROB
Large PRF
Issue queues
Renaming
Complex recovery
High power
Complex verification
```

---

## 33. Trade-Off #4 — Branch Prediction

Without prediction:

```
Branch → wait → resolve → fetch target
```

With prediction:

```
Branch
 ↓
Predict target
 ↓
Continue fetching
```

Good prediction:

```
Performance ↑
```

Bad prediction:

```
Flush
+
Lost cycles
+
Energy wasted
```

Therefore predictor complexity must be justified by workload behavior.

---

## 34. Trade-Off #5 — Memory System

CPU execution speed increasingly depends on memory.

Typical hierarchy:

```
Registers
   ↓
L1
   ↓
L2
   ↓
L3
   ↓
DRAM
```

The further down the hierarchy we go:

```
Capacity ↑
Latency ↑
Bandwidth characteristics change
Energy/access ↑
```

A modern OoO core therefore needs to hide memory latency.

That requires:

- non-blocking caches
- multiple outstanding misses
- load queues
- store queues
- memory dependency prediction/disambiguation
- prefetching
- speculative loads

---

## 35. Performance-Efficiency Optimization

Performance is not simply:

> "Maximum clock frequency."

A better model is:

$$
Performance \approx IPC \times Frequency
$$

where:

- **IPC** = instructions per cycle
- **Frequency** = cycles per second

Therefore:

```
High Fmax + terrible IPC
```

may be worse than:

```
Moderate Fmax + excellent IPC
```

---

## 36. CPI and IPC

Two equivalent perspectives:

$$
IPC = \frac{Instructions}{Cycles}
$$

and:

$$
CPI = \frac{Cycles}{Instruction}
$$

Approximately:

$$
IPC \approx \frac{1}{CPI}
$$

for simple steady-state comparisons.

The ideal scalar pipeline approaches:

$$
IPC = 1
$$

A 4-wide machine has theoretical:

$$
IPC_{max}=4
$$

but actual IPC may be:

$$
IPC_{actual}<4
$$

because of dependencies, branches, cache misses, and resource conflicts.

---

## 37. A Useful Performance Model

We can think of CPI as:

$$
CPI = CPI_{base} + CPI_{branch} + CPI_{memory} + CPI_{data} + CPI_{structural}
$$

Therefore optimization means reducing the largest contributors.

For example:

```
Base CPI             1.0
Branch penalties    +0.15
Cache misses        +0.30
Load dependencies   +0.10
Structural stalls   +0.05
--------------------------------
Effective CPI        1.60
```

Improving a feature that contributes only 0.01 CPI may be pointless.

This is why:

> Profiling and bottleneck identification must come before optimization.

---

## 38. Amdahl's Law

One of the most important optimization principles:

$$
Speedup = \frac{1}{(1-f)+\frac{f}{S}}
$$

where:

- $f$ = fraction of execution improved
- $S$ = speedup of that portion

Suppose memory stalls represent 40% of execution.

Even making computation infinitely fast cannot remove that 40%.

Maximum theoretical speedup:

$$
\frac{1}{1-0.4}=1.67
$$

This is why:

> The bottleneck matters more than the average.

---

## 39. Energy Efficiency

Performance alone isn't enough.

We also care about:

$$
Performance/Watt
$$

or:

$$
Energy/Instruction
$$

A massive OoO core might achieve:

```
High performance
```

but consume considerably more:

- Area
- Power
- Energy
- Verification effort

A tiny in-order processor may be much more efficient for its target workload.

---

## 40. Performance Density

Another useful metric:

$$
Performance/Area
$$

For embedded systems, we may care about:

$$
\frac{Performance}{mm^2}
$$

For FPGA:

$$
\frac{Performance}{LUT}
$$

For battery-powered devices:

$$
\frac{Useful\ work}{Joule}
$$

Therefore "fastest" is not always "best."

---

## 41. ECO-RISC-V: Deep-Dive Design Target

The ten reference cores define a broad RISC-V microarchitectural design space, ranging from extremely small bit-serial processors to wide out-of-order machines.

ECO-RISC-V is evaluated as an optimization problem within this design space.

The objective is not to maximize performance at any cost, but to identify the most efficient amount of hardware parallelism for the intended workload.

### Primary Design Objective

> Maximize useful performance per unit of area and energy while maintaining a simple and predictable microarchitecture.

```
                   ECO-RISC-V
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
   Performance      Area Efficiency   Energy Efficiency
        │               │               │
        └───────────────┼───────────────┘
                        ▼
                Design Simplicity
```

### Core Design Philosophy

The goal is therefore not maximum hardware complexity.

The goal is:

> The minimum hardware complexity required to achieve the required performance.

---

## 42. ECO-RISC-V: Key Microarchitectural Design Questions

The ECO-RISC-V architecture should be evaluated across five major dimensions:

### 1. Pipeline Architecture

- How many pipeline stages are required?
- Why is that depth appropriate for the target workload?
- Where is the critical path?
- Where should pipeline registers be inserted?
- At which stage should branches be resolved?
- What is the branch misprediction penalty?
- What is the load-use latency?

### 2. Superscalar Execution

If multiple instructions are issued per cycle:

- What is the issue width?
- What limits instruction pairing?
- Can both issue slots access memory?
- How many ALUs are required?
- How many register-file read/write ports are needed?
- Does increased issue width provide sufficient IPC improvement?

### 3. Out-of-Order Execution

If dynamic scheduling is considered:

- How large should the ROB be?
- How many physical registers are required?
- How large should the issue queue be?
- How many instructions should remain in flight?
- What is the cost of wakeup/select logic?
- Is the additional ILP worth the area and energy overhead?

### 4. Instruction-Level Parallelism

- What dependencies dominate the workload?
- How much independent work exists?
- How much ILP can the compiler expose?
- How much additional ILP can hardware discover dynamically?

### 5. Memory & Branch Subsystems

**Memory**

- L1 cache size and associativity
- Cache latency
- Memory bandwidth
- Outstanding memory requests
- Load/store queue requirements
- Store-to-load forwarding

**Branch Prediction**

- Static or dynamic prediction?
- BTB requirement
- BHT requirement
- RAS requirement
- Branch resolution stage
- Misprediction recovery cost

---

## 43. ECO-RISC-V Optimization Philosophy

### More Hardware ≠ Automatically More Performance

A microarchitectural feature should be added only when the workload can meaningfully exploit the additional capability.

For example:

```
1-Wide Core
     │
   IPC 0.90
     │
     ▼
2-Wide Core
     │
   IPC 1.35
```

The theoretical issue width has doubled, but actual performance increases by only:

$$
\frac{1.35}{0.90}=1.5
$$

Therefore:

```
IPC improvement = 50%
```

But the hardware may experience:

- +70% area
- +60% power
- More routing congestion
- More complex hazard logic
- Greater verification effort

### Engineering Decision Metric

A useful architectural question is:

$$
\boxed{\frac{\Delta Performance}{\Delta Area}}
$$

and

$$
\boxed{\frac{\Delta Performance}{\Delta Power}}
$$

The objective is to maximize useful performance per unit of hardware cost, rather than simply maximizing peak IPC.

---

## 44. Ten-Core Architectural Design Space

| Core | Primary Execution Idea | Main Architectural Optimization |
|---|---|---|
| SERV | Bit-serial execution | Minimum hardware footprint |
| PicoRV32 | Multi-cycle FSM | Temporal resource reuse |
| Ibex | Small in-order pipeline | Security + efficiency |
| CV32E40P | Compact pipelined core | Embedded performance |
| SCR1 | Small in-order core | Area efficiency |
| VexRiscv | Configurable pipeline | FPGA adaptability |
| Rocket | Application-class in-order | Balanced performance |
| Shakti | 64-bit in-order | Practical SoC integration |
| SweRV EH1 | 2-wide in-order | Higher IPC without OoO |
| BOOM | Out-of-order superscalar | Dynamic ILP extraction |
| XiangShan | Wide out-of-order | Aggressive ILP exploitation |

### Architectural Progression

The cores represent progressively more aggressive methods of exploiting instruction-level parallelism.

---

## 45. The Central Question: How Much Parallelism?

The most useful way to compare the cores is not simply by their names or specifications.

It is by asking:

> How much parallelism does the architecture attempt to exploit?

```
SERV
 │
 │ Bit-level temporal execution
 ▼
PicoRV32
 │
 │ Multi-cycle resource reuse
 ▼
Small In-Order Cores
 │
 │ Pipeline overlap
 ▼
VexRiscv / Rocket / Shakti
 │
 │ Multiple instructions in flight
 ▼
SweRV
 │
 │ Multiple instructions per cycle
 ▼
BOOM
 │
 │ Dynamic scheduling
 ▼
XiangShan
 │
 │ Wide, aggressive dynamic ILP
 ▼
Maximum Practical Parallelism
```

### Important Distinction

Parallelism can be created in several different ways:

- **Temporal reuse** → reuse the same hardware across cycles
- **Pipelining** → overlap different instructions
- **Superscalar execution** → execute multiple instructions per cycle
- **Out-of-order execution** → dynamically execute whichever instructions are ready

---

## 46. Execution Paradigms

| Execution Paradigm | Example | Fundamental Idea |
|---|---|---|
| Bit-serial | SERV | Reuse a tiny datapath over many cycles |
| Multi-cycle | PicoRV32 | Reuse execution hardware across instruction phases |
| Scalar in-order | Ibex | One instruction progresses through an ordered pipeline |
| Pipelined in-order | CV32E40P / VexRiscv | Overlap multiple instructions |
| In-order superscalar | SweRV | Issue multiple instructions while preserving execution order |
| Out-of-order superscalar | BOOM | Dynamically execute ready instructions |
| Wide OoO | XiangShan | Aggressively exploit available ILP |

### The Fundamental Trade-Off

```
More Parallel Hardware
        │
        ▼
Higher Potential Throughput
        │
        ▼
More Area + Power + Complexity
        │
        ▼
Greater Verification & Routing Cost
```

The architectural challenge is determining where the additional parallelism stops being economically useful.

---

## 47. The Microarchitecture Performance Equation

At a high level:

$$
\boxed{Performance \approx Frequency \times IPC}
$$

where:

- **Frequency** = how many clock cycles occur per second
- **IPC** = Instructions Per Cycle

However, IPC is not determined by issue width alone.

$$
IPC =
f(
ILP,\;
Issue\ Width,\;
Dependencies,\;
Branch\ Prediction,\;
Memory\ Latency,\;
Execution\ Units
)
$$

### Hardware Cost

The corresponding implementation cost can be viewed as:

$$
Cost =
f(
Area,\;
Power,\;
Complexity,\;
Verification,\;
Routing,\;
Memory
)
$$

Therefore:

$$
\boxed{
Microarchitecture =
Performance
\leftrightarrow
Efficiency
\leftrightarrow
Complexity
}
$$

### Key Insight

A processor with a higher theoretical IPC is not automatically a better processor.

The architecture must match its workload and implementation constraints.

---

## 48. ECO-RISC-V: Microarchitectural Evaluation Framework

Every architectural block should be evaluated using the following questions:

### 1. What problem does it solve?

For example:

> A branch predictor reduces the number of cycles lost when control flow changes.

### 2. What performance bottleneck does it attack?

Does it improve:

- IPC?
- Frequency?
- Memory latency?
- Branch efficiency?
- Pipeline utilization?

### 3. What hardware does it add?

Examples:

- SRAM
- comparators
- multiplexers
- pipeline registers
- execution units
- queues
- prediction tables

### 4. What new bottleneck does it create?

Additional hardware may introduce:

- longer critical paths
- routing congestion
- increased power
- additional stalls
- larger verification space

### 5. Is the workload capable of exploiting it?

A powerful mechanism has little value if the workload cannot provide enough parallelism.

### 6. Could a simpler mechanism provide similar performance?

This is especially important for ECO-RISC-V.

### 7. What happens if the feature is removed?

This provides a direct way to quantify its actual architectural value.

> This is the difference between adding features and performing microarchitectural optimization.

---

## 49. Example: Should ECO-RISC-V Become 2-Wide?

Consider a hypothetical baseline:

### 1-Wide

$$
IPC = 0.90
$$

$$
Area = 1.0\times
$$

$$
Power = 1.0\times
$$

A 2-wide design produces:

$$
IPC = 1.35
$$

Therefore:

$$
Performance\ Gain =
\frac{1.35}{0.90}=1.5
$$

### Result

```
IPC improvement = 50%
```

But suppose implementation cost becomes:

$$
Area = 1.7\times
$$

$$
Power = 1.6\times
$$

The architecture has gained 50% performance at the cost of approximately 70% more area and 60% more power.

### Engineering Question

> Is the additional performance worth the additional hardware?

The answer depends on the target workload.

---

## 50. Example: Should ECO-RISC-V Use Out-of-Order Execution?

Consider another hypothetical comparison.

| Architecture | IPC | Area | Power |
|---|---|---|---|
| In-order | 1.1 | 1.0× | 1.0× |
| OoO | 1.7 | 2.5× | 2.3× |

The OoO design provides approximately:

$$
\frac{1.7-1.1}{1.1}
\approx54.5\%
$$

higher IPC.

But:

- Area increases by 150%
- Power increases by 130%

### When could OoO be justified?

Potentially:

- Linux application processors
- High-performance computing
- Memory-latency-sensitive workloads
- Workloads with substantial dynamic ILP

### When could it be excessive?

Potentially:

- Sensor controllers
- Tiny embedded systems
- Deterministic control applications
- Extremely area-constrained SoCs

### Key Principle

> The best execution paradigm depends on the workload, not on the theoretical maximum IPC.

---

## 51. The Fundamental Microarchitecture Philosophy

The comparison of these cores should not produce the conclusion:

> "Out-of-order is better than in-order."

The correct conclusion is:

> Out-of-order execution is more effective at extracting dynamic ILP when the workload and performance target justify its additional complexity and energy cost.

Similarly:

> A small in-order core is not simply a weaker OoO processor.

It is optimized for a different point in the design space.

And:

> SERV is not a "slow Rocket."

SERV solves a fundamentally different problem:

*How little hardware is required to execute RISC-V instructions?*

---

## 52. Unified Mental Model of a RISC-V Core

A useful way to understand every architecture is to follow an instruction from fetch to retirement.

```
                  INSTRUCTION STREAM
                         │
                         ▼
                  ┌────────────┐
                  │   FETCH    │
                  └─────┬──────┘
                        │
                        ▼
                  ┌────────────┐
                  │   DECODE   │
                  └─────┬──────┘
                        │
              ┌─────────┴─────────┐
              │                   │
              ▼                   ▼
          IN-ORDER                OoO
              │                   │
              │                RENAME
              │                   │
              │                DISPATCH
              │                   │
              │              ISSUE QUEUE
              │                   │
              │              READY CHECK
              │                   │
              ▼                   ▼
          EXECUTE             EXECUTE
              │                   │
              ▼                   ▼
           MEMORY             WRITEBACK
              │                   │
              └─────────┬─────────┘
                        ▼
                     RETIRE
```

### Questions to Ask for Every Core

- Where is parallelism created?
- Where is parallelism limited?
- Where are dependencies detected?
- Where are dependencies resolved?
- Where are branches predicted?
- Where are branches resolved?
- Where is memory latency hidden?
- Where is hardware duplicated?
- Where is hardware reused?
- Where is power consumed?
- Where is area consumed?

These questions form the foundation of microarchitectural analysis.

---

## 53. Ten-Core Microarchitecture Study Roadmap

The ten cores can be organized into six learning phases.

### Phase 1 — Foundations

**SERV**

Key concepts:

- Bit-serial computation
- Temporal vs. spatial computation
- Hardware reuse
- Area/performance trade-offs
- Minimal execution hardware

**PicoRV32**

Key concepts:

- FSM-based execution
- Multi-cycle datapaths
- Temporal resource sharing
- Latency vs. throughput
- Control complexity

---

### Phase 2 — In-Order Pipelining

**Ibex**

Key concepts:

- 2-stage pipelines
- Pipeline hazards
- Forwarding
- Stalls
- Branch handling
- Security-oriented design

**CV32E40P / SCR1**

Key concepts:

- Deeper embedded pipelines
- Pipeline balancing
- Branch penalties
- Load-use latency
- Embedded performance/area trade-offs

---

### Phase 3 — Configurable & Practical Pipelines

**VexRiscv**

Key concepts:

- Configurable pipeline architecture
- Plugin-based microarchitecture
- FPGA-specific optimization
- Configurable execution units
- Hardware/software generation

**Rocket**

Key concepts:

- Application-class in-order execution
- Cache hierarchy
- MMU and virtual memory
- Pipeline organization
- Scalability

**Shakti**

Key concepts:

- 64-bit in-order processing
- Practical SoC integration
- Linux-class execution
- Memory management
- Production-oriented RISC-V design

---

### Phase 4 — Superscalar Execution

**SweRV EH1**

SweRV represents the transition from:

$$
\boxed{1\text{-wide in-order}}
$$

to:

$$
\boxed{2\text{-wide in-order superscalar}}
$$

**Focus Areas**

- Issue width
- Dual execution pipelines
- Instruction pairing
- Dependency restrictions
- Bypass complexity
- Register-file port requirements
- IPC scaling
- Branch penalties

**Central Question**

> How much additional throughput can be obtained without introducing full out-of-order machinery?

---

### Phase 5 — Out-of-Order Execution

**BOOM**

BOOM introduces the major mechanisms required for dynamic ILP extraction:

- Register renaming
- Physical registers
- Register Alias Table (RAT)
- Reorder Buffer (ROB)
- Issue queues
- Wakeup/select logic
- Speculative execution
- Load/store queues
- Precise exceptions
- Branch recovery

**Central Question**

> How much additional parallelism can hardware discover dynamically?

---

### Phase 6 — High-Performance Out-of-Order

**XiangShan**

XiangShan represents a much more aggressive OoO design.

Focus areas include:

- Wide front-end
- Sophisticated branch prediction
- Large instruction windows
- Multiple execution units
- Aggressive memory systems
- Speculative execution
- High-bandwidth data paths
- Scaling limitations
- Power/performance trade-offs

**Central Question**

> What happens when the architecture aggressively attempts to exploit almost all available ILP?

---

## 54. Final Objective: Positioning ECO-RISC-V

After analyzing the ten reference architectures, ECO-RISC-V can be positioned according to a structured chain of design decisions:

```
Target Workload
       │
       ▼
Available ILP
       │
       ▼
Required Throughput
       │
       ▼
Required Latency
       │
       ▼
Area Budget
       │
       ▼
Power Budget
       │
       ▼
Execution Paradigm
       │
       ▼
Pipeline Depth
       │
       ▼
Issue Width
       │
       ▼
Memory System
       │
       ▼
Branch Prediction
       │
       ▼
Speculation Level
       │
       ▼
Required Complexity
```

### The Final Design Question

$$
\boxed{
\text{What is the minimum hardware complexity required to achieve the desired performance?}
}
$$

This question defines the ECO-RISC-V optimization problem.

---

## 55. One-Line Summary

> RISC-V defines what instructions a processor executes; microarchitecture determines how efficiently those instructions are executed; ECO-RISC-V focuses on selecting the right amount of parallelism rather than simply maximizing it.

---

## 56. Three Questions Every Architecture Engineer Should Be Able to Answer

### Q1. Why doesn't a 4-wide processor automatically achieve 4 IPC?

Because issue width represents hardware capacity, not guaranteed utilization.

Actual IPC is limited by:

- Data dependencies
- Control dependencies
- Branch mispredictions
- Cache misses
- Memory latency
- Available execution units
- Instruction-level parallelism in the workload

Therefore:

$$
\boxed{Issue\ Width \neq Actual\ IPC}
$$

---

### Q2. Why doesn't every processor use out-of-order execution?

Because OoO execution requires substantial additional hardware:

- Register renaming
- Physical register files
- ROB
- Issue queues
- Wakeup/select logic
- Load/store queues
- Speculative execution
- Recovery mechanisms

These increase:

- Area
- Power
- Routing complexity
- Verification complexity
- Design complexity

Therefore:

$$
\boxed{
More\ ILP\ extraction
\neq
Free\ performance
}
$$

---

### Q3. Why can a simpler processor sometimes be the better architecture?

Because the optimal architecture depends on:

$$
\boxed{
Workload+
Performance\ Target+
Area+
Power+
Cost+
Complexity
}
$$

A small in-order core may be substantially better than an OoO processor when:

- performance requirements are modest,
- deterministic execution is important,
- power is constrained,
- silicon area is limited,
- or the workload contains little exploitable ILP.

---

## Core Message

The ten architectures represent different solutions to the same fundamental problem:

> How should hardware execute a stream of RISC-V instructions efficiently?

**SERV**

How can hardware be minimized?

**PicoRV32**

How can hardware be reused across time?

**Ibex / CV32E40P / SCR1**

How can pipelining improve throughput without excessive complexity?

**VexRiscv / Rocket / Shakti**

How can practical in-order processors scale toward more capable systems?

**SweRV**

How can multiple instructions be executed per cycle without full OoO complexity?

**BOOM**

How can dynamic scheduling extract more ILP?

**XiangShan**

How far can aggressive OoO RISC-V scale?

**ECO-RISC-V**

What is the right amount of parallelism for the target workload?

---

## The Final Design-Space View

```
INCREASING PARALLELISM
                                ───────────────►

     SERV
      │
      │   Bit-serial
      ▼
   PicoRV32
      │
      │   Multi-cycle
      ▼
 Small In-Order
      │
      │   Pipelining
      ▼
 VexRiscv / Rocket / Shakti
      │
      │   Superscalar
      ▼
    SweRV
      │
      │   Dynamic scheduling
      ▼
     BOOM
      │
      │   Wide OoO
      ▼
   XiangShan


          AREA / POWER / COMPLEXITY
                    ▲
                    │
                    │
                    │
                    ▼
             PERFORMANCE / ILP
```

The important point is not that the design space has a single "best" processor.

It has a Pareto frontier.

A design is attractive when improving one objective significantly would require an undesirable sacrifice in another:

$$
\boxed{
Performance
\leftrightarrow
Area
\leftrightarrow
Power
\leftrightarrow
Complexity
}
$$

ECO-RISC-V's job is to find the right point on that frontier.

And that is where the deep dive becomes genuinely interesting: pipelining → superscalar execution → OoO execution → ILP → memory latency → branch prediction → performance/area/energy optimization, rather than simply memorizing ten RISC-V cores.
