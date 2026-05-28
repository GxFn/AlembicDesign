# AI Mock Removal 原始计划

Design Key：AI-MOCK-REMOVAL-2026-05-28
日期：2026-05-28
状态：ready-for-workspace / 新插入前置需求
维护窗口：AlembicDesign
总控：AlembicWorkspace
来源：用户新增需求
建议位置：`INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28`、038、039 之前

## 用户原始目标

把当前 AI mock 功能完整删除。用户判断该功能基本不使用，而且会误导 `AlembicTest` 窗口，因此希望在后续知识进化需求之前先处理。

## 背景证据

Design 做了轻量代码 / 文档扫描，发现 AI mock 不是单点：

- `Alembic` 有 `MockBootstrapPipeline`，当 AI provider 为 mock 时走轻量 mock bootstrap，并绕过真实 internal-agent finalizer。
- `Alembic` HTTP AI routes 暴露 mock provider、mock status、mock cleanup 等路径。
- `AlembicDashboard` 有 mock mode 提示和切换文案。
- `AlembicAgent` 有 `MockProvider`、`mock` provider id、factory fallback 和相关测试约定。
- `AlembicTest` 最新 PCVM Wave 4D 报告使用 `ALEMBIC_AI_PROVIDER=mock`，并明确指出 mock provider 会绕过真实 LLM provider 路径，导致不能证明真实 runtime evidence。

## 初步判断

这是 `bug / requirement-candidate / current-mainline-risk`：

- **bug / risk**：mock path 会让 Test 窗口误以为 cold-start / AI path 已运行，但实际上跳过真实 provider 或 finalizer。
- **requirement-candidate**：删除产品 runtime mock 功能需要跨仓库设计，不应作为随手清理。
- **current-mainline-risk**：当前 PCVM / runtime 验证已受 mock provider 口径干扰。

## 目标候选

删除产品可见 / runtime 可选的 AI mock 功能，让测试与运行时只能在以下两类路径中二选一：

1. 真实 AI provider：OpenAI / Claude / Gemini / DeepSeek / Ollama / 本地千问等真实 provider。
2. 测试专用 fixture：只存在于 test / probe / fixture 文件内，明确命名为 fixture，不作为产品 provider，也不对 Dashboard / CLI / daemon 暴露为 AI provider。

## 非目标候选

- 不删除单元测试里局部手写的 mock object，除非它伪装成产品 AI provider。
- 不删除 AlembicTest 的 fixture resident / fixture provider 机制，只要求避免叫 AI mock 或误导成真实 AI provider。
- 不在本需求内修复真实 AI provider 能力。
- 不改变 037 已归档结论，只把 mock 误导风险作为后续验证前置清理。

## 初步范围

- `Alembic`：删除 mock provider runtime 分支、`MockBootstrapPipeline`、mock cleanup API、Dashboard-visible mock provider config 和相关文案 / 状态。
- `AlembicAgent`：评估并删除或迁移 `MockProvider`，若测试需要，改成 test-local fixture provider，不再作为 `ProviderId` 或 factory fallback。
- `AlembicDashboard`：移除 mock mode 切换和提示，不再建议用户使用 mock bootstrap。
- `AlembicTest`：更新测试说明和 probe，禁止把 `ALEMBIC_AI_PROVIDER=mock` 作为真实 runtime 验证路径。
- `AlembicCore`：原则上不删除测试语义上的 mock / stub 文档，只处理被产品 AI provider 直接引用的 shared contract。

## 关键确认点

用户已于 2026-05-28 确认按 Design 推荐执行：

- “完整删除”覆盖产品 runtime AI mock provider。
- `AlembicAgent` 的 `MockProvider` 应从产品导出、provider registry 和 factory fallback 中移除。
- 测试内部 fake / stub / fixture 可保留，但必须隔离在 test / fixture / probe，不得命名或暴露为产品 AI mock provider。
- Dashboard mock cleanup API 优先直接删除；若总控 Stage 0 发现外部兼容依赖，再短期改为 unavailable / configuration-required。
- 现有 mock-generated candidate / Recipe / bootstrap result 是否需要 cleanup，由总控 Stage 0 代码事实调研确认真实数据后决定。

## 建议下一步

交给总控作为后续连续需求的最前置项。总控接收后应先做跨仓库代码事实调研，再决定 deletion wave。Design 建议执行顺序为：`AI-MOCK-REMOVAL-2026-05-28` -> `INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28` -> `FILE-MONITOR-EVOLUTION-2026-05-28` -> `PLUGIN-FALLBACK-EVOLUTION-2026-05-28`。
