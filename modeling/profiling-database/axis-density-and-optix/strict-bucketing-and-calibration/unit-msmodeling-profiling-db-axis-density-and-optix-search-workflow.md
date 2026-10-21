---
schema: mindie-graded-unit/1
id: unit-msmodeling-profiling-db-axis-density-and-optix-search-workflow
revision: 1
title: "MindStudio-Modeling Profiling Database V1 Axis Density Calibration & OptiX Closed-Loop Search Workflow"
tier: L3
category: workflow
evidence_strength: software_contract
status: active
domain_path: modeling/profiling-database/axis-density-and-optix/strict-bucketing-and-calibration
inherits_from:
  - unit-ascend-serving-ttft-tpot-and-operator-attribution-workflow
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: modeling
triggers:
  error_signatures:
    - strict_signature mismatch
    - config_preflight failed
    - validate_candidates rejected
  keywords:
    - msmodeling
    - profiling-database-axis-density
    - strict-signature
    - max-gap
    - max-ratio
    - optix-assistant
    - pso-mode
    - benchmark-spec
provenance:
  source_type: source_contract
  source_locator: Ascend/msmodeling@1adde03a04264fa976893fdf50495270175043e6:.agents/skills/profiling_database_axis_density/SKILL.md
  evidence_refs:
    - https://gitcode.com/Ascend/msmodeling/blob/1adde03a04264fa976893fdf50495270175043e6/.agents/skills/profiling_database_axis_density/SKILL.md#L1-L150
    - https://gitcode.com/Ascend/msmodeling/blob/1adde03a04264fa976893fdf50495270175043e6/.agents/skills/optix-assistant/SKILL.md#L1-L106
  supersedes: []
---

## Preconditions & Trigger

Use this workflow when calibrating operator profiling database collection grids (`axis_collection_density.yaml` and `tools/perf_data_collection/grid_generator/`) for TensorCast's `EmpiricalPerformanceModel`, or when running live serving parameter auto-tuning with `OptiX` (`optix-assistant` in `config`, `PSO`, or `agent` mode).

## Part A: Profiling Database Axis Density Calibration (`L-1 / L0 / L+1` Ablation)

### Step 1: Classify Axis Semantics & Strict Bucketing Fields (`strict_signature`)
- Inspect `tools/perf_data_collection/grid_generator/shape_grids.py` and `tensor_cast/performance_model/profiling_database/` to separate continuous interpolation axes from **strict bucketing fields**:
  - **Strict Bucketing Fields (Never Interpolated)**: `dtype`, `format`/`layout`, `transpose`, `sparse_mode`, `latency_source`, `num_devices`, communication topology, and any enum that alters the kernel execution path. All supported enum values must be listed in `required_values`.
  - **Continuous Numeric Axes**: Eligible for `range` and density bounds (`max_gap` for linear arithmetic progression; `max_ratio` for multi-order-of-magnitude geometric progression).
- Freeze critical production queries in a versioned JSONL manifest (`{"workload_id": ..., "critical": true, "query": {...}}`) deduplicated by `(workload_id, strict_signature)`. Never downgrade a critical query to non-critical after observing a lookup `MISS`.

### Step 2: Evaluate `L-1`, `L0`, and `L+1` Candidate Grids Against Holdout
- Construct the baseline point set `L0` containing all mandatory exact anchors (critical queries, range endpoints, alignment boundaries, algorithm switch points) plus midpoint fills (`max_gap` or `max_ratio`).
- Build two ablation point sets on the exact same holdout and production interpolation path:
  - `L-1`: Remove non-mandatory interior points from `L0` while keeping range endpoints and critical anchors fixed, proving sparser sampling degrades interpolation error or coverage.
  - `L+1`: Insert valid arithmetic/geometric midpoints into `L0` intervals to locate high-curvature or tail-error regions requiring segmented density refinement.
- Verify every measured reference row has positive, finite latency (`latency > 0`), and audit each query outcome as `exact`, `interpolated`, `compatible`, or `roofline`.

## Part B: OptiX Serving Auto-Tuning Workflow (`PSO` vs. `Agent` Mode)

### Step 3: Lock `benchmark_spec` & Execute Preflight Gate (`config_preflight.py`)
- Initialize the sparse-checkout OptiX knowledge base via `python .agents/skills/optix-assistant/scripts/init_knowledge.py` (never run a full wiki submodule clone).
- Register workload parameters (`--input-len`, `--output-len`, `--num-requests`, and fixed `--concurrency` or `--concurrency-min/--concurrency-max`) into `context.json` (`benchmark_spec`) via `collect_context.py`.
- Mutate `config.toml` exclusively via `config_writer.py` (for search parameters) and `config_preflight.py --set-*` (for `[agent_optimizer]` runtime sections), then enforce consistency between `config.toml`, `ais_bench` dataset configs, and `benchmark_spec` using:
  ```bash
  python .agents/skills/optix-assistant/scripts/config_preflight.py --check-only --context <context.json>
  ```

### Step 4: Validate Candidate Combinations Before Trial Execution
- In `agent` mode, verify candidate parameter combinations against `knowledge/modeling-knowledge/relationships.md` (§1 dependencies and §2 combination restrictions)—illegal combinations are vetoed prior to execution.
- Record non-empty `evidence_analysis`, `refs_analysis`, `candidates_analysis`, and `routing_review` in the round file, validate via `scripts/validate_candidates.py --run-dir <run-dir> --round N`, and archive `context.round-N.json` after each completed round.

## Verification

- Confirm `config_preflight.py --check-only` and `validate_candidates.py` exit with code `0` before launching any OptiX trial.
