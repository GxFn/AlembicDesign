# Workspace Workflow Optimization 原始计划

Design Key：WORKSPACE-WORKFLOW-OPTIMIZATION-2026-05-29
日期：2026-05-29
状态：ready-for-workspace / after-current-mainline
维护窗口：AlembicDesign
总控：AlembicWorkspace
关联 research：[workspace-workflow-optimization-analysis-2026-05-29.md](workspace-workflow-optimization-analysis-2026-05-29.md)

## 判断类型

这是 Workspace 整体工作流优化的 requirement-candidate / governance-design，不是产品实现需求，不是执行计划，也不是对当前总控状态或全局 TODO 的直接修改。

本需求参照 `CODEX-AUTOMATION-CLOSED-LOOP-2026-05-29` 的 VAD 收敛结论继续上移一层：VAD 已经拆清“总控写信，VAD 送信”；Workspace 整体也需要拆清“谁判断、谁记录、谁生成提示词、谁投递、谁回证据、谁验收”。

## 用户诉求

用户希望从更整体的角度优化 AlembicWorkspace 工作流，覆盖：

- 总控现有 `AGENTS.md`。
- governance 文档、reference、skill。
- 模板。
- 脚本。
- `.workspace-active` 当前账本。
- 子窗口配置和 managed `AGENTS.md`。
- 子窗口结果回填与总控验收关系。

用户要求：

- 参照 VAD 的优化逻辑。
- 边界清晰。
- 不产生分叉与冗余。
- 链路清晰、简洁。
- 不新增一套额外流程架构，优先使用现有机制。

## 已确认取舍

按 Design 推荐，当前设计先采用以下取舍：

- Stage 0 先做控制资产地图和归属复核，不直接修改 `AGENTS.md`、模板、脚本或子窗口配置。
- `AGENTS.md` 只保留常驻硬规则、停止卡、身份边界和入口地图；不得把最高硬规则下沉到 skill / reference 后消失。
- skill / reference 承载操作步骤、命令细节、字段模板、示例和排错说明。
- template 只承载结构契约，不承载长 playbook、当前事实或总控裁决。
- script 只做机械能力：导入、校验、同步、归档、packet / envelope 检查；不替代总控判断。
- `.workspace-active` 只保留当前工作面事实，避免变成长期规则仓库。
- 子窗口配置纳入设计范围，但作为既有 `workspace.config` + managed `AGENTS.md` 的整理视图，不新增独立调度系统。
- 最终验证需要跑一个小真实闭环：Design handoff -> 总控接收 -> TODO / current plan -> dispatch -> result envelope -> review。

## 最终目标

建立一条统一的 `WorkspaceControlLoop`：

```text
输入 / Design handoff
-> 总控裁决目标与完成定义
-> 写入唯一正确账本
-> 生成当前阶段 task package 和目标 prompt
-> manual / automation delivery 送达
-> 子窗口返回 result envelope
-> 总控 pull 原始证据验收
-> 更新账本 / 下一阶段 / 归档
```

这条路径应成为普通工作默认路径。其它全量 preflight、脚本诊断、测试交接、归档审计、配置同步都只在对应失败、风险或归档场景启动。

## 非目标

- 不替换总控。
- 不让 Design 直接派发实现。
- 不改产品仓库源码。
- 不直接更新 workspace 当前状态、全局 TODO 或 current plan。
- 不把脚本升级成决策引擎。
- 不把 VAD 再次扩展成总控派发系统。
- 不新增第二套 TODO / wave / automation 流程。
- 不把子窗口变成可自行领取其它窗口任务或自行验收的执行体。

## 需求范围

本需求只设计 Workspace 控制系统自身的职责分层和默认闭环。

第一版覆盖：

- `AGENTS.md` hard rule / map 边界。
- governance skill / reference 的触发边界。
- templates 的结构契约边界。
- scripts 的 contract / ledger / diagnostic 分类边界。
- `.workspace-active` 当前账本边界。
- `workspace.config` 与子窗口 managed `AGENTS.md` 的接入边界。
- 总控 planning / dispatch 与 delivery 的分离。
- 子窗口 result envelope 与总控 acceptance 的分离。
- happy path 与 diagnostic path 的触发门槛。

## 完成定义

Design 阶段完成时，应具备：

- 一份正式需求设计文档，说明 Workspace 每层资产的唯一职责、禁止职责和优化方向。
- 一条默认 happy path，不要求普通任务同时跑多套流程。
- 一组失败 / 风险场景触发的重量能力，不作为默认前置。
- 明确说明总控、Design、脚本、模板、VAD、子窗口和 TestWindow 的边界。
- 明确说明总控接手后的第一阶段不是改规则，而是做控制资产地图。
- 明确说明如何用一个小真实闭环验证优化是否成立。

总控实现阶段完成时，应具备：

- 控制资产地图完成并可复核。
- 重复规则和错误归属被定位。
- 至少一条普通闭环能按精简路径完成。
- 失败时能按诊断阶梯定位，而不是回到全量流程。
- 没有新增第二套派发、验收或 TODO 机制。

## 建议阶段

### Stage 0：控制资产地图

盘点 `AGENTS.md`、governance skill / reference、templates、scripts、current ledgers、workspace config、child AGENTS、result envelope 的现有职责和重复规则。

输出只应是地图和问题清单，不直接改规则。

### Stage 1：Workspace 默认闭环定稿

把普通路径固定为：

```text
Intake -> Decision -> Ledger -> Plan -> Deliver -> Result -> Review -> Record
```

同时明确每一步只由一个 owner 做最终判断。

### Stage 2：资产归属调整方案

根据 Stage 0 地图，给出规则迁移建议：

- 哪些必须保留在 `AGENTS.md`。
- 哪些移动到 skill / reference。
- 哪些缩成 template 字段。
- 哪些改成 script check。
- 哪些属于 current ledgers。
- 哪些写入子窗口 managed access card。

### Stage 3：脚本与模板瘦身方案

只整理现有机制，不新增流程系统：

- contract scripts：packet / envelope / structure check。
- ledger scripts：import / sync / archive。
- diagnostic scripts：preflight / full verify / failure audit。

普通 happy path 不默认跑 diagnostic scripts。

### Stage 4：子窗口接入边界整理

整理 `workspace.config` + child managed `AGENTS.md` 的接入卡：

- manual workspace task。
- compact automation heartbeat。
- local repository stop card。
- test boundary。

它是配置视图，不是新调度系统。

### Stage 5：真实小闭环验证

选择一个低风险 Design handoff 或维护 TODO，验证：

```text
Design handoff -> 总控接收 -> current plan task package
-> dispatch / delivery -> result envelope -> pull evidence review
```

验证结论必须包含成功路径、失败触发、证据质量和是否真的减少冗余。

## 建议总控下一步

暂不直接派发实现窗口。

若用户确认交给总控，建议总控先接收为治理型需求，并只启动 Stage 0 控制资产地图。Stage 0 不应直接改 `AGENTS.md`、脚本、模板或子窗口配置；它只回答“每条规则和每个字段现在属于谁、应该属于谁、是否重复或越权”。

## 证据状态

证据状态：Design research + 本地文档事实。

已参考：

- 当前 Design research 文档。
- `CODEX-AUTOMATION-CLOSED-LOOP-2026-05-29` 已确认的 VAD / 总控分层结论。
- AlembicWorkspace `AGENTS.md` 当前硬规则结构。
- control workspace skills / scripts / templates / current ledgers 的已读事实。

仍需总控在接手后复核真实代码和脚本行为。

## 仍需确认

当前没有阻塞 Design 继续整理的问题。

用户已确认升级为 `ready-for-workspace`，并说明处理完当前主线后再开始。总控接手后仍应从 Stage 0 资产地图开始，不直接进入规则或脚本改造。
