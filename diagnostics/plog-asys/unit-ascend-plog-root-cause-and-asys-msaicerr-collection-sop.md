---
schema: mindie-graded-unit/1
id: unit-ascend-plog-root-cause-and-asys-msaicerr-collection-sop
revision: 1
tier: L2
category: sop
domain_path: diagnostics/plog-asys
evidence_strength: software_contract
title: Five-step SOP from plog first error to asys and msaicerr evidence
conditions:
  log_family: CANN plog
  default_plog_glob: "~/ascend/log/debug/plog/plog-<pid>_*.log"
  collection_tool: asys
  dump_analyzer: msaicerr
  product_form: Ascend EP
  first_error_policy: first timestamp-ordered error, not the last error
content_digest: sha256:e2bea16acbf3ff66a02e38a813271cf828853cfc7902b02e250b847e16499688
inherits_from:
- unit-observed-dual-die-phy-id-topology
provenance:
  source_type: ecosystem_repo
  source_locator: docs/evidence/cannbot-observability-provenance.json
  evidence_refs:
  - docs/evidence/cannbot-observability-provenance.json#source_records.src-asys-command
  - docs/evidence/cannbot-observability-provenance.json#source_records.src-msaicerr-command
  - docs/evidence/cannbot-observability-provenance.json#observed_measurements.obs-aicpu-507018
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:tools/asys-toolkit/SKILL.md#L71-L107
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:tools/asys-toolkit/references/analyze-files.md#L331-L389
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:tools/msaicerr-toolkit/SKILL.md#L8-L79
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:tools/msaicerr-toolkit/references/aicore-error-analysis.md#L17-L72
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:tools/tool-exception-dump/SKILL.md#L203-L229
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:ops/cann-error-code-standard/references/development-standard.md#L10-L23
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:ops/ops-error-code-remediation/references/trigger-coverage.md#L30-L37
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:ops/ascendc-runtime-debug/references/error_codes.md#L19-L27
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:ops/pypto-op-knowledge/references/experience_classified/machine.md#L26-L34
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:tools/msnpureport-toolkit/SKILL.md#L8-L18
  source_digest: 814abc1f4a1141018c23b53388f21ee219ba15a1be1b87c1543f491685c1b68d
  supersedes: []
triggers:
  error_signatures:
  - EZ9999
  - EL0004
  - EI0006
  - "507015"
  - "507018"
  - "507035"
  - "there is an aicore error exception"
  - "there is an aivec error exception"
  - "Getting socket times out"
  - "Failed to allocate memory"
  - "rtStreamSynchronize:ErrCode="
  keywords:
  - plog
  - asys
  - asys collect
  - asys analyze
  - msaicerr
  - aicore error
  - aivec error
  - CANN error code
  - RTS return code
  - HCCL socket timeout
  - memory allocation failure
  - first error
  - fault information collection
  - npu-smi
---

Use this SOP when a CANN workload fails with a plog line, an ACL/RT return code, an AI Core or AI Vector exception, or a request to collect DFX evidence. It produces a time-ordered first-error record and a `msaicerr` `info.txt`. It does not replace the analyzer's own result, and a missing input is a stop condition rather than evidence that no fault occurred.

## Step 1: Freeze the environment and collect the whole failure set

Confirm the product form and load the same CANN environment that the failed process used:

```bash
source ${INSTALL_DIR}/set_env.sh
printf '%s\n' "${ASCEND_PROCESS_LOG_PATH:-$HOME/ascend/log}"
PLOG_DIR="$HOME/ascend/log/debug/plog"
ls -lt "$PLOG_DIR"/plog-*_*.log | head
```

Filter to the failed process PID when it is known. If the printed `ASCEND_PROCESS_LOG_PATH` differs, point `PLOG_DIR` at the directory that contains `plog-*_*.log`.

`asys` is an Ascend EP tool; it is not supported on RC. Collection must not run in parallel with another collection, and the six collection-relevant environment variables (`ASCEND_PROCESS_LOG_PATH`, `NPU_COLLECT_PATH`, `DUMP_GRAPH_PATH`, `ASCEND_WORK_PATH`, `ASCEND_CACHE_PATH`, `ASCEND_CUSTOM_OPP_PATH`) must match the service runtime. Device firmware logs, stackcore and some black-box data need elevated permission. Clear or rotate the trace-log directory first if it is large.

```bash
mkdir -p "$HOME/dfx"
asys collect --tar="True" --output="$HOME/dfx/collect-$(date +%Y%m%d-%H%M%S)"
```

`--output` is a prefix: the result is `{output}/asys_output_<timestamp>`. Preserve the whole directory, not only the first log file. The minimum useful set is `dfx/log/host/cann`, `dfx/ops` (the failing kernel's `.o` and `.json`) and `dfx/data-dump`. If the operator compile cache is known, add `--task_dir=<compile-cache>` so `asys` also collects build artifacts; otherwise verify `dfx/ops` contains the failing `.o` and `.json` after collection. If device-side stackcore or UB data is needed, export it first with `msnpureport report` and parse it with `asys analyze -r=stackcore` or `-r=ub`. `msnpureport -d` uses the tool's device-ID convention, which can differ from the physical ID printed by `npu-smi`; confirm the product-specific semantics before selecting a device.

Stop condition: no plog at all. Recapture with the service's actual log path or a persistent `msnpureport` collection; do not analyze a screen excerpt as if it were a full plog.

## Step 2: Read the first error, not the loudest one

AI Core failures normally cascade. The first timestamp-ordered error is the root-cause scene; later watchdog, peer-closed and cleanup errors are secondary. Read the head of the filtered stream before reading the tail:

```bash
PLOG_DIR="$HOME/ascend/log/debug/plog"
grep -nE "\[ERROR\]|AICORE|aicore error|aivec error|rtStreamSynchronize|ErrCode=|error code =|extend info|pc start|fault kernel_name|The error from device" \
  "$PLOG_DIR"/plog-*.log | head -n 80
```

Scope the glob to the failed PID's file when it is known.

Record one tuple from the first effective line: timestamp, PID, core ID, task ID, stream ID, kernel name, core type, `error code`, `aic error mask`, `pc start`, `current`, the six `*_ERR_INFO` values, `para base`, and `The extend info: errcode`. Do not skip a line because the code is `0`. A bare `error code = 0` is not proof of no error; the extended triple or a `*_ERR_INFO` field may carry the actual bits. Likewise, a generic `EZ9999` may be the outer wrapper; the lower-level line is the one that matters.

## Step 3: Decode the six-character and RTS surface codes

These codes answer different questions. A six-character CANN code usually identifies a component or a failure class; an RTS/ACL code is a runtime return surface. Neither is a substitute for the first plog line.

| Code | Layer and meaning | Immediate action | What it does not prove |
|---|---|---|---|
| `EZ9999` | Generic internal operator/wrapper code. It is not a precise semantic code and is absent from the public documented-code catalog. | Find the lower-level EE/EI/EL code or the first `[ERROR]` line. Reclassify by actual root cause. | That the wrapper is the root cause, or that another generic code should replace it. |
| `EL0004` | Driver memory allocation failure: "Failed to allocate memory." | Check HBM/free memory, process footprint, allocator or OOM evidence and `npu-smi` state. | A kernel out-of-bounds access. |
| `EI0006` | HCCL communication error: "Getting socket times out." | Read the `Reason` field, identify the two endpoints, check both processes are alive and reachable, then inspect the HCCL group setup. Redact endpoints in the report. | A network-hardware fault by itself. |
| `507015` | ACL/RT surface for an AI Core exception, typically `rtStreamSynchronize:ErrCode=507015, desc=[aicore exception]`. | Route to the first AI Core Error plog line and `msaicerr`. | The failing instruction or error bit. |
| `507018` | AICPU/task-side exception. Observed with `AICPU error: errorCode=0x2a` and downstream ACL `507018 / 0x715002a` in an AllReduce AICPU launch. | Read the first AICPU line, identify the task/kernel and collective, and check the AICPU launch plus the downstream ACL surface. | A timeout, or a specific AICore arithmetic bit. |
| `507035` | ACL/RT vector-core exception, usually reported through `aclrtStreamSynchronize`. | Run the UB checklist: UB-side 32-byte start alignment, VEC operand start, `blockLen` units, `padParams`, dynamic chunk alignment, tmpBuf sizing and total UB usage. | A single stable `aic_error` bit. |

The documented-code evidence layers are separate: public documentation proves public semantics, not target-version registration; target registration and negative runtime behavior require their own evidence. For `507018` and `507035`, the lower-level plog line and the runtime checklist are more informative than the bare return code.

## Step 4: Run `asys analyze` or `msaicerr` on the collected set

The one-click path is:

```bash
asys analyze -r=aicore_error \
  --path="$HOME/dfx/asys_output_<timestamp>" \
  --output="$HOME/dfx/aicore-analysis"
```

The result is `{output}/asys_output_<timestamp>` with an `info.txt`. If `--path` is omitted, `asys` auto-collects using the current environment; that is convenient only when the six environment variables still match the failed run. `--output` must not be inside `--path`.

The explicit path is:

```bash
cd ${INSTALL_DIR}/tools/msaicerr
python3 msaicerr.py \
  -p "$HOME/dfx/asys_output_<timestamp>" \
  -out "$HOME/dfx/msaicerr-result" \
  -dev 0
```

`msaicerr` analysis is local-only, needs Python 3.7.5 or later, is not supported on RC, and must run from its own tool directory. `-p` and `-out` must be disjoint; do not execute inside `-p` or any of its subdirectories. Before running, verify the three elements: `dfx/data-dump` contains a dump, `dfx/ops` contains the failing kernel's `.o` and `.json`, and `dfx/log/host/cann` contains the logs. If any is missing, the tool cannot extract the full AI Core Error and the report must say which chain is unavailable.

Read the terminal's `info.txt` path. The file has six sections; the third section carries `start pc`, `current pc` and the estimated assembly offset, and a `Corrected Info` block is preferred when present because it is backfilled from the AI Core error registers. If the tool itself fails or hangs, read `debug_info.txt` in the execution directory or its `info_<timestamp>/debug_info.txt`. If multiple AI Core Errors are present, the tool analyzes only the first in log-time order; fix that one and recapture before analyzing the next.

## Step 5: Close the evidence chain and state the limits

Write a short record with one row per fact:

| Field | Required content |
|---|---|
| First error | Timestamp, PID, core/task/stream IDs, kernel name and the first effective line |
| Code path | Six-character code, RTS/ACL code, and the lower-level line that explains it |
| Analysis input | Collection directory, dump present or absent, `.o`/`.json` present or absent, plog present or absent |
| Tool output | Exact `info.txt` path, corrected versus original PC, and `debug_info.txt` path if the tool failed |
| Proof status | What the evidence proves, what it only suggests, and what is still unknown |

Verify that a lower-level first error exists before accepting `EZ9999`, `507018` or `507035` as the explanation. Verify that `EI0006`'s `Reason` names the peer and that both sides were checked. Verify that `EL0004` is supported by memory evidence. If the first issue is fixed but the workload still fails, do not re-score the old collection; collect a new one and repeat from Step 1. If the data is incomplete, stop and recapture. A wrapper code or a screen line alone is not a root-cause finding.

Adoption boundary: this SOP is a software-contract procedure pinned to the evidence packet. The exact `507018` lower-level meaning, the `EZ9999` wrapper leaf and the `msaicerr` result depend on the target CANN, driver and SoC. The commands are read-only except for the explicitly created output directories. Do not change product code or kernel sources to make a diagnosis fit.
