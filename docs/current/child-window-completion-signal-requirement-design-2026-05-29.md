# Child Window Completion Signal 需求设计

Design Key：CHILD-WINDOW-COMPLETION-SIGNAL-2026-05-29
日期：2026-05-29
状态：ready-for-workspace / automation-flow-optimization
维护窗口：AlembicDesign
总控：AlembicWorkspace
原始计划：[child-window-completion-signal-original-plan-2026-05-29.md](child-window-completion-signal-original-plan-2026-05-29.md)

## 定位

本需求优化当前自动化流程中的“子窗口完成后回总控”路径。它不新增一套调度系统，而是在现有总控 / VAD / task package / handoff 机制内，增加一种更轻量的 completion signal。

目标是让子窗口用极短消息触发总控接收，例如：

```text
AlembicAgent 完成了
AlembicPlugin 阻塞了
AlembicDashboard 需要验收
```

总控收到信号后，主动读取当前计划、目标窗口回填、提交、diff、测试命令和风险，再做验收裁决。

## 最终目标

建立 `ChildWindowCompletionSignal`：

- 子窗口可以用自然语言短句声明完成 / 阻塞 / 待验收。
- 总控可以把短句解析成 window + state。
- 总控只把 signal 当作 review trigger，不当作完成事实。
- 总控负责 pull evidence、resolve active task、独立验收和决定下一步。
- 结构化 finish payload 仍可用于多任务、无人值守链式投递或复杂下一跳。

## Signal 语义

```ts
type ChildWindowCompletionSignal = {
  targetWindow: "Alembic" | "AlembicCore" | "AlembicAgent" | "AlembicDashboard" | "AlembicPlugin" | "AlembicTest";
  state: "completed" | "blocked" | "needs-review";
  taskHint?: string;
  evidenceHint?: string;
  rawMessage: string;
};
```

自然语言示例：

| 用户 / 子窗口消息 | 解析 |
| --- | --- |
| `AlembicAgent 完成了` | `{ targetWindow: "AlembicAgent", state: "completed" }` |
| `AlembicPlugin 阻塞了，缺 runtime build` | `{ targetWindow: "AlembicPlugin", state: "blocked", evidenceHint: "缺 runtime build" }` |
| `AlembicDashboard 可以验收` | `{ targetWindow: "AlembicDashboard", state: "needs-review" }` |

## Controller Pull Review

总控收到 signal 后必须执行 pull review：

1. 解析 target window 和 state。
2. 在当前计划 / dispatch queue / handoff board 中 resolve 当前窗口 active task。
3. 读取目标窗口回填文档或当前计划记录。
4. 读取相关仓库提交、diff、验证命令输出、runtime JSON、报告或截图。
5. 判断 signal 是否可信、证据是否足够、是否满足完成定义。
6. 输出 `accepted / needs-rework / blocked / no-active-task / ambiguous-task`。
7. 只有验收通过后，才允许进入下一波或关闭任务。

## 适用场景

- 一个窗口当前只有一个 active task。
- 子窗口已经把回填写入当前计划或目标回填文档。
- 总控能从 workspace 文档、git、测试输出或报告路径拉到原始证据。
- 用户在 Mac 前台手动观察到子窗口完成，只想快速提醒总控处理。
- 多 IDE / 多 LLM 场景中，外部 agent 无法调用 VAD finish 脚本，但能回一句自然语言完成信号。

## 不适用场景

- 同一窗口有多个 active task，短句无法区分。
- 子窗口没有任何回填、提交、测试或证据。
- 需要自动 chain-next / courier delivery。
- 任务需要真实 `AlembicTest` 边界判断。
- 任务涉及删除、迁移、能力降级、发布或安全风险。
- 目标窗口身份无法确认。

## 与结构化 finish 的关系

| 路径 | 用途 | 是否保留 |
| --- | --- | --- |
| 自然语言 completion signal | 人工 / 多 IDE / 简单单任务回总控。 | 新增轻量路径。 |
| `finish --json` | 结构化回填、脚本可读状态、复杂任务。 | 保留。 |
| `finish --chain-next --json` | 无人值守下一跳、courier delivery。 | 保留，但只在确实需要自动下一跳时使用。 |
| controller pull review | 总控独立验收。 | 必须保留，且是完成事实来源。 |

## 安全边界

- Signal 不等于 verdict。
- Signal 不等于 evidence。
- Signal 不等于 next action。
- Signal 不允许绕过总控验收。
- Signal 不允许目标窗口代替总控判断下一波。
- Signal 不允许没有 active task 的窗口制造新任务。
- Signal 不允许用 `完成了` 关闭证据不足的任务。

## 调度成本优化

该需求可以减少：

- 子窗口 finish prompt 长度。
- 子窗口为了回总控而重复写 AGENTS / 当前计划 / 验证边界。
- 简单任务中不必要的 chain-next JSON。
- 多 IDE 场景中因无法调用 VAD 脚本造成的阻塞。

但不会减少总控验收成本；总控仍必须读取证据。

## 分阶段建议

| 阶段 | 目标 | 主要动作 | 完成条件 |
| --- | --- | --- | --- |
| Stage 0 | 代码事实调研 | 总控调研当前 VAD finish、queue、current plan、backfill 和 controller-return 逻辑。 | 明确哪些逻辑可由 signal 触发，哪些必须保留结构化路径。 |
| Stage 1 | Signal parsing rule | 定义窗口名、状态词、taskHint / evidenceHint 的解析规则。 | `AlembicAgent 完成了` 可 resolve 为 review trigger。 |
| Stage 2 | Controller pull review | 总控收到 signal 后能找到 active task 并拉取证据。 | 证据不足时返回 needs-rework / blocked，不误判完成。 |
| Stage 3 | Prompt simplification | 子窗口派发提示词可改成“完成后回一句短 signal，证据写入原路径”。 | 简单任务 prompt 明显减少。 |
| Stage 4 | Optional integration | 视需要把 signal 记录进现有 VAD / current plan，不新增流程。 | 不破坏 structured finish。 |

## 完成定义

- 明确 natural-language completion signal 的格式和边界。
- 明确总控 pull review 是唯一验收路径。
- 简单单任务场景不再要求子窗口输出复杂 chain-next JSON。
- 结构化 finish / chain-next 仍可用于无人值守复杂场景。
- 无 active task、ambiguous task、证据不足都会停止，不会被 signal 自动关闭。
- 能支持多 IDE / 多 LLM 串行回总控。

## 与当前主线关系

- 不打断 `AI-MOCK-REMOVAL-2026-05-28`。
- 可支撑 `MULTI-LLM-DISPATCH-OPTIMIZATION-2026-05-29`。
- 可降低后续多仓库接口优化这类维护任务的调度成本。
- 是 VAD / 总控流程优化候选，不是产品功能需求。

## 建议总控下一步

总控接收后先做 Stage 0 调研，不直接改脚本：

- 当前自然语言 “X 完成了” 在总控对话中如何出现。
- 当前 active task resolution 需要哪些文档和状态。
- 当前 structured finish 哪些字段在简单任务中经常重复。
- 哪些验收证据可以由总控 pull，而不是子窗口推。

## 仍需确认

- 第一版是否只支持 `窗口名 + 完成/阻塞/待验收` 三类短句？Design 建议是。
- 是否允许用户手动发送 signal 代替子窗口脚本 finish？Design 建议允许，但只作为 review trigger。
- 是否需要把 completion signal 写入现有 VAD state？Design 建议 Stage 0 后再决定，第一版可只走总控对话解析。
