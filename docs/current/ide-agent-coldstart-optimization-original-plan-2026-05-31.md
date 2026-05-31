# IDE Agent Coldstart Optimization 原始计划

Design Key：IDE-AGENT-COLDSTART-OPTIMIZATION-2026-05-31
日期：2026-05-31
状态：ready-for-workspace / formal-requirement
维护窗口：AlembicDesign
总控：AlembicWorkspace
目标仓库：AlembicCore / AlembicPlugin

## 判断类型

- `new-requirement`
- `research`
- `requirement-design`
- `ide-agent-path`
- `cold-start-quality`
- `host-agent-workflow`

这是一个已经由用户确认升级的正式需求设计，不是实现派发、不是测试验收，也不是总控当前状态更新。

## 用户原始目标

用户要求在 Plugin cold-start / rescan 链路测试完成后，基于 Alembic 项目已有的 AST、模块依赖、方法流入流出等基础能力，结合业界最佳实践，重点处理 IDE Agent 路径的冷启动方式：

```text
使用 IDE Agent 做 cold-start 时，
不要只把 Mission Briefing 丢给 Agent 让它自行探索；
应该利用 Alembic 已有结构能力，让 IDE/host Agent 更准确地阅读、判断、产出 Recipe，
并最终用真实 IDE Agent 测试验证优化效果。
```

用户进一步确认：

- 本需求需要升级为正式需求。
- 总控在忙时，AlembicDesign 可以直接深挖真实代码逻辑，完成需求设计。
- 主要处理 IDE Agent 路径；Codex 只是当前 AlembicPlugin 里最先具备工具链和真实验收包的 adapter。
- 参考项目包括 `Lum1104/Understand-Anything`。
- Recipe 的源文件路径仍然是关键证据，不能丢失。
- 链路较长，应拆成合理多个阶段慢慢执行，最后使用真实 test 验证。

## 真实问题

当前 Alembic 不是缺少结构分析能力，而是 IDE Agent 冷启动路径没有把这些结构事实变成“执行型上下文”。

当前外部 cold-start 路径大致是：

```text
alembic_bootstrap
-> ProjectIntelligence Phase 1-4
-> Mission Briefing
-> IDE/host agent 自行阅读代码
-> alembic_submit_knowledge
-> alembic_dimension_complete
```

这条链路能工作，但它把过多判断压力交给 IDE Agent：

- Agent 要自己决定读哪些文件。
- Agent 要自己判断哪些 AST / dependency / call graph 事实重要。
- Agent 要自己把 sourceRefs 写对。
- Agent 要自己避免重复、漏读、读偏。
- Alembic 只在提交后做字段和质量检查，较少在提交前给出结构化任务单。

因此，本需求的目标不是“让某个具体 IDE Agent 变聪明一点”，而是让 Alembic 把确定性结构事实投影成 IDE/host Agent 可执行、可验证、可恢复的分析单元。

## 核心原则

推荐边界：

```text
AlembicCore 负责 deterministic structure planning。
AlembicPlugin 负责 IDE/host-agent MCP surface 和 session/progress 衔接。
IDE Agent 负责 semantic judgement、真实读源码、撰写 Recipe。
AlembicTest 负责真实 IDE Agent before/after 验证。
```

必须避免：

- 不把结构分析逻辑复制到 Plugin。
- 不让 Plugin 做 internal AI。
- 不把 source path 从证据链中降级。
- 不把 UI / Dashboard 纳入第一版。
- 不用 simulator / mock 代替真实 IDE Agent 验收。
- 不把全局 data-flow 完整求解作为第一阶段目标。
- 基础通用的非 Agent 能力能复用就复用；AST、依赖图、方法流入流出、sourceRefs、Recipe persistence、rescan planner、JobStore / checkpoint 等底座能力不应被改造成 IDE Agent 私有实现。
- 如果基础通用能力本身有缺口，优先在通用模块补齐契约，再由 IDE Agent projection / adapter 消费；不要在 Agent 层做第二套同类能力。

## 已深挖代码事实

### AlembicCore 已有能力

- `AlembicCore/src/workflows/capabilities/project-intelligence/ProjectIntelligenceRunner.ts` 是 cold-start / rescan 共享 Phase 1-4 管线，包含文件收集、AST、Code Entity Graph、Call Graph、Dependency Graph、Module Entities、Panorama、Guard 和 Dimension resolve。
- `AlembicCore/src/core/analysis/CallGraphAnalyzer.ts` 已支持调用图和数据流边生成，包含 full / incremental、超时 partial、方法调用和 DataFlowInferrer。
- `AlembicCore/src/service/knowledge/CodeEntityGraph.ts` 已将 method entities、`calls` edge、`data_flow` edge 落到图谱，并提供 callers / callees / impact radius 查询。
- `AlembicCore/src/workflows/capabilities/execution/external/MissionBriefingBuilder.ts` 已构建外部 Agent Mission Briefing，包含 AST、dependencyGraph、guardFindings、targets、dimensions、panorama、local packages 和 session。
- `AlembicCore/src/workflows/capabilities/execution/external/EvidenceStarterBuilder.ts` 已能按维度生成 evidence starters，包括 AST pattern、Guard、Dependency、CallGraph、Panorama hints。
- `AlembicCore/src/workflows/capabilities/execution/external/ExternalSubmissionTracker.ts` 已按维度追踪 submissions、sources、negative signals、quality report 和 accumulated evidence。
- `AlembicCore/src/workflows/capabilities/execution/external/BootstrapSession.ts` 已有 active session、dimension progress、snapshot cache、submission tracker、cross-dimension hints 和 TTL。

### AlembicPlugin 已有能力

- `AlembicPlugin/lib/external/mcp/handlers/bootstrap/ExternalColdStartWorkflow.ts` 执行 external cold-start：full reset、ProjectIntelligence Phase 1-4、Mission Briefing，不启动本地 AI pipeline。
- `AlembicPlugin/lib/external/mcp/handlers/rescan/ExternalKnowledgeRescanWorkflow.ts` 执行 external rescan：Recipe snapshot、KnowledgeSync、Phase 1-4、relevance audit、rescan plan、prescreen、evidencePlan、Mission Briefing。
- `AlembicPlugin/lib/external/mcp/handlers/consolidated.ts` 的 `alembic_submit_knowledge` 走 `RecipeProductionGateway.create()`，并把成功 / 拒绝提交写入 bootstrap session tracker。
- `AlembicPlugin/lib/external/mcp/handlers/dimension-complete/ExternalDimensionCompletionWorkflow.ts` 能恢复 referencedFiles / submittedRecipeIds，绑定 Recipe，持久化 checkpoint/key findings，并返回 progress、qualityFeedback、evidenceHints。
- `AlembicPlugin/lib/external/mcp/handlers/structure.ts` 已有 `alembic_structure`、`alembic_graph`、`alembic_call_context` 的 low-level query surface。
- `AlembicPlugin/lib/daemon/DaemonJobRunner.ts` 和 `AlembicCore/src/daemon/JobStore.ts` 已能把 host-agent bootstrap/rescan 作为 recoverable job 保存，但粒度主要是 job/session，不是 analysis unit。
- `AlembicPlugin/test/codex-acceptance-packs/architecture-recipe-loop/README.md` 已明确 real Codex thread acceptance pack，不再依赖 retired scenario simulator。
- `AlembicPlugin/scripts/lib/codex-recipe-loop-evidence-checker.mjs` 已能检查 tool order、Recipe persistence、source refs、rescan evidence surface 和 duplicate architecture recipe。

这些 Plugin 事实说明 Codex 是当前第一条可执行验证路径；它不代表能力边界只能服务 Codex。

### 关键证据链已存在

- `UnifiedValidator` 强制 `reasoning.sources` 非空，并提示完整相对路径。
- `RecipeProductionGateway` 将 `reasoning.sources` / `sourceRefs` 带入 Recipe 创建数据。
- `KnowledgeModule` 在 `knowledge:changed` 后把 `knowledge_entries.reasoning.sources` 填充到 `recipe_source_refs`。
- `SourceRefReconciler` 会从 `knowledge_entries.reasoning.sources` 重建 source refs，检测 missing / stale / renamed。
- Rescan evidence plan 会带 `allRecipes.sourceRefs` 和 audit hints。

这说明本需求不应新建一套证据系统，而应把现有 sourceRefs / Recipe / session / graph 能力前移成 IDE Agent 冷启动执行契约。

## 外部调研结论

### Understand-Anything

参考：[Lum1104/Understand-Anything](https://github.com/Lum1104/Understand-Anything)

公开 README 描述的关键模式：

- 用 deterministic parser / tree-sitter 提取 imports、exports、functions、classes、call sites、inheritance 等结构事实。
- LLM 负责语义摘要、tag、架构层、domain mapping、guided tours。
- multi-agent pipeline 包含 project-scanner、file-analyzer、architecture-analyzer、tour-builder、graph-reviewer。
- file analyzers 分批并行执行，支持增量更新。
- 结构图和中间产物持久化，而不是把所有内容塞进一次上下文。

对 Alembic 的启发：不要复制 UI 或整套图谱格式，而是采用类似分工：

```text
deterministic scan
-> bounded analysis units
-> host agent reads source
-> semantic Recipe
-> persisted evidence
-> incremental reuse
```

### Sourcegraph Cody context

参考：[Sourcegraph Cody Context](https://sourcegraph.com/docs/cody/core-concepts/context)

Sourcegraph 将 codebase context 拆成 keyword search、Sourcegraph search 和 code graph，code graph 用代码元素关系寻找上下文。这支持 Alembic 把结构图关系作为 IDE Agent cold-start 的一等输入，而不是只依赖长 prompt 或关键词搜索。

### CodeQL data-flow

参考：[CodeQL JavaScript/TypeScript data-flow docs](https://codeql.github.com/docs/codeql-language-guides/analyzing-data-flow-in-javascript-and-typescript/)

CodeQL 区分 local data flow 和 global data flow：local 更快但不完整；global 更强但更慢且更易产生噪声，通常需要明确 source/sink 配置。

对本需求的结论：第一版应做 bounded call/data-flow hints，不做全局数据流完整求解。适合先从关键 entry、changed files、high fan-in/out methods、event handler、service boundary 等局部单元开始。

### OpenAI Codex

参考：[Codex non-interactive mode](https://developers.openai.com/codex/noninteractive)、[Codex approvals & security](https://developers.openai.com/codex/agent-approvals-security)

Codex 支持 non-interactive、session resume、workspace sandbox 和 approval 边界。这里的价值是提供第一条真实 adapter 的约束样本：Alembic 不能假设任何 IDE/host Agent 是确定性函数；应把它们都看作真实 host agent：

- 输入要可审计。
- 进度要可恢复。
- 输出要由 Alembic gate。
- 验收要跑真实 IDE/host Agent thread / agent，而不是 mock；第一条验收可以使用 Codex。

## 最终目标

建立 IDE Agent 冷启动执行包能力，让 Alembic 通过已有结构事实指导 IDE Agent 产出更准确、更可验证、更可恢复的 Recipe。

最终闭环：

```text
ProjectIntelligence Phase 1-4
-> IDEAgentAnalysisPacket
-> IDEAgentAnalysisUnit queue
-> IDE Agent reads assigned source refs
-> submit_knowledge links Recipe to unit / source refs / structural evidence
-> dimension_complete records unit coverage and progress
-> rescan reuses Recipe/sourceRefs/unit evidence
-> real IDE Agent before/after validation
```

这条链路只新增 Agent-facing projection / packet / unit / evidence linkage；底层 ProjectIntelligence、CodeEntityGraph、CallGraph、SourceRef、Recipe、rescan 和 job recovery 能力应保持通用，可被 Codex、其它 IDE Agent、Alembic daemon/internal path 和后续自动化复用。

## 第一版完成定义

- `alembic_bootstrap` 不再只返回一份 one-shot Mission Briefing；它能提供 IDE Agent 可执行的 `IDEAgentAnalysisPacket` 摘要和首批 analysis units，或提供等价 retrieval surface。
- 每个 `IDEAgentAnalysisUnit` 至少包含：`unitId`、`dimensionId`、priority、reason、sourceRefs/readSet、target/module、structural hints、expected evidence、degraded flags。
- Source path 是硬证据：unit、Recipe、dimension_complete、rescan evidence 中必须保留项目相对路径，不得因压缩丢失。
- `alembic_submit_knowledge` / `alembic_dimension_complete` 能把 accepted Recipe 关联到 dimension + analysis unit + sourceRefs，或在 Stage 0 明确选择兼容字段承载。
- Session / job status 能恢复到 remaining units / completed units / accepted recipes / rejected reasons，而不是只恢复 job/session。
- 真实 IDE Agent 单维 before/after 测试证明新路径至少不回退，并在 source path completeness、assigned evidence coverage、structural relevance 或 duplicate/rejection 解释性上改善。
- 不引入 UI，不引入 internal AI rewrite，不复制结构分析到 Plugin。

## 建议阶段

| 阶段 | 名称 | 目标 | 完成条件 |
| --- | --- | --- | --- |
| Stage 0 | Read-only contract probe | 只读确认现有 structure / graph / call_context / JobStore / sourceRefs 能力能否支撑 analysis unit。 | 形成代码事实报告和最小 contract 决策，不改产品代码。 |
| Stage 1 | Core packet projection | 在 AlembicCore 中定义 deterministic `IDEAgentAnalysisPacket` / `IDEAgentAnalysisUnit` projection。 | Packet 从 ProjectIntelligence 结果生成，不含源码全文，保留 sourceRefs/readSet 和 degraded flags。 |
| Stage 2 | Plugin host-agent surface | AlembicPlugin 在 external bootstrap 中暴露 packet summary、首批 units 和 retrieval/progress surface。 | IDE Agent 能领取/查看 units，不需要读完整结构图 prompt。 |
| Stage 3 | Unit evidence gate | submit/dimension_complete 记录 unit coverage，Recipe 与 unit/sourceRefs/structural evidence 可回溯。 | accepted Recipe 能追溯到 unit；未覆盖 readSet 时给出 reject/warning/deviation reason。 |
| Stage 4 | Real IDE Agent validation | AlembicTest 用真实 IDE Agent 做单维 before/after；第一条 adapter 可以是 Codex。 | 报告包含 baseline、新路径、指标、transcript、DB/sourceRefs 证据和残留风险。 |
| Stage 5 | Rescan compatibility | 将 changed files/sourceRefs 映射到 affected units，支持 rescan verify-only/gap-fill。 | rescan 能返回 affected units / preserved Recipe / no duplicate / stale sourceRefs evidence。 |

## 第一版推荐验证维度

推荐：`event-and-data-flow`。

理由：

- 最贴合用户提出的“方法流入流出”。
- 能检验 call graph / data flow hints 是否真的帮助 IDE/host Agent 读对链路。
- 比 `architecture` 更能暴露 one-shot briefing 的短板。

备选：`architecture`。

理由：

- 低风险，当前 Plugin real Codex acceptance pack 已固定 architecture，可作为第一条 adapter smoke。
- 适合 Stage 0 / smoke，但不足以证明方法流入流出优化。

用户已确认按 Design 推荐推进：Stage 0 / smoke 用 `architecture` 做现状 probe，正式 before/after 用 `event-and-data-flow`。

## 与当前主线关系

这是 Plugin cold-start/rescan 测试优化完成后的 IDE Agent 通用能力优化候选，不应打断当前已被总控接收的 038/039 或其它主线，除非总控判断当前阶段正要继续优化 cold-start 质量。

建议优先级：`P1-after-plugin-coldstart`。

## 建议总控下一步

总控接收后不要直接派实现。先做 Stage 0 只读 contract probe：

- 核对 `alembic_call_context` 方法 ID 查询是否稳定，能否从 Mission Briefing / CodeEntityGraph 生成可查的 method key。
- 核对 `ProjectIntelligenceResultProjection` 是否已有 target/module/keyFiles 足够支撑 unit 初版。
- 核对 Mission Briefing compression 是否会删除 source path / evidenceStarters，哪些字段必须移入 packet。
- 核对 `ExternalSubmissionTracker` 是否可扩展 unitId，还是需要新增 lightweight unit ledger。
- 核对 `JobStore` / embedded host-agent recoverable job 是否能持久化 unit progress。
- 核对 real Codex acceptance pack 如何作为第一条 adapter 扩展成 IDE Agent before/after 指标报告。
- 核对哪些能力应归入基础通用模块复用，哪些才属于 IDE Agent adapter / projection；Stage 0 必须列出“禁止重造”的通用能力清单。

用户已确认以下默认取舍，但 Stage 0 仍要用真实代码事实校准最小实现面：

1. 第一条真实验证 adapter 使用 Codex；能力本身保持 IDE Agent 通用。
2. Stage 0 / smoke 使用 `architecture`，正式 before/after 使用 `event-and-data-flow`。
3. Unit progress 持久化位置不提前锁死，由 Stage 0 比较 BootstrapSession + checkpoint、JobStore 和轻量 ledger。
4. Packet retrieval surface 先验证现有 `structure` / `graph` / `call_context` 能否支撑；如果不足，再新增 generic IDE Agent packet retrieval tool。
5. 第一版优先 cold-start packet + unit evidence，rescan 只做兼容设计；完整 affected units 放 Stage 5。

## 已确认取舍

- 第一条 adapter：Codex first adapter。
- 验证维度：`architecture` smoke + `event-and-data-flow` formal before/after。
- Unit progress：Stage 0 决定持久化策略，不提前建表。
- Retrieval surface：Stage 0 先复核现有结构工具，必要时新增 generic IDE Agent packet retrieval。
- Rescan：第一版 cold-start 优先，rescan affected units 后置到 Stage 5。

## 仍需总控排期裁决

该需求排期是在 038/039 之后，还是作为 Plugin cold-start 后续能力优先插入，由总控根据当前主线裁决。

## 交接前自检

- 已按用户要求升级为正式需求。
- 已深挖真实代码链路。
- 未修改产品源码。
- 未修改 AlembicWorkspace 当前状态或全局 TODO。
- 已保留 source path 作为硬证据要求。
- 已明确总控下一步是 Stage 0 只读 probe，不是直接实现。
