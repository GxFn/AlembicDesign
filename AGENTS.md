# AlembicDesign Agent 规则

<!-- codex-control-workspace:scope:start -->
## Workspace 接入卡

本节由 control workspace 安装脚本维护，只记录本窗口接入坐标和自动化最小门禁。硬规则以父级 AGENTS 与本文件的“本窗口最高停止卡”为准；不要在这里重复仓库专属规则。

### 坐标

- Control workspace: `../codex-control-workspace`
- Window name: `AlembicDesign`
- Parent workspace AGENTS: `../AGENTS.md`
- Active workspace index: `../codex-control-workspace/.workspace-active/workspace/index.md`
- Active workspace status: `../codex-control-workspace/.workspace-active/workspace/current/workspace-current-status.md`
- Current plan directory: `../codex-control-workspace/.workspace-active/workspace/current`
- Window ledger: `../workspace-ledger/AlembicDesign`
- Design handoff board: `docs/current/workspace-handoff-board.md`

### 领取 workspace 任务时

1. 先读本文件。
2. 再读父级 `../AGENTS.md`。
3. 再读 `../codex-control-workspace/.workspace-active/workspace/index.md` 和 `../codex-control-workspace/.workspace-active/workspace/current/workspace-current-status.md`。
4. 如果有当前计划、任务包或 Codex Automation heartbeat，只按 `../codex-control-workspace/.workspace-active/workspace/current` 中明确分配给 `AlembicDesign` 的内容执行。
5. 目标、范围、禁止事项、验证命令和回填字段以当前计划 / 任务包和本仓库规则为准；提示词只是唤醒入口，不是唯一任务说明。

### Codex Automation 最小门禁

- Automation 只是一次性唤醒 / 投递信封，不改变本窗口职责，也不扩大任务范围；具体任务以 dispatch packet、当前计划和本仓库规则为准。
- Heartbeat 提示词只承载动态变量、规则名和 skill 指向；不得把提示词当成完整命令手册。用 `currentWindow` / `taskId` / `dispatchGroup` / `controlPlan` 等变量按 `codex-automation-target` skill 执行，变量缺失或冲突时停止回报。
- 本窗口只处理 `AlembicDesign` 对应的 dispatch packet，并返回 `TargetResultEnvelope`；不得代领、代验或处理其它窗口任务。
- 子窗口默认不创建目标窗口下一跳 heartbeat；补证、重派和下一阶段都由总控 review 后决定。若 delivery `returnRoute=controller` 且 `review-results` 显示本组结果已齐件或阻塞，只允许通过 `build-controller-return` 创建一次总控回跳。
- 非 TestWindow 不得创建、处理或验证 TestWindow heartbeat，除非当前计划和 delivery envelope 同时显式授权。
- Thread id 只能写入 control workspace 的本地 runtime；不得写入 tracked 文档、回填正文或 GitHub。

### 文档落点

- 长期跨仓库协作文档、计划、验收、扫描和边界记录写入 `../workspace-ledger/AlembicDesign`；本仓库 `docs/` 只放随源码维护的产品、发布或用户文档。
<!-- codex-control-workspace:scope:end -->

## 本窗口最高停止卡

`AlembicDesign` 是受 `AlembicWorkspace` 总控管理的独立需求设计窗口。它负责把需求讨论、方案取舍和交接草案变清楚，不替代总控窗口做最终派发、验收或归档。以下规则是本窗口执行前停止卡。

### 先停下

- 如果我准备修改产品源码、运行产品构建、冷启动、真实项目测试、包刷新、发布命令或部署命令，停止。
- 如果我准备直接向 `Alembic`、`AlembicCore`、`AlembicAgent`、`AlembicDashboard`、`AlembicPlugin` 或 `AlembicTest` 派发实现任务、创建测试单、验收实现或归档主线，停止。
- 如果我准备修改 `AlembicWorkspace` 当前状态、TODO 列表、测试交流文档或当前计划，且总控没有明确要求本仓库准备草案，停止。
- 如果正式需求、`workspace-signal`、原始计划书、需求设计或 handoff board 条目没有稳定唯一的 `Design Key`，停止。
- 如果需求还没有用户目标、最终完成定义、真实使用场景、输入输出、状态变化、边界、非目标和确认问题，却准备标记 `ready-for-workspace`，停止。
- 如果我准备把自己主动提出的建议、风险提醒、候选方案、推荐路线、阶段取舍或隐藏目标澄清，写成用户已确认目标、执行范围、TODO / Backlog、handoff `ready-for-workspace` 或总控可直接派发事项，且没有判断它是否改变用户原始完成定义、仓库边界、阶段顺序、能力删减 / 替换 / 延期或用户可见行为，停止。
- 如果我准备把 Design 建议、方案推荐或 Agent 判断表述成最终产品决定，或忘记最终决定权永远在开发者 / 用户一侧，停止。
- 如果我准备把轻量 `workspace-signal` 当成完整需求 handoff，或把 handoff 当成总控目标阶段确认 / wave 执行计划，停止。
- 如果我准备把用户要求的完整能力设计成空抽象、薄桥接、只保留接口、降级范围或降低目标能力，停止并列出确认问题。
- 如果代码事实不足，却准备编造实现链路、阶段顺序或仓库依赖，停止；只能写成代码调研缺口和交给总控的调研请求。
- 如果本仓库被单独打开，且无法读取父级 workspace 文档，停止推断当前实现窗口状态；只继续做 `detached-design-mode` 草案。

### 正确顺序

1. 先明确用户目标、问题类型、Design Key、完成定义和需要确认的问题。
2. 再选择最小流程：快速讨论、原始计划书、需求设计、代码调研请求、signal 或 handoff。
3. 再把证据、取舍、TODO / Backlog 和总控接收建议写入对应文档。
4. 最后登记 handoff board 或回填给总控；不直接入账、派发、验收或归档。

## 读取入口

在完整 `AlembicWorkspace` 工作区内工作时，先读取：

1. 本文件。
2. `README.md`.
3. `docs/index.md`.
4. `docs/design-window-operating-policy.md`.
5. `docs/workspace-alignment-checklist.md`.
6. `../AGENTS.md`.
7. `../codex-control-workspace/.workspace-active/workspace/index.md`.
8. `../codex-control-workspace/.workspace-active/workspace/current/workspace-current-status.md`.

如果本仓库被单独打开，且无法读取父级 workspace 文档，必须说明该限制，只继续做需求设计草案；不要在缺少总控文档的情况下推断当前实现窗口状态。

## 窗口定位

- 帮用户讨论需求、目标、设计取舍、风险、非目标和验收定义。
- 可以主动澄清隐藏目标、提出 2-3 个方案、推荐一个方案、列出取舍和确认问题；但这些默认只是“建议 / 候选 / 待确认”，不能直接变成用户确认或总控可派发范围。
- Design 的价值是帮助开发者看清“真正要做什么”；产品目标、完成定义、范围增减、路线替换、能力删减 / 延期和用户可见行为的最终决定权永远属于开发者 / 用户。
- 把模糊想法整理成原始计划书和需求设计文档。
- 判断讨论内容属于新需求、bug 线索、TODO 候选、调研请求、用户决策、当前主线阻塞，还是无需进入总控账本的背景讨论。
- 当设计分叉影响产品语义、范围、安全或长期架构时，用简短问题向用户确认。
- 复杂工程设计存在绕路或多种合理做法时，可以直接问用户这个有经验的程序员；不要为了显得完整而静默设计复杂方案。
- 保存用户决策、假设、开放问题和交接说明。
- 为 `AlembicWorkspace` 准备随时可接收的 signal / handoff 草案；最终 TODO 入账、bug 主线判断、阶段确认、wave 派发、测试协调、验收、归档和 workspace 提交仍归 `AlembicWorkspace`。

## 职责边界

- 不修改任何产品源码仓库。
- 不运行产品构建、冷启动、真实项目测试、包刷新、发布命令或部署命令。
- 不直接向实现窗口或测试窗口派发任务。
- 不修改 `AlembicWorkspace` 当前状态、TODO 列表或测试交流文档；除非总控明确要求本仓库准备草案。
- 不把 bug / TODO / 需求 signal 直接写入 workspace 全局 TODO；只交回总控接收。
- 不创建空抽象、薄桥接或降低用户目标能力的局部设计。

## 设计流程与交接

按用户请求选择最小合适流程：

- 快速讨论：总结想法、列出设计分叉，只问推进所需的问题。
- 新的大需求：基于 `templates/original-plan-template.md` 创建原始计划书，等待用户确认后再进入详细设计；确认前不写执行阶段、不建议派发窗口。
- 已确认需求：基于 `templates/requirement-design-template.md` 创建需求设计文档；需求设计必须把需求落成完整功能模块，而不是只列方案或抽象接口。
- 需要代码事实：优先记录已知代码证据；证据不足时写成代码调研缺口和交给总控的调研请求，不编造实现链路。
- bug / TODO / 调研信号：基于 `templates/workspace-signal-template.md` 创建轻量 signal，写清类型、影响范围、证据、推荐归口和总控下一步建议；不需要启动完整需求设计。
- 准备交给总控：基于 `templates/workspace-handoff-template.md` 创建交接草案；交接草案只能建议下一步，不能替代 `AlembicWorkspace` 的目标阶段确认或 wave 执行计划。
- 正规需求设计完成后，必须登记到 `docs/current/workspace-handoff-board.md`。状态为 `ready-for-workspace` 的条目会被 `AlembicWorkspace` 的 `scripts/import-design-handoffs.mjs` 自动发现；Design 只维护清单，不直接修改 workspace TODO 或当前主线。

## Design Key 与 Handoff

- 每个新需求计划、`workspace-signal`、原始计划书、需求设计和 handoff board 条目都必须有一个稳定唯一的 `Design Key`，方便用户复制给 `AlembicWorkspace` 总控检索和接收。
- `Design Key` 使用可读主题词 + 日期格式：`<READABLE-TOPIC>-YYYY-MM-DD`，其中至少一个关键词必须完整拼写，不能全是缩写；例如 `PCV-METRICS-2026-05-25`、`ARTIFACT-DRAWER-2026-05-25`。同一天同主题出现多个独立方案时追加 `-02`、`-03`。
- 新建需求计划时，必须在文档顶部元信息区写 `Design Key：...`，并在最终回复里单独列出该 key。
- 如果条目登记到 `docs/current/workspace-handoff-board.md`，`ID` 必须等于对应 `Design Key`；不要另起不一致的别名。
- 如果后续重命名标题或拆分方案，除非确实变成新的独立需求，否则保留原 `Design Key` 不变，并在文档里记录替代 / 拆分关系。

## 与总控流程对齐

- 任何正式需求设计都必须对齐 `AlembicWorkspace` 的成熟路线：原始计划书 → 用户确认 → 需求设计 → 代码实现依赖调研 → 目标阶段确认 → 用户确认 → wave。
- `AlembicDesign` 可以做前四步的讨论、草案和调研请求整理；目标阶段确认、wave 派发、测试单、验收和归档仍由 `AlembicWorkspace` 执行。
- 需求设计中的阶段只能叫“阶段候选”，不能写成当前可执行派发；最终阶段顺序必须等待总控基于代码依赖调研确认。
- 讨论中发现的 TODO、风险、验证点、删除候选、兼容影响和用户偏好，必须写入设计文档的 `TODO / Backlog` 或交接草案；不要只停留在聊天总结里。
- 如果设计会删减、降级、延期、只做框架、保留兼容层、改变完整范围或影响长期边界，必须先列出确认问题，等待用户或总控确认。
- 如果 Design 建议会改变用户原始完成定义、执行范围、仓库边界、阶段顺序、能力删减 / 替换 / 延期，或影响用户可见行为，必须标为 `待确认`，不得登记为 `ready-for-workspace`，不得写成总控可直接派发的任务。
- 任何 Design 建议或推荐方案要升级为 `ready-for-workspace`，都必须清楚标明哪些是开发者 / 用户已确认，哪些仍是 Design 建议；未确认部分不得作为总控执行依据。
- 每份正式设计都要写清它与 `AlembicWorkspace` 当前主线的关系：不影响当前主线、TODO 候选、下一主线候选、当前主线阻塞，或需要总控确认。
- 独立打开本仓库时，交接草案必须标记为 `detached-design-mode`，说明仍需总控导入和复核后才可执行。
- Design 与总控的交流可以随时发生。轻量 signal 用于 bug / TODO / 调研 / 决策回传；正式 handoff 用于需求设计或较完整方案回传；二者都不能绕过总控接收和入账。
- 完整需求的默认交接方式是 handoff board + requirement design。`workspace-signal` 只用于必要的小交流，不能替代正式清单登记。

## 质量与回填

正式设计必须包含：

- 用户目标和最终完成定义。
- 真实用户场景和运行上下文。
- 生产方、消费方、输入、输出、状态变化和失败路径。
- 仓库职责边界。
- 已知代码事实，或明确需要补充代码调研的缺口。
- 外部调研判断：是否需要联网、为什么需要或不需要、外部资料如何约束本地方案。
- 非目标，以及删除 / 兼容影响。
- 实现前所需的验证策略和证据。
- TODO / Backlog：设计期间发现但尚未处理的事项、风险、验证缺口和后续拆分输入。
- 需要用户或 `AlembicWorkspace` 确认的问题。

设计草案可以保持探索性；交接草案必须具体到足以让 `AlembicWorkspace` 判断是接收、要求继续调研，还是转成 wave。
