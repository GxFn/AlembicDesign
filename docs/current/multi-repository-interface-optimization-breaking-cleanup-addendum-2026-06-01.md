# Multi Repository Interface Optimization Breaking Cleanup 补充设计

Design Key：MULTI-REPOSITORY-INTERFACE-OPTIMIZATION-2026-05-28
补充日期：2026-06-01
状态：ready-for-workspace / breaking-cleanup addendum
维护窗口：AlembicDesign
总控：AlembicWorkspace
主需求设计：[multi-repository-interface-optimization-requirement-design-2026-05-28.md](multi-repository-interface-optimization-requirement-design-2026-05-28.md)
自动化准备附录：[multi-repository-interface-optimization-automation-readiness-2026-05-30.md](multi-repository-interface-optimization-automation-readiness-2026-05-30.md)

## 用户裁决

2026-06-01 用户确认：当前仍处于开发阶段，可以彻底清理兼容层。多仓库接口优化不再默认按“保守兼容”执行，而应按照 Alembic 主体和 AlembicPlugin 最新提交后的 canonical 命名、方法名和目录层级，去除冗余兼容分叉，保持接口契约干净整洁。

这不是授权 `AlembicDesign` 修改产品源码。Design 只整理需求设计、代码事实、任务边界和 handoff 建议；是否入账、派发、验收仍由 `AlembicWorkspace` 总控裁决。

## 用户确认记录

| 时间 | 确认项 | 影响 |
| --- | --- | --- |
| 2026-06-01 | 确认本需求正式采用开发期 breaking cleanup 口径。 | 不默认保留旧兼容分叉；Stage 0 目标改为确认 canonical producer / consumer 和可删除兼容层清单。 |
| 2026-06-01 | 确认第一优先级为 `core-host-agent-workflow-contract`。 | 总控接收后建议先处理 Core `External*` / `external-agent` / `*-external` 到 `HostAgent*` / `host-agent` 的 contract 收敛。 |
| 2026-06-01 | 确认第二优先级为 `api-ai-runtime-contract`。 | Core / Alembic / Dashboard / Plugin 应同步从 `internalAi` / `jobs.internal-ai.*` 收敛到 `apiAi` / `jobs.api-ai.*`，并删除 Alembic `ApiAiCompatibility.ts`。 |
| 2026-06-01 | 确认删除边界：纯旧命名、旧路径、旧字段、旧 wrapper、旧测试 fixture 可以删除；若涉及持久化历史数据、已发布包安装兼容、runtime cache 或真实外部输入则停止回总控。 | 总控可把兼容层删除列为可执行任务，但每个 dossier 必须写清 stop condition。 |
| 2026-06-01 | 确认 contract 字段和用户可见文案尽量统一为 `API AI`，避免继续混用 `internal AI`。 | Dashboard / Plugin docs / skill / status 文案跟随 `apiAi` contract 收敛。 |
| 2026-06-01 | 确认小项 `external_agent` role id 可改为 `host_agent`。 | 进入 Plugin constitution / tests / fixture 的 breaking cleanup 候选；需总控先复核是否有存量 auth 数据依赖。 |
| 2026-06-01 | 用户说明 Alembic `lib/external/mcp/README.md` retired marker 由用户自行删除。 | Design 不再把该文件删除作为任务包目标；总控 Stage 0 只需只读复核最新产品状态和 negative guard。 |

## 更新后的最终目标

在不保留历史兼容分叉的前提下，让 Alembic 系列仓库接口只剩当前 canonical contract：

- 新代码、新测试、新 runtime artifact 和新文档只引用 canonical 名称、路径和字段。
- 删除仅服务旧命名、旧目录、旧语义的 shim / alias / compatibility bridge。
- 保留真实能力和真实数据读取闭环；删除兼容层不能降级业务能力、测试可信度或用户可见行为。
- 如果某个旧入口承载存量数据迁移、历史记录读取、发布版本过渡或外部安装兼容，必须先交给总控裁决，不能自动删除。

## 本次只读代码事实

### Alembic

已读取提交：

- `c0ddb70 refactor: flatten host bridge module paths`
- `fd74bac refactor: clarify workflow api ai execution`

已确认 canonical 方向：

- `lib/repository/AuditRepository.ts`
- `lib/tools/v2/ToolContextFactory.ts`
- `lib/workflows/ai-execution/**`
- `lib/workflows/cold-start/ColdStartWorkflow.ts`
- `lib/workflows/knowledge-rescan/KnowledgeRescanWorkflow.ts`
- `lib/resident/tool-handlers/cold-start.ts`
- `lib/resident/tool-handlers/knowledge-rescan.ts`

已确认旧运行 import 不再出现：

- `#tools/v2/adapter/ToolContextFactory`
- `tools/v2/adapter/ToolContextFactory`
- `repository/audit/AuditRepository`
- `lib/external/mcp` TypeScript runtime module

仍需清理或裁决：

- `biome.json` 仍包含已不存在的 `lib/workflows/capabilities/execution/internal-agent/BootstrapProjections.ts` override。
- `lib/resident/tool-handlers/consolidated.ts` 注释仍写 `bootstrap-external.js（外部 Agent 路径）`。
- `lib/external/mcp/README.md` retired boundary marker：用户已说明由其自行删除；Design 2026-06-01 只读复核时当前本机工作树仍可见该文件且无删除状态，因此总控 Stage 0 应只读复核最新 product 状态。若仍存在，应按用户确认删除或转为 negative guard；若已删除，只需确认旧 `lib/external/mcp` 不再作为 runtime 入口。
- `ApiAiCompatibility.ts` 仍桥接 `apiAi` 与 Core `internalAi` contract；这是跨 Core / Alembic / Dashboard / Plugin 的真实兼容分叉，不能只在 Alembic 单仓删除。

### AlembicPlugin

已读取提交：

- `9ff9a6e Separate host-agent route smoke from Go support`
- `c2c7479 Clarify Codex MCP runtime layout`
- `16f50a8 Rename host-agent MCP handlers`
- `273ecb0 Move MCP surface under Codex`

已确认 canonical 方向：

- `lib/codex/mcp/**`
- `lib/codex/mcp/handlers/host-agent/**`
- `lib/codex/mcp/host-agent-workflows/**`

已确认旧运行 import 不再出现：

- `lib/external/mcp`
- `#external/mcp`
- `bootstrap-host-agent`
- `bootstrap-external` source route modules

仍需清理或裁决：

- `lib/codex/mcp/handlers/host-agent/bootstrap.ts`、`rescan.ts`、`dimension-completion.ts` 仍以 `Compatibility export / adapter` 描述当前 canonical host-agent wrapper，应改成 transport adapter / MCP boundary wording。
- `lib/codex/mcp/host-agent-workflows/*` 从 `@alembic/core/host-agent-workflows` import `External*` API 后再 alias 为 `HostAgent*`，说明 Core public facade 还没有真正改成 host-agent 语义。
- `config/constitution.yaml` 与 `templates/constitution.yaml` 角色 id 仍为 `external_agent`，name 已改成 `Codex Host Agent`；用户已确认 role id 可改为 `host_agent`，执行前仍需检查存量 auth / tests / fixture 是否承载该 id。

### AlembicCore

Core 是本次最大 producer 依赖。

当前 `@alembic/core/host-agent-workflows` facade 名称已经是 host-agent，但内部和导出仍使用大量 `External*` 语义：

- `createExternalWorkflowSession`
- `ExternalWorkflowSession`
- `ExternalSubmissionTracker`
- `runExternalDimensionCompletionWorkflow`
- `createExternalColdStartIntent`
- `createExternalKnowledgeRescanIntent`
- `projectExternalRescanEvidencePlan`
- `profile: 'cold-start-external' | 'rescan-external'`
- `executor: 'external-agent'`
- `completionPolicy: 'external-dimension-complete'`
- `sourceTag: 'bootstrap-external' | 'rescan-external'`

这会迫使 Plugin / Alembic 继续写 alias 和 external 测试名。若目标是彻底清理兼容层，Core 必须先提供 host-agent canonical API，并删除或同步替换 External 命名。

Core daemon contract 仍以 `internalAi`、`jobs.internal-ai.bootstrap`、`jobs.internal-ai.rescan` 表达 Alembic API AI runtime。Alembic 已出现 `apiAi` 命名，Dashboard 和 Plugin 仍消费 `internalAi`。用户已确认 API AI 是新的 canonical 语义，因此 Core daemon contract 必须迁移到 `apiAi` / `jobs.api-ai.*`，再删除 Alembic 的兼容桥。

### AlembicDashboard

Dashboard 仍消费 `runtimeBoundary.capabilities.internalAi` 和 `RuntimeInternalAiCapability`。它是 Core/Alembic daemon contract 的 consumer，不应自己发明 `apiAi` 兼容分叉。若 Core/Alembic producer 迁移到 `apiAi`，Dashboard 应同步使用新字段并删除 `internalAi` 读取。

### AlembicAgent

AlembicAgent 主要消费 `@alembic/core/host-agent-workflows` 的 terminal/pipeline helpers。当前没有发现它消费 Alembic / Plugin 旧路径。若 Core 把 `WorkflowExecutor` 从 `external-agent` 改为 `host-agent`，Agent 的测试和 runtime boundary 需要同步。

### AlembicTest

本轮未把 `AlembicTest` 作为修复目标。它当前工作区有未提交测试资产，Design 不触碰。后续如果 API response / runtime contract 改名，`AlembicTest` 只跟随产品 contract 更新 fixture / probe；不能作为旧字段继续存在的理由。

## 新的 breaking-cleanup 原则

1. 开发阶段允许 breaking change，旧路径 / 旧字段 / 旧函数名不需要为外部用户长期保留。
2. breaking cleanup 只能删除“兼容旧命名或旧层级”的代码，不能删除真实业务能力。
3. canonical producer 必须先确定，再让 consumer 跟随；不能在 consumer 里临时同时支持两套字段。
4. 代码、测试、生成物、runtime artifact 和设计文档必须同一轮收敛，否则会留下新的兼容分叉。
5. 删除兼容层后必须加 negative guard：旧 import、旧路径、旧字段、旧字符串不能重新出现。
6. 如果旧字段涉及持久化历史数据、已发布包、用户安装目录、runtime cache 或真实外部输入，必须停下交给总控裁决。

## 推荐阶段候选

### Stage 0：Breaking Cleanup Area Plan

总控先选择本次 breaking cleanup 的首个 area。Design 推荐顺序：

1. `core-host-agent-workflow-contract`
2. `api-ai-runtime-contract`
3. `plugin-codex-mcp-runtime-layout`
4. `alembic-host-bridge-flat-paths`
5. `test-fixture-contract-followup`

Stage 0 输出完整 area plan，不改源码：

```text
Area:
Canonical producer:
Canonical names / paths / fields:
Consumers:
Old compatibility branches:
Delete candidates:
Must-migrate consumers:
Negative guard:
Verification matrix:
Stop conditions:
```

### Stage 1：Core host-agent workflow contract cleanup

目标：`@alembic/core/host-agent-workflows` 对外只暴露 host-agent 语义。

建议变更范围：

- `External*` 类型、函数、class、file name 改为 `HostAgent*`。
- `external-agent` 改为 `host-agent`。
- `external-dimension-complete` 改为 `host-agent-dimension-complete`。
- `bootstrap-external` / `rescan-external` profile/sourceTag 改为 host-agent 命名。
- 更新 Core public API smoke / package tests。
- 同步 Alembic / Plugin / Agent 真实 imports 和 tests。
- 删除 Plugin 中 `External* as HostAgent*` alias。

完成定义：

- `rg "createExternal|ExternalDimension|ExternalWorkflow|ExternalSubmission|projectExternal|buildExternal|presentExternal|external-agent|external-dimension-complete|bootstrap-external|rescan-external"` 在 source/test 中只剩非本语义的普通英文或历史归档，不存在 runtime contract。
- Core producer test 和 Alembic / Plugin / Agent consumer typecheck 通过。
- Plugin host-agent workflow files 不再通过 alias 修正 Core 命名。

### Stage 2：API AI runtime contract cleanup

目标：Alembic daemon / resident / Dashboard / Plugin 对 API AI runtime 使用同一套 `apiAi` contract。

建议变更范围：

- Core daemon contract：`AlembicInternalAiCapability`、`ProjectRuntimeInternalAiSummary`、`internalAi` 字段、`jobs.internal-ai.*` feature 迁移为 `ApiAi` / `apiAi` / `jobs.api-ai.*`。
- Alembic 删除 `ApiAiCompatibility.ts`，直接生产 Core canonical `apiAi`。
- Plugin `EnhancementRoute`、`AlembicResidentServiceClient`、status 文案同步 `apiAi`。
- Dashboard `RuntimeInternalAiCapability` 和 `runtimeBoundary.capabilities.internalAi` 消费同步为 `apiAi`。
- docs / skill 文案中可继续面向用户解释 “API AI”，但不再混用 “internal AI” 作为 contract 字段。

完成定义：

- `rg "internalAi|internal-ai|InternalAi|Internal AI"` 在 runtime source 中只剩用户文案或明确非 contract 语义。
- `rg "ApiAiCompatibility"` 无 source 命中。
- Core daemon tests、Alembic daemon health tests、Dashboard typecheck、Plugin resident/status tests 同步通过。

### Stage 3：Plugin Codex MCP runtime layout residue cleanup

目标：Plugin 只保留 `lib/codex/mcp/**` canonical runtime layout。

建议变更范围：

- 确认 `scripts/prepare-codex-plugin-runtime.mjs`、`smoke-codex-plugin.mjs`、`verify-codex-plugin.mjs` 已指向 `dist/lib/codex/mcp/CodexMcpServer.js`。
- host-agent handler wrapper 注释从 compatibility 改为 transport / MCP boundary。
- 清理 constitution `external_agent` role id 到 `host_agent`。用户已确认可改；若 role id 是数据 contract，必须先做一次 source/test 搜索和 migration 判断；如果只是模板开发期命名，直接改。
- 刷新 plugin runtime artifact / vendored Core 后，确认 runtime 中不再出现旧 `lib/external/mcp`。

完成定义：

- Plugin source、scripts、tests、runtime artifact 都不再消费旧 MCP layout。
- 旧 handler 名称 `bootstrap-host-agent` / `*-external` 不再作为 runtime route。
- MCP tool names 保持用户可见稳定：`alembic_bootstrap`、`alembic_rescan` 等不因内部清理改变。

### Stage 4：Alembic host bridge flat path residue cleanup

目标：Alembic 主体只保留最新扁平 host bridge 路径。

建议变更范围：

- 只读确认 `lib/external/mcp/README.md` retired marker 最新状态；用户已说明由其自行删除，若总控最新复核仍看到该文件，再按用户确认处理或转为 negative guard。
- 更新 `biome.json` stale path override。
- 更新 `consolidated.ts` 旧注释。
- 确认 release staging / `.release` / generated dist 不携带 `lib/repository/audit`、`lib/tools/v2/adapter/ToolContextFactory` 旧路径；如存在，走正常 clean/build/release artifact 刷新，不手工 patch dist。

完成定义：

- `rg "#tools/v2/adapter/ToolContextFactory|repository/audit/AuditRepository|lib/tools/v2/adapter/ToolContextFactory|lib/repository/audit/AuditRepository"` source/test/scripts 无命中。
- `find lib/tools/v2 lib/repository` 不存在旧目录层级。
- negative test 覆盖旧路径不回归。

### Stage 5：Test fixture follow-up

目标：测试资产只验证新 contract，不把旧字段当成功路径。

建议变更范围：

- `AlembicTest` 仅在 producer / consumer contract 完成后跟随更新。
- 若真实 scenario report 中引用旧路径，只作为历史证据保留，不作为当前 fixture。
- 新 fixture 必须引用 canonical `host-agent` / `apiAi` contract。

完成定义：

- 新 probe / fixture 不再使用旧字段。
- 真实场景测试如需启动，由总控通过 `AlembicTest` 测试单裁决；Design 不创建测试单。

## 任务包模板更新

```text
任务包 ID：MRI-BREAKING-<area>-<YYYY-MM-DD>
Design Key：MULTI-REPOSITORY-INTERFACE-OPTIMIZATION-2026-05-28
类型：breaking-compat-cleanup

Area:
Canonical producer:
Canonical contract:
Consumers:
Delete candidates:
Must-update files:
Forbidden:
  - 不删除真实业务能力
  - 不保留双分支兼容
  - 不改用户可见 MCP tool name / CLI command，除非总控确认
Negative guard:
Verification:
Backfill evidence:
Stop condition:
```

## 更新后的停止条件

允许删除：

- 旧路径 re-export。
- 旧字段 alias。
- 旧函数名 alias。
- 旧测试 fixture。
- 只服务旧命名的 compatibility adapter。
- stale marker / stale doc / stale lint override。

必须停止：

- 删除会改变用户可见功能。
- 删除会破坏持久化历史数据读取。
- 删除会改变已发布 package 的安装 / runtime cache 兼容，而总控未确认开发期可破坏。
- producer 未确定，consumer 之间冲突。
- 需要跨仓同步但缺少任一 consumer 的验证命令。
- 需要真实项目验证但未通过总控创建 `AlembicTest` 测试单。

## Workspace 交接建议

该补充设计已满足 `ready-for-workspace`。建议总控接收后把原 `MULTI-REPOSITORY-INTERFACE-OPTIMIZATION-2026-05-28` 从保守 maintenance backlog 升级为 `breaking-cleanup maintenance`。

首个总控动作仍应是只读 Stage 0，但 Stage 0 的目标不再是“找低风险兼容修复”，而是“确认 canonical producer / consumer 和可删除兼容层清单”。之后先派 `Core host-agent workflow contract cleanup`，因为它是 Plugin 和 Alembic 继续保留 `External* -> HostAgent*` alias 的上游原因。
