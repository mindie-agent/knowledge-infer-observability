---
schema: mindie-graded-unit/1
id: unit-ascend-msaicerr-34-step-exception-dump-and-return-code-workflow
revision: 1
tier: L3
category: workflow
domain_path: diagnostics/aicore-dump
evidence_strength: software_contract
title: Closed-loop 34-step AI Core exception dump workflow across Stars/David register decoding, special scenarios, and 101-107 return codes
conditions:
  collection_tool: asys
  dump_analyzer: msaicerr
  product_form: Ascend EP
  repository: cann/cannbot-skills
  revision: f305e200aa4b1e1fed73cd21ff1f083b8df25a1e
inherits_from:
- unit-ascend-plog-root-cause-and-asys-msaicerr-collection-sop
- unit-ascend-npu-objdump-and-aicore-exception-bitmask-triage-sop
provenance:
  source_type: ecosystem_repo
  source_locator: docs/evidence/cannbot-exception-compute-and-error-code-provenance.json
  source_digest: 4ae68ba4f787ebc533cc4d094982459de194695b06facb4621218e7f5a0de045
  evidence_refs:
  - docs/evidence/cannbot-exception-compute-and-error-code-provenance.json
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/tools/tool-exception-dump/SKILL.md#L11-L360
  supersedes: []
triggers:
  error_signatures:
  - "The built-in sample operator runs successfully, The environment is normal."
  - "Do not run msaicerr in the directory specified by -p or its subdirectory"
  - "Failed to get dump data of error op!"
  - "dha status 1"
  - "Begin to dump callback exception"
  - "fftsplus task execute failed"
  keywords:
  - tool-exception-dump
  - msaicerr
  - asys
  - Stars
  - David
  - aicerror
  - llvm-symbolizer
  - cce-objdump
  - return code 101
  - return code 104
  - return code 105
---
When executing end-to-end AI Core / AI Vector exception diagnosis (`msaicerr` + `asys`) on Ascend NPU, follow the 6-phase (`P0`–`P5`) pipeline so preflight gates, chip-generation register decoding (`Stars` vs `David`), and conclusion codes (`101`–`107`) remain deterministic (`tools/tool-exception-dump/SKILL.md:11-360`).

## 1. Phase 0 & Phase 1 — Preflight Gates, Unsupported Fused Ops, and 3-Element Validation

1. **Local Execution & Environment Gates (`P0.1`–`P0.6`)**:
   - `msaicerr` must run on the **same machine and CANN installation** where the fault occurred (`Ascend EP` mode, Python $\ge 3.7.5$, `ASCEND_OPP_PATH` non-empty).
   - Run golden-op hardware self-check first: `python3 msaicerr.py -e -dev <device_id>`. Exit code `0` confirms hardware health; exit code `103` indicates a device/driver fault (`103: hardware error`) and halts `-p` analysis.
   - **Intercept 8 Unsupported Operators (`P0.5`)**: `msaicerr` does not support `MatmulAllReduce` family (`MatmulAllReduce`, `MatmulAllReduceAddRmsNorm`, `MatmulAllReduceInplaceAddRmsNorm`), `AllGatherMatmul`, `MatmulReduceScatter`, `GroupedMatmulAllReduce`, `MemSet`, or `NonMaxSuppressionBucketize`.
   - **Directory Disjointness (`P0.6`)**: Never run `msaicerr.py` inside the `-p` collection directory, and ensure `-out` is outside `-p` (violation exits with code `2`).
2. **Manual `tar.gz` Extraction & 3-Element Check (`P1.1`–`P1.5`)**:
   - If the input is an `*.tar.gz` archive, extract it manually to a directory containing a single `asys_output_*` subdirectory (`msaicerr.py`'s internal `--tar_file` branch is not registered in `main()`).
   - Verify the three collection elements under `<collection_dir>/dfx/`: `data-dump/` (L0 exception dump), `ops/` (`*.o` and `*.json`), and `log/host/cann/` (host plog). Missing `plog` is a hard stop (exit code `8` on `-p`).

## 2. Phase 2 & Phase 3 — Scenario Flags, `Stars` vs `David` Decoding, and Conclusion Codes `101`–`107`

1. **Independent Scenario Flags (`P2.1`)**:
   - `grep -rn "\[AIC_INFO\] dev_func:" <dir>` → hit = `L1` (has `.cce` and TBE `_loc.json` line mapping via `llvm-symbolizer` or `cce-objdump -d --line-numbers`), miss = `L0` (assembly PC offset only).
   - `grep -rn "fftsplus task execute failed" <dir>` → `ffts` mode (check for `D2H failed`).
   - `grep -rn "Begin to dump callback exception" <dir>` → `SK` (SuperKernel fused mode; produces only `host.o` and degrades single-op reproduction—disable operator fusion and rerun if line attribution is ambiguous).
2. **`Stars` (`Ascend 910B`) vs `David` (`Ascend 950`) Register Decoding (`P3.1`–`P3.3`)**:
   - **Method 1 (Error Code Format)**: `0x`-prefixed hexadecimal bitmask → `Stars` (`176`-bit dictionary in `constant.py:155-369` with `IFU/MTE/CCU/BIU/CUBE/VEC` sub-registers); comma-separated decimal bit numbers (e.g., `257, 341`) → `David` (`187`-entry dictionary in `constant.py:378-1123` segmented as `CUBE: 0+`, `MTE: 64+`, `L1: 128+`, `SC: 192+`, `SU: 256+`, `VEC: 320+`).
   - **Method 2 (Dump Register Keys — Overrides Method 1)**: Presence of `ifu/ccu/biu error info:` → `Stars`; presence of `sc/su/l1 error info:` → `David`.
3. **Five Priority Special Scenarios & Return Codes (`P3.4`–`P3.5`)**:
   - **Scenario 1 (Atomic Overflow → `105`/`107`)**: Error code exactly `0x800000` + `dha status 1` + missing `AtomicLaunchKernelWithFlag` memset check → `105` (framework missing memset zeroing).
   - **Scenario 2 (Args Corrupted → `106`)**: Section ④ `args before` $\neq$ `args after` execution → `106` (operator workspace/args stomped by preceding kernel).
   - **Scenario 3 (Outstanding Dual-PC)**: Fallback to `AICORE_ERR_OCCUR_OST`—match PC to the faulting task.
   - **Scenario 4 (`isconcurrentexe`)**: $>2$ deduplicated concurrent tasks invalidate automatic atomic attribution.
   - **Scenario 5 (Bare `0x0`)**: Reconstruct error code from `The extend info: errcode:` triple before falling back to `trap_or_timeout`.
   - **Concurrency Capacity Check**: If `rts_block_dim > 2 * driver_aicore_num`, flag insufficient physical AI Cores for operator concurrency.
   - **Standard Conclusion Codes**: `101` (single-operator bug), `102` (memory allocation / invalid dump address / `D2H failed`), `103` (hardware fault), `104` (NaN/Inf in operator input data), `105` (missing framework memset), `106` (operator args stomped), `107` (atomic accumulation overflow).

## 3. Phase 4 — Dump-Driven (`Path A`) vs Synthetic (`Path B`) Single-Op Reproduction

- When `dfx/data-dump/<device_id>/exception_info.*` exists (`Path A`), run `python3 msaicerr.py -d <dump_path> -out <out_dir> -dtype <dtype>` (where `input_type == 7` in the dump header holds tiling data, and `bfloat16` requires `bfloat16ext`) to inspect inputs for `NaN`/`Inf` (`104`) and replay the single operator in isolation. When exception dump is absent (`L1` without dump or `102` dump failure), construct synthetic inputs (`Path B`) using recovered tiling parameters.
