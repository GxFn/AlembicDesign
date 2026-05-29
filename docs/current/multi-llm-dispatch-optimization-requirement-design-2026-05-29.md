# Multi LLM Dispatch Optimization 需求设计

Design Key：MULTI-LLM-DISPATCH-OPTIMIZATION-2026-05-29
日期：2026-05-29
状态：draft / needs-design
维护窗口：AlembicDesign
总控：AlembicWorkspace
原始计划：[multi-llm-dispatch-optimization-original-plan-2026-05-29.md](multi-llm-dispatch-optimization-original-plan-2026-05-29.md)

## 定位

本需求不是替换现有 VAD，也不是新增某个 IDE 插件。它要把已经成立的事实正式设计清楚：

提示词任务包本身已经是一种 host-agnostic 协作协议。Codex、Claude Code、其它 IDE agent、终端 agent 或人工复制，都可以通过同一任务包与回填包参与多 LLM 分工。

当前需要优化的是调度成本、协议复用和回填可验收性。

## 最终目标

建立一套多 LLM / 多 IDE 可复用的轻量调度协议：

- 任务包不绑定 Codex。
- 回填包不绑定 Codex。
- Delivery channel 可替换。
- 总控仍统一验收原始证据。
- 调度提示词的重复成本可度量、可压缩。

## 抽象模型

```text
Workspace Task Envelope
-> Delivery Channel
-> Agent / IDE Execution
-> Backfill Envelope
-> Controller Review
```

### Workspace Task Envelope

任务包必须表达：

- task id / design key / source TODO。
- target repository / target window / role boundary。
- user goal and completion definition。
- allowed changes / forbidden changes。
- required evidence。
- verification commands.
- backfill schema.
- stop conditions.

### Delivery Channel

delivery channel 只负责投递，不改变任务语义：

- Codex visible dispatch / heartbeat automation。
- 手动复制提示词。
- Claude Code session。
- 其它 IDE agent。
- 终端 agent。

第一版只要求手动复制和 Codex VAD 都能消费同一 envelope，不要求打通外部 IDE 自动化 API。

### Backfill Envelope

回填必须表达：

- task id。
- repository。
- commit hash or no-code-change reason。
- changed files summary。
- command outputs summary。
- runtime JSON / report / screenshot / artifact evidence。
- unresolved risks。
- next suggested action。

## 调度浪费指标

为了优化调度成本，建议记录以下指标：

| 指标 | 含义 | 用途 |
| --- | --- | --- |
| `promptTokenEstimate` | 派发提示词估算 token。 | 观察任务协议是否过长。 |
| `repeatedInstructionRatio` | AGENTS / 禁止事项 / 回填格式等重复内容占比。 | 判断哪些内容可模板化。 |
| `manualCopyCount` | 用户或总控手动复制次数。 | 衡量跨 IDE 串行成本。 |
| `handoffRepairCount` | 因提示词不清导致返工 / 重新派发次数。 | 衡量任务 envelope 清晰度。 |
| `backfillAcceptanceRate` | 回填一次通过率。 | 衡量回填 schema 是否可验收。 |
| `blockedByHostCapability` | 因宿主工具能力不足而阻塞次数。 | 区分协议问题和 IDE 能力问题。 |

## 设计原则

- 总控协议 host-agnostic，delivery adapter host-specific。
- 自动化能力是增强，不是前提。
- Prompt task envelope 是第一等产物，不是临时文字。
- 回填证据比执行者身份更重要。
- 不同 LLM 可以串行或并行协作，但总控必须保留最终裁决。
- 任何 agent 不能因为拿到任务包就获得其它窗口 / 仓库职责。

## 可先做的轻量优化

- 把可复制提示词拆成固定 envelope 段落，减少每次重新写。
- 把常见回填格式模板化。
- 把 AGENTS / 仓库定位 / 禁止事项变成引用式检查，而不是每次长篇重复。
- 对不同 delivery channel 标注能力：`manual`、`codex-heartbeat`、`codex-visible-thread`、`claude-code-manual`。
- 为任务包加 `hostAssumptions`，明确哪些步骤依赖 Codex tool，哪些只依赖 shell / git / file access。

## 停止条件

- 任务需要真实 Codex thread id，但目标 channel 不是 Codex。
- 任务需要 MCP tool，但目标 agent 没有对应能力。
- 任务需要 GUI / Browser / automation API，但目标 IDE 不支持。
- 回填不能提供原始证据。
- 多 LLM 对同一文件并行修改产生冲突，必须回总控裁决。

## 与当前主线关系

- 不打断 `AI-MOCK-REMOVAL-2026-05-28`。
- 不替代 VAD。
- 可支撑 `MULTI-REPOSITORY-INTERFACE-OPTIMIZATION-2026-05-28` 这类维护型需求后续跨 IDE 执行。
- 可作为后续 automation governance / dispatch overhead optimization 需求候选。

## 建议总控下一步

先作为 Design 草案保留，不急于实现。

如果总控接收，Stage 0 应只做现有任务包 / 回填包样本调研：

- 现有可复制提示词的重复段落。
- VAD queue / backfill 字段。
- 哪些字段是 Codex-specific。
- 哪些字段可抽象为 host-agnostic envelope。
- 调度浪费指标能否从现有文档和脚本中估算。

## 仍需确认

- 第一版是否只支持手动复制 + Codex VAD 两类 delivery channel？Design 建议是。
- 是否需要把 Claude Code 作为明确目标 channel，还是先只定义 generic manual IDE agent？Design 建议先 generic。
- 调度浪费指标是否进入 PCV / metrics 体系，还是先作为 workspace governance 指标？Design 建议先放 workspace governance。
