---
category: measurement
conditions:
  hardware:
  - Ascend 910B (A2)
  - Ascend 910C (A3)
  - Ascend 950 (A5)
  os:
  - Linux
  software_stack:
  - CANN 8.0+
  - msprof op
  workload:
  - Single-operator hardware PMU profiling, stable latency measurement, and vector
    roofline bound analysis
created_at: '2026-10-21'
domain_path: profiling/msprof-and-simulator/op-and-camodel/pmu-metrics-and-roofline
estimate_tokens: 620
evidence_strength: software_contract
id: unit-msprof-op-pmu-metrics-and-a5-vector-roofline-axioms
inherits_from:
- unit-ascend-msprof-aicore-pipe-and-host-wall-clock-axioms
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/cann/ops-profiling/SKILL.md#L1-L240
  promotion_status: verified
  source_locator: cann/cannbot-skills@f305e200aa4b1e1fed73cd21ff1f083b8df25a1e:cann/ops-profiling/SKILL.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Establishes the 10-pair warmup-target median latency protocol, MC2 multi-process
  prohibition for msprof op, PMU metric group semantics, and Ascend 950 5-factor vector
  roofline equations.
tier: L1
title: msprof op 10-Pair Median Timing, PMU Metric Groups, and Ascend 950 Vector Roofline
  Axioms
triggers:
  error_signatures:
  - vec_bankgroup_cflt_ratio
  - vec_bank_cflt_ratio
  - gm_rw_contention
  keywords:
  - msprof op 10-pair warmup
  - Ascend 950 vector roofline
  - core_split_units
  - fill_drain
updated_at: '2026-10-21'
---
# msprof op 10-Pair Median Timing, PMU Metric Groups, and Ascend 950 Vector Roofline Axioms

## Core Invariant

Hardware PMU counter replay (`msprof op`) perturbs L2 cache residency and inter-kernel dispatch gaps when invoked naively on a single kernel call. Reliable operator latency and bound attribution require a **10-pair interleaved `warmup_op -> target_op` execution schedule** (taking the median of the 10 target runs after discarding warmup pairs) and explicit separation of five physical bottleneck factors (`max_pipe`, `fill_drain`, `core_balance`, `launch_floor`, and `gm_rw_contention`).

## `msprof op` Measurement Protocol & MC2 Prohibition

1. **10-Pair Interleaved Median Protocol**:
   - In the benchmark script driven by `msprof op --output=<dir> python3 bench_op.py`, execute **10 consecutive pairs** of `(warmup_op(), target_op())`.
   - The `warmup_op` warms instruction caches and stabilizes AICore frequency (`aic_frequency`), while the 10 `target_op` rows in `OpBasicInfo.csv` / `PipeUtilization.csv` provide a statistically robust **median `Task Duration(us)`** immune to first-launch compilation and clock-ramp outliers.
2. **MC2 Forked Multi-Process Prohibition**:
   - **Never wrap a `torch.multiprocessing` (`mp.spawn`) or multi-rank MC2 (`MatmulAllReduce` / `AllGatherMatmul`) script inside `msprof op`**. `msprof op` hooks single-process runtime device contexts; wrapping forked ranks causes child processes to deadlock on profiling IPC sockets or overwrite each other's PMU buffers. For MC2 or multi-rank operators, use `torch_npu.profiler` per rank or `msprof --application`.

## `msprof op` PMU Metric Group Semantics

Configure `--aic-metrics=<Group>` according to the suspected bottleneck:

| `--aic-metrics` Group | Key CSV Columns | Physical Bottleneck Revealed |
| --- | --- | --- |
| `PipeUtilization` (default) | `vec_ratio`, `mac_ratio`, `scalar_ratio`, `mte1_ratio`, `mte2_ratio`, `mte3_ratio`, `fixpipe_ratio` | Dominant execution pipe across Vector, Cube, Scalar, MTE1/2/3, and Fixpipe |
| `Memory` / `MemoryUB` / `MemoryL0` | `ub_read_bw`, `ub_write_bw`, `l2_cache_hit_rate`, `main_mem_read_bw` | Off-chip HBM bandwidth saturation vs on-chip UB/L1/L0 bandwidth limits |
| `ResourceConflictRatio` | `vec_bankgroup_cflt_ratio`, `vec_bank_cflt_ratio`, `vec_resc_cflt_ratio` | UB bank conflicts (multiple vector lanes accessing the same 32B bank in one cycle) |
| `ArithmeticUtilization` | `mac_fp16_ratio`, `vec_fp32_ratio`, `vec_fp16_ratio` | Instruction mix and vector lane utilization efficiency |
| `L2Cache` | `write_hit_rate`, `read_hit_rate`, `l2_Victim_rate` | L2 cache thrashing across tiles or workspace passes |

## Ascend 950 (A5) Five-Factor Vector Roofline Model

On Ascend 950 (`RegBase` vector architecture, `56` AIV cores (`N_cores`), `f_core = 1.65 GHz`, 32-bit/16-bit vector register width `256 B`, peak HBM read `1450 GB/s`, write `1250 GB/s`, bidirectional RW contention penalty $\eta_{\text{rw}} \approx 0.72$), theoretical single-operator floor latency $T_{\text{roofline}}$ is governed by five coupled factors:

$$T_{\text{steady}} = \max\left(T_{\text{MTE2}},\; T_{\text{VEC}},\; T_{\text{MTE3}},\; \frac{B_{\text{read}} + B_{\text{write}}}{\eta_{\text{rw}} \cdot BW_{\text{peak}}}\right)$$

$$T_{\text{roofline}} = \max\left(T_{\text{launch\_floor}},\; T_{\text{steady}} \cdot \left\lceil \frac{U_{\text{split}}}{N_{\text{cores}}} \right\rceil \Big/ \left(\frac{U_{\text{split}}}{N_{\text{cores}}}\right) + T_{\text{fill\_drain}}\right)$$

- **`core_balance` (`core_split_units` $U_{\text{split}}$)**: When $U_{\text{split}}$ is not a multiple of $N_{\text{cores}} = 56$ (e.g., 64 split units on 56 cores forces 2 waves where Wave 2 runs at $8/56 = 14.3\%$ occupancy), tail-wave quantization inflates latency by up to $1.875\times$.
- **`fill_drain`**: First-tile MTE2 prologue + last-tile MTE3 epilogue where double buffering cannot overlap with Vector compute.
- **`gm_rw_contention`**: Simultaneous MTE2 GM reads and MTE3 GM writes across cores contend on HBM read/write turnaround, reducing effective bidirectional bandwidth by $20\%\text{--}28\%$.
