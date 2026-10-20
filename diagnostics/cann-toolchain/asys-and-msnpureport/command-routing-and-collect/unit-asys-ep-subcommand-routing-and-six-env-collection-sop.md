---
category: sop
conditions:
  hardware:
  - Ascend 910B (A2)
  - Ascend 910C (A3)
  - Atlas 300I Duo
  os:
  - Linux
  software_stack:
  - CANN 8.0+
  - asys
  workload:
  - Ascend EP failure log bundling, stackcore/coretrace extraction, and hardware stress
    diagnostics
created_at: '2026-10-21'
domain_path: diagnostics/cann-toolchain/asys-and-msnpureport/command-routing-and-collect
estimate_tokens: 580
evidence_strength: software_contract
id: unit-asys-ep-subcommand-routing-and-six-env-collection-sop
inherits_from:
- unit-ascend-plog-root-cause-and-asys-msaicerr-collection-sop
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/cann/asys-Troubleshooting/SKILL.md#L1-L395
  promotion_status: verified
  source_locator: cann/cannbot-skills@f305e200aa4b1e1fed73cd21ff1f083b8df25a1e:cann/asys-Troubleshooting/SKILL.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Selects the exact asys subcommand across collect, launch, analyze, info,
  health, diagnose, config, and profiling modes while enforcing the six-variable environment
  consistency contract.
tier: L2
title: asys EP Subcommand Matrix, Output Prefix Rules, and Six-Var Environment Collection
  SOP
triggers:
  error_signatures:
  - asys collect --task_dir=
  - asys analyze -r=aicore_error
  - asys diagnose -r=stress_detect
  keywords:
  - asys launch
  - asys analyze stackcore
  - ASCEND_WORK_PATH
  - asys_output_
updated_at: '2026-10-21'
---
# asys EP Subcommand Matrix, Output Prefix Rules, and Six-Var Environment Collection SOP

## Core Principle

`asys` is the unified Ascend EP troubleshooting CLI (`asys` supports EP mode only and requires running as the installation user). Every `asys` invocation that runs or inspects a task (`launch`, `collect`, `analyze`, `profiling`) must share the exact same environment variable state across six diagnostic variables; mismatching the environment between faulty task execution and `asys collect` or `asys analyze` causes missing graph/operator dumps or silent symbol resolution failures.

## Mandatory Six-Variable Environment Contract

Before invoking `asys launch` or running a workload followed by `asys collect`, align these six environment variables in the same shell session:

| Variable | Diagnostic Setting | Purpose & Side Effect |
| --- | --- | --- |
| `ASCEND_WORK_PATH` | Absolute writable path | Redirects temporary graph/op dump and log directories away from `$HOME` |
| `ASCEND_SLOG_PRINT_TO_STDOUT` | `0` (default) or `1` | Streams CANN `slog`/`plog` to stdout for immediate correlation |
| `ASCEND_GLOBAL_LOG_LEVEL` | `0` (`DEBUG`), `1` (`INFO`), `3` (`ERROR`) | Controls verbosity; use `1` during active repro and restore to `3` in production |
| `ASCEND_GLOBAL_EVENT_ENABLE` | `0` or `1` | Enables event-level plog entries when tracing runtime queue stalls |
| `ASCEND_PROCESS_LOG_PATH` | `<dir>/plog` | Customizes process log destination (`asys collect` must pass `-r` or share this var) |
| `DUMP_GE_GRAPH` | `2` (or `1`/`3`) | Dumps GE IR graphs required by `asys collect` and `asys analyze` |

## Subcommand Routing Matrix & Output Directory Prefixes

1. **Task Wrapping vs Post-Mortem Bundling (`launch` vs `collect`)**:
   - **Controlled Repro (`asys launch --task="<cmd>" [--output=<dir>] [--tar=T]`)**: Automatically exports diagnostic environment variables (`ASCEND_GLOBAL_LOG_LEVEL=1`, GE graph dump, and CANN crash collection), executes `<cmd>`, and bundles logs, stackcore, and device status into `asys_output_<timestamp>/`.
   - **Post-Mortem Collection (`asys collect [--task_dir=<dir>] [--output=<dir>] [-r=<plog_dir>] [--remote=<id> --all | --quiet]`)**: Gathers existing host/device logs, `dfx_data`, graph dumps, and `npu_collect_intermediates` without re-running the workload.
2. **Offline & Post-Mortem Artifact Analysis (`asys analyze`)**:
   - `-r=trace`: Parses binary `.bin` trace files into human-readable timelines.
   - `-r=coredump`: Resolves host process coredump symbol tables (requires `--exe_file` and `--core_file`).
   - `-r=stackcore`: Resolves `stackcore` text files against debug symbol paths (`--symbol_path`).
   - `-r=coretrace`: Parses `coretrace` files captured on host crash.
   - `-r=ub`: Inspects Unified Buffer (UB) memory snapshots on supported hardware.
   - `-r=aicore_error`: Invokes `msaicerr` parsing on collected `dfx_data` / `plog` directories (`--path=<collect_dir>`).
3. **Live Inspection & Hardware Detection (`info`, `health`, `diagnose`, `config`, `profiling`)**:
   - **Read-Only Live Queries**: `asys info -r=status|software|hardware [-d=<dev_id>]` and `asys health [-d=<dev_id>]` execute safe online queries and **never create an `asys_output_*` directory**.
   - **Exclusive Hardware Diagnostics (`asys diagnose -r=stress_detect|hbm_detect|cpu_detect|aicore_stl_detect|component`)**: Requires root privileges and **zero active NPU processes** on the target device (`-d=<id>`). Outputs to `asys_output_<timestamp>/`.
   - **Configuration Management (`asys config --get|--restore --stress_detect [-d=<id>]`)**: Reads or restores AIC/bus voltage/frequency configurations modified during stress testing.
   - **Integrated Profiling (`asys profiling -r=aicore,dvpp,os,hccl -p=<sec> [-d=<id>]`)**: Collects bounded duration (`1..512` seconds) device/host profiling into `asys_output_<timestamp>/`.

## Verification Checklist

- Confirm `asys health` and `asys info -r=status` are used first during live triage before running intrusive `asys diagnose`.
- Verify `--tar=T` (or `--tar=True`) is appended only when archiving `asys_output_<timestamp>` for cross-host transfer.
