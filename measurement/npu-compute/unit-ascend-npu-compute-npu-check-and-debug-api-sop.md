---
schema: mindie-graded-unit/1
id: unit-ascend-npu-compute-npu-check-and-debug-api-sop
revision: 1
tier: L2
category: sop
domain_path: measurement/npu-compute
evidence_strength: software_contract
title: Kernel-level profiling and correctness SOP for npu-compute, npu-check memcheck/synccheck, and AscendC debug APIs
conditions:
  accelerator: ascend
  tools:
  - npu-compute
  - npu-check
  - ascendc-debug-api
  repository: cann/cannbot-skills
  revision: f305e200aa4b1e1fed73cd21ff1f083b8df25a1e
inherits_from:
- unit-ascend-msprof-aicore-pipe-and-host-wall-clock-axioms
provenance:
  source_type: ecosystem_repo
  source_locator: docs/evidence/cannbot-exception-compute-and-error-code-provenance.json
  source_digest: b71bc139badf896ec20c0ae939a1fc7ad7c9e20c5931a87eedc965c6963df43c
  evidence_refs:
  - docs/evidence/cannbot-exception-compute-and-error-code-provenance.json
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/tools/tool-npu-compute/SKILL.md#L8-L176
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/tools/tool-npu-check/SKILL.md#L6-L126
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/tools/tools-npu-debug-api/SKILL.md#L10-L72
  supersedes: []
triggers:
  error_signatures:
  - "npu-compute: report="
  - "npu-compute: unpacked="
  - "ResourceConflictRatio"
  - "npu-check --tool memcheck"
  - "npu-check --tool synccheck"
  keywords:
  - npu-compute
  - npu-rep
  - PipeTrace.json
  - ArithmeticUtilization
  - PipeUtilization
  - ResourceConflictRatio
  - npu-check
  - memcheck
  - synccheck
  - asc_dump
  - asc_time_stamp
---
Use this SOP when collecting or interpreting kernel-level hardware counters with `npu-compute`, auditing GM out-of-bounds or flag/buffer synchronization bugs with `npu-check`, or instrumenting AscendC kernels with in-kernel debug APIs (`tools/tool-npu-compute/SKILL.md:8-176`, `tools/tool-npu-check/SKILL.md:6-126`, `tools/tools-npu-debug-api/SKILL.md:10-72`).

## 1. `npu-compute` Metric Collection, `.npu-rep` Unpacking, and Interpretation Rules

1. **Collection (`Set` vs `Section`) & Unpacking Verification (`tool-npu-compute/SKILL.md:52-110`)**:
   - Use `--set basic` (default when omitted) or `--set full` for pre-grouped metric sets, and `--section <Name>` (case-sensitive: `ArithmeticUtilization`, `PipeUtilization`, `ResourceConflictRatio`, `Memory`, `MemoryL0`, `MemoryUB`, `L2Cache`) for individual metric tables. When combining `--set` and `--section`, sections expand in CLI order and deduplicate by first occurrence. Always separate target binary flags with `--` (e.g., `npu-compute --set basic -- ./app --arg1`).
   - **Collection Success Gate**: Requires exit code `0`, terminal line `npu-compute: report= <path>`, and physical existence of the `.npu-rep` file.
   - **Unpack Success Gate**: Run `npu-compute --import <file.npu-rep> [--export <dir>]`; requires exit code `0`, terminal line `npu-compute: unpacked= <dir>`, and readable `summary.jsonl`, `HardwareInfo.jsonl`, `PipeTrace.json`, and Section CSV files. Never merge identically named CSVs from distinct sub-result directories.
2. **Metric Semantics (`tool-npu-compute/SKILL.md:140-148`)**:
   - `NA` denotes unavailable or inapplicable hardware counters and must **never** be treated as `0.0`.
   - Columns containing `ratio` are raw ratios ($[0, 1]$), whereas columns containing `(%)` are percentages ($[0, 100]$).
   - Compare rows across runs by matching `(block_id, sub_block_id)` and core type rather than raw CSV line numbers.
   - In `PipeTrace.json`, `ts` and `dur` are in **microseconds ($\mu\text{s}$)**, and events on distinct pipeline tracks may overlap in time.

## 2. `npu-check` (`memcheck` & `synccheck`) Correctness Verification (`tool-npu-check/SKILL.md:17-121`)

1. **Capabilities & Scope Constraints**:
   - `--tool memcheck` (default): Tracks active GM allocations from successful `aclrtMalloc` / `aclrtFree` calls, rounds each allocation size up to a **32-byte boundary**, and verifies that every DMA data-transfer instruction falls completely within a live allocation (does not check scalar direct memory accesses).
   - `--tool synccheck`: Audits intra-core event synchronization (`SET_FLAG -> WAIT_FLAG`) and buffer lifecycle (`GET_BUF -> RLS_BUF`) for duplicate pre-ops, missing post-ops, or count mismatches.
   - **Hard Scope Limits**: Supported on `Ascend 950` (`dav-3510`) for standalone single-process, single-thread executables launching a single `<<<>>>` kernel invocation (not direct PyTorch scripts).
2. **Stream Synchronization Gate**:
   - Diagnostic records are exported and processed only when the application calls `aclrtSynchronizeStream` or `aclrtSynchronizeStreamWithTimeout` after kernel launch. If the binary exits without stream synchronization or is killed by an external timeout, classify the run as **incomplete**—never report "zero errors found" as a pass.

## 3. AscendC In-Kernel Debug API Selection (`tools-npu-debug-api/SKILL.md:23-44`)

| Debug Goal | AscendC Interface | Key Constraint |
|---|---|---|
| Scalar values & branch path | `printf` / `AscendC::printf` | Subject to FIFO capacity; does not auto-traverse tensors |
| In-kernel invariant check | `assert` / `ascendc_assert` | Disabled by `NDEBUG`; never rely on assertions for production bounds safety |
| Explicit SIMT thread halt | `__trap` | Valid only on supported SIMT VF paths, not general SIMD pipelines |
| On-chip UB/L1/GM tensor dump | `asc_dump` / `asc_dump_*` | `dump_size` must not exceed allocated elements; account for 32B padding and `ASCENDC_DUMP=0` |
| Local cycle counter / SIMD trace | `clock` / `asc_time_stamp` | `clock` returns raw `uint64_t` cycles without waiting for async pipes; `asc_time_stamp` emits structured SIMD markers |
