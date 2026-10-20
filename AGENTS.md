# Inference Observability Domain Boundaries (`domain: "infer-observability"`)

This repository (`/Volumes/data/code/knowledge-infer-observability`) stores
observability, profiler attribution, measurement, and recovery knowledge units
under the canonical domain `infer-observability`. Runtime engine code lives
exclusively in `mindie-agent/knowledge`.

## Operating Rules

1. **Domain Scope & Isolation**: Keep profiler attribution semantics, clock/time
   window accounting, telemetry/HBM breakdown methodology, capacity measurement,
   and fault-recovery diagnostics in `infer-observability`. Do not mix operator
   kernels (`operators`), general request/KV engine state machines
   (`infer-core`), build/packaging compatibility rules (`infer-ops-compat`), or
   `vllm-ascend` adapter corpus files (`vllm-ascend`) into this repository.
   Reject foreign-domain `mindie://<other-domain>/...` URIs on `explain` and
   `feedback`.
2. **Unit Protocol & Index Rebuild**: Every unit must validate as
   `mindie-graded-unit/1` with a path-independent `id`, monotonic `revision`,
   exact `content_digest`, explicit `conditions`, and line-level
   `provenance.source_locator` / `evidence_refs`. Rebuild
   `index/routing-manifest.json` and `index/shard-topology.json` via the graded
   indexer after any unit change.
3. **Evidence Honesty & Three-Valued Conditions**: Bind source-derived units to
   the exact inspected `repository` and `revision` (e.g.,
   `Ascend/msprof-analyze` @ `639eb8702c1884813dfab59ddb371f7838caab6d`). Keep
   analyzer formula inspection distinct from live hardware measurements, never
   promote `single_observation` source readings to `reproduced_empirical` or
   `L4` without real execution/ablation receipts, and never treat
   `condition_unknown` as a match.
4. **Publication Gate**: [`./publication-contract.json`](./publication-contract.json)
   (`mindie-publication-contract/2`) and thin maintenance entry points under
   `scripts/` define the validator pin and local checks; run
   `python scripts/run_all_checks.py` after any unit or index edit. Follow
   `/Volumes/data/temp/mindie-003-execution/execution-status.json` for release
   state and do not push to the public remote or cut over installed Harness pins
   before the paired runtime/plugin release is finalized.
