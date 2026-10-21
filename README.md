<p align="center">
  <img src="https://raw.githubusercontent.com/mindie-agent/mindie-agent/main/assets/brand/mindie-agent-logo.png" alt="MindIE Agent logo" width="128" height="128">
</p>

<h1 align="center">MindIE Agent Knowledge — 推理可观测性、诊断与性能剖析知识库 (<code>knowledge-infer-observability</code>)</h1>

<p align="center">
  <strong>面向昇腾 NPU 故障诊断、性能剖析与精度验证的 L1–L4 分级工程知识库：覆盖 CANN/plog/msaicerr 报错排障、msprof/torch_npu Profiling、精度对齐与基准测量</strong>
</p>

<p align="center">
  <a href="https://github.com/mindie-agent/mindie-agent"><img src="https://img.shields.io/badge/MindIE%20Agent-Ecosystem-7045f5.svg" alt="MindIE Agent"></a>
  <a href="./publication-contract.json"><img src="https://img.shields.io/badge/Schema-mindie--graded--unit%2F1-0f766e.svg" alt="Schema mindie-graded-unit/1"></a>
  <a href="./publication-contract.json"><img src="https://img.shields.io/badge/domain-infer--observability-0284c7.svg" alt="Domain infer-observability"></a>
  <a href="./index/routing-manifest.json"><img src="https://img.shields.io/badge/Graded%20Units-23%20Units-d97757.svg" alt="23 Graded Units"></a>
  <a href="./index/shard-topology.json"><img src="https://img.shields.io/badge/Levels-L1%20to%20L4-10a37f.svg" alt="Levels L1-L4"></a>
</p>

<p align="center">
  <strong>中文</strong> · <a href="./README.en.md">English</a> · <a href="#-一句话接入与检索">⚡ 一句话接入</a> · <a href="#️-领域全景与-l1l4-分级架构">🏗️ 分级架构</a> · <a href="#-子领域分布与代表性知识单元">📊 语料全景</a> · <a href="#-仓库结构与隐私质量门禁">🛡️ 结构与门禁</a>
</p>

---

## ⚡ 一句话接入与检索

你不需要手动克隆或配置索引。在 **Claude Code**、**OpenAI Codex CLI**、**Kimi Code** 或 **DeepSeek Harness (DSH)** 中，直接复制并发送下面这句话给你的 AI Agent：

```text
帮我安装一下这个 https://github.com/mindie-agent/mindie-agent
```

宿主插件就绪后会自动连接 [`mindie-knowledge`](https://github.com/mindie-agent/knowledge) 本地运行时，按需同步本仓库（`--domain infer-observability`）并在本地构建毫秒级确定性检索索引。只读检索与全文读取**完全在本地执行，不消耗大模型 Token，也不要求开启社区共享**。

<details>
<summary>🤖 <b>AI Agent 自动安装与知识检索协议（点击展开 / Agent 读取本页时可直接解析执行）</b></summary>

> **给 AI Agent 的执行指南**：
>
> #### 1. 一键安装宿主插件
> 若当前宿主尚未安装 MindIE Agent，请先读取主仓库 [`https://github.com/mindie-agent/mindie-agent`](https://github.com/mindie-agent/mindie-agent) 的安装协议完成对应宿主插件注册。
>
> #### 2. 命令行同步与检查 `infer-observability` 领域语料
> ```bash
> # 同步 infer-observability 领域最新经校验的公开知识源（零模型调用）
> mindie-knowledge sync --domain infer-observability
>
> # 查看本地知识索引状态与脱敏规则版本
> mindie-knowledge status --domain infer-observability
> python3 -m mindie_knowledge.redact --profile
> ```
>
> #### 3. MCP 工具检索、精确全文读取与版本化反馈
> 在解决 `infer-observability` 相关工程任务时，遵循 **先按实测条件检索 (`knowledge_query`) → 再按精确修订号读全文 (`knowledge_explain`) → 验证后可选反馈 (`knowledge_feedback`)** 的标准工作流：
>
> ```python
> # Step 1: 仅传入当前任务真实观测到的环境/版本条件（严禁猜测或抄写未观测条件）
> knowledge_query(
>     query="plog first error asys msaicerr aicore exception bitmask",
>     conditions={
>         "soc": "Ascend910B4"
>     },
> )
>
> # Step 2: 使用返回的精确修订引用 (@revision) 读取完整正文、前置条件与继承闭包
> knowledge_explain(
>     ref="mindie://infer-observability/units/unit-ascend-plog-root-cause-and-asys-msaicerr-collection-sop@1",
> )
>
> # Step 3: 实机验证有效后，针对实际观测过的修订号记录本地有用性反馈（可选）
> knowledge_feedback(
>     ref="mindie://infer-observability/units/unit-ascend-plog-root-cause-and-asys-msaicerr-collection-sop@1",
>     rating="up",
>     reason="Located root-cause inner error in plog and collected msaicerr dump evidence.",
> )
> ```

</details>

---

## 🏗️ 领域全景与 L1–L4 分级架构

本仓库承载 MindIE Agent 的 **`infer-observability` 领域分级知识源**（共 **23 个分级知识单元**：`L1: 4`、`L2: 14`、`L3: 5`）。聚焦昇腾 NPU 推理与算子开发全链路可观测性：涵盖 CANN 四层错误码与最内层首报定位、plog / asys / msnpureport / msaicerr 34 步 AI Core 异常 Dump 解析、`npu-objdump` PC 偏移反汇编、`mssanitizer` 与 `npu-check` 内存/同步检测、`msprof` / `torch_npu.profiler` / `npusim` 性能剖析、端到端 TTFT/TPOT 时延归因，以及多精度（MERE/MARE）容差标准与 CPU/NPU 孪生白盒测试门禁。

### 1. `mindie-graded-unit/1` 四级知识单元体系（L1–L4）

本知识库拒绝碎片化的聊天摘要或脱离前提的“万能经验”，所有知识单元均按工程抽象粒度与可操作性划分为 **L1–L4 四个层级**（当前分布：**L1: 4 · L2: 14 · L3: 5 · L4: 0**）：

| 层级 | 标准类别 (`category`) | 工程定位 | 核心内容与准入要求 |
| :---: | --- | --- | --- |
| **L1** | `canonical-fact` (`axiom` / `measurement`) | **事实与约束标准** | 硬件微架构规格（UB/L1/HBM 对齐、双 Die 拓扑）、框架核心不变量、ABI 注册规则与四视图 Roofline 测量基线。常作为上层单元的继承前提（`parents`）。 |
| **L2** | `sop` | **标准排障与开发规程** | 确定性的分步诊断、算子开发、图编译护栏排查与精度比对标准作业程序（SOP），明确列出观测指标、判别分支与验证命令。 |
| **L3** | `workflow` | **多阶段工程工作流** | 端到端多阶段闭环工作流（如 0-day 模型适配、TTFT/TPOT 性能瓶颈收敛、分布式死锁根因定位、留一法消融实验）。 |
| **L4** | `optimization-pattern` (`mutator`) | **深度调优模式** | 经过实测验证的 Kernel / 计算图 / 通信并行代码变换模式（Mutator），附带严格的适用条件（`conditions`）、消融前后性能包络与源码溯源（`provenance`）。 |

### 2. 条件化路由与精确全文检索架构

```mermaid
flowchart LR
  Agent["🤖 AI Coding Agent (Claude Code / Codex / Kimi / DSH)"]
  Query["🔎 knowledge_query (查询词 + 实测环境条件)"]
  Manifest["📋 index/routing-manifest.json (23 个单元元数据 · 条件与继承闭包)"]
  Topology["🗂️ index/shard-topology.json (子领域分片拓扑 · L1-L4 统计)"]
  Match{"⚖️ 条件匹配引擎 (matched / mismatch / condition_unknown)"}
  Explain["📖 knowledge_explain (按精确修订号 @revision 精读)"]
  Units["📄 units/{subdomain}/{level}/unit-*.md (L1-L4 正文 + Provenance 溯源)"]

  Agent --> Query --> Manifest --> Topology --> Match
  Match -- "声明条件全部满足 (matched)" --> Explain --> Units
```

> [!IMPORTANT]
> **严格条件匹配原则（No Silent Guesswork）**：查询时，`mindie-knowledge` 会将任务真实观测到的 `conditions` 与知识单元声明的适用前提逐项比对。只有所有声明条件均满足时才判定为 `matched`；缺失或未知的条件判定为 `condition_unknown` 并默认过滤，避免将特定硬件型号或特定源码提交下的结论误用于不兼容环境。

---

## 📊 子领域分布与代表性知识单元

[`index/routing-manifest.json`](./index/routing-manifest.json) 与 [`index/shard-topology.json`](./index/shard-topology.json) 是由全量 Markdown 单元派生的权威实时清单。当前仓库共包含 **23 个分级知识单元**（**L1: 4 · L2: 14 · L3: 5 · L4: 0**）：

### 1. 子领域（Subdomain）分布统计

| 子领域目录 | 单元数量 | 层级分布 (`L1–L4`) | 核心覆盖范围 |
| --- | :---: | :---: | --- |
| `diagnostics` | **10** | `L2: 7, L3: 3` | CANN 错误码四层证据链工作流、plog 首报定位、asys/msnpureport 黑匣子采集、msaicerr 34 步异常解析（Stars/David 寄存器解码）、`npu-objdump` PC 反汇编与 `mssanitizer` 内存异常检测 |
| `measurement` | **5** | `L1: 1, L2: 3, L3: 1` | `msprof` 流水线利用率与 Host 挂钟时间边界公理、MFU FLOPs 归因与 Step 重叠区间计算、`npu-compute`/`npu-check` 算子级测量及服务化 TTFT/TPOT 四阶段回归归因工作流 |
| `profiling` | **5** | `L1: 1, L2: 3, L3: 1` | `msprof op` PMU 指标与 Ascend 950 Vector Roofline 公理、Simulator 四表交叉分析与 `npusim` 气泡诊断、`torch_npu.profiler` Launch Gap 归因、Triton 算子 MFU 网格搜索与端到端 HCCL 通信掩盖工作流 |
| `validation` | **2** | `L1: 1, L2: 1` | 多数据类型精度标准（MERE/MARE vs `allclose`、NaN 比较陷阱、Host BF16 转换）公理与 AscendC CPU/NPU 孪生 UT/ST 白盒 TilingKey 覆盖规程 |
| `hardware` | **1** | `L1: 1` | 双 Die NPU 机型上 NPU ID、Chip ID 与物理 `Phy-ID` 拓扑映射实测事实 |

### 2. 代表性分级知识单元（Representative Units）

| 知识单元 ID（点击查看原文） | 子领域 | 层级 / 类别 | 工程主题与摘要 | 固定修订引用 (`ref`) |
| --- | :---: | :---: | --- | --- |
| [`unit-ascend-msprof-aicore-pipe-and-host-wall-clock-axioms`](./measurement/msprof/unit-ascend-msprof-aicore-pipe-and-host-wall-clock-axioms.md) | `measurement` | **L1** / `measurement` | Ascend msprof pipe ratios and host wall clock are separate measurement domains | `mindie://infer-observability/units/unit-ascend-msprof-aicore-pipe-and-host-wall-clock-axioms@1` |
| [`unit-msprof-op-pmu-metrics-and-a5-vector-roofline-axioms`](./profiling/msprof-and-simulator/op-and-camodel/pmu-metrics-and-roofline/unit-msprof-op-pmu-metrics-and-a5-vector-roofline-axioms.md) | `profiling` | **L1** / `measurement` | msprof op 10-Pair Median Timing, PMU Metric Groups, and Ascend 950 Vector Roofline Axioms | `mindie://infer-observability/units/unit-msprof-op-pmu-metrics-and-a5-vector-roofline-axioms@1` |
| [`unit-multi-dtype-tolerance-mere-mare-and-bf16-comparator-axioms`](./validation/testing-and-golden/ut-st-benchmarks/tolerance-and-comparators/unit-multi-dtype-tolerance-mere-mare-and-bf16-comparator-axioms.md) | `validation` | **L1** / `axiom` | Multi-Dtype Precision Standards (MERE/MARE vs allclose), NaN Comparator Traps, and Host BF16 Conversion Axioms | `mindie://infer-observability/units/unit-multi-dtype-tolerance-mere-mare-and-bf16-comparator-axioms@1` |
| [`unit-observed-dual-die-phy-id-topology`](./hardware/topology/unit-observed-dual-die-phy-id-topology.md) | `hardware` | **L1** / `measurement` | Observed NPU, Chip and Phy-ID mapping on one dual-die machine | `mindie://infer-observability/units/unit-observed-dual-die-phy-id-topology@1` |
| [`unit-ascend-plog-root-cause-and-asys-msaicerr-collection-sop`](./diagnostics/plog-asys/unit-ascend-plog-root-cause-and-asys-msaicerr-collection-sop.md) | `diagnostics` | **L2** / `sop` | Five-step SOP from plog first error to asys and msaicerr evidence | `mindie://infer-observability/units/unit-ascend-plog-root-cause-and-asys-msaicerr-collection-sop@1` |
| [`unit-ascend-npu-objdump-and-aicore-exception-bitmask-triage-sop`](./diagnostics/aicore-dump/unit-ascend-npu-objdump-and-aicore-exception-bitmask-triage-sop.md) | `diagnostics` | **L2** / `sop` | Five-step PC and aic_error bitmask triage for Ascend AI Core exception dumps | `mindie://infer-observability/units/unit-ascend-npu-objdump-and-aicore-exception-bitmask-triage-sop@1` |
| [`unit-mssanitizer-six-anomaly-classes-and-npu-check-synccheck-sop`](./diagnostics/cann-toolchain/sanitizer-and-debug-api/mssanitizer-and-npu-check/unit-mssanitizer-six-anomaly-classes-and-npu-check-synccheck-sop.md) | `diagnostics` | **L2** / `sop` | mssanitizer Six Memory Anomaly Classes, Caching Allocator Hooks, and npu-check synccheck SOP | `mindie://infer-observability/units/unit-mssanitizer-six-anomaly-classes-and-npu-check-synccheck-sop@1` |
| [`unit-torch-npu-profiler-schedule-and-launch-gap-attribution-sop`](./profiling/msprof-and-simulator/system-and-timeline/torch-npu-profiler-and-gaps/unit-torch-npu-profiler-schedule-and-launch-gap-attribution-sop.md) | `profiling` | **L2** / `sop` | torch_npu.profiler Deterministic Schedule, Dual-Path Baseline, and Host-to-Device Launch Gap Attribution SOP | `mindie://infer-observability/units/unit-torch-npu-profiler-schedule-and-launch-gap-attribution-sop@1` |
| [`unit-ascend-cann-error-code-four-layer-evidence-and-remediation-workflow`](./diagnostics/error-codes/unit-ascend-cann-error-code-four-layer-evidence-and-remediation-workflow.md) | `diagnostics` | **L3** / `workflow` | CANN operator error-code four-layer evidence verification, innermost first-report rule, and E*9*** remediation workflow | `mindie://infer-observability/units/unit-ascend-cann-error-code-four-layer-evidence-and-remediation-workflow@1` |
| [`unit-ascend-msaicerr-34-step-exception-dump-and-return-code-workflow`](./diagnostics/aicore-dump/unit-ascend-msaicerr-34-step-exception-dump-and-return-code-workflow.md) | `diagnostics` | **L3** / `workflow` | Closed-loop 34-step AI Core exception dump workflow across Stars/David register decoding, special scenarios, and 101-107 return codes | `mindie://infer-observability/units/unit-ascend-msaicerr-34-step-exception-dump-and-return-code-workflow@1` |
| [`unit-ascend-serving-ttft-tpot-and-operator-attribution-workflow`](./measurement/end-to-end/unit-ascend-serving-ttft-tpot-and-operator-attribution-workflow.md) | `measurement` | **L3** / `workflow` | Four-stage attribution of serving TTFT and TPOT regressions across host, graph, HCCL and AICore | `mindie://infer-observability/units/unit-ascend-serving-ttft-tpot-and-operator-attribution-workflow@1` |
| [`unit-end-to-end-hccl-overlap-and-eight-step-precision-triage-workflow`](./profiling/msprof-and-simulator/system-and-timeline/hccl-overlap-and-precision-flow/unit-end-to-end-hccl-overlap-and-eight-step-precision-triage-workflow.md) | `profiling` | **L3** / `workflow` | End-to-End HCCL Communication Overlap Analysis and Eight-Step Operator Precision Triage Workflow | `mindie://infer-observability/units/unit-end-to-end-hccl-overlap-and-eight-step-precision-triage-workflow@1` |

---

## 🛡️ 仓库结构与隐私质量门禁

### 1. 仓库目录结构

```text
knowledge-infer-observability/
├── publication-contract.json           # 发布契约 (mindie-publication-contract/2) 与固定 Validator 修订号
├── AGENTS.md                           # 维护者与 AI Agent 仓库操作边界规范
├── README.md                           # 中文主文档
├── README.en.md                        # 英文主文档 (English Documentation)
├── index/
│   ├── routing-manifest.json           # [派生索引] 全量单元 ID、路径、层级、条件、继承闭包与 Token 估算
│   └── shard-topology.json             # [派生索引] 子领域分片计数与层级拓扑
├── <subdomain>/.../unit-*.md           # [核心语料] L1–L4 分级知识单元 Markdown 正文与 YAML Frontmatter
└── docs/
    └── evidence/                       # 源码审计收据、实验消融数据与溯源证据
```

### 2. 双层隐私脱敏与确定性 CI 发布门禁（`.github/workflows/check.yml`）

本仓库作为公开工程知识源，在本地构建与 GitHub Actions CI（`check.yml`）中强制执行**零信任数据校验与双层隐私脱敏门禁**：

1. **不可变契约与纯数据校验（`publication_check.validate`）**：
   - CI 从可信基准读取 [`publication-contract.json`](./publication-contract.json)，安装精确固定提交的 `mindie-knowledge` 校验器；
   - 所有候选文件仅作为 Git 数据对象（`git cat-file --batch`）流式读取，**绝不在 CI 中执行任何候选代码**；
   - 严格校验每个 `unit-*.md` 的 Schema（`mindie-graded-unit/1`）、路径与稳定 ID 一致性、修订号与 SHA-256 内容摘要、父子继承有向无环图（DAG），以及 `index/routing-manifest.json` 和 `index/shard-topology.json` 与实际单元的 100% 一致性。
2. **双层闭锁式隐私扫描（Gitleaks + `mindie_knowledge.redact` `r3`）**：
   - 全仓所有文本文件、知识单元 Frontmatter/正文、公开反馈及 PR 标题/描述均经过确定性隐私扫描；
   - **严禁包含**任何本地宿主绝对路径（如个人主目录、挂载卷路径）、私有内网 IPv4/IPv6 地址段、SSH 登录凭据、API 密钥或内部机器名，一旦命中立即阻断合入。

---

## 🔗 关联仓库与生态导航

- **主仓库与一键安装入口**：[`mindie-agent/mindie-agent`](https://github.com/mindie-agent/mindie-agent)
- **知识检索与隐私脱敏运行时**：[`mindie-agent/knowledge`](https://github.com/mindie-agent/knowledge)（Python 包 `mindie-knowledge`）
- **七大公开领域知识库矩阵**：
  - [`knowledge-vllm-ascend`](https://github.com/mindie-agent/knowledge-vllm-ascend) — `vLLM-Ascend` 适配、图编译、MoE 与 PD 分离（43 Units）
  - [`knowledge-operators`](https://github.com/mindie-agent/knowledge-operators) — AscendC / BiShengIR / Triton / TileLang 算子开发与优化（87 Units）
  - [`knowledge-infer-core`](https://github.com/mindie-agent/knowledge-infer-core) — 核心推理引擎调度、KV Cache、多流并行与服务化架构（78 Units）
  - [`knowledge-infer-distributed`](https://github.com/mindie-agent/knowledge-infer-distributed) — HCCL 集合通信、RoCE/HCCS 拓扑、多维并行与 SHMEM（23 Units）
  - [`knowledge-infer-observability`](https://github.com/mindie-agent/knowledge-infer-observability) — CANN 故障诊断、`msprof` 性能剖析与精度验证（23 Units）
  - [`knowledge-infer-ops-compat`](https://github.com/mindie-agent/knowledge-infer-ops-compat) — GE/TorchAir 图编译、自定义算子注册与 CUDA 迁移兼容（22 Units）
  - [`knowledge-agent-engineering`](https://github.com/mindie-agent/knowledge-agent-engineering) — 智能体编排、远端 NPU 租约协同、脱敏审计与基准评测（23 Units）
