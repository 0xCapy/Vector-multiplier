# Pipelined 8-bit Vector MAC

A configurable Verilog RTL datapath for **unsigned 8-bit vector dot products** on FPGA. The design combines pipelined Wallace-tree multipliers, a binary reduction tree, and a shared 32-bit accumulator.

**[Technical Report (PDF)](./Vector_MAC_report.pdf)**

## Architecture

![Vector MAC architecture and module-level test hierarchy](./doc/hi_dia.png)

- **Parallelism:** 1, 4, 8, or 16 lanes, selected at elaboration through `ACTIVE_LANES`.
- **Arithmetic:** unsigned 8-bit operands, 16-bit products, and 32-bit accumulated results.
- **Vector length:** `ELEMS` parameter (default: 1,000 elements).
- **Control:** valid-driven pipeline; no ready/backpressure interface. Unused lanes in a partial final beat must be zero-padded.

## FPGA Results

Historical implementation results targeting the **Xilinx Artix-7 XC7A100T**:

| Lanes | Reported Fmax | Peak MAC rate |
| ---: | ---: | ---: |
| 1 | 211 MHz | 0.211 GMAC/s |
| 4 | 217 MHz | 0.868 GMAC/s |
| 8 | 209 MHz | 1.672 GMAC/s |

Peak MAC rate is calculated as `lanes × Fmax` and excludes vector startup, drain, and data-transfer overhead. These figures are taken from the original implementation study, **not a new implementation run**; see the [technical report](./Vector_MAC_report.pdf) for methodology, area-efficiency comparisons, and limitations. The 16-lane configuration has RTL support but no comparable implementation result in the study.

## Repository

| Path | Contents |
| --- | --- |
| [`src/`](./src/) | Wallace multiplier, lane wrappers, adder tree, accumulator, and top-level RTL |
| [`sim/`](./sim/) | Module-level and integrated Verilog testbenches |
| [`constr/`](./constr/) | Existing FPGA timing constraints |
| [`doc/`](./doc/) | Architecture and performance figures |
| [`Vector_MAC_report.pdf`](./Vector_MAC_report.pdf) | Design, verification, and FPGA implementation analysis |
