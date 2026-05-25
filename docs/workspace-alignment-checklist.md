# AlembicDesign 与 AlembicWorkspace 能力对齐检查

更新日期：2026-05-25

本文件用于确认 `AlembicDesign` 的文档和配置能完整承接 `AlembicWorkspace` 的需求设计前置能力。它不是执行计划，也不替代总控规则。

## 角色边界

| 总控能力 | AlembicDesign 对应产物 | 必须保留的边界 |
| --- | --- | --- |
| 识别用户目标和最终完成定义 | `original-plan`、`requirement-design`、`workspace-handoff` | 只定义目标，不宣布实现完成。 |
| 原始计划确认 | `original-plan` 的确认问题和用户确认状态 | 用户确认前不写执行阶段、不建议发送窗口。 |
| 完整功能闭环设计 | `requirement-design` 的用户场景、输入、输出、状态变化、生产方、消费方、失败路径 | 不接受空接口、空 adapter、只改类型的设计。 |
| 真实代码事实和调研缺口 | `requirement-design` 的代码事实与调研缺口，或 `workspace-handoff` 的调研请求 | 证据不足时写缺口，不编造调用链。 |
| 仓库覆盖判断 | `original-plan` 初步影响、`requirement-design` 仓库边界、`workspace-handoff` 建议覆盖 | 只能建议覆盖，不直接派发实现窗口。 |
| TODO / Backlog 账本 | `requirement-design` 和 `workspace-handoff` 的 TODO / Backlog | 设计期 TODO 不替代 workspace 全局 TODO，需总控接收后归口。 |
| bug / TODO / 调研 / 决策即时回传 | `workspace-signal` | Signal 可随时交回总控，但不能直接改 workspace 当前状态。 |
| 阶段顺序 | `requirement-design` 和 `workspace-handoff` 的阶段候选 | 候选不是 wave；最终阶段顺序由总控确认。 |
| 测试交接 | `requirement-design` 的验证策略和 `workspace-handoff` 的验证需求 | 不直接创建 `AlembicTest` 测试单，不跑真实项目测试。 |
| 当前主线保护 | `workspace-handoff` 的当前主线关系 | 不打断当前主线；是否提升为主线由总控决定。 |
| 归档和提交 | 无直接执行产物 | 不提交 `AlembicWorkspace`，不归档总控计划。 |

## 文档流

1. 想法讨论：记录目标、约束、关键分叉和确认问题。
2. 原始计划书：只框定需求，不写执行计划。
3. 用户确认：确认目标、范围、非目标和关键约束。
4. 需求设计：补完整功能闭环、仓库边界、验证策略、代码事实或调研缺口。
5. Workspace signal：讨论中出现 bug / TODO / 调研 / 决策 / 当前主线风险时，先用轻量 signal 交回总控。
6. Workspace handoff：把较完整的设计状态、建议下一步、风险和 TODO 交给 `AlembicWorkspace`。
7. 总控接收：由 `AlembicWorkspace` 决定进入 TODO、继续调研、目标阶段确认、测试单或 wave。

## Signal 必填检查

交给总控的轻量 signal 至少确认：

- 类型已经标注：`bug` / `todo` / `research` / `decision` / `current-mainline-risk` / `requirement-candidate`。
- 是否建议打断当前主线已经说明。
- 证据状态已经说明：用户描述、截图、代码证据、测试证据、待调研。
- 推荐归口是建议，不是派发。
- 下一步建议是给总控评审，不是执行窗口提示词。

## Handoff 必填检查

交给总控前，至少确认：

- 用户目标和最终完成定义已经写清。
- 当前主线关系已经标注：不影响当前主线 / TODO 候选 / 下一主线候选 / 当前主线阻塞 / 需要总控确认。
- 原始计划确认状态明确；未确认时不能建议执行。
- 需求设计状态明确；没有完整闭环时不能建议目标阶段确认。
- 已知代码事实与待调研问题分开记录。
- 仓库覆盖是建议，不是派发。
- 阶段顺序是候选，不是 wave。
- TODO / Backlog 已记录设计中发现的风险、偏好、验证缺口和后续拆分点。
- 任何删减、降级、延期、兼容保留、职责边界变化都列为待确认。
- 如果处于 `detached-design-mode`，必须提醒总控导入后重新校验当前状态。

## 禁止口径

- “可以直接发给某仓库执行。”
- “只需要做一个接口 / provider / adapter，后续再接。”
- “不影响当前目标，所以不用记录 TODO。”
- “代码事实大概是这样。”
- “总控可以跳过目标阶段确认直接 wave。”
- “这是 bug / TODO，所以我已经改了全局 TODO。”

正确口径是：`AlembicDesign` 只把讨论内容判断清楚，并给 `AlembicWorkspace` 一个可评审、可接收、可继续调研的 signal 或 handoff。
