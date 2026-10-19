---
category: measurement
conditions:
  device_name: Ascend910
  machine_identity_sha256: c09dc2994a84315148b486a9b4f2e34b656d0112fe5c9d9e663bd57fb2922bab
  npu_smi_version: 26.0.rc1
  topology_sha256: 69790a51d503a1816fb4145c0516d46168123b646d9da24063edc4f0099fa638
content_digest: sha256:0d2597e6da58b50f8ad69425dd708d6ae40fecd9c7729c8ef4a6b429a79790ea
domain_path: hardware/topology
evidence_strength: reproduced_empirical
id: unit-observed-dual-die-phy-id-topology
inherits_from: []
provenance:
  evidence_refs:
  - docs/evidence/l1-topology-observations.json
  source_digest: 62e41437b0f444780eff9b2b8870e2e8890818c83c2f7378d24c17912e3b418a
  source_locator: docs/evidence/l1-topology-observations.json
  source_type: run_manifest
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
tier: L1
title: Observed NPU, Chip and Phy-ID mapping on one dual-die machine
triggers:
  error_signatures: []
  keywords:
  - observed
  - npu
  - chip
  - and
  - phy-id
  - mapping
  - 'on'
  - one
  - dual-die
  - machine
---
Two read-only npu-smi 26.0.rc1 captures on the same anonymized machine, about 72 seconds apart, show the same mapping. The raw device name is Ascend910. NPU IDs 0 through 7 each occur in two table sections: one for Chip 0 and one for Chip 1. Their Phy-ID values span 0 through 15. For NPU 0 the pairs are Chip 0 / Phy-ID 0 / Bus-Id 0000:9D:00.0 and Chip 1 / Phy-ID 1 / Bus-Id 0000:9F:00.0; for NPU 7 they are Chip 0 / Phy-ID 14 / Bus-Id 0000:81:00.0 and Chip 1 / Phy-ID 15 / Bus-Id 0000:83:00.0. A board NPU ID, a within-board Chip ID and a global Phy-ID therefore identify different levels in this observed table. Resolve a requested (NPU ID, Chip ID) through the captured mapping before passing an identifier to an API that requires Phy-ID. The exact mapping is applicable only to the recorded machine, topology digest, device-name string and npu-smi version. A different or missing binding requires a fresh inventory. Both captures reported no running NPU processes; this fact does not demonstrate allocation anti-preemption, cross-host portability or performance. The raw name does not establish a 910C model designation.
