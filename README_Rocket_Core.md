# Rocket Core (UC Berkeley / CHIPS Alliance): The 5-Stage In-Order RISC-V Application Core

## 1. Executive Architectural Summary

**Rocket Core**, developed by the University of California, Berkeley architecture research group and maintained under the CHIPS Alliance, is the standard open-source application-class processor for RISC-V. Authored in **Chisel** (Constructing Hardware In a Scala Embedded Language), it generated the original reference implementation of the unprivileged and privileged RISC-V specifications and served as the direct microarchitectural baseline for early commercial silicon (such as the SiFive Freedom U540 and E31/E51 series).

Unlike embedded microcontrollers that lack memory translation hardware, Rocket is engineered as a full Unix/Linux-capable system. It implements the RV64GC (or RV32GC) ISA, featuring a classical in-order 5-stage pipeline, non-blocking Level-1 caches with Miss Status Holding Registers (MSHRs), an Sv39/Sv48 Translation Lookaside Buffer (TLB) and hardware page table walker, and coherent on-chip bus fabrics via Berkeley's **TileLink** protocol.

---

## 2. Microarchitectural Anatomy: The 5-Stage In-Order Pipeline

Rocket uses a single-issue, in-order 5-stage pipeline with separate instruction and data caches (Harvard microarchitecture):

```text
[ Stage 1: IF (Instruction Fetch) ]
   ├── Program counter generation & reset vector controller
   ├── Branch prediction suite: BTB, 2-bit BHT & Return Address Stack (RAS)
   ├── L1 instruction cache access (tag comparison & data slice)
   └── 16-bit compressed instruction decompressor (RVC expander)
           │
           ▼
[ Stage 2: ID (Instruction Decode & Register Read) ]
   ├── Instruction format slicing & control signal generation
   ├── Physical register file read (dual 64-bit read ports, x0 hardwired)
   ├── Scoreboard long-latency dependency checker & interlock unit
   └── Immediate value generation & sign extender
           │
           ▼
[ Stage 3: EX (Execution & Effective Address Generation) ]
   ├── 64-bit integer arithmetic logic unit (ALU)
   ├── Branch condition evaluation & misprediction comparator
   ├── Address generation unit (AGU: rs1 + imm)
   └── Multi-cycle dispatch arbiter (pipelined multiplier, divider, FPU)
           │
           ▼
[ Stage 4: MEM (Memory Access & Cache Translation) ]
   ├── Non-blocking L1 data cache access (tag check, hit/miss detection)
   ├── Translation Lookaside Buffer (DTLB: virtual-to-physical translation)
   ├── Hardware page table walker interface
   └── Load-store unit MSHR allocation on cache miss
           │
           ▼
[ Stage 5: WB (Architectural Writeback & Retirement) ]
   ├── Writeback arbiter (ALU vs. D-cache vs. RoCC coprocessor vs. FPU)
   ├── Register file synchronous write port commit (rd)
   ├── Control & Status Register (CSR) state update
   └── Precise architectural exception & trap retirement boundary
```

### 2.1 The Front-End (Fetch, RVC & Branch Prediction)

- **Fetch engine:** Fetches a 32-bit or 64-bit word from the L1 instruction cache using physical addresses translated by the ITLB.
- **Compressed decoder:** Instructions using the 16-bit compressed extension (RVC) are expanded combinationally into 32-bit internal instructions at the IF/ID pipeline boundary.
- **Dynamic branch predictor:**
  - **Branch Target Buffer (BTB):** Direct-mapped or set-associative cache storing target addresses for taken branches and jumps.
  - **Branch History Table (BHT):** Array of 2-bit saturating counters providing dynamic direction prediction.
  - **Return Address Stack (RAS):** Tracks function return addresses for `JALR` instructions targeting link register `x1`/`x5`.

### 2.2 Decode & Scoreboarding (ID Stage)

- Decodes the 32-bit instruction and reads source registers `rs1` and `rs2`.
- **Hardware scoreboard:** Rocket features out-of-pipeline units (e.g., iterative hardware divider, non-blocking D-cache misses, floating-point units). The scoreboard tracks pending writes to destination registers (`rd`). If an instruction in Decode requires an operand still being evaluated by a long-latency unit, the decode stage stalls until the data is flagged ready.

### 2.3 Execution & Arithmetic (EX Stage)

- **64-bit integer ALU:** Single-cycle adder/subtractor, bitwise logic unit, and single-cycle barrel shifter.
- **Branch resolution:** Conditional branch comparisons resolve combinationally. If a branch outcome diverges from the IF stage prediction, an immediate pipeline redirect signal is generated.
- **AGU:** Computes virtual memory addresses ($rs1 + \text{imm}$) for all load and store operations.

### 2.4 Non-Blocking Memory Access (MEM Stage)

- **L1 data cache (D$):** Virtually Indexed, Physically Tagged (VIPT) cache. Tag checking occurs concurrently with data array lookup.
- **Hit-under-miss capability:** If a load instruction misses in the cache, the transaction allocates a Miss Status Holding Register (MSHR) and forwards the request over TileLink. Independent downstream instructions that do not depend on the missed data can continue advancing through execution.

### 2.5 Retirement & Writeback (WB Stage)

- Commits the validated result back to the register file `rd`.
- Serves as the precise trap point: interrupts, page faults, or illegal instruction exceptions retire cleanly without altering machine state.

---

## 3. Pipeline Hazards & Bypass Architecture

Rocket mitigates pipeline stalls using multi-level data forwarding and dynamic prediction:

```text
Pipeline Data Forwarding Matrix:
  Stage 3 (EX)   ──► ALU output available
                          │
                          ├─────────────────────────┐
                          ▼                         ▼
  Stage 4 (MEM)  ──► [Forward to EX input]    D-cache hit data available
                          │                         │
                          ├─────────────────────────┼─────────┐
                          ▼                         ▼         ▼
  Stage 5 (WB)   ──► [Forward to EX input]    [Forward to EX input]
```

### 3.1 Data Hazards (RAW Dependencies)

- **ALU-to-ALU forwarding (0-cycle penalty):** Data generated in Stage 3 (EX) is forwarded from the Stage 4 (MEM) and Stage 5 (WB) boundary registers back into the Stage 3 ALU multiplexers. Back-to-back arithmetic instructions execute at 1 cycle per instruction (CPI = 1) without stalls.
- **Load-use data hazard (1-cycle bubble):** Data read from the L1 data cache is not valid until the end of Stage 4 (MEM). If an instruction in Stage 2 (ID) depends directly on the result of an active load instruction in Stage 3 (EX), the core injects exactly one stall bubble into the pipeline.

### 3.2 Control Hazards

- **Branch penalty:** Branches evaluate in Stage 3 (EX).
  - If predicted correctly by the BTB/BHT: 0 stall cycles.
  - On a branch misprediction: Stages 1 (IF) and 2 (ID) are flushed with synchronous clear strobes, inflicting an exact 2-cycle flush penalty.

---

## 4. Hardware Coprocessor Extensibility: The RoCC Interface

One of Rocket's defining architectural features is the **RoCC (Rocket Custom Coprocessor)** interface. Rather than placing accelerators on a high-latency system bus (like AXI), RoCC connects custom accelerators directly into the CPU pipeline:

```text
               ┌──────────────────────────────────────────────┐
               │                 Rocket Core                  │
               │   [IF] ──► [ID] ──► [EX] ──► [MEM] ──► [WB]  │
               └──────────────┬──────────────────▲────────────┘
                              │ Custom instructions (opcode)
                              │ & register operands (rs1, rs2)
                              ▼                  │
               ┌─────────────────────────────────┴────────────┐
               │           RoCC Hardware Accelerator          │
               │  - Direct L1 D-cache access (PTW/memory)     │
               │  - Dedicated register file writeback (rd)    │
               │  (e.g., Gemmini systolic array / cryptography)│
               └──────────────────────────────────────────────┘
```

- Accelerators read operands directly from CPU registers and commit results into the architectural register file.
- The coprocessor shares the L1 data cache translation units and TLBs, allowing hardware accelerators to access virtual memory safely.

---

## 5. FPGA Synthesis & Physical Hardware Metrics

Baseline utilization across common FPGA families for an RV64GC configuration (64-bit, Integer, Multiply/Divide, Compressed, Atomic, Single/Double Floating Point, 16KB I-cache, 16KB D-cache, Sv39 MMU):

| Target FPGA Architecture | Slice LUTs | Flip-Flops (FF) | Block RAMs (BRAM) | DSP Slices | Max Clock ($F_{\max}$) | CoreMark / MHz |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **AMD/Xilinx Artix-7 (100T)** | ~15,500 – 18,200 | ~11,200 – 13,500 | 24 – 36 | 8 – 16 | ~50 – 65 MHz | ~2.8 – 3.2 |
| **AMD Kintex-7 / UltraScale+** | ~12,800 – 15,100 | ~10,500 – 12,200 | 24 – 36 | 8 – 16 | ~110 – 150 MHz | ~2.8 – 3.2 |
| **Microchip PolarFire (MPF300)** | ~16,200 – 19,500 | ~11,800 – 14,200 | 28 – 42 | 8 – 16 | ~65 – 85 MHz | ~2.8 – 3.2 |

> **Note:** When configured as an ultra-compact 32-bit microcontroller (RV32I without MMU, caches, or FPU), Rocket synthesizes into ~3,500 LUTs, though in-order cores like Ibex or VexRiscv are typically preferred for that footprint tier.

---

## 6. Architectural Trade-Off Analysis

| Metric | Ibex (2-Stage) | Vega ET1031 (3-Stage) | Rocket Core (5-Stage) | SonicBOOM (OoO v3) |
| :--- | :--- | :--- | :--- | :--- |
| **Datapath width** | 32-bit | 32-bit | 64-bit / 32-bit | 64-bit |
| **Execution paradigm** | In-order scalar | In-order scalar | In-order scalar | Out-of-order superscalar |
| **Operating system** | Bare-metal / RTOS | Bare-metal / RTOS | Full Linux / Unix (MMU) | Full Linux / server-class |
| **L1 cache architecture** | None / fixed buffer | Optional blocking | Non-blocking (MSHRs) | Non-blocking (multi-port) |
| **Bypass complexity** | None (single exec) | 1-level (MEM → EX) | 2-level (MEM & WB → EX) | Dynamic CDB broadcast |
| **Interconnect standard** | Native / OBI | AHB-Lite / AXI4 | TileLink (coherent) | TileLink (coherent) |

### Strengths

- **Complete software compatibility:** Boots unmodified Debian, Fedora, and OpenWRT Linux distributions with mature MMU, privilege mode isolation, and standard device trees.
- **Proven silicon maturity:** Fabricated across hundreds of commercial and academic tapeouts globally, serving as the benchmark standard against which novel RISC-V extensions are evaluated.
- **TileLink coherency:** Native support for multi-core symmetric multiprocessing (SMP) with directory-based or snooping cache coherence agents.

### Limitations

- **Chisel HDL overhead:** Synthesizes flattened, auto-generated Verilog netlists with obfuscated wire names, making post-synthesis waveform inspection in GTKWave more challenging than handwritten RTL.
- **High FPGA slice consumption:** Cache controllers, TLB page table walkers, and TileLink crossbars require significant LUT and BRAM resources on lower-tier FPGA boards.

---

## 7. Primary Target Use Cases

1. **Linux-capable edge application processors:** Network gateways, smart industrial controllers, and application platforms running high-level operating systems.
2. **Hardware accelerator evaluation:** Prototyping custom crypto, AI/ML, or homomorphic encryption instructions tightly coupled via the RoCC interface.
3. **Academic semiconductor tapeouts:** Providing a verified, low-risk processor subsystem on multiproject wafer (MPW) runs from 180nm down to 3nm process nodes.

---

## 8. References

1. **Rocket Chip generator repository:** <https://github.com/chipsalliance/rocket-chip>
2. **TileLink specification:** <https://chipsalliance.github.io/tilelink-spec/>
3. **UC Berkeley architecture research:** <https://bar.eecs.berkeley.edu/>
