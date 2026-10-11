# Vega Microprocessors (C-DAC India): The 3-Stage In-Order RISC-V Core

## 1. Introduction

The **Vega** processor family was developed by the **Centre for Development of Advanced Computing (C-DAC)** under the Government of India's Digital India Microprocessor Program (Microprocessor Development Programme, MDP). It represents India's indigenous silicon family, spanning low-power IoT microcontrollers up to Linux-capable 64-bit application processors.

Instead of a single monolithic core, the Vega platform is partitioned into distinct microarchitectural tiers:

- **Vega ET1031:** 32-bit (RV32I/IM), 3-stage in-order pipeline targeting embedded sensors and low-power IoT.
- **Vega AS1061 / AS2161:** 64-bit (RV64GC), 5-stage in-order dual-issue/pipelined core targeting industrial automation, edge gateways, and indigenous embedded SoCs (such as the Aries v3 board).

This document focuses on the **Vega ET1031** 3-stage microarchitecture, as it directly matches the pipeline depth and power-constrained footprint of high-efficiency embedded control cores.

---

## 2. Microarchitectural Anatomy: The 3-Stage Pipeline (Vega ET1031)

Vega ET1031 uses a classical, compact 3-stage datapath that balances cycle time against instruction latency:

```text
[ Stage 1: IF (Instruction Fetch) ]
   - Next-PC generation, branch target latching, reset vector control
   - 32-bit memory request generator (AHB-Lite / AXI4-Lite instruction interface)
   - Synchronous fetch response buffer
                      │
                      ▼
[ Stage 2: ID / EX (Instruction Decode & Execution) ]
   - 32x32 register file read ports
   - Instruction decoder & sign-extension immediate generator (Imm-Gen)
   - RV32I arithmetic logic unit (ALU)
   - Branch comparator & early branch target calculation
   - Iterative / pipelined multiplier-divider unit (MDU)
   - Load-store address generator (AGU)
                      │
                      ▼
[ Stage 3: MEM / WB (Memory Access & Writeback Commit) ]
   - Data memory access controller (load/store alignment: byte, halfword, word)
   - Writeback arbiter & multiplexer (ALU result vs. memory load data)
   - Register file synchronous write port commit
```

### 2.1 Detailed Stage Operations

- **Instruction Fetch (IF stage):**
  - Computes the program counter sequentially (`PC + 4`) or latches non-sequential branch/jump/trap targets from Stage 2.
  - Interfaces with synchronous on-chip SRAM via a dedicated AHB-Lite instruction bus.
  - Flushes the pre-fetched word in the event of an evaluated branch redirect from Stage 2.
- **Decode / Execution (ID/EX stage):**
  - Operands `rs1` and `rs2` are read combinationally or via early clocked arrays from the register file.
  - **ALU operations:** Arithmetic, logical, and shift operations execute in 1 cycle.
  - **Branch resolution:** Branch condition evaluation (e.g., `BEQ`, `BNE`, `BLT`) and branch target calculation occur in Stage 2. If a branch is taken, a redirect signal is sent combinationally to Stage 1, resulting in a 1-cycle branch penalty.
  - **Address calculation:** For loads and stores, effective memory addresses ($rs1 + \text{imm}$) are calculated combinationally and forwarded to Stage 3.
- **Memory / Writeback (MEM/WB stage):**
  - Drives the data bus for `LW`, `SW`, and sub-word access with proper sign/zero extension (`LB`, `LBU`, `LH`, `LHU`).
  - The final result (ALU computation or memory read data) is multiplexed and committed back to the register file destination index `rd`.

---

## 3. Hazard Handling & Interlock Mechanics

Because Vega ET1031 splits execution from memory access, it requires active forwarding and interlocking logic.

### 3.1 ALU-to-ALU Data Forwarding

- Without forwarding, an instruction directly following an arithmetic operation would stall waiting for Stage 3 writeback.
- Vega integrates a forwarding path from Stage 3 (MEM/WB) back into the Stage 2 (ID/EX) ALU inputs.
- Back-to-back arithmetic instructions execute at 1 CPI (0 stall cycles):

```asm
add x1, x2, x3   # In MEM/WB
sub x4, x1, x5   # In ID/EX (receives x1 forwarded from Stage 3) -> 0 stalls
```

### 3.2 Load-Use Data Hazard

If an instruction in Stage 2 depends on the destination register of a load instruction currently in Stage 3:

```asm
lw  x1, 0(x2)    # In MEM/WB
add x3, x1, x4   # In ID/EX (needs x1, but memory data is not ready yet)
```

The hardware hazard unit detects the following condition and triggers a 1-cycle interlock stall. Stage 1 and Stage 2 freeze, injecting a 1-cycle bubble into Stage 3.

```text
(id_ex_mem_read) && ((id_ex_rd == if_id_rs1) || (id_ex_rd == if_id_rs2))
```

### 3.3 Control Hazards (Branches / JAL / JALR)

Because branch comparisons resolve in Stage 2, the penalty for taken branches is strictly 1 cycle. The fall-through instruction already fetched in Stage 1 is invalidated by asserting a synchronous flush line.

---

## 4. Peripheral & Bus Subsystems (Aries SoC Integration)

- **Interconnect architecture:**
  - The Vega core is integrated using an AHB-Lite / AXI4 bus matrix.
  - Features separate instruction and data buses (Harvard architecture at the core boundary) converging on an internal crossbar to connect to unified on-chip BRAM.
- **MDU (multiplication/division extension, RV32M):**
  - Configurable between a multi-cycle iterative serial divider/multiplier (for small FPGA footprints) and a pipelined DSP-assisted multiplier.
- **Interrupt subsystem:**
  - Implements the standard RISC-V Machine-Mode timer interrupt (`MTIE`), software interrupt (`MSIE`), and external interrupt (`MEIE`) with a vectored/direct trap controller (`mtvec`).

---

## 5. Implementation Metrics (FPGA Synthesis Baseline)

For the Vega ET1031 (RV32IM, 3-stage configuration):

| Target FPGA Architecture | LUTs | Flip-Flops | Block RAMs | DSP Blocks | Max Frequency ($F_{\max}$) | CoreMark / MHz |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **AMD/Xilinx Artix-7** | ~1,800 – 2,400 | ~1,200 – 1,500 | 0 – 2 | 0 – 3 | ~100 – 130 MHz | ~1.7 – 2.1 |
| **Microchip PolarFire** | ~2,000 – 2,600 | ~1,300 – 1,600 | 0 – 2 | 0 – 3 | ~90 – 115 MHz | ~1.7 – 2.1 |
| **Intel Cyclone 10 LP** | ~2,200 – 2,900 | ~1,400 – 1,700 | 0 – 2 | 0 – 3 | ~85 – 110 MHz | ~1.7 – 2.1 |

---

## 6. Architectural Trade-Off Analysis

| Metric | PicoRV32 (Multi-Cycle) | Ibex (2-Stage) | Vega ET1031 (3-Stage) | Rocket Core (5-Stage) |
| :--- | :--- | :--- | :--- | :--- |
| **Datapath width** | 32-bit | 32-bit | 32-bit | 32-bit / 64-bit |
| **Execution model** | Multi-cycle FSM | 2-stage in-order | 3-stage in-order | 5-stage in-order |
| **Average CPI** | ~3.0 – 4.0 | ~1.2 – 1.6 | ~1.1 – 1.3 | ~1.0 – 1.2 |
| **Branch penalty** | 0 cycles | 1 cycle | 1 cycle | 2 cycles |
| **ALU-to-ALU stalls** | 0 | 0 | 0 (via forwarding) | 0 (via forwarding) |
| **Critical path location** | FSM next-state | ID/EX (read + ALU) | MEM/WB (bus + align) | Clock tree / cache tag |

### Strengths

- **Balanced logic depth:** Breaking fetch, execution, and memory into three distinct physical registers avoids the long critical path of Ibex's unified ID/EX/WB stage while keeping area far smaller than a 5-stage pipeline.
- **High IPC for control code:** 1-cycle branch resolution combined with 0-cycle ALU-to-ALU forwarding gives excellent IPC on standard embedded C workloads.
- **National ecosystem deployment:** Validated across Indian national silicon tapeouts and fully integrated with domestic BSPs (Board Support Packages) and SDKs (such as Aries v3).

### Limitations

- **Moderate frequency ceiling:** On low-power FPGA fabrics, the combinational path combining branch comparator evaluation and next-PC muxing can limit clock speeds to ~100–120 MHz.
- **Resource overhead vs. 2-stage:** Requires an explicit forwarding mux matrix and hazard detection unit that 2-stage cores largely bypass.

---

## 7. Primary Target Use Cases

1. **Indigenous IoT & sensor hubs:** Smart metering, agricultural monitoring nodes, and embedded environmental sensors (Aries microcontroller ecosystem).
2. **Industrial control systems:** PLC units and automation modules requiring standard RISC-V compliance with deterministic, low-latency execution.
3. **Strategic & defense electronics:** Trusted domestic microcontrollers free from undisclosed proprietary silicon IP blocks.
