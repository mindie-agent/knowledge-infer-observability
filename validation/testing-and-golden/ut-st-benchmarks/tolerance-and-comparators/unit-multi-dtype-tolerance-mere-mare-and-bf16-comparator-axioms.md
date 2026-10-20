---
category: axiom
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
  - PyTorch (`torch_npu`)
  - GTest
  workload:
  - Operator accuracy verification, golden reference construction, and C++/Python
    comparator design
created_at: '2026-10-21'
domain_path: validation/testing-and-golden/ut-st-benchmarks/tolerance-and-comparators
estimate_tokens: 600
evidence_strength: software_contract
id: unit-multi-dtype-tolerance-mere-mare-and-bf16-comparator-axioms
inherits_from:
- unit-observed-dual-die-phy-id-topology
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/cann/ops-precision-standard/SKILL.md#L1-L210
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/knowledge/contrib/ascendc/guides/testing/standalone_host_bf16_and_comparator_counterexamples.md#L1-L120
  - https://gitcode.com/cann/cannbot-knowledge/blob/be5882ef29fa9943305085652d2025a038ba491c/knowledge/contrib/ascendc/guides/testing/registry_st_comparison_failures_and_coverage.md#L1-L120
  promotion_status: verified
  source_locator: cann/cannbot-skills@f305e200aa4b1e1fed73cd21ff1f083b8df25a1e:cann/ops-precision-standard/SKILL.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Defines multi-dtype accuracy metrics (MERE/MARE and element error-count ratios),
  bans silent-pass C++ comparators (std::max with NaN, FloatToBf16 round-to-odd),
  and locks golden upcast rules.
tier: L1
title: Multi-Dtype Precision Standards (MERE/MARE vs allclose), NaN Comparator Traps,
  and Host BF16 Conversion Axioms
triggers:
  error_signatures:
  - std::max(0, NaN)
  - FloatToBf16
  - MERE
  - MARE
  keywords:
  - ops-precision-standard
  - round-to-odd
  - round-to-nearest-even
  - comparator silent pass
updated_at: '2026-10-21'
---
# Multi-Dtype Precision Standards (MERE/MARE vs allclose), NaN Comparator Traps, and Host BF16 Conversion Axioms

## Core Invariant

An accuracy test is only as trustworthy as its golden reference and comparator implementation. A comparator that uses `std::max(max_err, err)` when `err` is `NaN`, a host helper that converts `float` to `bfloat16` via bit-shift truncation or round-to-odd (`|= 1`), or a golden function that mutates input tensors in-place creates either **silent false passes** on broken kernels or **permanent 1-ULP false failures** on compliant hardware.

## Axiom 1: Multi-Dtype Tolerance Thresholds & Dual Gate (`allclose` + Error Ratio / MERE / MARE)

1. **Upcast Golden Rule**: Low-precision operators (`FP16`, `BF16`, `HIF8`, `FP8`, `INT8`) **must** compute their golden reference in higher precision (`FP32` on CPU/NPU, or `FP64` for reduction/norm/accumulation ops) and cast back to the target dtype only at the final output boundary.
2. **Standard Dtype Thresholds (`ops-precision-standard`)**:

| Data Type | Elementwise `rtol` / `atol` | Reduction / Composite `rtol` / `atol` | Max Failing Element Ratio (`err_ratio`) |
| --- | --- | --- | --- |
| `FP32` | `1e-4` / `1e-5` (strict `1e-5` / `1e-6`) | `1e-4` / `1e-5` | `<= 1e-4` (`0.01%`) |
| `FP16` | `1e-3` / `1e-5` | `1e-3` / `1e-4` | `<= 1e-3` (`0.1%`) |
| `BF16` | `4e-3` / `1e-5` | `1e-2` / `1e-3` | `<= 4e-3` (`0.4%`) |
| `INT8` / `INT32` / `BOOL` | `0` / `0` (exact match, or `atol=1` for quantized rounding) | `0` / `1` | `0%` |

3. **MERE & MARE for Tensor-Wide Quality**: Alongside per-element bounds ($|y - y_{\text{ref}}| \le \text{atol} + \text{rtol} \cdot |y_{\text{ref}}|$), evaluate **Maximum Element Relative Error (MERE)** and **Mean Absolute Relative Error (MARE)** with denominator floor $\epsilon$ to avoid division-by-zero blowup near zero crossings.

## Axiom 2: Five Forbidden Comparator & Golden Counterexamples

1. **The `std::max(max_err, NaN)` Silent-Pass Bug**:
   - In C++, `std::max(a, b)` returns `a` when `a < b` is `false`. Because `0.0f < NAN` evaluates to `false`, `max_err = std::max(max_err, abs_err)` **silently ignores `NaN` outputs** and reports `max_err == 0.0`!
   - **Rule**: Every comparator loop **must explicitly check `std::isnan(actual[i]) != std::isnan(expected[i])` and `std::isinf(actual[i]) != std::isinf(expected[i])` first**, failing immediately before computing arithmetic differences.
2. **Host `FloatToBf16` Round-to-Odd / Truncation Mismatch**:
   - Ascend AI Core `vconv` (`FP32 -> BF16`) implements IEEE 754 **round-to-nearest-even (RNE)**:
     ```cpp
     uint32_t bits = *reinterpret_cast<const uint32_t*>(&val);
     uint32_t lsb = (bits >> 16) & 1U;
     uint32_t rounding_bias = 0x7FFFU + lsb;
     uint16_t bf16_bits = static_cast<uint16_t>((bits + rounding_bias) >> 16);
     ```
   - Never use naive `bits >> 16` (round-toward-zero) or sticky-bit `|= 1` (round-to-odd) in standalone C++ UT/ST host helpers; doing so injects systematic 1-ULP discrepancies against hardware RNE.
3. **In-Place Golden Input Mutation**:
   - If the candidate NPU kernel runs before the golden reference and writes into an aliased input buffer (or vice versa), the golden function computes over already-mutated inputs. Always clone inputs before invoking the candidate kernel.
4. **FP64-to-FP32 Narrowing in Offset/Scan Accumulators**:
   - Host C++ golden loops for cumulative prefix scans (`cumsum`, `cumprod`, `linspace`) must accumulate in `double` (`FP64`) before casting each element to the output dtype; accumulating in `float` drifts beyond `1e-4` after $N > 4096$ steps.
5. **Statistical Distribution Tests (`Normal` / `Uniform` / `Dropout`)**:
   - Never use elementwise `allclose` on stochastic RNG operators; validate sample mean, variance, skewness, kurtosis, and zero-mask probability over $N \ge 10^5$ elements with analytical confidence intervals.
