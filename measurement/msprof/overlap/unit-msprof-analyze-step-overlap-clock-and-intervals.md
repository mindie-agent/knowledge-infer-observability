---
category: sop
conditions:
  repository: Ascend/msprof-analyze
  revision: 639eb8702c1884813dfab59ddb371f7838caab6d
content_digest: sha256:aa930636bf2fd97a76e483b379501b0ec1149919d78560cb552c3c9655b616b9
domain_path: measurement/msprof/overlap
evidence_strength: single_observation
id: unit-msprof-analyze-step-overlap-clock-and-intervals
inherits_from: []
provenance:
  evidence_refs:
  - docs/evidence/unit-msprof-analyze-step-overlap-clock-and-intervals-review.json
  - https://gitcode.com/Ascend/msprof-analyze/blob/639eb8702c1884813dfab59ddb371f7838caab6d/msprof_analyze/cluster_analyse/common_func/time_range_calculator.py#L41-L96
  - https://gitcode.com/Ascend/msprof-analyze/blob/639eb8702c1884813dfab59ddb371f7838caab6d/msprof_analyze/cluster_analyse/common_func/time_range_calculator.py#L99-L192
  - https://gitcode.com/Ascend/msprof-analyze/blob/639eb8702c1884813dfab59ddb371f7838caab6d/msprof_analyze/cluster_analyse/common_func/linearity_utils.py#L46-L119
  source_digest: 492233d66bff23074c60ec01142ab5c750364b026576ad2e63788fae4815c1b8
  source_locator: docs/evidence/unit-msprof-analyze-step-overlap-clock-and-intervals-review.json
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
tier: L2
title: Linearity selects overlapping events without clipping them to the step
triggers:
  error_signatures: []
  keywords:
  - msprof-analyze
  - linearity
  - selects
  - overlapping
  - events
  - without
  - clipping
  - them
  - to
  - the
  - step
---
At msprof-analyze revision 639eb8702c1884813dfab59ddb371f7838caab6d, LinearityUtils includes an event when event_end > step_start and event_start < step_end. The selected endpoints are not clipped to the step. Communication is filtered by parallelType; computation is not filtered by that field. Communication and step timestamps use startNs/endNs, while computation uses task_start_time/task_end_time, with no conversion in this function. By default total communication time sums raw selected intervals, including overlap. Its optional merge path merges touching or overlapping intervals. compute_uncovered_durations always merges communication and computation separately and subtracts their intersections. The raw communication total and the uncovered total can therefore use different interval sets. Each total is divided by the step duration, so a ratio can exceed one because events extend outside the step or raw communication intervals overlap. merge_continuous_intervals sorts the input list and extends its original interval objects in place.

Reviewed limits and corrections:

- An overlap filter does not establish containment or clipping.
- Use the same interval-set and clock semantics before comparing total and uncovered ratios.
- A ratio above one is a consequence permitted by this static formula, not a measured incident.

Adoption boundary: Use as the source contract for communication attribution. Validate clocks and interval normalization against each actual trace before reporting a utilization or overlap percentage.

Evidence scope: Root accepted only the rewritten mechanism. Independent Cursor reading and its broader findings remain in the immutable author packet. Root source reading is enumerated separately.

This unit is a source-derived inspection procedure at the exact repository revision in conditions. It does not establish a reproduced hardware outcome or measured performance benefit.

Pinned source ranges:

- [msprof_analyze/cluster_analyse/common_func/time_range_calculator.py:41-96](https://gitcode.com/Ascend/msprof-analyze/blob/639eb8702c1884813dfab59ddb371f7838caab6d/msprof_analyze/cluster_analyse/common_func/time_range_calculator.py#L41-L96)
- [msprof_analyze/cluster_analyse/common_func/time_range_calculator.py:99-192](https://gitcode.com/Ascend/msprof-analyze/blob/639eb8702c1884813dfab59ddb371f7838caab6d/msprof_analyze/cluster_analyse/common_func/time_range_calculator.py#L99-L192)
- [msprof_analyze/cluster_analyse/common_func/linearity_utils.py:46-119](https://gitcode.com/Ascend/msprof-analyze/blob/639eb8702c1884813dfab59ddb371f7838caab6d/msprof_analyze/cluster_analyse/common_func/linearity_utils.py#L46-L119)
