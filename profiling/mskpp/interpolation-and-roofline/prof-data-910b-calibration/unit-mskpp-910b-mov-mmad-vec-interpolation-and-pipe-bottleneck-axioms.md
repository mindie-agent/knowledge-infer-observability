---
schema: mindie-graded-unit/1
id: unit-mskpp-910b-mov-mmad-vec-interpolation-and-pipe-bottleneck-axioms
revision: 1
title: "Ascend 910B msKPP Hardware Profile Calibration Tables & Linear Interpolation Axioms (MovType, MmadType, VecType)"
tier: L1
category: axiom
evidence_strength: software_contract
status: active
domain_path: profiling/mskpp/interpolation-and-roofline/prof-data-910b-calibration
inherits_from:
  - unit-msprof-op-pmu-metrics-and-a5-vector-roofline-axioms
conditions:
  hardware:
    - Ascend 910B
  domain: profiling
triggers:
  error_signatures:
    - GetMovTypeData
    - GetMmadTypeData
    - GetVecTypeData
  keywords:
    - mskpp
    - resource910b
    - linear-interpolate
    - mov-type
    - mmad-type
    - vec-type
    - ops-per-cycle
provenance:
  source_type: source_contract
  source_locator: Ascend/mskpp@5fcea15fdddf0bca9aa72745b07fea6d3093e555:csrc/prof_data/resource910b.h
  evidence_refs:
    - https://gitcode.com/Ascend/mskpp/blob/5fcea15fdddf0bca9aa72745b07fea6d3093e555/csrc/prof_data/resource910b.h#L17-L86
    - https://gitcode.com/Ascend/mskpp/blob/5fcea15fdddf0bca9aa72745b07fea6d3093e555/csrc/prof_data/linearInterpolate.h#L1-L45
  supersedes: []
---

## Hardware Performance Modeling Axioms in `msKPP` (`csrc/prof_data/resource910b.h`)

Unlike cycle-accurate instruction simulators (`CAModel`) that emulate internal microarchitectural FIFOs cycle-by-cycle, `msKPP` achieves sub-second kernel performance prediction on Ascend 910B (`Ascend910B1`..`Ascend910B4`) by interpolating empirical hardware microbenchmark tables across three instruction families:

1. **Data Movement Instructions (`MovType` & `MovTypeRepeat`)**:
   - MTE transfer instructions (`GM_TO_L1`, `L1_TO_L0A`, `L1_TO_L0B`, `L0C_TO_GM`, `GM_TO_UB`, `UB_TO_GM`) are parameterized by `MovType`:
     - Independent variable: `numBursts` (or `repeat` in `MovTypeRepeat`).
     - Dependent empirical metric: achieved `bandwidth` ($\text{GB/s}$), alongside `totalData` ($\text{GB}$), `taskTime` ($\mu\text{s}$), `totalCycles`, `l1WriteBw`, and `mainMemReadBw`.
   - For any transfer of byte volume $V$ with burst count $B$, `msKPP` interpolates effective bandwidth $BW(B)$ from `GetMovTypeData(opName)` (or `GetMovRepeatTypeData(opName)`) using piecewise linear interpolation (`linearInterpolate.cpp`). Short bursts ($B \ll B_{\text{sat}}$) are latency-bound by MTE command startup overhead and achieve a fraction of peak HBM/L1 bandwidth.
2. **Cube Matrix Multiply-Accumulate Instructions (`MmadType`)**:
   - Cube `MMAD` instructions are parameterized by `MmadType`:
     - Shape dimensions $(m, k, n)$ and their product `mknSum` $= m \times k \times n$.
     - Dependent empirical metric: `calPerf` in **$\text{OPs/Cycle}$** (`GetMmadTypeData(opName)`).
   - Cube instruction cycle consumption is derived from $\text{Cycles}_{\text{mmad}} = (2 \cdot m \cdot k \cdot n) / (\text{calPerf}(m \cdot k \cdot n) \times \text{scale})$, capturing prologue/epilogue pipeline fill loss when $m \cdot k \cdot n$ is small relative to the $16 \times 16 \times 16$ Cube array throughput.
3. **Vector Compute Instructions (`VecType`)**:
   - Vector instructions (`VADD`, `VMUL`, `VEXP`, `VRSQRT`, `VCONV`, `VREDUCE`, etc.) are parameterized by `VecType`:
     - Independent variables: `repeat` count and total element count `numSum`.
     - Dependent empirical metric: `calPerf` ($\text{OPs/Cycle}$) and `totalCycles` (`GetVecTypeData(opName)`).
   - Small vector `repeat` counts suffer issue-pipeline startup overhead (`calPerf` far below peak Vector width), whereas large `repeat` counts saturate the Vector execution pipe.

## Pipeline Bottleneck Attribution Corollary

- Because MTE2 (`GM -> L1/UB`), MTE1 (`L1 -> L0A/L0B`), Cube (`MMAD`), Vector (`VEC`), and MTE3 (`UB/L0C -> GM`) execute on independent asynchronous hardware queues synchronized by hardware event flags, total tiled loop duration is lower-bounded by:
  $$T_{\text{tiled}} \ge \max\big(\sum T_{\text{MTE2}},\; \sum T_{\text{MTE1}},\; \sum T_{\text{Cube}},\; \sum T_{\text{Vector}},\; \sum T_{\text{MTE3}}\big)$$
- Increasing tile sizes $(m, k, n)$ increases both `numBursts` (pushing `MovType` toward peak bandwidth) and `mknSum` / `numSum` (pushing `MmadType` / `VecType` toward peak $\text{OPs/Cycle}$), up to the physical capacity limits of `L1` ($512\text{ KB}$), `L0A`/`L0B` ($64\text{ KB}$), `L0C` ($128\text{ KB}$), and `UB` ($192\text{ KB}$).
