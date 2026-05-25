# <标题> Workspace Signal

日期：YYYY-MM-DD
状态：草案
来源窗口：AlembicDesign
接收窗口：AlembicWorkspace

## Signal 类型

标记一项：

- bug
- todo
- research
- decision
- current-mainline-risk
- requirement-candidate

## 触发内容

记录用户原话或准确摘要。

```text
...
```

## Design 判断

- 为什么属于该类型：
- 是否影响 AlembicWorkspace 当前主线：
- 是否建议打断当前主线：
- 是否需要用户继续确认：

## 影响范围建议

| 窗口 / 仓库 | 建议状态 | 理由 |
| --- | --- | --- |
| Alembic | 参与 / 观察 / 无任务 / 待调研 |  |
| AlembicCore | 参与 / 观察 / 无任务 / 待调研 |  |
| AlembicAgent | 参与 / 观察 / 无任务 / 待调研 |  |
| AlembicDashboard | 参与 / 观察 / 无任务 / 待调研 |  |
| AlembicPlugin | 参与 / 观察 / 无任务 / 待调研 |  |
| AlembicTest | 参与 / 观察 / 无任务 / 待调研 |  |
| AlembicDesign | 设计完成 / 继续调研 / 无任务 |  |

## 证据状态

- 用户描述：
- 截图 / 日志：
- 已知代码事实：
- 待补代码事实：
- 测试 / 复现需求：

## 建议给总控的下一步

选择一项或多项：

- 加入 `global-todo-board`。
- 作为当前 wave 的阻塞 / 返修候选。
- 创建 `AlembicTest` 测试单。
- 开启原始计划确认。
- 继续由 AlembicDesign 做需求设计。
- 等当前主线完成后再评估。
- 不入账，作为背景信息。

说明：本建议只给 `AlembicWorkspace` 评审，不是执行窗口提示词。

## TODO / Backlog 建议

| ID | 类型 | 优先级建议 | 推荐归口 | 事项 | 依赖 / 触发 |
| --- | --- | --- | --- | --- | --- |
| SIGNAL-TODO-1 | bug / todo / research / decision | P0 / P1 / P2 |  |  |  |

## 开放问题

1. ...

## 交接前自检

- 已对照 `docs/workspace-alignment-checklist.md`：
- 本 signal 没有修改 workspace 当前状态或全局 TODO：
- 本 signal 没有包含可复制实现窗口提示词：
- 推荐归口只是建议，不是派发：
- 如处于 detached-design-mode，已标注需要总控导入复核：
