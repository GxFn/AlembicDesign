# Plugin Coldstart Rescan Test Optimization 原始计划

Design Key：PLUGIN-COLDSTART-RESCAN-TEST-OPTIMIZATION-2026-05-31
日期：2026-05-31
状态：ready-for-workspace / codex-thread-single-dimension-refined
维护窗口：AlembicDesign
总控：AlembicWorkspace
目标仓库：AlembicPlugin

## 判断类型

- `new-requirement`
- `testing-optimization`
- `plugin`
- `cold-start`
- `incremental-rescan`
- `requirement-candidate`

这是一个新的 Plugin 测试优化需求，不是 bug 回填，不是实现派发，也不是立即修改 AlembicPlugin 源码。

## 用户原始目标

Plugin 内部的冷启动和增量扫描逻辑需要优化测试。

用户 2026-05-31 已纠偏：Plugin 是一个独立整体，本需求只考虑 Plugin 的 cold-start 是否能产出 Recipe，以及是否支持基于已有 Recipe 的增量扫描。其它与这个闭环无关的逻辑不纳入本需求。

用户 2026-05-31 进一步确认：`CodexScenarioAgentSimulator` 可以删除，不再作为本需求的测试承接方式。Recipe loop 的端到端测试验收应由 Codex 直接操控窗口线程完成，而不是继续维护一个伪 Agent simulator。

因此本需求的真实目标不是扩大测试矩阵，而是建立 Plugin 级别的端到端测试证据：

```text
Plugin cold-start
-> 形成可执行任务 / briefing
-> Codex 操控真实窗口线程提交知识
-> dimension_complete
-> Recipe 持久化
-> Plugin rescan
-> 基于已有 Recipe 做增量判断 / 保留 / 补充 / 演进
```

## 代码事实复核

从 AlembicPlugin 读取到的当前事实：

- `package.json` 提供 `verify:codex-session`，运行 `scripts/verify-codex-session-scenarios.mjs --all`。
- `scripts/verify-codex-session-scenarios.mjs` 支持 `in-process` 与 `live-local` 模式；`live-local` 要求显式 `--scenario`、`--project-root`、`--real-alembic-home`，避免写入开发仓库。
- 当前 cold-start scenario 只有：
  - `test/codex-scenarios/cold-start/explicit-init.json`
  - `test/codex-scenarios/cold-start/init-then-bootstrap-ai-ready.json`
  - `test/codex-scenarios/cold-start/bootstrap-missing-ai.json`
- `lib/external/mcp/handlers/bootstrap-external.ts` 只是 compatibility export，真实 wrapper 在 `lib/external/mcp/handlers/bootstrap/ExternalColdStartWorkflow.ts`。
- External cold-start wrapper 调用 `ProjectIntelligenceCapability.run`，做 full reset、Phase 1-4、Mission Briefing，不启动本地 AI pipeline。
- `lib/external/mcp/handlers/rescan-external.ts` 只是 compatibility export，真实 wrapper 在 `lib/external/mcp/handlers/rescan/ExternalKnowledgeRescanWorkflow.ts`。
- External rescan wrapper 覆盖 recipe snapshot、rescan clean / force clean / no clean、knowledge store sync、Phase 1-4、recipe audit、knowledge rescan plan、prescreen、evidencePlan、Mission Briefing。
- `plugins/alembic-codex/skills/alembic/SKILL.md` 明确默认 `alembic_bootstrap` / `alembic_rescan` 是 Codex host-agent workflow；显式 `alembic_codex_bootstrap` / `alembic_codex_rescan` 才是 resident daemon job。
- `lib/external/mcp/McpServer.ts` 已把 `alembic_bootstrap`、`alembic_rescan`、`alembic_submit_knowledge`、`alembic_dimension_complete` 接到同一 Plugin MCP handler map。
- `lib/external/mcp/handlers/consolidated.ts` 的 `enhancedSubmitKnowledge` 已通过 `RecipeProductionGateway.create()` 统一创建 Recipe，并把创建结果追踪到 bootstrap session。
- `lib/external/mcp/handlers/dimension-complete/ExternalDimensionCompletionWorkflow.ts` 已能从 session submission tracker 恢复 `submittedRecipeIds`，并把 Recipe 绑定到 dimension / bootstrap session tags。
- `test/support/codex-session/AgentOutputAnalyzer.ts` 已能统计 `recipeFiles`、`candidateFiles` 和 `knowledge_entries` 总数 / lifecycle。
- `test/support/codex-session/ScenarioTypes.ts` 的 scenario expectation 已支持 `minRecipeFiles`、`minKnowledgeEntries`、`minCandidateFiles`。
- `test/support/codex-session/AgentSimulator.ts` 当前只会根据自然语言触发 `alembic_codex_status`、`alembic_codex_init`、`alembic_bootstrap`，没有执行 `alembic_submit_knowledge`、`alembic_dimension_complete` 或 `alembic_rescan`；按用户最新决策，它应作为退场 / 删除候选，不再扩展。
- `test/unit/ExternalDimensionCompletionWorkflow.test.ts` 已覆盖 dimension complete 从 tracker 恢复并绑定 Recipe 的单元逻辑。
- `test/unit/production-gateway.test.ts` 已覆盖 `RecipeProductionGateway` 创建 / 校验 Recipe 的单元逻辑。
- `test/unit/KnowledgeRescanPlan.test.ts` 已覆盖 rescan plan、prescreen、evidencePlan、source ref audit 等局部算法。

## 初步问题判断

用户判断“这个功能现在已经有代码逻辑在了”成立。当前不是缺少 cold-start / rescan 产品主链路，而是缺少一个 Plugin 级场景测试把已有链路串起来：

```text
alembic_bootstrap
-> Mission Briefing / external workflow session
-> alembic_submit_knowledge
-> RecipeProductionGateway
-> session submission tracker
-> alembic_dimension_complete
-> Recipe dimension/session binding
-> alembic_rescan
-> Recipe snapshot / audit / evidencePlan / preserved recipes
```

当前测试已经覆盖“冷启动工具是否推荐 / host-agent bootstrap 不依赖 internal AI Provider”的基础场景，也覆盖若干底层单元逻辑；但还没有证明 Plugin 作为独立整体能完成以下结果：

- cold-start 后是否真的产生 Recipe。
- Recipe 是否有可复核的 source refs / dimension / lifecycle / content。
- rescan 是否保留已有 Recipe，而不是清空重来。
- rescan 是否能识别增量变化，并产生补充 / 演进候选。
- rescan 是否能区分 unchanged / changed / stale / missing evidence。
- 整条链路是否可用 Plugin 自身测试证明，而不是只测某个内部函数。

因此需求应收敛为测试补强，不应派发产品功能重做或 cold-start / rescan 重构。

## 设计逻辑边界复核

本需求后续采用真实 Codex agent 操控窗口线程测试，因此边界必须清楚：

- 真实 Codex agent 负责端到端行为：读取 Mission Briefing、选择目标维度、调用 `alembic_submit_knowledge`、调用 `alembic_dimension_complete`、调用 `alembic_rescan`、解释 evidencePlan 并回填证据。
- AlembicPlugin 仓库内只保留机械能力：fixture、artifact analyzer、DB / Recipe 文件检查、handler / service 单测、验收包模板和证据报告。
- `CodexScenarioAgentSimulator` 不再代表真实 Agent 行为，也不再扩展；它是删除 / 退场对象。
- 单一维度验收只证明 Plugin Recipe loop 可用，不证明全部维度质量、不证明完整 bootstrap session 已完成、不证明所有 rescan evolution 策略都正确。
- rescan 的单维测试使用 `dimensions: ['architecture']` 过滤；cold-start 本身仍返回完整 Mission Briefing，但 Codex agent 只处理 `architecture` 一个维度。

推荐默认维度：`architecture`。

理由：

- `architecture` 是 Universal Layer 1 维度，最小 fixture project 也能自然产生模块边界、入口、依赖方向和项目结构类知识。
- 现有代码和单测已使用 `architecture` 作为 `dimension_complete` 示例，风险低。
- 该维度允许 `architecture`、`module-dependency`、`boundary-constraint` 类 Recipe，适合验证 source refs、dimension tags、lifecycle 和 rescan preservation。

## 最终目标候选

建立一套 Plugin cold-start Recipe 产出与增量扫描测试闭环，让 Plugin 作为独立整体具备可自动化、可复核的测试证据：

```text
fixture project
-> Plugin cold-start
-> Codex window thread handles architecture dimension
-> Recipe persisted
-> Plugin incremental rescan with dimensions=['architecture']
-> Recipe preserved / no duplicate
-> report evidence
```

第二批再加入 architecture source ref 变更 / 删除后的 evolve、supplement 或 evidence gap 验收。

## 非目标

- 不改变 cold-start / rescan 产品行为。
- 不新增 AI provider 依赖。
- 不引入真实外部 AI key 或联网 AI。
- 不把 Plugin 测试改成 AlembicTest 默认职责。
- 不在 AlembicPlugin 内恢复 Agent runtime / AI runtime。
- 不绕过 `@alembic/core` package boundary 引用 Core 源码。
- 不把测试优化扩大成 cold-start / rescan 重构。
- 不把工具可见性、status routing、resident daemon job、packaged runtime parity、live-local smoke 作为本需求第一目标。
- 不测试与 Recipe 产出 / 增量扫描无关的外围逻辑。

## 建议下一步

Design 已按用户要求检查真实代码，并确认需求应按“已有逻辑的测试补强”交给总控。总控接收后第一阶段仍应先做只读 Stage 0，但目标要更精确：

- 确认 `CodexScenarioAgentSimulator` 的删除影响面，以及是否需要保留纯机械 artifact analyzer / handler 单测。
- 确认现有 `AgentOutputAnalyzer` 的 `minRecipeFiles` / `minKnowledgeEntries` 是否足以作为第一版断言。
- 确认 Codex 窗口线程验收包如何以 `architecture` 单维执行 `submit_knowledge` + `dimension_complete`，而不是只看 `alembic_bootstrap` 返回。
- 确认 Codex 窗口线程验收包如何从 seeded / produced `architecture` Recipe 开始跑 `alembic_rescan({ dimensions: ['architecture'] })`，断言 preserved recipes、evidencePlan 和不重复。
- 确认是否需要补 source refs / dimension / lifecycle 的更细断言字段。

完成 Stage 0 后，建议总控派 AlembicPlugin 做小范围测试整理：删除或退场 `CodexScenarioAgentSimulator`，保留有价值的机械分析能力，并补 Codex 窗口线程验收包。

若真实 Codex 单维验收暴露产品链路问题，修复应按断点最小归口处理：bootstrap/session 投影、submit_knowledge 字段映射 / tracker、dimension_complete 绑定、rescan dimensions filter、rescan clean preserve、evidencePlan projection 或 duplicate prevention。不得把单点失败扩大成 cold-start / rescan 重构。

## 已排除范围

- explicit resident daemon job path：`alembic_codex_bootstrap` / `alembic_codex_rescan`。
- packaged runtime parity：`plugins/alembic-codex/runtime`。
- Codex status / tool recommendation routing。
- live-local real home smoke。
- Core file-diff / incremental planner 的独立算法正确性。

这些内容若后续需要，应另建需求或由总控在 Stage 0 发现真实阻塞后再裁决。
