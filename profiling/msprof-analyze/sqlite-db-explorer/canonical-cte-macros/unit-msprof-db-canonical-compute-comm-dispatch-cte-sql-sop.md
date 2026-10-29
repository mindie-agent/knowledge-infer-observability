---
category: sop
conditions:
  domain: profiling
domain_path: profiling/msprof-analyze/sqlite-db-explorer/canonical-cte-macros
evidence_strength: software_contract
id: unit-msprof-db-canonical-compute-comm-dispatch-cte-sql-sop
inherits_from: []
provenance:
  evidence_refs:
  - https://gitcode.com/Ascend/msagent/blob/521f8ead0ebda5db5b282f0c3c4147006e01e016/skills/profiler/ascend-profiler-db-explorer/SKILL.md#L1-L183
  source_locator: Ascend/msagent@521f8ead0ebda5db5b282f0c3c4147006e01e016:skills/profiler/ascend-profiler-db-explorer/SKILL.md
  source_type: source_contract
  supersedes: []
revision: 2
schema: mindie-graded-unit/1
status: active
tier: L2
title: Ascend PyTorch Profiler & msprof SQLite DB Canonical CTE Macros (compute_view,
  comm_view, dispatch_view) SOP
triggers:
  error_signatures:
  - no such table COMPUTE_TASK_INFO
  - STRING_IDS join missing
  - CONNECTION_IDS mismatch
  keywords:
  - ascend-pytorch-profiler-db
  - msprof-db
  - compute-view
  - comm-view
  - dispatch-view
  - string-ids
  - connection-ids
  - sqlite
---
## Applicability boundary for offline analysis

This is a host-side SQLite parsing procedure. Its prerequisites are the documented tables, columns, join keys and timestamp units in the artifact being inspected; inspect that schema before using a query. A saved capture or offline parser fixture does not require access to an NPU or a known SoC. Neither the SQL sums nor a parser test establish live device utilization, elapsed step time, bandwidth or hardware performance. Do not inherit device-topology or live-collection requirements merely because the artifact was produced by an Ascend profiler.

## Preconditions & Trigger

Apply this SOP when querying `ascend_pytorch_profiler_*.db` or `msprof_*.db` SQLite databases directly to attribute operator execution durations, HCCL collective communication latencies, or PyTorch-to-CANN-to-Device dispatch overheads.

## SQL Query Guardrails & Track Selection

1. **Bounded Result Set Rule**:
   - Every ad-hoc SQL query against an Ascend profiler `.db` file MUST include either an aggregate function (`SUM`, `AVG`, `COUNT`, `MAX`) or an explicit `ORDER BY ... LIMIT 20` clause to prevent multi-million-row `TASK` table dumps from exhausting context memory.
2. **String Interning & Connection Bridge Invariants (`STRING_IDS` & `CONNECTION_IDS`)**:
   - In Ascend profiler DBs, operator names (`c.name`), operator types (`c.opType`), API names (`CANN_API.name`, `PYTORCH_API.name`), and HCCL group names (`COMMUNICATION_OP.groupName`) are stored as integer foreign keys into `STRING_IDS(id, value)`.
   - Device `TASK` rows link to `CANN_API` directly via `t.connectionId = c.connectionId`, whereas linking `TASK` to host `PYTORCH_API` MUST bridge through `CONNECTION_IDS` (`t.connectionId = conn.connectionId` AND `p.connectionId = conn.id`). Never join `TASK.connectionId` directly to `PYTORCH_API.connectionId`.

## Canonical CTE Macros (Track A — Copy Verbatim)

### 1. Operator Compute Breakdown (`compute_view`)
Use to rank Top-K time-consuming AI Core / AI CPU operators:
```sql
WITH compute_view AS (
    SELECT
        c.globalTaskId,
        ROUND(t.endNs - t.startNs) AS duration_ns,
        n.value AS op_name,
        type_str.value AS op_type
    FROM COMPUTE_TASK_INFO c
    LEFT JOIN TASK t ON t.globalTaskId = c.globalTaskId
    LEFT JOIN STRING_IDS n ON n.id = c.name
    LEFT JOIN STRING_IDS type_str ON type_str.id = c.opType
)
SELECT
    op_type,
    COUNT(*) AS calls,
    ROUND(SUM(duration_ns) / 1e6, 3) AS total_ms,
    ROUND(AVG(duration_ns) / 1e3, 3) AS avg_us
FROM compute_view
GROUP BY op_type
ORDER BY total_ms DESC
LIMIT 20;
```

### 2. HCCL Collective Communication Breakdown (`comm_view`)
Use to aggregate collective latency by `op_type` (`hcom_allReduce_`, `hcom_allGather_`, `hcom_allToAllV_`) and communication domain (`group_name`):
```sql
WITH comm_view AS (
    SELECT
        ROUND(c.endNs - c.startNs) AS duration_ns,
        n.value AS op_name,
        t.value AS op_type,
        g.value AS group_name
    FROM COMMUNICATION_OP c
    LEFT JOIN STRING_IDS n ON n.id = c.opName
    LEFT JOIN STRING_IDS t ON t.id = c.opType
    LEFT JOIN STRING_IDS g ON g.id = c.groupName
)
SELECT
    group_name,
    op_type,
    COUNT(*) AS calls,
    ROUND(SUM(duration_ns) / 1e6, 3) AS total_ms,
    ROUND(AVG(duration_ns) / 1e3, 3) AS avg_us
FROM comm_view
GROUP BY group_name, op_type
ORDER BY total_ms DESC
LIMIT 20;
```

### 3. Host-to-Device Dispatch Mapping (`dispatch_view`)
Use to compare PyTorch API duration, CANN `aclrtLaunch` duration, and actual NPU `TASK` execution duration:
```sql
WITH dispatch_view AS (
    SELECT
        ROUND(t.endNs - t.startNs) AS task_duration_ns,
        ROUND(c.endNs - c.startNs) AS cann_duration_ns,
        ROUND(p.endNs - p.startNs) AS pytorch_duration_ns,
        c_str.value AS cann_api_name,
        p_str.value AS pytorch_api_name,
        t_str.value AS task_type
    FROM TASK t
    LEFT JOIN CANN_API c ON t.connectionId = c.connectionId
    LEFT JOIN CONNECTION_IDS conn ON conn.connectionId = t.connectionId
    LEFT JOIN PYTORCH_API p ON p.connectionId = conn.id
    LEFT JOIN STRING_IDS c_str ON c.name = c_str.id
    LEFT JOIN STRING_IDS p_str ON p.name = p_str.id
    LEFT JOIN STRING_IDS t_str ON t.taskType = t_str.id
)
SELECT
    pytorch_api_name,
    cann_api_name,
    task_type,
    COUNT(*) AS count,
    ROUND(AVG(pytorch_duration_ns) / 1e3, 2) AS avg_pytorch_us,
    ROUND(AVG(cann_duration_ns) / 1e3, 2) AS avg_cann_us,
    ROUND(AVG(task_duration_ns) / 1e3, 2) AS avg_task_us
FROM dispatch_view
GROUP BY pytorch_api_name, cann_api_name, task_type
ORDER BY avg_cann_us DESC
LIMIT 20;
```

## Verification

- Before querying long-tail tables outside Track A (e.g., PMU or memory tables), list existing tables via `sqlite3 <db_path> "SELECT name FROM sqlite_master WHERE type='table' ORDER BY name;"` and inspect documented column semantics via `get_schema.py --table_name <TABLE>`.
