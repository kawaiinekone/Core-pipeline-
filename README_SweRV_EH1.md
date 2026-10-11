# SweRV Core EH1 (Western Digital / CHIPS Alliance): The Dual-Issue Superscalar RISC-V Core

## 1. Introduction

Developed by Western Digital and open-sourced through CHIPS Alliance, **SweRV Core EH1** (now known as the **VeeR EL2/EH1** family) is an industrial-grade, dual-issue superscalar, 32-bit embedded core designed specifically for high-throughput flash controller firmware and storage subsystems.

Unlike simple microcontrollers (like Ibex or Vega) that are single-issue in-order, and unlike server-class cores that spend area on out-of-order schedulers (like SonicBOOM), SweRV EH1 implements an **in-order, 2-way superscalar, 9-stage dual pipeline** (EX1–EX4). It fetches and executes up to two instructions per clock cycle while maintaining a compact, deterministic, ASIC-optimized area profile.

---

## 2. Microarchitectural Anatomy: The 9-Stage Dual Pipeline

SweRV EH1 breaks execution into a deep 9-stage in-order pipeline with two execution pipes: **Pipeline I0** (primary) and **Pipeline I1** (secondary).

```text
[ Stage 1: Fetch 1 (F1) ] ──► Next-PC generation, Branch Target Buffer (BTB) & Gshare lookup
[ Stage 2: Fetch 2 (F2) ] ──► Instruction cache access & fetch alignment
[ Stage 3: Fetch 3 (F3) ] ──► 16-bit RVC expansion, 8-entry fetch buffer FIFO
[ Stage 4: Decode (DEC) ] ──► Dual-instruction decode, structural & dependency check
                                       │
            ┌──────────────────────────┴──────────────────────────┐
            ▼                                                     ▼
      [ Pipeline I0 ]                                       [ Pipeline I1 ]
[ Stage 5: EX1 ] ALU0, AGU (load/store)                [ Stage 5: EX1 ] ALU1, branch eval
[ Stage 6: EX2 ] D-cache tag/access, shift             [ Stage 6: EX2 ] Secondary ALU / shift
[ Stage 7: EX3 ] Load data alignment                   [ Stage 7: EX3 ] Early commit
[ Stage 8: EX4 ] Pipelined multiplier final            [ Stage 8: EX4 ] Pipe alignment
            └──────────────────────────┬──────────────────────────┘
                                       ▼
[ Stage 9: WB (Retire) ] Dual-instruction writeback to register file (up to 2 inst/cycle)
```

### 2.1 Detailed Stage Operations

- **Front-end (F1, F2, F3 stages):**
  - **Dual fetch:** Fetches a 64-bit aligned bundle every cycle from a tightly-coupled instruction cache (ICache) or Instruction Closely Coupled Memory (ICCM).
  - **Advanced branch predictor:** Contains a fully integrated BTB (Branch Target Buffer), a BHT (Branch History Table) using a Gshare-style hashing scheme, and a dedicated RAS (Return Address Stack).
  - **Fetch buffer FIFO:** Decouples front-end cache fetch latency from dual-issue decode.
- **Decode & dual-issue logic (DEC stage):**
  - Determines whether two sequential instructions can be dual-issued simultaneously:
    - **I0 pipe:** Can execute any integer ALU operation, loads, stores, branches, shifts, and multiply/divide.
    - **I1 pipe:** Executes a subset of operations: non-memory integer arithmetic, logic, and branches.
  - **Dual-issue rules:** If Instruction 1 and Instruction 2 have a RAW data dependency that cannot be bypassed, or if both are memory load/store operations (structural conflict on the single LSU), Instruction 2 is held back and the core issues in single-issue mode for that cycle.
- **Execution pipes (EX1 to EX4 stages):**
  - **EX1:** Arithmetic operations compute combinationally; load/store effective addresses ($rs1 + \text{imm}$) are calculated here.
  - **EX2:** Data cache / DCCM access occurs. Shift operations resolve.
  - **EX3:** Memory load data arrives and aligns (byte, halfword, word sign extension).
  - **EX4:** Hardware multiplier results settle.
- **Retirement & writeback (WB stage):**
  - Can retire up to 2 instructions per cycle into the physical 4-read/2-write port register file.
  - Maintains precise architectural exception semantics.

---

## 3. Hazard Handling & Interlock Mechanics

Because SweRV EH1 has two concurrent execution pipes spanning four execution stages each, dependency management requires an extensive bypass crossbar.

### 3.1 Cross-Pipe ALU Bypass Matrix

SweRV implements full combinational forwarding between both pipelines (I0 and I1). The $EX1 \to EX1$ bypass allows an independent instruction in I1 to consume an ALU result generated in I0 during the same cycle if scheduling allows, or with a 0-cycle bubble into the next cycle's EX1.

### 3.2 Load-to-Use Latency (2-Cycle Bubble)

Because load data returns at the end of EX2 and is aligned in EX3, a dependent instruction issued right after a load must wait:

```asm
lw  x1, 0(x2)    # Issues in I0
add x3, x1, x4   # Dependent on x1
```

SweRV EH1 incurs a 2-cycle load-use latency. The hazard detection unit freezes decode until the data reaches the EX3 bypass multiplexer.

### 3.3 Dual-Branch Handling

Both pipes can evaluate branches. When a branch mispredicts, the front-end fetch queue is flushed, inflicting an **8-cycle misprediction penalty** (due to the deep 9-stage pipeline). This makes the Gshare/BTB branch predictor critical for maintaining high IPC.

---

## 4. Tightly Coupled Memories (ICCM / DCCM)

- **Closely coupled memory subsystem:**
  - In addition to (or in place of) standard caches, SweRV integrates dedicated ICCM (Instruction CCM) and DCCM (Data CCM) SRAM arrays.
  - These provide deterministic, single-cycle access times to eliminate unpredictable cache miss penalties in real-time storage controller workloads.
- **ECC protection:**
  - All internal memories, queues, and register files feature SECDED (Single Error Correction, Double Error Detection) ECC bits for mission-critical enterprise storage reliability.

---

## 5. Implementation Metrics (FPGA Synthesis Baseline)

For the standard SweRV EH1 (RV32IMC, dual-issue superscalar, with ICCM/DCCM controllers):

| Target FPGA Architecture | LUT Utilization | Flip-Flop Count | Block RAMs (BRAM) | DSP Slices | Max Clock ($F_{\max}$) | CoreMark / MHz |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **AMD/Xilinx Artix-7** | ~18,000 – 24,000 | ~12,000 – 16,000 | 8 – 16 | 4 – 8 | ~60 – 80 MHz | ~4.9 – 5.1 |
| **AMD Kintex-7 / UltraScale+** | ~16,000 – 21,000 | ~12,000 – 15,000 | 8 – 16 | 4 – 8 | ~140 – 180 MHz | ~4.9 – 5.1 |
| **Microchip PolarFire (MPF300)** | ~19,000 – 25,000 | ~13,000 – 17,000 | 10 – 20 | 4 – 8 | ~75 – 95 MHz | ~4.9 – 5.1 |

> **Note:** On modern TSMC 16nm/7nm ASIC nodes, SweRV EH1 closes timing at well over 1.2 to 1.8 GHz because its deep pipeline keeps combinational logic levels per stage very small.

---

## 6. Architectural Trade-Off Analysis

| Metric | Vega ET1031 (3-Stage) | Rocket Core (5-Stage) | SweRV EH1 (9-Stage Dual-Issue) |
| :--- | :--- | :--- | :--- |
| **Issue width** | 1-wide (single-issue) | 1-wide (single-issue) | 2-wide superscalar |
| **Pipeline depth** | 3 stages | 5 stages | 9 stages (dual execution) |
| **Peak IPC** | 1.0 | 1.0 | 2.0 |
| **CoreMark / MHz** | ~1.8 – 2.1 | ~2.8 – 3.2 | ~4.9 – 5.1 |
| **Branch penalty** | 1 cycle | 2 cycles | 8 cycles (mitigated by BTB/BHT) |
| **Load-to-use stalls** | 1 cycle | 1 cycle | 2 cycles |
| **Register file ports** | 2 read / 1 write | 2 read / 1 write | 4 read / 2 write |

### Strengths

- **High IPC in an embedded footprint:** Reaches ~5.0 CoreMark/MHz without the immense silicon area and power dissipation of out-of-order scheduling windows.
- **Deterministic timing with CCMs:** Ideal for hard real-time storage firmware where cache jitter cannot be tolerated.
- **ASIC silicon frequency:** The deep pipeline enables gigahertz-range clocking on physical foundry nodes.

### Limitations

- **Deep mispredict penalty:** An 8-cycle flush penalty means complex, non-linear control code with poor branch predictability suffers significant throughput degradation.
- **FPGA routing congestion:** The 4R/2W register file and dual-pipe crossbar forwarding matrix create heavy routing congestion on low-cost FPGAs.

---

## 7. Primary Target Use Cases

1. **High-performance storage controllers:** SSD flash translation layers (FTL), NVMe command decoders, and HDD servo control engines.
2. **High-throughput network microcontrollers:** Ingesting packet headers, parsing network frame descriptors, and processing protocol stacks.
3. **Automotive & industrial real-time computing:** Deterministic compute tasks requiring high integer throughput paired with ECC fault resilience.
