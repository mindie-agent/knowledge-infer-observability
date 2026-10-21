---
schema: mindie-graded-unit/1
id: unit-vllm-ascend-ms-service-metric-multiproc-and-symbol-hook-sop
revision: 1
title: "vLLM-Ascend MS Service Metric Prometheus Multiprocess Directory & Symbol Hook Injection SOP"
tier: L2
category: sop
evidence_strength: software_contract
status: active
domain_path: profiling/vllm-ascend/prometheus-metrics/ms-service-metric-hooks
inherits_from:
  - unit-ascend-serving-ttft-tpot-and-operator-attribution-workflow
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: profiling
triggers:
  error_signatures:
    - PROMETHEUS_MULTIPROC_DIR
    - MS_SERVICE_METRIC_CONFIG_PATH
    - ms-service-metric
  keywords:
    - vllm-ascend
    - ms-service-metric
    - prometheus-multiproc-dir
    - observability
    - model-runner-phase-handler
    - symbol-hook
provenance:
  source_type: source_contract
  source_locator: vllm-project/vllm-ascend@175ef4d49c2072f9233619636f0309f4f88b4e2c:docs/source/user_guide/ms_service_metric.md
  evidence_refs:
    - https://github.com/vllm-project/vllm-ascend/blob/175ef4d49c2072f9233619636f0309f4f88b4e2c/docs/source/user_guide/ms_service_metric.md#L1-L67
  supersedes: []
---

## Preconditions & Trigger

Apply this SOP when enabling non-intrusive Prometheus runtime telemetry (`ms-service-metric`) on `vllm-ascend` worker processes, or when injecting custom function-level latency histograms and gauges via `MS_SERVICE_METRIC_CONFIG_PATH` and `vllm_ascend/observability/config/`.

## Per-Instance `PROMETHEUS_MULTIPROC_DIR` Isolation Contract

1. **Shared-Memory Multiprocess Directory Setup**:
   - Because `vllm-ascend` spawns multiple TP/DP worker subprocesses alongside the API server, `ms_service_metric` aggregates metrics across processes using `prometheus_client` multiprocess mmap files in `PROMETHEUS_MULTIPROC_DIR`:
     ```bash
     export PROMETHEUS_MULTIPROC_DIR="<shm_metrics_dir>"
     mkdir -p "$PROMETHEUS_MULTIPROC_DIR"
     ms-service-metric on
     vllm serve <model_path>
     ```
2. **Multi-Instance & Restart Hygiene Rules**:
   - **Per-Instance Isolation**: When running multiple `vllm serve` instances (such as Prefill and Decode instances on the same host), each instance MUST export a **distinct** `PROMETHEUS_MULTIPROC_DIR` path; sharing a directory across instances corrupts gauge/histogram buckets.
   - **Safe Stale-File Cleanup**: Before restarting an instance for validation, delete stale `.db` files inside `PROMETHEUS_MULTIPROC_DIR` *only after* verifying that no live worker process still holds open file descriptors to that directory.

## Custom Symbol Hook Configuration (`MS_SERVICE_METRIC_CONFIG_PATH`)

1. **Declarative YAML Function Hooking**:
   - `ms-service-metric` automatically loads built-in YAML definitions from `vllm_ascend/observability/config/` and optional user-defined YAML files from `MS_SERVICE_METRIC_CONFIG_PATH` (which may point to a single `.yaml` file or a directory of first-level `.yaml`/`.yml` files loaded in lexical order without recursive subdirectory traversal).
   - Each hook entry binds a fully qualified target Python symbol (`module:Class.method` or `module:function`) to a metric handler:
     ```yaml
     - symbol: vllm_ascend.worker.model_runner_v1:NPUModelRunner.execute_model
       handler: ms_service_metric.provider_handlers:model_runner_phase_handler
       metrics:
         - name: executor:model_runner_execute_model:duration
           type: histogram
     ```
2. **Fault-Isolation & Reload Semantics**:
   - **Graceful Degradation Invariant**: If a configured `symbol` no longer exists after an upstream refactor, `ms-service-metric` disables only that specific metric point and never crashes or blocks inference execution.
   - **Reload Boundary**: Run `ms-service-metric restart` after editing YAML metric definitions; however, if Python handler code (`vllm_ascend/observability/handlers.py` or external handler modules in `MS_SERVICE_METRIC_CONFIG_PATH`) is modified, you MUST restart the `vllm serve` process because imported Python handler modules are not hot-reloaded into active worker processes.

## Verification

- Send a test completion request and run `curl -s http://127.0.0.1:8000/metrics | grep executor:model_runner_execute_model:duration` to confirm non-zero histogram bucket counts.
