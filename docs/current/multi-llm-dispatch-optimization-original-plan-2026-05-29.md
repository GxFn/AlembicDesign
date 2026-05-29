# Multi LLM Dispatch Optimization 原始计划

Design Key：MULTI-LLM-DISPATCH-OPTIMIZATION-2026-05-29
日期：2026-05-29
状态：draft / requirement-candidate
维护窗口：AlembicDesign
总控：AlembicWorkspace
来源：用户新增架构洞察

## 用户原始目标

用户观察到：仅用提示词任务包，就已经完成了多 LLM 同时分工合作与无人值守自动化。代价是总控需要写大量调度提示词、任务边界、回填要求和验收逻辑，产生较多调度逻辑浪费。

用户希望把它定义为一个支持多 LLM 与调度优化的需求。

## 判断类型

- `requirement-candidate`
- `architecture insight`
- `automation-governance`
- `multi-llm-collaboration`
- `dispatch-optimization`

## 初步目标

把当前 Codex-oriented 的可复制提示词 / VAD 任务包，抽象成 host-agnostic 的任务协议：

```text
Workspace Task Envelope
-> Delivery Channel
-> Agent / IDE Execution
-> Backfill Envelope
-> Controller Review
```

Codex heartbeat automation 只是 delivery channel 之一；Claude Code、其它 IDE agent、终端 agent 或人工复制提示词都可以串行接入。

## 核心问题

- 当前多 LLM 协作已经可行，但靠自然语言重复描述任务边界。
- 不同 LLM / IDE 的能力不同，不能假设都有 Codex thread id、heartbeat automation 或 MCP tool。
- 总控为了安全，需要重复写 AGENTS 读取、仓库定位、禁止事项、证据格式和回填格式。
- 调度成本需要被看见：token、提示词长度、重复字段、handoff 失败、回填不可验收、人工介入次数。

## 非目标

- 不替换现有 VAD。
- 不要求所有 IDE 都支持自动化唤醒。
- 不在第一版实现跨 IDE API 或远程控制。
- 不让外部 LLM 绕过总控验收。
- 不把任务协议变成新执行窗口职责。

## 建议下一步

形成需求设计，定义 task envelope / backfill envelope / delivery adapter / scheduling overhead metrics。该需求应作为 VAD 和多仓库维护任务的后续治理候选，不打断当前 AI mock 删除、baseline、038/039。
