# Intent Knowledge Evolution Baseline 原始计划

Design Key：INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28
日期：2026-05-28
状态：ready-for-workspace / gated-by-ai-mock-removal
维护窗口：AlembicDesign
总控：AlembicWorkspace
前置主线：`GTODO-2026-05-24-037` 已完成已归档
新增前置：`AI-MOCK-REMOVAL-2026-05-28`
后续关联：`GTODO-2026-05-24-038`、`GTODO-2026-05-24-039`

## 用户原始目标

在 038 / 039 之前，补一个 intent knowledge route 后续收敛需求。用户随后新增更前置的 AI mock 删除需求，因此本收敛需求应排在 `AI-MOCK-REMOVAL-2026-05-28` 之后。目的不是继续扩功能，而是把已完成的 intent / episode / search / vector / relation / PrimeInjectionPackage 链路收束成后续需求可以稳定消费的边界。

## 背景

037 已在总控完成定义内归档，但归档结论包含边界：

- Stage 6A 是 embedded Plugin runtime + resident-shaped fixture smoke，不等于 full daemon / full cold-start / rescan。
- Codex.app Node / native sqlite addon 环境问题已转入后续 TODO。
- AI mock runtime path 已被用户判断为误导 Test 窗口的新前置风险，应先由 `AI-MOCK-REMOVAL-2026-05-28` 处理，避免本收敛包继续继承 mock evidence。
- 037 引入了多个后续需求会消费的新概念：`RecognizedIntentDraft`、`IntentExtractionFrame`、`IntentEpisode`、`IntentSearchPlan`、`PrimeInjectionPackage`、source refs、score breakdown、relation evidence。
- 038 / 039 若直接开始，容易各自重新解释 037 产物，导致边界漂移。

## 目标

形成一份 `IntentKnowledgeEvolutionBaseline`，明确：

- 哪些 intent knowledge route 产物可以被 038 / 039 作为前置契约使用。
- 哪些只经过 smoke 证明，不能扩大成生产级结论。
- 哪些仍是风险或后续 TODO。
- 038 / 039 设计时应该如何消费 intent、episode、source refs、PrimeInjectionPackage 和 evidence。

## 非目标

- 不新增产品功能。
- 不修复 Node / native addon 环境问题，只把它作为后续验证风险归口。
- 不启动 038 / 039 实现。
- 不创建 AlembicTest 测试单。
- 不把 037 fixture smoke 扩大成 full production validation。

## 初步交付物

- `IntentKnowledgeEvolutionBaseline`：面向总控和后续 Design 的基线说明，不使用 037 这类 TODO 代号作为产物名。
- 038 / 039 Consumer Boundary：后续需求能依赖和不能依赖的 intent knowledge route 能力清单。
- Risk / TODO Mapping：未闭合风险归口，例如 full daemon smoke、Node ABI、真实任务效果度量。
- Verification Baseline：后续需求开始前应复核哪些 intent knowledge route 字段和证据。

## 后续建议

先由总控接收并评审 `AI-MOCK-REMOVAL-2026-05-28`。该前置需求完成或至少完成删除边界确认后，再接收 `INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28`，产出 `IntentKnowledgeEvolutionBaseline`，并继续进入 `FILE-MONITOR-EVOLUTION-2026-05-28` 和 `PLUGIN-FALLBACK-EVOLUTION-2026-05-28`。
