# Shakti C-Class (IIT Madras / InCore Semiconductors): The Bluespec-Based 64-Bit RISC-V Core

## 1. Introduction

Developed by the RISE Lab at IIT Madras under the Government of India's Microprocessor Development Programme (MDP) and later commercialized through InCore Semiconductors, **Shakti C-Class** is India's flagship 64-bit production-grade RISC-V application processor.

It holds a special place in semiconductor history as India's first indigenous silicon tapeout (RISECREEK on Intel 22nm FFL and RIMO on SCL 180nm). Unlike traditional Verilog/VHDL cores or Chisel designs, Shakti is authored entirely in **Bluespec SystemVerilog (BSV)**, a high-level, mathematically formal guarded-atomic-action language that eliminates race conditions and non-deterministic concurrency by construction.

---

## 2. Microarchitectural Anatomy: The 5-to-6 Stage In-Order Pipeline

Shakti C-Class is an in-order, single-issue processor implementing RV64GC (and RV32GC variants) with supervisor mode support (Sv39 MMU), capable of booting full Linux, FreeBSD, and seL4 microkernels.

```text
[ Stage 1: Fetch (PC Gen + I-Cache / TLB) ]
   - Next-PC generation with Return Address Stack (RAS) & BHT predictor
   - Virtually Indexed Physically Tagged (VIPT) L1 instruction cache access
   - 16-bit compressed instruction decompressor (RVC)
                      │
                      ▼
[ Stage 2: Decode ]
   - Instruction field slicing, control generation & immediate expansion
   - Register scoreboard stall check & interlock assertions
                      │
                      ▼
[ Stage 3: RegRead & Dispatch ]
   - Register file read (dual-port 64-bit general-purpose registers)
   - Operand forwarding multiplexer matrix
                      │
                      ▼
[ Stage 4: Execute (EX / AGU) ]
   - 64-bit integer ALU & branch evaluation
   - Effective address generation unit (AGU) for load/store
   - Pipelined multiplier & restoring/early-out divider
   - Coprocessor dispatch (FPU single/double precision, IEEE 754)
                      │
                      ▼
[ Stage 5: Memory (MEM) ]
   - VIPT L1 data cache access & DTLB physical address translation
   - Alignment engine (byte, halfword, word, doubleword load/stores)
   - Atomic memory operation (AMO) controller (A-extension)
                      │
                      ▼
[ Stage 6: Writeback (WB / Retire) ]
   - Register commit multiplexer (ALU vs. memory vs. FPU)
   - Architectural state commit, CSR update, and trap handling unit
```

> **Note:** In earlier RTL releases, stages 2 and 3 were unified, presenting a classical 5-stage profile. Later hardened iterations decoupled Decode from RegRead for frequency scaling, giving 6 stages.

---

## 3. Guarded Atomic Actions: Why BSV Dictates the Microarchitecture

In Verilog cores (like Vega or PicoRV32), designers manually coordinate clock-enable wires, pipeline valid/ready strobes, and hazard stalls across modules.

In Shakti, every pipeline stage boundary is modeled as a **Bluespec Rule** (guarded atomic action):

- A rule executes if and only if its boolean guard condition is satisfied (e.g., `rule rl_execute (stage2.ready && stage3.has_free_entry)`).
- The BSV compiler automatically synthesizes the arbitration logic, handshake muxes, and stall signals, formally proving that:
  - No instruction can read uninitialized register states.
  - No pipeline register can undergo conflicting concurrent writes.
  - Structural conflicts between instruction fetch and memory writeback are scheduled deterministically.

---

## 4. Hazard Handling & Interlocks

### 4.1 Forwarding Crossbar Matrix

Combinational forwarding paths route results directly from:

- **Stage 4 (EX) → Stage 3 (RegRead):** Back-to-back arithmetic instructions execute with 0 stall cycles (1 CPI).
- **Stage 5 (MEM) → Stage 3 (RegRead):** Catches dependencies separated by an intervening cycle.

### 4.2 Load-Use Data Hazards (1-Cycle Bubble)

Data read from the L1 data cache is only ready after tag-matching in Stage 5 (MEM). If Stage 3 samples an operand that is the target of a load currently in Stage 4:

```asm
ld  x1, 0(x2)    # in EX/AGU -> advancing to MEM
add x3, x1, x4   # in RegRead -> dependent on x1
```

The hardware scoreboard detects the RAW hazard, pauses Stages 1–3, and injects a 1-cycle bubble into the execution pipeline until the load completes.

### 4.3 Control Hazards & Branch Prediction

- **RAS (Return Address Stack):** Dedicated hardware stack predicting function return addresses without penalty.
- **Branch Target Buffer (BTB) + BHT:** 2-bit saturating counter branch predictor.
- **Mispredict penalty:** When a branch evaluates in Stage 4 (EX), a mispredict asserts a synchronous flush line back to the front-end, inducing a 2- to 3-cycle misprediction penalty.

### 4.4 Memory Ordering & Atomic Extensions (A-Extension)

Integrates a dedicated reservation station for LR/SC (Load-Reserved / Store-Conditional) operations and native hardware atomics (`AMOSWAP`, `AMOADD`, etc.) at the L1 D-cache boundary.

---

## 5. Implementation Metrics & Synthesis Footprint

For a full RV64IMAFDC (RV64GC) configuration with 32KB L1 caches and Sv39 MMU:

| Target Platform / Process | Logic Resource Utilization | Block RAMs | DSP Slices | Max Clock ($F_{\max}$) | CoreMark / MHz |
| :--- | :--- | :---: | :---: | :---: | :---: |
| **Intel 22nm FFL (RISECREEK)** | Die area: ~16 mm² (with SoC) | Cache SRAMs | Dedicated | ~350 – 400 MHz | ~2.8 – 3.2 |
| **SCL Mohali 180nm (RIMO)** | Die area: ~144 mm² (SoC) | Cache SRAMs | Dedicated | ~70 MHz | ~2.8 – 3.2 |
| **Xilinx Kintex-7 (KC705)** | ~22,000 – 28,000 LUTs | 35 – 50 BRAMs | 8 – 16 | ~80 – 100 MHz | ~2.8 – 3.2 |
| **Microchip PolarFire (MPF300)** | ~24,000 – 30,000 LUTs | 40 – 55 BRAMs | 8 – 16 | ~75 – 90 MHz | ~2.8 – 3.2 |

> **Note:** When configured as an integer-only RV32IM / RV64IM without MMU or FPU, it synthesizes into ~7,000–9,000 LUTs on Artix-7.

---

## 6. Architectural Trade-Off Analysis

| Metric | Vega AS1061 (C-DAC) | Rocket Core (Berkeley) | Shakti C-Class (IIT-M) | Kodanda (4-wide) |
| :--- | :--- | :--- | :--- | :--- |
| **Language** | Verilog / SystemVerilog | Chisel (Scala) | Bluespec (BSV) | Verilog / SystemVerilog |
| **Issue width** | 1-wide (in-order) | 1-wide (in-order) | 1-wide (in-order) | 4-wide (in-order) |
| **Pipeline depth** | 5 stages | 5 stages | 5 to 6 stages | Variable / deep |
| **CoreMark / MHz** | ~3.0 – 3.4 | ~2.8 – 3.2 | ~2.8 – 3.2 | ~5.7 (synthetic peak) |
| **Random code IPC** | ~0.70 – 0.85 | ~0.65 – 0.85 | ~0.65 – 0.82 | ~0.65 (choked on memory) |
| **Memory isolation** | Physical / Sv39 | Sv39 / Sv48 | Sv39 MMU + PMP | PMP / MMU |

### Strengths

- **High verification reliability:** Bluespec's guarded atomic actions make hardware deadlock detection and race condition elimination mathematically provable during compilation.
- **Proven national silicon:** Validated across physical silicon tapeouts (Intel 22nm, SCL 180nm) and deployed across strategic national sectors.
- **Full OS capability:** Mature Linux, FreeRTOS, and Zephyr board support packages with full Sv39 virtual memory translation.

### Limitations

- **Bluespec compilation artifacts:** The BSV compiler (`bsc`) generates Verilog netlists with extensive automatically generated temporary wire names, making post-synthesis gate-level waveform debugging complex compared to handwritten Verilog.
- **Single-issue IPC cap:** Being a strict single-issue pipeline, its theoretical maximum IPC is 1.0, naturally limiting its performance compared to superscalar designs.

---

## 7. Primary Target Use Cases

1. **Indigenous Linux compute nodes:** Defense, aerospace, and strategic communications infrastructure requiring verified, royalty-free Indian silicon.
2. **Industrial control & gateways:** Power grid automation, railway signalling controllers, and secure edge telemetry.
3. **Automotive & security enclaves:** Used as the baseline processor for fault-tolerant and physically secure variants developed at IIT Madras and InCore.
