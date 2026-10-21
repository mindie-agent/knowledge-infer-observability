---
schema: mindie-graded-unit/1
id: unit-msmodeling-throughput-optimizer-aggregation-to-text-generate-sop
revision: 1
title: "MindStudio-Modeling ServingCast Throughput Optimizer Evidence Grading & Prefill/Decode Decomposition SOP"
tier: L2
category: sop
evidence_strength: software_contract
status: active
domain_path: modeling/serving-cast/throughput-optimizer/prefill-decode-decomposition
inherits_from:
  - unit-ascend-serving-ttft-tpot-and-operator-attribution-workflow
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: modeling
triggers:
  error_signatures:
    - percentage_breakdowns missing
    - dump-op-bound-results
    - insufficient evidence
  keywords:
    - serving-cast
    - throughput-optimizer
    - text-generate
    - prefill-batch-size
    - cube-vec-comm-mem
    - pd-ratio
provenance:
  source_type: source_contract
  source_locator: Ascend/msmodeling@1adde03a04264fa976893fdf50495270175043e6:.agents/skills/throughput-optimizer-explainer/SKILL.md
  evidence_refs:
    - https://gitcode.com/Ascend/msmodeling/blob/1adde03a04264fa976893fdf50495270175043e6/.agents/skills/throughput-optimizer-explainer/SKILL.md#L1-L117
  supersedes: []
---

## Preconditions & Trigger

Apply this SOP when analyzing parallel strategy rankings (`TP`, `DP`, `PP`, `EP`, `MOE-TP`, `MOE-DP`, and `PD ratio`) produced by `python -m cli.inference.throughput_optimizer` in `Ascend/msmodeling`, or when converting a winning `throughput_optimizer` row into standalone `python -m cli.inference.text_generate` Prefill and Decode validation runs.

## Five-Tier Evidence Classification Rule

Never overstate operator-level or hardware-bottleneck conclusions beyond the supplied artifact tier. Before attributing bottlenecks to Cube (`mma`), Vector (`gp`), Memory (`memory`), or Communication (`comm`), classify the available evidence into one of five levels:
1. **`macro_only`**: Standard `throughput_optimizer` summary table (TTFT, TPOT, concurrency, QPS, throughput) without phase breakdowns. Any bottleneck claim at this tier MUST be labeled an inference.
2. **`optimizer_phase_breakdown`**: Output from `throughput_optimizer --dump-original-results` containing `percentage_breakdowns(p)` and `percentage_breakdowns(d)`. Supports phase-level Cube/Vec/Comm/Mem attribution.
3. **`text_generate_phase_breakdown`**: Output from `python -m cli.inference.text_generate` containing stage `Stats` breakdowns.
4. **`text_generate_op_bound`**: Output from `text_generate --dump-op-bound-results` containing per-operator `Bound` classifications and `memory`/`comm`/`mma`/`gp` percentage columns. Supports **simulated** operator-level attribution (must be explicitly distinguished from real hardware profiler traces).
5. **`profiler_trace`**: Real Ascend `msprof` / `torch_npu.profiler` hardware timeline and PMU counters.

## Exact Equations Mapping Aggregated Optimizer Rows to Prefill & Decode Validation

1. **Why Aggregated Throughput Cannot Map to a Single Forward**:
   - An aggregated (`aggregation` mode) best row combines Prefill forward execution, Decode forward iterations, and continuous-batching scheduling equations. Comparing a single `text_generate` forward TPS directly against aggregated service throughput is invalid.
2. **Decompose into Two `text_generate` Commands (Prefill + Decode)**:
   Given an aggregated row with input length $I$, output length $O$, concurrency $C$, `max_batched_tokens` $M$, `prefix_cache_hit_rate` $H \in [0, 1)$, and MTP speculative tokens $N$:
   - **Effective Uncached Input Length**:
     $$I_{\text{eff}} = \max\!\big(1,\; I - \lfloor I \times H \rfloor\big)$$
   - **Prefill Validation Leg**:
     $$\text{prefill\_batch\_size} = \lfloor M / I_{\text{eff}} \rfloor, \quad \text{query\_length} = I_{\text{eff}}$$
   - **Decode Validation Leg**:
     $$\text{decode\_num\_queries} = C, \quad \text{decode\_query\_length} = N + 1, \quad \text{decode\_context\_length} = L_{\text{base\_ctx}} + \lfloor O / 2 \rfloor$$
3. **Generate Op-Bound Breakdown Commands**:
   - Run `scripts/build_text_generate_commands.py --include-op-bound` (which appends `--dump-op-bound-results`) to generate both the Prefill and Decode `text_generate` invocations, and compare phase breakdowns via `scripts/compare_phase_breakdowns.py --op-bound`.

## Verification

- Confirm every throughput explanation explicitly states its evidence tier (`macro_only` through `profiler_trace`) and concludes with a reasonableness verdict (`basically reasonable`, `partly explainable`, `suspicious`, or `insufficient evidence`) plus the minimal next verification command.
