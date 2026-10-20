---
category: sop
conditions:
  hardware:
  - Ascend 910B (A2)
  - Ascend 910C (A3)
  os:
  - Linux
  software_stack:
  - CANN 8.0+
  - Triton-Ascend
  - PyTorch (`torch_npu`)
  workload:
  - Operator-level MFU ceiling discovery, shape scaling grid search, and per-kernel
    FLOPs decomposition
created_at: '2026-10-21'
domain_path: profiling/msprof-and-simulator/op-and-camodel/triton-mfu-decomposition
estimate_tokens: 610
evidence_strength: software_contract
id: unit-triton-operator-mfu-grid-search-and-dtype-peak-calib-sop
inherits_from:
- unit-msprof-analyze-mfu-range-flops-attribution
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/plugins-community/triton-profiler-assistant/skills/triton-mfu-analyzer/SKILL.md#L1-L260
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/knowledge/contrib/ops/triton/guides/tooling/mfu_workflow_and_metrics.md#L1-L110
  promotion_status: verified
  source_locator: cann/cannbot-skills@f305e200aa4b1e1fed73cd21ff1f083b8df25a1e:plugins-community/triton-profiler-assistant/skills/triton-mfu-analyzer/SKILL.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: 'Executes the 6-step Triton-Ascend MFU evaluation workflow: 2D (case, scale)
  grid search with 5-gate shape filtering, min(traced_flops, hardware_flops) semantic
  numerator, and per-dtype peak calibration.'
tier: L2
title: Triton-Ascend Operator MFU 2D Grid Search, Five-Gate Shape Filter, and Per-Dtype
  Peak Calibration SOP
triggers:
  error_signatures:
  - MFU_nom
  - MFU_ach
  - peak_calib.py
  keywords:
  - triton-mfu-analyzer
  - check_flops.py
  - traced_flops
  - hardware_flops
updated_at: '2026-10-21'
---
# Triton-Ascend Operator MFU 2D Grid Search, Five-Gate Shape Filter, and Per-Dtype Peak Calibration SOP

## Core Principle

Operator Model FLOPs Utilization (`MFU`) on Ascend NPU is **non-monotonic with tensor scale**: small shapes are launch/tail bound, while oversized shapes spill L2 cache (`> 192 MB`) or trigger unfavorable tile splits. Therefore, discovering an operator's true hardware ceiling requires a **2D `(case, scale)` grid search** filtered by five validity gates, a **semantic FLOPs numerator** bounded by `min(traced_flops, hardware_flops)` so redundant mask/recompute work cannot inflate MFU, and **per-dtype denominator calibration (`MFU_nom` vs `MFU_ach`)**.

## 6-Step Operator MFU Evaluation Pipeline

### Step 1: 2D `(case, scale)` Grid Search & Five-Gate Candidate Filter

Sweep across test cases and geometric scale multipliers (`scale = 1, 2, 4, 8, ...`) rather than assuming the largest shape yields peak MFU. Every candidate `(case, scale)` must pass **all 5 validity gates**:
1. **Memory Gate (`--max-bytes`)**: Total tensor footprint fits comfortably within HBM without OOM.
2. **Reference Traceability Gate**: The reference PyTorch `Model` executes cleanly so `FlopCounterMode` can trace baseline mathematical FLOPs.
3. **Target Execution Gate**: The custom `ModelNew` (`triton.jit` or Ascend C op) runs without runtime or accuracy errors.
4. **Latency Bound Gate (`--max-ms`)**: Single-run latency remains below `--max-ms` so profiling does not time out.
5. **Physical Bandwidth Sanity Gate**: Implied memory bandwidth does not exceed physical HBM limit (`<= 1.3 TB/s` on Ascend 910B); any measurement implying `> 1.3 TB/s` indicates a missing `torch.npu.synchronize()` or cached output bug.

### Step 2: Semantic FLOPs Numerator (`min(traced_flops, hardware_flops)`)

Never use raw hardware PMU instruction counts alone as the MFU numerator—causal masking (`triu`), unmasked padding tiles, or FlashAttention online-rescaling recomputation execute real hardware instructions that do not contribute to net algorithmic FLOPs:

$$\text{FLOPs}_{\text{effective}} = \min\left(\text{FLOPs}_{\text{traced}}(M_{\text{ref}}),\; \text{FLOPs}_{\text{hardware}}(M_{\text{new}})\right)$$

- Verify custom FLOPs formulas with `check_flops.py` (which monkey-patches `triton.jit` launches into no-ops to ensure 100% of compute is accounted for inside the kernel).

### Step 3: Per-Dtype Denominator Calibration (`MFU_nom` vs `MFU_ach`)

Run `peak_calib.py` to calibrate the denominator for the operator's exact compute unit and dtype (`Cube FP16/BF16`, `Cube INT8`, `Vector FP32`, `Vector FP16`):

$$\text{MFU}_{\text{nom}} = \frac{\text{FLOPs}_{\text{effective}} / T_{\text{op}}}{P_{\text{nominal}}(\text{dtype}, f_{\text{AIC}})}, \qquad \text{MFU}_{\text{ach}} = \frac{\text{FLOPs}_{\text{effective}} / T_{\text{op}}}{P_{\text{achieved\_gemm}}(\text{dtype})}$$

- **Multi-Kernel Decomposition**: When `ModelNew` dispatches $K$ distinct kernels ($k = 1..K$), compute per-kernel time share $w_k = t_k / \sum_j t_j$ and per-kernel $\text{MFU}_k$ from `OpBasicInfo.csv` + `ArithmeticUtilization.csv` so optimization targets the highest-weight low-MFU sub-kernel first.
