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
  - msprof op simulator
  - npusim
  - Triton-Ascend
  workload:
  - Cycle-accurate CAModel instruction trace analysis, source-line attribution, and
    pipeline bubble diagnosis
created_at: '2026-10-21'
domain_path: profiling/msprof-and-simulator/op-and-camodel/camodel-and-npusim-trace
estimate_tokens: 630
evidence_strength: software_contract
id: unit-msprof-simulator-four-table-and-npusim-bubble-diagnosis-sop
inherits_from:
- unit-ascend-msprof-aicore-pipe-and-host-wall-clock-axioms
provenance:
  evidence_refs:
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/plugins-community/triton-profiler-assistant/skills/triton-simulator-profiler/SKILL.md#L1-L260
  - https://gitcode.com/cann/cannbot-skills/blob/f305e200aa4b1e1fed73cd21ff1f083b8df25a1e/cann/npusim/SKILL.md#L1-L180
  promotion_status: verified
  source_locator: cann/cannbot-skills@f305e200aa4b1e1fed73cd21ff1f083b8df25a1e:plugins-community/triton-profiler-assistant/skills/triton-simulator-profiler/SKILL.md
  source_type: ecosystem_repo
  supersedes: []
revision: 1
schema: mindie-graded-unit/1
status: active
summary: Runs msprof op simulator (CAModel) with four-table (cubecore/veccore instr_exe
  + code_exe) cross-analysis and npusim record/report to diagnose cross-core WAIT_FLAG_DEVI
  stalls and vectorization bubbles.
tier: L2
title: msprof op simulator Four-Table Instruction Cross-Analysis and npusim Bubble
  Diagnosis SOP
triggers:
  error_signatures:
  - libruntime_camodel.so
  - WAIT_FLAG_DEVI
  - fixp_l0c2ub
  keywords:
  - msprof op simulator
  - instr_exe.csv
  - code_exe.csv
  - npusim
  - simd_vf_metrics
updated_at: '2026-10-21'
---
# msprof op simulator Four-Table Instruction Cross-Analysis and npusim Bubble Diagnosis SOP

## Core Principle

When hardware PMU counters (`msprof op`) show low pipe utilization (`vec_ratio` or `mac_ratio`) or unexplained stall cycles, cycle-accurate simulator tracing (`msprof op simulator` backed by `libruntime_camodel.so`, or `npusim record/report`) attributes every cycle to exact assembly instructions (`instr_exe.csv`) and high-level source lines (`code_exe.csv`). Because CAModel simulation runs $10^3\times\text{--}10^4\times$ slower than hardware, simulator runs must execute **exactly 1 invocation (zero warmup)** on a minimal representative shape.

## Part 1: `msprof op simulator` (CAModel) Four-Table Cross-Analysis

### Step 1: Mandatory Pre-Flight Environment & AST Gate

1. Append `<CANN_HOME>/tools/simulator/<SoCVersion>/lib` (containing `libruntime_camodel.so`) to `LD_LIBRARY_PATH`.
2. For Triton-Ascend kernels, export debug line-table flags so `code_exe.csv` maps PCs back to Python lines:
   ```bash
   export TRITON_DEBUG=1
   export TRITON_ALLWAYS_COMPILE=1
   export TRITON_DISABLE_LINE_INFO=0
   ```
3. **AST Coverage Gate**: Verify that 100% of the target operator compute resides inside `@triton.jit` or the Ascend C kernel; any fallback to `torch.*` ops outside the kernel distorts simulator attribution.
4. Run `msprof op simulator --soc-version=<SoCVersion> --output=<out_dir> python3 run_once.py`.

### Step 2: Cross-Analyze the Four Core CSV Tables

Navigate to `<out_dir>/OPPROF_*/simulator/core0.<core_type>0/` for **both** `core0.cubecore0` (AIC) and `core0.veccore0` (AIV) — never inspect only one core type in a CV mixed kernel:

| Table Pair (`core0.cubecore0` & `core0.veccore0`) | Key Columns | Diagnostic Action |
| --- | --- | --- |
| `code_exe.csv` | `Line`, `Call Count`, `Cycles`, `Duration(us)` | Rank source lines by total `Cycles` to isolate the top 3 hotspot statements |
| `instr_exe.csv` | `pipe`, `name`, `pc`, `start`, `end`, `duration` | Inspect the instruction timeline (`start..end` cycles) mapped to each hotspot `Line` |

- **Diagnosing `WAIT_FLAG_DEVI` Cross-Core Stalls**: When `core0.cubecore0/instr_exe.csv` shows massive `duration` on `WAIT_FLAG_DEVI` (`PIPE_S` waiting on AIV flag) while `core0.veccore0/instr_exe.csv` is busy executing `fixp_l0c2ub` or scalar address setup (`ADDI`/`MOV_XD_IMM`), the Cube core is starved waiting for Vector core buffer release—increase ping-pong buffer depth or hoist scalar offset arithmetic out of the inner loop.

## Part 2: `npusim` 4-Step Bubble & Vectorization Diagnosis

For Ascend 950 / RegBase workloads using `npusim`:

```bash
npusim record -o sim_trace -- ./run_op_once
npusim report -i sim_trace -f json -o summary.json
```

Inspect `summary.json` in strict 4-step order:
1. **Core Balance (`ai_core_active` & `imbalance_ratio`)**: If `imbalance_ratio > 0.15`, cores have uneven tile counts or early-exit branch divergence.
2. **Pipeline Overlap & Bubble Ratio**: Compare active cycles of `MTE2`, `VEC`/`CUBE`, and `MTE3` against total `ai_core_active` cycles; high gap cycles indicate missing double buffering (`TQue` depth = 1) or overly broad `PipeBarrier<PIPE_ALL>()`.
3. **Vector Lane Efficiency (`simd_vf_metrics`)**: Check vector instruction ratio vs scalar fallback inside Vector VF blocks; low vector instruction density indicates scalar loop indexing inside the vector body.
4. **On-Chip Data Path Bypass (`L0C->UB` vs `L0C->GM->UB`)**: Verify CV mixed kernels route Matmul accumulators directly through `Fixpipe` (`L0C -> UB`) on supported architectures rather than round-tripping through GM (`L0C -> GM -> MTE2 -> UB`).
