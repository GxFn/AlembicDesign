# Plugin Architecture Interface Refactor 代码事实复核

Design Key：PLUGIN-ARCHITECTURE-INTERFACE-REFACTOR-2026-05-31
日期：2026-05-31
状态：code-fact-review / superseded-by-code-logic-research
维护窗口：AlembicDesign
目标仓库：AlembicPlugin

## 结论摘要

AlembicPlugin 当前不是“缺能力”，而是能力已经连续叠加后，接口边界不够清晰：

- Codex MCP shim 很轻，但后面接了两层 MCP server：`CodexMcpServer` 负责 Codex 可见工具、projectRoot、preflight、daemon / resident route；内层 `McpServer` 仍是完整 Alembic V3 tool server 和 DI container。
- Plugin 已有正确边界种子：`ToolPolicy`、`Preflight`、`ServiceRequestBoundary`、`ModuleBoundary`、`ProjectSkillService`、`ProjectSkillDelivery`、`PluginOpportunisticEvolution` 都在表达“插件只拥有 Codex-facing surface 与 adapter”。
- 风险集中在结构承载方式：大文件承载多个职责、tool catalog 分散、resident client 单体化、HTTP / daemon / service container 保留主体仓库形态，容易让后续实现窗口误判 Plugin 又拥有主体 runtime、file monitor、Dashboard 或 internal AI。
- 本次重构应优先做功能区和接口边界重组。用户已补充确认“不做兼容，直接使用新接口与逻辑”；因此旧入口不应长期保留，但删除必须和真实消费者迁移、测试、文档、runtime packaging 同阶段完成。

## 真实入口链路

```text
bin/codex-mcp.ts
-> ensureCodexRuntimeEnvironment()
-> startCodexMcpServer()
-> CodexMcpServer.registerHandlers()
-> preflightCodexTool()
-> local codex tool OR plugin-owned tool
-> Embedded McpServer._executeMcpHandler()
-> consolidated handlers / task / bootstrap / rescan / dimension_complete
-> ServiceContainer + @alembic/core contracts
```

关键证据：

- `../AlembicPlugin/bin/codex-mcp.ts`：shim 只设置 Codex runtime env、shutdown 和 stdio MCP server。
- `../AlembicPlugin/lib/external/mcp/CodexMcpServer.ts:205`：外层 Codex server class。
- `../AlembicPlugin/lib/external/mcp/CodexMcpServer.ts:317`：tool call 先解析 execution context、knowledge gate、preflight。
- `../AlembicPlugin/lib/external/mcp/CodexMcpServer.ts:952`：普通 Alembic tools 进入 plugin-owned embedded MCP handler。
- `../AlembicPlugin/lib/external/mcp/McpServer.ts:142`：内层 Alembic V3 MCP server class。
- `../AlembicPlugin/lib/external/mcp/McpServer.ts:339`：内层统一执行 handler、gateway mapping、session tracking。

## 当前功能区事实

| 功能区 | 当前代码位置 | 事实判断 |
| --- | --- | --- |
| Codex runtime env / channel / plugin identity | `lib/codex/runtime/RuntimeContext.ts` | 清晰，属于 Plugin owned。 |
| Codex local tools and visibility policy | `lib/codex/ToolPolicy.ts` | 清晰，但与 `tools.ts` 分散维护。 |
| MCP core tool catalog | `lib/external/mcp/tools.ts` | 实际 core tools 为 19 个，文件顶部仍写 16+2=18，存在 catalog 注释漂移。 |
| Request preflight | `lib/codex/preflight/Preflight.ts` | 清晰，负责 projectRoot、knowledge gate、tier / admin gate。 |
| Service request boundary | `lib/codex/ServiceRequestBoundary.ts` | 清晰，声明 Codex-facing MCP tools Plugin-owned，resident service 只是显式 API usage。 |
| Outer Codex execution router | `lib/external/mcp/CodexMcpServer.ts` | 职责过多：transport、projectRoot scope、init-on-demand、dashboard/job route、resident scope、embedded MCP server、opportunistic evolution surface。 |
| Inner Alembic MCP server | `lib/external/mcp/McpServer.ts` | 职责过多：SDK transport、handler registry、gateway resolver、session / intent tracking、drift tracking、response envelope。 |
| Resident service adapter | `lib/service/resident/AlembicResidentServiceClient.ts` | 单体过大，合并了 probe、ProjectScope、search、intent episode、jobs、dashboard 和 fallback / compact meta。 |
| Prime / task lifecycle | `lib/external/mcp/handlers/task.ts` | 功能已成型，但 prime、intent intake、search material、visible receipt instruction、IntentEpisode handoff、task lifecycle 混在一个 handler。 |
| Search intent handoff | `lib/service/task/HostIntentFrame.ts`、`PrimeSearchPipeline.ts`、`handlers/search.ts` | 是较好的可复用边界；应保留 Recipe source refs / evidence refs。 |
| Project Skill delivery | `lib/service/skills/ProjectSkillService.ts`、`lib/codex/ProjectSkillDelivery.ts` | 是当前最清晰的 interface style，可作为重构目标样式。 |
| Embedded daemon / HTTP compatibility | `bin/daemon-server.ts`、`lib/http/**`、`lib/injection/**` | 仍保留主体仓库形态，当前作为 embedded plugin runtime / resident compatibility 存在；不应混淆为 Plugin 长期拥有 Alembic daemon。 |
| File-change / git diff evolution | `lib/service/evolution/**`、`lib/codex/evolution/PluginOpportunisticEvolution.ts` | 038/039 后已有分界：long-lived file monitor 属于 Alembic；Plugin 只保留 embedded checkpoint compatibility 和 one-shot git diff opportunistic surface。 |

## 主要结构问题

### 1. Tool surface 分散且注释漂移

`CODEX_LOCAL_TOOLS` 在 `ToolPolicy.ts` 有 9 个 Codex local tools；`TOOLS` 在 `tools.ts` 有 19 个 core tools。外层 list tools 会合并两类可见工具，但 catalog、annotation、gateway map、description、tier policy 分散在两个模块。

证据：

- `../AlembicPlugin/lib/codex/ToolPolicy.ts:117`
- `../AlembicPlugin/lib/external/mcp/tools.ts:2`
- `../AlembicPlugin/lib/external/mcp/tools.ts:144`
- `../AlembicPlugin/lib/external/mcp/tools.ts:193`
- `../AlembicPlugin/lib/external/mcp/tools.ts:254`

风险：

- 新增 / 删除工具时容易只改一边。
- 顶部工具数量注释已经与真实数量不一致。
- policy、preflight、gateway、annotation 的“一个工具是什么”没有单一事实源。

### 2. 外层 CodexMcpServer 是多职责路由器

`CodexMcpServer` 同时处理：

- SDK list / call handler。
- projectRoot argument scoped server。
- knowledge gate 和 auto-init。
- local codex status / diagnostics / init / dashboard / job / cleanup。
- daemon enhancement route。
- resident ProjectScope identity。
- embedded plugin-owned MCP server lifecycle。
- opportunistic evolution surface injection。

证据：

- `../AlembicPlugin/lib/external/mcp/CodexMcpServer.ts:250`
- `../AlembicPlugin/lib/external/mcp/CodexMcpServer.ts:317`
- `../AlembicPlugin/lib/external/mcp/CodexMcpServer.ts:841`
- `../AlembicPlugin/lib/external/mcp/CodexMcpServer.ts:1006`
- `../AlembicPlugin/lib/external/mcp/CodexMcpServer.ts:1034`
- `../AlembicPlugin/lib/external/mcp/CodexMcpServer.ts:1120`

风险：

- 后续功能容易直接塞进 server class。
- service route 与 execution route 交织，容易把 Alembic resident 能力误认为 Plugin-owned behavior。
- env / cwd scope restore 逻辑与 MCP execution 逻辑耦合。

### 3. 内层 McpServer 仍承载主体式 V3 tool server

内层 `McpServer` 既是 SDK server，也是 handler registry、gateway resolver、session tracker、intent drift tracker。

证据：

- `../AlembicPlugin/lib/external/mcp/McpServer.ts:188`
- `../AlembicPlugin/lib/external/mcp/McpServer.ts:339`
- `../AlembicPlugin/lib/external/mcp/McpServer.ts:391`
- `../AlembicPlugin/lib/external/mcp/McpServer.ts:572`
- `../AlembicPlugin/lib/external/mcp/McpServer.ts:634`

风险：

- Plugin 的 Codex-facing handler 和 Alembic V3 generic handler 混合在一起。
- intent session tracking 作为 MCP server 内部状态存在，不利于与 resident IntentEpisode / prime injection package 对齐。
- gateway mapping 与 handler registry 没有形成独立可测试 contract。

### 4. ServiceContainer / HTTP / daemon 需要重新命名边界

`ServiceContainer` 仍初始化完整模块：infra、signal、app、knowledge、vector、guard、skill hooks、panorama，还包含 ProjectGraph build cache。`daemon-server.ts` 启动 HTTP server 并注册 GitDiffCheckpoint。`HttpServer` 仍挂载 health、daemon、jobs、auth、monitoring、guard、search、extract、commands、modules、knowledge、candidates、panorama、evolution、skills 等 route。

证据：

- `../AlembicPlugin/lib/injection/ServiceContainer.ts:114`
- `../AlembicPlugin/lib/injection/ServiceContainer.ts:121`
- `../AlembicPlugin/lib/injection/ServiceContainer.ts:146`
- `../AlembicPlugin/lib/injection/ServiceContainer.ts:242`
- `../AlembicPlugin/lib/injection/ServiceContainer.ts:300`
- `../AlembicPlugin/bin/daemon-server.ts:20`
- `../AlembicPlugin/bin/daemon-server.ts:95`
- `../AlembicPlugin/bin/daemon-server.ts:160`
- `../AlembicPlugin/lib/http/HttpServer.ts:244`

风险：

- 从目录上看，Plugin 像拥有完整 Alembic runtime。
- 后续执行窗口容易在 Plugin 内修 Alembic 主体职责。
- HTTP route 可以保留为 embedded compatibility，但需要有明确的 compatibility boundary 和可删减清单。

### 5. Resident service client 是单体接口

`AlembicResidentServiceClient` 包含 project scope identity、search、intent episode、job、dashboard、probe、fallback meta compact 等所有 resident API。

证据：

- `../AlembicPlugin/lib/service/resident/AlembicResidentServiceClient.ts:301`
- `../AlembicPlugin/lib/service/resident/AlembicResidentServiceClient.ts:331`
- `../AlembicPlugin/lib/service/resident/AlembicResidentServiceClient.ts:450`
- `../AlembicPlugin/lib/service/resident/AlembicResidentServiceClient.ts:517`
- `../AlembicPlugin/lib/service/resident/AlembicResidentServiceClient.ts:556`
- `../AlembicPlugin/lib/service/resident/AlembicResidentServiceClient.ts:740`
- `../AlembicPlugin/lib/service/resident/AlembicResidentServiceClient.ts:1631`

风险：

- 任何 resident API 增加都扩大单文件。
- search / intent / projectScope / jobs 的失败语义混在一起，难以做 interface-level tests。
- 对 Plugin 的“需要 Alembic resident 还是 Plugin fallback”判断不够集中。

### 6. Prime handler 混合业务材料和宿主可见回复协议

`task.ts` 的 `_prime` 已经做了 host intent intake、deterministic intent extraction、PrimeSearchPipeline、IntentEpisode handoff、prime knowledge material、visible shout instruction。它保留了 sourceRefs / evidenceRefs，这是正确的；但 presenter / instruction 层和 lifecycle handler 混在同一文件。

证据：

- `../AlembicPlugin/lib/external/mcp/handlers/task.ts:253`
- `../AlembicPlugin/lib/external/mcp/handlers/task.ts:338`
- `../AlembicPlugin/lib/external/mcp/handlers/task.ts:413`
- `../AlembicPlugin/lib/external/mcp/handlers/task.ts:531`
- `../AlembicPlugin/lib/external/mcp/handlers/task.ts:801`
- `../AlembicPlugin/lib/service/task/HostIntentFrame.ts:149`
- `../AlembicPlugin/lib/service/task/HostIntentFrame.ts:192`
- `../AlembicPlugin/lib/service/task/PrimeSearchPipeline.ts:111`

风险：

- 改 prime 文案时可能误动 intent / search / resident handoff。
- 改 intent 路由时可能误动开发者可见 receipt。
- Recipe source file path / evidence refs 不能在重构中丢失。

### 7. Evolution fallback 已有正确能力，但命名仍容易和 038 混淆

`PluginOpportunisticEvolution` 已明确 service gate、git diff evidence、strong / weak / no-op evidence gate；`CodexMcpServer` 只在 task close outcome 后附加 surface。与此同时，`FileChangeHandler` / `FileChangeDispatcher` / `GitDiffCheckpointService` 仍在 service tree 下，由 embedded daemon server 注册 checkpoint。

证据：

- `../AlembicPlugin/lib/codex/evolution/PluginOpportunisticEvolution.ts:8`
- `../AlembicPlugin/lib/codex/evolution/PluginOpportunisticEvolution.ts:21`
- `../AlembicPlugin/lib/codex/evolution/PluginOpportunisticEvolution.ts:74`
- `../AlembicPlugin/lib/codex/evolution/PluginOpportunisticEvolution.ts:129`
- `../AlembicPlugin/lib/codex/evolution/PluginOpportunisticEvolution.ts:155`
- `../AlembicPlugin/lib/injection/modules/KnowledgeModule.ts:344`
- `../AlembicPlugin/lib/injection/modules/KnowledgeModule.ts:355`

风险：

- 后续开发容易把 Plugin-only git diff fallback 误写成 file monitor。
- embedded runtime checkpoint 和 Codex one-shot opportunistic surface 需要目录 / 类型名表达清楚。

## 正向样板

### Project Skill 是较好的目标结构

`ProjectSkillService` 清楚说明 source storage、runtime export 和 replacement boundary；`ProjectSkillDelivery` 负责 receipt、authorization、managed marker 和 symlink export。它可以作为本次重构的方法论样板：功能服务与交付 receipt 分开，runtime 投影有明确授权和 marker。

证据：

- `../AlembicPlugin/lib/service/skills/ProjectSkillService.ts:85`
- `../AlembicPlugin/lib/service/skills/ProjectSkillService.ts:89`
- `../AlembicPlugin/lib/service/skills/ProjectSkillService.ts:165`
- `../AlembicPlugin/lib/service/skills/ProjectSkillService.ts:288`
- `../AlembicPlugin/lib/codex/ProjectSkillDelivery.ts:16`
- `../AlembicPlugin/lib/codex/ProjectSkillDelivery.ts:68`
- `../AlembicPlugin/lib/codex/ProjectSkillDelivery.ts:141`

### Boundary tests 已经存在

已有边界测试可作为重构防护：

- `../AlembicPlugin/test/unit/CodexModuleBoundary.test.ts`
- `../AlembicPlugin/test/unit/CodexToolPolicy.test.ts`
- `../AlembicPlugin/test/unit/CodexServiceRequestBoundary.test.ts`
- `../AlembicPlugin/test/unit/PluginHttpSurfaceBoundary.test.ts`
- `../AlembicPlugin/test/unit/ProjectSkillService.test.ts`
- `../AlembicPlugin/test/unit/ProjectSkillDelivery.test.ts`
- `../AlembicPlugin/test/unit/PluginOpportunisticEvolution.test.ts`
- `../AlembicPlugin/test/unit/PrimeSearchPipelineResidentSearch.test.ts`

## 重构风险边界（已按用户最新裁决修正）

本节原先按 no-behavior / compatibility-first 风险口径书写；2026-05-31 用户已裁决“不做兼容，直接使用新接口与逻辑”。修正后的边界如下：

- 不删除 cold-start / rescan / submit / dimension complete 已通过验收的能力；但可以重排内部接口，旧入口不长期保留。
- 不把 Plugin 重新扩展为 Agent runtime、AI provider runtime 或 Tool V2 runtime。
- 不把 long-lived file monitor 放回 Plugin。
- Plugin Codex-facing `alembic_skill` legacy alias 已成为删除候选；删除前必须迁移 Plugin skills、tests、schema、README 和 visible tool policy。Alembic resident `alembic_skill` 不是同一个问题。
- HTTP / daemon compatibility route 不应凭目录观感直接删除；用户已确认纳入本需求后半段，先拆职责并确认 runtime package / smoke / `alembic_codex_*` recovery path 仍需要什么，再同阶段迁移、保留 Plugin 必需能力并清理删除多余主体式能力。
- 不丢失 Recipe source refs / source file path / evidence refs。

最新补充调研见：[plugin-architecture-interface-refactor-code-logic-research-2026-05-31.md](plugin-architecture-interface-refactor-code-logic-research-2026-05-31.md)。

## 建议总控下一步

1. 将本需求作为 `ready-for-workspace` 的 requirement handoff 接收，先不打断当前 `PLUGIN-BUNDLE-IDE039-P1`。
2. 先做 Stage 0 interface contract dossier：功能区 / public surface / allowed imports / owner map / runtime package consumers / cross-repo consumers。
3. 后续阶段按新接口推进，不保留长期兼容双轨；删除旧入口必须同阶段迁移真实消费者和验证脚本。
4. 每个阶段都必须有边界测试或 focused unit，证明工具可见性、request boundary、resident service、ProjectSkill、opportunistic evolution 和 prime sourceRefs 未回退。
