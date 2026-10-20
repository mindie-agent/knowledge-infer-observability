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
  - msnpureport
  - ascend-dmi
  workload:
  - Device-side slog, stackcore, and blackbox firmware crash export and AIC/AIV core-mask
    isolation
created_at: '2026-10-21'
domain_path: diagnostics/cann-toolchain/asys-and-msnpureport/blackbox-and-slog
estimate_tokens: 560
evidence_strength: software_contract
id: unit-msnpureport-permanent-blackbox-and-slog-coremask-sop
inherits_from:
- unit-ascend-plog-root-cause-and-asys-msaicerr-collection-sop
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/cann/msnpureport-log-export/SKILL.md#L1-L260
  promotion_status: verified
  source_locator: cann/cannbot-skills@f305e200aa4b1e1fed73cd21ff1f083b8df25a1e:cann/msnpureport-log-export/SKILL.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Governs msnpureport one-shot vs permanent blackbox/slog export, logical vs
  physical -d device ID mapping across flags, SMP concurrency limits, and temporary
  debug switches.
tier: L2
title: msnpureport One-Shot vs Permanent Blackbox Export, Logical/Physical ID Semantics,
  and Debug Flags SOP
triggers:
  error_signatures:
  - msnpureport --permanent
  - msnpureport --accelerator_recover
  - msnpureport --singlecommit
  keywords:
  - msnpureport
  - --aic_switch
  - --aiv_switch
  - --icachecheck
updated_at: '2026-10-21'
---
# msnpureport One-Shot vs Permanent Blackbox Export, Logical/Physical ID Semantics, and Debug Flags SOP

## Core Principle

`msnpureport` exports device-side `slog`, `hislog`, `stackcore`, and firmware `bbox` (blackbox) memory rings to the host filesystem (`<CANN_HOME>/driver/tools/msnpureport` or `<CANN_HOME>/latest/toolkit/tools/ide_daemon/bin/msnpureport`). Because `-d <device_id>` switches between **logical ID** (`0..63`) and **physical ID** depending on which flag it accompanies, passing the wrong ID type targets the wrong NPU or returns invalid device errors.

## Export Modes & SMP Concurrency Constraint

1. **One-Shot Snapshot Export (`msnpureport [-f] [-a] [-t 2 | -t 3]`)**:
   - Creates `<cwd>/YYYY-MM-DD-HH-MM-SS/` containing `slog/` (`dev-os-*`, `device-*`), `hislog/`, `stackcore/`, and `bbox/`.
   - `-f`: Exports runtime logs (`slog`/`stackcore`) only, skipping historical flash logs for rapid triage.
   - `-a`: Includes extra diagnostic artifacts (`imu_Debug.txt`).
   - `-t 2` / `-t 3`: Sets the host-to-device transfer buffer to `2 MB` or `3 MB` (default `1 MB`) when exporting heavy `stackcore` or `bbox` rings.
2. **Pre-Task Continuous Daemon Mode (`msnpureport --permanent`)**:
   - Must be started **before** launching a workload suspected of triggering a kernel panic, heartbeat loss, or OS reset.
   - Continuously streams device `slog` and `bbox` events to disk so the final pre-crash ring buffer survives a PCIe link drop.
   - **SMP Shared-OS Constraint**: Under Multi-Chip Shared OS (SMP) mode, `msnpureport --permanent` allows **at most 1 concurrent daemon process** across the host; launching a second daemon fails immediately.

## Logical vs Physical `-d` Device ID Semantics

Never assume `-d` has uniform semantics across `msnpureport` options:

| Option Pattern | `-d` Semantics | Valid Range | Purpose |
| --- | --- | --- | --- |
| `-e <module>:<level> -d <id>` | **Logical ID** | `0..63` | Sets device module log level (`debug`/`info`/`warning`/`error`/`null`) |
| `-g <level> -d <id>` | **Logical ID** | `0..63` | Sets global device log level |
| `-p <enable\|disable> -d <id>` | **Logical ID** | `0..63` | Toggles event log printing on logical device `<id>` |
| `--aic_switch=<hex_mask> -d <id>` | **Physical ID** | Physical NPU ID | Masks active AI Core Cube units (bit `1` = enabled, `0` = disabled) |
| `--aiv_switch=<hex_mask> -d <id>` | **Physical ID** | Physical NPU ID | Masks active AI Vector units (`0x0` queries current mask; `0xFFFFFFFFFFFF` restores all) |
| `--icachecheck=<0\|1> -d <id>` | **Physical ID** | Physical NPU ID | Enables (`1`) or disables (`0`) AI Core instruction cache parity checking |

## Temporary Debug Switches & Mandatory Restoration

The following host-wide switches alter runtime execution semantics and **do not persist across reboots**, but **must be restored immediately** after fault reproduction:

1. **Disable Accelerator Error Recovery (`msnpureport --accelerator_recover 0`)**: Prevents the driver from resetting the AI Core upon exception so register state remains frozen for `msaicerr` inspection. Restore with `msnpureport --accelerator_recover 1`.
2. **Single-Operator Synchronous Commit (`msnpureport --singlecommit 1`)**: Forces serial task submission to isolate which operator in an asynchronous stream triggers an AI Core fault. Restore with `msnpureport --singlecommit 0`.
3. **Physical Core Bisection (`--aic_switch` / `--aiv_switch`)**: When isolating a suspected faulty physical core on a specific NPU, query physical ID via `npu-smi info`, bisect the bitmask with `--aic_switch=<mask> -d <phy_id>`, and restore `--aic_switch=0xFFFFFFFFFFFF -d <phy_id>` after testing.
