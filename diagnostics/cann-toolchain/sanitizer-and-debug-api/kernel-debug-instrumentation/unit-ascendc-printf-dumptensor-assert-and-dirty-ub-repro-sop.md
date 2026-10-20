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
  - Ascend C
  - msaicerr
  workload:
  - Device-side tensor/scalar inspection and isolated single-operator fault reproduction
created_at: '2026-10-21'
domain_path: diagnostics/cann-toolchain/sanitizer-and-debug-api/kernel-debug-instrumentation
estimate_tokens: 590
evidence_strength: software_contract
id: unit-ascendc-printf-dumptensor-assert-and-dirty-ub-repro-sop
inherits_from:
- unit-ascend-npu-compute-npu-check-and-debug-api-sop
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/cann/kernel-debug-api/SKILL.md#L1-L210
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/cann/msaicerr-dump-analysis/SKILL.md#L210-L320
  promotion_status: verified
  source_locator: cann/cannbot-skills@f305e200aa4b1e1fed73cd21ff1f083b8df25a1e:cann/kernel-debug-api/SKILL.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Guides device-side Ascend C printf/DumpTensor/assert/timestamp instrumentation
  and msaicerr three-variant single-op reproduction with run_dirty_ub uninitialized-UB
  exposure.
tier: L2
title: Ascend C Device printf/DumpTensor/assert Instrumentation and msaicerr Dirty-UB
  Single-Op Repro SOP
triggers:
  error_signatures:
  - DumpTensor
  - ASCENDC_DUMP=0
  - error_single_op
  keywords:
  - AscendC::DumpTensor
  - ascendc_assert
  - run_dirty_ub
  - host_single_op
updated_at: '2026-10-21'
---
# Ascend C Device printf/DumpTensor/assert Instrumentation and msaicerr Dirty-UB Single-Op Repro SOP

## Core Principle

When inspecting intermediate LocalTensor/GM values or reproducing intermittent AI Core exceptions (`msaicerr`), combining lightweight device-side debug APIs (`AscendC::printf`, `AscendC::DumpTensor`, `assert`) with `msaicerr`'s three-variant single-operator harness (`host_single_op` -> `single_op` -> `error_single_op`) and `run_dirty_ub` deterministically exposes uninitialized UB reads, out-of-bounds tail writes, and bad tiling parameters without running the full model.

## Device-Side Debug API Rules & Capacity Guardrails

| Debug API Category | Ascend C / CCE Primitives | Mandatory Constraints & Silent-Failure Pitfalls |
| --- | --- | --- |
| **Formatted Scalar Print** | `AscendC::printf(fmt, ...)` | Supports `%d, %u, %x, %f, %lf, %s, %p` only. Silently disabled if `ASCENDC_DUMP=0` is exported or if the launch lacks a subsequent stream synchronize. |
| **Tensor Slice Dump** | `AscendC::DumpTensor(tensor, desc, dumpSize)` / `asc_dump` | `dumpSize` (in elements) **must make total byte length a multiple of 32 bytes**; non-32B-aligned `dumpSize` causes DMA alignment faults or truncated dumps. |
| **Device Assertions** | `assert(cond)` / `ascendc_assert(cond, fmt, ...)` / `__trap()` | Silently stripped by the compiler if `-DNDEBUG` is present in `CMakeLists.txt` or `op_host` build flags! |
| **Cycle Timestamps** | `AscendC::DumpTimeStamp(descId)` / `asc_time_stamp` / `clock()` | Inserts scalar cycle reads (`sys_cnt`); wrap target compute blocks with `PipeBarrier<PIPE_ALL>()` only during isolated micro-benchmarking, never in production. |

**Per-Core Dump FIFO Quota**: The device dump ring buffer is strictly bounded (default `1 MB` per core per kernel launch). Always guard `printf` and `DumpTensor` with `if (AscendC::GetBlockIdx() == 0)` and restrict dumps to a single loop iteration (`iter == 0`) to prevent FIFO saturation and dropped records.

## `msaicerr` Three-Variant Single-Op Reproduction & `run_dirty_ub`

When `msaicerr -p <collect_dir>` extracts a faulting operator from `dfx_data` / `plog`, it generates a standalone reproduction script testing three execution variants in sequence:

1. **Three-Variant Triage Sequence (`host_single_op` -> `single_op` -> `error_single_op`)**:
   - `host_single_op`: Re-runs the operator using normal host tiling and newly allocated device buffers.
   - `single_op`: Re-runs the extracted `.o` kernel directly with the dumped input tensors and workspace.
   - `error_single_op`: Re-runs the exact faulting core slice / tiling configuration captured from the exception dump.
   - **Diagnostic Interpretation**:
     - If `error_single_op` fails with AI Core exception while `single_op` passes, the bug is specific to the faulting core's boundary tile or `block_idx` offset.
     - If all three variants pass initially, the fault depends on **residual dirty data in Unified Buffer (UB)** left by a preceding kernel!
2. **Exposing Uninitialized UB Dependencies (`run_dirty_ub`)**:
   - `msaicerr` executes a helper kernel (`run_dirty_ub`) immediately before the target operator to pre-fill all UB addresses with non-zero poison values (`NaN` / `0xDEADBEEF` bit patterns).
   - If the target kernel passes on a clean device but triggers `vec_err` (`INF/NAN`) or precision corruption immediately after `run_dirty_ub`, the kernel is reading uninitialized UB elements (e.g., `ReduceMax`/`ReduceSum` reading unmasked padding tails or `Duplicate` zero-init omitted before accumulation).
3. **Out-of-Bounds Tail Detection (`magic` & `tail` Canary)**:
   - `msaicerr` appends canary bytes (`magic` header and `tail` trailer) around each allocated output and workspace GM buffer; any corruption of `magic` or `tail` after kernel completion proves a GM write buffer overflow (`Return Code 104`).
