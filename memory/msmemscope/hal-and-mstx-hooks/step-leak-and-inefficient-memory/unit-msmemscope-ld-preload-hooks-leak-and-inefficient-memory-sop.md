---
schema: mindie-graded-unit/1
id: unit-msmemscope-ld-preload-hooks-leak-and-inefficient-memory-sop
revision: 1
title: "Ascend msMemScope LD_PRELOAD Hook Instrumentation, Intra-Step Leak Detection & Inefficient Memory SOP"
tier: L2
category: sop
evidence_strength: software_contract
status: active
domain_path: memory/msmemscope/hal-and-mstx-hooks/step-leak-and-inefficient-memory
inherits_from:
  - unit-ascend-msprof-aicore-pipe-and-host-wall-clock-axioms
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: memory
triggers:
  error_signatures:
    - libleaks_ascend_hal_hook.so
    - Early Allocation
    - Late Deallocation
    - Temporary Idleness
  keywords:
    - msmemscope
    - memscope
    - memory-leak
    - inefficient-memory
    - hal-hook
    - mstx-hook
    - kernel-hook
provenance:
  source_type: source_contract
  source_locator: Ascend/msmemscope@2cd53f2fd34745240a1e8c4a9e66edeb25dea0d8:docs/en/overview.md
  evidence_refs:
    - https://gitcode.com/Ascend/msmemscope/blob/2cd53f2fd34745240a1e8c4a9e66edeb25dea0d8/docs/en/overview.md#L1-L25
    - https://gitcode.com/Ascend/msmemscope/blob/2cd53f2fd34745240a1e8c4a9e66edeb25dea0d8/example/leaks/README.md#L1-L69
    - https://gitcode.com/Ascend/msmemscope/blob/2cd53f2fd34745240a1e8c4a9e66edeb25dea0d8/example/inefficient/README.md#L1-L46
  supersedes: []
---

## Preconditions & Trigger

Apply this SOP when diagnosing intra-step NPU HBM memory leaks (at either the PyTorch/MindSpore caching allocator layer or the CANN HAL driver layer) or when identifying operator-level inefficient memory lifetimes (**Early Allocation**, **Late Deallocation**, **Temporary Idleness**) using `msMemScope` (`Ascend/msmemscope`).

## Mandatory `LD_PRELOAD` Hook Chain & Compatibility

1. **Three-Library `LD_PRELOAD` Instrumentation**:
   - `msMemScope` intercepts HAL allocations, MSTX step markers, and kernel launches via three shared libraries that MUST be preloaded before starting the Python process:
     ```bash
     export TOOL_PATH=/usr/local/Ascend/msmemscope
     export LD_PRELOAD="${TOOL_PATH}/lib64/libleaks_ascend_hal_hook.so:${TOOL_PATH}/lib64/libascend_mstx_hook.so:${TOOL_PATH}/lib64/libascend_kernel_hook.so"
     export LD_LIBRARY_PATH="${TOOL_PATH}/lib64:${LD_LIBRARY_PATH}"
     ```
   - Compatibility baselines: CANN ATB operators (`>= 8.2.RC1`), Ascend Extension for PyTorch (`>= 7.0.0`), Aten operator dispatch/access events (`PyTorch >= 2.3.1`).
2. **Non-Python / C++ Fatbin CLI Mode**:
   - For standalone C/C++ AscendC binaries where Python hooks are unavailable, mark step boundaries using MSTX (`mstxMarkA("step start", stream)`) and launch directly under `msmemscope ./<kernel_binary>.fatbin`.

## Intra-Step Memory Leak & Inefficient Memory Identification SOP

1. **Configure Python API Collection Scope (`memscope.config` / `start` / `step` / `stop`)**:
   - To avoid gigabyte trace bloat across warmup iterations, instrument only 2–5 steady-state steps:
     ```python
     import memscope

     memscope.config(
         format="db",              # "csv" (default) or "db"
         output_path="./memscope_out",
         events="alloc,free,launch,access",
         analysis="leaks,inefficient",
     )
     memscope.start()
     for step_idx in range(3):
         run_one_step()
         memscope.step()           # Marks step boundary for intra-step leak accounting
     memscope.stop()
     ```
   - **Event Requirement Invariant**:
     - `analysis="leaks"` requires `events` to include at least `alloc` and `free` (`launch` enables kernel-launch-correlated memory delta tracking). Framework memory-pool leaks print directly to stdout at `memscope.step()`, while CANN HAL-level leaks are written to the output log file.
     - `analysis="inefficient"` additionally correlates tensor allocation/deallocation timestamps against operator read/write `access` events.
2. **Classify & Remediate the Three Inefficient Memory Patterns**:
   - **Early Allocation (`过早申请`)**: A tensor is allocated many operator launches before its first consumer reads or writes it. Remediation: Move tensor creation (`torch.empty` / projection buffer allocation) immediately adjacent to its first consuming operator.
   - **Late Deallocation (`过迟释放`)**: A tensor remains alive in Python scope long after its last consuming kernel completes. Remediation: Explicitly `del intermediate_tensor` (or set attribute to `None`) immediately after the last consumer forward/backward op.
   - **Temporary Idleness (`临时闲置`)**: A large buffer is accessed early in the step and again late in the step, sitting idle across dozens of intermediate operators. Remediation: Offload to Host pinned DDR during the idle window or recompute/re-materialize before the second use.

## Verification

- Inspect the generated `memscope_out` dump/DB file and confirm zero unfreed intra-step framework/HAL blocks across `memscope.step()` boundaries and reduced peak HBM reservation after fixing `Early Allocation` / `Late Deallocation` spans.
