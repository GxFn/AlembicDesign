# Multi Repository Interface Optimization 自动化准备附录

Design Key：MULTI-REPOSITORY-INTERFACE-OPTIMIZATION-2026-05-28
附录日期：2026-05-30
状态：ready-for-workspace / automation-ready addendum
维护窗口：AlembicDesign
总控：AlembicWorkspace
主需求设计：[multi-repository-interface-optimization-requirement-design-2026-05-28.md](multi-repository-interface-optimization-requirement-design-2026-05-28.md)
breaking-cleanup 补充：[multi-repository-interface-optimization-breaking-cleanup-addendum-2026-06-01.md](multi-repository-interface-optimization-breaking-cleanup-addendum-2026-06-01.md)

## 判断类型

这是现有 `MULTI-REPOSITORY-INTERFACE-OPTIMIZATION-2026-05-28` 的自动化准备补充，不是新需求，不新增流程架构，不直接派发实现窗口。

目标是让总控后续可以把该需求作为无人值守维护型需求领取，但第一步仍必须是总控裁决后的只读事实清单。

2026-06-01 更新：用户确认当前仍处于开发阶段，可以彻底清理兼容层。自动化安全边界因此从“保守兼容修复”调整为“有 canonical producer 和验证命令的 breaking compatibility cleanup”。旧路径、旧字段、旧函数名和旧 wrapper 可以作为删除对象，但不得删除真实业务能力或破坏持久化 / 发布 / 安装兼容而未经总控裁决。用户同时确认首批优先级为 `core-host-agent-workflow-contract`、`api-ai-runtime-contract`，确认 `API AI` 作为用户可见和 contract 统一口径，并确认 `external_agent` role id 可作为 `host_agent` breaking cleanup 候选。

## 自动化最终目标

让多仓库接口优化变成可批量、可停下、可验收的维护任务来源：

```text
Stage 0 选择一个完整 Interface Area 并做只读事实清单
-> 生成 Interface Dossier 候选
-> 总控选择一个低风险任务包
-> 目标仓库最小修复
-> producer + consumer 验证
-> 总控 pull evidence 验收
-> 再决定是否领取下一项
```

自动化只负责提高低歧义维护任务的吞吐，不改变总控裁决权。

## 区域先行规则

用户已确认：按最佳实践方案推进，但每个阶段都先规划完整区域来执行。

因此自动化领取必须采用 `Area-first`：

```text
Pick one Interface Area
-> Build Area Plan
-> Classify dossiers
-> Execute one safe dossier
-> Review area evidence
-> Pick next dossier or next area
```

自动化不得直接从全仓扫描结果里随手挑一个小修；也不得把 area plan 变成大重构。

### Interface Area 枚举

第一版只允许总控从以下 area 中选择：

- `core-shared-contracts`
- `alembic-http-resident-api`
- `plugin-runtime-mcp-skill`
- `agent-provider-contract`
- `test-fixture-contract`

2026-06-01 breaking-cleanup 首批可细化为：

- `core-host-agent-workflow-contract`
- `api-ai-runtime-contract`
- `plugin-codex-mcp-runtime-layout`
- `alembic-host-bridge-flat-paths`
- `test-fixture-contract-followup`

如果发现候选不属于这五类，先标为 `needs-control-decision`，不要自动化实现。

### Area Plan 模板

```text
Area:
Goal:
Producer(s):
Consumer(s):
Known anchors:
Missing anchors:
Known failures / drift:
Automation-safe dossiers:
Needs-control-decision dossiers:
Not-actionable items:
Verification matrix:
Stop conditions:
```

`Area Plan` 是阶段入口。`Interface Dossier` 是执行单元。

## 自动化领取前置条件

每个可自动化任务必须先具备 `Interface Dossier`：

```text
Interface:
Producer:
Consumers:
Contract anchor:
Current evidence:
Failure / drift evidence:
Allowed change:
Forbidden change:
Compatibility impact:
Verification:
Backfill evidence:
Stop condition:
```

没有 dossier 的接口优化，只能停在 Stage 0 事实清单，不能进入无人值守实现。

## 自动化安全任务类型

第一版只允许以下五类任务进入无人值守自动化。

| 类型 | 可自动化条件 | 示例 | 验证 |
| --- | --- | --- | --- |
| `consumer-stale-type-fix` | producer contract 已明确，consumer 本地类型 / import / field usage 落后。 | Dashboard client 使用旧 field name。 | producer route / schema check + consumer typecheck / targeted test。 |
| `runtime-generated-sync` | source contract 明确，生成命令确定，diff 只涉及 runtime / generated artifact。 | Plugin bundled runtime export list 落后 source。 | generation command + runtime smoke / package check。 |
| `fixture-schema-align` | 产品 schema 明确，fixture 偏离真实 schema，修复不改变产品行为。 | AlembicTest fixture 使用旧 response shape。 | fixture schema check + product contract probe。 |
| `export-boundary-fix` | public export 已存在但 consumer import path / package boundary 错误。 | AlembicCore export 可用，AlembicPlugin import 仍走内部路径。 | producer export list + consumer build / typecheck。 |
| `contract-anchor-probe` | 只新增或更新可 diff 的只读 anchor，不改变 runtime 行为。 | 生成 API surface report 或 endpoint sample snapshot。 | anchor generation + no product diff except snapshot/report。 |
| `breaking-compat-cleanup` | 最新 canonical contract 已明确，旧入口仅为兼容层，producer / consumer 验证和 negative guard 清楚。 | 删除 `External* -> HostAgent*` alias、旧 path re-export、旧字段 compatibility reader。 | producer build/typecheck + consumer typecheck/test + old token negative scan。 |

这些任务共同要求：

- 单次只处理一个 interface dossier。
- 单次只允许一个 producer-consumer 关系，除非只是只读清单。
- 必须有明确验证命令。
- 必须能回填 commit / diff / command output / report path。
- 不得改变用户可见行为。
- 不得保留新旧双分支兼容，除非总控确认存在真实存量数据或发布兼容要求。

## 自动化必须停止的任务

命中任一条件必须停止，交回总控裁决：

- 需要新增产品能力。
- 需要删除、降级、替换或迁移现有业务能力。
- 删除对象可能承载持久化历史数据读取、已发布包、用户安装目录、runtime cache 或真实外部输入兼容。
- 需要把 contract 下沉到 `AlembicCore`，但当前未确认 producer / consumer。
- producer / consumer 谁是正确 contract 不清楚。
- 多个 consumer 之间语义冲突。
- 需要改 HTTP / resident API 的业务语义。
- 需要改 AI provider capability、model fallback、unavailable path 或真实 provider 行为。
- 需要真实发布策略、版本策略或历史数据迁移。
- 只有“更干净 / 更统一”的抽象理由，没有失败或 drift 证据。
- 验证命令缺失、失败原因不属于本接口，或需要 `AlembicTest` 真实场景但没有 test exchange。

## 自动化分阶段

### Stage 0：Read-only Interface Inventory

总控或自动化先选择一个 `Interface Area`，再做只读扫描，不改代码。

输出：

- area plan。
- producer / consumer 列表。
- contract anchor 候选。
- 当前失败或 drift 证据。
- 自动化安全任务候选。
- 必须停下确认的候选。

建议扫描顺序：

1. `AlembicCore` public export 与消费者 import。
2. `Alembic` HTTP / resident API 与 `AlembicDashboard` / `AlembicPlugin` / `AlembicTest` 消费。
3. `AlembicPlugin` source / bundled runtime / MCP schema / skill export。
4. `AlembicAgent` provider contract 与 `Alembic` internal AI 调用方。
5. `AlembicTest` fixture / probe / report schema 与产品 contract。

Stage 0 完成定义：

- 选定 area 的主要 producer / consumer 被记录，或明确说明当前没有足够证据。
- 每个候选都有文件路径证据。
- 至少分出 `breaking-cleanup-safe`、`needs-control-decision`、`not-actionable` 三类。
- 对 breaking cleanup 候选写清旧入口、canonical 替代、consumer 更新范围和 negative guard。
- 不产生产品源码 diff。

### Stage 1：Pick One Minimal Dossier

总控从 Stage 0 选择一个自动化安全候选。

选择优先级：

1. 最新代码已经明确 canonical contract，旧兼容层仍残留。
2. 已有 typecheck / build / targeted test 失败。
3. 生成物明显落后源码且生成命令确定。
4. fixture 明显偏离真实 schema。
5. import/export boundary 错误且 producer export 已存在。

禁止优先选择：

- 最大接口面。
- 最抽象接口面。
- 需要多仓同时改的接口面。
- 需要新增 shared contract 的接口面。
- 只因为“命名更漂亮”但没有最新 canonical code 事实的接口面。

Stage 1 完成定义：

- 选中的 dossier 有唯一 producer。
- 有唯一主要 consumer。
- 允许改动和禁止改动清楚。
- 验证命令可以运行。

### Stage 2：Minimal Interface Repair

目标仓库只做最小修复。

允许：

- consumer 跟随 producer contract。
- 更新生成物。
- 更新 fixture。
- 修正 import/export path。
- 补 targeted verification。
- 删除旧路径 / 旧字段 / 旧函数名 / 旧 wrapper 兼容层。
- 新增旧入口 negative guard。

禁止：

- 抽新 shared abstraction。
- 改业务语义。
- 改多个无关 interface。
- 为未来接口预留空字段 / 空 adapter。
- 扩大到全仓重构。
- 保留新旧双分支读取。

Stage 2 完成定义：

- diff 只覆盖 dossier 声明范围。
- producer 和 consumer 验证都通过，或明确说明 producer 无需改动。
- 回填有 commit / diff summary / command output / risk。

### Stage 3：Controller Review And Queue Next

总控验收后只做一个判断：

```text
accepted / needs-rework / blocked / pick-next-dossier / stop
```

不要让执行窗口自动领取下一项。

## 自动化候选评分

总控可用轻量评分选择第一批任务：

| 因子 | 高分条件 |
| --- | --- |
| Evidence | 有明确失败命令、diff、fixture drift 或 runtime mismatch。 |
| Scope | 单 producer、单 consumer、单接口面。 |
| Safety | 不改变业务行为，不涉及删除 / 迁移 / 发布。 |
| Verification | producer + consumer 验证命令清楚。 |
| Reversibility | diff 小，容易复核和回退。 |
| Automation Fit | 目标仓库边界清楚，回填证据可机器读取。 |

建议第一批只领取总分高、且没有停止条件的任务。

## 自动化任务包模板

总控后续可把单个 dossier 转成任务包：

```text
任务包 ID：MRI-<interface-key>-<YYYY-MM-DD>
Design Key：MULTI-REPOSITORY-INTERFACE-OPTIMIZATION-2026-05-28
类型：consumer-stale-type-fix | runtime-generated-sync | fixture-schema-align | export-boundary-fix | contract-anchor-probe

Area:
Interface:
Producer:
Consumer:
Contract anchor:
Failure / drift evidence:

目标：
允许改动：
禁止改动：
验证命令：
回填要求：
停止条件：
```

## 建议总控第一步

如果用户选择自动化该需求，Design 建议总控不要立即派实现窗口。

第一步只创建 Stage 0 read-only inventory 任务：

```text
目标：为 MULTI-REPOSITORY-INTERFACE-OPTIMIZATION 建立第一版 Interface Inventory。
范围：先选择一个 Interface Area，再只读扫描该 area 的 producer-consumer 接口面。
输出：Area Plan + Interface Dossier 候选表，分为 automation-safe / needs-control-decision / not-actionable。
禁止：改代码、改配置、改总控状态、创建实现任务。
```

总控拿到 Stage 0 后，再决定是否把某个 dossier 转为自动化任务包。

## 与当前主线关系

这是后续维护型自动化候选，不应打断当前已经进行中的 Workspace Workflow Optimization。

如果总控进入无人值守模式，也应把本需求当作 `maintenance backlog`，一次只领取一个 dossier。

## 证据状态

证据状态：Design 方案准备完成；尚未做总控 Stage 0 代码事实扫描。

自动化前仍需总控复核真实仓库状态、当前失败命令和是否存在更高优先级主线。
