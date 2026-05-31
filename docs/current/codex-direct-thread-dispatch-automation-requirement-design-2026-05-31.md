# Codex Direct Thread Dispatch Automation 需求设计

Design Key：CODEX-DIRECT-THREAD-DISPATCH-2026-05-31
日期：2026-05-31
状态：ready-for-workspace
维护窗口：AlembicDesign
总控：AlembicWorkspace
原始计划：[codex-direct-thread-dispatch-automation-original-plan-2026-05-31.md](codex-direct-thread-dispatch-automation-original-plan-2026-05-31.md)
代码事实复核：[codex-direct-thread-dispatch-automation-code-fact-review-2026-05-31.md](codex-direct-thread-dispatch-automation-code-fact-review-2026-05-31.md)

## 原始计划书

- 原始计划书：[codex-direct-thread-dispatch-automation-original-plan-2026-05-31.md](codex-direct-thread-dispatch-automation-original-plan-2026-05-31.md)
- 原始计划书确认状态：用户已直接要求新建需求并指定目标方向。
- 用户确认时间：2026-05-31。

## 已确认目标

将 Codex Automation Closed Loop 的默认跳转能力，从 1 分钟 heartbeat automation 投递，改为直接向目标窗口线程投递 prompt。

完成后：

- 总控仍生成 dispatch packet / delivery envelope。
- delivery adapter 默认直接向目标 Codex thread 发送 prompt。
- 目标窗口应立即收到新 turn 并执行，不需要等待 `FREQ=MINUTELY;INTERVAL=1`。
- target result envelope 和 controller review 仍保留。
- controller-return 同样默认 direct thread dispatch。
- heartbeat 退为 fallback / keep-live / 显式 schedule 能力。

## 当前主线关系

`TODO 候选 / automation-flow-optimization`。

理由：

- 本需求优化 workspace 自动化闭环，不改变 038 / 039 当前产品主线完成定义。
- 它直接接续 `CODEX-AUTOMATION-CLOSED-LOOP-2026-05-29`，但只替换 delivery transport，不重新设计派发协议。
- 若总控近期继续推进无人值守自动化，它应作为 P1 automation flow 优化接收。

## 用户场景

开发者或总控开启无人值守多窗口执行后：

1. 总控判断下一步应发给 `AlembicPlugin`。
2. 总控生成目标化 prompt 与 dispatch packet。
3. delivery adapter 直接把 prompt 送入 `AlembicPlugin` 对应 Codex 线程。
4. 若目标线程空闲，使用新 turn 立即处理；若目标线程正在执行，使用 steering / busy-aware delivery 追加到 active turn 或 fail closed，不能默认打断。
5. `AlembicPlugin` 完成后写 `TargetResultEnvelope`。
6. 若本组 ready，controller-return 直接把总控验收 prompt 送入总控线程。
7. 总控读取 result envelope、commit、diff、命令输出和报告，决定接受、返工、阻塞或下一波。

用户不再需要等待 1 分钟 heartbeat，也不会看到普通路径残留的一次性 automation 对象。

## 功能闭环

- 输入：
  - `ControllerDispatchPacket` 或 `ControllerReturnEnvelope`。
  - 本地 thread registry 中的真实 target / controller thread id。
  - 总控生成的 target-specific prompt。
  - transport policy：默认 `direct-thread`，失败可按策略 fallback。
- 输出：
  - direct thread delivery 状态记录。
  - 目标线程中的新 turn，或 active turn 中的 steering input。
  - `TargetResultEnvelope` 或总控验收 turn。
- 状态变化：
  - `.workspace-local/codex-automation-loop/` 增加 delivery transport result / sentAt / fallbackReason / status。
  - 不把 raw thread id 写入 tracked 文档。
  - 默认不创建 Codex heartbeat automation。
- 生产方：
  - 总控计划层生产 dispatch packet 与 prompt。
  - delivery adapter 生产 direct send action / delivery log。
- 消费方：
  - 目标 Codex thread / controller thread。
  - `codex-automation-target` / `codex-automation-controller` skill。
  - 总控 review。
- 失败路径：
  - direct capability 不存在：fail closed 或按显式策略 fallback heartbeat。
  - thread id 缺失 / 占位：停止，不投递。
  - idle thread 上 API 成功但 UI 不刷新 / turn 不启动：标为 delivery failure，不当作已送达。
  - busy thread 上 active turn 可 steer：使用 `turn/steer` 或总控 thread-send 工具等价能力，并记录 `steered`。
  - busy thread 上 active turn 不可 steer：记录 `activeTurnNotSteerable` / busy failure，交回总控裁决，不静默 interrupt。
  - direct send 重复：按 deliveryId / correlationId 去重。
  - fallback heartbeat 创建后仍需 one-shot cleanup。

## 需求明确性检查

- 完整功能闭环：已明确 controller -> target -> result -> controller review，只替换 delivery transport。
- 验证方式：unit tests + direct capability probe + 真实两窗口 smoke。
- 完成定义：见本文“验证策略”和“完成定义”。
- 已补充事实：当前 Codex App host tool 面已暴露 `create_thread` / `send_message_to_thread` / `read_thread`；Design 已用 projectless thread probe 验证 idle、busy 单条、busy 多条和 invalid thread id 行为。
- 已补充事实：idle thread 上 `send_message_to_thread` 会创建新 turn 并执行；busy thread 上 follow-up 会追加到同一 in-progress turn，最终同一 turn 同时包含初始任务和 follow-up marker；invalid thread id fail closed。
- 仍需总控复核的问题：如何把 host thread-send surface 接入 `codex-control-workspace` delivery adapter / envelope 输出、如何记录 delivery run、真实目标 / 总控已登记线程 smoke 是否需要 AlembicTest 辅助观察。

## 仓库边界

| 仓库 / 窗口 | 职责 | 包含范围 | 不包含范围 |
| --- | --- | --- | --- |
| AlembicWorkspace / codex-control-workspace | 主实现归属 | `codex-automation-loop.mjs`、tests、skills、README、AGENTS managed block、local runtime delivery log | 产品功能、总控验收裁决自动化 |
| AlembicDesign | 需求设计 | 本需求文档、代码事实复核、handoff board | 实现、验收、TODO 入账 |
| AlembicTest | 可选验证 | 真实 Codex 两窗口 smoke 或 UI 刷新观察 | 产品真实项目测试 |
| AlembicPlugin | 观察 | 可能作为 target window 参与 smoke | 不实现 Plugin 产品逻辑 |
| Alembic | 无任务 / 观察 | 无 | 不改 Alembic daemon / Dashboard server |
| AlembicCore | 无任务 | 无 | 不新增 core contract |
| AlembicAgent | 无任务 | 无 | 不改 agent runtime |
| AlembicDashboard | 无任务 | 无 | 不做 UI |

## 外部调研判断

- 是否需要联网：是，已做官方轻量核对。
- 判断理由：Codex direct thread delivery 是新近能力，必须确认官方 thread / turn 模型，不可只凭历史 automation 经验。
- 优先来源：OpenAI / Codex 官方文档。
- 外部结论：
  - Codex App Server 提供 thread lifecycle 与 turn 级输入模型：`turn/start` 开始新 turn，`turn/steer` 对 active turn 追加输入，`turn/interrupt` 取消 in-flight turn。
  - 官方工程文章展示 App Server 支持通过 debug client 发送消息并驱动 agent turn。
  - 官方 Automations 文档仍支持 schedule/thread automation，但这只是 delivery channel 之一，不应再绑定为 AlembicWorkspace 的唯一默认路径。
  - 本机 app-server probe 已验证：busy active turn 使用 `turn/steer` 可追加输入并返回同一 `turnId`；最终响应同时包含初始任务标记和 steering 标记。
  - 本机 Codex App host thread-send probe 已验证：idle follow-up 创建新 turn；busy follow-up 不开第二 turn、不默认 interrupt，而是追加为同一 in-progress turn 的后续 userMessage。
- 对本地方案约束：
  - direct thread dispatch 应优先走 Codex App Server / SDK / host thread tool 等正式能力，并区分 idle start / busy steer / explicit interrupt。
  - 不应把 UI 点击模拟作为默认实现。
  - `send_message_to_thread` 已能作为第一版 direct delivery 的 host tool 语义依据；若总控要求“可见窗口 UI 刷新”作为验收证据，仍需用真实 target/controller 窗口 smoke 补观察。

参考：

- [Codex App Server](https://developers.openai.com/codex/app-server)
- [Unlocking the Codex harness: how we built the App Server](https://openai.com/index/unlocking-the-codex-harness/)
- [Codex Automations](https://openai.com/academy/codex-automations)

## 代码事实与调研缺口

### AlembicWorkspace / codex-control-workspace

- 已有能力：
  - `register-thread` 写本地 thread registry，并校验占位 thread id。
  - `create-dispatch` 生成 `ControllerDispatchPacket`。
  - `build-delivery` 生成 `DeliveryEnvelope`。
  - `build-controller-return` 生成 `ControllerReturnEnvelope`。
  - `submit-result` 生成 `TargetResultEnvelope`。
  - `review-results` 聚合 result envelope readiness。
  - stdout redaction 与本地 runtime raw thread id 分离。
- 关键文件：
  - `scripts/codex-automation-loop.mjs`
  - `scripts/codex-automation-loop.test.mjs`
  - `skills/dev/codex-automation-target/SKILL.md`
  - `skills/dev/codex-automation-controller/SKILL.md`
  - `skills/dev/control-workspace-governance/references/codex-automation-loop.md`
  - `skills/dev/control-workspace-governance/references/control-architecture.md`
  - `README.md`
  - `AGENTS.md`
- 缺口：
  - `build-delivery` / `build-controller-return` hardcode heartbeat。
  - 没有 direct transport enum / adapter / capability probe。
  - tests 没覆盖 direct transport。
  - skill 和 docs 的普通路径仍写 heartbeat cleanup。

### AlembicDesign / 总控交接

- 本设计窗口已完成：
  - 代码事实复核。
  - 官方资料轻量核对。
  - Codex App host thread-send idle / busy / invalid thread probe。
  - 原始计划与需求设计。
  - handoff board 登记。
- 仍需总控复核：
  - `codex-control-workspace` delivery adapter 如何消费 direct thread payload。
  - 是否要求真实可见 target/controller UI smoke 作为进入实现前或实现后的验收证据。
  - heartbeat fallback 的实现策略。
- 是否处于 detached-design-mode：否，已读取父级 workspace index / 当前状态。

### AlembicTest / 真实验证

- 是否纳入：建议 Stage 5 纳入，但不是 Stage 0 必需。
- 理由：direct thread dispatch 的核心验证是“目标窗口 UI 是否立即出现并处理”，这比普通 unit test 更接近真实 Codex 桌面行为。
- 目标项目：不需要真实产品项目，可用 AlembicWorkspace 自身和一个目标仓库窗口做 smoke。

## 代码实现依赖调研

- 是否需要单独调研附件：总控接收后建议把 Design probe 结果整理为 Stage 0 capability baseline，再进入 transport contract；不需要再从零证明 host thread-send 是否存在。
- 建议调研入口：
  - `send_message_to_thread` payload 如何由 delivery adapter 调用或由总控 operator 消费。
  - Codex App Server / SDK / CLI debug 是否作为 host tool 不可用时的备选 direct adapter。
  - `.workspace-local/codex-automation-loop/thread-registry/` 与 direct API id 是否同源。
  - direct send 后 UI / thread readback / target execution 证据。
- 关键生命周期：
  - register thread -> create dispatch -> build delivery -> direct send -> target result -> review-results -> direct controller return。
- 共享状态 / 持久化位置：
  - `.workspace-local/codex-automation-loop/dispatch-packets`
  - `.workspace-local/codex-automation-loop/delivery-envelopes`
  - `.workspace-local/codex-automation-loop/target-results`
  - `.workspace-local/codex-automation-loop/thread-registry`
  - 建议新增 `.workspace-local/codex-automation-loop/delivery-runs`
- producer / consumer 硬依赖：
  - direct send 必须消费真实 target thread id。
  - target skill 必须消费 deliveryId / currentWindow / taskId / dispatchGroup。
  - controller skill 必须消费 controller return prompt。
- 不能切换 / 不能删除 / 不能提前消费的边界：
  - 不能删除 thread registry 校验。
  - 不能把 direct send 成功当作任务完成。
  - 不能把 heartbeat fallback 删除到无法无人值守兜底。
  - 不能让 target window 决定下一波。
- 待总控补证的问题：
  - direct delivery adapter 的真实入口、参数、权限、错误输出、UI refresh 要求和 fallback policy。

## 设计选项

### 选项 A：Direct Thread Dispatch First，Heartbeat Fallback

- 描述：
  - `build-delivery` 默认生成 `transport.kind = "codex-direct-thread"`。
  - delivery adapter 直接向目标 thread 投递 prompt。
  - direct 不可用或失败时，根据策略显式 fallback heartbeat。
- 优点：
  - 满足用户“把 1 分钟投递换成直接线程投递”的目标。
  - 保留当前 dispatch/result 协议。
  - 迁移风险可控。
  - 不让 direct 能力不可用时彻底断链。
- 风险：
  - direct send API 已在当前 Codex App host tool 面验证可用，但仍需把它落到 `codex-control-workspace` delivery adapter。
  - fallback 策略需要避免悄悄回到 heartbeat default。

### 选项 B：完全删除 Heartbeat，只保留 Direct

- 描述：
  - 删除 heartbeat payload，direct send 不可用直接失败。
- 优点：
  - 最干净，最符合“换成直接线程投递”的表面目标。
- 风险：
  - 如果当前 Codex direct API 未暴露或 UI 不刷新，整个无人值守闭环立即不可用。
  - 失去 keep-live / schedule 场景能力。
  - 对现有 tests / docs / fallback 不友好。

### 选项 C：继续 Heartbeat，仅缩短 prompt / 文档

- 描述：
  - 不切 transport，只优化 prompt 和 cleanup。
- 优点：
  - 实现成本低。
- 风险：
  - 不满足用户当前明确目标。
  - 仍有 1 分钟延迟和 automation residue。

## 推荐方案

推荐选项 A：`direct-thread` 作为默认，`heartbeat` 退为显式 fallback / keep-live。

推荐理由：

- 它尊重用户明确目标：普通自动化跳转不再等 1 分钟 heartbeat。
- 它不破坏已经收敛好的 dispatch/result/controller review 协议。
- 它把已验证的 Codex host thread-send 行为落成 transport contract，而不是盲目删除现有兜底能力。
- 它延续 VAD 优化的原则：VAD / delivery 只负责投递，不复制总控派发逻辑。

## 推荐接口形态

### DeliveryEnvelope 增量字段

建议在现有 envelope 上增加 transport 字段，而不是改名重建整个协议：

```ts
type DeliveryTransport =
  | {
      kind: "codex-direct-thread";
      targetThreadRef: "registered-thread";
      targetWindow: string;
      oneShot: true;
      fallback?: "codex-heartbeat" | "none";
    }
  | {
      kind: "codex-heartbeat";
      rrule: "FREQ=MINUTELY;INTERVAL=1";
      staggerSeconds?: number;
    };
```

`DeliveryEnvelope` 保留：

```ts
type DeliveryEnvelope = {
  kind: "DeliveryEnvelope";
  deliveryId: string;
  sourcePacketId: string;
  targetWindow: string;
  taskId: string;
  dispatchGroup?: string;
  controlPlan: string;
  prompt: string;
  returnRoute: "controller" | "none";
  oneShot: true;
  keepLive: boolean;
  correlationId: string;
  targetThread?: RedactedOrLocalThreadRef;
  transport: DeliveryTransport;
};
```

### CLI 行为建议

新增或调整参数：

```text
build-delivery --transport auto|direct-thread|heartbeat
build-controller-return --transport auto|direct-thread|heartbeat
```

默认：

```text
--transport auto
```

`auto` 语义：

1. 如果 direct thread capability available，使用 direct thread。
2. 如果 direct unavailable 且 fallback allowed，输出 heartbeat fallback payload 并明确 `fallbackReason`。
3. 如果 `--no-fallback` 或高风险场景，fail closed。

### Delivery run 记录

建议新增本地运行态：

```text
.workspace-local/codex-automation-loop/delivery-runs/<deliveryId>.json
```

字段：

```ts
type DeliveryRun = {
  deliveryId: string;
  transportKind: "codex-direct-thread" | "codex-heartbeat";
  targetWindow: string;
  targetThreadRedacted: true;
  status: "sent" | "failed" | "fallback-created";
  sentAt?: string;
  fallbackReason?: string;
  error?: string;
  uiObserved?: boolean;
  threadReadbackObserved?: boolean;
};
```

该记录不能成为任务完成证据，只能证明 delivery 动作发生。

## 用户确认记录

| 时间 | 决策 / 偏好 | 影响 |
| --- | --- | --- |
| 2026-05-31 | 要检查当前自动化无人值守模式逻辑，并把 1 分钟投递换成直接线程投递，新建需求 | 本需求进入 ready-for-workspace；默认目标是 direct thread dispatch first |

## 禁止的伪实现

- 只把文案从 heartbeat 改成 direct，但代码仍默认 `FREQ=MINUTELY;INTERVAL=1`。
- 只新增 `transport` 类型，不调用真实 Codex direct send 能力。
- 只用 UI 点击 / 粘贴模拟作为默认实现。
- direct send API 返回成功但目标 UI 不刷新，也标记为已送达。
- 把 delivery success 当作 target task completed。
- 把 heartbeat fallback 静默当作 direct 成功。
- 把 raw thread id 写入 tracked 文档、提示词、GitHub 或回填正文。

## 差距分析

| 能力 | 当前状态 | 缺口 | 归属窗口 | 风险 |
| --- | --- | --- | --- | --- |
| target delivery transport | 默认 heartbeat | 需要 direct-thread 默认与 fallback policy | AlembicWorkspace | P0 |
| controller return transport | 默认 heartbeat | 需要 direct-thread 默认 | AlembicWorkspace | P0 |
| capability probe | Design 已完成 host thread-send 基础 probe | 需要总控转成 Stage 0 baseline，并确认 delivery adapter 接入方式 / UI smoke 边界 | AlembicWorkspace | P0 |
| delivery status | 只有 envelope 文件 | 需要 delivery run 状态与 fallback reason | AlembicWorkspace | P1 |
| target/controller skill | heartbeat wakeup 文案 | 需要 direct wakeup + fallback cleanup 分层 | AlembicWorkspace | P1 |
| tests | heartbeat redaction tests | 需要 direct / fallback / no raw id / controller return tests | AlembicWorkspace | P1 |
| real smoke | 无 direct smoke | 需要真实 target + controller 线程验证，覆盖 idle 和 busy 两种目标窗口状态 | AlembicWorkspace / AlembicTest | P1 |

## TODO / Backlog

| ID | 状态 | 类型 | 严重度 / 优先级 | 归属 | 事项 / TODO | 影响目标 / 派发 | 依赖 / 触发 | 推荐窗口 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| CDTDA-P0-A | 待总控接收 | 调研 / 验证 | P0 | `codex-control-workspace` | Direct thread capability baseline：Design 已验证 App Server busy-turn steering、host `send_message_to_thread` idle 新 turn、busy 同 turn follow-up、invalid thread fail closed；总控需确认 adapter 接入方式、delivery log、真实 target/controller smoke 和 fallback policy | 是 | 接收本需求后第一步 | `AlembicWorkspace` |
| CDTDA-P0-B | 待总控接收 | 实现 | P0 | `codex-control-workspace` | `build-delivery` / `build-controller-return` 增加 transport policy，默认 direct-thread | 是 | P0-A 成功或明确 adapter path | `AlembicWorkspace` |
| CDTDA-P0-C | 待总控接收 | 实现 / 安全 | P0 | `codex-control-workspace` | 保留 thread id local-only、redaction、placeholder validation，并补 direct transport 测试 | 是 | P0-B | `AlembicWorkspace` |
| CDTDA-P1-A | 待总控接收 | 文档 / skill | P1 | `codex-control-workspace` | target/controller skill、README、AGENTS managed block 改为 direct-first / heartbeat fallback | 是 | P0-B | `AlembicWorkspace` |
| CDTDA-P1-B | 待总控接收 | 验证 | P1 | `AlembicWorkspace / AlembicTest` | 两窗口 smoke：controller -> target direct，target -> controller direct，不创建默认 heartbeat | 是 | P0-B/P1-A | `AlembicWorkspace` 或 `AlembicTest` |

## 阶段候选

1. Stage 0：Capability Probe / API 定位
   - Design 已查明当前 Codex App host tool 暴露 `create_thread` / `send_message_to_thread` / `read_thread`，可稳定向既有 projectless thread 发送 follow-up。
   - Design 已记录 idle / busy / invalid thread readback 行为；不写 raw thread id。
   - 已用 App Server 验证 busy active turn 可 `turn/steer` 并返回同一 `turnId`。
   - 总控仍需把这些证据转成 workspace Stage 0 baseline：确认 thread registry id 与 host send id 同源、delivery adapter 能消费 direct payload、是否需要真实 target/controller UI smoke。

2. Stage 1：Transport Contract
   - 给 `DeliveryEnvelope` / `ControllerReturnEnvelope` 增加 direct transport policy。
   - 默认 `auto -> direct-thread first`。
   - 保留 explicit `heartbeat` fallback。

3. Stage 2：Delivery Adapter
   - 实际执行 direct thread send。
   - 写 delivery run 状态。
   - direct failure 才 fallback，并写 `fallbackReason`。

4. Stage 3：Controller Return Direct
   - target result group ready 后，controller-return 默认 direct 投递到总控 thread。
   - 不再普通路径创建 controller-return heartbeat。

5. Stage 4：Rules / Skill / Docs Refresh
   - 将 heartbeat 默认文案改为 direct-first。
   - target cleanup 只保留在 heartbeat fallback 小节。
   - AGENTS managed block 不再把 heartbeat 当成唯一 automation entry。

6. Stage 5：Real Smoke
   - 单 target direct dispatch。
   - N target / 1 controller-return direct smoke。
   - 验证没有默认 1 分钟 automation created。

阶段候选不等于执行派发。最终阶段顺序必须由 `AlembicWorkspace` 基于 Stage 0 代码事实和本机 Codex 能力确认。

## 验证策略

- 最小验证：
  - `node --test scripts/codex-automation-loop.test.mjs`
  - 覆盖 direct transport envelope、fallback envelope、redaction、controller-return direct。
- 集成验证：
  - `node scripts/workspace-control.mjs loop ...` 路径仍能创建 dispatch / delivery / result / review。
  - delivery run 不泄漏 raw thread id。
  - fallback heartbeat 只有 direct unavailable 或显式 `--transport heartbeat` 时出现。
- 真实 Codex smoke：
  - 注册 controller 与 target thread。
  - 从 controller direct send 到 idle target，target thread 出现新 turn 并执行；若验收要求 UI 证据，再补 UI 可见观察。
  - target 正在执行时再次 direct send；当前 Design probe 已观察到同一 in-progress turn 追加 userMessage，正式实现需记录为 busy follow-up / steer-like policy。
  - target submit result。
  - controller-return direct send 到总控线程。
  - 总控收到并可 review-results。
  - 无默认 heartbeat automation 对象创建；若 fallback 出现，必须有 `fallbackReason`。
- 测试窗口交接：
  - 如总控无法自己观察两个真实窗口，可交给 `AlembicTest` 做非真实项目的 Codex thread smoke。

## 完成定义

- 默认自动化跳转不再依赖 `FREQ=MINUTELY;INTERVAL=1`。
- target fan-out 和 controller-return 都能 direct thread dispatch。
- direct transport 明确区分 idle thread、busy steerable thread、busy not-steerable thread 和 explicit interrupt。
- direct send 失败不会伪装成功；fallback 显式可见。
- heartbeat 能力保留但退为 fallback / keep-live / schedule mode。
- 总控派发层、delivery transport 层、target execution 层和 controller review 层仍清楚分开。
- 真实 smoke 证明 UI 可见、线程收到、任务可执行、总控可回跳。

## 非目标与禁止捷径

- 不重新引入旧 VAD `claim / finish / chain-next`。
- 不新增第二套总控派发逻辑。
- 不把 direct delivery 写进 Alembic 产品 runtime。
- 不用产品源码仓库承担 workspace delivery。
- 不做 UI。
- 不改变总控验收标准。
- 不把 thread id 写入 tracked 文档。

## 开放问题

1. 当前 Codex 直投能力的稳定入口到底是 host thread tool、App Server、SDK，还是 CLI debug？
   - Design 当前答案：第一版可优先用 Codex App host thread tool；App Server / SDK / CLI debug 可作为备用 direct adapter 候选。
2. direct send 成功判据采用 API success、thread readback、UI visible，还是三者都要求？
   - Design 当前建议：最小实现要求 API success + thread readback / target turn observed；UI visible 作为 smoke 证据，不宜作为每次 delivery 必需条件。
3. 目标窗口正在执行时，实际 thread-send 行为是 steer、排队、拒绝，还是打断？
   - Design 当前答案：host tool readback 表明 busy follow-up 进入同一 in-progress turn；不是默认 interrupt，也不是第二 turn。
4. fallback heartbeat 是否默认允许，还是只有 `--allow-fallback` 才启用？

Design 推荐：

- Stage 0 baseline 用三重判据：API success + thread readback + target turn started / follow-up observed；UI visible 作为可选 smoke。
- 普通自动化禁止默认 interrupt；interrupt 只能用于用户明确停止、改范围、危险行为纠偏或总控硬阻断。
- fallback 可默认允许一次，但必须输出 `fallbackReason`，不能在日志上伪装 direct success。

## Workspace 交接准备状态

准备交给 AlembicWorkspace 评审。

## 进入总控流程建议

- 建议下一步：总控接收本需求后先把 Design probe 结果作为 Stage 0 baseline 入账，再进入 transport contract；不要直接改总控派发 / result 协议。
- 是否需要补充代码调研：需要，重点是 `codex-control-workspace` delivery adapter 如何消费 direct thread payload、如何记录 delivery run、fallback heartbeat 是否显式开启。
- 是否需要进入 `AlembicWorkspace` 目标阶段确认：需要，特别是 fallback policy 和真实 smoke 边界。
- 明确不应派发的窗口：
  - 不派 Alembic / AlembicCore / AlembicAgent / AlembicDashboard 产品实现。
  - 不派 AlembicPlugin 产品任务，除非总控另行决定 Plugin 作为 smoke target window。

## 交接前自检

- 已对照 `docs/workspace-alignment-checklist.md`：是。
- 原始计划已确认：用户直接确认目标方向。
- 完整功能闭环已写清：是。
- 代码事实 / 调研缺口已分开：是。
- 阶段仍是候选，没有写成 wave：是。
- TODO / Backlog 已记录：是。
- 仍需用户确认的问题已列出：无必须等待用户的问题；Stage 0 后由总控裁决 fallback policy。
