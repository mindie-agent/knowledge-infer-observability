---
schema: mindie-graded-unit/1
id: unit-ascend-serving-ttft-tpot-and-operator-attribution-workflow
revision: 1
tier: L3
category: workflow
domain_path: measurement/end-to-end
evidence_strength: reproduced_empirical
title: Four-stage attribution of serving TTFT and TPOT regressions across host, graph, HCCL and AICore
conditions:
  workload: online LLM serving
  metrics:
  - TTFT
  - TPOT
  measurement_sources:
  - serving histogram
  - step_trace_time.csv
  - op_summary_*.csv
  - trace_view.json
  - cluster communication DB
  execution_modes:
  - eager
  - ACLGraph replay
  load_control: matched concurrency, input and output lengths, warmup and prefix-cache state
content_digest: sha256:4dacb5b487d24134b7d3ecaf797f23b75ed345a99237de58f182897a7fe7d132
inherits_from:
- unit-ascend-msprof-aicore-pipe-and-host-wall-clock-axioms
- unit-msprof-analyze-step-overlap-clock-and-intervals
- unit-msprof-analyze-mfu-range-flops-attribution
provenance:
  source_type: run_manifest
  source_locator: docs/evidence/cannbot-observability-provenance.json
  evidence_refs:
  - docs/evidence/cannbot-observability-provenance.json#observed_measurements.obs-serving-single-request
  - docs/evidence/cannbot-observability-provenance.json#observed_measurements.obs-serving-concurrent-32
  - docs/evidence/cannbot-observability-provenance.json#observed_measurements.obs-step-trace-128k
  - docs/evidence/cannbot-observability-provenance.json#observed_measurements.obs-itl-stall
  - docs/evidence/cannbot-observability-provenance.json#observed_measurements.obs-op-summary-wait
  - docs/evidence/cannbot-observability-provenance.json#observed_measurements.obs-aclgraph-replay
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:model/model-infer-profiling/SKILL.md#L185-L283
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:model/model-infer-graph-mode/references/llm-model-guide.md#L11-L75
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:model/model-infer-graph-mode/references/llm-model-guide.md#L213-L221
  - cannbot-skills@1fa2cbbc9fdddb4377ca26734e9b94d91c42b975:plugins-community/triton-optimizer/skills/triton-npu-optimize/references/triton-npu-analyze-round-performance/triton-npu-profiling-analysis.md#L41-L71
  - Ascend/msagent@521f8ead0ebda5db5b282f0c3c4147006e01e016:skills/profiler/ascend-communication-analysis/SKILL.md#L8-L15
  - Ascend/msagent@521f8ead0ebda5db5b282f0c3c4147006e01e016:skills/profiler/ascend-communication-analysis/SKILL.md#L168-L196
  source_digest: 814abc1f4a1141018c23b53388f21ee219ba15a1be1b87c1543f491685c1b68d
  supersedes: []
triggers:
  error_signatures:
  - TTFT regression
  - TPOT regression
  - long-tail ITL spike
  - "generation throughput=0"
  - graph break
  - aclgraph capture failure
  - ".item() sync in forward"
  - HCCL collective wait
  keywords:
  - TTFT
  - TPOT
  - ITL
  - serving
  - ACLGraph
  - graph capture
  - graph replay
  - D2H
  - item
  - HCCL
  - collective wait
  - op_summary
  - step_trace
  - AICore
  - AIV
  - attribution
  - prefill
  - decode
---

A serving regression is not the same object as a slow kernel. TTFT and TPOT are end-to-end service metrics; `Task Duration` and pipe ratios are device-side operator measurements. Attribute a regression only by moving from the service metric to the step budget, then to the launch or collective path, and only then to the operator. Each stage must produce a measurement that can fail independently.

## Stage 1: Pin the SLO and the load protocol

Define the two metrics explicitly:

- **TTFT**: from request arrival to the first generated token. In a monolithic deployment it includes queueing and prefill; in a PD deployment, state whether the measured interval includes the prefill handoff or only the decode-side wait.
- **TPOT**: the steady-state inter-token interval during decode. It is not total latency divided by output tokens, and it is not TTFT.

For vLLM-style serving, read the histogram per engine and per role: `vllm:time_to_first_token_seconds` and `vllm:request_time_per_output_token_seconds` (some versions document the latter as `vllm:time_per_output_token_seconds`). Use `_sum` and `_count` for a mean, and bucket boundaries for quantiles only when the buckets resolve the SLO. In a PD deployment, the serving histogram's TTFT and TPOT are decode-scoped; do not aggregate prefill nodes into a decode metric unless the SLO explicitly includes prefill time.

Freeze the load before comparing anything: model and weights, CANN and driver, parallelism, concurrency, input and output lengths, sampling parameters, speculative-decoding settings, prefix-cache state, maximum tokens and run duration. Warm up compilation and graph capture outside the measured window. Put `torch.npu.synchronize()` at the boundaries of any host-clock measurement, and prefer the server's histograms for TTFT/TPOT because a client clock also contains network and client queueing. Record cold and warm phases separately.

The recorded packet shows why this matters. A single engine-0 request measured TTFT about 1.258 s and TPOT about 14.9 ms. In the same recorded eval directory, after a 128-concurrency phase, engine 0's 32 successful requests averaged TTFT about 76.37 s and TPOT about 700 ms. That is not a clean regression: concurrency, batching and queueing changed. It is a warning that a single-request baseline cannot be compared with a saturated run.

Stage output: a run manifest (model, CANN, parallelism, concurrency, sequence lengths, cache state, warmup), a baseline and candidate metric table with `n`, mean and quantiles, and a statement of which phase is being attributed.

Decision: if TTFT rises while TPOT is flat, look at queueing and prefill. If TPOT rises while TTFT is flat, look at the decode loop. If both rise, check load, collective exposure and profiler overhead before blaming an operator.

## Stage 2: Split host, device and collective time

Use `step_trace_time.csv` as the step budget. The observed fields are `Computing`, `Communication(Not Overlapped)`, `Overlapped`, `Communication`, `Free`, `Stage`, `Bubble` and `Preparing`. In the recorded 128K decode window the identity was:

```text
Stage = Computing + Communication - Overlapped + Free
```

The same window reported `Stage` 2525.010 ms, `Computing` 554.151 ms, `Communication` 1903.954 ms, `Overlapped` 9.408 ms and `Free` 76.314 ms. The exposed communication was 1894.546 ms, about 75% of the window, but that is a single-rank view and communication time includes waiting. It is not a bandwidth measurement.

Read the budget in this order:

1. **`Free / Stage` high**: host scheduling, Python overhead, launch gaps or profiler stop/parse. Inspect `api_statistic` for host API cost, `operator_details` for host self/total time, and `trace_view` for gaps. Check whether the measurement window contains profiler start/stop or synchronous parse. In the recorded ITL investigation, `stop_profile` waited about 45 s and 56 s while 19 and 23 requests were running, and generation throughput was zero. That is measurement interference, not a kernel regression. Use asynchronous parse and keep the load test separate from profiling.
2. **`Communication(Not Overlapped) / Stage` high**: collective exposure. Separate wait-caused from transfer-slow. At rank level, compare the same op's start and end across ranks; a rank that starts early and finishes late is waiting. In the cluster DB, use `COMMUNICATION_OP` timing, `Notify Wait`, cross-rank same-op alignment and, only for non-wait cases, transfer size, link type and bandwidth. A single rank's low bandwidth is not proof of a network fault. In the recorded startup, an AllReduce AICPU launch produced `507018 / 0x715002a` with AICPU `errorCode=0x2a`; that is a task-side failure, not a bandwidth conclusion.
3. **`Computing / Stage` high and `Free` low**: the device is continuously busy. Move to Stage 4 for operator attribution. Do not stop at "device-bound"; identify the operator and pipe.

If `step_trace_time.csv` is missing, reconstruct task gaps from `task_time` start/stop timestamps and label them "task gaps", not pure host time. A gap between tasks can also come from a dependency or a collective, so corroborate with the trace and API view.

Stage output: one row per candidate cause with the measured share, the supporting artifact, and a counter-check. Do not carry a Stage 1 regression label into Stage 2 without a matching load.

## Stage 3: Verify graph capture and D2H synchronization

Before operator attribution, verify which execution path actually ran. Count `aclmdlRIExecuteAsync` and `MODEL_EXECUTE` per decode step. In the recorded 50-step sample, the two windows had 52 and 48 `aclmdlRIExecuteAsync` calls and 52 and 48 `MODEL_EXECUTE` events, matching a main-plus-draft replay pattern. That sample covered about six seconds and did not show a whole-model eager fallback. It does not prove the absence of fallback elsewhere, and graph-out metadata, sampling or logits matmul do not by themselves prove a full fallback.

Check the graph path:

- Look for capture and replay evidence, not only a configuration flag. `Replaying aclgraph` or a backend-specific replay frame is stronger than "graph mode enabled".
- Check `torch._logging.set_logs(recompiles=True)` and the recompilation reason. A shape, guard, memory-address or Python-control-flow change can force a new capture.
- Check `capture_limit` and the number of captures per step. Repeated capture inside a serving loop is a capture break, even if each individual capture succeeds.
- Check for D2H synchronization inside forward: `.item()`, `.cpu()`, `int(tensor)`, `bool(tensor)`, `float(tensor)`, `torch.nonzero(...).item()`, and any Python branch that reads a device scalar. `.item()` forces a graph break and a device-to-host copy; the fix is a static parameter, a precomputed value or an input tensor.
- A/B the same workload with `force_eager` or the backend's eager switch. If graph mode is slower but eager is normal, the regression is in capture/replay or a graph break. If both are slow, the cause is host or operator, not the graph wrapper.

A graph-out small operator is not evidence of a whole-model fallback. Use the replay count and the full step budget together.

Stage output: a graph-path verdict (replay, repeated capture, fallback or unknown), the evidence line, and the D2H/sync sites found.

## Stage 4: Attribute to a specific AICore or AIV operator

Start with `op_statistic.csv` to rank operators by total time, then use `op_summary_*.csv` for the candidate rows. `op_summary` is a per-op aggregate, not per-core data. Read `Task Wait Time` before `Task Duration`: if the wait is much larger than the duration, the operator is queueing or scheduling, not executing slowly. In the recorded export, a `Muls` row had 14.158 us duration and 5300.659 us wait; a `MlaPrologV3` row had 52.778 us duration and 1650.358 us wait. Both waits dominate their device durations.

Then read the pipe ratios with their limits:

| Observation | Candidate attribution | Corroboration |
|---|---|---|
| Low `aic_mac_ratio` with high `aic_mte2_ratio` on a Cube operator | Tile/feed or memory-path limitation | Shape, tiling, active bandwidth, `cube_utilization`, trace overlap |
| High `aiv_vec_ratio` on a Vector operator | Vector-instruction bound | Vector instruction mix, scalar ratio, bank/conflict counters |
| High `aiv_mte3_ratio` | Writeback pressure | Output size, store pattern, overlap with compute |
| High `aic_fixpipe_ratio` | FixPipe or address-alignment pressure | `Task Type`, layout, alignment, output stride |
| High `aic_scalar_ratio` or `aiv_scalar_ratio` | Scalar/control overhead, masking or address generation | Code mapping, instruction mix, branch/count logic |
| High `Task Wait Time` with low device time | Queueing or scheduling, not the kernel | `step_trace` gaps, host API, stream dependencies |

The recorded MlaPrologV3 row had `aic_mte2_ratio` 0.582, `aic_mac_ratio` 0.259 and `cube_utilization(%)` 91.09. The high cube-utilization label did not mean a high MAC share; MTE2 was the larger AIC pipe. That is the kind of distinction Stage 4 must preserve.

Confirm the operator attribution with one-factor A/B: same model, same load, same shapes, only the candidate operator or tiling changed. Report the median, spread and sample count. Use `kernel_details.csv` for per-launch sequences and `trace_view.json` for overlap. Do not sum overlapping stream times as wall time, and do not multiply a single-layer module saving by layer count and present it as a measured service gain. The packet's module-level comparisons are explicitly labelled as estimates, not whole-service ITL results.

Stage output: operator name and type, launch count, `Task Duration`, `Task Wait Time`, the dominant pipe ratios, the A/B result and a counter-evidence line. If the operator's wait dominates, return to Stage 2 rather than proposing a kernel rewrite.

## Attribution rule

A regression is attributed only when the Stage 1 metric moves under a matched load, the Stage 2 budget moves in the same direction, the Stage 3 path explains where the time is spent, and the Stage 4 operator or collective has a corroborating A/B. If any stage is missing, report the strongest supported statement and the missing evidence. A single `op_summary` row, a single slow kernel, or a single graph-break log is not enough.

Adoption boundary: the workflow is grounded in the recorded serving, step-trace, graph-replay and operator measurements in the provenance packet. Those runs used different parallelism and load settings; they illustrate the stages and the failure modes, not a universal performance target. This distillation did not execute new NPU work, and none of the observed values is a fleet-wide baseline.
