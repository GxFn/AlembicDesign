# AlembicDesign

`AlembicDesign` 是 Alembic 工作区的独立需求设计窗口。它帮助用户和 `AlembicWorkspace` 总控在派发实现 wave 之前，讨论产品意图、梳理需求计划、比较设计方案，并准备交回总控的交接草案。

本仓库不实现 Alembic 产品代码。它受 `AlembicWorkspace` 总控窗口管理。

## 职责

- 与用户讨论新需求，并把目标具体化。
- 产出原始计划书、需求设计、设计笔记、方案对比和交接草案。
- 将讨论中的 bug 线索、TODO 候选、调研请求或用户决策整理为轻量 signal，随时交回 `AlembicWorkspace`。
- 识别缺失事实、决策点、风险、非目标、验证需求和 TODO / Backlog。
- 请求或总结代码调研；证据不足时明确写成调研缺口，不编造实现链路。
- 按总控成熟路线准备前置材料：原始计划书、用户确认、需求设计、代码实现依赖调研请求和交接草案。
- 将已确认的设计材料交回 `AlembicWorkspace`，由总控继续做阶段确认、wave 派发、验证和归档。

## 非职责

- 不修改 `Alembic`、`AlembicCore`、`AlembicAgent`、`AlembicDashboard`、`AlembicPlugin`、`AlembicTest` 或真实测试项目源码。
- 不作为权威来源创建实现 wave 计划。
- 不把阶段候选写成可执行派发，不替代目标阶段确认。
- 除非 `AlembicWorkspace` 已经分派，不通知实现窗口开始工作。
- 不替代 workspace TODO 列表、测试交流或最终验收流程。

## 入口

- [AGENTS.md](AGENTS.md)：设计窗口运行规则。
- [docs/index.md](docs/index.md)：设计文档地图。
- [docs/design-window-operating-policy.md](docs/design-window-operating-policy.md)：长期协作规则和交接契约。
- [docs/workspace-alignment-checklist.md](docs/workspace-alignment-checklist.md)：与 `AlembicWorkspace` 总控需求设计能力的逐项对应检查。
- [templates/original-plan-template.md](templates/original-plan-template.md)：原始计划书模板。
- [templates/requirement-design-template.md](templates/requirement-design-template.md)：需求设计模板。
- [templates/workspace-signal-template.md](templates/workspace-signal-template.md)：交回总控的轻量 bug / TODO / 调研 / 决策 signal 模板。
- [templates/workspace-handoff-template.md](templates/workspace-handoff-template.md)：交回 `AlembicWorkspace` 的交接模板。

## 工作模式

- 在完整 `AlembicWorkspace` 中运行：读取父级总控文档，明确当前主线关系，再产出设计草案或 handoff。
- 独立打开本仓库：只做 `detached-design-mode` 草案，不推断实现窗口状态，不把草案当成可执行计划。
- 讨论中出现 bug / TODO / 调研请求：先产出轻量 signal，交给总控决定是否入账、派发或升级为主线。
- 交给总控前：使用对齐检查清单确认目标、闭环、证据、TODO、阶段候选和确认问题都已经写入文档。
