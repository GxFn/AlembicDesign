# AI Mock Removal 需求设计

Design Key：AI-MOCK-REMOVAL-2026-05-28
日期：2026-05-28
状态：ready-for-workspace / deletion requirement
维护窗口：AlembicDesign
总控：AlembicWorkspace
原始计划：[ai-mock-removal-original-plan-2026-05-28.md](ai-mock-removal-original-plan-2026-05-28.md)
建议顺序：先于 `INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28`、038、039

## 最终目标

删除 Alembic 系列产品 runtime 中的 AI mock 功能，避免 Test 窗口或总控把 mock path 当成真实 AI / cold-start / runtime evidence。删除后，产品运行时不再提供 `mock` AI provider、mock bootstrap、mock cleanup 或 Dashboard mock mode；测试只允许使用明确的 fixture / stub，并且不得被描述成 AI provider。

## 真实问题

当前 mock AI 有两个风险：

1. **验证误导**：`AlembicTest` 已出现使用 `ALEMBIC_AI_PROVIDER=mock` 的报告，结论明确说 mock path 会绕过 internal-agent finalizer，不能证明真实 LLM provider 路径。
2. **产品语义混乱**：mock provider 同时出现在 Agent provider factory、Alembic bootstrap pipeline、Dashboard AI settings / cleanup 和若干 runtime skip 分支中，容易让用户或测试窗口误以为 mock 是一个受支持的运行模式。

## 代码事实初扫

已观察到的入口：

- `Alembic/lib/workflows/capabilities/execution/internal-agent/MockBootstrapPipeline.ts`
- `Alembic/lib/workflows/capabilities/execution/internal-agent/InternalDimensionExecutionPipeline.ts`
- `Alembic/lib/workflows/capabilities/execution/internal-agent/InternalDimensionFillPreparation.ts`
- `Alembic/lib/http/routes/ai.ts`
- `Alembic/lib/service/module/ModuleService.ts`
- `Alembic/lib/service/vector/ContextualEnricher.ts`
- `Alembic/vendor/AlembicDashboard/src/components/Layout/Header.tsx`
- `Alembic/vendor/AlembicDashboard/src/components/Modals/LlmConfigModal.tsx`
- `Alembic/vendor/AlembicDashboard/src/i18n/locales/en.ts`
- `Alembic/vendor/AlembicDashboard/src/i18n/locales/zh.ts`
- `AlembicAgent/src/external/ai/providers/MockProvider.ts`
- `AlembicAgent/src/external/ai/AiFactory.ts`
- `AlembicAgent/src/external/ai/registry/model-defs.ts`
- `AlembicAgent/src/external/ai/AiProviderManager.ts`
- `workspace-ledger/AlembicTest/pcvm-wave4d-coldstart-scorecard-smoke-2026-05-28.md`

本扫描只是 Design 级线索；总控接收后需要做完整代码事实调研。

## 删除边界

### 应删除

- 产品 runtime 可选择的 `mock` AI provider。
- Alembic mock bootstrap pipeline 和 mock mode fallback。
- Dashboard / API 中对 mock mode 的配置、提示、切换和 cleanup 能力。
- CLI / daemon / workflow 中把 mock 当作合法 provider 或可运行路径的逻辑。
- AlembicTest 文档和任务中推荐或使用 `ALEMBIC_AI_PROVIDER=mock` 作为 runtime 验证路径的口径。

### 可保留但需重命名 / 隔离

- 单元测试内局部 mock object。
- 明确位于 test / fixture / probe 中的 fake provider。
- AlembicTest 的 resident-shaped fixture 或 HTTP fixture，但命名应避免 AI mock provider 语义。

### 不应保留

- 被 Dashboard、CLI、daemon、MCP、cold-start 或 rescan 暴露给用户 / Test 的 mock provider。
- 任何会生成 mock candidate / mock Recipe / mock bootstrap result 的产品路径。
- `mock` 作为真实 provider registry id 或 fallback provider。

## 建议完成定义

- 搜索产品 runtime 代码，`mock` 不再作为 AI provider id、factory fallback、Dashboard option、HTTP route option、cold-start / rescan provider 或 bootstrap pipeline。
- `MockBootstrapPipeline` 删除或从产品 build 中彻底断开。
- Dashboard 不再出现 mock mode 切换和 mock cleanup。
- AlembicTest 不再使用 `ALEMBIC_AI_PROVIDER=mock` 作为真实 runtime 证据路径；相关历史报告保留为背景，但新测试单禁止使用。
- 单元测试仍可使用 test-local fixtures，但名称、路径和文档不能误导为产品 AI provider。
- 文档明确：没有真实 AI provider 时，应返回 unavailable / configuration-required，而不是进入 mock 成功路径。

## 分阶段候选

| 阶段 | 目标 | 主要仓库 | 验收 |
| --- | --- | --- | --- |
| Stage 0 | 完整代码事实调研 | 总控 + Alembic / AlembicAgent / Dashboard / Test | mock provider 全入口清单、删除影响、测试依赖清单。 |
| Stage 1 | Alembic runtime mock path 删除 | `Alembic` | mock bootstrap / ai route / workflow fallback 不再存在，targeted tests 更新。 |
| Stage 2 | AlembicAgent provider registry 清理 | `AlembicAgent` | `mock` 不再是产品 provider id / factory fallback；测试 fixture 隔离。 |
| Stage 3 | Dashboard / docs / Test 口径清理 | `AlembicDashboard` / `AlembicTest` / workspace ledger | UI 不再暴露 mock；Test 不再推荐 mock runtime path。 |
| Stage 4 | 总控验证 | AlembicWorkspace | 全仓库扫描、targeted tests、runtime unavailable path 验证。 |

## 对后续需求的影响

- `INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28` 应把 AI mock removal 作为前置事实或风险关闭项。
- 038 file monitor evolution 的真实验证不能依赖 mock AI provider。
- 039 Plugin fallback evolution 不能把 mock AI output 当作 host-agent 信号。
- PCVM / cold-start / runtime 测试应使用真实 provider、本地千问、Ollama 或明确 fixture，不得使用产品 mock provider。

## 非目标

- 不修复真实 AI provider 配置。
- 不要求所有测试连接真实外部 API。
- 不删除普通测试语义中的 mock / stub / fixture。
- 不删除历史归档中提到 mock 的事实记录。
- 不改变 037 已归档结论。

## 用户确认

2026-05-28 用户确认按 Design 推荐执行：

- 删除范围覆盖 `AlembicAgent` 的 `MockProvider` 产品导出、provider registry 和 factory fallback。
- 允许保留 test-local fake / stub / fixture，但不得命名或暴露为产品 AI mock provider。
- 历史 mock-generated candidate / Recipe / bootstrap result 是否需要一次性 cleanup，由总控 Stage 0 查证真实数据后决定。
- Dashboard mock cleanup UI / API 优先直接删除；若总控 Stage 0 发现外部兼容 contract，再短期改为 `configuration-required` / `unavailable`，但不得继续表现为可用 mock 能力。

## 总控阶段复核点

- Stage 0 需要确认是否存在真实 mock-generated 数据需要 cleanup。
- Stage 0 需要确认 Dashboard mock cleanup API 是否存在外部兼容 contract。
- Stage 0 需要区分产品 runtime mock provider 与 test-local fixtures，避免误删普通单元测试 mock。

## 建议总控下一步

用户已确认该需求需要插队先做，并确认上述删除取舍。建议总控把 `AI-MOCK-REMOVAL-2026-05-28` 接收为 037 收敛、038、039 之前的前置清理需求。先做代码事实调研和目标阶段确认，再决定是否派实现窗口。
