---
schema: mindie-graded-unit/1
id: unit-ascend-cann-error-code-four-layer-evidence-and-remediation-workflow
revision: 1
tier: L3
category: workflow
domain_path: diagnostics/error-codes
evidence_strength: software_contract
title: CANN operator error-code four-layer evidence verification, innermost first-report rule, and E*9*** remediation workflow
conditions:
  log_family: CANN plog and ErrorManager
  repository: cann/cannbot-skills
  revision: f305e200aa4b1e1fed73cd21ff1f083b8df25a1e
inherits_from:
- unit-ascend-plog-root-cause-and-asys-msaicerr-collection-sop
provenance:
  source_type: ecosystem_repo
  source_locator: docs/evidence/cannbot-exception-compute-and-error-code-provenance.json
  source_digest: cad5302e2dfa4eaeec30182dda55f298fd1e1e60da892ffa76877dbf5ba227b3
  evidence_refs:
  - docs/evidence/cannbot-exception-compute-and-error-code-provenance.json
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/ops/cann-error-code-standard/SKILL.md#L6-L39
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/ops/ops-error-code-remediation/SKILL.md#L6-L106
  supersedes: []
triggers:
  error_signatures:
  - EZ9999
  - E89999
  - EI9999
  - REPORT_PREDEFINED_ERR_MSG
  - REPORT_INNER_ERR_MSG
  keywords:
  - cann-error-code-standard
  - ops-error-code-remediation
  - ErrorManager
  - OP_LOGE
  - documented-error-codes.json
  - audit_operator_error_codes
  - scan_error_code_candidates
---
When designing, auditing, or remediating CANN C/C++ operator error codes (`EZ*`, `EI*`, `EL*`, `E8*`, `EE*`, including generic `E*9***` catch-all codes), enforce the four-layer evidence hierarchy and innermost single-first-report workflow (`ops/cann-error-code-standard/SKILL.md:6-39`, `ops/ops-error-code-remediation/SKILL.md:6-106`).

## 1. Four Non-Substitutable Evidence Layers (`cann-error-code-standard/SKILL.md:18-26`)

Never assume an error code documented in public manuals is registered and callable in a given CANN build. Verify and report which of the four layers have been established:

1. **Layer 1 — Public Documentation Catalog (`documented-error-codes.json`)**: Establishes public error code semantics and user troubleshooting guidance, but does **not** prove target-version source registration.
2. **Layer 2 — Target-Version Source Registry & Headers**: Proves that the error code identifier, `Arglist` parameter signature, and reporting macro exist in the target repository's `opbase` / component error registry (never borrow error codes across unrelated component registries).
3. **Layer 3 — CMake & Include-Chain Resolution**: Resolves `target_include_directories` and macro aliases for the modified translation unit to prove which `log.h` defines `OP_LOGE` / `CHECK_RET` / `TR_EXECUTE` and whether the macro expands to a formal `ErrorManager` leaf (`REPORT_PREDEFINED_ERR_MSG` or `REPORT_INNER_ERR_MSG`) versus a PLOG-only leaf.
4. **Layer 4 — Installed Package & Negative Runtime Test**: Proves that the compiled binary emits the expected formal error code, diagnostic fields (`opName`, `parameter`, `actual`, `expected`, `reason`, `rc`), and public return status on a negative test case without duplicate `ErrorManager` reports.

## 2. Innermost First-Report Rule & Failure-Exit Enumeration (`ops-error-code-remediation/SKILL.md:56-91`)

1. **Enumerate All Observable Failure Exits**:
   - Traverse parameter validation, `InferShape`, `Tiling`, workspace/memory sizing, executor creation, kernel selection, and runtime/driver launch boundaries.
2. **Single `ErrorManager` First-Report Invariant**:
   - Emit the formal `ErrorManager` report (`REPORT_PREDEFINED_ERR_MSG` or `opbase` semantic macro with exact `Arglist` arity) **at the innermost layer that possesses the root-cause facts (`actual`, `expected`, `reason`, `rc`)**.
   - When a lower layer has already emitted a formal first-error report, outer layers must **only append PLOG context and propagate the return code unchanged**—never emit a second `ErrorManager` first-error or overwrite a specific root-cause code with a generic outer `E*9***` wrapper.
   - If neither the documentation catalog nor the target-version registry contains a matching semantic code, file a structured formal error-code proposal rather than fabricating an unregistered code or borrowing across modules.

## 3. Candidate-to-Audit Reconciliation Gate (`ops-error-code-remediation/SKILL.md:44-106`)

1. Run `scan_error_code_candidates.py` and `audit_operator_error_codes.py` (`--root <operator_dir> --cmake-root <repo_root> --cann-root <cann_root>`) before and after remediation.
2. **Closure Gate**: A remediation task cannot be marked closed until candidate-to-audit reconciliation shows zero unaccounted trigger sites, all `log.h` include chains are resolved (using compiler `-E`/`-H` when generated headers or preprocessor conditionals obscure static resolution), and negative UT/ST cases verify both the public return code and single-first-error emission.
