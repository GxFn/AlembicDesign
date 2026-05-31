# Knowledge Evolution 038 / 039 Code Fact Review

Design Key：KNOWLEDGE-EVOLUTION-CODE-FACT-REVIEW-2026-05-31
日期：2026-05-31
状态：design-code-fact-review / read-only / ready-for-workspace-input
维护窗口：AlembicDesign
关联需求：

- `FILE-MONITOR-EVOLUTION-2026-05-31`
- `PLUGIN-OPPORTUNISTIC-EVOLUTION-2026-05-31`

## 判断类型

- 类型：`research + code-fact-review + requirement-refinement`
- 证据状态：只读代码检查；未修改产品源码，未运行产品 build / test / runtime smoke。
- Design 边界：本文只作为总控 Stage 0 的种子证据，不替代总控验收，也不直接派发实现窗口。
- 用户确认：038 有 watcher 时走 native filesystem watcher；038 没有 watch / watcher 不可用时至少用 git diff 降级。039 Plugin 不做文件监控；Alembic 主体服务可用时，Plugin 不处理监控、git diff 降级或进化判断，交给 Alembic 主体服务；只有主体服务不可用时，039 才默认使用 git diff 能力获得当前任务文件变化证据。
- 用户确认：039 自己根据 git diff 判断知识进化的逻辑，如果需要 Codex Agent 能力测试，必须使用真实 Codex Agent。mock / simulator 不能作为涉及 Agent 推理、tool 调用或窗口线程的验收证据。

## 检查范围

只读检查了以下源码路径：

- `Alembic/lib/service/evolution/DaemonFileChangeCollector.ts`
- `Alembic/lib/service/evolution/FileChangeHandler.ts`
- `Alembic/lib/service/FileChangeDispatcher.ts`
- `Alembic/lib/http/routes/file-changes.ts`
- `Alembic/lib/http/routes/daemon.ts`
- `Alembic/lib/injection/modules/KnowledgeModule.ts`
- `AlembicCore/src/shared/source-contracts.ts`
- `AlembicCore/src/repository/evolution/ProposalRepository.ts`
- `AlembicCore/src/service/knowledge/SourceRefReconciler.ts`
- `AlembicCore/src/workflows/capabilities/planning/knowledge/EvolutionPrescreen.ts`
- `AlembicPlugin/lib/shared/schemas/mcp-tools.ts`
- `AlembicPlugin/lib/service/task/HostIntentFrame.ts`
- `AlembicPlugin/lib/service/task/PrimeSearchPipeline.ts`
- `AlembicPlugin/lib/external/mcp/handlers/task.ts`
- `AlembicPlugin/lib/external/mcp/handlers/search.ts`
- `AlembicPlugin/lib/service/evolution/git-diff-checkpoint/*`
- `AlembicPlugin/lib/codex/ModuleBoundary.ts`
- `AlembicDashboard/src/types.ts`
- `AlembicDashboard/src/api.ts`
- `AlembicDashboard/src/components/Views/EvolutionPanel.tsx`

## 038 File Monitor Evolution 代码事实

### 已存在的真实链路

- Alembic 已有 daemon-owned git worktree collector：`DaemonFileChangeCollector` 会采样 `git diff --name-status`、`git diff --cached` 和 untracked files，并把事件标成 `git-worktree`（`Alembic/lib/service/evolution/DaemonFileChangeCollector.ts:37`、`:55`、`:88`、`:136`、`:176`）。
- Alembic 已有 HTTP file-change 入口：`POST /api/v1/file-changes` 校验事件后取 `fileChangeDispatcher` 并执行 `dispatcher.dispatch(validEvents)`（`Alembic/lib/http/routes/file-changes.ts:47`、`:59`、`:104`、`:109`）。
- Alembic 的 `KnowledgeModule` 已把 `fileChangeHandler` 注册到 `fileChangeDispatcher`（`Alembic/lib/injection/modules/KnowledgeModule.ts:355`、`:372`、`:375`）。
- `FileChangeHandler` 已能按 `sourceRefs` 处理 rename / delete / modified：删除时会用 `EvolutionGateway.submit({ action: "deprecate", source: "file-change" })`，modified 命中 pattern 时会创建 `update` proposal（`Alembic/lib/service/evolution/FileChangeHandler.ts:249`、`:279`、`:282`、`:337`、`:371`、`:382`、`:396`、`:399`）。
- `SourceRefReconciler` 已负责从 `knowledge_entries.reasoning.sources` 填充 / 维护 `recipe_source_refs`（`AlembicCore/src/service/knowledge/SourceRefReconciler.ts:4`、`:81`、`:333`）。
- Dashboard 已有 proposal consumer：`ProposalRecord` / `ProposalSource` 类型支持 `file-change`，API 支持按 Recipe 读取、执行、观察、拒绝 proposal，`EvolutionPanel` 会按 Recipe 读取 proposal（`AlembicDashboard/src/types.ts:660`、`:671`、`:675`，`AlembicDashboard/src/api.ts:3302`、`:3320`、`:3331`、`:3337`、`:3342`，`AlembicDashboard/src/components/Views/EvolutionPanel.tsx:71`）。

### 当前缺口和风险

- 没找到 `DaemonFileChangeCollector` 在 daemon 生命周期里的实例化 / start 接线。源码搜索只发现类本身和单元测试实例，未发现 `new DaemonFileChangeCollector` 的生产注册（搜索结果：`Alembic/lib/service/evolution/DaemonFileChangeCollector.ts:66`、`:69`；测试：`Alembic/test/unit/DaemonFileChangeCollector.test.ts:80`）。这意味着 038 的第一阻塞点不是 handler，而是 collector lifecycle 或 capability 口径。
- daemon health 目前按 `ALEMBIC_DAEMON_MODE === "1"` 且 `ALEMBIC_DAEMON_FILE_CHANGES !== "0"` 宣告 file monitor 可用，并标成 `daemon-git-worktree`（`Alembic/lib/http/routes/daemon.ts:67`、`:150`、`:152`、`:174`、`:278`、`:279`）。如果 collector 未实际启动，health 可能高估能力。
- 现有 `FileChangeHandler` 的 evidence 主要是 changed path、sourceRefs、diff token score 和 matched tokens；没有直接读取 `IntentEpisode`、`IntentSearchPlan` 或 `PrimeInjectionPackage`（`Alembic/lib/service/evolution/FileChangeHandler.ts:371`、`:396`）。038 设计中的 intent / prime enrichment 还需要新增桥接。
- Core proposal source 目前只有 `host-agent`、`alembic-agent`、`metabolism`、`decay-scan`、`consolidation`、`relevance-audit`、`file-change`、`rescan-evolution`，没有独立的 `file-monitor-evolution` 或 `plugin-opportunistic` source（`AlembicCore/src/shared/source-contracts.ts:10`、`:17`、`:18`、`:19`）。
- Proposal type 目前只有 `update | deprecate`，Dashboard 类型也一样（`AlembicCore/src/repository/evolution/ProposalRepository.ts:44`，`AlembicDashboard/src/types.ts:660`）。038 设计里的 `verify-recipe` / `weak hint` 若要落库，需要映射为 evidence/action 字段或新增类型。

### 038 建议收敛

- 第一阶段先修正 / 验证 native watcher 与 git diff fallback lifecycle：有 watcher 时 `fileMonitor.available=true` 才表示 watcher running；无 watcher / watcher unavailable 时必须明确降级到 git diff。
- 复用现有 `source: "file-change"` 和 `type: "update" | "deprecate"`，不要第一版就新增 proposal source / type，除非总控决定需要 Core contract 变更。
- Intent / prime / search enrichment 应作为 proposal evidence enhancer，不应阻塞 changed file + sourceRefs 的基本强 proposal。
- 038 Stage 0 的完成证据至少需要：watcher lifecycle 事实、无 watcher 时 git diff 降级事实、file-change route 到 handler 的最小 probe、sourceRefs hit / no-hit negative path、Dashboard/API consumer 可读 proposal。

## 039 Plugin Opportunistic Evolution 代码事实

### 已存在的真实链路

- Plugin MCP schema 已支持 `hostDeclaredIntent`、`hostTurnMeta` 和 `sourceRefs`：search 和 task input 均能携带 host intent / turn metadata（`AlembicPlugin/lib/shared/schemas/mcp-tools.ts:38`、`:56`、`:66`、`:111`、`:114`、`:117`、`:427`、`:429`、`:432`）。
- Plugin `alembic_task prime` 已把 `userQuery`、`activeFile`、`hostDeclaredIntent`、`hostTurnMeta` 和 request `_meta` 合并成 `HostIntentFrame`，再交给 `PrimeSearchPipeline`（`AlembicPlugin/lib/external/mcp/handlers/task.ts:264`、`:266`、`:267`、`:268`、`:306`）。
- Plugin 会把 prime 结果中的 sourceRefs 投影为 evidence refs，也会把 intent episode start request 发给 resident service，包含 activeFile、hostIntent、searchMeta、sourceRefs 和 turnId（`AlembicPlugin/lib/external/mcp/handlers/task.ts:478`、`:497`、`:898`、`:943`、`:949`、`:951`、`:952`）。
- `HostIntentFrame` 能从 MCP request `_meta` 读取 thread/session/turn/source/surface/language，也能读取 `activeFile`、`cwd`、`projectRoot`、`workspaceRoot`（`AlembicPlugin/lib/service/task/HostIntentFrame.ts:306`、`:319`、`:328`、`:329`、`:330`、`:331`）。
- `PrimeSearchPipeline` 会把 resident intent handoff 转交 resident semantic search，字段包括 confidence、degraded、hostDeclaredIntent、hostTurnMeta、intentContext、scenario、sessionHistory 和 sourceRefs（`AlembicPlugin/lib/service/task/PrimeSearchPipeline.ts:342`、`:343`、`:349`）。
- `alembic_search` handler 同样能构建 resident intent handoff，并在 response searchMeta 中返回 resident `intentEvidence` / `primeInjectionPackage`（`AlembicPlugin/lib/external/mcp/handlers/search.ts:97`、`:98`、`:109`、`:148`、`:155`、`:272`、`:279`）。

### 当前缺口和风险

- 没找到 039 所需的 `opportunistic evolution detector`、`evidence gate`、`strong proposal / weak hint / no-op` 明确实现。现有 `evolution-prescreen` 只是复用 Core rescan prescreen，不是 Plugin 当前 turn 机会式检测（`AlembicPlugin/lib/external/mcp/handlers/evolution-prescreen.ts:7`，`AlembicCore/src/workflows/capabilities/planning/knowledge/EvolutionPrescreen.ts:38`）。
- Plugin 内部有 `GitDiffCheckpointService`，但它的 status 明确是 `mode: "git-diff-checkpoint"`、`surface: "codex-plugin"`、等待 explicit checkpoint；搜索未发现生产 singleton 注册，只在 health/status 读取和测试里出现（`AlembicPlugin/lib/service/evolution/git-diff-checkpoint/GitDiffCheckpointService.ts:129`、`:138`、`:140`、`:142`；搜索结果没有 `c.singleton("gitDiffCheckpoint")`）。它不能被当成 039 后台 file monitor。
- Plugin ModuleBoundary 明确把 long-lived file monitor 归属 Alembic，Plugin 只保留 embedded runtime checkpoint 兼容能力（`AlembicPlugin/lib/codex/ModuleBoundary.ts:252`、`:263`）。
- 共享 schema 里的 `HostTurnMetaInput` 不暴露 `activeFile/cwd/projectRoot/workspaceRoot`，但 `HostIntentFrame.ts` 的服务层类型和 request `_meta` reader 支持这些字段（`AlembicPlugin/lib/shared/schemas/mcp-tools.ts:66`、`:92`，`AlembicPlugin/lib/service/task/HostIntentFrame.ts:29`、`:44`、`:46`、`:47`、`:48`）。如果 039 需要 host 显式传这些字段，应统一 schema 口径。
- Core proposal source 目前没有 `plugin-opportunistic`。039 第一版若要写入现有 proposal repo，要么映射到 `host-agent` source 并用 evidence 标 producer，要么先由总控裁决新增 source（`AlembicCore/src/shared/source-contracts.ts:10`、`:19`）。

### 039 建议收敛

- 第一版不要使用 Plugin git diff 伪装 file monitor；039 必须先判断 Alembic 主体服务是否可用。主体服务可用时 route/defer to Alembic 并 no-op；主体服务不可用时，039 默认使用 git diff 作为 changed file evidence，同时消费 prime/search/guard/submit/record_decision 等当前 tool outcome 和 host-visible evidence。
- 优先实现独立 evidence gate：`strong proposal` 需要 ProjectScope + sourceRefs 或 visible file evidence + tool evidence；`weak hint` 只进 tool result surfaced / telemetry；`no-op` 不写候选。
- 测试边界：git diff parser、sourceRefs matching、evidence gate 可以用 unit / fixture；涉及 Codex Agent 能力的 acceptance 必须使用真实 Codex Agent，不得用 mock / simulator 替代。
- 如果 038 proposal envelope 未稳定，039 可以先实现 gate + surfaced hint，不急于写 proposal repo。
- 如果要持久化，建议先复用 `source: "host-agent"`，把 `producerKind: "plugin-opportunistic"` 放 evidence，避免提前扩 Core source 枚举。

## 总体结论

- 038 不是从零开始：file-change route、dispatcher、handler、sourceRefs、proposal repo、Dashboard consumer 都有底座。最大现实缺口是 daemon collector lifecycle 和 intent / prime evidence enrichment。
- 039 也不是从零开始：Plugin 已能接收 host intent、turn meta、activeFile、sourceRefs，并已有 git diff checkpoint 相关代码可参考。最大现实缺口是 Alembic service availability gate，以及主体服务不可用时默认 git diff evidence 如何进入机会式检测 / evidence gate，而不是 host intent 输入。
- 038 / 039 第一版都应保守复用现有 proposal source/type，除非总控明确把 Core contract 变更列为本阶段目标。

## 边界 / 隔离 / 统一管理 / 连通性审计

### 已由代码确认

- 文件监控归属边界已存在：`RuntimeBoundary` 明确 fileMonitor 的 `longLivedOwner` 是 `alembic-daemon`，dispatcher 是 `FileChangeDispatcher`。这支持“Plugin 不做长期文件监控”的边界。
- Plugin 工具所有权边界已存在：`ServiceRequestBoundary` 明确 Codex-facing MCP tools 仍由 Plugin owns；本地 Alembic 只能通过 explicit resident service APIs 请求能力。这意味着 039 不能把工具所有权转给 Alembic，但可以在能力可用时 route/defer 具体 monitor/evolution 判断。
- Plugin enhancement 能力来源已有统一入口：`EnhancementRoute` 优先使用 residentService capability，runtimeBoundary 只是 compatibility fallback。这支持 039 的 Alembic service availability gate，但当前 gate 还没有专门服务 039。
- source 归一已有边界：`SourceBoundary` 把旧 `ide-agent` / `mcp` 等写入来源归一到 `host-agent`，可支撑 039 第一版不新增 `plugin-opportunistic` proposal source，而在 evidence 中标 producer。
- file-change 后半段连通性已确认：`POST /api/v1/file-changes` -> `fileChangeDispatcher` -> `FileChangeHandler` -> `EvolutionGateway` -> proposal repo / Dashboard consumer 已有路径。
- Plugin intent/search 连通性已确认：`hostDeclaredIntent` / `hostTurnMeta` / `activeFile` / `sourceRefs` 能进入 `HostIntentFrame`、`PrimeSearchPipeline`、resident search 和 intent episode handoff。

### 尚未由代码确认

- 038 native filesystem watcher 主路径不存在或未找到。当前只确认到 git-worktree collector 和 route/handler，未确认替代 VSCode API 级 watcher 的 native watcher 实现。
- 038 git diff 降级 lifecycle 未确认。`DaemonFileChangeCollector` 有代码和测试，但未找到生产启动接线，也未看到 watcher unavailable -> git diff fallback 的统一状态机。
- 039 Alembic service availability gate 未实现。现有 capability summary 可作为输入，但没有看到 039 detector 在运行前按“主体服务可用则 no-op/defer，不可用才 Plugin fallback”的门禁。
- 039 默认 git diff evidence 未接入 opportunistic detector。Plugin 有 `GitDiffCheckpointService` 相关代码，但未找到生产 singleton 注册，也没有看到它输出到 039 evidence gate。
- 039 opportunistic detector / evidence gate 尚未实现。当前只有需求设计层定义，代码里没看到 strong / weak / no-op 分流。
- 真实 Codex Agent acceptance 尚未确认。当前只是测试边界裁决，尚未看到 039 的真实 Codex Agent 验收包或自动化入口。

### 隔离方式判断

- 应隔离的不是 MCP tool ownership，而是 monitor/evolution 判断执行权：Plugin 仍 owns Codex-facing tool surface；Alembic service 可用时，Plugin 将 monitor / git diff / evolution 判断 route/defer 给 Alembic service。
- 038 与 039 的 producer 必须隔离：038 producer 是 `alembic-file-monitor` / Alembic service；039 producer 是 `plugin-opportunistic`，且只在主体服务不可用时出现。
- git diff 能力应统一为 shared evidence/scanner contract，但执行位置按 availability gate 隔离：主体服务可用时由 Alembic 运行；主体服务不可用时 Plugin fallback 运行。

### 统一管理判断

- capability / runtime boundary 已有统一管理雏形，但 file monitor status 目前仍偏粗。需要区分 `watcher-running`、`git-diff-degraded`、`disabled`、`unavailable`，而不是只给 `available + mode`。
- git diff evidence 还没有统一 contract。当前 Alembic 有 `DaemonFileChangeCollector`，Plugin 有 `GitDiffCheckpointService`，二者语义相近但未统一到共享 Core contract。
- proposal source/type 建议第一版继续统一使用现有 `host-agent` / `file-change` 与 `update|deprecate`，把 producerKind、gateResult、gitDiffEvidence 放 evidence，避免过早扩展 Core schema。

### 连通性判断

- 038 handler 链路连通，但事件生产未闭合：route/dispatcher/handler/proposal/Dashboard 连上了，watcher/fallback 到 route/dispatcher 还没确认生产连通。
- 039 host intent/search 链路连通，但 evolution 链路未闭合：host context 到 resident search / intent episode 已连通，git diff evidence -> evidence gate -> surfaced hint/proposal 未连通。
- Alembic service availability 信息可读，但尚未变成 039 的执行门禁：EnhancementRoute / residentService capability 有输入，缺少 039 侧强制 no-op/defer 行为。

## 前置验证与修复项

以下不是后续优化，而是 038 / 039 进入主开发前必须带上的 prerequisite。总控接收后应先作为前置任务包处理：先验证，验证失败就修复；不能跳过后直接派主实现。

### P0-A 038 Watcher / Git Fallback 前置

- 目标：确认并修复 038 的事件生产链路。
- 验证：native filesystem watcher 是否存在、是否随 Alembic daemon / resident 启动、是否能产出 create / modify / delete / rename 事件。
- 修复：若 native watcher 不存在，补 Alembic 主体 watcher；若 watcher 不可用，必须自动降级到 git diff / git-worktree scan。
- 完成证据：watcher running 状态、无 watcher 时 git diff degraded 状态、file-change route 收到事件、handler 创建可读 proposal。

### P0-B 038 Capability / Status 前置

- 目标：防止 Plugin / Dashboard 被错误 file monitor 状态误导。
- 验证：`fileMonitor.available=true` 是否真实代表 watcher running。
- 修复：状态至少区分 `watcher-running`、`git-diff-degraded`、`disabled`、`unavailable`；git diff fallback 不能伪装成 native watcher。
- 完成证据：daemon health / runtimeBoundary / Dashboard/API 能读到真实状态。

### P0-C Shared Git Diff Evidence 前置

- 目标：避免 Alembic 主体与 Plugin 各自维护两套不一致 git diff 语义。
- 验证：`DaemonFileChangeCollector` 与 `GitDiffCheckpointService` 的输出是否能统一成 shared evidence/scanner contract。
- 修复：抽出或对齐 git diff evidence DTO：changed files、event kind、oldPath/newPath、projectScope、ignored reason、diff source、degraded reason。
- 完成证据：038 fallback 和 039 Plugin-only fallback 使用同一套 evidence 语义，执行位置按 availability gate 隔离。

### P0-D 039 Alembic Service Availability Gate 前置

- 目标：主体服务可用时，Plugin 不重复处理同一套文件的 monitor / git diff / evolution。
- 验证：Plugin 是否能可靠判断 Alembic 主体服务可用并能承接当前 project scope。
- 修复：新增 039 gate：available -> route/defer/no-op；unavailable -> Plugin-only git diff fallback。
- 完成证据：主体服务可用时 039 不生成自身 proposal；主体服务不可用时才进入 fallback。

### P0-E 039 Git Diff Evidence / Evidence Gate 前置

- 目标：闭合 039 的 Plugin-only fallback 主链路。
- 验证：默认 git diff evidence 是否进入 opportunistic detector。
- 修复：实现 git diff evidence -> sourceRefs lookup -> evidence gate -> strong / weak / no-op -> surfaced hint/proposal。
- 完成证据：unit / fixture 覆盖 strong、weak、no-op、无 ProjectScope、无 sourceRefs、主体服务可用 no-op。

### P0-F 039 Real Codex Agent Acceptance 前置

- 目标：涉及 Codex Agent 能力的验收不被 mock / simulator 污染。
- 验证：039 是否提供真实 Codex Agent acceptance pack。
- 修复：补真实 Codex Agent 验收路径；无法运行真实 Agent 时标记 `待真实 Codex Agent 验证`，不得关闭。
- 完成证据：真实 Codex Agent 触发 tool / host context / git diff evidence / surfaced hint 的回填证据。

## 建议总控下一步

1. 接收 038 / 039 时，把本文作为 Stage 0 code fact seed，但仍由总控或源仓库窗口复核关键证据。
2. 先创建前置任务包 P0-A 到 P0-F；这些是需求自带验证与修复项，不是后续优化。
3. P0-A / P0-B / P0-C 通过后，再推进 038 主实现。
4. P0-D / P0-E / P0-F 通过后，再推进 039 主实现。
5. 若任一阶段要新增 proposal source/type 或持久化 hint queue，先回到总控裁决 Core contract。

## 仍需确认的问题

- 038：health 中 `fileMonitor.available` 是否必须代表 native watcher 已真实启动，而不是仅 daemon mode 可用？
- 038：intent / prime enrichment 第一版是否必须落在 proposal evidence，还是只在 review consumer 展示？
- 039：`plugin-opportunistic` 是否要成为 Core canonical proposal source，还是先映射到 `host-agent`？
- 039：weak hint 是否需要持久化，还是只在当前 MCP tool result 中 surfaced？
- 039：shared MCP schema 是否需要显式加入 `activeFile/cwd/projectRoot/workspaceRoot`，以便 host 主动传递而不只依赖 request `_meta`？
