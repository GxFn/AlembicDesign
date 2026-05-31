# Plugin Architecture Interface Refactor 需求设计

Design Key：PLUGIN-ARCHITECTURE-INTERFACE-REFACTOR-2026-05-31
日期：2026-05-31
状态：ready-for-workspace
维护窗口：AlembicDesign
总控：AlembicWorkspace
目标仓库：AlembicPlugin
原始计划：[plugin-architecture-interface-refactor-original-plan-2026-05-31.md](plugin-architecture-interface-refactor-original-plan-2026-05-31.md)
代码事实复核：[plugin-architecture-interface-refactor-code-fact-review-2026-05-31.md](plugin-architecture-interface-refactor-code-fact-review-2026-05-31.md)
代码逻辑与真实使用调研：[plugin-architecture-interface-refactor-code-logic-research-2026-05-31.md](plugin-architecture-interface-refactor-code-logic-research-2026-05-31.md)

## 原始计划书

- 原始计划书：[plugin-architecture-interface-refactor-original-plan-2026-05-31.md](plugin-architecture-interface-refactor-original-plan-2026-05-31.md)
- 原始计划书确认状态：用户已补充确认。
- 用户确认时间：2026-05-31。

## 已确认目标

用户目标：

```text
深度检查 Plugin 的代码结构和接口关系，新需求为 Plugin 基于功能和接口的架构重构与优化。
```

用户补充裁决：

```text
你直接进行代码逻辑调研，真实的代码实现和功能使用，我建议是不要做兼容，直接使用新接口与逻辑，确认好其他仓库对接就可以
```

用户追加裁决：

```text
按照你的建议，拆分职责，只保留 Plugin 需要的内容，然后清理删除
```

Design 目标：

按功能和接口边界重构 AlembicPlugin，使 Codex-facing MCP surface、resident adapter、embedded runtime compatibility、host-agent workflow、Project Skill delivery、prime / intent / search、opportunistic evolution fallback 都有清晰的 owner、contract、输入输出、测试和禁止边界。旧接口不长期保留；所有旧入口删除必须同阶段迁移真实消费者、测试、文档和 runtime packaging。

`daemon-server.js` / embedded HTTP runtime 纳入本需求后半段处理：先拆分职责，确认 Plugin 仍需要的 runtime、recovery、health/status、smoke 内容，再清理删除不属于 Plugin 的主体式能力。

## 当前主线关系

下一主线 / TODO 候选，可交给总控接收评审。

当前总控发送给 AlembicPlugin 的 `PLUGIN-BUNDLE-IDE039-P1` 仍需由总控判断状态。本需求不由 Design 直接打断正在进行的 Plugin implementation；总控接收后可按当前 Plugin 主线回填情况决定插入时机。

## 用户场景

### 场景 1：实现窗口修改 Plugin MCP tool

开发者要新增、删除或修改 Codex-facing tool。理想状态下，它能从一个 tool surface contract 看清：

- tool 是否 Codex local tool 还是 core MCP tool。
- tier / admin / visibility / knowledge gate。
- gateway mapping。
- handler owner。
- 是否允许 resident service。
- 输出 envelope 和 sourceRefs 证据要求。
- 旧 tool 若删除，哪些 skills / README / verify / smoke / tests 必须同阶段迁移。

### 场景 2：实现窗口判断一段逻辑该放哪里

开发者遇到 ProjectScope、resident search、git diff fallback、Project Skill export 或 prime injection 需求。理想状态下，它能直接判断：

- Plugin owned：Codex host adapter、MCP schema、runtime export、visible receipt、fallback surface。
- Core owned：共享模型、resident service contract、RecipeProductionGateway、SourceRef、ProjectScope contract。
- Alembic owned：long-lived daemon、Dashboard、file monitor、internal AI jobs。
- Embedded runtime compatibility：只为 Plugin runtime 包提供本地可用性，不代表 Plugin 长期拥有主体能力。

### 场景 3：总控验收 Plugin 结构优化

总控不需要读完整大文件，就能通过 contract dossier、boundary tests 和 focused smoke 判断工具可见性、resident service、sourceRefs、ProjectSkill、opportunistic evolution 和 runtime package 没有回退。

## 功能闭环

- 输入：AlembicPlugin 现有代码、AGENTS 边界、当前总控状态、已验收功能链路、代码事实复核、代码逻辑与真实使用调研。
- 输出：重构后的功能区结构、接口 contract、迁移清单、删除候选清单、边界测试、runtime package / smoke 验证。
- 状态变化：
  - 代码目录 / 模块职责更清晰。
  - Tool surface、execution router、resident adapter、prime presenter、embedded compatibility 有明确 contract。
  - 旧入口不作为长期兼容层保留。
  - 测试按接口边界重组。
- 生产方：
  - AlembicPlugin MCP runtime。
  - Codex host adapter。
  - Embedded runtime compatibility server。
  - ProjectSkill delivery service。
- 消费方：
  - Codex host / MCP client。
  - AlembicWorkspace 总控验收。
  - Alembic resident daemon API。
  - Project skill runtime `.agents/skills`。
  - Recipe / sourceRefs / IntentEpisode / search / evolution consumers。
- 失败路径：
  - tool visibility 漂移。
  - 旧入口删除但真实消费者未迁移。
  - runtime package / smoke / Codex wrapper 契约被误删。
  - resident unavailable / ProjectScope mismatch。
  - sourceRefs / evidenceRefs 丢失。
  - public envelope 或 telemetry 被改坏。

## 仓库边界

| 仓库 / 窗口 | 职责 | 包含范围 | 不包含范围 |
| --- | --- | --- | --- |
| Alembic | 观察 / contract producer | resident daemon API、runtime capability、Dashboard / job / file monitor source of truth | 不迁移 Alembic 主体实现；Plugin 不做 Alembic daemon fallback 兼容。 |
| AlembicCore | 观察 / shared contract | RecipeProductionGateway、SourceRef、ProjectScope、resident service contracts、host-agent workflows public contract | 不在 Core 放 Codex / MCP / plugin runtime implementation。 |
| AlembicAgent | 无任务 | 无 | 不重建 Agent runtime / AI provider。 |
| AlembicDashboard | 观察 | 直接消费 Alembic daemon health、runtimeBoundary 和 ProjectScope endpoints | 不消费 Plugin API；无 UI 任务。 |
| AlembicPlugin | 目标仓库 | MCP surface、Codex adapter、resident adapter、embedded runtime compatibility、ProjectSkill、prime / search / evolution fallback、tests | 不拥有 long-lived file monitor、Dashboard frontend、internal AI runtime。 |
| AlembicTest | 观察 | 后续需要真实 Codex Agent 验收时由总控裁决 | 第一版不直接启动真实项目测试。 |

## 代码事实与调研结论

### AlembicCore

- `@alembic/core` 提供 daemon、workspace、search、knowledge、project-intelligence、host-agent-workflows、resident service、ProjectScope、SourceRef、RecipeProductionGateway 等共享能力。
- Core 边界测试禁止 Codex / MCP / plugin runtime implementation 进入 Core。
- 本需求暂未证明需要 Core public contract 变更；若 Stage 0 inventory 发现 Plugin 只能靠内部结构访问共享能力，需单独交总控裁决。

### Alembic

- Alembic resident daemon / Dashboard / jobs / native file monitor 是主体 source of truth。
- `/api/v1/daemon/health` 已返回 canonical `residentService` 与 `runtimeBoundary`。
- Plugin resident client 当前消费 `/api/v1/search`、`/api/v1/jobs`、`/api/v1/project-scope/resolve-folder`、`/api/v1/intent-episodes`。
- Stage 0 需要复核 Plugin 删除 `runtimeBoundary` fallback 后，status/dashboard/job/search/ProjectScope 是否均由 `residentService` 覆盖。

### AlembicDashboard

- Dashboard 直接消费 Alembic daemon health、runtimeBoundary、ProjectScope endpoints。
- Plugin 重构不应给 Dashboard 派任务；Dashboard 只在 Alembic daemon contract 变化时观察。

### AlembicPlugin

已存在能力：

- Codex MCP shim 和 outer router。
- ToolPolicy / Preflight / ServiceRequestBoundary / ModuleBoundary。
- Embedded V3 MCP handler tree。
- Resident service client。
- ProjectSkill service / delivery receipt。
- Prime intent / search / resident handoff。
- Plugin opportunistic evolution surface。
- Boundary tests 和 acceptance pack。

缺口：

- Tool catalog 分散。
- `CodexMcpServer` / `McpServer` / `AlembicResidentServiceClient` 大文件职责过多。
- Plugin Codex-facing `alembic_skill` 是 legacy compatibility alias，已可迁移到 `alembic_project_skill`。
- `runtimeBoundary` fallback 是 compatibility path，用户裁决后应成为删除候选。
- HTTP / daemon / ServiceContainer compatibility 边界命名不够清楚。
- Prime visible receipt presenter 与 lifecycle handler 混合。

## 代码实现依赖调研

已补充真实代码逻辑调研：[plugin-architecture-interface-refactor-code-logic-research-2026-05-31.md](plugin-architecture-interface-refactor-code-logic-research-2026-05-31.md)。

关键生命周期：

- Codex plugin install / wrapper / runtime package start。
- MCP list / call。
- projectRoot / ProjectScope resolution。
- knowledge gate / auto init。
- local codex tools。
- plugin-owned tool execution。
- resident service request。
- prime search / intent episode。
- ProjectSkill export。
- opportunistic evolution surface。
- embedded daemon / HTTP compatibility.

真实跨仓库消费者：

- Codex host：`.mcp.json`、wrapper、runtime package binary。
- AlembicCore：resident / ProjectScope / host-agent workflow contracts。
- Alembic：daemon health、search、jobs、ProjectScope、IntentEpisode HTTP API。
- AlembicDashboard：Alembic daemon runtimeBoundary / ProjectScope，不消费 Plugin API。

不能切换 / 不能删除 / 不能提前消费的边界：

- 不能在未迁移 Plugin shell / skills / README / verify / smoke 的情况下删除 Codex-visible old tool。
- 不能删除 embedded daemon routes before runtime package dependency inventory。
- 不能改变 Recipe sourceRefs evidence surface。
- 不能把 resident unavailable 当成 product failure without route metadata。

## 设计选项

### 选项 A：先做功能区 contract dossier，再逐阶段迁移到新接口

- 描述：先输出功能区 / owner / API / allowed imports / tests / runtime consumers map；随后按 tool surface、execution router、resident client、prime presenter、embedded runtime compatibility 分阶段迁移到新接口，并删除旧入口。
- 优点：符合用户“不做兼容”的裁决，同时避免误删真实消费者；符合总控先证据后动作。
- 风险：需要每阶段同步 docs/tests/smoke，不能只移动代码。

### 选项 B：直接按目录大迁移

- 描述：一次性把 `lib/external/mcp`、`lib/codex`、`lib/service`、`lib/http` 重排成新目录。
- 优点：视觉上变化快。
- 风险：高概率破坏 import、packaged runtime、tests；容易变成大规模无意义 churn。

### 选项 C：只补文档和边界测试，不动结构

- 描述：保留代码结构，只新增架构说明和 regression tests。
- 优点：最安全。
- 风险：无法解决大文件继续堆叠和接口漂移；后续实现窗口仍会迷路。

## 推荐方案

推荐选项 A。

具体策略：

1. 先建立 `PluginInterfaceContractDossier`，把功能区、接口 owner、真实消费者和可删除旧入口写清。
2. 以 `ProjectSkillService` / `ProjectSkillDelivery` 的风格作为样板：service、delivery receipt、runtime export、authorization / marker / conflict status 分开。
3. 先 consolidation tool surface，再拆 outer router，因为 tool surface 是所有 route 的入口。
4. 每阶段只处理一个 interface area；不保留长期 compatibility facade，旧入口要随真实消费者迁移后删除。
5. 每阶段必须有 focused tests 证明能力不回退，且旧消费者已迁移。

## 推荐目标结构

| Interface Area | Owner | 目标职责 | 典型文件 |
| --- | --- | --- | --- |
| `codex-runtime` | AlembicPlugin | runtime env、channel、projectRoot trust、package identity | `lib/codex/runtime/*`, `ProjectRootResolver` |
| `plugin-runtime-contract` | AlembicPlugin | plugin shell、wrapper、runtime package、marketplace、channel、skills、verify/smoke | `plugins`, `channels`, scripts |
| `mcp-tool-surface` | AlembicPlugin | tool catalog、schema、annotations、gateway map、visibility policy、admin gate | `ToolPolicy`, `tools`, `Preflight` |
| `codex-execution-router` | AlembicPlugin | list/call transport、local codex tools、resident route decision、embedded handler executor | `CodexMcpServer` split |
| `embedded-mcp-handlers` | AlembicPlugin + Core contracts | existing Alembic tool handlers for Codex-facing execution | `McpServer`, `handlers/*` |
| `resident-adapters` | AlembicPlugin | daemon probe、ProjectScope、search、intent episode、jobs、dashboard clients | `AlembicResidentServiceClient` split |
| `host-agent-workflows` | AlembicPlugin + Core contracts | bootstrap / rescan / submit / dimension_complete Codex host-agent route | `handlers/bootstrap*`, `rescan*`, `consolidated` |
| `prime-intent-package` | AlembicPlugin | host intent frame、search/vector/relation injection package、sourceRefs / receipt presenter | `HostIntentFrame`, `PrimeSearchPipeline`, task presenter |
| `project-skill-delivery` | AlembicPlugin | project skill source storage、receipt、authorization、runtime export | `ProjectSkillService`, `ProjectSkillDelivery` |
| `opportunistic-evolution-surface` | AlembicPlugin | one-shot git diff fallback surface when resident service cannot handle project scope | `PluginOpportunisticEvolution` |
| `embedded-runtime-compat` | AlembicPlugin compatibility | daemon-server, HTTP API, ServiceContainer, git diff checkpoint needed by packaged runtime | `bin/daemon-server`, `lib/http`, `lib/injection` |

接口命名原则：

- `Surface`：宿主可见输出或 MCP 暴露面。
- `Router`：只选择执行路径，不做业务处理。
- `Adapter` / `Client`：调用外部或 resident service。
- `Presenter`：把内部材料转换成 Codex-visible message / receipt。
- `Compatibility`：保留的 embedded runtime 兼容能力，不代表长期 owner。
- `Contract`：跨模块稳定输入输出。

## 推荐删除候选

| 候选 | 推荐处理 | 边界 |
| --- | --- | --- |
| Plugin Codex-facing `alembic_skill` alias | 迁移到 `alembic_project_skill` 后删除 | 不等于删除 Alembic resident `alembic_skill`。 |
| Plugin `runtimeBoundary` fallback | 当前 Alembic health `residentService` 覆盖后删除 fallback | Dashboard 直接消费 Alembic daemon，不通过 Plugin fallback。 |
| `ALEMBIC_CHANNEL` env fallback | 迁移到 `ALEMBIC_CHANNEL_ID` 后删除 | 需要同步 `.mcp.json` / docs / tests。 |
| 分散 tool catalog 注释和重复常量 | 迁移到单源 `PluginToolSurfaceCatalog` 后删除 | 需要覆盖 policy / gateway / handler / annotations。 |

## TODO / Backlog

| ID | 状态 | 类型 | 严重度 / 优先级 | 归属 | 事项 / TODO | 影响目标 / 派发 | 依赖 / 触发 | 推荐窗口 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| PAIR-TODO-1 | 观察 | 调研 | P0 | AlembicPlugin | Stage 0 生成 Plugin interface contract dossier / import graph / public surface map / runtime package consumer map。 | 是 | 总控接收后 | AlembicPlugin |
| PAIR-TODO-2 | 观察 | 重构 | P0 | AlembicPlugin | Stage 1 建立单源 PluginToolSurfaceCatalog，并删除 Plugin Codex-facing `alembic_skill` alias，迁移到 `alembic_project_skill`。 | 是 | PAIR-TODO-1 | AlembicPlugin |
| PAIR-TODO-3 | 观察 | 重构 | P1 | AlembicPlugin | Stage 2 拆分 CodexMcpServer execution router：projectRoot scope、preflight、local tool dispatcher、embedded executor、resident route、evolution surface。 | 是 | PAIR-TODO-2 | AlembicPlugin |
| PAIR-TODO-4 | 观察 | 重构 | P1 | AlembicPlugin | Stage 3 拆分 resident adapter capability clients，并删除 runtimeBoundary fallback，统一使用 Core residentService contract。 | 是 | PAIR-TODO-1 | AlembicPlugin / Alembic observe |
| PAIR-TODO-5 | 观察 | 重构 | P1 | AlembicPlugin | Stage 4 拆分 prime / intent / search 交付包，保留 sourceRefs / evidenceRefs / vector / relation / IntentEpisode evidence。 | 是 | PAIR-TODO-3 / PAIR-TODO-4 | AlembicPlugin |
| PAIR-TODO-6 | 观察 | 收敛 | P2 | AlembicPlugin | Stage 5 基于 runtime package consumer map 拆分 `daemon-server.js` / embedded HTTP runtime 职责，只保留 Plugin 需要的 runtime / recovery / smoke 内容，并清理删除主体式多余能力。 | 是 | PAIR-TODO-1 / PAIR-TODO-4 | AlembicPlugin |
| PAIR-TODO-7 | 观察 | 验证 | P1 | AlembicPlugin | 重组 focused tests：tool surface、execution boundary、resident service、ProjectSkill、prime material、opportunistic evolution、runtime package smoke。 | 是 | 每阶段同步 | AlembicPlugin |

## 阶段候选

1. Stage 0：Interface Contract Dossier
   - 只读输出功能区、owner、public surface、imports、tests、embedded runtime route map、runtime package consumer map。
   - 总控确认后再进入代码重构。

2. Stage 1：Tool Surface New Interface
   - 统一 Codex local tools + core MCP tools + annotations + gateway + policy。
   - 删除 Plugin `alembic_skill` alias，迁移到 `alembic_project_skill`。
   - 更新 skills / README / schema / tests / verify。

3. Stage 2：Codex Execution Router Decomposition
   - 拆 `CodexMcpServer` 多职责。
   - 保持 Codex host 启动契约。
   - 增加 projectRoot、resident scope、auto-init、local codex tool、embedded executor tests。

4. Stage 3：Resident Adapter Clients
   - 拆 ProjectScope / Search / IntentEpisode / Jobs / Dashboard / Probe clients。
   - 删除 runtimeBoundary fallback，以 residentService 为 canonical。

5. Stage 4：Prime Package and Presenter Cleanup
   - 把 PrimeKnowledgeMaterial / visible receipt presenter 独立。
   - 保留 Recipe sourceRefs / evidenceRefs / primeInjectionPackage。

6. Stage 5：Embedded Runtime Compatibility Pruning
   - 基于 runtime consumer map 拆分 `daemon-server.js` / HTTP / ServiceContainer compatibility 的职责。
   - 只保留 Plugin 需要的 runtime start、recoverable job/status、health、packaged smoke 支撑能力。
   - 清理删除 Alembic 主体式 daemon / HTTP 能力外观中未被 Plugin 消费的部分。

7. Stage 6：Validation and Packaged Runtime Smoke
   - build / lint / diff check。
   - focused tests。
   - verify codex plugin / session smoke。
   - runtime package verify。

阶段候选不等于执行派发。最终阶段顺序必须由 `AlembicWorkspace` 基于当前 Plugin 主线回填和代码实现依赖调研确认。

## 验证策略

- 最小验证：
  - tool surface sync test。
  - `CodexToolPolicy` / `CodexServiceRequestBoundary` / `CodexModuleBoundary` focused tests。
  - router split tests：projectRoot scoped call、auto-init gate、resident ProjectScope availability。
  - resident adapter tests：search unavailable, ProjectScope ready/unready, jobs/dashboard unavailable。
  - prime material tests：sourceRefs / evidenceRefs retained。
  - opportunistic evolution tests：resident available no-op, unavailable git diff weak / strong.
- 集成验证：
  - `npm run build:check`
  - `npm run lint`
  - `git diff --check`
  - `npm run verify:codex-plugin`
  - `npm run smoke:codex-plugin` when runtime package changes.
- 测试窗口交接：
  - 第一版不需要 AlembicTest。
  - 若 Stage 5/6 触及真实 Codex thread 或 packaged runtime behavior，再由总控决定是否交 AlembicTest。

## 非目标与禁止捷径

- 不做 UI。
- 不迁移 Alembic daemon / Dashboard / file monitor 主体职责。
- 不引入新 AI provider。
- 不改 `@alembic/core` public contract，除非 Stage 0 发现硬缺口并经总控确认。
- 不为了“不要兼容”而未迁移消费者就删除 runtime / HTTP / tool 入口。
- 不把本需求和正在执行的 Plugin 合并任务包混在同一个实现提交里。

## 用户确认记录

| 时间 | 决策 / 偏好 | 影响 |
| --- | --- | --- |
| 2026-05-31 | 用户提出新需求：Plugin 基于功能和接口的架构重构与优化，并要求深度检查代码结构和接口关系。 | Design 进行代码事实复核并形成需求设计。 |
| 2026-05-31 | 用户补充确认：不要做兼容，直接使用新接口与逻辑；先查真实代码实现和功能使用，确认其它仓库对接。 | 需求从 compatibility-first 草案改为 new-interface-first；旧入口删除需同阶段迁移真实消费者。 |
| 2026-05-31 | 用户确认：`daemon-server.js` / embedded HTTP runtime 按 Design 推荐纳入本需求后半段，拆分职责，只保留 Plugin 需要内容，再清理删除。 | Stage 5 不再另起独立 runtime simplification 需求；纳入本需求的 embedded runtime compatibility pruning。 |

## Workspace 交接准备状态

已准备交给 AlembicWorkspace 接收评审。

建议总控接收后先做 Stage 0 interface contract dossier，不直接派大重构；随后每个阶段先有 area plan，再由归属仓库执行。

## 仍需总控裁决

- 当前 Plugin 主线回填后，是否先执行本需求 Stage 0，还是先做已有 038/039/IDE Agent 主线收敛。
- Alembic resident `alembic_skill` 是否未来也要重命名；这超出本 Plugin-only 需求，应另起 Alembic 主体接口需求。

## 交接前自检

- 已对照 `docs/workspace-alignment-checklist.md`：是。
- 原始计划已确认：是，用户已补充确认。
- 完整功能闭环已写清：是。
- 代码事实 / 调研缺口已分开：是。
- 阶段仍是候选，没有写成 wave：是。
- TODO / Backlog 已记录：是。
- 仍需总控裁决的问题已列出：是。
