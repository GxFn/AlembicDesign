# Multi Repository Interface Optimization 需求设计

Design Key：MULTI-REPOSITORY-INTERFACE-OPTIMIZATION-2026-05-28
日期：2026-05-28
状态：ready-for-workspace / maintenance requirement
维护窗口：AlembicDesign
总控：AlembicWorkspace
原始计划：[multi-repository-interface-optimization-original-plan-2026-05-28.md](multi-repository-interface-optimization-original-plan-2026-05-28.md)

## 定位

本需求是在现有 AlembicWorkspace 工作模式下承接的维护型需求。它的内容是多仓库接口优化，不是新流程架构，也不是新的自动化系统。

总控仍然按现有规则判断是否入账、何时启动、如何拆任务包、是否允许 VAD 自动化领取和如何验收。

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

### 5. 兼容策略默认保守

接口优化默认走兼容路线：

- 新增字段通常允许，但必须说明 consumer 是否依赖。
- 重命名字段是 breaking change，必须走 change gate。
- 删除字段是 breaking change，必须走总控确认。
- 改变字段语义比改类型更危险，必须写 migration / adapter 策略。
- consumer 只因美观而要求 producer 改名，不自动通过。

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
- 需要删除、降级、迁移或替换现有能力。
- contract 应该下沉到 `AlembicCore` 还是留在局部仓库不清楚。
- producer 和 consumer 各自都有合理但冲突的 contract。
- 没有失败证据、没有真实消费方或只有“看起来更干净”的抽象理由。
- 涉及发布链路、安全 / 权限、真实 provider、外部服务或数据迁移。

## 分阶段建议

| 阶段 | 目标 | 主要动作 | 完成条件 |
| --- | --- | --- | --- |
| Stage 0 | 接口事实清单 | 总控扫描各仓库当前 known breaks、typecheck / build / lint 失败、重复 contract 和 runtime sync 点。 | 形成 producer / consumer / evidence 清单，不改代码。 |
| Stage 1 | contract anchor 选择 | 为每个接口面选 API report、schema snapshot、runtime export list 或 fixture schema。 | anchor 可生成、可 diff、可被总控复核。 |
| Stage 2 | 第一批最小接口修复 | 只选择失败证据明确、影响面小、验证命令可运行的接口漂移。 | 每个修复有最小 diff 和 producer + consumer 验证。 |
| Stage 3 | runtime / generated artifact 同步 | 对 source contract 与 runtime package / generated client 不一致的项做同步。 | 生成命令和 diff 可复核。 |
| Stage 4 | fixture 与测试 contract 收敛 | 让 `AlembicTest` fixture / probe 跟随真实 schema，不使用误导性 mock path。 | 测试证据能对应产品 contract。 |
| Stage 5 | 自动化候选规则固化 | 把已验证可无人值守的类型沉淀为总控 TODO 领取边界。 | 不新增流程，只补现有总控任务模板 / checklist。 |

## 完成定义

- 有完整 producer / consumer 接口面清单。
- 第一批任务只从真实失败证据或明确 drift 中选择。
- 每个任务都写清允许改动、禁止改动和验证命令。
- 不新增空接口、空 adapter、空 provider。
- 不改变业务行为。
- 不绕过总控验收；执行窗口回填只作为待审输入。
- 后续可被 VAD 自动化领取的事项有清晰边界和停止条件。
- 有至少一个接口面的 contract anchor 和 consumer verification 被证明可用。

## 与当前主线关系

- 不打断 `AI-MOCK-REMOVAL-2026-05-28`。
- 不替代 `IntentKnowledgeEvolutionBaseline`。
- 不提前启动 038 / 039。
- 适合作为总控维护型 backlog，在主线间隙或自动化模式下领取低风险任务包。

## 建议总控下一步

总控接收后，先做 Stage 0 代码事实清单，不直接派实现窗口。

Stage 0 应输出：

- producer / consumer 接口清单。
- 当前真实失败证据。
- 每个接口面建议采用的 contract anchor。
- 第一批可无人值守的候选任务。
- 必须停下等确认的候选任务。

## 仍需确认

- 第一批是否只允许从 build / typecheck / lint / targeted test 明确失败中选择？Design 建议是。
- 是否接受第一版采用轻量 contract anchor，而不是引入 Pact Broker / 完整 OpenAPI / Nx / Buf 这类重型平台？Design 建议是。
- runtime / generated artifact 同步是否允许无人值守执行？Design 建议允许，但必须有确定生成命令和 source contract。
- contract 下沉到 `AlembicCore` 是否一律作为停止条件？Design 建议是，除非已有总控确认和真实消费方证据。

## 调研来源

- Pact consumer-driven contract testing: <https://docs.pact.io/consumer>
- OpenAPI description structure: <https://learn.openapis.org/specification/structure.html>
- API Extractor API report: <https://api-extractor.com/pages/setup/configure_api_report/>
- Buf breaking change detection: <https://buf.build/docs/breaking/>
- Nx module boundaries: <https://nx.dev/docs/features/enforce-module-boundaries>
- Semantic Versioning: <https://semver.org/>
