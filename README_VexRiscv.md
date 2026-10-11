# VexRiscv: The SpinalHDL Ultra-Configurable FPGA RISC-V Core

## 1. Introduction

Winner of the FOSSi Foundation RISC-V Softcore Contest, **VexRiscv** (created by Charles Papon) is unique because it is not written in traditional Verilog, SystemVerilog, or Chisel. It is generated using **SpinalHDL**, a Scala-based hardware description language.

VexRiscv is engineered around a **plugin-driven microarchitecture**. Instead of a fixed pipeline, its hardware stages (Fetch, Decode, Execute, Memory, WriteBack) are dynamic buses. Features like branch predictors, MMU, CSRs, and caches are implemented as discrete plugins that insert pipeline stages, register signals, or inject bypass multiplexers automatically.

---

## 2. Microarchitectural Anatomy: The Flexible 5-Stage Datapath

In its standard configuration, VexRiscv is a balanced, in-order 5-stage pipeline:

```text
[ Stage 1: Fetch ]
   - Program counter generation
   - Instruction cache access / Wishbone / AXI4 bus request
   - Static / dynamic branch prediction injection
                      │
                      ▼
[ Stage 2: Decode ]
   - Instruction format slicing & immediate generation
   - Register file read (asynchronous distributed RAM or synchronous BRAM)
   - Hazard detection interlock logic
                      │
                      ▼
[ Stage 3: Execute ]
   - RV32I arithmetic logic unit (ALU)
   - Branch target evaluation & comparison resolution
   - Shift operations (single-cycle barrel shifter or iterative)
   - Address generation unit (AGU) for data access
   - Hardware multiplier / divider (RV32M plugin)
                      │
                      ▼
[ Stage 4: Memory ]
   - Data cache / Wishbone / AXI4 bus master access
   - Alignment logic for sub-word accesses (byte, halfword)
   - Optional Sv32 MMU / TLB translation
                      │
                      ▼
[ Stage 5: WriteBack ]
   - Arbitration between memory load data and ALU/plugin results
   - Register file commit to destination register (rd)
```

### 2.1 The SpinalHDL Plugin Mechanism

In classical Verilog, adding an instruction requires manually cutting wires across `decode.v`, `execute.v`, and `hazard_unit.v`.

In VexRiscv, every function is an isolated SpinalHDL software plugin:

- A **multiplier plugin** requests access to the Execute and WriteBack stages, declaring its own pipeline latency and hazard bypass lines.
- If a feature is disabled (e.g., removing the MMU or switching from a barrel shifter to an iterative shifter), the generator rebuilds the pipeline netlist with zero leftover logic gates, eliminating dead silicon.

---

## 3. Hazard Handling & Forwarding Topology

- **Multi-stage forwarding (bypass logic):**
  - Configurable depth: can be synthesized with full bypass paths from both $\text{Memory} \to \text{Execute}$ and $\text{WriteBack} \to \text{Execute}$.
  - Allows back-to-back arithmetic instructions to run at 1 CPI (0 stalls).
- **Load-use hazards (1-cycle interlock):**
  - When an instruction immediately consumes the output of a load (`LW`), the hazard unit injects a 1-cycle stall bubble into the Decode stage while the data is fetched from the cache/bus during the Memory stage.
- **Branch resolution options:**
  - **No prediction:** Resolves branches in Execute, inflicting a fixed 2-cycle penalty.
  - **Static branch prediction:** Assumes backward branches (loops) are taken and forward branches are not taken.
  - **Dynamic branch predictor plugin:** Instantiates a Branch Target Buffer (BTB) in the Fetch stage to drop the branch penalty to 0 cycles on correct hits.

---

## 4. Native Integration & Bus Interfaces

- **LiteX SoC ecosystem:**
  - VexRiscv is the default CPU core for the LiteX SoC generator framework, making it the de facto core for open-source FPGA developers building Linux-capable platforms, retro-computing systems, and hardware testbeds.
- **Bus interconnects:**
  - Directly supports AXI4, AXI3, Avalon, and Wishbone B4 without external bridge wrappers.
- **Linux capability (Sv32 MMU):**
  - Supports an optional hardware page table walker and TLB plugin, enabling it to run a full Linux kernel on budget FPGAs.

---

## 5. Implementation Metrics (FPGA Synthesis Baseline)

For a standard RV32IM configuration (5-stage, with bypass forwarding and barrel shifter):

| Target FPGA Architecture | LUT Utilization | Flip-Flop Count | Block RAMs (BRAM) | DSP Slices | Max Clock ($F_{\max}$) | CoreMark / MHz |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **AMD/Xilinx Artix-7** | ~1,200 – 1,800 | ~900 – 1,300 | 0 – 2 | 0 – 3 | ~180 – 220 MHz | ~2.4 – 2.9 |
| **Lattice ECP5** | ~1,500 – 2,200 | ~1,000 – 1,400 | 0 – 2 | 0 – 3 | ~80 – 110 MHz | ~2.4 – 2.9 |
| **Microchip PolarFire** | ~1,400 – 2,000 | ~950 – 1,350 | 0 – 2 | 0 – 3 | ~130 – 160 MHz | ~2.4 – 2.9 |
| **Intel Cyclone 10 LP** | ~1,600 – 2,400 | ~1,000 – 1,400 | 0 – 2 | 0 – 3 | ~120 – 150 MHz | ~2.4 – 2.9 |

> **Note:** In an ultra-minimal RV32I configuration without a barrel shifter or caches, it shrinks to ~500–700 LUTs, closing timing at high frequencies.

---

## 6. Architectural Trade-Off Analysis

| Metric | PicoRV32 (Multi-Cycle) | Ibex (2-Stage) | VexRiscv (5-Stage) | Rocket Core (5-Stage) |
| :--- | :--- | :--- | :--- | :--- |
| **Datapath width** | 32-bit | 32-bit | 32-bit | 32-bit / 64-bit |
| **Average CPI** | ~3.0 – 4.0 | ~1.2 – 1.6 | ~1.1 – 1.3 | ~1.05 – 1.25 |
| **Area efficiency** | High (compact FSM) | Medium | Very high (FPGA-tailored) | Low (large footprint) |
| **FPGA optimization** | Generic RTL | ASIC-focused | Native LUT/BRAM mapping | ASIC-focused |
| **Custom extensibility** | Hard (Verilog edits) | Hard (fixed stages) | Trivial (SpinalHDL plugin) | Medium (RoCC interface) |

### Strengths

- **FPGA-native implementation:** Unlike Rocket or Ibex (which were created for ASIC fabrication and adapted to FPGAs), VexRiscv's SpinalHDL generator produces Verilog structured to match FPGA vendor LUT, carry-chain, and distributed RAM topologies.
- **Extreme granularity:** Can be configured from a tiny 500-LUT microcontroller up to a full Linux-capable dual-core with MMU and caches.
- **Mature open-source toolchain support:** Seamless integration with LiteX, Yosys, nextpnr, and SymbiFlow.

### Limitations

- **SpinalHDL barrier:** For developers accustomed to SystemVerilog or Verilog, navigating Scala and SpinalHDL meta-code to debug signal waveforms can be unintuitive.
- **Deterministic real-time hardening:** Lacks the formal security/fault-injection verification suites found in commercial-grade roots of trust like Ibex/OpenTitan.

---

## 7. Primary Target Use Cases

1. **LiteX FPGA SoC designs:** Powering open-source FPGA developer boards, HDMI video generators, USB controllers, and network stacks.
2. **Budget Linux platforms:** Booting embedded Linux on low-cost FPGAs (like Lattice ECP5 or Artix-7 35T) where Rocket Core is too large to fit.
3. **Domain-specific architectural prototyping:** Rapidly writing a custom arithmetic plugin to test new ISA extensions before committing to dedicated hardware RTL.
