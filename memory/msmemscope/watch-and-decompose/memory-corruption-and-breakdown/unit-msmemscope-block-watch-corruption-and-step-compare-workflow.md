---
schema: mindie-graded-unit/1
id: unit-msmemscope-block-watch-corruption-and-step-compare-workflow
revision: 1
title: "Ascend msMemScope Memory Block Corruption Watch, Step Comparison & Layer Decomposition Workflow"
tier: L3
category: workflow
evidence_strength: software_contract
status: active
domain_path: memory/msmemscope/watch-and-decompose/memory-corruption-and-breakdown
inherits_from:
  - unit-ascend-msprof-aicore-pipe-and-host-wall-clock-axioms
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: memory
triggers:
  error_signatures:
    - memory corruption between operators
    - OOM between steps
    - memory decomposition mismatch
  keywords:
    - msmemscope
    - memory-watch
    - memory-compare
    - memory-decomposition
    - silent-data-corruption
    - tensor-stomping
provenance:
  source_type: source_contract
  source_locator: Ascend/msmemscope@2cd53f2fd34745240a1e8c4a9e66edeb25dea0d8:docs/en/overview.md
  evidence_refs:
    - https://gitcode.com/Ascend/msmemscope/blob/2cd53f2fd34745240a1e8c4a9e66edeb25dea0d8/docs/en/overview.md#L9-L25
    - https://gitcode.com/Ascend/msmemscope/blob/2cd53f2fd34745240a1e8c4a9e66edeb25dea0d8/example/watch/README.md#L1-L55
    - https://gitcode.com/Ascend/msmemscope/blob/2cd53f2fd34745240a1e8c4a9e66edeb25dea0d8/example/compare/README.md#L1-L50
    - https://gitcode.com/Ascend/msmemscope/blob/2cd53f2fd34745240a1e8c4a9e66edeb25dea0d8/example/decompose/README.md#L1-L50
  supersedes: []
---

## Preconditions & Trigger

Use this workflow when diagnosing:
1. **Cross-Step Memory Growth / OOM (`compare`)**: HBM usage increases unexpectedly between two steps or between two framework configurations.
2. **CANN vs. Framework Memory Attribution (`decompose`)**: Unaccounted gap between `torch.npu.memory_allocated()` and physical `npu-smi info` HBM usage.
3. **Inter-Operator Memory Stomping / Silent Corruption (`watch`)**: A tensor's contents are silently overwritten by an out-of-bounds neighbor operator during foundation model execution.

## Step-by-Step Memory Triage Workflow

### Step 1: Decompose CANN vs. PyTorch Memory Footprint (`decompose`)
- When `npu-smi info` reports significantly higher HBM consumption than PyTorch tensor allocations, run `msMemScope` decomposition (`example/decompose`) with `LD_PRELOAD` hooks enabled.
- `msMemScope` categorizes every HBM allocation across both the **CANN runtime/GE/ATB layer** (HCCL communication buffers, stream workspaces, operator tiling/descriptor caches) and the **Ascend for PyTorch framework layer** (model weights, gradients, optimizer states, and dynamic activations).
- Use the decomposition breakdown to determine whether the HBM pressure originates from HCCL comm workspace bloat (`HCCL_BUFFSIZE`) or unfreed activation tensors.

### Step 2: Diff Memory Events Across Steps or Runs (`compare`)
- Capture `msMemScope` memory traces for the baseline step (`step_a`) and the anomalous step (`step_b`).
- Run the `msMemScope` comparison analyzer (`example/compare`) to align operator launch sequences and highlight exact allocation deltas (newly retained blocks, enlarged workspace sizes, or dynamic-shape bucket misses that forced new caching-allocator segment expansions).

### Step 3: Pinpoint Inter-Operator Memory Stomping via Block Monitoring (`watch`)
- When a specific weight, KVCache block, or residual tensor is suspected of being corrupted by an adjacent kernel's out-of-bounds write:
  1. Register the target memory block range (or tensor pointer + byte length) with `msMemScope` block watch (`example/watch`) via Python API or CLI before the suspect forward/backward region.
  2. With `libascend_kernel_hook.so` active, `msMemScope` snapshots checksums/bytes of the watched memory block immediately **before and after each operator execution** on the NPU stream.
  3. Inspect the watch log to identify the exact operator launch $(Op_{k} \to Op_{k+1})$ where the watched block mutated unexpectedly without `Op_{k}` declaring that tensor as an output operand.

## Verification

- Once the stomping operator $Op_{k}$ is isolated via `watch`, run `mssanitizer --tool=memcheck` on a standalone single-op reproducer of $Op_{k}$ to locate the exact Global Memory out-of-bounds instruction offset.
