# Codex Agent Coldstart Optimization Workspace Signal

Design Key：CODEX-AGENT-COLDSTART-OPTIMIZATION-2026-05-31

日期：2026-05-31
状态：已升级并纠偏为 IDE Agent 通用正式需求 / see formal plan
来源窗口：AlembicDesign
接收窗口：AlembicWorkspace

## Signal 类型

- research
- requirement-candidate
- upgraded-to-formal-requirement

## 正式需求文档

- 原始计划：[ide-agent-coldstart-optimization-original-plan-2026-05-31.md](ide-agent-coldstart-optimization-original-plan-2026-05-31.md)
- 需求设计：[ide-agent-coldstart-optimization-requirement-design-2026-05-31.md](ide-agent-coldstart-optimization-requirement-design-2026-05-31.md)

补充裁决：用户已明确冷启动和增量扫描等功能不是只针对 Codex，而是 IDE Agent 通用能力。Codex 仅作为当前 AlembicPlugin 中第一条可验证 adapter / acceptance path。

## 触发内容

```text
现在总控已经完成了 Plugin 的冷启动和增量链路和测试，我需要你基于 Alembic 项目的基础能力 AST 模块依赖 方法流入流出等能力，联网查找业界最佳实践，思考使用 codex agent 进行冷启动的方式，存在哪些问题，哪些需要优化。

你主要处理 Codex Agent 路径。

这是给你参考的一个项目：Lum1104/Understand-Anything。
```

## Design 判断

- 为什么属于该类型：用户要求基于本地代码事实和外部最佳实践，分析 Codex Agent 冷启动路径的问题和优化方向；这不是立即实现，也不是总控验收，而是新需求候选的前置调研。
- 是否影响 AlembicWorkspace 当前主线：不直接打断当前主线。它与 Plugin cold-start/rescan 测试优化完成后的后续能力路线相关，可作为下一阶段候选。
- 是否建议打断当前主线：否。除非总控判断当前主线下一步正要继续优化 cold-start 质量，否则建议作为后续需求候选接收。
- 是否需要用户继续确认：需要。当前结论可以进入总控评审，但是否升级为正式需求、是否优先于 038/039 或 PCVM 后续，由用户 / 总控确认。

## 核心判断

Alembic 现有冷启动不是缺 AST、依赖、调用链或数据流基础能力；真实短板在 Codex Agent 路径的“执行型投喂”不够强。

当前 `alembic_bootstrap` 外部路径会同步跑 Phase 1-4，构建一次性 Mission Briefing，然后等待 Codex host agent 自己阅读代码、提交 Recipe、完成维度。这个模式可用，但对大项目和高质量 Recipe 生产来说，过度依赖 Agent 自行决定读哪些文件、如何覆盖调用/依赖关系、如何证明 sourceRefs，与“结构事实已经在 AlembicCore 里”的能力不匹配。

推荐方向是：**Alembic 负责 deterministic structure planning；Codex Agent 负责 semantic judgement and recipe writing**。

也就是说，不应让 Codex Agent 重新发现项目结构；应先由 AlembicCore 把 AST、Dependency Graph、Code Entity Graph、CallGraph/DataFlow、Panorama 和 Guard 事实投影成可审计的 `ColdstartAnalysisUnit` / `CodexAnalysisPacket`，再让 Codex 按小单元阅读源码、判断规则、提交 Recipe。Alembic 侧保留 sourceRefs、结构依据、提交证据、质量门和恢复状态。

## 外部调研结论

### Understand-Anything 参考价值

参考项目：[Lum1104/Understand-Anything](https://github.com/Lum1104/Understand-Anything)

公开 README 说明它通过 multi-agent pipeline 扫描项目，抽取 files / functions / classes / dependencies，并把知识图谱保存为 `.understand-anything/knowledge-graph.json`；后续支持 dashboard、chat、diff、explain、onboard、domain 等使用路径。

项目说明还强调：

- 中间结果写入 `.understand-anything/intermediate/`，不是全部塞回上下文。
- Agent pipeline 包含 project-scanner、file-analyzer、architecture-analyzer、tour-builder、graph-reviewer 等职责拆分。
- `knowledge-graph.json` 是可分享、可复用的产物；增量更新可以通过 commit hook 或重跑触发。

对 Alembic 的启发不是复制 UI 或静态图谱，而是保留这个思路：

```text
deterministic scan / structure graph
-> bounded analysis units
-> agent reads source and writes semantic findings
-> persisted graph / recipe evidence
-> incremental update reuses prior structure
```

这与 Alembic 现有 ProjectIntelligence 更适配：Alembic 已经能产出结构事实，缺的是把结构事实切成 Codex 可执行、可验收、可恢复的工作包。

### Sourcegraph / code intelligence 参考价值

Sourcegraph Cody docs 将 codebase-aware 的关键定义为 context retrieval：keyword search、Sourcegraph search、code graph 三者组合，code graph 负责基于代码元素关系找上下文。

这支持 Alembic 不只依赖 semantic / keyword 搜索，而要把结构图关系作为 Codex 冷启动上下文的第一等输入。对本需求来说，最佳实践不是“给 Agent 更多文本”，而是“给 Agent 更准确的结构化上下文入口”。

### CodeQL / data-flow 参考价值

CodeQL 官方文档区分 local data flow 和 global data flow：local 更快、更容易用但不完整；global 更强但更慢、更容易有噪声，并通常需要明确 source/sink 配置。

这对 Alembic 的启发是：冷启动第一版不应让 Codex 或 Core 试图全局解释所有方法流入流出；应把 call/data-flow 投影为局部、可验证的分析单元，例如：

- changed / important file 的 caller/callee neighborhood。
- service entry -> downstream dependency 的一跳或两跳流。
- dimension-specific source/sink，例如 event handler、API handler、persistence adapter。

### Codex 官方路径参考价值

OpenAI Codex 官方 docs 表明 Codex 可以运行在 sandbox / approval 边界内，`workspace-write` 需要网络或越界访问时会走审批；`codex exec` 支持 non-interactive 脚本 / CI、JSONL event stream、structured output schema 和 session resume。

这意味着 Alembic 设计 Codex Agent 冷启动时不应假设 Agent 是确定性函数调用。更稳的边界是：

- AlembicCore / AlembicPlugin 生成可审计 packet 和质量 gate。
- Codex Agent 作为真实 host agent 执行阅读、判断、提交。
- 接受/拒绝、progress、quality、sourceRefs 和恢复状态由 Alembic 记录。
- 真实验证使用 Codex Agent，不再使用 mock / simulator 伪装 agent 行为。

## 已知代码事实

### AlembicCore

- `ProjectIntelligenceRunner` 是 cold-start 和 rescan 共享 Phase 1-4 管线，已经包含文件收集、AST、Code Entity Graph、Dependency Graph、Module Entities、Panorama、Guard、Dimension resolve。
- `CallGraphAnalyzer` 已有调用图 / 数据流边管线：CallSiteExtractor、SymbolTableBuilder、ImportPathResolver、CallEdgeResolver、DataFlowInferrer，并支持增量、超时和 partial result。
- `MissionBriefingBuilder` 已把 AST、dependencyGraph、guardFindings、targets、dimensions、panorama、mustCoverModules 和 session 组织进外部 Mission Briefing。
- `EvidenceStarterBuilder` 已能按维度从 AST、Guard、Dependency Graph、CallGraph、Panorama 生成 evidence starter。
- `ExternalSubmissionTracker` 已能按维度追踪 submissions、source files、negative signals、quality report 和 accumulated evidence。

### AlembicPlugin

- `ExternalColdStartWorkflow` 明确是外部 Agent 驱动冷启动：同步执行 Phase 1-4、构建 Mission Briefing、不启动本地 AI pipeline。
- `alembic_submit_knowledge` 统一走 `RecipeProductionGateway.create()`，并把成功 / 拒绝提交写入 bootstrap session tracker。
- `alembic_dimension_complete` 能恢复 referencedFiles / submittedRecipeIds，绑定 Recipe，创建 skill，持久化 checkpoint/key findings，并返回 progress、qualityFeedback、evidenceHints。

## 当前问题

| 问题 | 当前表现 | 风险 |
| --- | --- | --- |
| Mission Briefing 是总览，不是执行型计划 | Agent 获得维度和摘要，但没有稳定的 per-unit read set / call neighborhood / expected evidence contract | Codex 可能读偏、读少、重复读，Recipe 质量依赖提示词和模型运气 |
| 结构图被压缩成 summary | `codeEntityGraph` / `callGraph` 在 briefing 里主要是数量摘要；AST 和 evidenceStarters 在超预算时会继续删减 | 方法流入流出、模块边界和 source path 证据无法稳定驱动 Agent 行动 |
| sourceRefs 仍偏提交后检查 | 提交 schema 强调路径，但路径是否覆盖 assigned structural evidence 不是 packet-level gate | 可能继续出现 sourceRefs 可读但不代表覆盖正确结构单元 |
| 质量评分偏“数量 + 文本结构” | coverage / evidence / diversity / coherence 主要看 submission count、file count、content length、confidence 等 | 可能高分通过但没有覆盖关键模块、调用链或变化影响面 |
| cross-dimension evidence 是被动返回 | `dimension_complete` 可返回 accumulated evidence/hints，但没有强制成为下一维度的 work queue | 维度之间仍可能重复发现、重复提交或丢失前序边界 |
| session 恢复粒度偏会话级 | active bootstrap session TTL 2 小时；Codex 真实长线程/中断/恢复需要更强的 packet progress ledger | 长任务中断后难以恢复到“下一个待分析单元” |
| 一次性 briefing 承担太多 | 100KB 上限正确，但会让大项目的执行细节被压缩掉 | 大项目更需要按需查询 / 分步领取，而不是更长 briefing |

## 建议优化方向

### 1. CodexAnalysisPacket / ColdstartAnalysisUnit

新增从 ProjectIntelligence 派生的 Codex 专用执行包，不包含源码全文，只包含结构化工作单元。

建议字段：

```ts
interface ColdstartAnalysisUnit {
  unitId: string;
  dimensionId: string;
  priority: number;
  reason: string;
  sourceRefs: Array<{ path: string; line?: number; symbol?: string }>;
  symbols: Array<{ name: string; kind: string; path: string; line?: number }>;
  dependencyNeighborhood: {
    incoming: string[];
    outgoing: string[];
  };
  callNeighborhood?: {
    callers: string[];
    callees: string[];
    dataFlowHints: string[];
  };
  requiredReadSet: string[];
  expectedEvidence: string[];
  completionContract: {
    minRecipes?: number;
    minDistinctFiles: number;
    mustReferenceAssignedSources: boolean;
  };
}
```

关键原则：

- source path 是关键证据，不能因为压缩丢失。
- 每个 unit 必须可被 Codex 独立执行和 Alembic 独立验收。
- unit 不是新知识库，只是 ProjectIntelligence 的派生执行视图。

### 2. 从 one-shot briefing 改为 guided retrieval loop

保留当前 Mission Briefing 的项目总览，但不要让它承载所有执行细节。建议第一版只在 briefing 里放：

- analysisPacket summary。
- next 3-5 个最高优先级 units。
- tool hint：如何领取 unit / 查询 graph neighborhood / 完成 unit。
- progress ledger id。

后续通过 MCP 工具或现有结构工具扩展，支持 Codex 领取下一批分析单元，而不是一次性把全项目结构塞进 response。

### 3. 把 evidence gate 从 dimension-level 升到 unit-level

`alembic_submit_knowledge` 和 `alembic_dimension_complete` 之外，需要让 tracker 知道“这条 Recipe 对应哪个 analysis unit”。

建议 gate：

- Recipe 的 `reasoning.sources` 必须与 unit.requiredReadSet 或 unit.sourceRefs 有交集。
- `referencedFiles` 必须覆盖本 dimension 已领取 unit 的 read set，或给出 deviation reason。
- 同一 unit 不得重复提交同一 trigger / title / doClause 语义。
- 每个 accepted Recipe 必须可回溯到 unitId、dimensionId、sourceRefs、structural evidence。

### 4. 首批单维度验证选择 event-and-data-flow

第一版不建议全维度铺开。推荐先做单维度真实验证：

- 维度：`event-and-data-flow`
- 原因：最能验证用户关心的“方法流入流出”、调用图、数据流 hint 是否真的帮助 Codex 产出更准的 Recipe。
- 测试方式：真实 Codex Agent，不使用 `CodexScenarioAgentSimulator`。
- 对比：同一 fixture / project 下，before 是当前 Mission Briefing 路径，after 是 packet-guided path。

### 5. 指标用真实 before/after，不用主观“更好”

建议最小指标：

| 指标 | 目标 |
| --- | --- |
| source path completeness | accepted Recipe `reasoning.sources` 全为项目相对路径，且可解析 |
| assigned evidence coverage | accepted Recipe 覆盖 assigned unit sourceRefs/readSet |
| structural relevance | Recipe 至少引用一个 call/dependency/AST 派生结构事实 |
| duplicate rate | 同维度重复 title/trigger/doClause 降低 |
| rejection rate | V3 字段拒绝、readiness 拒绝和 semantic review 数量可解释 |
| recovery fidelity | 中断后能显示 remaining units / accepted recipes / rejected reasons |

## 影响范围建议

| 窗口 / 仓库 | 建议状态 | 理由 |
| --- | --- | --- |
| AlembicCore | 参与 | 负责 deterministic ProjectIntelligence -> CodexAnalysisPacket / ColdstartAnalysisUnit projection；不能放到 Plugin 里重复实现结构分析。 |
| AlembicPlugin | 参与 | Codex host-agent MCP 入口，负责外部 bootstrap briefing、unit retrieval、submit/dimension_complete gate surface。 |
| Alembic | 观察 / 后续参与 | daemon/internal AI 路径后续可复用 packet，但本需求第一版聚焦 Codex Agent path，不应先改 internal AI。 |
| AlembicAgent | 观察 | 本需求不处理 Alembic internal Agent prompt/runtime；只在后续若复用 packet 到 internal path 时参与。 |
| AlembicDashboard | 无任务 | 用户当前要求处理 Codex Agent 路径，不做 UI。 |
| AlembicTest | 参与验证 | 最终需要真实 Codex Agent 测试单，单维度 before/after 验证。 |
| AlembicDesign | 设计完成 / 待用户确认 | 当前完成 research + requirement-candidate signal；若用户要求，可继续升级为 original-plan / requirement-design。 |

## 证据状态

- 用户描述：已明确聚焦 Codex Agent path，参考 Understand-Anything。
- 截图 / 日志：无。
- 已知代码事实：已只读核对 AlembicCore / AlembicPlugin 关键链路。
- 外部调研：已查阅 Understand-Anything README / CLAUDE.md、Sourcegraph Cody context docs、CodeQL data-flow docs、OpenAI Codex docs。
- 待补代码事实：
  - 现有 `alembic_structure` / `alembic_graph` / `callContext` 工具能否复用为 unit retrieval，不新增工具是否可行。
  - `ProjectIntelligenceResultProjection` 是否已有足够 target/module/keyFiles 投影，能否作为第一版 packet 输入。
  - 外部 session progress 是否已通过 JobStore / embedded recoverable job 持久化到足够粒度。
- 测试 / 复现需求：需要真实 Codex Agent 单维度 before/after，不跑 full cold-start。

## 建议给总控的下一步

- 作为 `requirement-candidate` 接收，不直接派实现。
- 若用户确认升级，先开启原始计划确认：`CODEX-AGENT-COLDSTART-OPTIMIZATION-2026-05-31`。
- Stage 0 建议只读调研：确认现有 structure/graph/callContext/query 能否承接 `ColdstartAnalysisUnit` retrieval，列出最小改动面。
- Stage 1 才考虑 AlembicCore packet projection + AlembicPlugin external bootstrap surface。
- Stage 2 再由 AlembicTest 用真实 Codex Agent 做单维度 before/after。

说明：本建议只给 `AlembicWorkspace` 评审，不是执行窗口提示词。

## TODO / Backlog 建议

| ID | 类型 | 优先级建议 | 推荐归口 | 事项 | 依赖 / 触发 |
| --- | --- | --- | --- | --- | --- |
| CODEX-COLDSTART-TODO-1 | research | P1 | AlembicCore / AlembicPlugin | 只读核对现有 structure / graph / callContext 是否足以支持 packet retrieval | 用户确认升级为正式需求 |
| CODEX-COLDSTART-TODO-2 | requirement | P1 | AlembicCore | 设计 `ColdstartAnalysisUnit` projection，保留 sourceRefs / method in-out / dependency neighborhood | Stage 0 调研完成 |
| CODEX-COLDSTART-TODO-3 | requirement | P1 | AlembicPlugin | 外部 Mission Briefing 改为 summary + next units + progress ledger，不把所有细节塞入 one-shot response | Core packet contract 确认 |
| CODEX-COLDSTART-TODO-4 | validation | P1 | AlembicTest | 真实 Codex Agent 单维度 before/after 验证，首选 `event-and-data-flow` | 产品实现完成 |

## 开放问题

1. 是否将该候选升级为正式需求，并排在 038/039 之前、之后，还是作为 Plugin cold-start/rescan 后续优化队列？
2. 第一版验证维度是否按 Design 推荐选择 `event-and-data-flow`？
3. Stage 0 是否允许总控只读运行现有 structure/graph/callContext probe，以确认无需新增重复工具？
4. before/after 指标是否沿用 PCVM 风格形成节点级可复验报告，还是先采用 lightweight acceptance pack？

## 交接前自检

- 已对照 `docs/workspace-alignment-checklist.md`：是。
- 本 signal 没有修改 workspace 当前状态或全局 TODO：是。
- 本 signal 没有包含可复制实现窗口提示词：是。
- 推荐归口只是建议，不是派发：是。
- 如处于 detached-design-mode，已标注需要总控导入复核：不适用，已读取父级 workspace 文档。
