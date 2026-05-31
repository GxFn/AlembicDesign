# File Monitor Evolution 需求设计

Design Key：FILE-MONITOR-EVOLUTION-2026-05-31
日期：2026-05-31
状态：ready-for-workspace / concrete-requirement / gated-by-intent-baseline
维护窗口：AlembicDesign
总控：AlembicWorkspace
Source TODO：`GTODO-2026-05-24-038`
原始计划：[file-monitor-evolution-original-plan-2026-05-31.md](file-monitor-evolution-original-plan-2026-05-31.md)
代码事实检查：[knowledge-evolution-038-039-code-fact-review-2026-05-31.md](knowledge-evolution-038-039-code-fact-review-2026-05-31.md)

## 定位

本需求是 `GTODO-2026-05-24-038` 的具体化：Alembic 本地 file monitor 观察项目文件变化，并基于 Recipe / Guard source refs、intent episode、prime package 和检索/关联证据产出知识进化 proposal。

它不是 039 Plugin fallback，不由 Plugin 模拟 file monitor；也不是自动知识写入系统。第一版只把真实 file change 转成可审查、可解释、可降级的 proposal。

2026-05-31 用户确认补充：038 要替代旧 VSCode 扩展 API 级文件监控逻辑，第一版按 Design 推荐推进。Alembic 主体以 native filesystem watcher 作为主路径；当 038 没有 watch / watcher 不可用时，至少必须使用 git diff 能力降级。039 Plugin 不做文件监控；当 Alembic 主体服务可用时，同一套文件的监控、git diff 降级和进化判断都交给 Alembic 主体服务，Plugin 不重复处理。

## 核心原则

- File monitor 的事实来源必须是 Alembic 主体 native filesystem watcher / daemon / resident service 捕获的真实文件事件，用于替代旧 VSCode extension API 级 watcher。
- Git diff / git-worktree scan 是共享降级与补偿层，不是 038 的主监控语义；降级时 capability/status 必须明确标记 degraded。
- Recipe 源文件路径和 `sourceRefs` 是关键证据，不能丢失。
- Intent / prime / search / vector / relation 只做证据增强和解释，不能替代真实 changed file evidence。
- 低置信度、无 source refs、ignored path、generated/vendor path 必须降级或忽略。
- Proposal 默认 pending review，不自动 submit、不自动更新 Recipe、不自动 publish。
- 038 与 039 必须通过 `producerKind`、evidence type 和 trigger source 明确区分。

## 推荐数据边界

### FileChangeBatch

```ts
type FileChangeBatch = {
  projectRoot: string;
  projectScopeId?: string;
  batchId: string;
  producerKind: "alembic-file-monitor";
  capturedAt: string;
  debounceWindowMs: number;
  files: Array<{
    path: string;
    changeKind: "created" | "modified" | "deleted" | "renamed";
    oldPath?: string;
    ignored: boolean;
    ignoreReason?: "outside-project-scope" | "generated" | "vendor" | "temporary" | "unsupported";
    classifier?: "source" | "docs" | "test" | "config" | "recipe-source" | "unknown";
  }>;
};
```

### RecipeImpactCandidate

```ts
type RecipeImpactCandidate = {
  recipeId: string;
  recipeTitle?: string;
  matchedSourceRefs: Array<{
    sourcePath: string;
    matchKind: "exact" | "ancestor" | "renamed" | "deleted";
    evidenceLevel: "strong" | "medium" | "weak";
  }>;
  suggestedImpact:
    | "verify-current"
    | "possibly-stale"
    | "source-missing"
    | "needs-supplement"
    | "no-action";
};
```

### FileMonitorEvolutionProposal

```ts
type FileMonitorEvolutionProposal = {
  proposalId: string;
  sourceTodo: "GTODO-2026-05-24-038";
  designKey: "FILE-MONITOR-EVOLUTION-2026-05-31";
  producerKind: "alembic-file-monitor";
  status: "pending-review" | "dismissed" | "accepted-for-evolution" | "superseded";
  projectRoot: string;
  fileChangeBatchId: string;
  changedFiles: string[];
  impactedRecipes: RecipeImpactCandidate[];
  intentEvidence?: Array<{
    intentEpisodeId?: string;
    relation: "same-task" | "correction" | "supersede" | "background" | "unknown";
    confidence: number;
    why: string;
  }>;
  primeEvidence?: Array<{
    packageId?: string;
    selectedKnowledgeIds?: string[];
    omittedKnowledgeIds?: string[];
    whyRelevant: string;
  }>;
  scoreBreakdown: {
    sourceRefScore: number;
    intentScore: number;
    searchRelationScore?: number;
    riskScore?: number;
    finalConfidence: number;
  };
  proposedAction:
    | "verify-recipe"
    | "propose-evolution"
    | "propose-supplement"
    | "propose-decay-review"
    | "no-action";
  reviewSummary: string;
};
```

这些类型只是需求设计边界，不强制实现文件名或语言。总控 Stage 0 应按真实代码结构决定是否下沉为 Core contract、Alembic local type 或 API DTO。

## 输入允许范围

038 可以消费：

- Alembic file monitor 的真实 file change event / batch。
- ProjectScope、workspace root、ignore / generated / vendor 规则。
- Recipe / Guard 的 `sourcePath` / `sourceRefs`。
- `IntentEpisode`：用于判断同一任务、纠错、supersede 或背景信息关系。
- `PrimeInjectionPackage.selectedKnowledge` / `omitted`：用于解释最近知识注入和遗漏。
- `IntentSearchPlan`、keyword / BM25 / vector / relation evidence：用于辅助 impact lookup 和 confidence。
- recipe audit / source ref audit / rescan evidence，只能作为补充证据。

038 不能消费为强事实：

- Plugin host-only signal。
- 普通对话中的“我觉得有变化”。
- AI mock output。
- fixture-only smoke 结论。
- 无 ProjectScope、无 sourceRefs、无 changed file evidence 的候选。

## 主要流程

```text
NativeFileSystemWatcherEvent
-> ProjectScope gate
-> ignore / generated / vendor filtering
-> debounce + batch
-> changed file classifier
-> sourceRefs impact lookup
-> intent / prime / search enrichment
-> proposal scoring
-> FileMonitorEvolutionProposal pending-review
-> review consumer
```

关键要求：

- ProjectScope gate 失败时直接 no-op，并记录可诊断原因。
- Native watcher 是 038 主事件源；当 watcher 不可用、daemon restart 后补偿或发生 missed-event reconciliation 时，038 必须降级到 git diff / git-worktree scan。
- 039 不做 watcher，默认使用 git diff 能力作为当前任务的文件变化证据，但不得把 git diff 改造成后台文件监控。
- Alembic 主体服务存在 / 可用时，039 不应运行 Plugin-side git diff fallback 或 evolution detector；由 038 / Alembic 主体服务统一处理同一套文件。
- 第一版不承诺未保存 editor buffer 级事件；只处理落盘后的 filesystem event / git fallback evidence。
- watcher 范围按 ProjectScope，而不是只监控已知 sourceRefs；但 strong proposal 仍必须依赖 sourceRefs / visible evidence。
- sourceRefs 命中是强 proposal 的主要依据。
- intent / prime evidence 只能提高解释质量或让弱 proposal 进入 review，不能替代 sourceRefs。
- proposal 必须可复核，不能只输出自然语言结论。
- review consumer 不等于自动 submit；自动 submit 若未来需要，应另开需求。

## 验证路径

### Targeted Unit

- Path filter：ProjectScope 内外路径、ignore、generated、vendor、temporary。
- Debounce：多次修改聚合为 batch，不因 hover / temp file 重复触发。
- SourceRefs matching：exact、ancestor、deleted、renamed。
- Scoring：sourceRefs 命中高于 intent-only；低置信度降级。

### Integration Fixture

最小 fixture：

```text
fixture-project/
  src/module-a.ts
  docs/architecture.md
  recipes/
    recipe-a.json or recipe-a.md with sourceRefs: ["src/module-a.ts"]
```

验收场景：

- 修改 `src/module-a.ts`：生成 `propose-evolution` 或 `verify-recipe` proposal。
- 修改无关文件：不生成强 proposal。
- 修改 ignored/generated/vendor 文件：不生成 proposal。
- 删除 sourceRef 文件：生成 `source-missing` / `propose-decay-review` proposal。
- 只有 intent 相关但无 sourceRefs：最多 weak hint，不自动写知识。

### Runtime Validation

因为用户已确认 038 要替代旧 VSCode 扩展 API 级文件监控，runtime validation 应覆盖 native watcher running 状态、ProjectScope watch roots、create / modify / delete / rename、debounce、ignored path、daemon restart 后 git reconciliation。若实现仓库 fixture 无法证明真实 watcher runtime，再交给 `AlembicTest` 做真实项目 runtime monitor 证据。

## 与其它需求的关系

| 需求 | 关系 |
| --- | --- |
| `INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28` | 038 的前置基线，定义可消费 intent / prime / sourceRefs 边界。 |
| `PLUGIN-OPPORTUNISTIC-EVOLUTION-2026-05-31` | 039，Plugin-only 无 file monitor 的机会式路径；可复用 proposal envelope，但不能复用 file monitor producer。 |
| `AI-MOCK-REMOVAL-2026-05-28` | 前置清理，避免 runtime evidence 被 AI mock 污染。 |
| Plugin cold-start / rescan 测试优化 | 可作为后续真实 Recipe evidence 的测试参考，但不属于 038 第一版完成定义。 |

## 自动化可执行边界

总控可以自动化推进：

- Stage 0 只读 inventory。
- sourceRefs impact lookup 的单元 / fixture 验证。
- proposal envelope 生成和持久化。
- negative path 测试。
- review consumer 的最小展示或报告。

必须停下确认：

- 要让 proposal 自动 submit / update / publish Recipe。
- 要把 039 Plugin host signal 合并进 038 file monitor。
- 要新增跨仓库 Core public contract。
- 要修改真实测试项目。
- 要把 full daemon runtime monitor 作为第一版必测。
- 要改变 Recipe sourceRefs 隐私暴露策略。

## 前置验证与修复项

以下项目是本需求自带 prerequisite。总控接收 038 后必须先验证，不通过就修复；不能当作后续优化延期。

| ID | 前置项 | 验证 | 修复方向 | 完成证据 |
| --- | --- | --- | --- | --- |
| `038-P0-A` | native watcher / git fallback | native watcher 是否真实存在、启动、产出 create / modify / delete / rename；无 watcher 时是否能 git diff 降级 | 补 Alembic 主体 watcher；补 watcher unavailable -> git diff fallback lifecycle | watcher running / degraded status；file-change route 收到事件；proposal 可读 |
| `038-P0-B` | capability / status | `fileMonitor.available=true` 是否真实代表 watcher running | 区分 `watcher-running`、`git-diff-degraded`、`disabled`、`unavailable` | daemon health / runtimeBoundary / Dashboard/API 显示真实状态 |
| `038-P0-C` | shared git diff evidence | Alembic 与 Plugin 的 git diff evidence 是否语义一致 | 抽出或对齐 shared evidence/scanner contract | 038 fallback 与 039 fallback 可复用同一 evidence 语义 |
| `038-P0-D` | route -> handler probe | file event / git fallback evidence 是否能进入 dispatcher / handler / proposal | 修补 producer 到 `fileChangeDispatcher` 的生产接线 | sourceRefs hit 生成 proposal；no-hit / ignored path negative 通过 |

## 完成定义

- 有真实 file monitor event 到 proposal 的最小代码链路。
- native filesystem watcher 是主事件源，git-worktree scan 是明确 degraded / reconciliation / fallback。
- `fileMonitor.available=true` 必须代表 watcher 真实 running；git fallback 单独暴露 degraded / fallback status。
- proposal 的 producer 明确为 `alembic-file-monitor`。
- changed file、matched sourceRefs、matched Recipe、intent/prime evidence、score breakdown 都可复核。
- strong proposal 必须有 changed file + sourceRefs 证据。
- no-sourceRefs / low-confidence 场景不自动写入知识。
- negative tests 证明无关 / ignored / generated / vendor 文件不会误触发。
- 不依赖 AI mock。
- 不复制 039 的 Plugin-only opportunistic path。

## 建议总控接收口径

总控可把本需求作为 `GTODO-2026-05-24-038` 的具体需求接收，但应标注依赖：

```text
depends-on:
  - INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28
  - AI-MOCK-REMOVAL-2026-05-28 boundary confirmed
sequence:
  - Stage 0 code fact inventory
  - 038 implementation / validation
  - 039 opportunistic evolution after proposal boundary is stable
```

Design 已补只读代码事实检查，可作为总控 Stage 0 的种子证据：现有 file-change route、dispatcher、handler、sourceRefs、proposal repo 和 Dashboard consumer 均有底座；第一阻塞点是 native filesystem watcher 与 `DaemonFileChangeCollector` / git fallback 的生产 lifecycle、health capability 口径，以及 intent / prime evidence enrichment 尚未接入 file-change proposal。

## 仍需确认

- Stage 0 后是否需要把 proposal envelope 下沉到 Core。
- 第一版 review consumer 是否必须进 Dashboard，还是 API / report 即可。
- intent-only weak hint 是否允许进入同一 proposal queue，还是单独 hint queue。
- `AlembicTest` 是否只在实现仓库 fixture 不能证明 native watcher runtime 时介入。
