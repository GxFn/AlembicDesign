# AlembicDesign

`AlembicDesign` 是 Alembic 工作区的独立需求设计窗口。它帮助用户和 `AlembicWorkspace` 总控在派发实现 wave 之前，讨论产品意图、梳理需求计划、比较设计方案，并准备交回总控的交接草案。

本仓库不实现 Alembic 产品代码。它受 `AlembicWorkspace` 总控窗口管理。

## 职责

- 与用户讨论新需求，并把目标具体化。
- 产出原始计划书、需求设计、设计笔记、方案对比和交接草案。
- 识别缺失事实、决策点、风险、非目标和验证需求。
- 请求或总结代码调研，但不直接派发实现窗口。
- 将已确认的设计材料交回 `AlembicWorkspace`，由总控继续做阶段确认、wave 派发、验证和归档。

## 非职责

- 不修改 `Alembic`、`AlembicCore`、`AlembicAgent`、`AlembicDashboard`、`AlembicPlugin`、`AlembicTest` 或真实测试项目源码。
- 不作为权威来源创建实现 wave 计划。
- 除非 `AlembicWorkspace` 已经分派，不通知实现窗口开始工作。
- 不替代 workspace TODO 列表、测试交流或最终验收流程。

## 入口

- [AGENTS.md](AGENTS.md)：设计窗口运行规则。
- [docs/index.md](docs/index.md)：设计文档地图。
- [docs/design-window-operating-policy.md](docs/design-window-operating-policy.md)：长期协作规则和交接契约。
- [templates/original-plan-template.md](templates/original-plan-template.md)：原始计划书模板。
- [templates/requirement-design-template.md](templates/requirement-design-template.md)：需求设计模板。
- [templates/workspace-handoff-template.md](templates/workspace-handoff-template.md)：交回 `AlembicWorkspace` 的交接模板。
