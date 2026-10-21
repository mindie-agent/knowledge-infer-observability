---
schema: mindie-graded-unit/1
id: unit-mskpp-chip-tensor-mmad-hierarchy-and-cycle-prediction-sop
revision: 1
title: "MindStudio Kernel Performance Prediction (msKPP) Chip DSL, Memory Hierarchy & Tiling Prediction SOP"
tier: L2
category: sop
evidence_strength: software_contract
status: active
domain_path: profiling/mskpp/dsl-and-pipe-modeling/mmad-mov-vec-prediction
inherits_from:
  - unit-msprof-op-pmu-metrics-and-a5-vector-roofline-axioms
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: profiling
triggers:
  error_signatures:
    - instruction_cycle_consumption.html
    - L1-TO-L0A EVENT
    - mmad invalid param
  keywords:
    - mskpp
    - kernel-performance-prediction
    - mmad
    - l0a
    - l0b
    - l0c
    - tiling-design
    - task-schedule
provenance:
  source_type: source_contract
  source_locator: Ascend/mskpp@5fcea15fdddf0bca9aa72745b07fea6d3093e555:test/cases/test_mmad.py
  evidence_refs:
    - https://gitcode.com/Ascend/mskpp/blob/5fcea15fdddf0bca9aa72745b07fea6d3093e555/README_EN.md#L25-L53
    - https://gitcode.com/Ascend/mskpp/blob/5fcea15fdddf0bca9aa72745b07fea6d3093e555/test/cases/test_mmad.py#L20-L123
  supersedes: []
---

## Preconditions & Trigger

Apply this SOP when using **MindStudio Kernel Performance Prediction (`msKPP`)** (`from mskpp import Chip, Tensor, mmad, ...`) to estimate AscendC / custom operator execution time, MTE/Cube/Vector pipeline overlap, and tiling efficiency in seconds before writing or running cycle-level CAModel simulations.

## Ascend Cube Memory Hierarchy & Dataflow Contract in `msKPP`

1. **Strict On-Chip Storage Hierarchy (`GM -> L1 -> L0A/L0B/BT -> L0C -> GM`)**:
   - `msKPP` models the physical Ascend AI Core memory hierarchy and enforces legal MTE transfer paths via `Tensor.load()`:
     - **Global Memory (`GM`) to `L1`**: Input tiles (`gm_x`, `gm_y`, and optional `gm_bias`) must first be loaded from `"GM"` into `"L1"` (`l1_x = Tensor("L1", format="NZ"); l1_x.load(gm_x)`).
     - **`L1` to Cube Input Buffers (`L0A`, `L0B`, `BT`)**: Matrix $A$ slices load from `"L1"` into `Tensor("L0A")`, Matrix $B$ slices load from `"L1"` into `Tensor("L0B")`, and bias vectors load from `"L1"` into Bias Table `Tensor("BT")`.
     - **Cube Matrix Multiply-Accumulate (`mmad` -> `L0C`)**: `mmad(x_l0a, y_l0b, z_l0c_or_bt, is_inited)()` executes on the Cube unit and outputs its accumulator tile in `"L0C"` (typically `"FP32"` in fractal `"NC1HWC0"` or `"NZ"` format), which is then written back to `"GM"` via `in_z.load(out_z)`.
   - **Dependency Causal Ordering**: `task_schedule` automatically chains pipeline events so the $k$-th `L1-TO-L0A` transfer cannot start until the corresponding $k$-th `GM-TO-L1` transfer completes.
2. **Fractal Alignment Constraint**:
   - Cube `mmad` dimensions $(M, K, N)$ must align with hardware fractal tile granularity (multiples of `16`); passing unaligned non-fractal shapes with invalid initialization flags raises a schedule validation exception.

## Authoring a Tiling Evaluation Script & Generating Pipeline Traces

1. **Instantiate `Chip` Context & Enable Trace/Metrics Output**:
   ```python
   from mskpp import Chip, Tensor, mmad
   from mskpp._C import task_schedule

   with Chip("Ascend910B3") as chip:
       chip.enable_trace()
       chip.enable_metrics()
       in_x = Tensor("GM", "FP16", [32, 48], format="ND")
       in_y = Tensor("GM", "FP16", [48, 16], format="ND")
       in_z = Tensor("GM", "FP32", [32, 16], format="NC1HWC0")
       # Model tiled L1 -> L0A/L0B double buffering:
       l1_x1, l1_x2, l1_y = Tensor("L1"), Tensor("L1"), Tensor("L1")
       l1_x1.load(in_x[0:16, :])
       l1_x2.load(in_x[16:32, :])
       l1_y.load(in_y)
       x1, x2, y = Tensor("L0A"), Tensor("L0A"), Tensor("L0B")
       x1.load(l1_x1)
       x2.load(l1_x2)
       y.load(l1_y)
       z = Tensor("L0C", "FP32", [16, 16], format="NC1HWC0")
       out1 = mmad(x1, y, z, True)()[0]
       out2 = mmad(x2, y, z, True)()[0]
       in_z.load(out1)
       in_z.load(out2)
   task_schedule.Schedule().clean()
   ```
2. **Clean Singleton Schedule State Across Sweeps**:
   - **Singleton Reset Invariant**: `task_schedule.Schedule()` is a C++ process-wide singleton (`csrc/core/singleton.h`). When sweeping multiple candidate tiling sizes $(M_{\text{tile}}, K_{\text{tile}}, N_{\text{tile}})$ in a single Python process, you MUST call `task_schedule.Schedule().clean()` after each `with Chip(...)` block; otherwise, residual instruction nodes from prior tiling candidates accumulate in the DAG and corrupt subsequent cycle predictions.

## Verification

- Inspect `MSKPP_*/trace.json` (in MindStudio Insight / Chrome Tracing) and `instruction_cycle_consumption.html` to verify whether `GM-TO-L1` MTE2 transfers hide behind `mmad` Cube cycles.
