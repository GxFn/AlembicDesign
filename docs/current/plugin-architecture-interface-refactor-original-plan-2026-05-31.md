# Plugin Architecture Interface Refactor 原始计划书

Design Key：PLUGIN-ARCHITECTURE-INTERFACE-REFACTOR-2026-05-31
日期：2026-05-31
状态：已确认 / ready-for-workspace
维护窗口：AlembicDesign
总控：AlembicWorkspace

## 用户目标

深度检查 AlembicPlugin 的代码结构和接口关系，形成一个新的架构重构与优化需求：按功能和接口边界重新组织 Plugin，降低后续实现窗口误判职责、重复实现和继续堆叠逻辑的风险。

## 用户原话 / 关键约束

```text
深度检查 Plugin 的代码结构和接口关系，新需求为 Plugin 基于功能和接口的架构重构与优化
```

补充确认：

```text
你直接进行代码逻辑调研，真实的代码实现和功能使用，我建议是不要做兼容，直接使用新接口与逻辑，确认好其他仓库对接就可以
```

继承已确认的长期约束：

- Plugin 是独立整体，但职责是 Codex plugin / MCP / skill / channel / marketplace / runtime adapter / install validation / Codex host adaptation。
- 不重新长出 Agent runtime、AI provider runtime、Tool V2 runtime。
- 不把 Alembic 主体 daemon、Dashboard、long-lived file monitor 或 internal AI 当成 Plugin 长期拥有能力。
- Recipe source file path / sourceRefs / evidenceRefs 仍是关键证据，不能在重构中丢失。
- 本窗口是 AlembicDesign，不直接修改 AlembicPlugin 源码或派发实现窗口。

补充裁决：

- `daemon-server.js` / embedded HTTP runtime 纳入本需求后半段处理，不另起独立 runtime simplification 需求。
- 处理方式是先拆分职责，确认 Plugin 仍需要的 runtime / recovery / smoke 内容，再清理删除不属于 Plugin 的主体式能力。

## 为什么现在做

近期 Plugin 已连续完成或接入：

- Project Skill service / knowledge scope skill governance。
- Plugin cold-start / rescan Recipe loop 测试优化。
- 038 / 039 相关 native watcher / Plugin opportunistic evolution 边界。
- IDE Agent cold-start packet / Plugin surface 合并任务。

这些能力让 Plugin 真实链路更完整，也暴露出结构问题：现有代码中 Codex-facing surface、embedded Alembic runtime compatibility、resident service adapter、MCP handler、HTTP / daemon compatibility 和 service container 仍混在主体式分层里。若不先整理接口边界，后续需求会继续在大文件中堆逻辑，或把主体职责误放回 Plugin。

## 期望结果

需求完成后应成立：

- Plugin 代码按功能区和接口边界表达职责，开发者能快速判断某能力属于 Plugin owned、Core shared、Alembic resident、embedded compatibility 还是 legacy alias。
- Codex tool surface 有单一可复核 catalog / policy / gateway / annotation 入口，不再出现工具数量、描述、可见性和权限映射漂移。
- 外层 Codex MCP router、内层 handler executor、resident adapter、prime material presenter、project skill delivery、opportunistic evolution surface 都有独立 contract 和 focused tests。
- 不保留长期兼容双轨；旧入口删除必须同阶段迁移真实消费者、测试、文档、runtime packaging 和跨仓库对接。

## 最终完成定义草案

本需求完成时，至少要有：

1. Plugin interface inventory：功能区、公开工具、内部 service、resident API、embedded runtime compatibility、legacy alias、测试覆盖和禁止边界清单。
2. Tool surface consolidation：Codex local tools + core MCP tools + annotations + gateway mapping + policy 的单一事实源或可机械校验同步机制。
3. Execution router decomposition：`CodexMcpServer` 的 projectRoot / preflight / local codex tools / resident route / embedded MCP executor / opportunistic evolution surface 分离到命名明确的小模块。
4. Resident service adapter decomposition：拆出 ProjectScope、search、IntentEpisode、jobs、dashboard、probe capability clients；若存在 assembly facade，只能作为同阶段迁移内部结构，不作为长期兼容入口。
5. Prime / task surface cleanup：保留 intent、search、vector、relation、Recipe sourceRefs / evidenceRefs；把 visible receipt / presenter 与 lifecycle handler 分离。
6. Embedded runtime compatibility boundary：HTTP / daemon / ServiceContainer / git-diff checkpoint 哪些保留、哪些只是兼容、哪些未来可删，有文档、测试和代码命名支撑。
7. 验证：`npm run build:check`、边界 lint、focused interface tests、至少一个 Codex MCP smoke 或 scenario verify 通过。

用户已确认不按 compatibility-first 推进。进入实现前仍需总控确认接收时机、阶段顺序和当前主线关系。

## 当前主线关系

下一主线候选 / TODO 候选。

理由：

- 当前总控状态显示 AlembicPlugin 已有执行中的合并任务包 `PLUGIN-BUNDLE-IDE039-P1`，本需求不应由 Design 直接打断该回填。
- 本需求属于架构治理和接口边界优化，可在当前 Plugin 主线回填后接入。
- 若当前 Plugin 合并任务包回填暴露结构阻塞，总控可把本需求提升为前置收敛。

## 初始范围

包含：

- AlembicPlugin 代码结构、接口关系、工具 surface、MCP execution path、resident adapter、embedded runtime compatibility、Project Skill、prime / intent / search、opportunistic evolution fallback、测试结构。
- 以新接口 / 新逻辑为目标的阶段候选；旧入口不长期保留。
- 总控可接收的代码事实复核和重构 TODO / Backlog。

不包含：

- UI / Dashboard 改造。
- AlembicCore public contract 大改。
- Alembic daemon / Dashboard / file monitor 主体职责迁入 Plugin。
- Agent runtime / AI provider runtime 重建。
- 删除已验收的 cold-start / rescan / Project Skill / 039 opportunistic 能力。
- 为兼容而长期保留旧新双入口。
- 在 Design 窗口直接修改 Plugin 源码。

## 初步仓库影响

| 仓库 / 窗口 | 初步判断 | 理由 |
| --- | --- | --- |
| Alembic | 观察 / contract producer | 需要确认 resident service / daemon capability contract 是否足够清晰；Plugin 新接口应对接当前 Alembic HTTP resident API，不做旧 fallback。 |
| AlembicCore | 观察 / shared contract | Tool schema、RecipeProductionGateway、ProjectScope、SourceRefs、resident service、host-agent workflow 等 public contract 优先复用；若发现 contract 缺口再单独确认。 |
| AlembicAgent | 无任务 | 本需求禁止重建 Agent runtime / AI provider。 |
| AlembicDashboard | 无任务 | 无 UI。 |
| AlembicPlugin | 参与 | 目标仓库。 |
| AlembicTest | 观察 | 第一版可由 Plugin focused tests / smoke 验证；真实 Codex Agent 验收只在后续需要时交给总控裁决。 |
| AlembicDesign | 草案 / 代码事实复核 | 负责需求设计和交接，不派发实现。 |

## 已知事实

已完成代码事实复核：

- 第一轮代码事实：[plugin-architecture-interface-refactor-code-fact-review-2026-05-31.md](plugin-architecture-interface-refactor-code-fact-review-2026-05-31.md)
- 代码逻辑与真实使用调研：[plugin-architecture-interface-refactor-code-logic-research-2026-05-31.md](plugin-architecture-interface-refactor-code-logic-research-2026-05-31.md)

核心事实：

- `bin/codex-mcp.ts` 是轻 shim。
- `CodexMcpServer` 是 Codex-facing outer router。
- `McpServer` 是 embedded Alembic V3 handler executor。
- `ToolPolicy` 与 `tools.ts` 分散维护 tool surface。
- `AlembicResidentServiceClient` 是大 facade，合并多个 resident API。
- `ProjectSkillService` / `ProjectSkillDelivery` 是当前较清晰的目标样式。
- `PluginOpportunisticEvolution` 已表达 service gate + git diff evidence + evidence gate，但需要和 embedded checkpoint / file-change compatibility 命名分清。
- Codex host 的真实消费面是 `.mcp.json`、wrapper、`runtime.tgz`、`alembic-codex-mcp`、skills、marketplace 和 release/smoke scripts。
- AlembicDashboard 直接消费 Alembic daemon health / ProjectScope，不消费 Plugin API。
- AlembicCore 禁止承接 Codex / MCP / plugin runtime implementation，只能承接 shared public contracts。

## 待补代码事实

- 运行总控可接受的 dependency / import graph，确认各功能区实际 import 方向。
- 生成 packaged runtime 必需文件 / HTTP route consumption dossier，作为裁剪 embedded compatibility 的前置。
- 检查 current Plugin 合并任务包回填后新增的 IDE packet / 039 代码是否改变重构入口。
- 统计每个候选阶段的 affected files 和 focused test set。

## 外部调研初判

- 是否需要联网：第一阶段不需要。
- 理由：本需求当前核心是本地代码结构与已确认仓库职责边界；外部最佳实践不能替代真实代码事实。
- 若后续总控需要方法论补充，可查官方 MCP SDK 文档、plugin architecture / hexagonal architecture / adapter boundary / facade decomposition 资料，但应只作为方法参考，不替代本地 contract。

## 风险 / 设计分叉

- 风险：把“不要兼容”误解成不迁移消费者就直接删除，导致已验收 cold-start / rescan / Project Skill / 039 回退。
- 风险：过度追求“干净目录”，在未替换 runtime package、smoke 和 `alembic_codex_*` recovery 依赖前，把仍需要的 route 错删。
- 风险：只拆文件不建立 contract tests，未来仍会漂移。
- 分叉 1：先整理 tool surface，还是先拆 CodexMcpServer。推荐先 tool surface，因为它是所有后续路由的输入。
- 分叉 2：HTTP / daemon compatibility 是先建立 runtime contract 还是直接删减。推荐先做 runtime contract dossier，再按未消费证据删除。
- 分叉 3：Resident client 是一次拆完还是 facade 内拆。推荐按 capability clients 同阶段迁移调用方，facade 不作为长期兼容层。

## 开放问题

1. 已确认：不做长期兼容，直接新接口与新逻辑。
2. 已确认：先查真实代码实现和功能使用，确认其它仓库对接。
3. 已确认：`daemon-server.js` / embedded HTTP runtime 纳入本需求后半段裁剪，不另起 runtime simplification 需求。
4. 仍需总控裁决：当前 Plugin 合并任务包回填前，本需求是否立即接收为下一阶段，还是等待当前 Plugin 主线验收后接收。

## 需要确认

本原始计划书不是实现计划。用户已确认“不做兼容，直接新接口与新逻辑”；可以交给总控接收评审。

总控接收前仍禁止：

- 写执行 wave。
- 建议发送实现窗口。
- 把阶段候选写成派发顺序。
