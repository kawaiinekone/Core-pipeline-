# SonicBOOM: The Berkeley Out-of-Order RISC-V Core (BOOM v3)

## 1. Executive Architectural Summary

**SonicBOOM** (Berkeley Out-of-Order Machine v3), engineered within UC Berkeley's Chipyard framework, is an open-source, superscalar, out-of-order (OoO) RISC-V application core. Written in Chisel, it targets commercial, high-performance general-purpose computing comparable to ARM Cortex-A72/A73 and modern Neoverse designs.

Unlike in-order cores that freeze when an instruction hits a memory cache miss or dependency hazard, SonicBOOM decouples fetch and commit from execution. It implements dynamic Tomasulo-style out-of-order execution with explicit register renaming, distributed issue queues, an indexed physical register file (PRF), and an in-order architectural Reorder Buffer (ROB).

---

## 2. Microarchitectural Anatomy: The Decoupled OoO Pipeline

SonicBOOM is parameterized as a 2-wide, 3-wide, or 4-wide superscalar machine. The core divides execution across three primary operating regimes:

```text
[ IN-ORDER FRONT-END ]
  F1/F2 (Fetch)   ──► TAGE branch predictor, BTB, RAS, L1-I$ cache access
  F3 (Pre-Decode) ──► 16-bit RVC expansion, instruction packet alignment
  Decode          ──► Up to 4-wide instruction decode into micro-ops (μOPs)
  Rename/Dispatch ──► RAT renaming, ROB allocation, issue queue insertion
                             │
                             ▼
[ OUT-OF-ORDER EXECUTION ENGINE ]
  Issue Window    ──► Wakeup & select logic (operands ready check)
  RegRead         ──► Unified physical register file (PRF) access
  Execute (EX)    ──► Parallel functional units (ALU0–3, FPU, pipelined AGUs)
                             │
                             ▼
[ IN-ORDER COMMIT & RETIREMENT ]
  Writeback (WB)  ──► Broadcast on common data bus (CDB), wake up dependent μOPs
  ROB Commit      ──► Strict in-order architectural commit & precise traps
```

### 2.1 The Front-End (In-Order Fetch & TAGE Predictor)

- **Fetch engine:** Fetches a 64-bit or 128-bit aligned instruction packet per cycle from the level-1 instruction cache (L1-I$).
- **Branch prediction suite:**
  - **TAGE predictor:** Tagged Geometric history length branch predictor handling complex correlated branch patterns.
  - **BTB (Branch Target Buffer):** Caches predicted branch targets for zero-cycle jump redirection.
  - **RAS (Return Address Stack):** Tracks nested subroutine return addresses.
- **Pre-decode:** Slices 16-bit compressed RISC-V instructions (RVC) combinationally and translates them into canonical 32-bit internal micro-ops before entering the decode queues.

### 2.2 Rename & Dispatch (Eliminating Name Dependencies)

- **Register Alias Table (RAT):**
  - Eliminates false hazards (Write-After-Read (WAR) and Write-After-Write (WAW)) by mapping the 32 architectural registers (`x0`–`x31`) onto a large pool of physical registers (typically 80 to 128 registers in the PRF).
- **Resource allocation:**
  - Allocates an entry at the tail of the Reorder Buffer (ROB), which is 64 to 128 entries deep.
  - Dispatches the renamed micro-op into an available slot in the unified or distributed Issue Queue (IQ).

### 2.3 Issue Queue, Wakeup & Execution Ports (Out-of-Order Engine)

- **Wakeup & select:**
  - Instructions wait in the Issue Queue until all dependent physical registers are tagged as valid.
  - The selection logic prioritizes and dispatches ready instructions to available execution ports regardless of original program order.
- **Execution resources:**
  - Multiple integer ALUs (single-cycle arithmetic, logic, and branches).
  - Pipelined multiplier/divider unit.
  - Full IEEE-754 single/double precision floating-point unit (FPU).
  - Dual Address Generation Units (AGUs) for concurrent load/store address calculations.

### 2.4 Load-Store Unit & Memory Disambiguation

- **Out-of-order LSU:** Features a dedicated Load Queue (LQ) and Store Queue (SQ).
- **Speculative loads:** Loads execute before older stores whose addresses have not yet resolved. If an address conflict is detected upon store address calculation, the pipeline squashes speculative dependents and replays the load.
- **Store-to-load forwarding:** Data written by an uncommitted store in the SQ is forwarded directly to a matching load without waiting for L1-D$ cache commit.

### 2.5 Reorder Buffer (ROB) & In-Order Commit

- Instructions write computed values into the Physical Register File and mark their ROB entries as finished.
- Instructions commit to architectural state strictly in program order from the head of the ROB.
- **Precise exceptions:** If a mispredicted branch or page fault occurs, all subsequent instructions in flight are squashed instantaneously, rolling the RAT back to the checkpointed register state without corrupting memory or architectural registers.

---

## 3. Dynamic Hazard Handling & Latency Hiding

| Hazard Type | Handling in Standard In-Order Cores | SonicBOOM Hardware Mechanism |
| :--- | :--- | :--- |
| **RAW data hazard** | Pipeline stall interlock or bypass multiplexer delay. | **Dynamic wakeup.** Operands broadcast across the Common Data Bus (CDB); instructions wake up dynamically in the Issue Queue. |
| **WAR / WAW hazards** | Pipeline stall or strict sequential commitment. | **Eliminated.** Register renaming assigns separate physical registers to conflicting register names. |
| **Branch mispredict** | 1- to 3-cycle flush penalty. | **8 to 11-cycle penalty.** Rolls back the ROB and RAT state; mitigated by deep TAGE branch prediction. |
| **L1 cache miss** | Core stalls completely until DRAM returns data. | **Latency hiding.** Hit-under-miss non-blocking caches allow independent instructions to execute around the stall. |

---

## 4. Hardware Footprint & Implementation Metrics

Because SonicBOOM instantiates dynamic scheduling windows, renaming tables, and multi-port register files, its logic footprint is substantially larger than in-order cores:

| Target Platform / Synthesis Target | Configuration | Logic Resource Utilization | Block RAMs | Max Clock ($F_{\max}$) | CoreMark / MHz |
| :--- | :--- | :--- | :---: | :---: | :---: |
| **TSMC 28nm / 16nm ASIC** | 3-wide, 96-ROB, RV64GC | Die: ~1.5 – 2.5 mm² | Dedicated SRAMs | ~1.2 – 1.8 GHz | ~6.2 |
| **AMD Xilinx UltraScale+ (VU9P)** | 2-wide Medium BOOM | ~45,000 – 65,000 LUTs | 40 – 80 BRAMs | ~40 – 60 MHz | ~5.5 – 6.2 |
| **AMD Artix-7 (XC7A100T)** | Any BOOM configuration | Exceeds device capacity | Exceeds limits | N/A | N/A |

> **Note:** SonicBOOM is typically emulated on large FPGA clusters using AWS EC2 F1 instances or FireSim rather than low-cost development boards.

---

## 5. Architectural Trade-Off Analysis

| Metric | Rocket Core (5-Stage) | SweRV EH1 (9-Stage Dual-Pipe) | SonicBOOM (BOOMv3) |
| :--- | :--- | :--- | :--- |
| **Issue model** | 1-wide in-order | 2-wide in-order superscalar | 2- to 4-wide out-of-order |
| **In-flight instructions** | Max 5 instructions | Max 18 instructions across dual pipes | 64 to 128 instructions (ROB) |
| **Register storage** | 32 architectural entries | 32 architectural entries (multi-port) | 80–128 physical registers (PRF) |
| **Branch penalty** | 2 cycles | 8 cycles | 8 to 11 cycles (flushes ROB) |
| **Real-world IPC** | ~0.65 – 0.85 | ~1.0 – 1.35 | ~1.6 – 2.4 |
| **Transient execution** | Immune to Spectre-v1/v2 | Immune to Spectre-v1/v2 | Requires hardware fence mitigations |

### Strengths

- **High instruction-level parallelism (ILP):** Reaches high IPC on branch-heavy and memory-intensive applications by extracting parallelism out-of-order.
- **Non-blocking memory system:** Memory access latency to L2/L3 caches and DRAM is overlapped with arithmetic execution.

### Limitations

- **Area & power overhead:** High gate count and dynamic power consumption make it unsuited for embedded microcontrollers.
- **Complex timing closure:** Multi-ported issue queues and register alias tables create long critical paths during physical synthesis.

---

## 6. Primary Target Use Cases

1. **High-performance RISC-V silicon:** Server processors, datacenter accelerators, and application processors running full Linux workloads.
2. **Computer architecture prototyping:** Evaluating cache coherence, speculative execution vulnerabilities, and memory consistency models.
3. **Cloud emulation via FireSim:** Running cycle-accurate full-system benchmarks on AWS EC2 FPGA instances.

---

## 7. References

1. **BOOM official repository:** <https://github.com/riscv-boom/riscv-boom>
2. **Chipyard framework:** <https://github.com/ucb-bar/chipyard>
3. **SonicBOOM microarchitecture paper (CARRV):** *The SonicBOOM Architecture*
