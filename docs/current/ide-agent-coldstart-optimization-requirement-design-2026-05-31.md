# IDE Agent Coldstart Optimization 需求设计

Design Key：IDE-AGENT-COLDSTART-OPTIMIZATION-2026-05-31
日期：2026-05-31
状态：ready-for-workspace / formal-requirement
维护窗口：AlembicDesign
总控：AlembicWorkspace
目标仓库：AlembicCore / AlembicPlugin
原始计划：[ide-agent-coldstart-optimization-original-plan-2026-05-31.md](ide-agent-coldstart-optimization-original-plan-2026-05-31.md)

## 定位

本需求优化 IDE/host-agent cold-start 路径：让 Alembic 现有 AST、Dependency Graph、Code Entity Graph、Call Graph、Data Flow、Panorama、Guard 和 SourceRef 能力，形成 IDE Agent 可执行、可验证、可恢复的冷启动分析包。

它不是 UI 需求，不是 internal AI 重写，不是 Plugin 独立结构分析，也不是完整 rescan evolution 需求。第一版只聚焦 IDE Agent 如何更准确地拿到冷启动任务、阅读源码、产出 Recipe，并用真实 IDE Agent 验证前后效果。

## 设计总目标

把当前 one-shot Mission Briefing 升级为 guided structure-to-recipe loop：

```text
ProjectIntelligence snapshot
-> IDEAgentAnalysisPacket
-> IDEAgentAnalysisUnit
-> IDE/host Agent reads assigned source refs
-> Recipe links unit + sourceRefs + structural evidence
-> dimension_complete reports unit coverage
-> checkpoint/job recovers unit progress
-> real IDE Agent test compares before/after
```

目标不是让 briefing 更长，而是让 Agent 拿到更准的下一步。

## 通用能力复用原则

本需求只新增 IDE/host Agent 面向的执行投影、领取协议、证据链接和 adapter 验证；基础通用的非 Agent 能力应尽量复用，不为 Agent 路径重造第二套。

应优先复用的通用能力：

- ProjectIntelligence Phase 1-4。
- AST / Code Entity Graph / Call Graph / Data Flow / Dependency Graph / Panorama / Guard。
- `alembic_structure` / `alembic_graph` / `alembic_call_context` 等结构查询能力。
- `reasoning.sources`、`recipe_source_refs`、`SourceRefReconciler`。
- `RecipeProductionGateway`、UnifiedValidator、KnowledgeService / KnowledgeRepository。
- Knowledge rescan plan、prescreen、evidencePlan、relevance audit。
- BootstrapSession、DimensionCheckpoint、JobStore / recoverable job。

如果这些底座能力本身不足，应在通用模块补齐契约，再由 IDE Agent packet / unit 消费。只有下列内容属于本需求的 Agent-facing 范围：

- `IDEAgentAnalysisPacket` / `IDEAgentAnalysisUnit` projection。
- Agent 可领取 / 可恢复的 unit progress。
- Agent 提交 Recipe 时的 unit evidence linkage。
- 第一条 adapter 的真实 before/after 验收。

禁止把通用能力复制到 Plugin 或某个 IDE adapter 内部，例如第二套 AST、第二套 sourceRefs、第二套 rescan planner、第二套 method-flow query。

## 现有链路边界

### 当前 cold-start

```text
alembic_bootstrap
-> ExternalColdStartWorkflow
-> ProjectIntelligenceCapability.run
-> buildExternalMissionBriefing
-> buildMissionBriefing
-> IDE/host agent reads briefing
-> alembic_submit_knowledge
-> alembic_dimension_complete
```

当前优点：

- Phase 1-4 结构能力已经集中在 AlembicCore。
- Plugin 不做本地 AI，边界正确。
- Recipe 创建统一走 `RecipeProductionGateway`。
- `reasoning.sources` 已是必填并进入 source refs bridge。
- dimension complete 已有 progress、checkpoint、quality feedback、evidence hints。

当前缺口：

- Mission Briefing 是项目总览，不是可领取的 unit queue。
- `codeEntityGraph` / `callGraph` 在 briefing 里主要是数量摘要。
- evidence starters 是 hints，不是 assigned read set。
- response budget aggressive compression 会删除 `dimension.evidenceStarters` 和 AST class/protocol file path。
- submission tracker 只知道 dimension submissions，不知道 unit coverage。
- JobStore 只恢复 job/session，不恢复 analysis unit 进度。
- real Codex acceptance pack 已有 architecture Recipe loop，但还没有 before/after 指标验证。

这里 Codex 只是当前最成熟的第一条 adapter 验证路径；通用 contract 必须面向 IDE/host Agent，不得写成 Codex 私有能力。

## 推荐核心模型

### IDEAgentAnalysisPacket

由 AlembicCore 从 ProjectIntelligence 结果派生，不含源码全文，不替代现有 knowledge graph。

```ts
interface IDEAgentAnalysisPacket {
  packetId: string;
  projectRootHash: string;
  generatedAt: string;
  profile: 'cold-start' | 'rescan';
  projectSummary: {
    primaryLanguage: string;
    fileCount: number;
    targetCount: number;
    materialization: Record<string, boolean | string | number>;
    degraded: string[];
  };
  units: IDEAgentAnalysisUnit[];
  retrievalHints: {
    structureTools: string[];
    callContextAvailable: boolean;
    graphAvailable: boolean;
  };
  budget: {
    includedUnits: number;
    totalUnits: number;
    omittedReason?: string;
  };
}
```

### IDEAgentAnalysisUnit

最小执行单位。一个 unit 不要求直接等于一条 Recipe，但每条 accepted Recipe 应能回溯到至少一个 unit。

```ts
interface IDEAgentAnalysisUnit {
  unitId: string;
  dimensionId: string;
  targetName?: string;
  moduleName?: string;
  priority: number;
  reason: string;
  sourceRefs: Array<{
    path: string;
    line?: number;
    symbol?: string;
    role?: 'entry' | 'caller' | 'callee' | 'dependency' | 'guard' | 'example';
  }>;
  requiredReadSet: string[];
  structuralHints: {
    ast?: string[];
    dependencies?: Array<{ from: string; to: string; relation: string }>;
    callers?: string[];
    callees?: string[];
    dataFlowHints?: string[];
    guardFindings?: string[];
    panorama?: string[];
  };
  completionContract: {
    minDistinctFiles: number;
    mustReferenceAssignedSources: boolean;
    expectedEvidence: string[];
    allowNoRecipeWithReason?: boolean;
  };
  degraded?: Array<'ast-partial' | 'callgraph-partial' | 'depgraph-unavailable' | 'source-path-compressed'>;
}
```

### Unit Progress

用户已确认：第一版不提前锁死持久化策略，Stage 0 比较 BootstrapSession + checkpoint、JobStore 和轻量 ledger 后再决定。但概念上必须有：

```ts
interface ColdstartUnitProgress {
  unitId: string;
  status: 'pending' | 'claimed' | 'completed' | 'blocked' | 'rejected' | 'skipped';
  claimedAt?: string;
  completedAt?: string;
  submittedRecipeIds: string[];
  referencedFiles: string[];
  rejectedReasons: string[];
  deviationReason?: string;
}
```

可选承载位置：

- `BootstrapSession` 内存态 + dimension checkpoint。
- `JobStore.result` / host-agent recoverable job result。
- Recipe metadata / agentNotes / reasoning extension。
- 独立 lightweight unit ledger。

Design 推荐先做 Stage 0 只读 probe 后决定，不在需求设计里提前锁死存储位置；用户已确认这个取舍。

## 功能链路

### 1. Packet 生成

入口：

```text
ProjectIntelligenceCapability.run
-> ProjectIntelligenceResultProjection
-> IDEAgentAnalysisPacketBuilder
```

输入：

- `allFiles`
- targets / localPackageModules
- `astProjectSummary`
- `codeEntityResult`
- `callGraphAnalysis` / `callGraphResult`
- dependency graph
- panorama result
- guard audit
- active dimensions
- rescan evidencePlan（rescan profile 时）

输出：

- packet summary
- prioritized units
- degraded flags
- retrieval hints

原则：

- 不读写产品源码。
- 不包含源码全文。
- 不生成 Recipe。
- 不让 LLM 参与 deterministic packet projection。
- path 必须是项目相对路径。

### 2. Bootstrap 暴露

当前 `buildMissionBriefing` 可以继续作为总览。第一版推荐：

```text
Mission Briefing:
  - project summary
  - active dimensions
  - packet summary
  - next 3-5 units
  - retrieval / progress instruction
```

不要把所有 units 塞进 one-shot briefing。大项目下必须支持后续 retrieval 或至少保留 packet id。

用户已确认：Stage 0 先验证现有 `structure` / `graph` / `call_context` 能否支撑；如果不足，再新增 generic IDE Agent packet retrieval tool。Stage 0 需要比较两种实现路径：

| 路线 | 描述 | 优点 | 风险 |
| --- | --- | --- | --- |
| A. Extend existing tools | 在 Mission Briefing 放 unit summary，细节通过 `alembic_structure` / `alembic_graph` / `alembic_call_context` 查询 | 改动小，不新增 MCP tool | Agent 仍要自己编排查询；progress 弱 |
| B. Dedicated packet surface | 新增或扩展外部 workflow surface，支持领取 next units、记录 unit progress | 语义清晰，可恢复，可验收 | 改动较大，需要 schema 和测试 |

Design 推荐 B，但 Stage 0 允许先证明 A 是否足够；最终不得为了省 tool 让 Agent 继续低效自编排。

### 3. IDE Agent 执行

IDE Agent 的任务不再是“自己发现项目结构”，而是：

```text
领取 unit
-> 读 requiredReadSet / sourceRefs
-> 必要时查 graph/call_context
-> 判断是否形成 Recipe
-> submit_knowledge(reasoning.sources includes assigned source paths)
-> 记录 unit result
-> dimension_complete 汇总 unit coverage
```

Agent 可以跳过 unit，但必须给出 deviation reason，例如：

- assigned file is generated/test/vendor and not relevant
- call graph partial / source missing
- no stable project rule found after reading required files
- duplicate with accepted Recipe

### 4. Recipe 证据门

第一版 gate：

- `reasoning.sources` 必须非空，保持现有 UnifiedValidator 行为。
- `reasoning.sources` 必须使用完整相对路径。
- accepted Recipe 的 sources 必须与 assigned unit `sourceRefs` / `requiredReadSet` 有交集。
- `dimension_complete.referencedFiles` 必须覆盖 claimed/completed units 的主要 read set，或给出 deviation reason。
- duplicate / rejection 必须写入 unit progress，不能只在 submit 结果里丢失。

不建议第一版把“无 unitId 的 submit”硬拒绝，因为需要保持兼容；但应在 packet-guided profile 中给 warning / lower quality score。

### 5. Dimension Complete 汇总

`alembic_dimension_complete` 当前能恢复 submittedRecipeIds 和 referencedFiles。新需求要求它额外考虑：

- 本 dimension claimed units 数。
- completed / blocked / skipped units。
- accepted Recipe 覆盖的 unit 数。
- uncovered high-priority units。
- sourceRefs 覆盖比例。
- next dimension hints 是否来自已完成 units，而不是只来自 submissions。

第一版可以在返回的 `qualityFeedback` / `evidenceHints` 中加入 unit coverage summary，不必直接改变所有消费者。

### 6. Rescan 兼容

第一版 cold-start 优先，rescan 只做兼容设计：

```text
changed files / sourceRefs
-> affected units
-> verify-only units
-> gap-fill units
-> evolve / skip / deprecate evidence
```

后续 Stage 5 再把 rescan evidencePlan 与 unit ledger 合并：

- preserved Recipe 对应原 unit/sourceRefs。
- stale sourceRefs 生成 verify unit。
- changed file 影响 caller/callee/dependency neighborhood。
- no duplicate 仍以 trigger/title/coreCode + unit coverage 判定。

## 代码改动面建议

### AlembicCore

建议新增或扩展：

- `src/workflows/capabilities/project-intelligence/IDEAgentAnalysisPacketBuilder.ts`
- `src/workflows/capabilities/execution/external/IDEAgentAnalysisPacketContracts.ts`
- `ProjectIntelligenceResultProjection` 增加 sourceRefs/readSet/target module projection。
- `MissionBriefingBuilder` 支持 packet summary 注入，不再让 source path 只依赖 AST/evidenceStarters。
- `ExternalSubmissionTracker` 增加 optional unit progress 模型。
- `ExternalDimensionCompletionWorkflow` 的 quality/evidence hints 增加 unit coverage summary。

约束：

- Core 只做 deterministic projection 和 contract。
- Core 不依赖任一 IDE/host Agent 专有 UI。
- Core 不调用外部 AI。
- Core 不读取网络。
- Core 中新增内容必须优先复用现有通用 ProjectIntelligence / SourceRef / Recipe / rescan / JobStore 能力；若发现通用能力缺 contract，先补通用 contract，再接 Agent-facing projection。

### AlembicPlugin

建议新增或扩展：

- `ExternalColdStartWorkflow` 在 ProjectIntelligence 之后生成 packet 并放入 briefing/session。
- MCP handler surface 暴露 packet retrieval / progress。如果不新增 tool，应明确复用哪些现有 tool。
- `enhancedSubmitKnowledge` 支持 optional unitId / analysisUnitIds / evidence linkage。
- `dimension-complete` wrapper 保留 unit coverage 回填。
- `codex-acceptance-packs` 增加 before/after pack 或扩展现有 architecture pack。
- evidence checker 增加 assigned evidence coverage / structural relevance 指标。

说明：AlembicPlugin 当前承担 Codex adapter 和 MCP surface，所以第一版可先从 Codex acceptance pack 验证；但 contract 命名、字段和 Core 投影不能绑定 Codex。

约束：

- Plugin 不做结构分析复制。
- Plugin 不引入 internal AI。
- Plugin 不做 Dashboard UI。
- Plugin 不承担 Alembic daemon file monitor / internal job。
- Plugin adapter 只负责暴露、传递、记录和验证通用 packet / unit contract；不得在 Plugin 内另建 AST、sourceRefs、rescan plan 或 method-flow 系统。

### AlembicTest

只在 Stage 4 参与真实 IDE Agent validation：

- 准备同一 fixture / test project。
- baseline：当前 one-shot Mission Briefing path。
- after：packet-guided path。
- 收集 transcript、DB、sourceRefs、Recipe、rescan result、report。

不让 AlembicTest 修产品代码。

### Alembic / AlembicAgent / AlembicDashboard

第一版观察。

- Alembic daemon/internal AI 后续可复用 packet contract，但不作为第一阶段完成定义。
- AlembicAgent internal prompt/runtime 不在本需求第一版修改。
- Dashboard 不做 UI。

## 指标设计

第一版指标必须来自真实 before/after，不用主观“更好”。

| 指标 | 说明 | 目标 |
| --- | --- | --- |
| source path completeness | accepted Recipe 的 `reasoning.sources` 是否为完整相对路径且文件存在 | 不低于 baseline，目标 100% |
| assigned evidence coverage | accepted Recipe 是否覆盖 assigned unit sourceRefs/readSet | after 明显高于 baseline |
| structural relevance | Recipe 是否引用 AST / dependency / call/data-flow / guard / panorama 派生事实 | after 高于 baseline |
| duplicate rate | 同维度重复 title/trigger/coreCode/doClause | after 不高于 baseline |
| rejection explainability | V3/readiness/semantic review rejection 是否能映射到 unit/reason | after 可解释 |
| recovery fidelity | 中断/恢复后能否看到 remaining units、accepted recipes、blocked units | after 可复核 |
| agent tool burden | IDE/host Agent 为产出同等 Recipe 需要的探索性 tool calls | after 不高于 baseline，或增加有明确收益 |

不建议第一版追求：

- 全维度 Recipe 数最大化。
- 全局 data-flow 完整率。
- UI 展示指标。
- 大型真实项目性能指标。

## 验证设计

### Stage 0 Probe

只读验证问题：

1. `alembic_call_context` 输入 methodName 的 ID 合约是否稳定。
2. `CodeEntityGraph.getCallers/getCallees/getCallImpactRadius` 能否从 ProjectIntelligence materialized graph 查询到真实数据。
3. Mission Briefing compression 在大项目/小 budget 下会删除哪些路径证据。
4. Existing `recipe_source_refs` / `reasoning.sources` 是否足以承接 Recipe path evidence。
5. JobStore / bootstrap session 是否足以恢复 unit progress，或需要新 ledger。
6. Existing architecture acceptance pack 可否作为 baseline harness。
7. 哪些基础通用能力可直接复用，哪些需要先在通用模块补 contract，哪些才属于 IDE Agent adapter 层；输出“复用 / 补通用 / Agent-facing”三分表。

输出：

- 代码事实报告。
- minimal contract 决策。
- 不改产品代码。

### Stage 4 Real IDE Agent Before/After

用户已确认按推荐维度推进：

- baseline smoke：`architecture`
- formal before/after：`event-and-data-flow`

执行：

```text
Run A: current Mission Briefing path
Run B: packet-guided path
Same fixture/project
Same dimension
Same target completion definition
Collect transcript + DB + Recipe + sourceRefs + report
```

第一条真实 adapter 使用 Codex，因为当前 AlembicPlugin 已有 Codex MCP tools 和 architecture acceptance pack；后续可以用同一 packet contract 扩展到其它 IDE Agent。

通过条件：

- after 在 primary 指标不回退。
- after 至少在 assigned evidence coverage 或 structural relevance 上改善。
- source path completeness 保持硬门。
- no duplicate 不回退。
- report 可由总控复核。

失败条件：

- after 丢 sourceRefs。
- after 无法恢复 remaining units。
- after 通过更多 mock/simulator 才能成立。
- after 需要 UI 或 internal AI 才能成立。

## 分阶段任务建议

| 阶段 | 推荐归口 | 任务 | 输出 |
| --- | --- | --- | --- |
| Stage 0 | AlembicCore + AlembicPlugin | 只读 contract probe：callContext、graph query、briefing compression、sourceRefs、JobStore、acceptance pack。 | 代码事实报告和 implementation contract。 |
| Stage 1 | AlembicCore | 定义 `IDEAgentAnalysisPacket` / `IDEAgentAnalysisUnit` contract 和 projection builder。 | Core unit tests + fixture packet。 |
| Stage 2 | AlembicPlugin | 将 packet summary / next units 接入 external bootstrap；设计 retrieval/progress surface。 | MCP tests + bootstrap response snapshot。 |
| Stage 3 | AlembicCore + AlembicPlugin | submit/dimension_complete unit evidence linkage 和 quality feedback。 | Unit coverage tests + sourceRefs tests。 |
| Stage 4 | AlembicTest | 真实 IDE Agent before/after。 | transcript/report/DB/sourceRefs evidence。 |
| Stage 5 | AlembicCore + AlembicPlugin | rescan affected unit / verify-only / gap-fill 兼容。 | rescan unit evidence tests。 |

用户已确认第一版优先 cold-start packet + unit evidence；rescan 在第一版只做兼容设计，完整 affected units / verify-only / gap-fill 放 Stage 5。

## 已排除范围

- Dashboard UI。
- Alembic internal AI job 重写。
- AlembicAgent prompt/runtime 第一版改造。
- Plugin 复制 AST / call graph / dependency graph。
- 全局 data-flow 完整求解。
- 多 LLM dispatch。
- VAD / automation delivery。
- 真实用户项目默认扫描。
- Product source direct implementation by Design window。

## 总控接收建议

建议总控将本需求作为 `ready-for-workspace` 接收，但不要直接派实现窗口。

推荐总控第一步：

```text
接收 IDE-AGENT-COLDSTART-OPTIMIZATION-2026-05-31
-> 创建 Stage 0 read-only contract probe
-> 目标窗口 AlembicCore + AlembicPlugin
-> 只读检查，不改代码
-> 回填 implementation contract 和 first validation dimension
```

## 用户已确认的默认取舍

- 第一条真实验证 adapter：Codex first adapter；能力本身保持 IDE Agent 通用。
- 验证维度：Stage 0 / smoke 用 `architecture`，正式 before/after 用 `event-and-data-flow`。
- Unit progress：Stage 0 比较 BootstrapSession + checkpoint、JobStore、轻量 ledger，不提前锁死。
- Packet retrieval surface：Stage 0 先验证现有 `structure` / `graph` / `call_context`；不足时新增 generic IDE Agent packet retrieval tool。
- Rescan：第一版 cold-start packet + unit evidence 优先，rescan affected units 后置到 Stage 5。

## 仍需总控排期裁决

该需求排期是在 038/039 之后，还是作为 Plugin cold-start 后续能力优先插入，由总控根据当前主线裁决。

## 证据状态

| 类型 | 状态 | 说明 |
| --- | --- | --- |
| 用户确认 | 已确认 | 用户要求升级为正式需求，并要求 Design 深挖真实代码逻辑；用户已确认 IDE Agent 通用能力边界、通用底座复用原则和默认执行取舍。 |
| 本地代码事实 | 已只读核对 | 已核对 Core ProjectIntelligence、CallGraph、MissionBriefing、EvidenceStarter、SubmissionTracker、BootstrapSession、Plugin cold-start/rescan/submit/dimension complete、JobStore、acceptance pack、sourceRefs。 |
| 外部调研 | 已核对 | 已查阅 Understand-Anything、Sourcegraph Cody context、CodeQL data-flow、OpenAI Codex docs。 |
| 测试证据 | 待 Stage 4 | 当前只做需求设计，不跑真实 IDE Agent before/after；第一条 adapter 可用 Codex。 |
| 实现证据 | 无 | Design 窗口未改产品源码。 |

## 交接结论

本需求可以交给 AlembicWorkspace 接收为正式需求。建议总控先做 Stage 0 只读 contract probe，确认最小改动面、通用能力复用边界、unit progress 存储策略和 retrieval surface，再派 AlembicCore / AlembicPlugin 实现。
