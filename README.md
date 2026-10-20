# Inference Observability, Measurement & Recovery Knowledge (`infer-observability`)

This independent local repository (`/Volumes/data/code/knowledge-infer-observability`,
canonical domain `infer-observability`) holds source-verified `L2`
procedures and mechanisms for inference observability, profiler attribution
formulas, step/communication/computation interval accounting, capacity
measurement, and fault-recovery diagnostics. Runtime indexing, query,
distillation, and synchronization code remains in `mindie-agent/knowledge`.
Authoritative validator pinning and program execution status are recorded in
[`./publication-contract.json`](./publication-contract.json) and
`/Volumes/data/temp/mindie-003-execution/execution-status.json`.

## Directory Structure & Admitted Units

- `measurement/msprof/mfu/unit-msprof-analyze-mfu-range-flops-attribution.md`
  (`L2`, `category: sop`, `evidence_strength: single_observation`,
  `revision: 1`,
  `content_digest: sha256:67c9ea7f5458e067a0eb36b7c16b324fb0e02e1a234eba8fd707b334aa6620a3`)
- `measurement/msprof/overlap/unit-msprof-analyze-step-overlap-clock-and-intervals.md`
  (`L2`, `category: sop`, `evidence_strength: single_observation`,
  `revision: 1`,
  `content_digest: sha256:aa930636bf2fd97a76e483b379501b0ec1149919d78560cb552c3c9655b616b9`)
- `docs/evidence/reviewed-source-admission-v1.json` — Admission provenance from
  `/Volumes/data/temp/mindie-003-execution/wave4/research/editorial-review-packets/root-observability-ops-editorial-review.json`,
  plus per-unit review records under `docs/evidence/*-review.json`.
- `index/routing-manifest.json` & `index/shard-topology.json` — Derived routing
  and shard indexes rebuilt from the Markdown units.
- `publication-contract.json`, `.github/workflows/check.yml`, & `scripts/`
  (`validate_units.py`, `rebuild_index.py`, `check_publication_contract.py`,
  `apply_feedback.py`, `run_all_checks.py`) — Local `mindie-publication-contract/2`
  declaration and thin CLI wrappers delegating to `mindie_knowledge.graded`.

## Unit Protocol, Evidence & Revision Rules

- **Schema**: Every unit conforms to `mindie-graded-unit/1` (`id`, `revision`,
  `content_digest`, `tier`, `category`, `evidence_strength`, `title`,
  `domain_path`, `inherits_from`, `conditions`, `triggers`, `provenance`).
- **Exact Source Conditions**: Both initial units are bound to
  `conditions.repository: "Ascend/msprof-analyze"` and
  `conditions.revision: "639eb8702c1884813dfab59ddb371f7838caab6d"`. Default
  `Store` queries match only when the exact `repository` and `revision` are
  supplied; mismatched revisions are rejected and omitted revisions remain
  `condition_unknown` (visible only when `include_unknown=True`).
- **Evidence Scope & Revision Discipline**: These `L2` units document static
  source attribution and interval-overlap formulas (`MFUCalculator` full-range
  FLOPs assignment per selected kernel and `LinearityUtils` unclipped step
  overlap vs. merged uncovered durations) and do not claim live NPU utilization
  measurements. `id` remains stable across directory moves; any substantive
  edit must increment `revision`, recompute `content_digest`, and rebuild
  `index/*.json`.

## Publication & Strict `Feed` Boundary

Validator revision and contract metadata are tracked in
[`./publication-contract.json`](./publication-contract.json), and live release
readiness is tracked in `/Volumes/data/temp/mindie-003-execution/execution-status.json`.
Although the public GitHub remote
`https://github.com/mindie-agent/knowledge-infer-observability` has been created,
new content has not yet been pushed (`new_content_pushed: false`) and the paired
formal runtime/plugin release cutover remains pending.
