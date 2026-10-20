---
category: workflow
conditions:
  hardware:
  - Ascend 910B (A2)
  - Ascend 910C (A3)
  - Ascend 950 (A5)
  os:
  - Linux
  software_stack:
  - CANN 8.0+
  - msprof
  - HCCL
  - Ascend C
  workload:
  - Multi-rank communication overlap optimization and deterministic 8-step operator
    precision failure debugging
created_at: '2026-10-21'
domain_path: profiling/msprof-and-simulator/system-and-timeline/hccl-overlap-and-precision-flow
estimate_tokens: 650
evidence_strength: software_contract
id: unit-end-to-end-hccl-overlap-and-eight-step-precision-triage-workflow
inherits_from:
- unit-msprof-analyze-step-overlap-clock-and-intervals
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/cann/precision-debug/SKILL.md#L1-L260
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/cann/end-to-end-profiling/SKILL.md#L1-L220
  promotion_status: verified
  source_locator: cann/cannbot-skills@f305e200aa4b1e1fed73cd21ff1f083b8df25a1e:cann/precision-debug/SKILL.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Combines multi-stream HCCL communication overlap decomposition with the strict
  8-step (Step 0..Step 7) Ascend operator precision triage workflow, including DISABLE_L2_CACHE=1
  cross-case isolation.
tier: L3
title: End-to-End HCCL Communication Overlap Analysis and Eight-Step Operator Precision
  Triage Workflow
triggers:
  error_signatures:
  - batch_precision_check.py
  - DISABLE_L2_CACHE=1
  - Not Overlapped Communication
  keywords:
  - 8-step precision workflow
  - stage dump bisection
  - 1-ULP rounding proof
  - communication overlap ratio
updated_at: '2026-10-21'
---
# End-to-End HCCL Communication Overlap Analysis and Eight-Step Operator Precision Triage Workflow

## Core Principle

End-to-end distributed optimization requires maximizing HCCL communication overlap with AI Core compute while strictly preserving numerical accuracy. When an optimized kernel or overlap schedule fails accuracy gates, never guess or tweak tolerances—execute the **8-step deterministic precision triage workflow (`Step 0..Step 7`)** to isolate cross-case cache contamination, multi-core races, stage-level divergence, and 1-ULP rounding boundaries.

## Part 1: Multi-Stream HCCL Communication Overlap Decomposition

From `step_trace_time.csv` and `communication.json` in `msprof` / `ASCEND_PROFILER_OUTPUT`:
1. Compute the **Communication Overlap Ratio**:
   $$R_{\text{overlap}} = 1 - \frac{T_{\text{comm\_not\_overlapped}}}{T_{\text{comm\_total}}}$$
2. Separate `T_comm_not_overlapped` into:
   - **Straggler Wait (`Wait Time`)**: Rank skew where faster ranks block on `HcclAllReduce` / `HcclAllGather` waiting for the slowest rank (inspect per-rank `Computing` variance first).
   - **True Unoverlapped Transit (`Transit Time` without concurrent AI Core stream)**: Communication scheduled on the default stream or blocked by an premature `record_event`/`wait_event` before independent GEMM/Attention chunks are enqueued.

## Part 2: Eight-Step Operator Precision Triage Workflow (`Step 0..Step 7`)

Execute `Step 0` through `Step 7` in order; stop as soon as the root cause is proven:

### Step 0: External Standard Lock & Deterministic Seed Freeze
- Lock the dtype tolerance standard (`ops-precision-standard`, e.g., FP16 `rtol=1e-3, atol=1e-5`, BF16 `rtol=4e-3, atol=1e-5`, FP32 `rtol=1e-4, atol=1e-5`) and fix RNG seeds (`torch.manual_seed(42)`, `np.random.seed(42)`). **Never relax external tolerance thresholds to make a test pass.**

### Step 1: Machine-Readable Batch Baseline (`batch_precision_check.py`)
- Run all test cases through a machine-readable harness with strict exit codes:
  - `exit 0`: All cases pass tolerance.
  - `exit 1`: Numerical tolerance exceeded (`max_abs_err`, `max_rel_err`, `err_ratio`, and first mismatch index reported).
  - `exit 2`: Runtime crash, shape mismatch, or `NaN`/`Inf` injection.

### Step 2: Single-Run vs Batch Stability & Cross-Case Cache Isolation
- Re-run the failing case(s) 5 times in isolation vs inside the full batch:
  - **Fails in batch, passes alone**: Cross-case state contamination—either `op_host` global/static C++ variables retain stale tiling state, an ACLNN executor cache reuses a mismatched workspace, or dirty L2/GM memory leaks across cases. Test with `DISABLE_L2_CACHE=1` and audit `Init()` zeroing of output/workspace GM (`SetAtomicAdd` outputs **must** be zero-initialized before accumulation).
  - **Varies across repeated isolated runs**: Intra-kernel race condition (`SetFlag`/`WaitFlag` missing or multi-core atomic/non-atomic GM overwrite). Run `mssanitizer --tool=racecheck` and `npu-check --tool synccheck`.

### Step 3: Pipeline Stage Bisection via Intermediate Dumps
- For deterministic single-case failures, bisect the kernel pipeline (`CopyIn -> Stage1 -> Stage2 -> CopyOut`) by dumping intermediate UB/L0C tensors (`AscendC::DumpTensor` on `block_idx == 0`) or writing stage outputs to GM and comparing each stage against an exact FP32/FP64 reference slice.

### Step 4: Error Spatial Pattern Classification
- Inspect the spatial distribution of mismatched elements in the failing tensor:
  - **Tail-only mismatch (last 1..32 elements of a row or last core)**: Non-32B-aligned `DataCopyPad` / tail mask calculation bug, or `ReduceMax`/`ReduceSum` reading unmasked padding trash in UB.
  - **Block-strided mismatch (every $K$-th block or specific `block_idx >= B_0`)**: 32-bit `int32_t` byte-offset overflow (`offset > 2^31 - 1` on large tensors; promote indexing to `int64_t` / `uint64_t`) or uneven core split remainder calculation.
  - **Uniform small drift across all elements**: Accumulation order or intermediate precision downgrade (accumulating in FP16/BF16 instead of FP32).

### Step 5: Active Target Architecture Variant Verification (`dav_*`)
- Confirm the source file/branch being edited is the one actually compiled for the target SoC (`ascend910b` / `dav_c220` vs `ascend950` / `dav_m300`); editing an inactive `#ifdef` branch leaves the binary unchanged.

### Step 6: 1-ULP Hardware Rounding Proof (When Error $\le 1\text{--}2$ ULP)
- If the discrepancy is strictly bounded to 1 ULP (`|actual - golden| == 1 ULP` of target dtype) at half-way rounding boundaries, verify whether the hardware instruction (`vconv` round-to-nearest-even vs host C++ `FloatToBf16` round-to-odd/truncation, or fused FMA vs separate Mul+Add) explains the exact bit difference.

### Step 7: Closed-Loop Audit Report
- Re-run the full batch (`batch_precision_check.py` returning `exit 0`) and record the root-cause stage, fix diff, and before/after max error metrics.
