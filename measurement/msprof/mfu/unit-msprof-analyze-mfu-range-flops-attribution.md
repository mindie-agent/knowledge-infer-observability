---
category: sop
conditions:
  repository: Ascend/msprof-analyze
  revision: 639eb8702c1884813dfab59ddb371f7838caab6d
content_digest: sha256:67c9ea7f5458e067a0eb36b7c16b324fb0e02e1a234eba8fd707b334aa6620a3
domain_path: measurement/msprof/mfu
evidence_strength: single_observation
id: unit-msprof-analyze-mfu-range-flops-attribution
inherits_from: []
provenance:
  evidence_refs:
  - docs/evidence/unit-msprof-analyze-mfu-range-flops-attribution-review.json
  - https://gitcode.com/Ascend/msprof-analyze/blob/639eb8702c1884813dfab59ddb371f7838caab6d/msprof_analyze/cluster_analyse/cluster_kernels_analysis/operator_mfu/mfu_calculator.py#L41-L170
  source_digest: 0a1b3e92953c2b893be5c42fbfc603c8ef6c6a3fa5508f9f1d0a5d3260cb1447
  source_locator: docs/evidence/unit-msprof-analyze-mfu-range-flops-attribution-review.json
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
tier: L2
title: Recorded FLOPs are assigned to every selected kernel in msprof MFU
triggers:
  error_signatures: []
  keywords:
  - msprof-analyze
  - recorded
  - flops
  - are
  - assigned
  - to
  - every
  - selected
  - kernel
  - in
  - msprof
  - mfu
---
At msprof-analyze revision 639eb8702c1884813dfab59ddb371f7838caab6d, MFUCalculator joins CPU-op/kernel correlation with kernel shapes on kernel_name, kernel_ts and kernel_end. A recorded MSTX FLOPs range selects op_ts with the closed condition startNs <= op_ts <= endNs, then drops duplicate kernel_ts/kernel_end pairs. The sum of task_duration is only a positivity gate. Dtype is taken from the first selected row before non-positive task durations are removed. Each remaining kernel receives the entire recorded range FLOPs, using flops / (task_duration * 1e-9) / chip_peak. FLOPs are not divided among the selected kernels. The 1e-9 conversion belongs to this database path; it does not inherit the separate CSV column label Task Duration(us). This source reading establishes the attribution formula, not a measured hardware-utilization result. Exporter schemas and clock definitions remain outside the admitted source set.

Scope and caveats:

- Keep the inclusive point-selection predicate and full-range FLOPs attribution explicit.
- Do not describe total_duration as a per-kernel FLOPs weighting.
- Keep database timing units separate from the other analyzer's CSV labels.

Adoption boundary: Use to interpret an MFU table only after checking the actual exporter clock, dtype and FLOPs-range semantics. No NPU execution or measured efficiency is established here.
This unit is a source-derived inspection procedure at the exact repository revision in conditions. It does not establish a reproduced hardware outcome or measured performance benefit.

Pinned source ranges:

- [msprof_analyze/cluster_analyse/cluster_kernels_analysis/operator_mfu/mfu_calculator.py:41-170](https://gitcode.com/Ascend/msprof-analyze/blob/639eb8702c1884813dfab59ddb371f7838caab6d/msprof_analyze/cluster_analyse/cluster_kernels_analysis/operator_mfu/mfu_calculator.py#L41-L170)
