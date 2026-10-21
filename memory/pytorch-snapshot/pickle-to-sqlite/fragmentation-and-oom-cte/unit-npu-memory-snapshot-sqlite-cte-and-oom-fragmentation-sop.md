---
schema: mindie-graded-unit/1
id: unit-npu-memory-snapshot-sqlite-cte-and-oom-fragmentation-sop
revision: 1
title: "Ascend PyTorch Memory Snapshot (_dump_snapshot) SQLite Conversion, Fragmentation & OOM Triage SOP"
tier: L2
category: sop
evidence_strength: software_contract
status: active
domain_path: memory/pytorch-snapshot/pickle-to-sqlite/fragmentation-and-oom-cte
inherits_from:
  - unit-ascend-msprof-aicore-pipe-and-host-wall-clock-axioms
conditions:
  hardware:
    - Ascend 910B
    - Ascend 910C
  domain: memory
triggers:
  error_signatures:
    - NPU out of memory
    - _dump_snapshot
    - active_allocated
    - segment_alloc
  keywords:
    - memory-snapshot
    - dump-snapshot
    - record-memory-history
    - snapshot-to-db
    - fragmentation
    - expandable-segments
    - oom-triage
provenance:
  source_type: source_contract
  source_locator: Ascend/msagent@521f8ead0ebda5db5b282f0c3c4147006e01e016:skills/profiler/ascend-npu-snapshot-analyzer/SKILL.md
  evidence_refs:
    - https://gitcode.com/Ascend/msagent/blob/521f8ead0ebda5db5b282f0c3c4147006e01e016/skills/profiler/ascend-npu-snapshot-analyzer/SKILL.md#L1-L194
  supersedes: []
---

## Preconditions & Trigger

Apply this SOP when analyzing offline PyTorch NPU memory snapshot pickle files exported via `torch_npu.npu.memory._record_memory_history()` and `torch_npu.npu.memory._dump_snapshot("snapshot.pkl")` to diagnose HBM fragmentation, monotonic memory leaks, peak memory spikes, or Out-Of-Memory (`OOM`) crashes.

## Step 1: Convert Pickle Snapshot to Indexed SQLite Database (`snapshot_to_db.py`)

1. **Offline Pickle-to-SQLite Conversion**:
   - Raw `_dump_snapshot()` pickle files (`dict` or `list` schema containing `segments`, `blocks`, `device_traces`, and nested Python `frames`) are too large for direct in-memory inspection. Convert the pickle file into a normalized SQLite database (`devices`, `segments`, `blocks`, `traces`, `call_stacks`):
     ```bash
     python skills/profiler/ascend-npu-snapshot-analyzer/scripts/snapshot_to_db.py snapshot.pkl
     ```
   - For large snapshots (`> 1 GB`), pass `--no-indexes` during bulk ingestion and build SQLite indexes after import completes.

## Step 2: Canonical SQLite CTE Queries for Fragmentation & Top Allocations

1. **Device Reserve vs. Active Fragmentation Ratio (`device_overview` CTE)**:
   ```sql
   WITH device_overview AS (
       SELECT
           d.device_index,
           SUM(s.total_size)     AS reserved_bytes,
           SUM(s.allocated_size) AS allocated_bytes,
           SUM(s.active_size)    AS active_bytes,
           ROUND((SUM(s.total_size) - SUM(s.allocated_size)) * 100.0 / NULLIF(SUM(s.total_size), 0), 2) AS frag_pct,
           COUNT(s.id)           AS segment_count,
           SUM(s.is_expandable)  AS expandable_segments
       FROM segments s
       JOIN devices d ON s.device_id = d.id
       GROUP BY d.device_index
   )
   SELECT * FROM device_overview;
   ```
   - **Fragmentation Interpretation**: When `frag_pct > 20%` and `expandable_segments == 0`, large cached segments hold unallocated holes that cannot satisfy a new contiguous allocation request, causing false OOMs. Enable `PYTORCH_NPU_ALLOC_CONF="expandable_segments:True"` to allow virtual-memory page stitching across non-contiguous physical segments.
2. **Block State Distribution & Top Active Call Stacks (`top_allocations` CTE)**:
   ```sql
   WITH top_allocations AS (
       SELECT
           d.device_index,
           b.size,
           b.requested_size,
           b.state,
           cs.frames_json
       FROM blocks b
       JOIN segments s ON b.segment_id = s.id
       JOIN devices d ON s.device_id = d.id
       LEFT JOIN call_stacks cs ON b.stack_id = cs.id
       WHERE b.state = 'active_allocated'
       ORDER BY b.size DESC
       LIMIT 20
   )
   SELECT * FROM top_allocations;
   ```
   - Compare `b.size` vs. `b.requested_size` to detect rounding waste, and inspect `b.state IN ('active_pending_free', 'inactive')` via `block_state_dist` to identify blocks held alive by uncompleted asynchronous HCCL/compute streams.

## Step 3: Automated Six-Mode Deep Analysis (`snapshot_analyze.py`)

Run `snapshot_analyze.py` according to the symptom:
- `--mode overview`: Health summary card and per-device segment breakdown.
- `--mode peak`: Temporal trace replay (`segment_alloc`, `segment_map`, `segment_free`, `segment_unmap`) to pinpoint the exact stack frame at peak HBM utilization.
- `--mode fragment`: True vs. false fragmentation across small/large pools.
- `--mode leak`: Monotonically growing allocations and long-lived unreleased blocks with Python stack attribution.
- `--mode oom`: Backtraces the final failing allocation request and active consumers at the moment of OOM.
- `--mode compare --ref baseline.db`: Cross-snapshot `ATTACH DATABASE` diff between two versions or batch sizes.

## Verification

- Generate the self-contained diagnostic report via `python scripts/snapshot_analyze.py snapshot.db --mode all -o report.html` and verify `frag_pct` drops after enabling `expandable_segments:True`.
