---
schema: mindie-graded-unit/1
id: unit-mindie-sd-perf-gate-window-ab-and-accuracy-gate-sop
title: MindIE-SD and vLLM-Omni perf-gate same-window A/B/A drift correction and accuracy-gate three-tier equivalence SOP
tier: L2
category: sop
evidence_strength: reproduced_empirical
domain_path: validation/mindie-sd-and-omni/gates
inherits_from:
  - unit-ascend-msprof-aicore-pipe-and-host-wall-clock-axioms
  - unit-multi-dtype-tolerance-mere-mare-and-bf16-comparator-axioms
conditions:
  cann_versions:
    - ">=8.5.0"
  hardware:
    - Ascend 910B (Atlas A2)
    - Ascend 910C (Atlas A3)
    - Ascend 950PR/DT
  workloads:
    - Shared multi-NPU performance benchmarking, drift correction, and lossless/lossy accuracy verification for DiT and Omni pipelines
triggers:
  error_signatures:
    - "cross-window wall-clock drift exceeds optimization delta on shared NPU host"
    - "output md5 matches baseline only because optimization branch was silently bypassed"
    - "curl time_total inflated by profiler capture or queue contention"
  keywords:
    - perf-gate
    - accuracy-gate
    - window-ab-protocol
    - engagement_check
    - bit_exact_harness
    - silent-failure-localization
    - content-health-gate
    - layer_bisect
    - op_threshold_sweep
provenance:
  source_type: architecture_and_runtime_guide
  source_locator: Ascend/MindIE-SD@9fe772dfb90696e21b5e5d4e92ca07dffab84cf4:.agents/skills/perf-gate/SKILL.md
  source_digest: 995b847b6c689a49ee6e45bac0af3ca4df705cb5796da29ed61a252dd7de6fb9
  evidence_refs:
    - Ascend/MindIE-SD@9fe772dfb90696e21b5e5d4e92ca07dffab84cf4:.agents/skills/perf-gate/SKILL.md
    - Ascend/MindIE-SD@9fe772dfb90696e21b5e5d4e92ca07dffab84cf4:.agents/skills/accuracy-gate/SKILL.md
  supersedes: []
revision: 1
---
## Standard Operating Procedure

1. **Enforce `perf-gate` Dual-State Separation & Same-Window A/B/A Drift Correction**:
   - **Exploratory vs Acceptance State**:
     - **Exploratory (`[探索]`)**: Single-shot or few-step probes used to screen candidates. Must be explicitly tagged `[探索]` and **must never** be written into overview acceptance tables.
     - **Acceptance**: Requires same-window control + experimental arms (1 cold warmup request recorded separately + $\ge 3$ hot requests within the same server run, excluding any profiler-instrumented request from latency averages).
   - **A/B/A Bracketing (`base_1 -> arm -> base_2`)**: On shared multi-NPU hosts where thermal or neighbor load causes monotonic intra-window drift comparable to the optimization delta ($\sim 1\text{--}3\%$), bracket the experimental arm between two control runs and compute net delta as:
     $$\Delta_{\text{net}} = t_{\text{arm}} - \frac{t_{\text{base\_1}} + t_{\text{base\_2}}}{2}$$
   - **Three-Proof Rule for Every Reported Number**: Every latency or throughput claim requires (1) `HTTP 200` status code, (2) verified output byte size (preventing 500 error JSON payloads from being counted as fast completions), and (3) server-side stage breakdown log lines (cross-checked against client `curl` `time_total`).
   - **Bidirectional Engagement Check (`engagement_check.py`)**: Because `md5(arm) == md5(base)` can mean either "lossless optimization" or "the optimization gate silently fell back and never ran", prove engagement with runtime hit/fallback counters and remote file hashes (ON arm hit count $> 0$; OFF arm hit count $== 0$).

2. **Enforce `accuracy-gate` Three-Tier Equivalence Hierarchy (In Order)**:
   - **Faithful Reference Prerequisite**: Before comparing any optimized arm against a baseline, verify that the baseline faithfully applies all functional components (LoRA adapters, hooks, conditioning branches). Bypassing half of a LoRA adapter's MLP delta (`fc1`/`fc2` fast-path bypass) invalidates any PSNR/SSIM comparison against that unfaithful baseline.
   - **Tier ① — Same-Config Repeatability (`L1` Bit-Exact `torch.equal`)**: Run the exact same configuration twice (same topology, card count, seed, and shapes) and assert `torch.equal` per shape. Embed `bit_exact_harness.py` assertions directly inside replaced module `forward` methods during development to catch index/phase/order bugs on first invocation.
   - **Tier ② — Cross-Config Numerical Gate + End-to-End `md5` Parity (`L1` / `L2`)**: Compare baseline vs modified config in the same window. Any change in output `md5` when no lossy feature is enabled requires either an explicit `L2` numerical tolerance justification (e.g., dtype change or multi-rank reduction order) or immediate rollback.
   - **Tier ③ — Lossy Quality & Content-Health Gate (`L3`)**:
     - For quantization, sparsity, step caching, or few-step distillation: require **Quantitative Gate** (PSNR / SSIM / LPIPS against frozen same-seed baseline) + **Visual Artifact Gate** (`evals/rubrics/visual-artifact-gate.md`) + **Off-Identity Attribution** (turning the feature off restores baseline behavior).
     - **Machine-Readable Content-Health Gate (`content-health-gate.md`)**: For preview-grade VAE replacements or headless runs where PSNR is insensitive to high-frequency loss or scale shifts, evaluate **grayscale correlation** (scale-invariant check that latent wiring is correct), **clamped pixel ratio** (`[0, 1]` saturation check), **output standard deviation**, and **inter-frame difference / high-frequency energy buckets**.

3. **Localize Silent Numerical Failures (`silent-failure-localization.md`)**:
   - When output is visually corrupted or `md5` diverges with zero errors and zero NaNs, **audit indexing, phase, stride, padding, and channel order before blaming floating-point precision**:
     1. Establish CPU ground truth on identical inputs.
     2. Run `layer_bisect.py` to find the **first diverging layer/block**.
     3. Isolate the single operator and run `op_threshold_sweep.py` across batch/sequence sizes to distinguish shape-threshold bugs and stale HBM reads from numerical rounding drift.
