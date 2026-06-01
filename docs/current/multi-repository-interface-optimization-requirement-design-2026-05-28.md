# Multi Repository Interface Optimization 需求设计

Design Key：MULTI-REPOSITORY-INTERFACE-OPTIMIZATION-2026-05-28
日期：2026-05-28
状态：ready-for-workspace / automation-ready maintenance requirement
维护窗口：AlembicDesign
总控：AlembicWorkspace
原始计划：[multi-repository-interface-optimization-original-plan-2026-05-28.md](multi-repository-interface-optimization-original-plan-2026-05-28.md)
自动化准备附录：[multi-repository-interface-optimization-automation-readiness-2026-05-30.md](multi-repository-interface-optimization-automation-readiness-2026-05-30.md)
breaking-cleanup 补充：[multi-repository-interface-optimization-breaking-cleanup-addendum-2026-06-01.md](multi-repository-interface-optimization-breaking-cleanup-addendum-2026-06-01.md)

## 定位

本需求是在现有 AlembicWorkspace 工作模式下承接的维护型需求。它的内容是多仓库接口优化，不是新流程架构，也不是新的自动化系统。

总控仍然按现有规则判断是否入账、何时启动、如何拆任务包、是否允许 VAD 自动化领取和如何验收。

## 2026-06-01 用户裁决：开发阶段允许彻底清理

用户在 Alembic 主体和 AlembicPlugin 已完成文件名、方法名、文件夹层级歧义修正后，进一步确认：当前仍处于开发阶段，可以彻底清理兼容层；需求目标应从“保守接口维护”升级为“按最新 canonical code 去除兼容分叉，保持接口契约干净整洁”。

因此，本需求后续不再默认保留旧字段、旧路径、旧函数名、旧目录或旧 wrapper。只要旧入口仅服务历史命名兼容，且不承载真实业务能力、持久化数据读取、已发布外部安装兼容或用户可见行为，就应作为删除候选。

详细代码事实、删除候选和阶段候选见 breaking-cleanup 补充设计。

## 最终目标

让 Alembic 系列仓库之间的真实接口更稳定、更可验证、更少漂移：

- producer / consumer 关系清楚。
- 类型、schema、API response、package export、runtime artifact 和测试 fixture 与真实 contract 对齐。
- 跨仓库接口问题能用最小修复闭环解决。
- 后续无人值守自动化可以领取低歧义接口治理任务，但不能绕过总控判断和确认门禁。

## 真实使用场景

1. `Alembic` route response shape 变化，`AlembicDashboard` API client 或 UI 类型未同步。
2. `AlembicCore` contract export 变化，`Alembic` / `AlembicPlugin` import 或 typecheck 失败。
3. `AlembicPlugin` source 已更新，但 bundled runtime package / skill / MCP export 仍旧。
4. `AlembicAgent` AI provider / model / tool contract 与 `Alembic` internal AI 调用方语义不一致。
5. `AlembicTest` fixture 或 probe 使用了过期 schema，导致测试通过但不能代表真实产品接口。

## 接口优化对象

| 接口面 | Producer | Consumer | 典型问题 | 验证 |
| --- | --- | --- | --- | --- |
| Shared contracts | `AlembicCore` | `Alembic` / `AlembicPlugin` / `AlembicAgent` | export 缺失、类型漂移、重复定义。 | producer + consumer typecheck / boundary lint。 |
| HTTP / resident API | `Alembic` | `AlembicDashboard` / `AlembicPlugin` / `AlembicTest` | response shape、error shape、field name 不一致。 | route unit、client typecheck、fixture check。 |
| Plugin runtime package | `AlembicPlugin` | Codex host / Alembic runtime | source 与 bundled runtime 不一致，MCP schema 未同步。 | runtime build、prepare script、MCP smoke。 |
| AI provider contract | `AlembicAgent` / `Alembic` | internal AI jobs / Plugin handoff | provider id、capability、unavailable path 不一致。 | targeted unit、provider registry check。 |
| Test fixture contract | `AlembicTest` | 总控验收 / 产品仓库 | fixture 偏离真实 schema 或 mock path。 | fixture schema check、real contract probe。 |

## 外部模式调研映射

Design 已联网查找业界常见接口治理模式。结论是：Alembic 不适合直接引入一个重型 contract platform；更适合吸收几类成熟模式，落成现有总控机制内的轻量组合。

| 外部模式 | 关键思想 | Alembic 适配 |
| --- | --- | --- |
| Consumer-driven contract testing / Pact | Consumer 用测试表达自己依赖 provider 的交互，provider 再验证这些 contract；测试应覆盖真实 consumer 逻辑，不能绕过消费代码。 | 对 `AlembicDashboard` -> `Alembic`、`AlembicPlugin` -> `Alembic`、`AlembicTest` -> 产品 schema 使用 consumer-driven targeted tests；test-local mock / fixture 可以存在，但必须被 provider verification 约束，不能变成产品 runtime mock。 |
| OpenAPI / machine-readable HTTP description | HTTP-like API 可用机器可读 description 描述 path、component、schema，供工具生成 docs、client、test。 | 对稳定的 `Alembic` HTTP / resident API 逐步生成 OpenAPI-like contract 或 schema snapshot；第一版不追求全量 OpenAPI，只覆盖 Dashboard / Plugin / Test 真实消费的 endpoint。 |
| API Extractor / API report | TypeScript public API surface 可以生成 Git-tracked report，让 PR 中 API signature diff 可见，并可要求 owner review。 | 对 `AlembicCore`、`AlembicPlugin` runtime export、可能的 `AlembicAgent` provider public surface 采用 API surface snapshot / report；先做报告，不自动下沉 contract。 |
| Breaking-change detection / Buf 思路 | 把当前 schema 与过去 baseline 比较，机械发现 breaking change，并按严格度分类。 | 为 Alembic 自定义 `interface baseline diff`：比较 API surface、schema snapshot、runtime package export、fixture schema；发现 breaking change 后由总控裁决是否允许。 |
| Module boundary enforcement / Nx 思路 | 通过项目标签和 lint / conformance 规则限制依赖方向，避免任意互相引用。 | 不引入 Nx 架构，但吸收 tag / boundary 思路：用现有 lint / scripts 检查 `AlembicCore`、`Alembic`、`AlembicPlugin`、`AlembicDashboard`、`AlembicAgent` 的 import/export 边界。 |
| SemVer / Changesets | public API 必须明确；版本和 changelog 用来表达兼容性影响。 | 当前重点不是发布流程；但对 publishable package / plugin runtime 可记录 interface-impact note，后续再评估 changeset。 |

## 推荐方案

采用 `Interface Baseline + Consumer Verification` 模式：

1. `Interface Map`：先列 producer / consumer / evidence，避免抽象先行。
2. `Contract Anchor`：每个接口面选择一个轻量锚点，例如 TypeScript API report、HTTP schema snapshot、runtime export list、fixture schema。
3. `Consumer Verification`：consumer 用 targeted test 表达依赖，provider 用对应 test / fixture replay 验证。
4. `Breaking Gate`：对 contract anchor 做 diff，breaking change 进入总控裁决，不自动通过。
5. `Boundary Lint`：用现有 lint / script 检查 import/export 方向和 runtime package sync。

不建议第一版直接引入 Pact Broker、完整 OpenAPI 全量生成、Buf/Protobuf 化、Nx monorepo 改造或全仓库 Changesets。那些都比当前问题重，容易把维护型需求变成平台工程。

2026-06-01 更新：在该模式上增加 `Breaking Cleanup` 约束。baseline 不再用于维持旧兼容，而用于确认当前 canonical contract 和 negative guard；consumer verification 不再同时支持新旧两套字段，而是证明 consumer 已经跟随 producer 的唯一 contract。

## 推进口径：最佳实践 + 区域先行

用户 2026-05-30 补充确认：该需求应按照最佳实践方案推进，但每个阶段都先规划完整区域，再进入执行。

Design 将其解释为 `Area-first, dossier-by-dossier execution`：

```text
先选择一个完整接口区域
-> 为该区域建立 producer / consumer / anchor / verification 全图
-> 总控裁决该区域的安全任务和停止项
-> 每次只执行一个 Interface Dossier
-> 完成后回到区域计划更新证据和下一项
```

这能同时避免两种风险：

- 只做零散小修，最后没有形成稳定接口面。
- 一开始就做大平台化重构，超出维护型自动化边界。

### Interface Area

`Interface Area` 是一次阶段规划的完整区域，不等于一次实现任务。第一版建议只使用以下区域：

| Interface Area | 范围 | 典型 anchor | 典型验证 |
| --- | --- | --- | --- |
| `core-shared-contracts` | `AlembicCore` public export 与各 consumer import / type usage。 | API surface report / export list。 | producer build + consumer typecheck。 |
| `alembic-http-resident-api` | `Alembic` HTTP / resident response 与 Dashboard / Plugin / Test 消费。 | schema snapshot / route sample。 | route unit + client typecheck / targeted UI API test。 |
| `plugin-runtime-mcp-skill` | `AlembicPlugin` source、bundled runtime、MCP schema、skill export。 | runtime export list / MCP schema snapshot。 | prepare/build + MCP smoke。 |
| `agent-provider-contract` | `AlembicAgent` provider / tool / unavailable path 与 `Alembic` internal AI 调用方。 | provider registry snapshot / capability matrix。 | targeted provider registry + internal AI caller test。 |
| `test-fixture-contract` | `AlembicTest` fixture / probe / report schema 与产品真实 contract。 | fixture schema snapshot / replay sample。 | fixture schema check + product contract probe。 |

每个阶段可以只选择一个 area。选择 area 后，要先完成该 area 的完整规划，再挑一个 dossier 实施。

### Area Plan

每个 area 启动前，总控应先形成最小 `Area Plan`：

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

没有 `Area Plan` 的阶段，不应进入自动化实现。

### 区域内执行规则

- 区域规划必须完整覆盖该 area 的主要 producer / consumer，不只看一个文件。
- 实现仍然一次只做一个 `Interface Dossier`。
- 如果 dossier 执行中发现 area plan 错误，停止并更新 area plan，不继续扩改。
- 一个 area 完成后，总控再决定进入下一个 area，不能由执行窗口自动跳转。
- 跨 area 的修复默认停止，除非总控明确把它拆成两个 dossier。

## 接口设计方法论

Alembic 多仓库接口设计采用同一套方法论：`Owner -> Consumer -> Anchor -> Compatibility -> Verification -> Change Gate`。

这套方法论的目标是保证接口风格一致，而不是把所有接口抽成同一种技术形态。HTTP response、TypeScript export、runtime artifact、fixture schema 和 AI provider contract 可以有不同锚点，但必须遵守同一套设计口径。

### 1. 先定接口所有权

每个接口必须有唯一 producer。producer 负责定义 contract、默认值、错误语义和兼容策略；consumer 只能适配和验证，不能在本仓库发明同名变体。

判断规则：

- 数据从哪里产生，哪里就是默认 producer。
- 多个 consumer 共享且稳定后，才考虑抽到 `AlembicCore`。
- 只有一个 consumer 的局部类型，不急着下沉。
- Test fixture 不是 producer；它只能跟随产品 contract。

### 2. 先写消费场景，再写字段

接口字段必须服务真实 consumer，不为未来假设预留空壳。

每个接口设计至少回答：

- 谁调用。
- 调用后要做什么判断。
- 哪些字段是必需，哪些字段是诊断 / trace / evidence。
- consumer 缺字段时如何降级。
- 哪些字段不能暴露给 UI / Plugin / Test。

### 3. 每个接口有一个 contract anchor

不同接口面选择不同轻量锚点：

| 接口类型 | contract anchor |
| --- | --- |
| TypeScript public export | API surface report / export list / typecheck fixture |
| HTTP / resident route | schema snapshot / route fixture / response envelope sample |
| Plugin runtime package | runtime export list / generated artifact diff / MCP schema |
| AI provider capability | provider registry snapshot / capability matrix / unavailable path fixture |
| Test fixture | product schema snapshot / fixture replay check |

anchor 必须能被 diff、能被总控复核、能连接到 producer / consumer 验证。

### 4. 统一接口形状审美

新接口优先采用可解释、可扩展、可降级的形状：

- 顶层返回应区分 `status`、`data`、`diagnostics`、`evidence`、`errors`。
- 状态值使用枚举语义，不用裸 boolean 表达复杂状态。
- 字段命名表达业务含义，避免 `info`、`data2`、`resultExtra` 这类含混字段。
- evidence 字段要能追溯来源，例如 source refs、command、test、commit、artifactRef。
- 诊断字段不应成为业务必需字段。
- optional 字段要有明确缺省语义；不能靠 consumer 猜。
- error / unavailable / degraded 是一等状态，不用 mock success 或空对象假装成功。

### 5. 兼容策略更新为开发期 breaking cleanup

本需求原始版本默认走保守兼容路线。用户 2026-06-01 已确认当前仍处于开发阶段，可以彻底清理兼容层。因此本需求更新为：

- 新增字段仍需说明 consumer 是否依赖，避免继续扩大 contract。
- 重命名字段允许作为 breaking cleanup 执行，但必须一次同步 producer、consumer、test、runtime artifact 和 negative guard。
- 删除旧字段 / 旧路径 / 旧函数名允许作为 breaking cleanup 执行，但必须证明它们只是兼容层，不是业务能力。
- 改变字段语义仍比改名更危险，必须写 migration / adapter 删除策略。
- consumer 不得同时保留新旧两套读取分支，除非总控确认有真实存量数据或发布兼容要求。

### 6. Consumer verification 必须靠真实消费路径

验证不能只测 producer 自己：

- producer test 证明接口能产出。
- consumer test 证明真实消费代码能使用。
- fixture test 证明测试数据没有偏离产品 contract。
- runtime package test 证明打包产物没有落后源码。

如果只能用 mock / fixture，必须明确它是 fixture，不得描述成真实 runtime evidence。

### 7. breaking change 交给总控裁决

自动化只能处理低歧义兼容修复。以下变化必须停下：

- contract 所属仓库变化。
- shared contract 下沉到 `AlembicCore`。
- consumer 可见行为变化。
- provider capability / unavailable path 变化。
- fixture 与产品 schema 冲突但无法判断谁对。
- 需要迁移历史数据或发布版本策略。

### 8. 每个接口任务都要有 Interface Dossier

总控 Stage 0 或执行窗口领取前，应能写出最小 `Interface Dossier`：

```text
Interface:
Producer:
Consumers:
Contract anchor:
Current evidence:
Allowed change:
Forbidden change:
Compatibility impact:
Verification:
Stop condition:
```

没有 dossier 的“接口优化”不能进入无人值守自动化。

## 可自动化领取的任务

这类任务可以进入现有 TODO / VAD 自动化候选，但必须有明确证据：

- build / typecheck / lint 指向跨仓库 interface break。
- API fixture 与真实 route / client 不一致。
- package export / import boundary 明确失败。
- generated runtime artifact 与 source contract 明确不同步。
- 测试 fixture 与正式 schema 不一致，且有真实消费方。

## 必须停止确认的任务

命中以下情况，不得无人值守推进：

- 需要新增产品能力或改变用户可见行为。
- 需要删除、降级、迁移或替换现有业务能力。
- 旧入口可能承载持久化历史数据、已发布包、用户安装目录、runtime cache 或真实外部输入兼容。
- contract 应该下沉到 `AlembicCore` 还是留在局部仓库不清楚。
- producer 和 consumer 各自都有合理但冲突的 contract。
- 没有最新 canonical code 证据、没有真实消费方或只有“看起来更干净”的抽象理由。
- 涉及发布链路、安全 / 权限、真实 provider、外部服务或数据迁移。

## 分阶段建议

| 阶段 | 目标 | 主要动作 | 完成条件 |
| --- | --- | --- | --- |
| Stage 0 | breaking cleanup 接口事实清单 | 总控先选择一个 `Interface Area`，扫描该区域 canonical producer、consumer、旧兼容分叉、delete candidates、negative guard。 | 形成该 area 的 `Area Plan` 与 producer / consumer / delete evidence 清单，不改代码。 |
| Stage 1 | canonical contract anchor 选择 | 为该 area 的接口面选 API report、schema snapshot、runtime export list、fixture schema 或 negative import guard。 | anchor 可生成、可 diff、可被总控复核，并能阻止旧入口回归。 |
| Stage 2 | 第一批兼容层删除 | 从该 area 内选择 canonical 已清楚、旧入口仅为兼容层、验证命令可运行的一个 dossier。 | 删除旧分支并完成 producer + consumer 验证；不保留双分支兼容。 |
| Stage 3 | 区域内同步收敛 | 对该 area 内 source contract、runtime package、generated client、fixture、测试名称和文档口径做同步。 | 生成命令和 diff 可复核，不跨 area 扩大。 |
| Stage 4 | 区域验收与下一 area 选择 | 总控复核该 area 的 anchor、verification、remaining stop items。 | 决定 accepted / needs-rework / pick-next-area / stop。 |
| Stage 5 | 自动化候选规则固化 | 把已验证可无人值守的 breaking-cleanup area + dossier 类型沉淀为总控 TODO 领取边界。 | 不新增流程，只补现有总控任务模板 / checklist。 |

## 完成定义

- 有完整 producer / consumer 接口面清单。
- 每个执行阶段先有完整 `Interface Area` 规划，再挑单个 dossier 实施。
- 第一批任务只从最新 canonical code 事实、真实失败证据或明确 drift 中选择。
- 每个任务都写清允许改动、禁止改动和验证命令。
- 不新增空接口、空 adapter、空 provider。
- 不改变业务行为，但允许删除纯兼容旧命名 / 旧路径 / 旧字段的分支。
- 不保留新旧双分支 contract；删除后有 negative guard。
- 不绕过总控验收；执行窗口回填只作为待审输入。
- 后续可被 VAD 自动化领取的事项有清晰边界和停止条件。
- 有至少一个接口面的 contract anchor 和 consumer verification 被证明可用。

## 与当前主线关系

- 不打断 `AI-MOCK-REMOVAL-2026-05-28`。
- 不替代 `IntentKnowledgeEvolutionBaseline`。
- 不提前启动 038 / 039。
- 适合作为总控维护型 backlog，在主线间隙或自动化模式下领取低风险任务包。

## 建议总控下一步

总控接收后，先做 Stage 0 read-only interface inventory，不直接派实现窗口。

Stage 0 应输出：

- producer / consumer 接口清单。
- 当前 canonical code 证据、真实失败证据或 drift 证据。
- 旧兼容入口和删除候选清单。
- 每个接口面建议采用的 contract anchor。
- 第一批可无人值守 breaking-cleanup 候选任务。
- 必须停下等确认的候选任务。

用户 2026-05-30 补充计划自动化该需求。Design 建议自动化第一步只做只读 inventory，并要求每个实现任务先具备 `Interface Dossier`。自动化安全类型、停止条件和任务包模板见自动化准备附录。

## 自动化确认口径

用户已说明计划自动化该需求。按 Design 推荐，自动化默认采用以下口径：

- 第一批只允许从 build / typecheck / lint / targeted test 明确失败、source / runtime / fixture 明确 drift，或最新提交已经明确 canonical 但旧兼容层残留的事项中选择。
- 第一版采用轻量 contract anchor，不引入 Pact Broker / 完整 OpenAPI / Nx / Buf 这类重型平台。
- runtime / generated artifact 同步允许无人值守执行，但必须有确定生成命令和 source contract。
- 兼容层删除允许无人值守执行，但必须有唯一 canonical producer、明确 consumer 验证和 negative guard。
- contract 下沉到 `AlembicCore` 一律作为停止条件，除非已有总控确认和真实消费方证据。
- 没有 `Interface Dossier` 的候选只能停在 Stage 0 事实清单。

## 已确认执行口径

- 用户 2026-06-01 已确认本需求可以按开发期 breaking cleanup 执行，目标是彻底清理冗余兼容分叉，而不是继续保守保留旧命名 / 旧路径 / 旧字段。
- 用户已确认第一优先级为 `core-host-agent-workflow-contract`。Design 建议总控 Stage 0 先围绕 Core `External*` / `external-agent` / `*-external` 到 `HostAgent*` / `host-agent` 的 producer / consumer contract 做只读 inventory。
- 用户已确认第二优先级为 `api-ai-runtime-contract`。Core / Alembic / Dashboard / Plugin 应统一迁移到 `apiAi` / `jobs.api-ai.*` contract，并删除 Alembic `ApiAiCompatibility.ts` 这类桥接层。
- 用户已确认删除边界：纯旧命名、旧路径、旧字段、旧 wrapper、旧测试 fixture 可以删除；若涉及持久化历史数据、已发布包安装兼容、runtime cache 或真实外部输入，必须停止交回总控。
- 用户已确认用户可见文案和 contract 口径尽量统一为 `API AI`，避免继续把 API AI runtime 写成 `internal AI`。
- 用户已确认 Plugin 小项 `external_agent` role id 可改为 `host_agent`。执行前仍需总控只读复核是否存在存量 auth / constitution 数据依赖。

## 仍需总控裁决

- Stage 0 是否现在进入当前自动化队列，还是等 Workspace Workflow Optimization 当前阶段完成后再领取。
- Stage 0 inventory 是由总控窗口自执行，还是拆给相关仓库窗口做只读回填。Design 推荐总控先自执行最小 inventory，再决定是否拆窗口。
- `external_agent` -> `host_agent` 是否存在存量 auth / role-id 数据迁移风险；若只是开发期模板 / fixture 命名，可以直接作为 breaking cleanup 处理。
- Alembic `lib/external/mcp/README.md` retired marker 的最新产品状态：用户说明已自行删除；Design 只读复核时仍曾看到本机工作树存在该文件且无删除状态，因此总控 Stage 0 应重新读取最新 Alembic 状态后再裁决是否还需处理。

## 调研来源

- Pact consumer-driven contract testing: <https://docs.pact.io/consumer>
- OpenAPI description structure: <https://learn.openapis.org/specification/structure.html>
- API Extractor API report: <https://api-extractor.com/pages/setup/configure_api_report/>
- Buf breaking change detection: <https://buf.build/docs/breaking/>
- Nx module boundaries: <https://nx.dev/docs/features/enforce-module-boundaries>
- Semantic Versioning: <https://semver.org/>
