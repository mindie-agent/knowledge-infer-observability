---
category: workflow
conditions:
  hardware:
  - Ascend 910B (A2)
  - Ascend 910C (A3)
  - Ascend 950 (A5)
  os:
  - Linux
  software_stack:
  - CANN 8.0+
  - npu-objdump
  - msobjdump
  - cce-objdump
  - llvm-objdump
  workload:
  - Disassembling fatbin/kernel_meta .o binaries and mapping AI Core fault PCs to
    source lines
created_at: '2026-10-21'
domain_path: diagnostics/cann-toolchain/msaicerr-and-objdump/pc-offset-disassembly
estimate_tokens: 640
evidence_strength: software_contract
id: unit-npu-objdump-mix-symbol-and-err-info-pc-disassembly-workflow
inherits_from:
- unit-ascend-npu-objdump-and-aicore-exception-bitmask-triage-sop
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/cann/npu-objdump/SKILL.md#L1-L160
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/cann/msaicerr-dump-analysis/SKILL.md#L150-L310
  promotion_status: verified
  source_locator: cann/cannbot-skills@f305e200aa4b1e1fed73cd21ff1f083b8df25a1e:cann/npu-objdump/SKILL.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Executes the end-to-end object extraction, SHA256 verification, _mix_aic/_mix_aiv
  symbol base resolution, ERR_INFO PC byte normalization, and _loc.json source line
  attribution workflow.
tier: L3
title: npu-objdump/cce-objdump Mix-Kernel Symbol Selection, ERR_INFO PC Normalization,
  and Source Mapping Workflow
triggers:
  error_signatures:
  - msobjdump --dump-elf
  - npu-objdump --extract-elf
  - Current PC is not in kernel text range
  keywords:
  - msobjdump --list-elf
  - _mix_aic
  - _mix_aiv
  - _loc.json
updated_at: '2026-10-21'
---
# npu-objdump/cce-objdump Mix-Kernel Symbol Selection, ERR_INFO PC Normalization, and Source Mapping Workflow

## Core Principle

Locating the exact faulting instruction in a compiled Ascend operator requires matching the runtime binary (`check_hash` via SHA256), extracting embedded device ELFs via `npu-objdump` (renamed `msobjdump` in CANN 9.0+), selecting the correct `_mix_aic` vs `_mix_aiv` entry symbol base, and normalizing the subdomain `ERR_INFO` low-16-bit PC offset before consulting the disassembly and `_loc.json` line table.

## 5-Stage PC-to-Source Attribution Workflow

### Stage 1: Verify Object Identity & Extract Device ELF (`npu-objdump` / `msobjdump`)

1. **SHA256 Checksum Gate**: Compare the ` sha256=` field recorded in `plog` against `sha256sum <kernel>.o`. If they differ, the `.o` on disk was recompiled or belongs to another rank; stop and extract the exact `.o` from the `asys collect` bundle.
2. **Fatbin Inspection & Extraction**: When debugging a host `.so` or executable containing device kernels, use `npu-objdump` (`msobjdump` on CANN 9.0+):
   ```bash
   # List embedded Ascend device ELF sections
   npu-objdump --list-elf libcustom_op.so
   # Extract all embedded device ELFs into current directory
   npu-objdump --extract-elf libcustom_op.so
   # Or directly disassemble all embedded device ELFs with source/address annotations
   npu-objdump --dump-elf libcustom_op.so > device_sass.txt
   ```

### Stage 2: Disassemble `.o` and Resolve `_mix_aic` vs `_mix_aiv` Symbol Base

1. Run `cce-objdump -d -l <kernel>.o > <kernel>.cce.dump` (or `llvm-objdump -d -S --line-numbers <kernel>.o`).
2. **Mixed-Kernel Symbol Selection**: In Cube+Vector (`MIX`) kernels, a single `.o` contains multiple entry symbols (`<kernel>_mix_aic` for Cube cores and `<kernel>_mix_aiv` for Vector cores).
   - Inspect the exception core type (`AIC` vs `AIV` / `VEC`) in the `plog` `[AIC_INFO]` header.
   - Select the matching symbol start address (`symbol_base`) from the disassembly table before computing relative offsets.

### Stage 3: Compute `diff_pc` and Normalize Subdomain `ERR_INFO` Offsets

1. **Primary Offset**:
   $$\text{diff\_pc} = \text{current\_pc} - \text{start\_pc} + \text{symbol\_base}$$
2. **Subdomain `ERR_INFO` Low-16-Bit PC Correction**: Because AI Core pipelines execute asynchronously, `CURRENT_PC` frequently points tens of instructions past the actual faulting instruction. Extract `ERR_INFO[15:0]` from the faulting subdomain (`MTE_ERR_INFO`, `VEC_ERR_INFO`, `CUBE_ERR_INFO`, `IFU_ERR_INFO`, `CCU_ERR_INFO`, `BIU_ERR_INFO`):
   - **Instruction-Word vs Byte Offset Trap**: On architectures where `CCU_ERR_INFO` or `MTE_ERR_INFO` stores instruction-word indices (4-byte instructions) or omits high address bits, multiply or mask `ERR_INFO[15:0]` so its low bits align with the 4-byte instruction boundary in `<kernel>.cce.dump`, and search the disassembly window `[ERR_INFO_PC - 0x20, CURRENT_PC]` for instructions belonging to the faulting pipe (`MTE2`, `MTE3`, `V`, `M`, `S`).

### Stage 4: Map Disassembly Address to Source via `_loc.json` and `llvm-symbolizer`

1. Pass the normalized hex offset to `llvm-symbolizer -e <kernel>.o -f -C -i 0x<diff_pc>` to obtain the inlined C++/AscendC call stack (`file:line:column`).
2. For TBE / `.cce` operators accompanied by `<kernel>_loc.json`, map the disassembly `cce_line` key in `_loc.json` back to the Python/DSL operator line stack.

### Stage 5: Audit Hardware Synchronization Around the Faulting PC

Inspect the 16 instructions preceding `ERR_INFO_PC` in `<kernel>.cce.dump`:
- Verify every `set_flag(PIPE_X, PIPE_Y, EVENT_IDn)` has a matching `wait_flag(PIPE_X, PIPE_Y, EVENT_IDn)` before the destination buffer is read or overwritten.
- If `CURRENT_PC` is stalled on `wait_flag` or `bar.pipe_all`, inspect the peer pipe (or peer `MIX_AIC`/`MIX_AIV` core) for an early branch return that skipped the corresponding `set_flag`.
