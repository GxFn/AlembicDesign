# AlembicDesign Workspace Handoff Board

更新日期：2026-05-25
维护窗口：AlembicDesign
接收窗口：AlembicWorkspace

## 定位

本文件是 AlembicDesign 自己维护的正式交接清单。Design 完成原始计划、需求设计、用户确认和 TODO / Backlog 挂载建议后，把条目登记到这里；AlembicWorkspace 通过 `scripts/import-design-handoffs.mjs` 自动发现并生成总控收件箱。

本清单不是 workspace 全局 TODO，也不是执行计划。总控脚本只能自动发现、校验和汇总；是否正式入账、是否打断当前主线、是否进入目标阶段确认或 wave，仍由 AlembicWorkspace 裁决。

## 状态枚举

- `draft`：Design 仍在整理，暂不交给总控。
- `ready-for-workspace`：Design 已完成交接准备，等待总控接收。
- `accepted-by-workspace`：总控已正式接收并转入 workspace 账本。
- `needs-design`：总控或用户要求 Design 继续补充。
- `paused`：用户或 Design 暂停。
- `archived`：已归档或不再需要总控接收。

## Handoff 清单

| ID | 状态 | 标题 | 原始计划 | 需求设计 | Handoff | 用户确认 | 当前主线关系 | 建议 TODO | 优先级 | 下一步 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| PCVM-2026-05-25 | ready-for-workspace | Progressive Chain Validation Metrics | [original-plan](progressive-chain-validation-metrics-original-plan-2026-05-25.md) | [requirement-design](progressive-chain-validation-metrics-requirement-design-2026-05-25.md) | 未单独生成；需求设计已包含交接信息 | 用户已确认 | 下一主线候选；不打断当前 LLM 输入优化 Test-08 | 建议并入 `GTODO-2026-05-25-003`，当前主线完成后由总控做代码事实调研和目标阶段确认 | P0-after-current | 总控接收评审；正式入账后补代码事实调研，不直接派发实现窗口 |

## 使用规则

- `ready-for-workspace` 条目必须有原始计划、需求设计、用户确认、当前主线关系、建议 TODO 和下一步。
- Handoff 文档可选；如果需求设计已经包含足够的交接信息，可以在 `Handoff` 列说明原因。
- 清单中可以出现多个 ready 条目，但总控只能按当前主线、优先级、依赖和目标阶段确认逐个接收。
- Design 不直接修改 `docs/workspace/current/global-todo-board.md`，也不直接创建 wave。
