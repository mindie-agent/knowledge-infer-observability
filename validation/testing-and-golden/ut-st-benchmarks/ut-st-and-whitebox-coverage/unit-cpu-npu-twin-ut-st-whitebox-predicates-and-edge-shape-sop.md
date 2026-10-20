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
  - tikicpulib
  - GTest
  - lcov
  workload:
  - Operator UT/ST test suite design, whitebox TilingKey branch coverage, empty tensor
    semantics, and LLT coverage verification
created_at: '2026-10-21'
domain_path: validation/testing-and-golden/ut-st-benchmarks/ut-st-and-whitebox-coverage
estimate_tokens: 640
evidence_strength: software_contract
id: unit-cpu-npu-twin-ut-st-whitebox-predicates-and-edge-shape-sop
inherits_from:
- unit-multi-dtype-tolerance-mere-mare-and-bf16-comparator-axioms
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/knowledge/contrib/ascendc/guides/testing/kernel_cpu_simulation_and_aicpu_test_boundaries.md#L1-L120
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/knowledge/contrib/ascendc/guides/testing/empty_tensor_semantics.md#L1-L120
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/knowledge/contrib/ascendc/guides/testing/whitebox_test_case_design_workflow.md#L1-L120
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/knowledge/contrib/ascendc/guides/testing/coverage_denominators_and_execution_evidence.md#L1-L120
  promotion_status: verified
  source_locator: cann/cannbot-knowledge@be5882ef29fa9943305085652d2025a038ba491c:knowledge/contrib/ascendc/guides/testing/index.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Specifies CPU twin simulation (tikicpulib GmAlloc/ICPU_RUN_KF vs NodeDefBuilder),
  empty/scalar/non-32B edge-shape semantics, SHMEM multi-rank patterns, whitebox TilingKey
  predicates, and LLT 80% coverage denominators.
tier: L2
title: Ascend C CPU/NPU Twin UT/ST Boundaries, Whitebox TilingKey Predicates, and
  Edge-Shape Coverage SOP
triggers:
  error_signatures:
  - ICPU_RUN_KF
  - NodeDefBuilder
  - GmAlloc
  keywords:
  - tikicpulib
  - empty tensor semantics
  - whitebox TilingKey
  - TTK 26-column CSV
updated_at: '2026-10-21'
---
# Ascend C CPU/NPU Twin UT/ST Boundaries, Whitebox TilingKey Predicates, and Edge-Shape Coverage SOP

## Core Principle

Operator verification requires aligning **whitebox host tiling path predicates** (`TilingKey` & `GetWorkspaceSize` branches) with **device kernel execution paths**, separating what CPU simulation (`tikicpulib` / `NodeDefBuilder`) can prove from what requires real NPU hardware (`aclnn` ST), and testing algebraic boundary shapes (0D scalars, 0-volume empty tensors, and non-32B-aligned tails) without inflating coverage denominators.

## 1. CPU Simulation (`tikicpulib` / `NodeDefBuilder`) vs Real NPU ST Boundaries

| Execution Layer | Harness API | What It Verifies | What It Cannot Prove (Requires Real NPU ST) |
| --- | --- | --- | --- |
| **Host Tiling UT** | Direct call to `TilingFunc` / `InferShape` / `InferDataType` | `TilingKey` selection, workspace byte size, core split arithmetic, and invalid-arg error codes | Device kernel DMA alignment, UB capacity overflow, or AI Core pipeline races |
| **Ascend C Kernel CPU Twin** | `AscendC::GmAlloc`, `AscendC::SetKernelMode(AIV_MODE)`, `ICPU_RUN_KF(kernel, blkDim, ...)` (`tikicpulib`) | Functional C++ logic of `CopyIn/Compute/CopyOut` and `AscendC::printf` / `assert` | Hardware-specific `Fixpipe` saturation modes, real MTE2/MTE3 asynchronous overlap timing, and 910B2C same-launch GM write-after-read hazards |
| **AICPU Kernel UT** | `NodeDefBuilder` + `CpuKernelRegister::Instance().RunCpuKernel` | Host/device AICPU operator C++ execution on CPU memory | ACLNN two-stage (`GetWorkspaceSize` + `aclnn<Op>`) NPU stream execution |

## 2. Mandatory Edge-Shape & Algebraic Boundary Semantics

Every operator UT/ST suite must include deterministic boundary cases covering these four classes:

1. **Zero-Volume Empty Tensors vs Non-Empty Outputs**:
   - **Elementwise / Activation (`shape = [3, 0, 5]`, `numel == 0`)**: `GetWorkspaceSize` and `Tiling` must return `ACL_SUCCESS` with `workspaceSize = 0` without launching a kernel (`blockDim = 0` or host early-return) and without dereferencing `nullptr` device pointers.
   - **Reduction / MatMul over Zero-Length Contracting Axis (`[M, 0] x [0, N] -> [M, N]` or `Sum([3, 0, 5], dim=1) -> [3, 5]`)**: The output tensor has **non-zero volume (`M * N > 0`)** even though the input contracting dimension is `0`! A naive `if (input.numel() == 0) return;` leaves the output tensor uninitialized—the operator must fill the output with the algebraic identity (`0` for sum/matmul, `1` for prod, `-inf` for max, `+inf` for min).
2. **0D Scalar Tensors (`shape = []`, `dim == 0`, `numel == 1`)**:
   - Verify `GetShape().GetDimNum() == 0` does not underflow `dim - 1` indexing in `op_host` tiling (`std::max<int64_t>(dim, 1)`).
3. **Non-32B-Aligned Tails & Single-Core vs Multi-Core Thresholds**:
   - Include prime element counts (`1, 7, 31, 33, 255, 257`) that exercise `DataCopyPad` partial-block tails, single-core (`blockDim == 1`), full-core (`blockDim == maxCoreNum`), and tail-wave (`totalTiles % maxCoreNum != 0`) splits.
4. **SHMEM Symmetric Multi-Rank Cases**:
   - For symmetric-memory (`SHMEM`) collective kernels, initialize each PE (`my_pe`) with **rank-distinguishable input payloads** (`value = f(pe, idx)`); using identical constant tensors across all PEs masks source-PE vs destination-PE addressing swaps.

## 3. Whitebox `TilingKey` Predicate Design & Coverage Denominator Hygiene

1. **Whitebox Predicate Table**: Before generating test cases (`TTK` 26-column CSV or GTest parameterized fixtures), enumerate every `TilingKey` predicate in `op_host/<op>_tiling.cpp` (`dtype` $\times$ `format` $\times$ `alignment` $\times$ `ub_fit` $\times$ `broadcast_mode`) and construct **at least one test case per reachable `TilingKey`** plus one negative test per defensive error return.
2. **Coverage Denominator Rule (`>= 80%` Line & Branch)**:
   - Measure `lcov` / `gcov` coverage strictly over the **target operator's own `op_host/*.cpp` and `op_kernel/*.cpp` files**.
   - Never dilute or inflate the coverage denominator by including autogenerated protobuf stubs, third-party headers, or test-harness files themselves, and always attach real execution logs alongside the coverage summary.
