# Intent Knowledge Evolution Baseline 需求设计

Design Key：INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28
日期：2026-05-28
状态：ready-for-workspace / gated-by-ai-mock-removal
维护窗口：AlembicDesign
总控：AlembicWorkspace
原始计划：[intent-knowledge-consolidation-original-plan-2026-05-28.md](intent-knowledge-consolidation-original-plan-2026-05-28.md)
前置主线：`GTODO-2026-05-24-037` 已完成已归档
新增前置：`AI-MOCK-REMOVAL-2026-05-28`
后续关联：`GTODO-2026-05-24-038`、`GTODO-2026-05-24-039`

## 定位

本需求是 intent knowledge route 归档后的基线收敛需求，排在 `AI-MOCK-REMOVAL-2026-05-28` 之后、038 / 039 之前。

它不继续扩大意图识别、检索、向量、关联关系或 prime 注入功能；也不启动 file monitor 或 Plugin fallback evolution。它只产出一份 `IntentKnowledgeEvolutionBaseline`，让总控、Design 和后续执行窗口都能用同一组证据边界继续讨论 038 / 039。

`037` 只作为历史 TODO 代号出现在来源字段和总控记录中，不进入交付物命名。

## 最终目标

产出一份 `IntentKnowledgeEvolutionBaseline`，明确回答：

- 038 / 039 可以稳定继承哪些 intent knowledge route 契约。
- 哪些结论只在 targeted test、fixture smoke 或当前完成定义内成立。
- 哪些证据需要在 AI mock 删除后重新解释或复核。
- 038 / 039 分别能消费哪些 intent、episode、search、vector、relation、source refs 和 prime package 信息。
- 哪些风险应独立进入 TODO / Backlog / 验证项，而不是混入 038 / 039 的完成定义。

## 真实使用场景

1. 总控准备启动 038 file monitor evolution，需要判断文件变化是否可以消费 `IntentEpisode`、`sourceRefs` 和 `PrimeInjectionPackage`。
2. 总控准备启动 039 Plugin no-monitor evolution，需要判断 Plugin-only 场景能消费哪些 host intent、Guard、search、submit 和 prime package 信号。
3. 设计窗口继续细化 038 / 039，需要避免各自重新解释 intent knowledge route 已完成内容，造成边界漂移。
4. 测试窗口准备真实 runtime 验证时，需要知道哪些证据不能再使用 AI mock path，哪些必须重新跑真实 provider / fixture-separated path。

## Baseline 结构草案

```ts
type IntentKnowledgeEvolutionBaseline = {
  designKey: "INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28";
  sourceTodo: "GTODO-2026-05-24-037";
  artifactName: "IntentKnowledgeEvolutionBaseline";
  status: "ready-for-workspace" | "accepted" | "superseded";
  generatedAfter: {
    aiMockRemoval: "pending" | "boundary-confirmed" | "completed";
    latestCodeFactReview: "required" | "completed";
  };
  stableCapabilities: Array<{
    capability:
      | "RecognizedIntentDraft"
      | "IntentExtractionFrame"
      | "IntentEpisode"
      | "IntentSearchPlan"
      | "PrimeInjectionPackage"
      | "SourceRefs"
      | "ScoreBreakdown"
      | "RelationEvidence";
    evidenceLevel:
      | "code-reviewed"
      | "targeted-tested"
      | "fixture-smoke"
      | "real-runtime-smoke"
      | "needs-revalidation";
    consumerUse: string;
    sourceDocs: string[];
    sourceCommits?: string[];
    caveats: string[];
  }>;
  consumerBoundary: {
    fileMonitorEvolution038: {
      allowedInputs: string[];
      forbiddenInputs: string[];
      firstVersionOutput: "evolution-hint" | "evolution-proposal";
    };
    pluginNoMonitorEvolution039: {
      allowedInputs: string[];
      forbiddenInputs: string[];
      firstVersionOutput: "opportunistic-hint" | "evolution-proposal";
    };
  };
  riskLedger: Array<{
    risk: string;
    ownerCandidate: "Alembic" | "AlembicPlugin" | "AlembicTest" | "AlembicWorkspace";
    blocks: "038" | "039" | "runtime-validation" | "release-confidence" | "none";
    recommendation: string;
  }>;
  goNoGoForNextDesign: {
    canStart038: boolean;
    canStart039: boolean;
    reasons: string[];
  };
};
```

## 稳定能力矩阵

| 能力 | 后续消费方式 | 当前证据判断 | 收敛要求 |
| --- | --- | --- | --- |
| `RecognizedIntentDraft` / `IntentExtractionFrame` | 为 038 / 039 提供用户目标、约束、非目标、置信度和低置信度降级原因。 | targeted-tested / code-reviewed | 记录字段来源；明确 Codex 不强制传 intent，Plugin 必须能自提取。 |
| `IntentEpisode` | 为跨会话 evolution 判断提供最近任务上下文和 correction / supersede 关系。 | targeted-tested / fixture-smoke | 复核 storage / session 边界；不能把 fixture smoke 写成 full daemon evidence。 |
| `IntentSearchPlan` | 为后续 knowledge lookup 和 candidate discovery 提供 query plan、whySelected 和 omitted 语义。 | targeted-tested | 明确 038 的 file event 是否可触发 search plan，039 是否只消费 host-agent 可见 query。 |
| `PrimeInjectionPackage` | 汇总 intent、search、vector、relations、selectedKnowledge、omitted、source refs 和 injection summary。 | fixture-smoke / needs-revalidation | AI mock 删除后，复核 runtime evidence；不能把 package 存在等同于真实任务效果提升。 |
| `sourcePath` / `sourceRefs` | 连接 Recipe / Guard / 文件变化 / proposal 的关键证据。 | code-reviewed / targeted-tested | 必须保留；不得丢失源文件路径；不得泄露不该暴露的本机绝对路径。 |
| `ScoreBreakdown` / `RelationEvidence` | 为 038 / 039 判断 why selected / why omitted / relation boost 提供解释。 | targeted-tested / design-boundary | 需要区分 keyword、vector、intent anchor、relation boost 的证据贡献。 |

## 不得继承的假设

- 不得把 intent knowledge route 完成解释成 file monitor evolution 已完成。
- 不得把 Plugin runtime smoke 解释成 full Alembic daemon / cold-start / rescan 已验证。
- 不得把 `PrimeInjectionPackage` 存在解释成真实任务效果已经显著提升。
- 不得假设 Core typed contract 已经必须下沉；038 / 039 需要各自代码事实判断。
- 不得让 039 复制 038 的 file monitor，或让 Plugin fallback 假装拥有完整文件事件流。
- 不得把 AI mock path 产生的证据继续当成真实 runtime evidence。

## 038 消费边界

038 可以消费：

- `IntentEpisode`：判断文件变化是否与最近任务上下文、correction 或 supersede 有关。
- `sourceRefs`：把 file change 与 Recipe / Guard / source evidence 连接。
- `PrimeInjectionPackage.selectedKnowledge` / `omitted`：理解最近哪些知识被注入或被排除。
- `IntentSearchPlan`：补充变更相关的 knowledge lookup。
- `ScoreBreakdown` / `RelationEvidence`：解释 evolution hint / proposal 的证据贡献。

038 不应消费：

- Plugin host-only signal 作为 file event 事实。
- fixture smoke 作为 full daemon monitor 可靠性证据。
- 无 source refs / 无 changed file evidence 的候选作为强 proposal。
- AI mock provider 输出作为 file monitor evolution 的 runtime evidence。

038 第一版建议输出：`evolution hint / proposal`，不自动 submit knowledge。

## 039 消费边界

039 可以消费：

- `RecognizedIntentDraft` 和 `IntentEpisode`：判断当前 Codex host-agent 任务是否存在知识进化机会。
- `PrimeInjectionPackage`：理解当前 prime 注入、遗漏和低置信度降级。
- Guard / search / submit outcome：作为机会式 proposal 的证据。
- 可见 diff / active files / tool result：仅作为 host-agent 可见信号，不得伪装成 daemon file event。

039 不应消费：

- Alembic daemon file monitor 的生产责任。
- 未经确认的本地文件事件。
- 无 evidence 的普通对话内容。
- 自动发布或静默修改 Recipe 的能力。
- AI mock output 作为 host-agent 信号。

039 第一版建议输出：`opportunistic hint / proposal`，默认不自动写知识；是否允许 pending candidate 留给 039 独立需求确认。

## 风险与 TODO 归口

| 风险 | 归口建议 | 对后续影响 |
| --- | --- | --- |
| 产品 runtime AI mock 仍可被 Test 使用 | `AI-MOCK-REMOVAL-2026-05-28` | 作为本需求前置门禁；否则 baseline 可能继续继承 mock evidence。 |
| Node / native sqlite addon 环境 | `GTODO-2026-05-27-001` | 阻塞 full daemon / release runtime confidence；不阻塞基线文档，但影响 038 runtime 验证。 |
| full daemon / cold-start / rescan 未覆盖 prime package | 后续验证 TODO | 038 若依赖 daemon runtime，需要单独验证。 |
| 真实任务效果未度量 | 后续 research / evaluation | 不阻塞 038/039 设计，但不能把价值提升写成已证明事实。 |
| source refs 泄露或丢失 | 038 / 039 必须继承的 evidence guard | 任何 proposal schema 都必须保留可复核源路径，同时遵守隐私边界。 |

## 分阶段建议

| 阶段 | 目标 | 主要动作 | 交付 |
| --- | --- | --- | --- |
| Stage 0 | 前置门禁复核 | 确认 `AI-MOCK-REMOVAL-2026-05-28` 至少完成删除边界确认；列出受影响 runtime evidence。 | 前置状态说明。 |
| Stage 1 | 证据矩阵补齐 | 总控复核归档文档、提交、测试输出、fixture smoke 和当前 HEAD 代码事实。 | stable capabilities evidence table。 |
| Stage 2 | 消费边界固定 | 明确 038 / 039 allowed inputs、forbidden inputs、first output shape。 | consumer boundary。 |
| Stage 3 | 风险归口 | 把 Node ABI、full daemon 验证、真实效果度量、source refs 风险分流到 TODO / Backlog。 | risk ledger。 |
| Stage 4 | Handoff | 标记 baseline 可供 038 original plan 使用。 | `IntentKnowledgeEvolutionBaseline` 文档或总控账本条目。 |

## 完成定义

- 交付物命名使用 `IntentKnowledgeEvolutionBaseline`，不使用 `G037...` 或其它 TODO 代号命名。
- 每个稳定能力都有证据等级、来源文档 / 提交 / 测试线索和 caveat。
- 明确 AI mock 删除对 runtime evidence 的影响。
- 明确 038 / 039 各自可消费和不可消费的输入。
- 明确 038 / 039 第一版默认只做 hint / proposal，不自动 submit / 发布知识。
- 明确后续风险归口，不把未验证 runtime、Node ABI、真实效果度量混入 038 / 039 完成定义。
- 给总控一个 go / no-go 判断：是否可以启动 038 original plan，039 是否必须等 038 proposal 语义稳定后再启动。

## 非目标

- 不改产品源码。
- 不做实现验收。
- 不创建测试单。
- 不启动 038 / 039。
- 不替总控修改 global TODO 或当前状态。
- 不把 baseline 伪装成 runtime API contract；是否下沉为 Core contract 必须由 038 / 039 代码事实决定。

## 建议总控下一步

总控先接收并评审 `AI-MOCK-REMOVAL-2026-05-28`。该前置需求完成或至少完成删除边界确认后，接收本需求，补一次轻量代码事实 / 证据复核，产出 `IntentKnowledgeEvolutionBaseline`。

完成 baseline 后，再启动 `FILE-MONITOR-EVOLUTION-2026-05-28` 的 original plan 和 requirement design；`PLUGIN-FALLBACK-EVOLUTION-2026-05-28` 保持候选，等 038 的 proposal 语义稳定后再进入完整设计。

## 仍需确认

- baseline 是否由总控写入 workspace 账本，还是由 Design 先产出最终 handoff 后交给总控接收？Design 建议总控接收后写入账本，因为它需要复核原始证据。
- 038 第一版是否确认只做 `evolution hint / proposal`，不自动 submit knowledge？Design 建议是。
- 039 是否等待 038 proposal 语义稳定后再进入完整设计？Design 建议是。
