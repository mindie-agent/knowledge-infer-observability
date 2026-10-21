---
schema: mindie-graded-unit/1
id: unit-ops-test-kit-error-manager-cleaner-and-xpu-circuit-breaker-sop
revision: 1
title: "CANN ops-test-kit (TTK) Forkserver ErrorManager Cleaner, SimplifiedKey Registry & XPU Circuit Breaker SOP"
tier: L2
category: sop
evidence_strength: software_contract
status: active
domain_path: validation/ops-test-kit/forkserver-and-remote-xpu/error-cleaner-and-circuit-breaker
inherits_from:
  - unit-cpu-npu-twin-ut-st-whitebox-predicates-and-edge-shape-sop
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: validation
triggers:
  error_signatures:
    - XpuLegBrokenError
    - third-party leg failed
    - ClearErrorManager
    - FindGenSimplifiedKeyFuncs
  keywords:
    - ops-test-kit
    - ttk
    - error-manager-cleaner
    - op-registry-accessor
    - simplified-key
    - xpu-breaker
    - forkserver
provenance:
  source_type: source_contract
  source_locator: cann/ops-test-kit@92496503b70e2e05892d15dd5b78d1f80eae77df:ttk/remote/xpu_breaker.py
  evidence_refs:
    - https://gitcode.com/cann/ops-test-kit/blob/92496503b70e2e05892d15dd5b78d1f80eae77df/csrc/error_manager_cleaner/README-EN.md#L1-L36
    - https://gitcode.com/cann/ops-test-kit/blob/92496503b70e2e05892d15dd5078d1f80eae77df/csrc/op_registry_accessor/README-EN.md#L1-L42
    - https://gitcode.com/cann/ops-test-kit/blob/92496503b70e2e05892d15dd5b78d1f80eae77df/ttk/remote/xpu_breaker.py#L1-L111
  supersedes: []
---

## Preconditions & Trigger

Apply this SOP when running multi-case operator verification batches or cross-hardware (`NPU` vs. remote `XPU`/GPU) golden/performance comparisons with CANN `ops-test-kit` (`ttk`), or when diagnosing false cascading tiling failures across `forkserver` worker processes and silent empty third-party benchmark runs.

## 1. Preventing Cross-Testcase Contamination in `forkserver` Workers (`libttk_error_manager_cleaner.so`)

1. **Root Cause of Stale Tiling Failures (`PROCESS_MODE` ErrorManager)**:
   - `ttk` executes multiple operator testcases sequentially inside persistent `forkserver` worker processes to amortize Python/CANN startup cost.
   - However, CANN's `ErrorManager` (`liberror_manager.so`, `<base/err_mgr.h>`) runs in `PROCESS_MODE` and shares a single global error buffer across the process lifetime. If testcase $k$ intentionally or accidentally triggers a CANN error and leaves messages in the container, testcase $k+1$'s tiling phase reads those uncleaned error messages and falsely aborts.
2. **Mandatory Pre-Testcase Cleanup (`clear_error_manager`)**:
   - Before every `compilation_process` and `profile_process` entrypoint, invoke `ttk.core_modules.npu.error_cleaner.clear_error_manager()`, which calls `ClearErrorManager()` in `libttk_error_manager_cleaner.so` (`GetErrMgrRawErrorMessages()`) to drain and discard residual CANN error records.

## 2. Invoking Custom Operator `simplifiedKey` Callbacks (`libttk_op_registry_accessor.so`)

- Complex CANN operators (such as `MatMulV3` and `BatchMatMulV3`) register custom `gen_simplifiedkey` callbacks in `libopp_registry.so` / `libexe_graph.so` that require a synthesized `TilingContext` to compute binary-matching keys (e.g., `binary_mode2_ops.csv`).
- Use `ttk.core_modules.operator.registries` (backed by `FindGenSimplifiedKeyFuncs` and `InvokeGenSimplifiedKey` in `libttk_op_registry_accessor.so`) rather than string-templating operator keys manually; this ensures exact binary-cache hit parity with runtime `aclnn` dispatch.

## 3. Remote Third-Party (`XPU`) Circuit Breaker (`xpu_breaker.py`)

1. **Preventing Silent Empty-Data Passes (`XpuLegBrokenError`)**:
   - When collecting comparative golden/performance data from a remote third-party endpoint (`dispatch_xpu`), transient network blips return `{"status": "FAIL"}` for a few cases while the endpoint self-heals within `ENDPOINT_RECOVERY_WAIT_S` (default `180s`).
   - However, systemic configuration failures (e.g., mismatched transfer protocol, dead remote daemon, or missing remote spec dependencies) fail on *every* case; without a circuit breaker, a `--compare close` batch runs to completion with empty `device_us` metrics and falsely reports `PASS`.
2. **Flock-Synchronized Consecutive Failure Threshold (`DEFAULT_LIMIT = 30`)**:
   - `ttk/remote/xpu_breaker.py` persists cross-worker state in `<root>/.ttk/xpu_breaker.json` protected by `fcntl.flock(LOCK_EX)`:
     - Call `reset(root_path)` at batch startup to zero `consec_fail`, `total_fail`, `total_ok`, and `tripped`.
     - Every requested remote collection calls `record(root_path, ok, reason)`. Any single success (`ok=True`) resets `consec_fail = 0`.
     - When `consec_fail >= TTK_XPU_FAIL_LIMIT` (default `30`, chosen strictly larger than the `4..5` cases that fail during a transient SSH tunnel reconnect), `tripped` is set to `True` and the batch aborts immediately with `XpuLegBrokenError` and a non-zero exit code.

## Verification

- Run `pytest tests/test_accessor.py tests/cli/test_xpu_perf_collect.py` in `ops-test-kit` to confirm `libttk_error_manager_cleaner.so`, `libttk_op_registry_accessor.so`, and `xpu_breaker` trip/reset transitions behave as specified.
