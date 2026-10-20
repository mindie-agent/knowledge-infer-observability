---
category: sop
conditions:
  hardware:
  - Ascend 910B (A2)
  - Ascend 910C (A3)
  - Ascend 950 (A5)
  os:
  - Linux
  software_stack:
  - CANN 8.0+
  - msaicerr
  workload:
  - AI Core exception register decoding across Stars and David chip architectures
created_at: '2026-10-21'
domain_path: diagnostics/cann-toolchain/msaicerr-and-objdump/exception-register-decoding
estimate_tokens: 620
evidence_strength: software_contract
id: unit-msaicerr-stars-david-bitmask-and-six-subdomain-decode-sop
inherits_from:
- unit-ascend-npu-objdump-and-aicore-exception-bitmask-triage-sop
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/cann/msaicerr-dump-analysis/SKILL.md#L1-L320
  promotion_status: verified
  source_locator: cann/cannbot-skills@f305e200aa4b1e1fed73cd21ff1f083b8df25a1e:cann/msaicerr-dump-analysis/SKILL.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Decodes AI Core exception registers across Stars (hex bitmask, 176 bits,
  6 subdomains + 5 MTE sub-dicts) and David (decimal bit-index list across CUBE/MTE/L1/SC/SU/VEC)
  architectures.
tier: L2
title: msaicerr Stars Hex Bitmask vs David Decimal Bit-Index Six-Subdomain Register
  Decoding SOP
triggers:
  error_signatures:
  - aicore_err_extend_info
  - aic_error_info_dict
  - soc_err_info_dict
  keywords:
  - Stars bitmask
  - David bit index
  - fmc_err_info_dict
  - unzip_err_info_dict
updated_at: '2026-10-21'
---
# msaicerr Stars Hex Bitmask vs David Decimal Bit-Index Six-Subdomain Register Decoding SOP

## Core Principle

`msaicerr` supports two fundamentally different AI Core exception encoding formats depending on the NPU microarchitecture: **Stars (`ai_core_arch = stars`)** encodes exceptions as a `0x`-prefixed hexadecimal bitmask (`hex2bin`, reversed so string index equals bit position across `0..175`), whereas **David (`ai_core_arch = dav_m300`)** encodes exceptions as a comma-separated list of **decimal bit indices** (e.g., `65, 194`). Decoding a David decimal list as a Stars hex mask (or vice versa) misattributes MTE/VEC faults to completely unrelated hardware units.

## Architecture Detection & Decoding Procedure

1. **Format Discrimination**:
   - If the `error code` token starts with `0x` (e.g., `0x800000`, `0x100000000000000`), use the **Stars** decoder.
   - If the `error code` token is a comma-separated integer list without `0x` (e.g., `0, 64, 320`), use the **David** decoder.

2. **Stars Architecture Decoding (`176` bits across 6 Subdomains)**:
   - Convert the hex code to a binary bitstring and reverse it (`bit_index = i` where `(code >> i) & 1 == 1`), then look up each set bit in `aic_error_info_dict[i]`:

| Subdomain | Hardware Unit | Auxiliary Status / Sub-Dictionary | Primary Failure Semantics |
| --- | --- | --- | --- |
| `IFU` | Instruction Fetch Unit | `IFU_ERR_INFO` | `icache` parity error, bus read fault (`IFU_READ_ERR`), or PC jump into non-executable GM |
| `CCU` | Scalar Control Unit | `CCU_ERR_INFO` | Loop stack overflow, illegal scalar instruction, or out-of-range event ID (`event_id > 7`) |
| `BIU` | Bus Interface Unit | `BIU_ERR_INFO` | L2/GM bus transaction error or write response error (`0x800000`) |
| `CUBE` | Matrix Cube Unit | `CUBE_ERR_INFO` | L0A/L0B/L0C buffer wraparound or illegal matrix instruction configuration |
| `VEC` | Vector Execution Unit | `VEC_ERR_INFO` (`vec_err_data`, `vec_err_rcnt`) | UB out-of-bounds, FP16/FP32 INF/NAN trap, `sqrt` of negative input, or div-by-zero |
| `MTE` | Memory Transfer Engine | 5 sub-dicts: `soc_err_info_dict`, `fmc_err_info_dict`, `fmd_err_info_dict`, `unzip_err_info_dict`, `aipp_err_info_dict` | GM/UB/L1 read-write fault, `FMC`/`FMD` compression error, `UNZIP` overflow, or `AIPP` config fault |

3. **Special Stars Override Rules**:
   - **`0x800000` + `dha status 1`**: When `error code == 0x800000` (`biu_err`) and the log contains `dha status 1`, override the generic BIU fault with **Return Code `105` (Atomic Accumulation Value Overflow)** (`SetAtomicAdd` produced `INF`/`NAN` in GM).
   - **`0x0` v300 Extend-Info Reconstruction**: When `error code == 0x0`, extract `(err_time, err_pc, err_status)` triples from `aicore_err_extend_info` (`v300`), shift bits (`err_status << 16` or subdomain offset), and re-decode the reconstructed bitmask.

4. **David Architecture Decoding (`dav_m300` Decimal Bit Offsets)**:
   - Parse each integer `b` in the comma-separated list directly into the 64-bit-aligned David subdomain ranges:
     - `0..63`: `CUBE` (`CUBE_ERR_INFO`)
     - `64..127`: `MTE` (`MTE_ERR_INFO`)
     - `128..191`: `L1` (`L1_ERR_INFO` — L1 buffer bank conflict / ECC / out-of-bounds)
     - `192..255`: `SC` (`SC_ERR_INFO` — System/Scalar Control)
     - `256..319`: `SU` (`SU_ERR_INFO` — Scalar Unit)
     - `320..383`: `VEC` (`VEC_ERR_INFO`)
