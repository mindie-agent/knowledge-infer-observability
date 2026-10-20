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
  - mssanitizer
  - npu-check
  workload:
  - Ascend C and Triton-Ascend kernel memory out-of-bounds, multi-core race, uninitialized
    read, and event sync verification
created_at: '2026-10-21'
domain_path: diagnostics/cann-toolchain/sanitizer-and-debug-api/mssanitizer-and-npu-check
estimate_tokens: 600
evidence_strength: software_contract
id: unit-mssanitizer-six-anomaly-classes-and-npu-check-synccheck-sop
inherits_from:
- unit-ascend-npu-compute-npu-check-and-debug-api-sop
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/cann/mssanitizer-Detection/SKILL.md#L1-L310
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/cann/npu-check/SKILL.md#L1-L150
  promotion_status: verified
  source_locator: cann/cannbot-skills@f305e200aa4b1e1fed73cd21ff1f083b8df25a1e:cann/mssanitizer-Detection/SKILL.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Configures mssanitizer across memcheck/racecheck/initcheck for all 6 memory
  anomaly classes (including PyTorch caching allocator hooks) and npu-check for hardware
  event pair auditing.
tier: L2
title: mssanitizer Six Memory Anomaly Classes, Caching Allocator Hooks, and npu-check
  synccheck SOP
triggers:
  error_signatures:
  - SanitizerReportMalloc
  - SanitizerReportFree
  - --check-unused-memory=yes
  keywords:
  - mssanitizer
  - Multi-core Overwrite
  - SET_FLAG->WAIT_FLAG
  - GET_BUF->RLS_BUF
updated_at: '2026-10-21'
---
# mssanitizer Six Memory Anomaly Classes, Caching Allocator Hooks, and npu-check synccheck SOP

## Core Principle

`mssanitizer` instruments GM/UB/L1 memory transactions and cross-core barriers at runtime, while `npu-check` validates hardware memory bounds (`memcheck`) and pipe synchronization pairings (`synccheck`). When running `mssanitizer` on PyTorch (`torch_npu`) workloads, the framework's caching allocator pre-allocates large GM pools (`aclrtMalloc`), which masks small tensor out-of-bounds accesses unless explicit `SanitizerReportMalloc` / `SanitizerReportFree` hooks or direct ACLNN/C++ runners are used.

## `mssanitizer` Sub-Tools & Six Memory Anomaly Classes

Invoke `mssanitizer` with `--tool=memcheck` (default), `--tool=racecheck`, or `--tool=initcheck`:

```bash
# Full memory + leak + unused-allocation check targeting a single custom kernel
mssanitizer --tool=memcheck --leak-check=yes --check-unused-memory=yes   --kernel-name=<target_kernel_substr> -- ./run_op_test
```

| Anomaly Class | Sub-Tool & Flag | Detection Mechanism & Root Cause |
| --- | --- | --- |
| **1. Illegal Read / Write** | `--tool=memcheck` | GM/UB/L1 address falls outside allocated tensor bounds (e.g., `DataCopy` tail overshoot) |
| **2. Multi-core Overwrite** | `--tool=memcheck` | Block `i` writes to a GM address range owned by Block `j` without atomic mode (`owner` tag violation) |
| **3. Misaligned Access** | `--tool=memcheck` | DMA or vector instruction violates hardware alignment (e.g., UB address not 32B-aligned, L1 not 512B-aligned) |
| **4. Illegal Free** | `--tool=memcheck` | Double free or freeing an unregistered GM pointer |
| **5. Memory Leak** | `--leak-check=yes` | Device memory allocated during the session is never released before process exit |
| **6. Unused Memory** | `--check-unused-memory=yes` | Allocated GM tensor or workspace buffer receives zero device reads/writes across the kernel launch |

## Mandatory PyTorch Caching Allocator Rule & Compilation Flags

1. **Compiler Debug Symbols**: Build the operator with `-g` or `--cce-enable-sanitizer` (and `export TRITON_ENABLE_SANITIZER=true` for Triton-Ascend) so `mssanitizer` resolves exact `file:line:col` frames instead of raw PC offsets.
2. **PyTorch Caching Allocator Workaround**: Because `c10_npu::NPUCachingAllocator` sub-allocates tensors inside a 2 MB / 20 MB block, a 128-byte buffer overflow stays inside the driver block and goes undetected by default `aclrtMalloc` hooks. Either:
   - Run the operator through a standalone `aclnn` C++ runner where each tensor uses an exact-sized `aclrtMalloc`, or
   - Instrument the custom allocator wrapper with `SanitizerReportMalloc(ptr, size)` and `SanitizerReportFree(ptr)` from `<ascend_sanitizer.h>` so `mssanitizer` tracks logical tensor boundaries.

## `npu-check` Hardware Synchronization Auditing (`synccheck`)

When a kernel hangs or exhibits non-deterministic precision corruption without out-of-bounds addresses, run `npu-check`:

```bash
npu-check --tool synccheck ./run_op_test
```

- **Audited Event Pairs**: `npu-check --tool synccheck` verifies 1-to-1 hardware instruction pairing across all pipes:
  - `SET_FLAG -> WAIT_FLAG` (unconsumed `SET_FLAG` left dangling at kernel exit, or duplicate `SET_FLAG` on the same `(src_pipe, dst_pipe, event_id)` before `WAIT_FLAG`).
  - `GET_BUF -> RLS_BUF` (Ascend 950 / RegBase hardware buffer acquisition without matching release).
- **Mandatory Host Flush**: The test binary **must call `aclrtSynchronizeStream(stream)`** (or `torch.npu.synchronize()`) before exiting; otherwise `npu-check` terminates before the device hardware exception report buffer flushes to host memory.
