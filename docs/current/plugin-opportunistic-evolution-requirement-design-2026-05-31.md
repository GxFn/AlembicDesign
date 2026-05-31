# Plugin Opportunistic Evolution 需求设计

Design Key：PLUGIN-OPPORTUNISTIC-EVOLUTION-2026-05-31
日期：2026-05-31
状态：ready-for-workspace / concrete-requirement / gated-by-file-monitor-boundary
维护窗口：AlembicDesign
总控：AlembicWorkspace
Source TODO：`GTODO-2026-05-24-039`
原始计划：[plugin-opportunistic-evolution-original-plan-2026-05-31.md](plugin-opportunistic-evolution-original-plan-2026-05-31.md)
代码事实检查：[knowledge-evolution-038-039-code-fact-review-2026-05-31.md](knowledge-evolution-038-039-code-fact-review-2026-05-31.md)

## 定位

本需求是 `GTODO-2026-05-24-039` 的具体化：当 AlembicPlugin 运行在 Codex host-agent 环境中，且没有 Alembic daemon file monitor 可用时，利用当前任务可见信号发现知识进化机会。

它不是 038 的降级实现，不模拟 file monitor，不承担 Alembic daemon 的文件监听职责。039 只处理 Plugin 当前能合法看到的 host-agent context、intent、prime/search/Guard/submit 结果和 visible file evidence。

2026-05-31 用户确认补充：039 Plugin 不做文件监控；因为没有 watcher，039 默认使用 git diff 能力获得当前任务的文件变化证据。这里的 git diff 是按任务 / tool 调用获取 evidence，不是后台 watcher、轮询 monitor 或 daemon file event。

2026-05-31 用户继续确认：当 Alembic 主体服务存在 / 可用时，Plugin 可以不处理监控、git diff 降级和知识进化判断，因为面对的是同一套项目文件，应交给 Alembic 主体服务统一处理。039 只在 Alembic 主体服务不存在、不可用或无法承接该 project scope 时，才进入 Plugin-only fallback。

2026-05-31 用户继续确认：039 自己根据 git diff 判断知识进化的逻辑，如果需要使用 Codex Agent 能力测试，必须用真实 Codex Agent 能力验收。deterministic gate / diff parser / sourceRefs matching 可以用 unit 或 fixture；涉及 Agent 推理、工具调用、窗口线程、host context 或 Codex 行为的测试，不能用 mock / simulator 替代真实 Codex Agent。

## 核心原则

- Plugin 只能使用 host-agent 当前可见信号和默认 git diff evidence，不能声称拥有后台 file event。
- Plugin 不做后台文件监控；git diff 只能作为当前任务 evidence source，不得改造成 long-lived watcher。
- Alembic 主体服务可用时，Plugin 不运行自身监控 / evolution detector / git diff fallback；只做 capability routing、结果展示或把请求交给 Alembic 主体服务。
- 039 git diff evolution 判断中，纯确定性逻辑可以单测；需要 Codex Agent 的验收必须使用真实 Codex Agent，不得用模拟 Agent 代替。
- 039 的强 proposal 必须有 evidence，不能从普通对话直接推导长期知识变化。
- `sourceRefs` / sourcePath 仍是关键证据；Recipe 源文件路径不能丢失。
- 039 可以复用 038 的 review consumer，但 producer 和 evidence type 必须不同。
- 第一版只做 proposal / hint，不自动 submit、不自动 mutate、不自动 publish。
- 没有 ProjectScope、sourceRefs、tool outcome 或 visible file evidence 时，默认 no-op 或 low-confidence hint。

## 推荐数据边界

### OpportunisticEvolutionSignal

```ts
type OpportunisticEvolutionSignal = {
  signalId: string;
  producerKind: "plugin-opportunistic";
  projectRoot?: string;
  projectScopeId?: string;
  hostTurnMeta?: {
    sessionHash?: string;
    turnId?: string;
    surface?: "codex" | "unknown";
  };
  triggerKind:
    | "user-correction"
    | "guard-finding"
    | "search-no-result"
    | "search-omitted-relevant"
    | "submit-rejected"
    | "submit-supersede"
    | "prime-omitted"
    | "visible-diff"
    | "active-file";
  intentFrameRef?: string;
  intentEpisodeRef?: string;
  visibleFiles?: string[];
  toolEvidence?: Array<{
    tool: "prime" | "search" | "guard" | "submit_knowledge" | "record_decision" | "other";
    resultKind: string;
    summary: string;
  }>;
  confidence: number;
};
```

### OpportunisticEvolutionProposal

```ts
type OpportunisticEvolutionProposal = {
  proposalId: string;
  sourceTodo: "GTODO-2026-05-24-039";
  designKey: "PLUGIN-OPPORTUNISTIC-EVOLUTION-2026-05-31";
  producerKind: "plugin-opportunistic";
  status: "pending-review" | "dismissed" | "accepted-for-follow-up" | "superseded";
  signals: OpportunisticEvolutionSignal[];
  matchedRecipes?: Array<{
    recipeId: string;
    title?: string;
    sourceRefs: string[];
    matchReason: string;
  }>;
  evidenceGate: {
    hasProjectScope: boolean;
    hasSourceRefs: boolean;
    hasToolEvidence: boolean;
    hasVisibleFileEvidence: boolean;
    lowConfidence: boolean;
    gateResult: "strong-proposal" | "weak-hint" | "no-op";
  };
  proposedAction:
    | "review-existing-recipe"
    | "propose-supplement"
    | "propose-evolution"
    | "record-decision-candidate"
    | "wait-for-file-monitor-or-rescan"
    | "no-action";
  reviewSummary: string;
};
```

这些类型表达需求边界，不要求实现照搬。总控 Stage 0 应根据 AlembicPlugin 真实 tool handler、storage 和 MCP response 结构裁决最终落点。

## 输入允许范围

039 可以消费：

- Alembic 主体服务不可用时的 Plugin-only 当前任务上下文。
- `hostDeclaredIntent` / `hostTurnMeta`，如果 Codex 主动传递。
- Plugin 自行提取的 `RecognizedIntentDraft` / `IntentExtractionFrame`。
- `IntentEpisode` 和 correction / supersede / background 关系。
- `PrimeInjectionPackage.selectedKnowledge` / `omitted` / degraded reason。
- `alembic_search` 的 no-result、low-relevance、omitted-relevant、sourceRefs 结果。
- Guard finding / Guard failure / rule conflict。
- `submit_knowledge` 的 validation failure、consolidation overlap、supersede proposal、sourceRefs 缺失。
- 当前 Codex 可见 active file、visible diff、tool result file refs。
- 默认 git diff / git-worktree diff evidence，用于补足无 watcher 场景下的 changed file 证据。

039 不能消费为事实：

- Alembic 主体服务已可用时，Plugin 自行重新计算的 monitor / git diff / evolution 结论。
- Alembic daemon file event。
- Plugin 没有看到的后台文件变化。
- Plugin 自己创建的后台 watcher / long-lived file monitor。
- AI mock output。
- 无证据普通聊天。
- 用户没有确认的“可能需要长期记录”的随口想法。
- 不属于 ProjectScope 的本地路径。

## Evidence Gate

039 必须先过 evidence gate，再决定是否生成 proposal：

| Gate | strong proposal | weak hint | no-op |
| --- | --- | --- | --- |
| ProjectScope | 有明确 project root / scope | scope 可推断但不稳定 | 无 scope |
| SourceRefs | 命中 Recipe / Guard sourceRefs | 只有 active file / visible diff | 无文件 / 无 source refs |
| Tool Evidence | Guard/search/submit/prime 有明确结果 | 只有 intent / omitted 弱信号 | 无 tool evidence |
| Intent Confidence | 中高置信度且属于当前任务 | 低置信度但重复出现 | 无关或背景闲聊 |
| Actionability | 可指向 review / supplement / evolve | 只能提示后续观察 | 无可行动项 |

默认规则：

- `strong proposal` 才能进入 review queue。
- `weak hint` 只能 surfaced 给当前 Codex turn 或 telemetry，不能自动变知识。
- `no-op` 不写入任何候选。

## 主要流程

```text
Plugin tool invocation / Codex turn
-> Alembic service availability gate
   -> available: route/defer to Alembic service, Plugin no-op for evolution
   -> unavailable: Plugin-only fallback
-> host signal collection
-> intent frame / episode linkage
-> tool outcome normalization
-> default git diff evidence collection
-> sourceRefs / Recipe lookup
-> evidence gate
-> opportunistic proposal or weak hint
-> surfaced next action
```

推荐触发点：

- prime 返回时：只在 `PrimeInjectionPackage.omitted` 与当前 intent / sourceRefs 强相关时提示。
- search 返回时：no-result 或 low-relevance 且有 active file / sourceRefs 时提示。
- Guard 返回时：发现规则冲突、stale sourceRefs 或 missing evidence 时提示。
- submit 返回时：validation / consolidation / supersede 结果显示已有知识需要 evolution 时提示。
- record decision 返回时：用户明确决策可能应变成 Recipe / Guard 时提示。

## 与 038 的复用边界

039 可以复用：

- proposal review status 枚举。
- score / evidence breakdown 格式。
- matched Recipe / sourceRefs display。
- Dashboard / report / Codex nextAction 的展示方式。

039 不能复用：

- `FileChangeBatch`。
- `producerKind: "alembic-file-monitor"`。
- daemon watcher / debounce / file event assumptions。
- changed file event as strong fact。

039 可以复用 038 的 git diff scanner / diff normalizer，但输出必须标明来自 `git-diff` evidence，并保留 `producerKind: "plugin-opportunistic"`。git diff 是 039 的默认文件变化证据来源，不是 Plugin file monitor。

复用前必须先过 Alembic service availability gate。只要 Alembic 主体服务能承接当前 project scope，git diff scanner / diff normalizer 由 Alembic 主体服务使用；Plugin 不重复运行。

如果 038 proposal envelope 尚未稳定，039 Stage 0 可以先产出 adapter 设计，但不应先实现一套不可兼容的数据结构。

## 验证路径

### Targeted Unit

- Evidence gate：strong / weak / no-op 三种结果。
- Trigger normalization：search-no-result、guard-finding、submit-rejected、prime-omitted、user-correction。
- Producer separation：039 输出永远是 `plugin-opportunistic`。
- No file monitor claim：不接受 daemon file event 字段。
- Git diff default：默认读取 git diff evidence；后台 watcher / polling loop 必须失败或 no-op。
- Alembic service available：Plugin detector / git diff fallback 不运行，只验证 route/defer 行为。

### Codex Agent Acceptance

当验收对象涉及 Codex Agent 能力时，必须使用真实 Codex Agent：

- Codex Agent 触发 prime / search / guard / submit / record_decision 后，Plugin 如何收集 host context 和 tool outcome。
- Codex Agent 当前任务产生 git diff evidence 后，039 如何进入 evidence gate。
- Codex Agent 对 surfaced hint / proposal nextAction 的消费和回填。
- 多 turn / 新 session 下，039 是否仍只在 Alembic 主体服务不可用时运行 Plugin-only fallback。

不得使用 mock Agent、CodexScenarioAgentSimulator 或纯自然语言假回填替代这些验收。若当前环境无法启动真实 Codex Agent，则该项标为 `待真实 Codex Agent 验证`，不能写成完成。

### Plugin Fixture

正向场景：

- 当前 intent 与 active file 命中某 Recipe sourceRefs，search no-result：生成 review proposal。
- Guard finding 指向 stale Recipe sourceRefs：生成 review proposal。
- submit_knowledge 返回 overlap / supersede：生成 review proposal。
- prime omitted 反复遗漏当前任务高相关知识，并有 sourceRefs：生成 weak hint 或 proposal。

反向场景：

- 普通聊天：no-op。
- 无 ProjectScope：no-op 或 weak hint。
- 无 sourceRefs、无 tool evidence：no-op。
- AI mock output：忽略。
- Plugin 未看到文件变化但有人声称“文件变了”：不能作为 strong proposal。
- Plugin 试图以 git diff 常驻轮询：失败或 no-op，并标记违反边界。
- Alembic 主体服务可用时，Plugin 仍生成自身 evolution proposal：失败或 no-op，并标记违反边界。
- 需要 Codex Agent 能力但使用 mock / simulator 通过：失败，并标记证据无效。

### Runtime Smoke

若总控需要验证真实 Codex Plugin runtime，可用一个最小 Codex task：

```text
prime -> search no-result -> active file/sourceRefs visible -> proposal surfaced
```

该 smoke 只证明 Plugin-only opportunistic path 可 surfaced，不证明 daemon file monitor 或 full rescan。

## 自动化可执行边界

可以自动化推进：

- Stage 0 host signal inventory。
- evidence gate 单元测试。
- proposal adapter 与 038 review envelope 对齐。
- prime/search/Guard/submit 结果中的 nextAction surfaced。
- positive / negative fixture。
- 真实 Codex Agent acceptance pack：仅当测试对象涉及 Agent 推理、tool 调用或窗口线程时启动；不能用 mock / simulator 替代。

必须停下确认：

- 要让 039 自动创建 pending candidate。
- 要自动调用 `submit_knowledge` 或修改 Recipe。
- 要把 Plugin 做成后台 file monitor。
- 要在 Alembic 主体服务可用时让 Plugin 重复执行 monitor / git diff / evolution 判断。
- 要读取 daemon file events、后台 watcher events 或当前任务边界外的本地变化。
- 要用 mock / simulator 代替真实 Codex Agent 验收涉及 Agent 能力的逻辑。
- 要新增 Core public contract。
- 要把 039 和 038 合并成一个实现。

## 完成定义

## 前置验证与修复项

以下项目是本需求自带 prerequisite。总控接收 039 后必须先验证，不通过就修复；不能当作后续优化延期。

| ID | 前置项 | 验证 | 修复方向 | 完成证据 |
| --- | --- | --- | --- | --- |
| `039-P0-A` | Alembic service availability gate | Plugin 是否能判断主体服务可用且能承接当前 project scope | 新增 gate：available -> route/defer/no-op；unavailable -> Plugin-only fallback | 主体服务可用时 039 不生成自身 proposal；不可用时才 fallback |
| `039-P0-B` | default git diff evidence | 主体服务不可用时，Plugin 是否默认读取 git diff evidence | 接入 shared git diff evidence/scanner contract | git diff evidence 进入 039 detector 输入 |
| `039-P0-C` | evidence gate | 是否有 strong / weak / no-op 分流 | 实现 ProjectScope + sourceRefs + tool evidence + confidence gate | unit / fixture 覆盖 strong、weak、no-op、无 scope、无 sourceRefs |
| `039-P0-D` | surfaced hint / proposal adapter | gate 输出是否能被 Codex 当前 turn 消费 | 输出 nextAction / surfaced hint；必要时 adapter 到 proposal evidence | tool result 可见，且不自动 submit / mutate |
| `039-P0-E` | real Codex Agent acceptance | 涉及 Agent 能力时是否用真实 Codex Agent 验收 | 补真实 Codex Agent acceptance pack；禁用 mock / simulator 作为通过证据 | 真实 Codex Agent 回填；无法运行则标 `待真实 Codex Agent 验证` |

- Plugin-only 场景能从 host-visible evidence 生成 opportunistic proposal。
- Proposal 结构保留 sourceRefs / sourcePath、tool evidence、intent episode 和 confidence。
- strong proposal 需要 ProjectScope + sourceRefs 或 visible file evidence + tool evidence。
- weak hint / no-op 降级路径明确。
- 039 producer 与 038 file monitor producer 清楚分离。
- 039 默认使用 git diff evidence，但不声称 file monitor。
- Alembic 主体服务可用时，039 不运行 Plugin-only fallback；由 Alembic 主体服务处理同一套文件的 monitor / git diff / evolution。
- 需要 Codex Agent 能力的 039 git diff evolution 验收，必须有真实 Codex Agent 证据；没有真实 Agent 证据时只能标记待验证。
- 不自动 submit、不自动 mutate、不自动 publish。
- 测试覆盖正向、反向、低置信度、无 sourceRefs、无 ProjectScope、AI mock ignored。

## 建议总控接收口径

总控可把本需求作为 `GTODO-2026-05-24-039` 的具体需求接收，但建议序列为：

```text
1. INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28
2. FILE-MONITOR-EVOLUTION-2026-05-31 Stage 0 / proposal envelope boundary
3. PLUGIN-OPPORTUNISTIC-EVOLUTION-2026-05-31 Stage 0 host signal inventory
4. 039 implementation only after evidence gate and reuse boundary are clear
```

若总控要并行做 039 的 Stage 0，也只能做只读 host signal inventory，不应抢先实现 proposal contract。

Design 已补只读代码事实检查，可作为总控 Stage 0 的种子证据：Plugin 已有 hostDeclaredIntent / hostTurnMeta / activeFile / sourceRefs 到 resident search 和 intent episode 的输入链路；缺口是 Alembic service availability gate、Plugin-only fallback 下的 git diff evidence、opportunistic detector / evidence gate / surfaced hint 或 proposal adapter，而不是 file monitor。

## 仍需确认

- 是否允许 039 第一版写入 local pending proposal store，还是只在 tool result surfaced。
- weak hint 是否需要计数 / telemetry，用于后续重复信号聚合。
- 若 038 envelope 延迟，039 是否可先实现 adapter draft。
- 是否需要 AlembicTest 做真实 Codex Plugin smoke，还是实现仓库 fixture 即可。
