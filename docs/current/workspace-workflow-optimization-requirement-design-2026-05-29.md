# Workspace Workflow Optimization 需求设计

Design Key：WORKSPACE-WORKFLOW-OPTIMIZATION-2026-05-29
日期：2026-05-29
状态：ready-for-workspace / after-current-mainline
维护窗口：AlembicDesign
总控：AlembicWorkspace
原始计划：[workspace-workflow-optimization-original-plan-2026-05-29.md](workspace-workflow-optimization-original-plan-2026-05-29.md)
关联 research：[workspace-workflow-optimization-analysis-2026-05-29.md](workspace-workflow-optimization-analysis-2026-05-29.md)

## 定位

本需求优化的是 AlembicWorkspace 的控制系统，而不是某个产品功能。

它不新增一套流程架构，而是把现有机制整理成一条默认闭环：

```text
Intake -> Decision -> Ledger -> Plan -> Deliver -> Result -> Review -> Record
```

核心目标是减少重复判断和重复规则，让每层只做自己该做的事：

- Design 负责需求设计和 handoff。
- 总控负责目标、完成定义、阶段、派发和验收裁决。
- 账本负责记录已裁决事实。
- template 负责结构。
- skill / reference 负责操作说明。
- script 负责机械导入、校验、同步、归档和 packet 检查。
- delivery 负责送达。
- 子窗口负责执行自己窗口的任务并回证据。

## 最终目标

形成 `WorkspaceControlLoop` 的统一工作方式：

```text
用户输入或 Design handoff
-> 总控分类并裁决是否接收
-> 总控定义目标、完成定义、阶段和第一阻塞点
-> 只写入一个正确账本
-> 当前阶段形成 task package 和 target prompt
-> manual / automation delivery 送达
-> 子窗口返回 TargetResultEnvelope
-> 总控 pull 原始证据验收
-> 更新账本、进入下一阶段或归档
```

这是一条主链，不是多条并行流程。

所有重量能力都只作为支持能力：

- 不清楚时做 preflight。
- 证据冲突时做 pull review。
- 配置不一致时做 install / config check。
- 真实场景需要时走 test exchange。
- 归档前做 full verify。

普通闭环不默认跑全量脚本、全量 TODO 审计、全量 test boundary 或全量归档检查。

## 设计原则

### 1. 一个判断只归一个 owner

| 判断 | 唯一 owner | 其它层只能做什么 |
| --- | --- | --- |
| 用户目标和完成定义 | 总控，必要时由用户确认 | Design 提建议；脚本不判断 |
| 是否入 TODO / Backlog | 总控 | Design 给挂载建议 |
| 当前阶段和第一阻塞点 | 总控 | current plan 记录结果 |
| 派发窗口和任务包 | 总控 | VAD 不选择窗口 |
| prompt 内容 | 总控派发层 | delivery 不改语义 |
| 是否完成 | 总控验收层 | 子窗口只回 evidence |
| 是否需要 TestWindow | 总控 | 子窗口不能自建测试任务 |
| 规则放在哪里 | 总控治理 | skill / template 不自我扩权 |

### 2. 普通路径短，失败路径明确

默认只走一条短路径。失败时按触发原因进入对应诊断能力，不把所有诊断都放到普通前置。

### 3. 资产不互相代替

- `AGENTS.md` 不能变成长命令手册。
- skill / reference 不能藏最高硬规则。
- template 不能写当前事实。
- script 不能替代判断。
- TODO 不能驱动派发。
- result envelope 不能直接关闭任务。
- TestWindow 不能替代总控已知事实复核。

### 4. Manual 与 automation 共享任务语义

manual copy prompt 和 automation prompt 的 delivery 方式不同，但任务语义应来自同一个总控 task package。

区别只在包装：

- manual prompt：更自包含，方便用户复制给不确定窗口。
- automation prompt：更短，目标窗口和任务已知。

不要维护两套任务定义。

## 控制资产归属

| 资产 | 应负责 | 不应负责 | 优化方向 |
| --- | --- | --- | --- |
| Parent / Control `AGENTS.md` | 最高停止卡、身份边界、确认门禁、验收硬边界、分派和自动化不可越权规则、skill / reference 地图 | 当前 wave 事实、长命令手册、脚本排错细节、完整模板、子窗口任务书 | 保留常驻硬规则和地图；把操作细节引到 skill / reference |
| Governance skill / references | TODO、dispatch、testing、archive、automation、workspace governance 的操作手册和示例 | 替代总控判断、隐藏最高硬规则、记录当前状态 | 只在场景命中时打开；不要成为每次默认负担 |
| Templates | 固定章节、表格形状、脚本锚点、少量边界提醒 | 当前事实、长 playbook、总控裁决、一次性任务内容 | 小而稳定，只保证结构契约 |
| Scripts | import、sync、verify、archive、packet / envelope check、diagnostic probe | 用户目标判断、TODO 优先级、窗口选择、验收裁决、产品实现 | 分成 contract / ledger / diagnostic 三类 |
| `.workspace-active` current ledgers | 当前计划、当前状态、Design inbox、活跃 TODO、test exchange、当前回填 | 长期规则、产品文档、历史全量归档、子仓库源码说明 | 只放当前要判断和执行的事实 |
| `workspace.config` / `.workspace-local` | repository path、windowName、role、mode、managedAgents、本机覆盖 | 当前任务状态、TODO、验收结论、派发表 | 作为子窗口接入配置来源，不承载任务事实 |
| Child `AGENTS.md` managed block | 本仓库执行边界、workspace access card、manual task / automation heartbeat 最小门禁 | 总控当前 wave 完整任务、全局 TODO、其它窗口任务、验收结论 | 拆清 manual、automation、local stop card |
| Delivery / VAD | targetWindow、prompt、returnRoute、keepLive、oneShot、correlationId 的投递 | 解析 current plan、生成 prompt、选择窗口、验收证据 | 继续保持“给我信封，我负责送达” |
| TargetResultEnvelope | 窗口、taskId、status、commit / diff / test / report refs、风险 | 关闭 TODO、裁决 accepted、创建下一波 | 作为总控 pull review 的证据索引 |
| Test exchange | 真实项目、cold-start、Dashboard 手动观察、runtime monitoring 的测试单和边界 | 产品修复、总控已知事实复核、普通脚本验证 | 只在真实场景必要时启用 |

## 默认闭环

### Step 1：Intake

输入来源只做分类：

- user request。
- Design handoff。
- window result。
- TODO / Backlog。
- test result。
- current-mainline risk。

输出不是派发，而是一个总控待裁决问题：

```text
这是什么类型？
是否影响当前主线？
是否需要用户确认？
是否需要进入 TODO / current plan / test exchange / archive？
```

### Step 2：Decision

总控必须先定：

- 用户目标。
- 完成定义。
- 当前是否已达到。
- 剩余差距。
- 第一阻塞点。
- 下一步属于 intake / todo / plan / dispatch / review / archive / stop 哪一种。

这一步不能由脚本、TODO、窗口回填或 Design 建议替代。

### Step 3：Ledger

只写入一个正确账本：

- Design handoff inbox：等待总控接收的需求。
- TODO board：已裁决要跟踪的问题。
- Current plan：当前阶段执行任务包。
- Test exchange：真实测试请求。
- Archive：已验收事实。

原则：账本记录裁决，不制造裁决。

### Step 4：Plan

总控把当前阶段转为 task package：

- target window。
- target repository。
- task id / source key。
- objective。
- allowed / forbidden actions。
- required evidence。
- verification commands。
- stop conditions。
- backfill target。

这一步也生成 prompt 内容；delivery 只负责送达。

### Step 5：Deliver

Delivery 只能接收：

```text
targetWindow + prompt + returnRoute + keepLive + oneShot + correlationId
```

Delivery 不解析 TODO，不读取 current plan，不重新生成 prompt，不判断任务是否完成。

### Step 6：Result

子窗口返回：

```text
targetWindow
taskId
status
changedRepos
commits / no-code-change reason
evidenceRefs
verificationSummary
riskSummary
nextSuggestion
```

自然语言可以很短，但必须能对应这些字段。

### Step 7：Review

总控把 result envelope 当作证据索引，独立 pull 原始证据：

- commit。
- diff。
- command output。
- runtime JSON。
- report。
- screenshot。
- artifact。

然后对照完成定义裁决：

```text
accepted / needs-rework / blocked / next-wave / stop
```

### Step 8：Record

最后只把已经裁决的事实写回账本：

- 关闭或滚动 TODO。
- 更新 current plan / current status。
- 创建下一阶段任务包。
- 写 test exchange。
- 归档。

不得先改文档制造进展感。

## 失败诊断阶梯

失败路径只按触发条件进入，不与普通路径并行。

| 触发 | 使用能力 | 不做什么 |
| --- | --- | --- |
| 用户目标、完成定义或主线影响不清 | 需求确认 / Design 补充 | 不派发窗口 |
| Design handoff 改变范围、仓库边界或阶段顺序 | 总控接收审查 | 不直接写 TODO |
| 当前计划缺任务包、验证命令或窗口覆盖 | current plan 补齐 / check-dispatch-coverage | 不启动 delivery |
| 目标窗口或仓库定位不明 | workspace.config / managed AGENTS 检查 | 不让窗口猜 |
| 子窗口回填缺证据 | pull evidence / needs-rework | 不标 accepted |
| 回填与原始证据冲突 | 总控原始证据复核 | 不信任自然语言结论 |
| 需要真实项目或人工观察 | test exchange | 不把普通 unit 交给 AlembicTest |
| 自动化投递失败 | delivery diagnostic / keepLive check | 不改任务语义 |
| 归档前状态不一致 | full verify / archive diagnostic | 不继续下一波 |

## 规则归属判定法

总控治理规则时可用以下判定：

| 规则形态 | 放置位置 |
| --- | --- |
| “如果命中必须停下 / 不得继续 / 必须确认” | `AGENTS.md` |
| “在某场景如何操作、运行什么命令、如何排错” | governance skill / reference |
| “文档必须有什么章节、表格、字段” | templates |
| “能否机械检查、导入、同步、归档、生成 packet” | scripts |
| “这次任务当前事实、当前窗口状态、当前回填” | `.workspace-active` |
| “哪个仓库、窗口名、路径、本机覆盖、managedAgents” | `workspace.config` / `.workspace-local` |
| “目标仓库自己执行前的边界和 workspace 接入卡” | child `AGENTS.md` |
| “执行结果证据索引” | TargetResultEnvelope / backfill |

迁移规则前必须问：

```text
下沉后，总控是否还会在准备派发、验收、测试、改文档、创建 automation、领取 TODO 或归档前看到这条硬规则？
```

如果答案是否定，这条规则不能离开 `AGENTS.md`。

## 子窗口配置设计

第一版不新增独立配置系统，只把现有 `workspace.config` + managed `AGENTS.md` 视为一个 `ChildWindowAccessProfile`。

建议每个子窗口都能被总控复核出以下信息：

```text
windowName
repositoryPath
role
mode
managedAgents
manualWorkspaceTask:
  - 是否要求读 parent AGENTS
  - 是否要求读 active index / current plan
  - 是否要求读 local AGENTS
compactAutomation:
  - 是否接受 delivery envelope
  - 是否要求 correlationId
  - 是否 oneShot
  - 是否禁止普通 next delivery
testBoundary:
  - 是否能承接真实项目 / Dashboard / cold-start / runtime monitoring
```

这只是接入边界视图，不承载任务状态。

## Script Surface 设计

脚本按能力分三类：

### Contract scripts

负责结构和协议：

- dispatch packet check。
- delivery envelope check。
- target result envelope check。
- task package structure check。

不负责目标判断。

### Ledger scripts

负责账本动作：

- import Design handoffs。
- sync current plan / index / status。
- todo board check。
- archive support。

不负责决定是否入账。

### Diagnostic scripts

负责失败和归档前审计：

- full preflight。
- full verify。
- config mismatch audit。
- dispatch coverage audit。
- test boundary audit。
- automation loop cleanup audit。

不进入普通 happy path 默认前置。

## 与 VAD / Codex Automation Closed Loop 的关系

`CODEX-AUTOMATION-CLOSED-LOOP-2026-05-29` 解决的是 delivery 层和 Codex 内部闭环。

本需求解决的是 delivery 上游和下游的 Workspace 总控链路：

```text
Workspace 总控生成任务语义
-> VAD / delivery 只送达
-> 子窗口只回证据
-> Workspace 总控验收
```

因此本需求不能把 VAD 重新扩成派发系统，也不能在 Workspace workflow 里再复制一套 VAD 逻辑。

## 与 Multi LLM Dispatch 的关系

`MULTI-LLM-DISPATCH-OPTIMIZATION-2026-05-29` 是后续可扩展方向。

本需求第一版只保证 Workspace 内部链路干净。任务语义一旦稳定，将来可以被 Codex、Claude Code、其它 IDE agent 或人工复制复用，但第一版不以多 IDE 自动化为完成定义。

## 总控接手建议

接手顺序必须保守：

1. 只做 Stage 0 控制资产地图。
2. 列出重复、越权、漂移和缺口。
3. 由总控裁决哪些进入 TODO / Backlog。
4. 再决定是否进入 Stage 1-5。

不要一接手就改 `AGENTS.md`、脚本、模板或子窗口配置。

## 完成定义

本需求完整完成时，Workspace 应满足：

- 普通任务能按一条主链推进。
- 总控、Design、账本、模板、skill、script、delivery、子窗口、TestWindow 的职责不互相覆盖。
- 任一规则可以追溯到唯一主归属。
- 任一脚本可以被归为 contract / ledger / diagnostic。
- 任一子窗口可以看出 manual task 和 compact automation 的接入边界。
- 任一 result envelope 都不会被当成自动验收结论。
- 失败时进入对应诊断能力，而不是默认回到全量流程。
- 至少一个真实小闭环验证通过，并记录冗余是否减少、证据质量是否足够。

## 风险与防护

| 风险 | 防护 |
| --- | --- |
| 为了简化而删除硬规则 | 硬规则迁移前必须证明总控仍会看到 |
| 设计变成新流程架构 | 只整理现有机制和接入边界 |
| 脚本继续承担判断 | 所有脚本标注 `mustNotDecide` |
| 子窗口配置变成任务状态 | `workspace.config` 只保存接入信息 |
| happy path 过短导致证据不足 | result envelope 必须保留证据 refs，总控 pull review |
| diagnostic path 被默认化 | 每个 diagnostic 都必须有触发条件 |

## 当前状态

Design 已按用户推荐确认完成需求设计，并已按用户要求升级为 `ready-for-workspace`。

用户说明处理完当前主线后再开始，因此该需求应作为后续 Workspace governance 候选交给总控接收；不打断当前主线。

总控接手后仍只建议从 Stage 0 控制资产地图开始。

## 仍需确认

无业务设计阻塞。

总控接手后仍需在 Stage 0 裁决的执行边界：

- Stage 0 是否只覆盖 control workspace tracked / active 资产，还是同时读取外部 `AlembicDesign`、`AlembicTest` 和各产品子仓库的实际 `AGENTS.md` managed block。Design 推荐先覆盖 control workspace + 当前配置中登记的 managed blocks，不做产品源码扫描。
