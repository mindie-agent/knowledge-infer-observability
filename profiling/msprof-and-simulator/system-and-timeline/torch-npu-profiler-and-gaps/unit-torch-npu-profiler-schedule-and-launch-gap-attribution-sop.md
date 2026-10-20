---
category: sop
conditions:
  hardware:
  - Ascend 910B (A2)
  - Ascend 910C (A3)
  - Ascend 950 (A5)
  os:
  - Linux
  software_stack:
  - CANN 8.0+
  - PyTorch (`torch_npu`)
  workload:
  - PyTorch NPU operator/model timeline profiling, op_statistic.csv comparison, and
    host dispatch gap triage
created_at: '2026-10-21'
domain_path: profiling/msprof-and-simulator/system-and-timeline/torch-npu-profiler-and-gaps
estimate_tokens: 580
evidence_strength: software_contract
id: unit-torch-npu-profiler-schedule-and-launch-gap-attribution-sop
inherits_from:
- unit-ascend-msprof-aicore-pipe-and-host-wall-clock-axioms
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/cann/torch-npu-profiler/SKILL.md#L1-L220
  promotion_status: verified
  source_locator: cann/cannbot-skills@f305e200aa4b1e1fed73cd21ff1f083b8df25a1e:cann/torch-npu-profiler/SKILL.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Standardizes torch_npu.profiler (wait=0, warmup=5, active=5, repeat=1) dual-path
  benchmarking on NPU and attributes timeline inter-kernel bubbles to host dispatch,
  runtime queue, or AICPU fallback.
tier: L2
title: torch_npu.profiler Deterministic Schedule, Dual-Path Baseline, and Host-to-Device
  Launch Gap Attribution SOP
triggers:
  error_signatures:
  - ASCEND_PROFILER_OUTPUT
  - op_statistic.csv
  - AiCoreMetrics.PipeUtilization
  keywords:
  - torch_npu.profiler
  - ExperimentalConfig
  - aclrtLaunchKernel
  - AICPU fallback
updated_at: '2026-10-21'
---
# torch_npu.profiler Deterministic Schedule, Dual-Path Baseline, and Host-to-Device Launch Gap Attribution SOP

## Core Principle

Comparing a custom Ascend operator against a CPU reference or timing `time.perf_counter()` without stream synchronization produces meaningless speedup claims. All `torch_npu` performance evaluations must execute **both the custom operator and the PyTorch composite-op baseline on the NPU** under an identical `torch_npu.profiler.schedule(wait=0, warmup=5, active=5, repeat=1)` window, comparing `ASCEND_PROFILER_OUTPUT/op_statistic.csv` and `kernel_details.csv` side by side.

## Deterministic `torch_npu.profiler` Collection Template

```python
import torch
import torch_npu

experimental_config = torch_npu.profiler._ExperimentalConfig(
    export_type=[torch_npu.profiler.ExportType.Text],
    profiler_level=torch_npu.profiler.ProfilerLevel.Level1,
    msprof_tx=False,
    aic_metrics=torch_npu.profiler.AiCMetrics.PipeUtilization,
    l2_cache=False,
    op_attr=False,
    data_simplification=False,
)

with torch_npu.profiler.profile(
    activities=[
        torch_npu.profiler.ProfilerActivity.CPU,
        torch_npu.profiler.ProfilerActivity.NPU,
    ],
    schedule=torch_npu.profiler.schedule(wait=0, warmup=5, active=5, repeat=1),
    on_trace_ready=torch_npu.profiler.tensorboard_trace_handler("./prof_out"),
    record_shapes=True,
    profile_memory=False,
    with_stack=False,
    experimental_config=experimental_config,
) as prof:
    for step in range(10):
        out = target_fn(*inputs)
        torch.npu.synchronize()
        prof.step()
```

- **Mandatory `prof.step()` Alignment**: The loop iteration count (`10`) must equal `wait + warmup + active` (`0 + 5 + 5 = 10`) with `prof.step()` called at the end of every iteration; omitting `prof.step()` leaves `*_ascend_pt/ASCEND_PROFILER_OUTPUT/` empty.

## Three-Layer Inter-Kernel Gap & Fallback Attribution

Inspect `ASCEND_PROFILER_OUTPUT/op_statistic.csv`, `kernel_details.csv`, and `trace_view.json` to classify end-to-end overhead:

1. **Composite-Op Explosion vs Fused Kernel (`op_statistic.csv`)**:
   - Sum `Total Time(us)` and `Count` across all rows in `op_statistic.csv` for the PyTorch baseline vs the custom fused op. A fused operator should replace $N$ fine-grained `aclnn*` launches (and their $N-1$ intermediate GM read/write round-trips) with 1 kernel row.
2. **Host-to-Device Launch Bubble (`trace_view.json` `aclrtLaunchKernel` -> NPU Stream)**:
   - Compare the timestamp of host `aclrtLaunchKernel` (`AscendCL::aclrtLaunchKernel`) against the device execution start of the corresponding kernel:
     - **Host-Bound Dispatch Gap**: Device stream is idle (`gap > 5 us` between consecutive kernels) and `aclrtLaunchKernel` starts **after** the previous device kernel finished — Python wrapper overhead, eager shape inference, or host `op_host` tiling computation is starving the NPU. Enable `TASK_QUEUE_ENABLE=2` and hoist host tensor metadata checks.
     - **Device-Bound Queue Saturation**: `aclrtLaunchKernel` completes well before the previous device kernel finishes — NPU execution is continuous and optimization must focus on device compute/memory pipes.
3. **Silent `AI_CPU` Fallback Detection (`kernel_details.csv` `Type` / `AcceleratorCore`)**:
   - Filter `kernel_details.csv` for `AcceleratorCore == AI_CPU` (or `AICPU`). Any unexpected `Cast`, `IndexSelect`, `NonZero`, or `MaskedSelect` running on `AI_CPU` serializes the stream with $100\text{--}1000\,\mu\text{s}$ latency; replace with an `AI_VECTOR_CORE` (`AIV`) implementation or rewrite indexing via supported Ascend C vector primitives.
