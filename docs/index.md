# AlembicDesign 文档索引

更新日期：2026-05-25

本索引是 AlembicDesign 的文档地图。当前设计草案放在 `docs/current/`；稳定运行规则放在 `docs/`；可复用模板放在 `templates/`。

## 当前地图

| 类型 | 文档 | 状态 | 说明 |
| --- | --- | --- | --- |
| 运行规则 | [design-window-operating-policy.md](design-window-operating-policy.md) | 已生效 | 定义 AlembicDesign 如何与 AlembicWorkspace 协作。 |
| 当前草案 | [current/README.md](current/README.md) | 空目录说明 | 保存活跃原始计划、需求设计和交接草案。 |
| 原始计划书模板 | [../templates/original-plan-template.md](../templates/original-plan-template.md) | 已生效 | 用于用户确认前的第一轮需求框定。 |
| 需求设计模板 | [../templates/requirement-design-template.md](../templates/requirement-design-template.md) | 已生效 | 用于原始计划确认后的详细需求设计。 |
| Workspace 交接模板 | [../templates/workspace-handoff-template.md](../templates/workspace-handoff-template.md) | 已生效 | 用于交回 AlembicWorkspace 的交接草案。 |

## 文档落点

- `docs/current/<topic>-original-plan-YYYY-MM-DD.md`：活跃原始计划草案。
- `docs/current/<topic>-requirement-design-YYYY-MM-DD.md`：活跃需求设计草案。
- `docs/current/<topic>-workspace-handoff-YYYY-MM-DD.md`：交回 AlembicWorkspace 的交接草案。
- `docs/<topic>-policy.md` 或 `docs/<topic>-contract.md`：只保存稳定的设计窗口规则。

文件名使用小写 kebab-case，日期使用执行日 `YYYY-MM-DD`。
