---
category: workflow
conditions:
  domain: profiling
domain_path: profiling/msprof-analyze/cluster-fast-slow-rank/pseudo-fast-rank-and-wait
evidence_strength: reproduced_empirical
id: unit-msprof-analyze-cluster-pseudo-fast-rank-and-free-time-workflow
inherits_from: []
provenance:
  evidence_refs:
  - https://gitcode.com/Ascend/msagent/blob/521f8ead0ebda5db5b282f0c3c4147006e01e016/skills/profiler/ascend-cluster-fast-slow-rank-detector/SKILL.md#L1-L121
  - https://gitcode.com/Ascend/msprof-analyze/blob/639eb8702c1884813dfab59ddb371f7838caab6d/README_EN.md#L21-L50
  source_locator: Ascend/msagent@521f8ead0ebda5db5b282f0c3c4147006e01e016:skills/profiler/ascend-cluster-fast-slow-rank-detector/SKILL.md
  source_type: reproduced_run+source_contract
  supersedes: []
revision: 2
schema: mindie-graded-unit/1
status: active
tier: L3
title: Ascend msprof-analyze Cluster Fast/Slow Rank Triage & Host-Bound Pseudo-Fast
  Rank Detection Workflow
triggers:
  error_signatures:
  - slow_rank
  - slow_link
  - Free Time > 10%
  keywords:
  - msprof-analyze
  - cluster-time-summary
  - compute-op-sum
  - hccl-sum
  - cann-api-sum
  - pseudo-fast-rank
  - host-bound
  - straggler
---
## Applicability boundary for offline analysis

Offline rank attribution uses the supplied trace clocks, windows and rank identities. It does not require the analyst to supply an unobserved SoC, live synchronization policy or machine-topology digest. Verify the actual exporter/analyzer formulas and schema before comparing durations; `unit-msprof-analyze-step-overlap-clock-and-intervals` is a source-revision-specific reference when that implementation is being inspected, not an unconditional prerequisite for every trace. Treat the signatures below as hypotheses until the trace supports them. Re-collection, link bandwidth and thermal explanations additionally require the actual device, transport and measurement protocol; a passing offline parser check does not verify those claims.

## Preconditions & Trigger

Use this workflow when analyzing multi-rank or multi-node Ascend profiling directories (`ascend_pytorch_profiler_{rank_id}.db` or text traces) with `msprof-analyze` to isolate straggler ranks ("快慢卡"), uneven MoE/DP load balance, degraded HCCS/RoCE links, or CPU host-dispatch stalls.

## Expert Diagnostic Axiom: The "Pseudo-Fast Rank" (`Free Time` Illusion)

Never label a rank as the "fast rank" solely because its `Communication Time` (or `Communication Wait`) is minimal!
1. **Host-Dispatch Bottleneck ("伪快卡" / Pseudo-Fast Rank)**:
   - **Symptom**: Rank $X$ exhibits abnormally high `Free Time` (`> 10%` of step duration or far above cluster mean) while its `Compute Time` and `Communication Time` appear unusually short, whereas all peer ranks show massive `Communication Wait` time.
   - **Root Cause**: Rank $X$ is **the true straggler (slow rank)**, not the fast rank! CPU-side Python/CANN dispatch slowdown or synchronization (`aclrtSynchronizeDevice`, GIL contention, dataloader stall, or NUMA unpinned threads) starves Rank $X$'s NPU stream, inflating its device `Free Time`. By the time Rank $X$ finally launches its collective kernel, all peer ranks have already been waiting at the HCCL barrier; thus Rank $X$'s collective completes instantaneously with near-zero wait time.
2. **True Compute Straggler**:
   - **Symptom**: `Free Time` is uniformly low across all ranks, but Rank $Y$'s `Compute Time` significantly exceeds the cluster mean.
   - **Sub-Classification**: If operator `count` differs across ranks, the root cause is workload/expert/sequence-length imbalance; if `count` is identical but `avg_time` spikes on Rank $Y$, the root cause is dynamic shapes or hardware thermal/frequency throttling on Rank $Y$.
3. **True Communication / Slow-Link Bottleneck**:
   - **Symptom**: Collective transit/bus bandwidth (`SDMA` / `RDMA` in `hccl_sum` and `slow_link`) drops far below hardware spec (e.g., SDMA `< 2 GB/s`) due to tiny fragmented collectives, unaligned tensor addresses, or RoCE packet loss.

## Step-by-Step Cluster Triage Workflow

### Step 1: Execute Granular `msprof-analyze` Cluster Recipes
- Do not rely solely on `-m all`; execute each cluster analysis recipe into isolated output directories so every macro table is materialized:
  ```bash
  for mode in cluster_time_summary compute_op_sum hccl_sum slow_rank slow_link cann_api_sum; do
    msprof-analyze -m "$mode" -d ./cluster_data -o "./output/${mode}"
  done
  ```
- Inspect `cluster_time_summary` for per-rank `Free`, `Compute`, `Communication (Not Overlapped)`, and `Stage` durations alongside `slow_rank` and `slow_link`.

### Step 2: Run Microscopic Fast-vs-Slow Rank Diffs (`compare_op_stats` & `compare_api_stats`)
- Once the true slow rank (`<slow_rank_id>`) and baseline normal rank (`<fast_rank_id>`) are identified from Step 1:
  - **For Host-Dispatch / Pseudo-Fast Rank (`Free Time` anomaly)**:
    ```bash
    python skills/profiler/ascend-cluster-fast-slow-rank-detector/scripts/compare_api_stats.py \
      ./cluster_data <slow_rank_id> <fast_rank_id> --top 20 --json
    ```
    Focus on `aclrtLaunchKernel`, `aclrtSynchronizeStream`, and `aclrtSynchronizeDevice` call counts and inter-call launch gaps.
  - **For Compute Straggler (`Compute Time` anomaly)**:
    ```bash
    python skills/profiler/ascend-cluster-fast-slow-rank-detector/scripts/compare_op_stats.py \
      ./cluster_data <slow_rank_id> <fast_rank_id> --top 20 --json
    ```
    Distinguish operator `count` skew from per-call `avg_time` regression.

### Step 3: Run Single-Rank `advisor` on the Isolated Straggler
- Run `msprof-analyze advisor all -d <slow_rank_profiling_dir> -o ./advisor_slow_rank` to generate actionable CPU-affinity binding (`mindstudio-cpu-binding`), operator fusion, or stream synchronization fixes for the straggler rank.

## Verification

- After applying the fix (e.g., CPU core affinity binding or eliminating host sync points on Rank $X$), re-collect cluster traces and verify Rank $X$'s `Free Time` drops below `< 5%` and peer ranks' `Communication Wait` collapses correspondingly.
