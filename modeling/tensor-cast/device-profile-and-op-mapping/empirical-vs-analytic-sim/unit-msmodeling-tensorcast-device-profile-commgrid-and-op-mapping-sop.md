---
schema: mindie-graded-unit/1
id: unit-msmodeling-tensorcast-device-profile-commgrid-and-op-mapping-sop
revision: 1
title: "MindStudio-Modeling TensorCast DeviceProfile, CommGrid Invariants & op_mapping.yaml Calibration SOP"
tier: L2
category: sop
evidence_strength: software_contract
status: active
domain_path: modeling/tensor-cast/device-profile-and-op-mapping/empirical-vs-analytic-sim
inherits_from:
  - unit-ascend-serving-ttft-tpot-and-operator-attribution-workflow
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: modeling
triggers:
  error_signatures:
    - Duplicate DeviceProfile name
    - grid.ndim != len(topologies)
    - op_mapping.yaml missing kernel
  keywords:
    - msmodeling
    - tensor-cast
    - device-profile
    - comm-grid
    - op-mapping
    - empirical-performance-model
    - analytic-performance-model
provenance:
  source_type: source_contract
  source_locator: Ascend/msmodeling@1adde03a04264fa976893fdf50495270175043e6:AGENTS.md
  evidence_refs:
    - https://gitcode.com/Ascend/msmodeling/blob/1adde03a04264fa976893fdf50495270175043e6/AGENTS.md#L56-L207
    - https://gitcode.com/Ascend/msmodeling/blob/1adde03a04264fa976893fdf50495270175043e6/docs/ai-native/architecture.md#L1-L44
  supersedes: []
---

## Preconditions & Trigger

Apply this SOP when adding a new NPU hardware specification (`DeviceProfile`), defining multi-dimensional communication topologies (`CommGrid`), adapting a PyTorch model under `tensor_cast/transformers/builtin_model/`, or binding simulated operators to NPU kernel profiling records via `op_mapping.yaml` in `Ascend/msmodeling`.

## DeviceProfile & CommGrid Structural Invariants

1. **`DeviceProfile` Singleton Registration (`tensor_cast/device.py`)**:
   - `DeviceProfile.__post_init__` automatically registers each instantiated profile into the global `all_device_profiles` registry keyed by `name`.
   - **Uniqueness Invariant**: Every `DeviceProfile.name` MUST be globally unique across built-in and custom profiles; always check `all_device_profiles` before registering a new hardware variant and place custom hardware profiles under `tensor_cast/device_profiles/`.
2. **`CommGrid` Dimensionality & Topology Mapping**:
   - Multi-dimensional collective communication grids (`CommGrid`) enforce three strict structural invariants:
     1. `grid.ndim == len(topologies)`: The number of grid dimensions must equal the number of topology entries.
     2. Every dimension size in `grid` MUST be `>= 2` (never include degenerate size-1 axes in `CommGrid`).
     3. Keys in the `topologies` dictionary represent zero-based `start_dim` indices (`0, 1, ...`) mapping each parallel dimension (e.g., intra-node HCCS vs. inter-node RoCE) to its physical link bandwidth and latency model.

## Model Adaptation & Hybrid Performance Modeling (`Empirical` vs. `Analytic`)

1. **Non-Intrusive Model Wrapper Adaptation**:
   - Adapt upstream HuggingFace/Transformers models exclusively via wrapper or patch modules under `tensor_cast/transformers/builtin_model/`, preferring composition over monkey-patching and never mutating third-party site-packages in place.
   - **Zero Host-Device Sync Rule**: Never invoke `tensor.item()` or dynamic scalar extraction on the simulation/execution hot path, which triggers synchronous CPU-NPU stalls during tracing or meta-tensor replay.
2. **Bridging `EmpiricalPerformanceModel` and `AnalyticPerformanceModel` via `op_mapping.yaml`**:
   - TensorCast combines two performance estimators:
     - **`EmpiricalPerformanceModel`**: Queries measured NPU kernel latencies from the profiling SQLite/CSV database.
     - **`AnalyticPerformanceModel`**: Computes roofline/complexity estimates (Cube MMA, Vector, Memory bandwidth, and HCCL communication) when measured points are absent.
   - `op_mapping.yaml` binds each high-level TensorCast simulated operator to its underlying CANN/ATB/AscendC kernel name and axis extraction rules. Whenever adapting a new model (`model-adaptation` skill), reconcile the simulated operator invocation counts against a real single-step NPU trace (`doctor report`) before running fleet-level throughput sweeps.

## Verification

- Run `uv run pytest` on the affected `tensor_cast` unit tests and execute `python -m cli.inference.text_generate` on the adapted model profile to verify operator call counts and non-zero simulated stage latencies.
