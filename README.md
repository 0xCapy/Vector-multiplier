# Pipelined 8-bit Vector MAC Accelerator (Verilog)

**University of Sheffield — EEE232 Coursework 2 (2025)**  
A configurable, FPGA-oriented vector multiply–accumulate design built from 8-bit Wallace-tree multipliers, a pipelined adder tree, and a 32-bit accumulator.

## At a glance

| Item | Current implementation |
| --- | --- |
| Language | Verilog RTL; Verilog/SystemVerilog testbenches |
| Computation | Unsigned 8-bit vector dot product |
| Parallelism | 1, 4, 8, or 16 multiplier lanes (elaboration-time selection) |
| Multiplier | Custom 8 × 8 partial-product compressor with three registered stages |
| Reduction | Pipelined binary adder tree |
| Accumulation | 32-bit running sum; default vector length: 1,000 elements |
| Interface | `vec_valid` input and `result_valid` output; no backpressure |
| Original FPGA target | Xilinx Artix-7 XC7A100T-1CSG324 |
| Original design environment | Vivado 2021.1 (archived coursework) |

## 1. Design objective

The accelerator computes an unsigned vector dot product:

$$
S = \sum_{i=0}^{L-1} A_i B_i, \qquad A_i, B_i \in [0,255].
$$

The default vector length is **L = 1,000**. A configurable number of 8 × 8 multipliers process one input beat in parallel. Their products are reduced through an adder tree and accumulated across beats into a 32-bit result.

The hardware is **unsigned**, despite the term “INT8” used in some original coursework notes. The current RTL does **not** implement signed two's-complement arithmetic or dedicated sparse-data skipping.

## 2. Architecture

![Vector MAC architecture](doc/hi_dia.png)

The main datapath is:

```text
vec_a / vec_b (128-bit packed inputs)
                 |
       ACTIVE_LANES = 1 / 4 / 8 / 16
                 |
      Parallel 8x8 Wallace multipliers
                 |
        Pipelined adder tree
                 |
          Per-beat partial sum
                 |
      32-bit accumulator + beat counter
                 |
       result_sum / result_valid
```

### Key blocks

| Source | Responsibility |
| --- | --- |
| [`src/wallace_tree_multiplier.v`](src/wallace_tree_multiplier.v) | Partial-product generation, compression using half/full adders, and a three-stage pipelined 8 × 8 multiplier |
| [`src/mul1x8x8_wallace.v`](src/mul1x8x8_wallace.v) | 1-lane multiplier + reduction wrapper |
| [`src/mul4x8x8_wallace.v`](src/mul4x8x8_wallace.v) | 4-lane multiplier + reduction wrapper |
| [`src/mul_8x8x8_wallace.v`](src/mul_8x8x8_wallace.v) | 8-lane multiplier + reduction wrapper |
| [`src/mul16x8x8_wallace.v`](src/mul16x8x8_wallace.v) | 16-lane multiplier + reduction wrapper |
| [`src/adder_tree_var.v`](src/adder_tree_var.v) | Pipelined reduction tree whose depth depends on lane count |
| [`src/accumulator.v`](src/accumulator.v) | Accumulation across input beats and generation of end-of-vector valid pulse |
| [`src/vector_mac_top.v`](src/vector_mac_top.v) | Lane-mode selection and integration of datapath and accumulator |

**Design choice:** `ACTIVE_LANES` selects one of four existing lane-specific implementations at elaboration time. It is **not** a runtime-controlled lane count and the current design does not yet use a single generated lane-array wrapper.

### Top-level interface

The top module is `vector_mac_top_param` in `src/vector_mac_top.v`.

| Signal / parameter | Width | Meaning |
| --- | ---: | --- |
| `clk` | 1 | Clock |
| `rst_n` | 1 | Active-low asynchronous reset |
| `vec_valid` | 1 | Qualifies an input beat |
| `vec_a`, `vec_b` | 128 each | Packed bytes; only the lowest `ACTIVE_LANES` bytes are used |
| `result_valid` | 1 | Pulses when a complete dot product has been accumulated |
| `result_sum` | 32 | Unsigned accumulated dot-product result |
| `ELEMS` | parameter | Number of elements per vector (default `1000`) |
| `ACTIVE_LANES` | parameter | Supported modes: `1`, `4`, `8`, `16` (default `8`) |

For a vector containing `ELEMS` elements, the accumulator expects `ceil(ELEMS / ACTIVE_LANES)` valid beats. **Unused lanes in the final beat must be zero-padded by the source.** For example, the last beat of a 1,000-element vector in 16-lane mode contains 8 valid elements and 8 zero-padded lanes. The current top-level interface has no per-lane mask.

There is no `ready` signal: the interface does not support backpressure. Each lane mode has its own internal register depth, so do not assume one common fixed latency across configurations without checking the waveform.

## 3. Simulation without Vivado

The source-level simulations can be run with [Icarus Verilog](https://steveicarus.github.io/iverilog/) (for example, `brew install icarus-verilog` on macOS). Vivado is **not** required to compile or simulate the RTL.

```bash
git clone https://github.com/0xCapy/Vector-multiplier.git
cd Vector-multiplier
mkdir -p build

# Run the existing vector-level testbench in 4-lane mode.
iverilog -g2012 \
  -s tb_vector_mac_top_para \
  -Ptb_vector_mac_top_para.LANE_P=4 \
  -o build/vector_mac_4.vvp \
  src/*.v sim/tb_vector_mac_top.v

vvp build/vector_mac_4.vvp
```

To try a different configuration, change `LANE_P=4` to `1`, `8`, or `16`. The existing testbench instantiates `vector_mac_top_param` with the corresponding `ACTIVE_LANES` parameter.

**Verification status:** The repository includes directed, random, and stress-oriented testbenches in [`sim/`](sim/). The above command is a suggested way to invoke an existing testbench; it has not been rerun as part of this README-only update. Some older benches target legacy modules and are not part of this quick start. Current benches also need explicit non-zero process exit codes on failure, watchdog timeouts, and better end-of-vector padding checks before results should be described as a fully automated regression suite.

## 4. Archived FPGA results

The original README reported the following coursework figures for an Artix-7 implementation using Vivado 2021.1:

| Previously reported metric | Value | Context |
| --- | ---: | --- |
| 8-lane throughput | ~1.67 GMAC/s | Derived from 8 multiplications per cycle × 209 MHz |
| 8-lane clock frequency | 209 MHz | Historical reported build |
| Best throughput per CLB | 6.5 MMAC/s/CLB | Historical reported 4-lane configuration |

These are **archived figures, not newly reproduced measurements**. Before using them in a CV or claiming timing closure, the original build configuration, synthesis/implementation reports, clock constraints, definitions of MAC/s and CLB utilization, and power-estimation method must be checked.

![Original coursework resource/performance summary](doc/data_dia.png)

| Throughput efficiency | Estimated power efficiency |
| --- | --- |
| ![Archived throughput efficiency](doc/eff_dia.png) | ![Archived power efficiency](doc/pow_dia.png) |

The repository includes [`constr/mul_clock.xdc`](constr/mul_clock.xdc), but its current signal names and timing exceptions require review against `vector_mac_top_param`. The XDC should not be treated as validated timing sign-off for the current top level.

## 5. Design considerations and limitations

- **Parallelism versus area:** More multiplier lanes can increase peak work per cycle, but also increase LUT/FF use and routing demand.
- **Pipelining:** Registered multiplication and reduction stages target high clock frequency while introducing latency. Lane-specific wrappers currently have different depths.
- **Arithmetic:** Inputs are unsigned bytes; signed INT8 support would need an explicit RTL and verification change.
- **Non-multiple vector lengths:** The source must zero-pad inactive bytes on a partial final beat; there is no dedicated per-lane validity mask.
- **Verification:** The existing top-level testbench computes expected sums from the same driven beats, so it does not independently enforce a 1,000-element tail-padding contract in 16-lane mode.
- **Reproducibility:** There is no `Makefile` or automated Vivado batch script in the current repository. Older instructions referring to `make sim` or `make vivado` are not applicable.

## 6. Repository layout

```text
Vector-multiplier/
├── src/                  Verilog RTL
├── sim/                  Existing Verilog/SystemVerilog testbenches
├── constr/               Original FPGA clock constraints (review required)
├── doc/                  Architecture and archived results images
├── DesignReport.pdf      Original coursework report
└── README.md
```

## 7. Project provenance

This project originated as **EEE232 Coursework 2 at the University of Sheffield**. The current repository captures the original design plus its existing documentation and testbenches. A future revision can add reproducible simulation regression scripts, refined RTL parameterization, corrected tail handling, and freshly generated FPGA implementation reports.
