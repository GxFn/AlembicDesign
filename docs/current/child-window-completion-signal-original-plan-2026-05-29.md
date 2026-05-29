# Child Window Completion Signal 原始计划

Design Key：CHILD-WINDOW-COMPLETION-SIGNAL-2026-05-29
日期：2026-05-29
状态：ready-for-workspace / automation-flow-optimization
维护窗口：AlembicDesign
总控：AlembicWorkspace
来源：用户新增自动化流程优化需求

## 用户原始目标

优化当前自动化流程中“子窗口把任务传回总控”的路径。用户观察到，实际人工流程里只需要像截图中一样发送一句：

```text
AlembicAgent 完成了
```

总控就可以开始接收处理：读取当前计划、目标窗口回填、提交、diff、测试命令和遗留风险，再自行判断是否验收或进入下一波。

因此，子窗口回总控不一定需要写很多复杂的 chain-next / finish JSON / 调度逻辑。可以把它简化成轻量 completion signal，由总控负责 pull evidence 和裁决。

## 判断类型

- `new-requirement`
- `automation-flow-optimization`
- `dispatch-simplification`
- `controller-pull-review`

## 初步目标

建立 `ChildWindowCompletionSignal` 机制：

```text
child window says "AlembicAgent 完成了"
-> controller receives completion signal
-> controller resolves active task
-> controller pulls evidence
-> controller reviews and decides accepted / blocked / needs-rework / next-wave
```

## 核心原则

- completion signal 只是通知，不是验收结论。
- 子窗口不需要承担总控裁决。
- 总控负责读取原始证据并独立验收。
- 简单场景允许自然语言短句，不强制 JSON。
- 复杂场景仍允许结构化 finish payload。

## 非目标

- 不替换 VAD。
- 不删除现有 `finish --chain-next --json` 等结构化路径。
- 不让“完成了”直接关闭任务。
- 不让子窗口决定下一波派发。
- 不绕过提交、diff、测试、runtime JSON、日志、报告等证据复核。

## 建议下一步

交给总控作为自动化流程优化需求接收。总控先做代码事实调研：当前 VAD / task package / finish / backfill / controller-return 里哪些逻辑可以由 completion signal 触发，哪些仍必须保留结构化路径。
