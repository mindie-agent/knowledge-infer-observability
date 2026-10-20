---
schema: mindie-graded-unit/1
id: unit-ascend-npu-objdump-and-aicore-exception-bitmask-triage-sop
revision: 1
tier: L2
category: sop
domain_path: diagnostics/aicore-dump
evidence_strength: software_contract
title: Five-step PC and aic_error bitmask triage for Ascend AI Core exception dumps
conditions:
  binary_forms:
  - standalone device ELF
  - fused host ELF with .aicore_binary
  - .o and .json kernel artifacts
  - supported .ascend.kernel packages
  disassembler: npu-objdump or msobjdump plus a CANN llvm-objdump with the Ascend backend
  pc_fields:
  - "pc start"
  - "current"
  - "*_ERR_INFO"
  core_types:
  - AIC
  - AIV
  - MIX_AIC
  - MIX_AIV
  first_error_policy: first exception tuple in log time order
content_digest: sha256:f297932543b63230f57a8c23a5c7652eb3887e45da634d059dcffb95aa36c6f6
inherits_from:
- unit-ascend-plog-root-cause-and-asys-msaicerr-collection-sop
provenance:
  source_type: ecosystem_repo
  source_locator: docs/evidence/cannbot-observability-provenance.json
  evidence_refs:
  - docs/evidence/cannbot-observability-provenance.json#source_records.src-objdump-skill
  - docs/evidence/cannbot-observability-provenance.json#source_records.src-objdump-fault-pc
  - docs/evidence/cannbot-observability-provenance.json#aic_error_families
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:tools/npu-objdump-toolkit/SKILL.md#L20-L62
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:tools/npu-objdump-toolkit/references/fault-pc-disassembly.md#L5-L154
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:tools/tool-exception-dump/references/error-line-location.md#L94-L128
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:tools/tool-exception-dump/references/error-type-decode.md#L9-L169
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:ops/ascendc-runtime-debug/references/error_codes.md#L19-L27
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:ops/ascendc-runtime-debug/references/debug_workflow.md#L161-L275
  source_digest: 814abc1f4a1141018c23b53388f21ee219ba15a1be1b87c1543f491685c1b68d
  supersedes: []
triggers:
  error_signatures:
  - "there is an aicore error exception"
  - "there is an aivec error exception"
  - "aic error mask"
  - "CURRENT_PC"
  - "pc start"
  - "trap_or_timeout"
  - "507035"
  - "507015"
  keywords:
  - npu-objdump
  - msobjdump
  - bisheng-objdump
  - llvm-objdump
  - disassembly
  - aic_error
  - ERR_INFO
  - MTE2
  - UB
  - misaligned
  - NaN
  - Inf
  - queue deadlock
  - PC offset
  - AI Core Error
  - exception dump
---

Use this SOP when a plog or `info.txt` reports an AI Core or AI Vector exception, an `aic error mask`, an `error code`, or a `pc start` / `current` pair. The output is a binary-evidence mapping from the first exception PC to a specific Ascend instruction and, when available, a cce or TBE source line. The SOP does not by itself prove the root cause: registers, dump data, address ranges and a reproduction are separate evidence.

## Step 1: Freeze one exception tuple

Read the first effective error, not the last watchdog line. From the plog and `info.txt`, record:

- timestamp, PID, core ID, task ID and stream ID;
- kernel name and `Task Type` (`AIC`, `AIV`, `MIX_AIC` or `MIX_AIV`);
- `error code`, `aic error mask` and the complete `The extend info: errcode` triple;
- `pc start`, `current`, and every available `*_ERR_INFO` (`ifu`, `ccu`, `biu`, `cube`, `mte`, `vec`);
- `para base`, block/tiling information, and the `info.txt` Original versus Corrected block.

`current` (often rendered `CURRENT_PC`) is the pipeline-observed PC and may already be past the faulting instruction. A `Corrected Info` block or an adump `fixedCurrentPC` / `fixedPCOffset` is preferred when present. Do not mix a corrected PC with an uncorrected start PC, and do not carry a PC from one core into another core's record. If the error code is bare `0`, check the extended triple; zero is not proof that no error occurred.

## Step 2: Resolve the exact object and its identity

Prefer the failing kernel's `.o` and `.json` from `dfx/ops`. If they are absent, search the OPP package for the kernel name and SoC. Use `npu-objdump` or the compatible `msobjdump` for metadata and container handling:

```bash
OBJDUMP_CMD="$(command -v npu-objdump 2>/dev/null || command -v msobjdump 2>/dev/null)"
"$OBJDUMP_CMD" --dump-elf exact-kernel.o --verbose > elf.txt
"$OBJDUMP_CMD" --list-elf exact-kernel.o
"$OBJDUMP_CMD" --extract-elf exact-kernel.o --out-dir ./extracted
```

`input_file` may be a standalone device ELF, a fused host ELF containing `.aicore_binary`, a static library, or a supported `.ascend.kernel.*` package. The tool detects format from content, not from the extension. `--dump-elf` shows metadata; `--list-elf` lists embedded device ELF; `--extract-elf` writes files; `--sass` disassembles Ascend AICore instructions only when the installed interface supports it.

For MIX kernels, choose the physical symbol that matches the exception core type: `_mix_aic` for Cube or `_mix_aiv` for Vector. Do not disassemble the wrong side and then infer a fault from an unrelated instruction. Record the object SHA256 and compare it with the runtime object or report hash. If the hash does not match, stop mapping; a filename match is not identity proof.

`npu-objdump` is the metadata and ELF front end. The instruction backend is a CANN `llvm-objdump` with the Ascend target, not a host GNU `objdump`. If the site exposes `bisheng-objdump`, verify its version and interface first; do not assume that the name implies the required Ascend backend. `--sass` is also not guaranteed on every CANN installation.

The `.json` file is metadata, not executable code. Use it for kernel name, TilingKey and compilation information. The `_loc.json` companion, when present, maps a cce line to a TBE source file and line.

## Step 3: Normalize the PC in one address domain

The simplest offset is the direct byte difference:

```text
offset_bytes = current_pc - pc_start
```

Only do this when both values are in the same byte-address domain. If `current_pc < pc_start`, do not silently take an absolute value; apply the target platform's wrap or truncation rule, or record that the domain is unresolved.

When the exception register is the better source, decode the selected `ERR_INFO` by unit:

| Unit | Instruction-word address | Byte PC |
|---|---|---|
| CCU | `ERR_INFO[30:23] << 8 \| ERR_INFO[7:0]` | word address × 4 |
| MTE | `ERR_INFO[39:32] << 8 \| ERR_INFO[7:0]` | word address × 4 |
| VEC | `ERR_INFO[39:32] << 8 \| ERR_INFO[7:0]` | word address × 4 |
| CUBE | `ERR_INFO[31:24] << 8 \| ERR_INFO[7:0]` | word address × 4 |
| IFU | the complete `IFU_ERR_INFO` is already the PC | use as-is |

These formulas are unit-specific; do not apply one unit's bit field to another. A packed IFU register or a platform-corrected PC may need a target-specific extraction step. Compare the decoded result with the direct difference only after subtracting the same base:

```text
decoded_offset = decoded_pc - pc_base
direct_offset  = current_pc - start_pc
equal          = decoded_offset == direct_offset
```

If `decoded_pc` is already an object-relative offset, a `pc_base` of zero can be used explicitly; otherwise a missing base is an evidence gap, not a zero. Template kernels may need the `<kernel_name>_<tiling_key>$local` symbol base added before the offset is used.

## Step 4: Interpret the bitmask by family, not by a single number

The error code can be a hexadecimal mask on Stars/910B or a comma-separated decimal bit list on David/950. Decide the dialect first, and prefer the register-key method when the two disagree: `sc error info`, `su error info` and `l1 error info` indicate David; `ifu error info`, `ccu error info` and `biu error info` indicate Stars. A bare `0` is ambiguous and is not a bit.

The following table gives the common families. Bit numbers are from the pinned producer dictionaries; they are not a cross-generation universal enum.

| Suspected family | Candidate bits | Required corroboration |
|---|---|---|
| MTE2 invalid GM address or cross-device timeout (the usual "GM out-of-bounds read" family) | Stars 23 `mte_biu_rdwr_resp`; David 95 `MTE_BIU_RDWR_RESP`; Stars 0 `biu_l2_read_oob` | The access address, transfer length, allocated GM range, the LD/ST instruction and the dump tensor bounds. A bit alone does not identify the buffer. |
| Adjacent on-chip MTE overflow, often confused with a GM fault | Stars 32 `mte_gdma_read_overflow`; David 81 `MTE_GDMA_READ_OVERFLOW`; Stars 33 `mte_gdma_write_overflow`; David 82 `MTE_GDMA_WRITE_OVERFLOW` | Whether the address is in UB/L1 rather than GM; the pipe direction and the tile/block size. |
| UB unaligned or UB overflow/wrap | David 79 `MTE_ATM_ADD_ADDR_MISALIGN` (atomic only); Stars 62 `vec_ub_wrap_around`; David 341 `VEC_UB_WRAP_AROUND`; Stars 124 `mte_ub_wr_ovflw`; Stars 125 `mte_ub_rd_ovflw`; David 174 `L1_UB_WR_OVFLW` | UB-side start address modulo 32 bytes, VEC operand start, `DataCopy`/`DataCopyPad` `blockLen` in bytes, `padParams` left/right padding, dynamic chunk alignment and total UB budget. There is no single universal "UB unaligned" bit; `507035` is the usual runtime surface for that class. |
| Vector or Cube input is NaN/Inf | Stars 156 `vec_idata_inf_nan`; Stars 9 `cube_invld_input`; David 336 `VEC_ERR_IDATA_INF_NAN_T0`; David 4 `CUBE_INVLD_INPUT` | The dump's NaN/INF check, the input value range and the upstream producer. The bit names the unit, not the bad tensor. |
| Queue deadlock or no-progress | No dedicated bit | No-progress plus `507014` aicore timeout, `fftsplus task execute failed`, or `trap_or_timeout` when no bit is set. Check `set_flag`/`wait_flag` pairing, stream/task dependencies, collective ordering, and whether an earlier error is the real first cause. |

`trap_or_timeout` is the fallback printed when no position is set. It is not bit 0 and it is not a confirmed queue deadlock. Conversely, a bitmask of `0x0` does not mean "no error": rebuild the code from the `The extend info: errcode` triple when possible; if that also fails, keep the uncertainty visible.

For a `507035` vector-core exception, use the runtime checklist rather than inventing a bit: UB-side `DataCopy`/`DataCopyPad` start address and sub-tensor offset must be 32-byte aligned; VEC operand start addresses must be aligned; `blockLen` is in bytes and must be an integer multiple of `sizeof(T)` (it has no 32-byte alignment requirement); `padParams` left and right padding must not exceed 32 bytes; and total UB use must stay within the buffer budget. Distinguish 507035 from 507046 stream-sync timeout.

## Step 5: Confirm the instruction and write the evidence gap

Disassemble a narrow window around the normalized object PC, not the whole object by default:

```bash
PC=0x1234   # replace with the normalized object PC
llvm-objdump -d --start-address="$PC" --stop-address="$((PC + 0x100))" exact-kernel.o > pc-window.sass
# or, only when the installed interface supports it:
"$OBJDUMP_CMD" --sass exact-kernel.o > kernel.sass
```

The target address must land on a 4-byte-aligned instruction inside the selected symbol. If it does not, return to Step 2 and check the object identity, core type, symbol base and address domain. A nearest-instruction guess is not a mapping.

If line information is available, use `cce-objdump -d --line-numbers` or the CANN `llvm-objdump --line-numbers` form, find the reported offset, walk back to the nearest `cce:<line>` annotation, and then use `<kernel>_loc.json` to resolve `cce_line2loc` to a TBE source file and line. L0 records may stop at the assembly offset; that is a normal degradation, not a reason to invent a source line.

Write the final record with:

- object path and SHA256, plus the metadata and instruction-disassembler paths;
- runtime kernel name, SoC, core type and TilingKey (or "unknown");
- raw `pc start`, `current`, `ERR_INFO` and `aic error mask`;
- the exact bit-field formula, base and normalized object PC;
- physical symbol, symbol offset and a window containing the target instruction;
- every candidate bit name, the corroborating register/dump/range evidence, and explicit counter-evidence.

Stop and record an evidence gap when the object hash does not match, the target PC has no instruction, the core type is unresolved, or the bitmask cannot be tied to a unit. Disassembly proves only the contents of the selected object; it does not prove runtime load identity, the exact bad buffer or the root cause. A suspicious load/store becomes a root-cause claim only when the register values, access length and allocation range agree.

Adoption boundary: this SOP is pinned to the evidence packet and the producer's Stars/David dictionaries. Chip dialect, bit numbering, `ERR_INFO` layout and the availability of `--sass` or `bisheng-objdump` vary by CANN and SoC. Re-check the target environment before using a bit name in a customer report.
