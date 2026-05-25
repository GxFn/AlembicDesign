# Cold-start Stage Optimization Metrics Workspace Signal

日期：2026-05-25
状态：已被 original-plan 承接
来源窗口：AlembicDesign
接收窗口：AlembicWorkspace

承接文档：[progressive-chain-validation-metrics-original-plan-2026-05-25.md](progressive-chain-validation-metrics-original-plan-2026-05-25.md)

## Signal 类型

- requirement-candidate
- research
- decision

## 触发内容

用户希望继续讨论 Agent 各阶段指标，并在 cold-start 的小阶段定义优化指标，让这个数值随着优化逐步变小或变好。

```text
我想继续讨论关于 Agent 的各个阶段指标，我希望能在冷启动的小阶段定义一个优化指标，然后让这个数值慢慢变小或更好
```

后续澄清：用户明确指向 `LLM Input Optimization Research` 中的 `progressive-chain-validation`，希望补完整该 skill，用它支持确定指标的前后对比，再用对比结果优化 Alembic 内部 Agent 能力。

## Design 判断

- 为什么属于该类型：该想法可发展为独立需求主线，目标是把 cold-start / rescan 拆成小阶段，并为每个小阶段建立可比较、可回归、可持续优化的指标闭环；同时需要补真实代码事实和 metrics / trace 证据口径。
- 是否影响 AlembicWorkspace 当前主线：不应打断当前 `LLM 输入优化 Wave 5`。它与当前主线的 artifact / trace / metrics / Dashboard 可视化结果强相关，适合作为当前主线完成后的下一主线候选或补充调研。
- 是否建议打断当前主线：否。当前应先完成 Timeline artifact detail、metrics / trace 展示和后续 integration verification，再基于已有观测数据定义阶段优化指标。
- 是否需要用户继续确认：需要。核心确认点是指标更偏“质量门槛下的成本下降”，还是希望为每个小阶段定义更强的任务质量分数。

## 初步设计判断

建议不要只定义一个会单纯变小的 raw 指标，例如 token 数或耗时。单一成本指标容易诱导 Agent 少读、少查、少产证据，反而破坏冷启动质量。

更稳的口径是：

1. 先给每个小阶段定义质量门槛。
2. 在质量门槛满足的前提下，优化一个可下降的 `stage loss`。
3. `stage loss` 同时记录成本、延迟、重试、冗余、无效上下文和失败惩罚。
4. 每个小阶段再定义自己的 `useful unit`，避免把不同阶段硬压成同一种质量分。

一个可讨论的默认公式：

```text
stage_loss =
  normalized_cost
  + normalized_latency
  + retry_penalty
  + redundancy_penalty
  + invalid_output_penalty
  + missing_required_evidence_penalty
```

质量门槛单独判断，不直接被成本抵消：

```text
stage_is_optimizable =
  required_outputs_present
  && evidence_floor_met
  && no_regression_in_downstream_acceptance
```

也就是说，目标不是“让输入越短越好”，而是“在不降低阶段产物可用性的情况下，让单位有效产物的浪费越来越少”。

## 可能的小阶段指标草案

| Cold-start 小阶段 | useful unit 候选 | 可下降指标候选 | 质量门槛候选 |
| --- | --- | --- | --- |
| 项目识别 / scope resolution | 正确 project / folder / dataRoot 解析 | resolve latency、fallback count、重复解析次数 | projectScopeId / controlRoot / folders[] 正确且无 source folder 写入 |
| 工程扫描 / file selection | 后续实际使用的文件 / symbol / evidence | 扫描文件数、重复文件数、无用候选比例 | 覆盖入口、配置、测试和关键模块 |
| LLM input assembly | 被 provider 消费且后续有用的 section | prompt token、重复 section、raw dump 比例 | stage profile 正确、required sections 齐全 |
| tool planning / tool call | 成功产出 evidence 的 tool call | failed call、retry、无效 call、重复 read/search | toolChoice 与阶段策略一致 |
| Observation Ledger | 下游复用的 confirmed observation | 重复 observation、无证据 observation、debug 字段泄露 | ledger 可读、去敏、保留关键发现 |
| candidate / Recipe synthesis | 通过验证的候选知识 | candidate redundancy、validation failure | candidate 有触发条件、证据和边界 |
| consolidation / writeback | 被接受或演进的知识项 | merge churn、重复 proposal、无效 evolution | 不覆盖真实项目，不写错 dataRoot |

## 影响范围建议

| 窗口 / 仓库 | 建议状态 | 理由 |
| --- | --- | --- |
| Alembic | 待调研 / 参与 | 需要确认 daemon job artifact、trace envelope、metrics、job stage events 和 read API 是否足够承载阶段指标。 |
| AlembicCore | 观察 / 待调研 | 若指标 schema 或 ProjectScope / evidenceRef 成为共享 contract，可能需要 Core；初期不应先下沉。 |
| AlembicAgent | 参与 | Agent runtime 是阶段划分、LLM input assembly、tool planning、Observation Ledger 和 useful unit 的主要生产方。 |
| AlembicDashboard | 观察 / 参与 | 当前 Wave 5 正在消费 metrics / trace；后续可能需要趋势图、stage detail 和 baseline 对比 UI。 |
| AlembicPlugin | 观察 | 本议题优先针对 Alembic internal Agent cold-start，不改变 Codex host-agent 路由；若后续要看 Plugin prime/search 指标再纳入。 |
| AlembicTest | 参与 | 需要 test-mode 或真实项目 baseline 证明同一小阶段指标可重复采集、可比较、优化后无质量回退。 |
| AlembicDesign | 继续调研 | 适合继续沉淀 original-plan 或 requirement-design，明确指标语义、阶段边界和完成定义。 |

## 证据状态

- 用户描述：已有，明确希望在 cold-start 小阶段定义可持续优化的指标。
- 截图 / 日志：暂无。
- 已知代码事实：父级当前状态显示 `LLM 输入优化 Wave 4` 已生产 artifact / trace / metrics，`Wave 5` 正由 Dashboard 消费；具体字段和阶段事件仍需源码复核。
- 待补代码事实：
  - Agent 当前 cold-start / rescan stage 划分、stage profile 和 process event 名称。
  - `llmMetrics`、`traceEnvelope`、`artifactRefs` 当前字段是否足以按小阶段聚合。
  - Dashboard 是否已有 stage timeline / job detail 可挂载趋势和 loss 分解。
  - AlembicTest 是否已有可重复 baseline fixture，能对比同一阶段优化前后。
- 测试 / 复现需求：需要后续由总控决定是否创建 AlembicTest baseline 测试单；Design 不直接创建测试单。

## 建议给总控的下一步

- 等当前 `LLM 输入优化 Wave 5` 和后续 integration verification 完成后评估。
- 将本 signal 与现有 `GTODO-2026-05-25-003` 的 progressive-chain-validation 思路合并评审。
- 若用户继续确认该方向，建议由 AlembicDesign 先起草原始计划书，主题可为 `progressive-chain-validation-metrics`。2026-05-25 已起草：[progressive-chain-validation-metrics-original-plan-2026-05-25.md](progressive-chain-validation-metrics-original-plan-2026-05-25.md)。
- 进入原始计划前，建议总控先做或分派代码事实调研，确认当前 artifact / trace / metrics 是否已经能支撑 stage loss baseline。

说明：本建议只给 `AlembicWorkspace` 评审，不是执行窗口提示词。

## TODO / Backlog 建议

| ID | 类型 | 优先级建议 | 推荐归口 | 事项 | 依赖 / 触发 |
| --- | --- | --- | --- | --- | --- |
| SIGNAL-TODO-1 | research | P1 | AlembicDesign / AlembicWorkspace | 定义 cold-start 小阶段列表、每阶段 useful unit、质量门槛和可下降 `stage_loss` 草案。 | 用户继续确认指标方向。 |
| SIGNAL-TODO-2 | research | P1 | Alembic / AlembicAgent | 调研当前 artifact / trace / metrics / stage events 是否足以产出 baseline。 | Wave 5 / integration verification 完成后。 |
| SIGNAL-TODO-3 | todo | P1 | AlembicTest | 准备可重复 baseline：同一 cold-start / rescan 阶段的 loss、质量门槛和下游 acceptance 对比。 | 总控创建测试单后。 |

## 开放问题

1. 用户更希望默认优化目标是“质量门槛下的成本 / 冗余下降”，还是“阶段质量分数上升”，或者两者都要但分开展示？
2. 阶段指标是否需要先覆盖 cold-start，rescan 只作为后续扩展？
3. 是否允许每个小阶段使用不同 useful unit，还是必须汇总成一个顶层总分？
4. 趋势展示更偏开发者调优面板，还是也要进入普通 Dashboard Timeline？

## 交接前自检

- 已对照 `docs/workspace-alignment-checklist.md`：是。
- 本 signal 没有修改 workspace 当前状态或全局 TODO：是。
- 本 signal 没有包含可复制实现窗口提示词：是。
- 推荐归口只是建议，不是派发：是。
- 如处于 detached-design-mode，已标注需要总控导入复核：不适用，当前可读取父级 workspace 文档。
