# Codex Direct Thread Dispatch Automation 代码事实复核

Design Key：CODEX-DIRECT-THREAD-DISPATCH-2026-05-31
日期：2026-05-31
状态：code-fact-review / requirement-input
维护窗口：AlembicDesign
总控：AlembicWorkspace

## 判断类型

- `新需求`
- `research`
- `requirement-candidate`
- `automation-flow-optimization`
- `current-mainline-nonblocking`

## 用户目标

用户要求检查当前自动化无人值守模式逻辑，并基于 Codex 现在支持窗口线程直接投递提示词这一变化，重新梳理自动化跳转能力，把自动化默认的 1 分钟 heartbeat 投递换成直接线程投递，形成一个新需求。

本文只做 Design 代码事实复核，不修改 `AlembicWorkspace` 当前状态、全局 TODO 或产品源码。

## 读取范围

本轮复核了以下真实代码与规则面：

- `codex-control-workspace/scripts/codex-automation-loop.mjs`
- `codex-control-workspace/scripts/codex-automation-loop.test.mjs`
- `codex-control-workspace/scripts/workspace-control.mjs`
- `codex-control-workspace/skills/dev/codex-automation-target/SKILL.md`
- `codex-control-workspace/skills/dev/codex-automation-controller/SKILL.md`
- `codex-control-workspace/skills/dev/control-workspace-governance/references/codex-automation-loop.md`
- `codex-control-workspace/skills/dev/control-workspace-governance/references/control-architecture.md`
- `codex-control-workspace/README.md`
- `codex-control-workspace/AGENTS.md`
- `AlembicDesign/docs/current/automation-flow-current-state-research-2026-05-29.md`
- `AlembicDesign/docs/current/codex-automation-closed-loop-requirement-design-2026-05-29.md`
- `AlembicDesign/docs/current/codex-automation-closed-loop-rule-unification-2026-05-29.md`

## 当前实现结论

### 1. 现行活跃自动化协议已经从旧 VAD 转到 `codex-automation-loop.mjs`

当前 `codex-control-workspace/scripts/` 中没有 `visible-dispatch.mjs`。活跃脚本面是：

- `codex-automation-loop.mjs`
- `workspace-control.mjs loop ...` 聚合入口
- `codex-automation-loop.test.mjs`

旧 VAD 的复杂 `claim / finish / chain-next / target-courier` 已在 Design 文档中作为历史输入存在，但当前脚本主协议已经收敛为：

```text
ControllerDispatchPacket
-> DeliveryEnvelope / ControllerReturnEnvelope
-> TargetResultEnvelope
-> review-results
```

这意味着本需求不应重新设计总控派发模型，也不应把 VAD 旧逻辑搬回来。需要替换的是 `DeliveryEnvelope` 的默认传输方式。

### 2. `create-dispatch` 是总控派发包，不含传输调用

`commandCreateDispatch()` 负责生成 `ControllerDispatchPacket`：

- `targetWindow`
- `taskId`
- `dispatchGroup`
- `controlPlan`
- `objective`
- `scope`
- `forbidden`
- `evidenceRequired`
- `contextPolicy`
- `prompt`

它不选择 Codex API、不创建 heartbeat、不接受 evidence。这个边界是正确的，应保留。

### 3. `build-delivery` 当前硬编码 heartbeat transport

`commandBuildDelivery()` 会读取 dispatch packet、查本地 thread registry，然后生成 `DeliveryEnvelope`。当前关键事实：

```js
schedule: {
  kind: "heartbeat",
  rrule: "FREQ=MINUTELY;INTERVAL=1",
  staggerSeconds: stagger,
}
```

若存在 thread registration，还会生成：

```js
codexAutomation: {
  kind: "heartbeat",
  destination: "thread",
  targetThreadId: registration.threadId,
  name: `Codex automation ${packet.targetWindow}`,
  prompt: packet.prompt,
  rrule: envelope.schedule.rrule,
  status: "ACTIVE",
}
```

这正是用户指出的 1 分钟投递能力：它不是即时 thread send，而是准备 `codex_app.automation_update` 可消费的 heartbeat payload。

### 4. `build-controller-return` 同样硬编码 heartbeat transport

`commandBuildControllerReturn()` 创建 `ControllerReturnEnvelope`，同样写入：

```js
schedule: {
  kind: "heartbeat",
  rrule: "FREQ=MINUTELY;INTERVAL=1",
}
```

并在本地 controller thread registration 存在时输出 `codexAutomation.kind = "heartbeat"`。因此 target -> controller 回跳也仍是 1 分钟 heartbeat，不是直接回到总控线程。

### 5. 测试当前绑定 heartbeat payload 与 thread id redaction

`codex-automation-loop.test.mjs` 已覆盖：

- `build-delivery` 从已注册 target thread 生成 `codexAutomation.targetThreadId`。
- JSON stdout 默认 redacts thread id。
- 写入本地 ignored runtime 文件时保留真实 thread id。
- `build-controller-return --include-thread-id` 用于 immediate local `automation_update` 调用。

现有测试没有覆盖：

- direct thread send capability probe。
- direct delivery result 状态。
- UI 是否实时刷新。
- direct delivery 失败后 fallback heartbeat。

### 6. skill / README / AGENTS 文案均以 heartbeat 为默认入口

当前规则面仍明确写着：

- delivery adapter 创建 heartbeat。
- target window 收到 heartbeat 后若带 `automation_id`，先删除一次性 automation。
- controller return 创建 controller-return heartbeat。
- `Codex Automation Closed Loop lets the controller wake real Codex windows with heartbeat automations...`

因此如果本需求落地，必须同步更新：

- target skill 的 “Consume wakeup” 文案。
- controller skill 的 dispatch / return 文案。
- `codex-automation-loop.md` 的 layer / commands / prompt rules。
- `control-architecture.md` 的 automation classes。
- README 和 parent / child managed AGENTS block 中的 heartbeat 默认表述。

## Codex 线程直投能力证据

### 本地可调用工具发现

本轮先用工具发现查询 `send_message_to_thread / list_threads / read_thread / create_thread` 和 “已向对话线程发送消息 / 对话线程发送消息”。初始可见工具面曾只暴露 `multi_agent_v1` 子 agent 工具；用户已纠偏：`multi_agent_v1.spawn_agent/send_input` 不是 Codex App 真实窗口线程，不能作为窗口线程直投证据。

随后本窗口通过 `tool_search` 暴露到 Codex App 真实线程工具：

- `codex_app.create_thread`
- `codex_app.send_message_to_thread`
- `codex_app.read_thread`
- `codex_app.list_threads`

这组工具是本需求真正关心的 host thread-send surface：可以创建 Codex 背景线程、向既有线程发送 follow-up prompt、读取 thread / turn / item 状态。Design 已用该工具面完成 idle / busy / invalid thread 三类 probe。本文只记录行为证据，不写 raw thread id；真实 thread id 仍只能进入 `.workspace-local/` runtime。

Computer Use 对 `com.openai.codex` 仍被安全策略禁止，因此本文不把“鼠标点击 UI 后是否刷新”作为证据。当前证据来自 Codex App thread readback：thread 状态、turn 状态、同一 turn 内的 userMessage / agentMessage markers。

### 官方资料

官方 Codex App Server 文档和 OpenAI 工程文章显示 Codex 有线程 / turn 原语，并明确区分 start / steer / interrupt：

- `thread/start` 创建线程。
- `thread/resume` 继续已有线程。
- `turn/start` 对目标 `threadId` 增加用户输入并开始一个 turn。
- `turn/steer` 对正在执行的 active turn 追加用户输入，不创建新 turn。
- `turn/interrupt` 请求取消正在执行的 turn。
- `codex debug app-server send-message-v2 "..."` 是官方示例的直接发送消息调试入口。

官方 Codex Automations 文档仍描述 schedule / thread automation，可回到同一 conversation 继续工作，但它不是即时 direct thread send 的唯一方式。

本机 `codex app-server generate-json-schema --experimental` 也生成了当前安装版本的 v2 schema，包含：

- `TurnStartParams`：required `threadId` + `input`。
- `TurnSteerParams`：required `threadId` + `expectedTurnId` + `input`，其中 `expectedTurnId` 是 active turn id precondition。
- `TurnInterruptParams`：required `threadId` + `turnId`。
- schema 描述中还写明，当 `turn/start` 或 `turn/steer` 提交到不可 same-turn steering 的 active turn 时，会返回 `activeTurnNotSteerable`。

Design 结论：Codex 官方模型不是单一“send message to thread”。正确 transport policy 至少需要三种动作：

```text
idle thread -> turn/start
busy steerable thread -> turn/steer(expectedTurnId)
busy not steerable / unknown active turn -> fail closed or controller decision
explicit stop / urgent correction -> turn/interrupt + follow-up turn/start or steer
```

总控截图里的“已向对话线程发送消息”很可能是上层 UI / host thread-send 工具封装了上述 start / steer 路由。Design 当前已能调用 `codex_app.send_message_to_thread` 这一 host thread-send 工具，并通过 readback 观察到 idle / busy 行为；仍不能用 Computer Use 直接证明可见 UI 刷新。

### App Server busy-turn probe

2026-05-31，Design 使用 `codex app-server` 执行真实 Codex App Server thread probe：

```text
initialize
-> thread/start cwd=/private/tmp approvalPolicy=never sandbox=read-only
-> turn/start: 要求不要读写文件，运行一次 sleep 20，然后回复 PROBE_INITIAL_DONE
-> turn/steer expectedTurnId=<active turn id>: 追加 PROBE_STEER_MESSAGE，要求最终包含 PROBE_STEER_SEEN
-> turn/completed status=completed
```

关键结果：

- `thread/start` 返回真实 thread id。
- `turn/start` 返回真实 turn id，并发出 `turn/started status=inProgress`。
- 在 active turn 运行中调用 `turn/steer`，服务端立即返回同一个 `turnId`。
- 最终 agent message delta 组合为：

```text
PROBE_INITIAL_DONE PROBE_STEER_SEEN
```

结论：

- 在 App Server 协议下，busy thread 的正确“继续输入”语义是 `turn/steer(expectedTurnId)`，不是再开一个 `turn/start`。
- 该 steering 没有中断原 turn；原任务完成后同一 turn 的最终回答包含 steering 要求。
- `turn/interrupt` 才是取消 in-flight turn 的动作，普通自动化投递不应默认使用。

证据边界：

- 这证明了 Codex App Server thread/turn 协议的 busy-turn 行为。
- 这不等同于 Desktop UI 可见刷新；但后续 host thread-send probe 已验证 `send_message_to_thread` 在 busy 线程上的 readback 语义与 `turn/steer` 一致：同一 in-progress turn 内追加 userMessage，最终同一 turn 回答同时包含初始任务和 follow-up 标记。

### Host thread-send idle / busy / invalid probe

2026-05-31，Design 使用 Codex App thread tools 做了不读写文件的 projectless thread probe。为避免泄漏 raw thread id，本文只记录 marker、turn 结构和结论。

#### Idle baseline

步骤：

```text
create_thread: 要求直接回复 IDLE_BASE_DONE
read_thread: 等待线程 idle，确认首个 turn completed
send_message_to_thread: 向同一 idle thread 发送 follow-up，要求回复 IDLE_FOLLOWUP_DONE
read_thread: 确认出现新的 turn，并完成 follow-up
```

观察：

- 初始线程创建后有一个 in-progress turn，完成后 thread 状态变为 idle。
- 向 idle thread 发送 follow-up 后，readback 显示创建了新的 turn。
- follow-up turn 最终回答包含 `IDLE_FOLLOWUP_DONE`。
- 该路径没有等待 `FREQ=MINUTELY;INTERVAL=1`，也没有创建 heartbeat automation。

结论：

- idle thread 上 `send_message_to_thread` 等价于“向既有线程开新 turn 并立即处理”。
- 对自动化 delivery 来说，target thread idle 时可以把 direct follow-up 作为普通路径。

#### Busy single follow-up

步骤：

```text
create_thread: 要求执行一次 sleep 后回复 BUSY_BASE_DONE_AFTER_SLEEP
send_message_to_thread: 在 turn inProgress 期间发送 follow-up，要求最终包含 BUSY_FOLLOWUP_SEEN
read_thread: 观察同一 in-progress turn 的 items
等待完成后 read_thread
```

观察：

- busy 期间 `send_message_to_thread` 返回成功。
- readback 没有出现第二个 turn；follow-up 作为同一 in-progress turn 的第二个 `userMessage` 出现。
- 最终同一 turn 的 final answer 同时包含：

```text
BUSY_BASE_DONE_AFTER_SLEEP
BUSY_FOLLOWUP_SEEN
```

结论：

- busy thread 上 host thread-send 不是默认 interrupt，也不是创建并行新 turn。
- 当前行为等价于“向 active turn 追加 steering input”；初始任务没有被取消，follow-up 被同一 turn 消费。
- 自动化实现不能把 busy follow-up 当作独立新任务并假设立即分离执行；它应只用于同一任务补充 / controller-return / 允许被追加的唤醒语义，或在不适合追加时等待 idle / fail closed。

#### Busy multiple follow-ups

步骤：

```text
create_thread: sleep 后回复 MULTI_BUSY_BASE_DONE
send_message_to_thread: busy 时发送 MULTI_BUSY_A_SEEN
send_message_to_thread: busy 时继续发送 MULTI_BUSY_B_SEEN
read_thread: 观察同一 in-progress turn
等待完成后 read_thread
```

观察：

- A / B 两条 busy follow-up 均作为同一 in-progress turn 的后续 `userMessage` 出现。
- 最终同一 turn 的 final answer 包含：

```text
MULTI_BUSY_BASE_DONE
MULTI_BUSY_A_SEEN
MULTI_BUSY_B_SEEN
```

结论：

- 多次 busy follow-up 会继续进入同一 active turn；这对“追加上下文 / 回跳提醒”是可用的。
- 对“无限跳转”来说，delivery adapter 必须避免向同一 busy target 塞入多个彼此独立的任务包；需要 deliveryId / correlationId 去重和同窗口串行策略。

#### Invalid thread id

步骤：

```text
send_message_to_thread: 使用明显无效 thread id
```

观察：

```text
No AppServerManager registered for conversationId: ...
```

结论：

- 无效 thread id 会 fail closed，不会静默成功。
- delivery adapter 应把这种错误记录为 delivery failure，并按显式 fallback policy 决定是否生成 heartbeat fallback；不能把 direct send failure 伪装成已投递。

参考：

- [Codex App Server](https://developers.openai.com/codex/app-server)
- [Unlocking the Codex harness: how we built the App Server](https://openai.com/index/unlocking-the-codex-harness/)
- [Codex Automations](https://openai.com/academy/codex-automations)

## 真实问题

当前无人值守闭环的问题不是 dispatch/result 协议过重，而是 transport 仍以 heartbeat automation 为默认：

```text
dispatch packet
-> build-delivery
-> heartbeat rrule = FREQ=MINUTELY;INTERVAL=1
-> automation_update 创建定时 heartbeat
-> target 收到后删除 one-shot automation
```

这带来几个副作用：

- 最快也要等下一次 heartbeat tick，不能像人工 UI 投递一样立即出现。
- 每次投递都会创建 automation 对象，目标窗口需要清理 `automation_id`。
- controller-return 也要等 heartbeat，拉长 N -> 1 总控回跳。
- prompt / skill 文案把 “heartbeat” 当成正常醒来语义，和现在的 direct thread 能力不匹配。
- 自动化失败排查会混入 schedule、automation residue、one-shot cleanup，而不是聚焦“消息是否已经送达目标线程”。

## 推荐代码边界

本需求应保留：

- `ControllerDispatchPacket`
- `DeliveryEnvelope`
- `ControllerReturnEnvelope`
- `TargetResultEnvelope`
- `review-results`
- thread id local-only / redaction
- total-control pull review

本需求应替换：

- `build-delivery` 默认 transport。
- `build-controller-return` 默认 transport。
- skills / README / AGENTS 的 heartbeat 默认表述。
- tests 中把 `codexAutomation` 作为默认 payload 的断言。

本需求不应修改：

- 总控选择窗口、解析 TODO、派发任务包的逻辑。
- 子窗口执行边界。
- 总控验收标准。
- 产品仓库代码。
- 多 LLM / 多 IDE 通用调度抽象。

## 建议总控下一步

接收 `CODEX-DIRECT-THREAD-DISPATCH-2026-05-31` 后，先把本文 probe 作为 Stage 0 baseline 入账，而不是直接改总控派发协议：

1. 确认 `codex-control-workspace` delivery adapter 如何消费 `send_message_to_thread`：由 Codex host tool 执行、App Server / SDK / CLI debug 执行，还是只输出 direct payload 交给 operator。
2. 确认发送到目标线程与当前 thread registry 的真实 thread id 能稳定对应；raw thread id 仍只保存在 `.workspace-local/`。
3. 正式记录 busy policy：idle follow-up 创建新 turn；busy follow-up 追加到同一 in-progress turn；普通自动化不得默认 interrupt。
4. 定义 delivery run 日志：sent / failed / fallback-created、fallbackReason、thread readback observed，不把 delivery success 当成 target completed。
5. 确认失败时是否能给出可诊断错误，并按显式策略选择 heartbeat fallback。
6. 在实现前或实现后补一个真实 target/controller smoke：target fan-out direct、target result、controller-return direct，不创建默认 heartbeat automation。

Stage 0 完成前，不建议删除 heartbeat 能力；它应先退为 fallback / keep-live support。
