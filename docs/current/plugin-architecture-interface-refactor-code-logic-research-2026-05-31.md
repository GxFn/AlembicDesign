# Plugin Architecture Interface Refactor 代码逻辑与使用调研

Design Key：PLUGIN-ARCHITECTURE-INTERFACE-REFACTOR-2026-05-31
日期：2026-05-31
状态：code-logic-research / design-input
维护窗口：AlembicDesign
目标仓库：AlembicPlugin

## 触发与用户裁决

用户补充裁决：

```text
你直接进行代码逻辑调研，真实的代码实现和功能使用，我建议是不要做兼容，直接使用新接口与逻辑，确认好其他仓库对接就可以
```

用户追加裁决：

```text
按照你的建议，拆分职责，只保留 Plugin 需要的内容，然后清理删除
```

Design 对该裁决的执行解释：

- 本需求不以长期兼容层为目标，不做旧接口和新接口长期并行。
- 允许直接设计新接口 / 新逻辑，但必须先确认真实消费者和跨仓库契约。
- 对已存在的旧入口，推荐同阶段迁移消费者、更新测试与文档，然后删除旧入口。
- 对外部 Codex host 可见入口，若要改名或删除，必须把 plugin shell、skills、README、verify、smoke 和 marketplace/runtime 包一并迁移，不能只改 TS 内部实现。
- `daemon-server.js` / embedded HTTP runtime 纳入本需求后半段，不另起独立 runtime simplification 需求；处理顺序是先拆职责和 consumer map，再保留 Plugin 必需内容并清理删除主体式多余能力。

## 关键结论

AlembicPlugin 的真实使用面分成三层：

1. Codex 插件分发契约：`channel.json`、`.mcp.json`、wrapper、`runtime.tgz`、`alembic-codex-mcp` binary、skills、marketplace 和 release/smoke scripts。这是 Codex host 直接消费的外层契约。
2. Alembic resident service 契约：Plugin 通过 `@alembic/core/daemon` contract 和 Alembic daemon HTTP API 读取 `residentService`、ProjectScope、search、jobs、IntentEpisode 和 Dashboard handoff。
3. Plugin embedded fallback 契约：Plugin 自带 compiled runtime、embedded Core snapshot 和 daemon-server，用于 Codex host-agent recovery / local MCP 可用性；它不是 Alembic 主体 daemon、Dashboard、file monitor 或 internal AI 的 source of truth。

因此本需求的“新接口”应该定义在 Plugin 功能区边界，而不是把所有旧文件简单改名：

- 对 Codex host：保留一个清晰、单源的 Plugin surface catalog。
- 对 Alembic / Core：只消费 public contracts，不跨仓库 import 内部源码。
- 对 Plugin 内部：删除 legacy compatibility surface，迁移到新功能区接口。

## 跨仓库真实消费者矩阵

| 消费者 / 生产者 | 真实使用方式 | 代码证据 | 重构影响 |
| --- | --- | --- | --- |
| Codex host -> AlembicPlugin | 通过 `.mcp.json` 启动 `node ./bin/alembic-codex-mcp-wrapper.mjs`，wrapper 再使用 embedded `runtime.tgz` 启动 `alembic-codex-mcp`。 | `../AlembicPlugin/plugins/alembic-codex/.mcp.json`；`../AlembicPlugin/channels/codex/channel.json` | 新接口必须同步 plugin shell、runtime tarball、wrapper、channel 和 marketplace。 |
| Plugin release / verification -> runtime artifact | `prepare-codex-plugin-runtime.mjs` 要求 `dist/bin/codex-mcp.js`、`dist/bin/daemon-server.js`、`dist/lib/external/mcp/CodexMcpServer.js` 存在，并打进 runtime package。 | `../AlembicPlugin/scripts/prepare-codex-plugin-runtime.mjs` | 不能只删 `daemon-server.js` 或 `CodexMcpServer`；若更名/拆分，脚本和 smoke 必须同阶段改。 |
| Plugin smoke -> package contents | smoke 明确检查 package 与 embedded runtime 内同时存在 codex MCP、daemon-server、CodexMcpServer、Core wasm、runtime package、skills、README、release playbook。 | `../AlembicPlugin/scripts/smoke-codex-plugin.mjs` | 新 runtime contract 要同时改 package file list 和 smoke expected files。 |
| AlembicCore -> Plugin | Core 明确禁止 Codex / MCP / plugin runtime 目录、依赖和实现文件进入 Core。 | `../AlembicCore/test/CoreCodexBoundary.test.ts` | Core 只承接共享契约，不能承接 Plugin runtime / MCP SDK implementation。 |
| AlembicCore -> resident contracts | Core 已提供 resident service owner / route / capability / ProjectScope / host-agent workflow contract。 | `../AlembicCore/test/ResidentServiceContracts.test.ts`；`../AlembicCore/test/ProjectScopeContracts.test.ts` | 新接口应优先复用 Core public contract，必要时补 Core contract，而不是在 Plugin 复制结构。 |
| Alembic daemon -> Plugin | Alembic `/api/v1/daemon/health` 返回 `residentService` 和 `runtimeBoundary`；Plugin 以 residentService 为 canonical，runtimeBoundary 只是旧 fallback。 | `../Alembic/lib/http/routes/daemon.ts`；`../AlembicPlugin/lib/codex/EnhancementRoute.ts` | 按用户“不兼容”裁决，runtimeBoundary fallback 可作为删除候选，但要确认当前 Alembic health 已覆盖所有 Plugin 消费字段。 |
| Alembic daemon HTTP -> Plugin resident client | Plugin resident client 调用 `/api/v1/search`、`/api/v1/jobs`、`/api/v1/project-scope/resolve-folder`、`/api/v1/intent-episodes`、`/api/v1/daemon/health`。 | `../AlembicPlugin/lib/service/resident/AlembicResidentServiceClient.ts`；`../Alembic/lib/http/HttpServer.ts` | 新 resident adapter 应按 capability 拆分，但 endpoint contract 必须由 Alembic / Core 保持清晰。 |
| AlembicDashboard -> Alembic daemon | Dashboard 直接消费 Alembic daemon health、runtimeBoundary、ProjectScope endpoints，不消费 Plugin API。 | `../AlembicDashboard/src/api.ts`；`../AlembicDashboard/src/components/Layout/ProjectScopePanel.tsx` | Plugin 重构不应给 Dashboard 派任务；Dashboard 只需观察 daemon contract 是否变化。 |
| Alembic templates / Plugin skills -> tool names | Alembic 模板仍提到 `alembic_skill`；Plugin skill 明确说 Codex-facing replacement 是 `alembic_project_skill`，`alembic_skill` 只是 compatibility alias。 | `../Alembic/templates/instructions/agent-static.md`；`../AlembicPlugin/plugins/alembic-codex/skills/alembic/SKILL.md` | 删除 Plugin `alembic_skill` alias 时必须区分 Alembic resident canonical skill tool，不要误删 Alembic 主体工具。 |

## 真实功能链路

### 1. Codex 插件启动链路

```text
Codex host
-> plugins/alembic-codex/.mcp.json
-> plugins/alembic-codex/bin/alembic-codex-mcp-wrapper.mjs
-> plugins/alembic-codex/runtime.tgz
-> runtime package bin.alembic-codex-mcp
-> dist/bin/codex-mcp.js
-> CodexMcpServer
```

关键证据：

- `../AlembicPlugin/plugins/alembic-codex/.mcp.json`
- `../AlembicPlugin/plugins/alembic-codex/runtime/package.json`
- `../AlembicPlugin/scripts/verify-codex-plugin.mjs`
- `../AlembicPlugin/scripts/smoke-codex-plugin.mjs`

设计判断：

- `PluginRuntimeContract` 应该成为单独 interface area，覆盖 plugin shell、channel、runtime package、binary、wrapper、skills 和 verification。
- 如果重命名或拆分 `CodexMcpServer` / `daemon-server`，必须同步 `prepare-codex-plugin-runtime`、`verify-codex-plugin`、`smoke-codex-plugin` 和 release playbook。

### 2. Codex tool surface 链路

```text
CodexMcpServer list_tools
-> CODEX_LOCAL_TOOLS
-> embedded core TOOLS
-> ToolPolicy knowledge gate / tier / admin gate
-> visible tool list
```

关键证据：

- `../AlembicPlugin/lib/codex/ToolPolicy.ts`
- `../AlembicPlugin/lib/external/mcp/tools.ts`
- `../AlembicPlugin/test/unit/CodexToolPolicy.test.ts`
- `../AlembicPlugin/test/unit/CodexMcpServer.test.ts`

真实问题：

- Codex local tools 和 core MCP tools 分散维护。
- `alembic_project_skill` 是 Codex-facing 新工具；`alembic_skill` 在 Plugin 中已经是 legacy compatibility alias。
- Tool 数量注释已经漂移，说明 catalog 没有单一事实源。

新接口建议：

- 建立 `PluginToolSurfaceCatalog`，集中 tool id、owner、schema、annotation、gateway、visibility gate、handler owner、resident route policy。
- 删除 Plugin Codex-facing `alembic_skill` compatibility alias，迁移 Plugin skill / tests / README 到 `alembic_project_skill`；Alembic resident 的 `alembic_skill` 是否改变不属于本需求第一阶段。

### 3. Resident service 链路

```text
AlembicResidentServiceClient
-> /api/v1/daemon/health
-> residentService capabilities
-> /api/v1/search | /api/v1/jobs | /api/v1/project-scope/resolve-folder | /api/v1/intent-episodes
-> Plugin prime/search/job/dashboard surfaces
```

关键证据：

- `../AlembicPlugin/lib/service/resident/AlembicResidentServiceClient.ts`
- `../Alembic/lib/http/routes/daemon.ts`
- `../Alembic/lib/http/routes/search.ts`
- `../Alembic/lib/http/routes/jobs.ts`
- `../Alembic/lib/http/routes/project-scope.ts`
- `../Alembic/lib/http/routes/intent-episodes.ts`

真实问题：

- 一个 `AlembicResidentServiceClient` 同时处理 probe、ProjectScope、search、IntentEpisode、jobs、Dashboard、fallback 和 telemetry compact。
- 旧 `runtimeBoundary` fallback 仍在 `EnhancementRoute` 和 `ModuleBoundary` 中保留，但 Alembic 当前 health 已返回 canonical `residentService`。

新接口建议：

- 拆成 `ResidentProbeClient`、`ResidentProjectScopeClient`、`ResidentSearchClient`、`ResidentIntentEpisodeClient`、`ResidentJobClient`、`ResidentDashboardClient`。
- facade 可以在同阶段过渡，但不能成为长期兼容层；最终调用方按 capability client 消费。
- 删除 runtimeBoundary fallback 的前置证据：当前 Alembic daemon health producer 已覆盖 residentService；Plugin dashboard/status/job/search path 不再需要 fallback；相关 tests 更新为 residentService canonical。

### 4. Prime / intent / search 交付包链路

```text
alembic_task prime
-> host intent normalization / extraction
-> IntentSearchPlan
-> PrimeSearchPipeline
-> resident semantic search if available
-> relation / evidence / primeInjectionPackage
-> visible receipt / shout instruction
-> IntentEpisode start / latest / recent
```

关键证据：

- `../AlembicPlugin/lib/external/mcp/handlers/task.ts`
- `../AlembicPlugin/lib/service/task/HostIntentFrame.ts`
- `../AlembicPlugin/lib/service/task/PrimeSearchPipeline.ts`
- `../AlembicPlugin/lib/service/resident/AlembicResidentServiceClient.ts`
- `../Alembic/lib/http/routes/intent-episodes.ts`

新接口建议：

- 把 prime 拆成 `PrimeIntentIntake`、`PrimeSearchOrchestrator`、`PrimeInjectionPackageBuilder`、`PrimeVisibleReceiptPresenter`、`IntentEpisodeHandoffClient`。
- Recipe source file path、sourceRefs、evidenceRefs、IntentEpisode sourceRefs 是硬证据，不得在拆分中丢失。
- 低置信度 / resident unavailable 应保留 telemetry，不得伪装成失败。

### 5. Embedded runtime / HTTP / daemon 链路

```text
alembic_codex_* local tool
-> DaemonSupervisor
-> dist/bin/daemon-server.js
-> HttpServer
-> jobs / health / search / project-scope / intent-episodes / embedded JobStore
```

关键证据：

- `../AlembicPlugin/lib/daemon/DaemonSupervisor.ts`
- `../AlembicPlugin/bin/daemon-server.ts`
- `../AlembicPlugin/lib/http/HttpServer.ts`
- `../AlembicPlugin/scripts/prepare-codex-plugin-runtime.mjs`

真实边界：

- Plugin embedded runtime 当前仍需要 `daemon-server.js` 和 HTTP routes，至少用于 host-agent recoverable jobs、status / diagnostics、runtime package smoke。
- 它不是 long-lived Alembic daemon source of truth；Alembic 主体 daemon 才拥有 Dashboard、internal AI、native file monitor 和 resident service。

新接口建议：

- 建立 `EmbeddedRuntimeCompatibilityContract`，列出必须保留的 route 和可删除 route。
- 删除“旧 Alembic 主体式目录感”的方式不是直接删目录，而是先把 Package/runtime smoke 中实际需要的 route 写成 contract，再裁剪不被 contract 消费的 route。
- `daemon-server.js` 纳入本需求 Stage 5：先拆分 runtime start、recoverable job/status、health、packaged smoke 支撑能力，再删除 Alembic 主体式 daemon / HTTP 能力外观中未被 Plugin 消费的部分。

## 可删除 / 可迁移候选

以下是候选，不代表 Design 直接执行删除；总控接收后应作为 Stage 0 / Stage 1 实现输入。

| 候选 | 当前用途 | 推荐动作 | 需要同步的消费者 |
| --- | --- | --- | --- |
| Plugin Codex-facing `alembic_skill` alias | `alembic_project_skill` 的 legacy compatibility alias；可见性测试已默认隐藏多数场景。 | 删除 Plugin alias，全部迁移到 `alembic_project_skill`。 | Plugin tools、schemas、consolidated handler、skill docs、Codex tests、README；不直接删除 Alembic resident `alembic_skill`。 |
| `runtimeBoundary` fallback in Plugin enhancement route | 旧 daemon health payload fallback；当前 Alembic health 已有 `residentService`。 | 以 `residentService` 为唯一新接口；删除 fallback path 和 compatibility diagnostics。 | `EnhancementRoute`、`ModuleBoundary`、相关 unit tests；确认 Dashboard 不依赖 Plugin fallback。 |
| `ALEMBIC_CHANNEL` env fallback | `ALEMBIC_CHANNEL_ID` 的 legacy fallback。 | 删除或降为明确 unsupported migration note。 | `.mcp.json`、RuntimeContext、shared channel tests/docs。 |
| Tool catalog 注释 / 分散 map | 分散维护导致数量漂移。 | 新建单源 catalog 后删除旧分散注释和重复常量。 | `ToolPolicy`、`tools.ts`、`McpServer` gateway / handler registry、tests。 |
| Resident client 单体 facade | 所有 resident API 混在一个类。 | 同阶段迁移调用方到 capability clients；保留 facade 只作为短期内部 assembly，不作为 public compatibility。 | task/search/dashboard/job handlers、PrimeSearchPipeline、tests。 |

## 不应直接删除的契约

| 契约 | 原因 |
| --- | --- |
| `runtime.tgz` / wrapper / `.mcp.json` / marketplace | Codex host 直接启动依赖。 |
| `dist/bin/codex-mcp.js` | Plugin MCP binary 入口。 |
| `dist/bin/daemon-server.js` | 当前 release/smoke 和 embedded recoverable job path 依赖；纳入 Stage 5 拆分职责和清理删除，不允许未替换直接删除。 |
| Core resident / ProjectScope contracts | Alembic、Plugin、Dashboard 的共享语义来源。 |
| Recipe sourceRefs / source file path evidence | 用户已明确要求这是关键证据。 |
| Alembic resident `alembic_skill` | 它是 Alembic 主体 resident tool，不等于 Plugin compatibility alias。 |

## 阶段建议

### Stage 0：Interface Contract Dossier

只读输出：

- `PluginRuntimeContract`：plugin shell、runtime package、wrapper、channel、marketplace、skills、smoke。
- `PluginToolSurfaceContract`：tool id、schema、visibility、gateway、handler、owner。
- `ResidentServiceConsumerContract`：health/search/jobs/project-scope/intent-episodes/dashboard endpoints。
- `EmbeddedRuntimeCompatibilityContract`：当前 runtime 必需 HTTP routes 和可裁剪候选。
- `CrossRepoConsumerMap`：AlembicCore / Alembic / AlembicDashboard / AlembicPlugin 对接关系。

### Stage 1：Tool Surface 新接口

- 建立单源 tool catalog。
- 删除 Plugin `alembic_skill` compatibility alias，迁移到 `alembic_project_skill`。
- 更新 Codex visible tools、schema、tests、skills、README。
- 验证：CodexToolPolicy、CodexMcpServer list/call、Zod schema、verify-codex-plugin。

### Stage 2：Codex Execution Router 拆分

- `CodexMcpServer` 拆出 projectRoot scope、preflight、local tool dispatcher、embedded executor、resident route choice、evolution surface attachment。
- 不改变外部 MCP tool name / input / output，除 Stage 1 已删除的 explicit legacy alias。
- 验证：CodexMcpServer focused tests、ServiceRequestBoundary、smoke stdio。

### Stage 3：Resident Adapter 新接口

- 拆 capability clients。
- 删除 runtimeBoundary fallback。
- 以 Core residentService contract 为唯一 capability source。
- 验证：ResidentService contracts、PrimeSearchPipeline resident tests、dashboard/job/status tests。

### Stage 4：Prime / Intent Package 新接口

- 拆 intake、search orchestration、injection package、visible receipt presenter、IntentEpisode handoff。
- 固定 sourceRefs / evidenceRefs / relation / vector / intent 汇总交付包。
- 验证：prime output snapshot、IntentEpisode route tests、search telemetry tests。

### Stage 5：Embedded Runtime Compatibility 收敛

- 基于 Stage 0 contract 拆分 `daemon-server.js`、HTTP、ServiceContainer compatibility 的职责。
- 只保留 Plugin 需要的 runtime start、recoverable job/status、health、packaged smoke 支撑能力。
- 清理删除 Alembic 主体式 daemon / HTTP 能力外观中未被 Plugin 消费的部分；若最终移除 `daemon-server.js`，必须先替换 `alembic_codex_*` local tool recovery path 和 runtime smoke。
- 验证：prepare runtime、verify plugin、smoke plugin、release playbook checks。

## 建议总控下一步

1. 接收本需求为 `ready-for-workspace` 的 requirement handoff。
2. 不等待新的兼容确认；用户已确认“不做兼容，直接新接口与新逻辑”。
3. 总控接收后先派 Stage 0 只读 Interface Contract Dossier；每个后续阶段必须先列完整 area plan 再改代码。
4. 执行要求：所有旧入口删除必须同阶段迁移真实消费者、更新 tests / docs / smoke，不允许留下长期 dual path。
5. 不打断当前 Plugin 主线；由总控根据 `PLUGIN-BUNDLE-IDE039-P1` 回填情况决定插入时机。

## 仍需总控裁决的问题

- 当前 Plugin 主线回填后，是否先执行本需求 Stage 0，还是先做已有 038/039/IDE Agent 主线收敛。
- Alembic resident `alembic_skill` 是否未来也要重命名；这超出本 Plugin-only 需求，应另起 Alembic 主体接口需求。
