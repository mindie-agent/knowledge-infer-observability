---
category: measurement
conditions:
  artifact:
  - op_summary_*.csv
  - step_trace_time.csv
  clock_domains:
  - host wall clock
  - device task timestamps
  profiler_family:
  - msprof
  - torch_npu.profiler
  required_sync: torch.npu.synchronize or an equivalent device or stream boundary
  warmup_policy: compile, capture and DVFS warmup are excluded from the steady-state
    sample
content_digest: sha256:89c25b95813bf26a77b075b2e217e48a011e886bf1d98ed357fe0139fe73dac4
domain_path: measurement/msprof
evidence_strength: software_contract
id: unit-ascend-msprof-aicore-pipe-and-host-wall-clock-axioms
inherits_from: []
provenance:
  evidence_refs:
  - docs/evidence/cannbot-observability-provenance.json#source_records.src-msprof-op-summary-fields
  - docs/evidence/cannbot-observability-provenance.json#observed_measurements.obs-op-summary-wait
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:plugins-community/collaborative-agent-kernel-evolution/skills/code-performance-advisor/references/standards/op_summary_header_guide.md#L17-L64
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:ops/ops-profiling/references/csv_fields_reference.md#L30-L64
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:ops/ops-profiling/references/msprof-guide.md#L48-L67
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:model/model-infer-profiling/SKILL.md#L185-L245
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:plugins-community/triton-optimizer/skills/triton-npu-optimize/scripts/profile-operator/torch_npu_profiler_parser.py#L113-L136
  source_digest: 814abc1f4a1141018c23b53388f21ee219ba15a1be1b87c1543f491685c1b68d
  source_locator: docs/evidence/cannbot-observability-provenance.json
  source_type: ecosystem_repo
  supersedes: []
revision: 2
schema: mindie-graded-unit/1
tier: L1
title: Ascend msprof pipe ratios and host wall clock are separate measurement domains
triggers:
  error_signatures:
  - host wall clock and Task Duration disagree
  - Task Wait Time exceeds Task Duration
  - step_trace Free dominates Stage
  - Task Duration changes while host wall clock is unchanged
  keywords:
  - msprof
  - op_summary
  - Task Duration
  - Task Wait Time
  - aic_mac_ratio
  - aiv_vec_ratio
  - aic_mte2_ratio
  - aiv_mte2_ratio
  - aiv_mte3_ratio
  - step_trace
  - Stage
  - Computing
  - Free
  - torch.npu.synchronize
  - wall clock
  - warmup
  - pipe utilization
---
## Applicability boundary for offline analysis

The interval and exporter semantics here do not depend on a particular machine topology. The separate `unit-observed-dual-die-phy-id-topology` applies only when resolving device identities on its recorded machine; it is not a prerequisite for reading existing profiler artifacts. Live timing still requires the synchronization and warmup facts declared by this unit.

Ascend NPU work is enqueued asynchronously. A Python caller that returns from `model(...)` or an operator call has not observed device completion. A host-clock interval that is meant to include that work therefore needs an explicit `torch.npu.synchronize()` or an equivalent device/stream boundary. Without it, the measured interval is primarily launch and enqueue cost, and it can be far smaller than the device interval. Put the synchronization at the boundary of the measured region: once before the first timed iteration and once after the last. Do not insert a synchronization between every kernel when the question is overlap; that changes the schedule being measured.

`msprof` and the `torch_npu.profiler` export expose several quantities that are not interchangeable.

| Quantity | Where it appears | What it measures | What it does not measure |
|---|---|---|---|
| `Task Start Time(us)` | `op_summary_*.csv` | The task's start timestamp in the profiler clock | Host wall clock before the launch |
| `Task Duration(us)` | `op_summary_*.csv` | The profiler's per-task interval, including device-side scheduling and response as recorded for that task | The host gap before the task was enqueued |
| `Task Wait Time(us)` (legacy `Wait Time(us)`) | `op_summary_*.csv` | A separate wait column for the task; a high value is a scheduling or overlap symptom | Kernel execution time |
| Pipe ratios | `op_summary_*.csv` | Active cycles of a pipe divided by that core's total cycles in the row | A percentage of end-to-end wall time, or an additive breakdown |
| `Stage`, `Computing`, `Free` | `step_trace_time.csv` | A step-level union of device-active time, communication time, overlap and idle gaps | Per-operator attribution by themselves |

The pipe ratios are ratios of cycles, not percentages of the step. `aic_mac_ratio` is the MAC-instruction cycle share on the Cube side; producer documents also use `aic_cube_ratio` when the Cube group is reported. `aiv_vec_ratio` is the Vector-instruction cycle share on the AI Vector side. `aic_mte2_ratio`, `aiv_mte2_ratio` and `aiv_mte3_ratio` are the corresponding MTE active-cycle shares. Values are normally in the range 0 to 1. A value of `0`, `N/A` or a blank can mean the metric was not collected for that row, not that the hardware was idle. `Task Type` determines whether AIC or AIV columns are meaningful, and `Block Dim` or `Mix Block Dim` records launch shape rather than measured utilization. The exporter is also explicit that `op_summary_*.csv` is a per-op aggregate, not per-core data; per-core analysis needs the sample-based `aicore.db` path.

A real capture shows why the distinction matters. In one `op_summary_PipeUtilization.csv`, a `Muls` row has `Task Duration` 14.158 us and `Task Wait Time` 5300.659 us. A `MlaPrologV3` row in the same file has `Task Duration` 52.778 us and `Task Wait Time` 1650.358 us. The wait column is tens to hundreds of times larger than the device duration. That is queueing or scheduling evidence, not a slow kernel. In the MlaPrologV3 row, `aic_mte2_ratio` is 0.582, `aic_mac_ratio` is 0.259 and `cube_utilization(%)` is 91.09; the high cube-utilization label does not mean 91% of cycles are MAC work.

`step_trace_time.csv` is the step-level complement. Its columns include `Computing`, `Communication(Not Overlapped)`, `Overlapped`, `Communication`, `Free`, `Stage`, `Bubble` and `Preparing`. In the recorded 128K decode window the observed identity was:

```text
Stage = Computing + Communication - Overlapped + Free
```

That window reported `Stage` 2525.010 ms, `Computing` 554.151 ms, `Communication` 1903.954 ms, `Overlapped` 9.408 ms and `Free` 76.314 ms. The identity is a bookkeeping relation for this exporter, not a universal law, and the same report warns that communication includes waiting and that profiling overhead is inside the window. Use `Free / Stage` as a host-launch and Python/scheduler gap indicator, `Computing / Stage` as device-active time, and `Communication(Not Overlapped) / Stage` as exposed collective time. Do not add `Communication` and `Computing` without subtracting `Overlapped`, and do not call a single rank's communication time a bandwidth measurement.

Warmup is part of the measurement contract. The first iterations can contain graph capture, JIT or compilation, allocator growth, cold caches and DVFS. For a comparable timing run, use a fixed warmup and report the median of steady-state samples; the producer protocol uses adjacent warmup-to-target pairs and excludes warmup from the reported total. For diagnostic capture, run several untimed warmups and discard the first captured step when `step_trace` shows a compile-sized outlier. When using `torch_npu.profiler`, put the first `torch.compile`, TorchAir graph build or JIT outside the profiler window, and let the final `prof.step()` leave the active window. A profile schedule that stops inside `RECORD` can produce incomplete data.

Decision rules:

- Host wall clock rises while `Task Duration` and `Computing` stay flat: suspect host scheduling, Python overhead or launch gaps; inspect `step_trace` `Free` and the host API view.
- `Task Duration` rises while host wall clock and `Free` stay flat: the change is on the device path; move to operator and pipe attribution.
- `Task Wait Time` is comparable to or larger than `Task Duration`: the task is waiting before or around execution; do not call the kernel slow from `Task Duration` alone.
- `Free` dominates `Stage`: the step is not feeding the device continuously; inspect host work, synchronization and graph replay.
- `Communication(Not Overlapped)` dominates `Stage`: the exposure is communication wait or transfer; separate wait from transfer before naming a bandwidth fault.

Adoption boundary: these are measurement and interpretation axioms derived from the pinned producer contracts, not a claim that a particular ratio threshold is universally good or bad. Producer thresholds such as "MTE2 above 50%" are diagnostic heuristics. Apply them only to a matched workload, metric group and hardware, and state the warmup, synchronization and aggregation protocol with the number. The observed values in the provenance packet illustrate the clock separation; they do not establish a hardware performance guarantee.
