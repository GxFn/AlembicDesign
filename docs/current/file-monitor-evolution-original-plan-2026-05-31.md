# File Monitor Evolution 原始计划

Design Key：FILE-MONITOR-EVOLUTION-2026-05-31
日期：2026-05-31
状态：ready-for-workspace / concrete-requirement / gated-by-intent-baseline
维护窗口：AlembicDesign
总控：AlembicWorkspace
Source TODO：`GTODO-2026-05-24-038`
前置基线：`INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28`

## 判断类型

- `new-requirement`
- `TODO`
- `requirement-candidate`
- `knowledge-evolution`
- `file-monitor`
- `current-mainline-independent`

这是把 `GTODO-2026-05-24-038` 从顺序索引落实成独立需求，不是实现派发，不是 bug 回填，也不是 workspace 当前状态更新。

## 用户原始目标

`GTODO-2026-05-24-038` 的原始描述是：

```text
Alembic file monitor evolution，依赖/需要对齐 037。
```

结合后续 intent knowledge route 讨论，本需求的真实目标是：

让 Alembic 本地增强底座在具备真实 file monitor 的环境中，能够把项目文件变化、Recipe / Guard 的 `sourceRefs`、最近的 `IntentEpisode`、`PrimeInjectionPackage` 和检索/关联证据汇总起来，产出可审查的知识进化 proposal。

第一版目标不是静默修改 Recipe，也不是自动发布知识，而是把“文件变化可能影响哪些知识”形成可解释、可复核、可由总控或后续窗口处理的进化候选。

## 背景与前置事实

可以继承的前置事实来自 `INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28`：

- `RecognizedIntentDraft` / `IntentExtractionFrame` 可表达用户目标、约束、非目标和置信度。
- `IntentEpisode` 可表达跨会话任务连续、correction、supersede 和近期上下文。
- `IntentSearchPlan` 可表达意图到检索计划的转换。
- `PrimeInjectionPackage` 可汇总 intent、search、vector、relations、selectedKnowledge、omitted 和 injection summary。
- Recipe / Guard 的 `sourcePath` / `sourceRefs` 仍是关键证据，不能丢失。
- 低置信度时允许不强行注入或不生成强 proposal。

这些事实不能推出 file monitor evolution 已完成。038 仍需要总控在接收后复核真实 Alembic 代码：当前 file monitor 的生产方、ProjectScope、Recipe sourceRefs 存储、proposal 持久化、Dashboard / API 消费和 rescan / audit 的关系。

## 最终目标

建立 Alembic file monitor 到知识进化 proposal 的最小闭环：

```text
real file change
-> ProjectScope / ignore boundary
-> debounce / batch
-> changed file classification
-> Recipe / Guard sourceRefs impact lookup
-> IntentEpisode / PrimeInjectionPackage enrichment
-> evidence scoring
-> FileMonitorEvolutionProposal
-> review / dashboard / log / queue consumer
```

最终交付应能回答：

- 哪些文件变化与已有 Recipe / Guard 源证据有关。
- 这次变化和最近用户任务、纠错或 supersede 是否有关。
- 哪些 knowledge 可能 stale、需要补充、需要验证或需要废弃。
- 为什么生成 proposal，或为什么忽略 / 降级。
- proposal 的证据来自哪些文件、source refs、intent episode、prime package、search/vector/relation 结果。

## 第一版范围

第一版只做 file monitor evolution proposal，不做自动知识写入。

建议最小能力：

- 只处理 ProjectScope 内真实文件事件。
- 只对能关联到 Recipe / Guard `sourceRefs` 的变化生成强 proposal。
- 对没有 source refs 但与当前 `IntentEpisode` 高相关的变化，只能生成弱 hint 或不生成。
- 保留 changed file path、matched sourceRefs、matched Recipe ids、evidence breakdown、confidence 和 proposed action。
- proposal 默认进入 `pending-review`，不自动调用 `submit_knowledge`、不自动更新 Recipe、不自动发布。

## 真实使用场景

1. 用户修改一个被 Recipe `sourceRefs` 引用的源文件。
2. Alembic daemon / resident local service 捕获文件事件。
3. 系统在 debounce 后把变更聚合为 file change batch。
4. 系统查找受影响 Recipe / Guard，并读取最近的 intent episode / prime package。
5. 系统判断现有知识可能 stale 或需要补证，生成 proposal。
6. Dashboard、API、日志或 Codex review 流程展示该 proposal。
7. 后续由用户、总控或执行窗口决定是否进入 submit / evolve / decay。

## 非目标

- 不复活旧 VSCode 插件 file monitor 模式。
- 不让 AlembicPlugin 代替 Alembic daemon 做真实本地文件监控。
- 不把 Plugin host-only signal 伪装成 file event。
- 不自动修改、删除、发布或 supersede Recipe。
- 不绕过 ProjectScope、ignore rules、sourceRefs 和 evidence boundary。
- 不把所有文件变化都当作知识进化信号。
- 不使用 AI mock provider 输出作为 runtime evidence。
- 不把 038 和 039 混成同一条 fallback 逻辑。

## 建议总控 Stage 0

总控接收后不要直接派实现，先做代码事实 inventory：

- Alembic 当前是否已有 file monitor / watch service / daemon event source。
- ProjectScope、workspace root、多 folder、ignore / generated / vendor 文件规则在哪里。
- Recipe / Guard 的 `sourcePath` / `sourceRefs` 当前存储与查询方式。
- 是否已有 recipe audit、source ref audit、stale evidence、decay / evolve / proposal 数据结构。
- rescan / file monitor / dashboard timeline 是否已有共享事件或 queue。
- 当前 Dashboard / API 是否有可复用 proposal 展示面。
- 038 是否需要 Core typed contract，还是先在 Alembic 本地增强底座内闭环。

Stage 0 产物应是代码事实和实施边界，不应立即改代码。

## 建议阶段

| 阶段 | 名称 | 目标 |
| --- | --- | --- |
| Stage 0 | 代码事实 inventory | 找到真实 watcher、ProjectScope、sourceRefs、audit/proposal consumer 和 rescan 关系。 |
| Stage 1 | File event gate | 建立 ProjectScope、ignore、debounce、batch 和 event classification 的最小路径。 |
| Stage 2 | SourceRefs impact lookup | 文件变化能映射到受影响 Recipe / Guard，保留源路径证据。 |
| Stage 3 | Intent evidence enrichment | 引入 `IntentEpisode` / `PrimeInjectionPackage` / search relation 作为解释和置信度补充。 |
| Stage 4 | Proposal envelope | 生成 `FileMonitorEvolutionProposal`，进入 review 状态，不自动写知识。 |
| Stage 5 | Targeted validation | 用 fixture project 验证相关文件生成 proposal，无关文件不生成，生成文件 / vendor 被忽略。 |
| Stage 6 | Runtime validation | 如需真实 daemon / file monitor 证据，再交由总控判断是否启用 `AlembicTest`。 |

## 完成定义

- 对 ProjectScope 内、被 Recipe `sourceRefs` 引用的文件变更，能生成 reviewable proposal。
- proposal 必须包含 changed files、matched sourceRefs、matched Recipe ids、intent / prime evidence refs、confidence、why selected / why degraded。
- 无关文件、ignored 文件、generated/vendor 文件不会生成强 proposal。
- 无 sourceRefs 或低置信度时不会自动进入知识写入，只能降级为 weak hint 或 no-op。
- proposal 不自动调用 `submit_knowledge`，不自动修改 Recipe。
- 测试覆盖 positive、negative、ignored、low-confidence 和 no-sourceRefs 路径。
- 文档和代码证据明确区分 038 file monitor 与 039 Plugin opportunistic evolution。

## 建议总控下一步

接收本需求后，先确认 `INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28` 是否已经被总控接收并形成可消费 baseline。若 baseline 尚未完成，038 可以进入收件箱，但不应直接派实现。

可执行的第一步是 Stage 0 只读 inventory，目标是确认真实 watcher / sourceRefs / proposal consumer 的位置和缺口，再决定是否拆 Alembic / AlembicCore / AlembicDashboard 任务。

## 仍需确认的问题

- 当前 Alembic 真实 file monitor producer 是 daemon、resident service，还是仍需补生产方？
- proposal consumer 第一版放在哪里：Dashboard、API、日志、queue，还是 Codex review 包？
- `FileMonitorEvolutionProposal` 是否需要下沉到 Core contract，还是先作为 Alembic 本地结构？
- 第一版是否只支持 sourceRefs 命中文件，还是允许 intent 高相关但无 sourceRefs 的 weak hint？Design 建议允许 weak hint，但不能强 proposal。
- 真实 runtime file monitor 验证是否需要 `AlembicTest`，还是 targeted fixture 足够第一版验收？
