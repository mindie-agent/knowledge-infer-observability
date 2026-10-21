---
schema: mindie-graded-unit/1
id: unit-msprof-analyze-aicpu-elimination-and-fused-affinity-api-sop
revision: 1
title: "Ascend msprof-analyze AICPU Operator Elimination & torch_npu Fused Affinity API Replacement SOP"
tier: L2
category: sop
evidence_strength: reproduced_empirical
status: active
domain_path: profiling/msprof-analyze/aicpu-and-fusion-advisor/dtype-cast-and-affinity-api
inherits_from:
  - unit-ascend-msprof-aicore-pipe-and-host-wall-clock-axioms
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: profiling
triggers:
  error_signatures:
    - aclnnIndexPutImpl_IndexPut_IndexPut
    - AICPU operator detected
    - npu_scaled_masked_softmax
  keywords:
    - msprof-analyze
    - advisor
    - aicpu-operator
    - npu-rms-norm
    - npu-swiglu
    - npu-confusion-transpose
    - index-put
provenance:
  source_type: reproduced_run+source_contract
  source_locator: Ascend/msprof-analyze@639eb8702c1884813dfab59ddb371f7838caab6d:docs/en/aicpu_operator_replacement_example.md
  evidence_refs:
    - https://gitcode.com/Ascend/msprof-analyze/blob/639eb8702c1884813dfab59ddb371f7838caab6d/docs/en/aicpu_operator_replacement_example.md#L1-L176
    - https://gitcode.com/Ascend/msprof-analyze/blob/639eb8702c1884813dfab59ddb371f7838caab6d/docs/en/fused_operator_api_replacement_example.md#L1-L250
  supersedes: []
---

## Preconditions & Trigger

Apply this SOP when `msprof-analyze advisor all -d <prof_dir>` or `op_statistic.csv` / `kernel_details.csv` flags **AICPU operators** (`Task Type: AI_CPU`) or fragmented elementwise/transpose chains that stall the AI Core pipeline during PyTorch / `torch_npu` model execution.

## Part 1: Eliminating AICPU Fallback Operators (`AI_CPU` -> `AI_CORE`)

Operators fall back from AI Core to AICPU either because their input `dtype` lacks an AI Core kernel or because dynamic indexing/masking semantics trigger `aclnnIndexPutImpl`. Apply two targeted transformations:

1. **Data-Type Promotion/Demotion to AI Core Supported Types**:
   - **`MUL`**: AI Core supports `float32`, `float16`, `bfloat16`, `int32`, `int64`, `int8`, `uint8`, `complex64`; `int16` and `complex128` fall back to AICPU. Cast `int16` inputs to `int32` or `float16`.
   - **`Equal`**: AI Core supports `float32`, `float16`, `bfloat16`, `bool`, `int32`, `int64`, `int8`, `uint8`; `int16`, `complex64`, and `complex128` fall back to AICPU.
   - **`TensorEqual`**: AI Core supports `float32`, `float16`, `bfloat16`, `float64`, `bool`, `int32`, `int8`, `uint8`; `int16` and `int64` fall back to AICPU. Cast `int64` tensors to `int32` before `torch.equal`.
2. **Equivalent AI Core Operator Replacements for Indexing & Masking**:
   - **1D Index Slicing (`tensor[index]`)**: Replace dynamic indexing with `torch.index_select` (dispatches to AI Core `GatherV2`).
   - **Masked Scalar Assignment (`tensor_a[mask] = val`)**: Replace in-place masked `index_put` with `torch.where(mask, val, tensor_a)` (dispatches to AI Core `SelectV2`) while keeping tensor shapes invariant to avoid `.contiguous()` copies.
   - **Zero-Masking `IndexPut` (`masked_input[input_mask] = 0`)**:
     - Native `masked_input[input_mask] = 0` dispatches 5 host operators including AICPU `aclnnIndexPutImpl_IndexPut_IndexPut` (`9.902 ms` on shape `(512, 32, 64)` `float32`).
     - Replace with bitwise-inverted mask multiplication on AI Core:
       ```python
       masked_input *= torch.logical_not(input_mask)
       ```
       This reduces execution to 3 pure AI Core kernels and cuts total latency from `9.902 ms` to `226.131 us` (`43.8x` speedup).
   - **Masked Reduction (`tensor_a[torch.nonzero(mask).squeeze(1)].sum()`)**: Avoid `torch.nonzero` index extraction; compute `(tensor_a * mask).sum()` directly on AI Core Vector units.

## Part 2: `torch_npu` Fused Affinity API Replacements

Replace fragmented native PyTorch composite sequences flagged by `msprof-analyze advisor` with single-launch `torch_npu` affinity kernels:
1. **Permute + Reshape (`torch_npu.npu_confusion_transpose`)**:
   - Replace `torch.permute(x, (0, 2, 1, 3)).reshape(H, B, C * W)` with `torch_npu.npu_confusion_transpose(x, (0, 2, 1, 3), (H, B, C * W), transpose_first=True)`.
   - Replace `x.view(B, H * C * W).transpose(1, 0)` with `torch_npu.npu_confusion_transpose(x, (1, 0), (B, H * C * W), transpose_first=False)`.
2. **Scaled Masked Softmax (`torch_npu.npu_scaled_masked_softmax`)**:
   - Replace `torch.softmax((x * scale).masked_fill(mask, -inf), dim=-1)` with `torch_npu.npu_scaled_masked_softmax(x, mask, scale)`.
   - **Shape Alignment Invariant**: The last dimension of both `x` (`atten_scores`) and `mask` (`atten_mask`) MUST be within `[32, 8192]` and strictly divisible by `32`.
3. **Normalization, Activation & Optimizer Fusion**:
   - Replace composite `RMSNorm` (`x * torch.rsqrt(x.pow(2).mean(-1, keepdim=True) + eps) * weight`) with `torch_npu.npu_rms_norm(x, weight, epsilon=eps)[0]` (`dtype` in `{float16, bfloat16, float32}`).
   - Replace `SwiGLU` / `GELU` with `torch_npu.npu_swiglu` / `torch_npu.fast_gelu`, and replace `torch.optim.AdamW` + `clip_grad_norm_` with `torch_npu.optim.NpuFusedAdamW` + `optimizer.clip_grad_norm_fused_`.

## Verification

- Re-run `msprof-analyze advisor all -d <prof_dir>` after replacement and verify zero `AI_CPU` tasks remain in `kernel_details.csv` for the modified module.
