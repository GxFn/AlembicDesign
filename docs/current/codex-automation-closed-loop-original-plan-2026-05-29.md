# Codex Automation Closed Loop 原始计划

Design Key：CODEX-AUTOMATION-CLOSED-LOOP-2026-05-29
日期：2026-05-29
状态：confirmed-direction / needs-workspace-rule-review
维护窗口：AlembicDesign
总控：AlembicWorkspace

## 用户原始目标

用户纠偏：本次主要优化的是 Codex 内部自动化闭环，而不是单点的 `AlembicAgent 完成了` completion signal。

截图中的 `AlembicAgent 完成了` 只是人工执行的最小示范，用来说明 Agent 有足够推理能力，不需要把人工复制提示词中的通用长规则全部搬进 automation prompt。

真正目标是梳理一个精简、恰当、可闭合的 Codex 内部自动化闭环：

```text
总控 -> 1 * N 子窗口
子窗口 -> N * 1 总控
```

总控在自动化中知道：

- 给谁发。
- 发什么任务。
- 当前任务属于哪个 current plan / task package / dispatch group。
- 目标窗口应当回填什么证据。
- 哪些信息由总控 pull review。

因此 automation prompt 不应该继续照搬人工可复制提示词的通用模式。人工提示词保持通用，是方便用户复制粘贴；自动化投递应当是针对目标窗口和任务的精简提示。

## 用户确认 / 决策

- `AlembicAgent 完成了` 不是完整机制，只是最小示范。
- 自动化闭环是 controller fan-out + child converge，不是一句自然语言 completion。
- 人工复制提示词要保持通用性。
- 自动化中总控知道目标和任务，应生成 target-specific prompt。
- 自动化 prompt 不应强制重复通用 AGENTS / current status 阅读长步骤；应按目标任务提供必要上下文和证据入口。

## 判断类型

- `new-requirement`
- `decision`
- `automation-flow-optimization`
- `codex-internal-loop`
- `requirement-candidate`

## 初步目标

建立 `CodexAutomationClosedLoop`：

```text
Controller dispatch packet
-> target-specific Codex prompt
-> target window executes bounded task
-> target returns compact result envelope
-> controller aggregates N results
-> controller pulls evidence and decides
-> controller starts next unit or stops
```

## 非目标

- 不把 completion signal 当成完整设计。
- 不删除手动通用提示词。
- 不让子窗口代替总控验收。
- 不让 target-courier 成为普通路径默认行为。
- 不绕过总控证据复核。
- 不把 AGENTS 硬规则隐藏或删除；本需求只讨论 automation prompt 是否需要每跳重复强制阅读长文本。

## 建议下一步

进入需求设计，形成精简闭环模型、保留内容、抛弃冗余内容、prompt 分层和总控接收建议。
