# 当前设计草案

本目录初始化时保持为空。

这里只保存活跃需求讨论材料：

- `*-original-plan-YYYY-MM-DD.md`
- `*-requirement-design-YYYY-MM-DD.md`
- `*-workspace-signal-YYYY-MM-DD.md`
- `*-workspace-handoff-YYYY-MM-DD.md`
- `workspace-handoff-board.md`

`workspace-signal` 用于把 bug、TODO、调研请求、用户决策或当前主线风险随时交回总控；它不是全局 TODO，也不是执行计划。

`workspace-handoff-board.md` 是正式需求设计完成后的清单入口。Design 把准备交给总控的需求登记为 `ready-for-workspace`；总控通过 `scripts/import-design-handoffs.mjs` 自动读取并生成 workspace 收件箱。

当设计或 signal 被 `AlembicWorkspace` 接收后，由总控窗口决定是复制、链接、归档，还是转换成 workspace TODO、需求、测试单或 wave 文档。
