# Knowledge Evolution Follow-up Continuous Design Plan

Design Key：KNOWLEDGE-EVOLUTION-FOLLOWUP-2026-05-28
日期：2026-05-28
状态：draft / 后续连续需求设计计划
维护窗口：AlembicDesign
总控：AlembicWorkspace
来源：用户新增 AI mock 删除前置需求、intent knowledge baseline、`GTODO-2026-05-24-038`、`GTODO-2026-05-24-039`
前置事实：`GTODO-2026-05-24-037` 已完成已归档

## 定位

本文不是 038 或 039 的完整需求设计，也不是交给总控自动派发的执行计划。它是 intent knowledge route 完成后，继续讨论知识进化主线时的 Design 路线图：先插入 AI mock 删除前置需求，避免测试证据继续被 mock path 误导；再补一个 `IntentKnowledgeEvolutionBaseline`；最后把 038 / 039 作为连续需求设计推进，分别形成可交给总控的 original plan / requirement design / handoff。

## 当前判断类型

- 类型：`new-requirement + current-mainline-risk + TODO + requirement-candidate + follow-up-design-plan`
- 证据状态：037 已有总控归档和 Stage 0-6A 验收记录；AI mock 删除已有 Design 级轻量代码 / 文档证据，但尚未做总控完整代码事实调研；038 / 039 已补 Design 级只读代码事实检查：[knowledge-evolution-038-039-code-fact-review-2026-05-31.md](knowledge-evolution-038-039-code-fact-review-2026-05-31.md)，但仍需总控或源仓库窗口复核后才能作为执行验收事实。
- 总控状态：`GTODO-2026-05-24-037` 已完成已归档；AI mock 删除是用户新增插队需求，Design 建议先交给总控接收；intent knowledge baseline 已完成 Design 草案但需等待 AI mock 删除边界确认；038 / 039 在 global TODO 中为待排期 / 需独立确认。

## 可继承事实

038 / 039 后续设计可以继承以下 intent knowledge route 事实，但不得把历史 TODO 完成直接等同于 038 / 039 可执行：

- `RecognizedIntentDraft` / `IntentExtractionFrame` 已成为意图提取与管理的前置概念。
- `IntentEpisode` 已成为跨会话任务连续性的前置事实。
- `IntentSearchPlan` 可作为意图到检索计划的桥梁。
- `PrimeInjectionPackage` 已成为 prime 交付包候选结构，包含 intent、search、vector、relations、selectedKnowledge、omitted、source refs 和 injection summary。
- Recipe / Guard 的 `sourcePath` / `sourceRefs` 必须作为关键证据保留。
- 低置信度时允许不强行注入，需返回 degraded / needs-confirmation。

这些事实不能推出以下结论：

- 不能推出 file monitor evolution 已设计完成。
- 不能推出 Plugin 无 file monitor 时可以自动提交知识进化。
- 不能推出 full daemon / cold-start / rescan 验证已经覆盖 038 / 039。
- 不能推出 Core typed contract 必须或不必下沉；038 / 039 仍需各自代码事实判断。

## 连续设计顺序

| 顺序 | Design Key 候选 | Source TODO | 目标 | 当前状态 |
| --- | --- | --- | --- | --- |
| 0 | `AI-MOCK-REMOVAL-2026-05-28` | 用户新增前置需求 | 删除产品 runtime AI mock provider / mock bootstrap / mock cleanup / Dashboard mock mode，避免 Test 把 mock path 当成真实 runtime evidence。 | ready-for-workspace / user-confirmed |
| 1 | `INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28` | intent knowledge baseline | 产出 `IntentKnowledgeEvolutionBaseline`，收束可继承契约、不可继承假设、未闭合风险和 038 / 039 消费边界。 | ready-for-workspace / gated-by-ai-mock-removal |
| 2 | `FILE-MONITOR-EVOLUTION-2026-05-31` | `GTODO-2026-05-24-038` | Alembic file monitor evolution：基于本地文件变化触发知识进化候选 / proposal，并与 intent / evidence / source refs 对齐。 | ready-for-workspace / concrete-requirement |
| 3 | `PLUGIN-OPPORTUNISTIC-EVOLUTION-2026-05-31` | `GTODO-2026-05-24-039` | Plugin 无 file monitor 时的机会式知识进化：用 Codex host-agent 可见信号和 intent package 形成 evolution hint / proposal。 | ready-for-workspace / gated-by-file-monitor-boundary |

默认顺序调整为 AI mock 删除 -> 037 收敛 -> 038 -> 039。039 可以复用 038 的 evolution proposal 语义，但不得复制 Alembic daemon file monitor，也不得把 host-agent 机会信号伪装成真实文件监控。

## AI mock 删除前置需求入口

用户于 2026-05-28 新增插队需求：把当前 AI mock 功能完整删除，因为该功能基本不使用，而且会误导 `AlembicTest` 窗口。Design 已新增：

- Design Key：`AI-MOCK-REMOVAL-2026-05-28`
- 原始计划：[ai-mock-removal-original-plan-2026-05-28.md](ai-mock-removal-original-plan-2026-05-28.md)
- 需求设计：[ai-mock-removal-requirement-design-2026-05-28.md](ai-mock-removal-requirement-design-2026-05-28.md)

该需求建议作为 037 收敛、038 和 039 之前的前置清理。理由是：如果产品 runtime 仍暴露 `mock` AI provider / mock bootstrap / Dashboard mock mode，后续 cold-start、file monitor evolution、Plugin fallback evolution 的测试证据都可能继续被 mock path 污染。

用户已确认按 Design 推荐执行：删除范围覆盖 AlembicAgent 产品导出 / registry / factory fallback；test-local fake / fixture 可保留但必须隔离；历史 mock 数据由总控 Stage 0 查证后决定 cleanup；Dashboard mock UI / API 优先直接删除。

## Intent Knowledge Baseline 入口

用户于 2026-05-28 建议：在 038 / 039 之前补一个 intent knowledge route 后续收敛需求。Design 采纳该建议，并新增候选：

- Design Key：`INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28`
- 原始计划：[intent-knowledge-consolidation-original-plan-2026-05-28.md](intent-knowledge-consolidation-original-plan-2026-05-28.md)
- 需求设计：[intent-knowledge-consolidation-requirement-design-2026-05-28.md](intent-knowledge-consolidation-requirement-design-2026-05-28.md)

该需求的目标是把 intent knowledge route 已实现 / 已验证 / 未验证 / 后续风险整理成后续需求可消费的稳定边界，交付物命名为 `IntentKnowledgeEvolutionBaseline`。它不新增产品功能，也不启动 038 / 039。

收敛需求应优先回答：

- 038 / 039 可以依赖哪些 intent knowledge route 契约。
- 哪些结论只是 fixture smoke 或当前完成定义内成立。
- `GTODO-2026-05-27-001` 这类 runtime 风险是否需要作为 038 full daemon 验证前置风险。
- 038 / 039 消费 `IntentEpisode`、`PrimeInjectionPackage`、source refs、relation evidence 的边界。

## 038 后续需求设计入口

### 用户目标候选

让 Alembic 本地增强底座能够观察真实项目文件变化，并基于 037 已建立的 intent / evidence / source refs 语义，判断哪些 Recipe、Guard 或知识候选需要进化。

### 真实使用场景

- 用户在项目中修改源码、文档、配置或测试。
- Alembic daemon / resident file monitor 捕获变化。
- 系统根据文件路径、diff、Recipe source refs、IntentEpisode 和 PrimeInjectionPackage 判断是否存在知识进化机会。
- 结果不是静默改知识，而是形成可解释的 evolution candidate / proposal，供后续 review、Dashboard 或 submit 流程消费。

### 初步功能链路候选

```text
file change event
-> ProjectScope / folder boundary
-> changed file classifier
-> sourceRefs / Recipe linkage lookup
-> intent / episode / recent prime context lookup
-> evolution opportunity scoring
-> evolution proposal package
-> review / submit / Dashboard / log consumer
```

### 设计必须回答的问题

- file monitor 的真实生产方是谁：Alembic daemon、resident service、还是其它本地进程？
- 多 folder ProjectScope 下，文件事件如何归属到 project / folder / repo？
- 哪些文件变化能触发 evolution：source、docs、tests、config、generated artifact 是否不同处理？
- 如何把 file change 与 Recipe `sourcePath` / `sourceRefs` 对齐？
- 如何使用 037 的 `IntentEpisode` 和 `PrimeInjectionPackage`，避免只凭文件路径猜测？
- evolution 结果是 hint、proposal、candidate，还是允许自动 submit？
- 低置信度、无 source refs、生成文件、临时文件、vendor 文件如何降级或忽略？
- 需要 Dashboard UI 吗，还是第一版只做 API / log / report？
- 最终真实验证需要 `AlembicTest` 做 file change runtime monitor 证据吗？

### 非目标候选

- 不复活旧 VSCode 插件 file monitor 模式。
- 不让 AlembicPlugin 代替 Alembic daemon 做真实本地文件监控。
- 不自动修改或发布 Recipe。
- 不绕过 ProjectScope / sourceRefs / evidence 边界。
- 不把所有文件变化都当作知识进化信号。

### 038 Design 完成条件

- 有独立 original plan 和 requirement design。
- 写清 file event producer、consumer、状态变化、失败路径和降级路径。
- 写清 037 intent / evidence / source refs 如何被消费。
- 写清是否需要 Core contract、Dashboard consumer、AlembicTest 真实验证。
- 列出总控代码事实调研清单和需要用户确认的问题。

## 039 后续需求设计入口

### 用户目标候选

当用户没有安装 / 启动 Alembic file monitor，或 Plugin 只处在 Codex host-agent 插件环境时，仍能基于当前任务意图、diff、search、Guard、submit knowledge 和 PrimeInjectionPackage 发现机会式知识进化线索。

### 真实使用场景

- 用户只通过 Codex + AlembicPlugin 工作，没有 Alembic daemon file monitor。
- Plugin 在 prime、search、Guard、submit knowledge、record decision、tool calls 或可见 diff 过程中观察到知识过期、缺失或冲突。
- 系统生成机会式 evolution hint / proposal，让 Codex 或用户决定是否进一步处理。

### 初步功能链路候选

```text
host-agent task / tool signals
-> intent / episode context
-> diff / active file / guard / search / submit outcome signals
-> opportunistic evolution detector
-> proposal / shout / note / submit candidate
-> later Alembic resident or Dashboard consumer
```

### 设计必须回答的问题

- Plugin 能合法拿到哪些 host-agent 信号：active file、user query、tool calls、search result、Guard result、submit result、diff、decision？
- 没有 file monitor 时，如何避免把一次性对话误判成长期知识变化？
- 机会式结果第一版是只做 shout / hint，还是允许写入 pending candidate？
- 如何复用 037 的 `RecognizedIntentDraft`、`IntentEpisode`、`PrimeInjectionPackage`？
- 如何与 038 的 daemon file monitor evolution 去重？
- 哪些场景必须等待 Alembic resident 可用，哪些可由 Plugin 本地完成？
- 需要怎样的 source refs / evidence 才允许生成 proposal？
- 需要 `AlembicTest` 做真实 Codex plugin runtime smoke，还是总控 targeted probe 足够？

### 非目标候选

- 不复制 Alembic daemon file monitor。
- 不假装拥有完整文件事件流。
- 不自动发布知识。
- 不在无证据时把普通回答变成知识进化。
- 不把 Plugin fallback 做成绕过 ProjectScope 的独立知识系统。

### 039 Design 完成条件

- 有独立 original plan 和 requirement design。
- 写清与 038 的关系：fallback、补充，还是互斥路径。
- 写清 host-agent 信号输入、evidence 阈值、proposal 输出和降级策略。
- 写清是否需要 Alembic resident / Core contract / AlembicTest。
- 明确不复制 file monitor 的验证标准。

## 建议 Design 工作顺序

1. 先把 `AI-MOCK-REMOVAL-2026-05-28` 交给总控评审，作为后续连续需求的最前置项。
2. 再确认并完善 `INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28`，形成 `IntentKnowledgeEvolutionBaseline`。
3. 更新 038 original plan：只讨论 Alembic file monitor evolution，不混入 Plugin fallback。
4. 做 038 代码事实调研请求清单：Alembic 当前 file monitor、ProjectScope、Recipe sourceRefs、evolution service、Dashboard consumer。
5. 完成 038 requirement design，标记为 `ready-for-workspace` 或 `needs-code-fact`。
6. 再启动 039 original plan：只讨论 Plugin 无 file monitor 的机会式知识进化。
7. 用 038 的边界反向约束 039：039 不能复制真实 file monitor，只能消费 host-agent 可得信号。
8. 完成 039 requirement design 后再交给总控接收。

## 建议总控下一步

建议 AlembicWorkspace 先接收并评审 `AI-MOCK-REMOVAL-2026-05-28`，把它作为 037 收敛、038、039 之前的前置清理需求。AI mock 删除完成或至少完成总控阶段边界确认后，再推进 `INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28`。该收敛需求完成后，再推进 `FILE-MONITOR-EVOLUTION-2026-05-31` 的 Stage 0 代码事实 inventory；039 已落实为 `PLUGIN-OPPORTUNISTIC-EVOLUTION-2026-05-31`，但建议等 038 proposal / review 边界明确后再实现。

## 仍需确认的问题

- AI mock 删除是否作为总控最前置接收项直接插队到 037 收敛之前？Design 建议是。
- 037 收敛需求是否需要单独交给总控接收，还是只作为 Design 内部前置材料？
- 038 已独立完成需求设计；是否由总控立即接收，取决于 `IntentKnowledgeEvolutionBaseline` 是否已被接收。
- 038 第一版建议只做 evolution proposal / hint，不自动 submit knowledge。
- 039 已独立完成需求设计；第一版建议只做 proposal / hint，pending candidate 需总控或用户再确认。
- 后续真实验证是否优先覆盖 038 file monitor runtime，还是先做 039 Plugin-only runtime smoke？
