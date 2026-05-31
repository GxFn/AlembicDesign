# Plugin Opportunistic Evolution 原始计划

Design Key：PLUGIN-OPPORTUNISTIC-EVOLUTION-2026-05-31
日期：2026-05-31
状态：ready-for-workspace / concrete-requirement / gated-by-file-monitor-boundary
维护窗口：AlembicDesign
总控：AlembicWorkspace
Source TODO：`GTODO-2026-05-24-039`
前置关联：`FILE-MONITOR-EVOLUTION-2026-05-31`

## 判断类型

- `new-requirement`
- `TODO`
- `requirement-candidate`
- `plugin`
- `knowledge-evolution`
- `opportunistic-signal`

这是把 `GTODO-2026-05-24-039` 从顺序索引落实成独立需求，不是实现派发，不是 038 的复制实现，也不是 Plugin 接管 Alembic file monitor。

## 用户原始目标

`GTODO-2026-05-24-039` 的原始描述是：

```text
Plugin 无 file monitor 时的机会式知识进化，也等待 037 的意图同步设计。
```

结合用户对 Plugin / Alembic 主体关系的多次纠偏，本需求的真实目标是：

当用户只在 Codex + AlembicPlugin 环境中工作，且没有 Alembic daemon file monitor 可用时，Plugin 仍可以基于当前 host-agent 可见信号，发现知识可能缺失、过期、冲突或需要补证的机会，并产出可审查的 opportunistic evolution proposal。

第一版目标不是让 Plugin 模拟 file monitor，也不是后台静默改 Recipe，而是把 Codex 当前任务里已经可见的 intent、active files、diff、Guard/search/submit/prime 结果整理成轻量 proposal / hint。

## 与 038 的关系

039 排在 038 之后，是因为它最好复用 038 稳定下来的 proposal envelope、evidence terminology 和 review consumer。但 039 与 038 的事实来源不同：

| 维度 | 038 File Monitor Evolution | 039 Plugin Opportunistic Evolution |
| --- | --- | --- |
| producer | Alembic daemon / resident file monitor | AlembicPlugin in Codex host-agent context |
| 事实来源 | 真实 file change event | host-agent visible signals |
| 文件变化证据 | changed file event / batch | active file / visible diff / tool result，仅为可见信号 |
| 输出 | file-monitor evolution proposal | opportunistic evolution proposal |
| 禁止 | Plugin host signal 冒充 file event | 模拟 daemon file monitor / 背景监听 |

039 可以复用 038 的 review envelope，但必须标注 `producerKind: "plugin-opportunistic"`，不能写成 `alembic-file-monitor`。

## 最终目标

建立 Plugin-only 的机会式知识进化线索闭环：

```text
Codex host-agent turn
-> hostDeclaredIntent / IntentExtractionFrame
-> IntentEpisode
-> PrimeInjectionPackage
-> visible active files / diff / tool results
-> Guard / search / submit outcome
-> sourceRefs / Recipe lookup
-> OpportunisticEvolutionProposal
-> review / next action suggestion
```

它应能回答：

- 当前 Codex 任务是否暴露了知识缺失、过期、冲突或遗漏信号。
- 这些信号来自 intent、Guard、search、submit、prime omitted、visible diff 还是用户纠错。
- 关联到哪些 Recipe / Guard / sourceRefs。
- 为什么建议 review，或为什么低置信度不建议注入 / 不建议写知识。
- 后续应该由用户确认、Codex 继续整理、还是等待 Alembic daemon / rescan。

## 第一版范围

第一版建议只做 proposal / hint，不自动创建 pending candidate，不自动调用 `submit_knowledge`，不自动修改 Recipe。

可以触发 proposal 的信号：

- 用户明确纠正 Recipe / Guard / 工作流规则。
- `search` 找不到明显应存在的知识，且 intent / active file / sourceRefs 能支持缺口判断。
- `Guard` 发现与现有 Recipe 相关的冲突或过期证据。
- `submit_knowledge` 被拒绝、合并、supersede 或缺 sourceRefs，显示已有知识需要整理。
- `PrimeInjectionPackage.omitted` 反复遗漏与当前任务相关的知识。
- Codex 可见 diff / active file 与某个 Recipe sourceRefs 强相关，但没有 daemon file monitor。

不能触发强 proposal 的信号：

- 普通闲聊。
- 没有 ProjectScope / sourceRefs / tool evidence 的单句想法。
- AI mock output。
- 未经确认的本地文件事件。
- Plugin 无法看到的后台文件变化。

## 真实使用场景

1. 用户只安装 AlembicPlugin，通过 Codex 处理某个项目。
2. `prime` 或 `search` 没有注入应当出现的 Recipe。
3. Codex 在当前任务里打开了相关文件或产生 diff，并通过 Guard / submit / search 得到证据。
4. Plugin 根据当前 intent episode、visible file refs、sourceRefs 和 tool outcome 判断存在知识进化机会。
5. Plugin 产出 opportunistic proposal，建议用户或后续窗口 review。
6. 若用户确认或总控后续接收，再进入正式 submit / evolve / rescan 流程。

## 非目标

- 不实现 file monitor。
- 不模拟 daemon / resident watcher。
- 不监听 Plugin 不可见的后台文件变化。
- 不静默写入、发布或 supersede Recipe。
- 不绕过 ProjectScope 和 sourceRefs。
- 不把普通对话都变成知识进化。
- 不让 039 先于 038 定义一套不兼容的 proposal contract。
- 不依赖 AI mock。

## 建议总控 Stage 0

总控接收后先做代码事实 inventory：

- Plugin 当前 prime / search / Guard / submit / record decision 可拿到哪些 host context。
- `hostDeclaredIntent`、`hostTurnMeta`、session、active file、sourceRefs、tool result 在 Plugin 中如何传递。
- `PrimeInjectionPackage` 和 `IntentEpisode` 当前是否可被 039 读取。
- Plugin 是否已有 low-confidence / omitted / no-result / consolidation proposal 的记录点。
- 038 的 proposal envelope 是否已经稳定，039 是否直接复用或通过 adapter 输出。
- Proposal 结果应该返回给 MCP tool、写入 local data root、还是只作为 nextAction / report。

## 建议阶段

| 阶段 | 名称 | 目标 |
| --- | --- | --- |
| Stage 0 | Host signal inventory | 明确 Plugin 可见信号、不可见信号、sourceRefs 入口和 038 envelope 依赖。 |
| Stage 1 | Signal normalization | 把 intent、prime、search、Guard、submit、active file / visible diff 标准化为 evidence。 |
| Stage 2 | Evidence gate | 建立 no-evidence no-proposal、low-confidence degradation、ProjectScope / sourceRefs gate。 |
| Stage 3 | Proposal adapter | 输出 `OpportunisticEvolutionProposal`，与 038 review consumer 对齐但 producer 不同。 |
| Stage 4 | Tool result surfacing | 在 prime/search/Guard/submit 的合适返回中给出 review hint / next action。 |
| Stage 5 | Targeted validation | 用 Plugin fixture 验证 strong signal 生成 proposal，ordinary chat 不生成。 |

## 完成定义

- Plugin-only 场景不需要 daemon file monitor 也能根据 host-visible evidence 生成 proposal。
- proposal 的 producer 明确为 `plugin-opportunistic`。
- proposal 必须包含 intent / tool / sourceRefs / visible file evidence，不能只有自然语言猜测。
- 普通对话、无 sourceRefs、无 ProjectScope、低置信度场景不生成强 proposal。
- 不自动 submit / publish / mutate Recipe。
- 与 038 的 proposal envelope 或 review consumer 兼容，但事实来源清楚区分。
- 测试证明 039 没有模拟 file monitor，也没有消费不可见本地文件事件。

## 建议总控下一步

总控可把本需求作为 `GTODO-2026-05-24-039` 的具体需求接收，但建议排在 038 Stage 0 或 proposal envelope 边界稳定之后。

如果总控急需并行推进，039 也必须先做 Stage 0 host signal inventory，不进入实现；等 038 的 proposal / review boundary 明确后，再决定是否复用同一 envelope。

## 仍需确认的问题

- 039 第一版是否严格只做 proposal / hint，不创建 pending candidate？Design 建议是。
- 039 是否必须等待 038 proposal envelope 稳定后实现？Design 建议至少等待 038 Stage 0 和 envelope 裁决。
- proposal 应该在 prime、search、Guard、submit 哪些 tool result 中 surfaced？
- Plugin 是否允许记录本地 opportunistic proposal，还是只返回给当前 Codex turn？
- 低置信度但频繁出现的 omitted knowledge 是否进入 proposal，还是只进入 telemetry / research？
