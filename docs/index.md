# AlembicDesign 文档索引

更新日期：2026-05-25

本索引是 AlembicDesign 的文档地图。当前设计草案放在 `docs/current/`；稳定运行规则放在 `docs/`；可复用模板放在 `templates/`。

## 当前地图

| 类型 | 文档 | 状态 | 说明 |
| --- | --- | --- | --- |
| 运行规则 | [design-window-operating-policy.md](design-window-operating-policy.md) | 已生效 | 定义 AlembicDesign 如何与 AlembicWorkspace 协作。 |
| 总控能力对齐检查 | [workspace-alignment-checklist.md](workspace-alignment-checklist.md) | 已生效 | 将总控需求设计能力逐项映射到 AlembicDesign 的文档产物和禁止边界。 |
| 当前草案 | [current/README.md](current/README.md) | 当前说明 | 保存活跃原始计划、需求设计、signal、handoff 和 handoff board。 |
| Workspace Handoff Board | [current/workspace-handoff-board.md](current/workspace-handoff-board.md) | 维护中 | Design 正式需求完成后的交接清单；总控脚本从这里生成 workspace 收件箱。 |
| 原始计划书模板 | [../templates/original-plan-template.md](../templates/original-plan-template.md) | 已生效 | 用于用户确认前的第一轮需求框定。 |
| 需求设计模板 | [../templates/requirement-design-template.md](../templates/requirement-design-template.md) | 已生效 | 用于原始计划确认后的详细需求设计。 |
| Workspace Signal 模板 | [../templates/workspace-signal-template.md](../templates/workspace-signal-template.md) | 已生效 | 用于把 bug / TODO / 调研 / 决策 / 当前主线风险轻量交回总控。 |
| Workspace 交接模板 | [../templates/workspace-handoff-template.md](../templates/workspace-handoff-template.md) | 已生效 | 用于交回 AlembicWorkspace 的交接草案。 |

## 文档落点

- `docs/current/<topic>-original-plan-YYYY-MM-DD.md`：活跃原始计划草案。
- `docs/current/<topic>-requirement-design-YYYY-MM-DD.md`：活跃需求设计草案。
- `docs/current/<topic>-workspace-signal-YYYY-MM-DD.md`：随时交回总控的轻量 signal。
- `docs/current/<topic>-workspace-handoff-YYYY-MM-DD.md`：交回 AlembicWorkspace 的交接草案。
- `docs/current/workspace-handoff-board.md`：正式需求设计完成后的 Design 交接清单。
- `docs/<topic>-policy.md` 或 `docs/<topic>-contract.md`：只保存稳定的设计窗口规则。

文件名使用小写 kebab-case，日期使用执行日 `YYYY-MM-DD`。

## 交接前检查

任何交回 `AlembicWorkspace` 的正式草案，都要先对照 [workspace-alignment-checklist.md](workspace-alignment-checklist.md) 完成自检；自检不通过时，不要建议总控进入目标阶段确认或 wave。
