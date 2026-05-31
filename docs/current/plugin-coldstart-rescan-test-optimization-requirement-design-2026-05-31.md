# Plugin Coldstart Rescan Test Optimization 需求设计

Design Key：PLUGIN-COLDSTART-RESCAN-TEST-OPTIMIZATION-2026-05-31
日期：2026-05-31
状态：ready-for-workspace / codex-thread-single-dimension-refined
维护窗口：AlembicDesign
总控：AlembicWorkspace
目标仓库：AlembicPlugin
原始计划：[plugin-coldstart-rescan-test-optimization-original-plan-2026-05-31.md](plugin-coldstart-rescan-test-optimization-original-plan-2026-05-31.md)

## 定位

本需求优化 AlembicPlugin 的 cold-start Recipe 产出与增量扫描测试覆盖，不直接改产品行为。

用户已明确：Plugin 是一个独立整体，只需要考虑 Plugin 的冷启动产出 Recipe、支持增量扫描；其它没用的外围逻辑不纳入本需求。

2026-05-31 代码事实复核后修正：AlembicPlugin 已经有 cold-start / submit knowledge / dimension complete / rescan 的主链路代码，也有部分单元测试和 codex-session harness。当前需求不是重建功能，而是补一条 Plugin 级 Recipe loop 场景测试，把已有逻辑串成可复核证据。

2026-05-31 用户进一步裁决：`CodexScenarioAgentSimulator` 可以删除。Recipe loop 的端到端测试验收应由 Codex 直接操控窗口线程完成，不再扩展伪 Agent simulator。

它应让 Plugin 整体测试能清楚回答：

```text
给一个 fixture project，
Plugin cold-start 是否能产出 Recipe？
已有 Recipe 后，Plugin rescan 是否能基于增量变化保留、补充或演进 Recipe？
```

## 推荐测试模型：Plugin Recipe Loop

采用结果导向闭环：

```text
Fixture Project
-> Plugin Cold-start
-> Codex Window Thread
-> submit_knowledge / dimension_complete
-> Recipe Store Assertion
-> Fixture Project Change
-> Plugin Rescan
-> Recipe Preservation / Evolution Assertion
-> Report Evidence
```

这里不再设计新的 host-agent simulator。端到端验收由真实 Codex 窗口线程读取 Mission Briefing、调用 Plugin MCP 工具并回填证据；仓库内只保留必要的机械断言、artifact analyzer、handler / service 单测和验收包模板。

## 测试模式：Real Codex Single-Dimension Acceptance

后续验收直接使用真实 Codex agent 操控窗口线程，不再由仓库内 simulator 扮演 Agent。

默认测试维度：`architecture`。

选择理由：

- `architecture` 是 Universal Layer 1 维度，所有项目都应可激活。
- 最小 fixture project 也能提供模块边界、入口、依赖方向和项目结构证据。
- 现有单测已使用 `architecture` 走过 `dimension_complete` 路径，属于低风险默认维度。
- 该维度允许 `architecture`、`module-dependency`、`boundary-constraint` 类型 Recipe，适合验证 Recipe 产出、source refs、dimension tags 和 rescan preservation。

### 单维验收链路

```text
Disposable Fixture Project
-> alembic_bootstrap
-> Codex agent selects active dimension: architecture
-> alembic_submit_knowledge({ dimensionId: 'architecture', items: [...] })
-> alembic_dimension_complete({ dimensionId: 'architecture', ... })
-> mechanical evidence check: recipe file + DB entry + dimension/lifecycle/source refs
-> alembic_rescan({ dimensions: ['architecture'], reason: 'single-dimension-acceptance' })
-> Codex agent handles architecture rescan evidence
-> mechanical evidence check: preserved Recipe + architecture evidencePlan + no duplicate on unchanged run
```

### 单维测试能证明什么

- Plugin cold-start briefing 能被真实 Codex agent 消费。
- `submit_knowledge` 到 Recipe persistence 的路径可用。
- `dimension_complete` 能把单维提交绑定到 Recipe / session / checkpoint。
- `alembic_rescan({ dimensions: ['architecture'] })` 能在已有 Recipe 基础上运行，并返回单维 rescan evidence。
- fixture、Recipe 文件、DB 记录和 rescan response 能形成总控可复核证据。

### 单维测试不能证明什么

- 不证明所有维度的 Recipe 质量。
- 不证明完整 bootstrap session 所有维度已完成。
- 不证明每一种 evolution / decay / consolidation 分支都正确。
- 不证明真实大型项目的扫描性能。
- 不证明 resident daemon job、packaged runtime parity 或 AlembicTest 真实项目链路。

这些结论若后续需要，应另建多维 / 真实项目 / packaged runtime 验收，不混入本需求第一版。

## 真实代码链路评估

现有代码已经具备以下链路基础：

- `alembic_bootstrap` 由 `ExternalColdStartWorkflow` 执行 full reset、Phase 1-4 和 Mission Briefing，不启动本地 AI pipeline。
- `alembic_submit_knowledge` 由 `enhancedSubmitKnowledge` 调用 `RecipeProductionGateway.create()` 统一创建 Recipe，并记录到 bootstrap session submission tracker。
- `alembic_dimension_complete` 由 `ExternalDimensionCompletionWorkflow` 恢复 submitted recipe ids，给 Recipe 绑定 dimension 与 `bootstrap:<sessionId>` tag，并保存 checkpoint。
- `alembic_rescan` 由 `ExternalKnowledgeRescanWorkflow` snapshot Recipe、执行 rescan clean / sync、audit Recipe source refs、生成 knowledge rescan plan、prescreen 和 evidencePlan。
- `CodexSessionScenarioRunner` 已有 in-process / live-local 执行壳，`AgentOutputAnalyzer` 已能统计 recipe files、candidate files 和 `knowledge_entries`。

现有缺口也很明确：

- `CodexScenarioAgentSimulator` 当前只会触发 status / init / bootstrap，尚不会提交 knowledge、完成 dimension 或启动 rescan；按用户最新裁决，它不应继续扩展，应作为删除 / 退场对象处理。
- cold-start scenario 当前只验证 bootstrap 入口和边界，不验证 Recipe persisted。
- rescan 的测试主要在 cleanup / plan / evidencePlan 单元层，没有 Plugin scenario 证明已有 Recipe 被保留并驱动增量扫描。
- artifact 统计已经有 `minRecipeFiles` / `minKnowledgeEntries`，但缺少 source refs、dimension、lifecycle、preserved/evolved summary 等更细字段断言。

因此实现建议是退场伪 Agent simulator，保留可复用的机械分析和断言能力，并补 Codex 窗口线程验收包；不是新增平行测试框架，也不是修改 cold-start / rescan 产品行为。

## 测试区域

### Area A：Cold-start Recipe Production

目标：证明 Plugin cold-start 不是只返回 briefing，而是能通过 host-agent workflow 产出可持久化 Recipe。

需要覆盖：

- 使用最小 fixture project。
- 调用 Plugin cold-start 入口获得 Mission Briefing / session / dimensions。
- Codex 窗口线程只选择 `architecture` 单一维度，按 briefing 提交 1-N 条最小有效 knowledge。
- 调用 `alembic_dimension_complete` 完成 `architecture` 维度。
- 断言 Recipe 被持久化。
- 断言 Recipe 带有 dimension、lifecycle、content、source refs、usage / trigger 等必要字段。
- 断言 cold-start 不依赖真实 AI provider。

候选文件：

- `lib/external/mcp/handlers/bootstrap/ExternalColdStartWorkflow.ts`
- `lib/external/mcp/handlers/dimension-complete/ExternalDimensionCompletionWorkflow.ts`
- `lib/external/mcp/handlers/consolidated.ts`
- `test/support/codex-session/*`
- `scripts/verify-codex-session-scenarios.mjs`

### Area B：Incremental Rescan Recipe Preservation

目标：证明已有 Recipe 后，Plugin rescan 不会把知识清空重来，而是保留、审计并返回 `architecture` 单维 evidence。

需要覆盖：

- 从 Area A 的 persisted Recipe 或独立 seed Recipe 开始。
- 调用 `alembic_rescan({ dimensions: ['architecture'], reason: 'single-dimension-acceptance' })`。
- 断言已有 Recipe 被 snapshot / preserved。
- 断言 rescan briefing 包含 `architecture` 的 evidencePlan / prescreen / dimension gap。
- 断言 unchanged project 不重复生成等价 `architecture` Recipe。
- Codex 窗口线程根据 rescan briefing 只处理 `architecture`，不展开其它维度。

第二批再覆盖：

- 修改 fixture project 中与 Recipe source refs 相关的文件。
- 断言 changed / stale / missing evidence 能被标识。
- Codex 窗口线程根据 rescan briefing 只补充或演进相关 Recipe。
- 断言 changed Recipe 产生演进 / 补充证据。

候选文件：

- `lib/external/mcp/handlers/bootstrap-external.ts`
- `lib/external/mcp/handlers/rescan/ExternalKnowledgeRescanWorkflow.ts`
- `lib/external/mcp/handlers/rescan-external.ts`
- `test/support/codex-session/*`

### Area C：Incremental Boundaries

目标：明确 Plugin 增量扫描支持什么、不支持什么，避免测试把 Core 算法责任和 Plugin 整体行为混在一起。

需要覆盖：

- unchanged project rescan：不应重复生成等价 Recipe。
- changed source ref：应产生 stale / evolve / supplement 信号。
- deleted source ref：应产生 missing evidence 或 decay 相关信号。
- requested dimension filter：只扫描指定维度。
- force rescan：明确作为重扫策略，不等同普通增量扫描。

说明：

- Core file-diff / incremental planner 的算法正确性不是本需求第一目标。
- 本需求只验证 Plugin rescan 能把已有 Recipe、项目变化和 host-agent 任务产出串成可用闭环。

第一版单维测试建议拆成三个小模式：

- `cold-start-architecture-recipe`：只验证 cold-start + `architecture` Recipe 产出。
- `rescan-architecture-preserves-recipe`：只验证已有 `architecture` Recipe 在 rescan 中保留并进入 evidencePlan。
- `rescan-architecture-unchanged-no-duplicate`：只验证未变化项目不会重复生成等价 `architecture` Recipe。

`changed source ref` 和 `deleted source ref` 可以作为第二批，因为它们需要更明确的 fixture mutation 与 evidence 解释，不能把第一批验收复杂化。

### Area D：Codex Thread Acceptance Fit

目标：把端到端验收从伪 Agent scenario 转到真实 Codex 窗口线程，同时保留可机械复核的产物断言。

当前已有：

- `explicit-init.json`
- `init-then-bootstrap-ai-ready.json`
- `bootstrap-missing-ai.json`

当前已有 harness 能力：

- scenario runner 可执行多 turn。
- MCP harness 可直接 call tool 并记录 tool result。
- artifact analyzer 已能统计 Recipe 文件与 DB `knowledge_entries`。
- expectation 已支持最小 artifact count 断言。

建议调整方向：

- 删除或退场 `CodexScenarioAgentSimulator`，不要继续扩展自然语言正则或 action DSL。
- 若保留 `CodexSessionScenarioRunner`，只让它承担机械工具调用 / artifact 统计 / 报告输出，不再伪装成 Agent。
- 端到端 Recipe loop 验收由真实 Codex 窗口线程执行，验收包写清目标项目、工具调用顺序、必须回填的 Recipe / DB / evidencePlan 证据。
- artifact assertion 第一版复用 `minRecipeFiles` / `minKnowledgeEntries`，第二版再补 source refs / lifecycle / dimension / evidencePlan 结构断言。

建议新增验收包候选：

- `cold-start-architecture-recipe-acceptance.md`：cold-start 后完成 `architecture` submit / dimension completion，并断言 Recipe。
- `rescan-architecture-preserves-recipe-acceptance.md`：已有 `architecture` Recipe 后，rescan 保留 Recipe 并返回单维 evidencePlan。
- `rescan-architecture-unchanged-no-duplicate-acceptance.md`：无变化时不重复产出等价 `architecture` Recipe。
- 第二批候选：`rescan-architecture-deleted-source-ref-gap-acceptance.md`，source ref 删除时产生可审查 gap。

## 自动化安全边界

可以自动化补测试的事项：

- 新增 Plugin Recipe loop 机械断言和 Codex 窗口线程验收包。
- 新增 fixture project。
- 删除或退场 `CodexScenarioAgentSimulator`，改为 Codex 窗口线程验收包。
- 新增 Recipe persistence assertion。
- 新增 rescan preservation / supplement / evolve assertion。
- 补 verify script 对 Recipe loop 场景的证据报告。

必须停下确认的事项：

- 需要改 cold-start / rescan runtime 行为。
- 需要引入真实 AI Provider 或外部 API key。
- 需要迁移 Core workflow。
- 需要改 `@alembic/core` public contract。
- 需要修改 AlembicTest 真实项目。
- 需要把 packaged runtime / live-local / resident daemon job 拉入同一需求。
- 需要测试与 Recipe 产出或增量扫描无关的外围逻辑。

## 问题修复实现边界

本需求的实现分为两类，不能混在一起：

### A 类：测试承接与证据能力整理

这类可以直接作为 AlembicPlugin 测试优化实现，不代表产品行为有 bug：

- 删除 / 退场 `CodexScenarioAgentSimulator`，同步调整引用它的 codex-session scenario runner 或 verify script。
- 保留或迁移 `AgentOutputAnalyzer` 中有价值的机械证据采集能力，例如 Recipe 文件、DB `knowledge_entries`、lifecycle count。
- 新增 disposable fixture project，只服务 Plugin Recipe loop 验收。
- 新增 Codex thread acceptance pack，写清真实 Codex agent 要执行的工具顺序、目标维度、证据回填字段和停止条件。
- 新增机械 evidence checker，读取 Recipe 文件 / DB / rescan response，不模拟 Agent 推理。

### B 类：验收暴露的产品链路缺陷

只有真实 Codex 单维验收或机械 evidence checker 证明链路断裂时，才进入产品缺陷修复。修复必须贴近断点，不做大重构。

| 断点 | 可能归口 | 修复边界 |
| --- | --- | --- |
| `alembic_bootstrap` 没有返回可用 session / active dimensions / Mission Briefing | `ExternalColdStartWorkflow` 或 Mission Briefing presenter | 修 briefing/session 投影；不重做 cold-start pipeline。 |
| Codex 选择 `architecture` 后无法提交有效 Recipe | `enhancedSubmitKnowledge`、V3 schema、测试 item 字段 | 先修验收数据；若字段映射确有 bug，再修 handler / gateway 映射。 |
| `alembic_submit_knowledge` 创建了 Recipe 但 session tracker 没记录 | `enhancedSubmitKnowledge` 的 `_trackSubmission` 或 session manager | 修 tracker 绑定；不绕过 RecipeProductionGateway。 |
| `alembic_dimension_complete` 找不到 submitted recipe ids 或不能绑定 dimension | `ExternalDimensionCompletionWorkflow` | 修恢复逻辑 / 入参处理 / tags 更新；不改维度模型。 |
| Recipe 文件存在但 DB entry / lifecycle 不一致 | KnowledgeService / persistence sync | 修持久化一致性；不得用测试伪造 DB 通过。 |
| `alembic_rescan({ dimensions: ['architecture'] })` 没有限定到单维 | `KnowledgeRescanIntent`、rescan workflow plan | 修 dimensions 传递 / normalization / plan filter。 |
| rescan clean 删除或丢失已有 Recipe | `CleanupService.rescanClean` 或 rescan snapshot/sync | 修保留策略；不把 rescan 改成 full reset。 |
| rescan response 缺少 preserved Recipe / `architecture` evidencePlan | `ExternalKnowledgeRescanWorkflow`、audit / evidencePlan projection | 修 evidence 投影；不扩大到所有维度验收。 |
| unchanged project 产生重复等价 Recipe | consolidation / duplicate detection / acceptance prompt 边界 | 先判定是否 Codex agent 重复提交；若产品未拦截重复，再修去重链路。 |

### 必须暂停确认的修复

以下发现不能在本需求内顺手修：

- 需要改变 `@alembic/core` public contract。
- 需要新增或替换 AI provider。
- 需要把第一版从单维扩成多维。
- 需要修改 AlembicTest 真实项目。
- 需要改 resident daemon job、packaged runtime、Dashboard 或跨仓库安装流程。
- 需要把 cold-start / rescan 产品行为重构成新架构。

这类情况应回填总控为 `current-mainline-risk` 或新需求候选，由用户 / 总控裁决。

## 分阶段建议

| 阶段 | 目标 | 主要动作 | 完成条件 |
| --- | --- | --- | --- |
| Stage 0 | Recipe loop inventory | 只读确认现有 code / tests / harness 的已覆盖点和缺口，重点检查 scenario 是否已能驱动 submit / dimension complete / rescan。 | 形成代码事实缺口清单，不改代码。 |
| Stage 1 | Simulator retirement | 查清并删除 / 退场 `CodexScenarioAgentSimulator`，保留 artifact analyzer 等机械能力。 | 不再维护伪 Agent；现有入口测试不被误导。 |
| Stage 2 | Codex thread acceptance pack | 编写真实 Codex 窗口线程验收包，固定单一维度 `architecture`，明确 bootstrap / submit / dimension complete / rescan 的执行顺序和回填证据。 | Codex 线程能按验收包跑通 architecture Recipe loop。 |
| Stage 3 | Cold-start architecture loop | 用 fixture project 跑 cold-start -> submit architecture Recipe -> dimension complete -> Recipe persisted。 | Recipe 文件和 DB entry 可断言，字段包含 source refs / dimension / lifecycle。 |
| Stage 4 | Rescan architecture preservation loop | 基于已有 architecture Recipe 跑 `alembic_rescan({ dimensions: ['architecture'] })`。 | preserved Recipe、architecture evidencePlan、no duplicate 有可复核证据。 |
| Stage 5 | No-duplicate / preservation guard | 覆盖 unchanged project、requested dimension、force rescan 边界。 | 不重复等价 Recipe，不误删已有 Recipe。 |
| Stage 6 | Evidence report | verify script 输出 Recipe loop 证据路径、fixture state、Recipe count、evidencePlan / preservation summary。 | 总控可读证据足够验收。 |

## 第一版完成定义

- Plugin cold-start 能在真实 Codex 窗口线程测试中产出至少一条有效 `architecture` Recipe。
- Recipe 持久化字段、source refs、dimension=`architecture`、lifecycle 可断言。
- Plugin rescan 能基于已有 `architecture` Recipe 做保留，并返回 `architecture` evidencePlan。
- unchanged project 不重复生成等价 `architecture` Recipe。
- 测试不依赖真实 AI provider、联网 AI、AlembicTest 真实项目或 resident daemon job。
- 总控能从 verify 输出看到 fixture、Recipe count、rescan dimensions、preservation summary。
- 不引入 AI provider、联网 AI、真实用户项目副作用或产品行为变更。

## 第二批完成定义候选

- changed source ref 能产生可复核 evolution / supplement signal。
- deleted source ref 能产生可复核 evidence gap / decay signal。
- Codex 窗口线程能在 `architecture` 单维内执行 evolve -> gap-fill -> dimension_complete，不扩展到多维。

## 与当前主线关系

这是 Plugin 测试优化需求候选，不应打断当前 Workspace Workflow Optimization 或 PCVM 主线。

建议作为 `P2-automation-testing` 候选，由总控先做 Stage 0 Recipe loop inventory。

## 建议总控下一步

用户已确认范围，且 Design 已复核真实代码。总控接收后先做 Stage 0 只读 inventory，重点不是找产品功能缺失，而是确认测试缺口：

- 当前 cold-start scenario 为什么停在 `alembic_bootstrap`，没有产出 Recipe。
- `CodexScenarioAgentSimulator` 删除影响面，以及哪些 artifact analyzer / report 能力值得保留。
- Codex 窗口线程验收包应如何表达 `architecture` 单维工具调用顺序和回填证据。
- 第一版断言是否用已有 `minRecipeFiles` / `minKnowledgeEntries` 起步。
- 第二版是否补 source refs / dimension / lifecycle / evidencePlan 细断言。

Stage 0 后再派 AlembicPlugin 做 simulator 退场和验收包补齐；不要派 cold-start / rescan 产品逻辑重构。

## 已排除范围

- explicit resident daemon job path：`alembic_codex_bootstrap` / `alembic_codex_rescan`。
- packaged runtime parity：`plugins/alembic-codex/runtime`。
- Codex status / tool recommendation routing。
- live-local real home smoke。
- Core file-diff / incremental planner 独立算法测试。
- 与 Recipe 产出和增量扫描闭环无关的外围逻辑。
