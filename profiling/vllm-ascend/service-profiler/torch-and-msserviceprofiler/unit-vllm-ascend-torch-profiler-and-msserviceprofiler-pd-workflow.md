---
schema: mindie-graded-unit/1
id: unit-vllm-ascend-torch-profiler-and-msserviceprofiler-pd-workflow
revision: 1
title: "vLLM-Ascend Online Service Profiling (--profiler-config vs. MS Service Profiler) & PD Disaggregation Workflow"
tier: L3
category: workflow
evidence_strength: software_contract
status: active
domain_path: profiling/vllm-ascend/service-profiler/torch-and-msserviceprofiler
inherits_from:
  - unit-ascend-serving-ttft-tpot-and-operator-attribution-workflow
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: profiling
triggers:
  error_signatures:
    - 404 Not Found /start_profile
    - VLLM_TORCH_PROFILER_DIR deprecated
    - SERVICE_PROF_CONFIG_PATH
  keywords:
    - vllm-ascend
    - profiler-config
    - start-profile
    - stop-profile
    - msserviceprofiler
    - acl-task-time
    - libmspti
provenance:
  source_type: source_contract
  source_locator: vllm-project/vllm-ascend@175ef4d49c2072f9233619636f0309f4f88b4e2c:docs/source/developer_guide/performance_and_debug/service_profiling_guide.md
  evidence_refs:
    - https://github.com/vllm-project/vllm-ascend/blob/175ef4d49c2072f9233619636f0309f4f88b4e2c/docs/source/developer_guide/performance_and_debug/service_profiling_guide.md#L1-L250
  supersedes: []
---

## Preconditions & Trigger

Use this workflow when capturing operator-level (`Ascend PyTorch Profiler`) or service-framework-level (`MS Service Profiler` — Request, KVCache, BatchSchedule, ModelExecute, Communication) performance traces from a live `vllm-ascend` server in aggregated or Prefill-Decode (PD) disaggregated deployments.

## Track 1: Operator-Level Profiling via `--profiler-config` (`Ascend PyTorch Profiler`)

### Step 1: Launch Server with `--profiler-config` (Replacing Deprecated `VLLM_TORCH_PROFILER_DIR`)
- **Deprecation Invariant**: `VLLM_TORCH_PROFILER_DIR` is removed in mainline vLLM-Ascend (PR #5928). The `/start_profile` and `/stop_profile` HTTP routes are registered **only** when `--profiler-config` is passed at startup with a non-empty `"profiler": "torch"`; omitting it causes `curl -X POST .../start_profile` to fail with `HTTP 404 Not Found`.
- Always set `"torch_profiler_with_stack": false` unless debugging Python call stacks, as stack collection enabled by default inflates trace volume and adds host overhead:
  ```bash
  vllm serve <model_path> \
    --port 8080 \
    --profiler-config '{"profiler": "torch", "torch_profiler_dir": "./vllm_profile", "torch_profiler_with_stack": false}'
  ```

### Step 2: Trigger Collection in Aggregated vs. PD Disaggregated Topologies
- **Aggregated Serving**: Send `curl -X POST http://127.0.0.1:8080/start_profile`, issue steady-state inference requests, then send `curl -X POST http://127.0.0.1:8080/stop_profile`.
- **PD Disaggregated Serving Caveat**: The standard PD load-balance proxy (`load_balance_proxy_server_example.py`) does **NOT** forward `/start_profile` or `/stop_profile` (only the EPD layerwise proxy broadcasts them). You MUST configure distinct `torch_profiler_dir` paths on Prefill and Decode and invoke `/start_profile` and `/stop_profile` directly against **each Prefill and Decode instance's HTTP port**:
  ```bash
  curl -X POST http://<prefill_ip>:<prefill_port>/start_profile
  curl -X POST http://<decode_ip>:<decode_port>/start_profile
  # Send workload through proxy...
  curl -X POST http://<prefill_ip>:<prefill_port>/stop_profile
  curl -X POST http://<decode_ip>:<decode_port>/stop_profile
  ```
- Offline post-processing: Run `from torch_npu.profiler.profiler import analyse; analyse("./vllm_profile/*_ascend_pt/")` to materialize `ASCEND_PROFILER_OUTPUT/{trace_view.json, kernel_details.csv, step_trace_time.csv, ascend_pytorch_profiler_0.db}`.

## Track 2: Framework Workflow Profiling via `MS Service Profiler` (`msserviceprofiler`)

### Step 3: Configure `SERVICE_PROF_CONFIG_PATH` & Dynamic `enable` Toggle
- Export `SERVICE_PROF_CONFIG_PATH=./ms_service_profiler_config.json` (and optional `PROFILING_SYMBOLS_PATH=./service_profiling_symbols.yaml`) before launching `vllm serve`.
- Configure `ms_service_profiler_config.json` according to collection granularity:
  - `"enable": 0` at startup, then toggle live without restarting vLLM via `sed -i 's/"enable":\s*0/"enable": 1/' ./ms_service_profiler_config.json`.
  - `"domain": "Request;KVCache;ModelExecute;BatchSchedule;Communication"` (semicolon-separated, case-sensitive; leave empty for all domains).
  - `"acl_task_time"`: `0` (framework events only, lowest overhead), `1` (`ACL_PROF_TASK_TIME_L0`), `2` (MSPTI-based dump, requires preloading `<cann_dir>/lib64/libmspti.so` in `LD_PRELOAD`), or `3` (Torch Profiler dump).
  - `"timelimit"`: Set `>= 120` seconds (shorter windows may truncate batch/request lifecycle pairs).

### Step 4: Parse into SQLite & CSV Tables
- Parse raw records via `msserviceprofiler parse --input-path=<prof_dir> --output-path=./output` to produce `chrome_tracing.json`, `profiler.db`, `request.csv`, `kvcache.csv`, and `batch.csv`.

## Verification

- Verify `ASCEND_PROFILER_OUTPUT/trace_view.json` (Track 1) or `output/request.csv` and `output/batch.csv` (Track 2) contain complete request lifecycle timestamps without missing-domain warnings.
