# Pipelined INT8 Systolic-Array AI Accelerator — RTL to GDSII

> Design and 32nm physical implementation of a high-performance 8x8 INT8 systolic-array AI accelerator with pipelined Processing Elements.

![Verilog](https://img.shields.io/badge/RTL-Verilog-blue?style=flat-square)
![Node](https://img.shields.io/badge/Node-32nm_SAED-orange?style=flat-square)
![Freq](https://img.shields.io/badge/Freq-500MHz_2ns-green?style=flat-square)
![Tools](https://img.shields.io/badge/Tools-VCS_DC_ICC2-red?style=flat-square)
![Flow](https://img.shields.io/badge/Flow-RTL_to_GDSII-purple?style=flat-square)

**Author:** Galipelli Sai Raghava — M.Tech VLSI System Design, BVRIT | ASIC Physical Design Engineer
📧 galipelli.sairaghava@gmail.com | 💼 [LinkedIn](https://www.linkedin.com/in/sairaghava)

---

## 📌 Overview

AI/ML workloads are dominated by multiply-accumulate (MAC) operations for matrix multiplication in CNNs, fully-connected layers, and Transformers. General-purpose CPUs cannot exploit this parallelism efficiently.

This project implements an **8x8 systolic array with 64 Processing Elements (PEs)** for signed INT8 matrix multiplication with 32-bit accumulation. The key optimization is **PE-level pipelining** to break the long MAC critical path.

**Baseline problem:** Multiplier + sign-extension + 32-bit adder + accumulator setup in *same* cycle = 4.93ns critical path, only 202.8 MHz.

**Proposed solution:** Insert `product_pipe` register between multiplier and accumulator. Split into 2 stages:
- Stage 1: `product_pipe <= a_in * w_in`
- Stage 2: `acc <= acc + product_pipe`

> Result: **63.7% critical-path reduction, 146.5% frequency improvement, 2.36x throughput**, with complete RTL-to-GDSII signoff.

## 🎯 Key Specs

| Parameter | Value |
| :--- | :--- |
| Array | 8x8 = 64 PEs |
| Precision | Signed INT8 in, Signed 32-bit psum out |
| Technology | SAED 32nm LVT/HVT/RVT |
| Clock Target | 2.00 ns / 500 MHz |
| Top Module | `accel_top` |
| Control | LOAD-COMPUTE-DRAIN-DONE FSM, 22 cycles baseline → 23 cycles pipelined |
| Tools | VCS, Verdi, Design Compiler, IC Compiler II |
| Final Output | Routed layout + GDSII + 0 DRC |

## 🏗️ Architecture

**Top level (`accel_top.v`):** `accel_regs.v` + `systolic_array_8x8.v`
- CPU/Testbench → `accel_regs` (CTRL/STATUS/CONFIG/A/B buffers/skewed stream generator) → 8x8 PE grid → Result Matrix C
- Single clock domain, active-low `rst_n`, `irq` output

**PE Interface (`systolic_pe.v`):**
`clk, rst_n, clear, compute_en, a_in[7:0], w_in[7:0], a_out[7:0], w_out[7:0], psum_out[31:0]`
- Activations flow left→right, Weights flow top→bottom, Partial-sums drain bottom
- Baseline: `acc <= acc + (a_in * w_in)` in 1 cycle
- Pipelined: `product_pipe <= a_in * w_in` then `acc <= acc + product_pipe`

**Register Map:**
`CTRL 0x000, STATUS 0x004, CONFIG 0x008, A_MAT 0x100-0x13C, B_MAT 0x200-0x23C, RESULT 0x300-0x3FC`

Put your diagrams in `/images`: `fig3.1-array.png`, `fig3.2-baseline-pe.png`, `fig4.2-pipelined-pe.png`, `fig5.1-top-block.png`

## 🔄 RTL-to-GDSII Flow Followed

1. **RTL + Functional Sim (VCS):** `vcs -v ./../rtl/accel_top.v ./../rtl/tb_accel_top.v -kdb -full64 -lca -debug_access+all` → PASS=65 FAIL=0, DONE detected
2.  **Synthesis (DC `compile_ultra`):** 2ns SDC, `report_qor/timing/area/power`
3.  **Floorplan (ICC2):** `initialize_floorplan -core_utilization 0.6 -core_offset 20` + `place_pins -self`
4.  **Powerplan:** M7/M8 5um ring, M6/M7/M8 mesh (w2.4/2.0um, pitch 30/25um), M1 0.06um rails
5.  **Placement + Pre-CTS Opt:** density/congestion check, 0% overflow
6.  **CTS + Opt:** latency 0.27ns, skew 0.22ns
7.  **Routing + Post-route Opt:** 344k wires, 18k contacts
8.  **Physical Verification:** DRC/LVS/PG checks → GDSII `write_gds`

## 📊 Results

### Synthesis (DC, 2.0ns constraint)

| Metric | Baseline | Proposed Pipelined | Improvement |
| :--- | :--- | :--- | :--- |
| Critical Path | 4.93 ns | 1.79 ns | **63.7% ↓** |
| Max Freq | 202.8 MHz | 500 MHz | **146.5% ↑** |
| WNS | +0.02 ns | +0.09 ns | 4.5x better |
| TNS / Violations | 0 / 0 | 0 / 0 | Clean |
| Total Cell Area | 118,482 µm² | 206,897 µm² | +75.1% (pipeline regs) |
| Total Power | 7.91 mW | 8.60 mW | +8.7% |
| Cells | ~35.8k | 36,438 (25431 comb + 7448 seq + 3398 buf/inv) | — |

Proposed area: Comb 71239.35 + Non-comb 53000.20 = 124239.56 cell area, 166238.16 design area.

### Physical Design (ICC2)

| Metric | Baseline | Proposed | Note |
| :--- | :--- | :--- | :--- |
| Core Area | 191,627 µm² | 217,894 µm² | +13.7% |
| Utilization | 57.3% | 64.0% | Higher density |
| Std-cell Count | 35,872 | 39,991 | +11.5% |
| Setup WNS | +0.63 ns | +1.87 ns | **2.97x better** |
| Hold WNS | +0.01 ns | +0.02 ns | Clean |
| Clock Latency / Skew | 0.36 / 0.11 ns | 0.41 / 0.14 ns | Comparable |
| Total Power | 71.4 mW | 83.6 mW | +17.1% |
| Wirelength | 918,247 µm | 1,072,435 µm | +16.8% |
| DRC Signoff | 0 | 0 | Pass |
| GDSII | Yes | Yes | Done |

**Performance:** 8x8 MatMul = 22 cycles @202.8MHz = 108.48ns baseline vs 23 cycles @500MHz = 46.00ns proposed = **57.6% less execution time, 2.36x throughput.**

Verification: Placement legality 0 violations, PG DRC No errors, Route DRC 0, PG missing vias 0.

## 📁 Repo Structure
