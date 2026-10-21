---
schema: mindie-graded-unit/1
id: unit-msprobe-cross-rank-nan-source-and-construct-chain-triage-workflow
revision: 1
title: "Ascend MSProbe Distributed NaN/Inf Source-Rank Isolation & Construct-Chain Root-Cause Triage Workflow"
tier: L3
category: workflow
evidence_strength: software_contract
status: active
domain_path: validation/msprobe-accuracy/nan-overflow-triage/cross-rank-source-and-construct
inherits_from:
  - unit-multi-dtype-tolerance-mere-mare-and-bf16-comparator-axioms
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: validation
triggers:
  error_signatures:
    - loss is nan
    - gnorm is nan
    - Distributed.all_reduce
  keywords:
    - msprobe
    - nan-overflow-detection
    - dump-json
    - construct-json
    - stack-json
    - source-rank
    - propagation-rank
provenance:
  source_type: source_contract
  source_locator: Ascend/msagent@521f8ead0ebda5db5b282f0c3c4147006e01e016:skills/accuracy/nan-overflow-detection/SKILL.md
  evidence_refs:
    - https://gitcode.com/Ascend/msagent/blob/521f8ead0ebda5db5b282f0c3c4147006e01e016/skills/accuracy/nan-overflow-detection/SKILL.md#L1-L143
  supersedes: []
---

## Preconditions & Trigger

Use this workflow when distributed training or inference on Ascend NPUs exhibits `NaN` or `Inf` in loss, gradients (`gnorm`), or output logits, and per-rank MSProbe statistics dumps (`rank*/{dump.json, construct.json, stack.json}`) have been captured.

## Step-by-Step Distributed NaN/Inf Root-Cause Workflow

### Step 1: Isolate True Source Ranks vs. Collective Propagation Ranks (`cross_rank_analyzer.py`)
- In distributed workloads, a single rank's numerical overflow quickly contaminates all peers via `Distributed.all_reduce_*`, `all_gather_*`, or `reduce_scatter_*`. Inspecting an arbitrary rank that reports `NaN` loss will falsely point to the communication operator itself.
- Scan `rank*/dump.json` across all ranks (`python3 skills/accuracy/nan-overflow-detection/scripts/cross_rank_analyzer.py <input_path>`) and classify each rank at every `Distributed.*` collective boundary:
  1. **Source Rank (`源卡`)**: The collective operator's **input** (`input` / `input_args`) already contains `"Max": "nan"` / `"Min": "nan"` / `"inf"` / `"-inf"`.
  2. **Propagation Rank (`传播卡`)**: The collective operator's **input** is finite, but its **output** contains `NaN`/`Inf` (contaminated by a peer Source Rank during reduction).
  3. **Normal Rank (`正常卡`)**: Both input and output of the collective are finite.
- Restrict all subsequent single-rank operator tracing strictly to the identified **Source Rank(s)**.

### Step 2: Filter False-Positive Non-Compute & Legitimate `-inf` Masking Operators
- **Execution Order Invariant**: The key iteration order inside `dump.json["data"]` is the exact chronological execution order; never re-sort operators alphabetically or by numeric suffix.
- While scanning `dump.json["data"]` on the Source Rank from first to last, **exclude** two classes of benign `NaN`/`Inf` occurrences:
  1. **Uninitialized Memory Allocation & View Ops**:
     - `torch.empty`, `torch.full`, `torch.zeros`, `torch.ones`, `NPU.*_empty`, `NPU.*_full` (uninitialized HBM pages before kernel write frequently contain arbitrary `NaN`/`Inf` bit patterns).
     - Pure view/cast/copy pass-throughs (`Tensor.to`, `Tensor.clone`, `Tensor.detach`) whose inputs were already anomalous.
  2. **Legitimate Causal / Routing Masking (`-inf`)**:
     - `Tensor.masked_fill`, `Torch.masked_fill`, `Tensor.where`, `Torch.where`, `Tensor.triu`, `Torch.triu`, `Tensor.tril`, `Torch.tril` legitimately write `-inf` to mask future causal attention positions or dropped MoE expert logits. Never flag `-inf` outputs from these masking operators as numerical overflow.

### Step 3: Pinpoint the First Compute Anomaly & Reconstruct Module/Source Lineage
- Locate the **first compute operator** in `dump.json["data"]` whose inputs are finite (or legitimately `-inf` masked) and whose `output` introduces `"nan"` or `"inf"`.
- Join the operator key (e.g., `"Tensor.matmul.42.forward"`) against:
  1. **`construct.json`**: Walk the parent pointers upward (`op -> parent_module -> ...`) to reconstruct the full `nn.Module` hierarchy (`construct_chain`).
  2. **`stack.json`**: Extract the exact Python function name, file path, and line number (`stack_info`).

## Verification

- Confirm the isolated root-cause operator on the Source Rank has finite input tensors (`Max`/`Min` within expected dynamic range) and `NaN`/`Inf` output, and verify casting its accumulator to `fp32` or clamping its denominator eliminates the cross-rank `NaN` cascade.
