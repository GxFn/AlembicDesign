# Codex Direct Thread Dispatch Automation 原始计划书

Design Key：CODEX-DIRECT-THREAD-DISPATCH-2026-05-31
日期：2026-05-31
状态：已按用户方向确认 / requirement-design-ready
维护窗口：AlembicDesign
总控：AlembicWorkspace

## 用户目标

重新梳理当前 Codex 自动化无人值守模式的跳转能力。由于 Codex 现在支持窗口线程直接投递提示词，希望把当前默认的 1 分钟 heartbeat 投递，替换为直接线程投递，并将这个变化作为独立新需求交给总控接收。

## 用户原话 / 关键约束

```text
检查现在的自动化无人值守模式逻辑，因为 codex 现在支持了窗口线程直接投递提示词，重新梳理自动化跳转能力，把自动化 1 分钟投递换成直接线程投递，新建需求
```

## 为什么现在做

当前 `CODEX-AUTOMATION-CLOSED-LOOP-2026-05-29` 已经把总控派发、delivery envelope、target result envelope、controller review 的协议边界收敛得比较清楚。

但真实代码仍把 delivery transport 绑定在 Codex heartbeat automation 上：

- target fan-out 默认 `FREQ=MINUTELY;INTERVAL=1`。
- controller return 默认 `FREQ=MINUTELY;INTERVAL=1`。
- target/controller skill 默认以 heartbeat wakeup 为入口。
- target 需要清理 one-shot automation。

如果 Codex 已经能直接向目标窗口线程投递提示词，继续把 heartbeat 作为普通路径，会让闭环多等一分钟、产生多余 automation 对象，并增加 residue / cleanup / 误判复杂度。

## 期望结果

需求完成后，`AlembicWorkspace` 自动化闭环的默认跳转方式应变为：

```text
总控生成 dispatch packet
-> build delivery envelope
-> delivery adapter 直接向目标 Codex thread 投递 prompt
-> 目标窗口 UI 立即出现新 turn 并执行
-> target result envelope
-> controller return 也优先直接投递到总控 thread
-> 总控 pull review
```

heartbeat 不再是普通路径默认投递方式，只保留为：

- direct thread dispatch 不可用时的 fallback。
- keep-live / 定时检查等确实需要 schedule 的能力。
- 迁移期诊断或手动指定 transport。

## 最终完成定义草案

完成后应满足：

- `build-delivery` 的默认 transport 不再生成 1 分钟 heartbeat payload，而是生成 / 触发 direct thread dispatch 需要的 transport plan 或发送动作。
- `build-controller-return` 的默认 transport 同样走 direct thread dispatch。
- direct thread dispatch 能使用现有本地 thread registry，真实 thread id 仍只保存在 `.workspace-local/`。
- direct send 成功后，目标窗口 UI 能及时刷新并启动处理，不需要等待 1 分钟 automation tick。
- direct send 失败时，系统能明确记录失败原因；若启用 fallback，才创建 heartbeat payload。
- target / controller skill 不再把 heartbeat cleanup 当成普通路径第一步；cleanup 只在 heartbeat fallback 场景出现。
- README / AGENTS / reference 文档把“direct thread dispatch first, heartbeat fallback”写清。
- 有真实两窗口 smoke：controller -> target direct prompt、target -> controller direct return，不创建默认 heartbeat automation。

## 当前主线关系

`TODO 候选 / automation-flow-optimization`。

本需求不打断当前 038 / 039 / Plugin 主线；它属于 workspace 自动化闭环能力优化，可由总控在当前主线合适空档接收。它与 `CODEX-AUTOMATION-CLOSED-LOOP-2026-05-29` 相关，但不替换该需求：旧需求解决闭环协议与职责分层，本需求只替换默认 delivery transport。

## 初始范围

包含：

- 现有 unattended automation / closed loop 代码事实复核。
- direct thread dispatch transport 设计。
- target fan-out 的直接线程投递。
- controller-return 的直接线程投递。
- heartbeat fallback / keep-live 的降级定位。
- thread registry / raw thread id local-only 边界。
- skills / README / AGENTS / reference 的默认路径措辞更新。
- targeted unit tests 与真实 Codex 两窗口 smoke 验证设计。

不包含：

- 重新设计总控 TODO / current plan / task package 派发逻辑。
- 恢复旧 VAD `claim / finish / chain-next` 协议。
- 新增第二套总控调度器。
- 产品仓库功能实现。
- 多 LLM / 多 IDE 通用 envelope 抽象。
- 用 UI 点击模拟替代正式 thread dispatch API，除非总控 Stage 0 证明没有更稳定接口且用户另行确认。

## 初步仓库影响

| 仓库 / 窗口 | 初步判断 | 理由 |
| --- | --- | --- |
| Alembic | 无任务 / 观察 | 产品 runtime 不参与 workspace delivery transport。 |
| AlembicCore | 无任务 / 观察 | 不涉及 Alembic product core。 |
| AlembicAgent | 无任务 / 观察 | 不涉及 Agent runtime。 |
| AlembicDashboard | 无任务 | 不涉及 UI。 |
| AlembicPlugin | 无任务 / 观察 | 不涉及 Plugin MCP；除非后续总控决定把 direct thread ability 封装为 Plugin tool。 |
| AlembicTest | 可能参与验证 | 真实 Codex 两窗口 smoke 可能需要独立测试窗口，但总控可先自测 capability probe。 |
| AlembicDesign | 需求设计 | 负责新需求文档、代码事实复核和 handoff board。 |
| AlembicWorkspace / codex-control-workspace | 主要参与 | 脚本、skill、README、AGENTS managed block 和本地 runtime 都在总控工作区。 |

## 已知事实

- `codex-automation-loop.mjs` 当前只管理 contract，不调用 Codex automation API。
- `build-delivery` 与 `build-controller-return` 当前都硬编码 `schedule.kind = heartbeat` 与 `rrule = FREQ=MINUTELY;INTERVAL=1`。
- 当前 tests 断言 `codexAutomation.targetThreadId` 与 redaction。
- 当前 skill / README / AGENTS 都把 heartbeat 写成默认自动化 wakeup。
- 本窗口早期工具发现阶段没有暴露 `send_message_to_thread` 等 Codex App 真实窗口线程直投工具；当时只暴露了 `multi_agent_v1` 子 agent 工具，不能替代真实窗口线程证据。
- 官方 Codex App Server 文档显示 Codex 有 `thread/start`、`thread/resume`、`turn/start`、`turn/steer`、`turn/interrupt`；官方工程文章也展示 `codex debug app-server send-message-v2`。
- 本机 Codex CLI 生成的当前版本 App Server JSON Schema 也包含 `TurnStartParams`、`TurnSteerParams`、`TurnInterruptParams`，其中 `turn/steer` 需要 `expectedTurnId` 作为 active turn precondition。
- Design 已通过提权运行 `codex app-server` 创建真实 thread / turn，并在 active turn 中调用 `turn/steer`；服务端返回同一 `turnId`，最终回答包含初始任务完成标记和 steering 标记，说明 App Server busy-turn 路径是 steer 而非 interrupt。
- 后续本窗口通过 `tool_search` 暴露到 Codex App 真实线程工具 `create_thread` / `send_message_to_thread` / `read_thread`，并完成 projectless thread probe：idle follow-up 创建新 turn；busy follow-up 追加到同一 in-progress turn；连续 busy follow-up 也进入同一 turn；无效 thread id fail closed。原始 thread id 不写入本文档。

## 待补代码事实

- `codex-control-workspace` delivery adapter 如何消费 `send_message_to_thread` 这类 host thread-send capability，或是否需要 App Server / SDK / CLI debug 作为备选 adapter。
- 总控真实 target / controller 窗口是否需要 UI 可见 smoke；Design 当前证据来自 thread readback，不来自 Computer Use UI 观察。
- 直接投递到既有线程后，idle thread 已验证会自动开始新 turn；busy thread 已验证会追加到 active turn。仍需总控确认该语义如何进入正式 transport policy 和日志。
- 已注册 thread id 是否能被 direct dispatch API 直接使用，还是需要 session id / window id / app-server connection id。
- direct send 和 Codex automations 是否共享同一 thread id 命名空间。
- direct send 失败时的错误形态、是否能安全 fallback。

## 外部调研初判

- 是否需要联网：已做轻量官方资料核对。
- 理由：Codex thread direct delivery 是新近产品能力，必须用官方资料约束设计，不能只凭记忆。
- 参考来源：
  - [Codex App Server](https://developers.openai.com/codex/app-server)
  - [Unlocking the Codex harness: how we built the App Server](https://openai.com/index/unlocking-the-codex-harness/)
  - [Codex Automations](https://openai.com/academy/codex-automations)

## 风险 / 设计分叉

- direct thread dispatch 可能只在 App Server / SDK 可用，不一定作为当前 Codex app tool 暴露给本窗口。
- 对 busy thread 不能直接假设“新 turn 立即执行”。官方模型要求 `turn/steer` 追加到 active turn，`turn/interrupt` 才是取消当前 turn。
- direct send 可能能写入线程历史，但不保证 UI 实时刷新或自动启动 turn；Stage 0 必须验证 idle / busy / not-steerable 三种状态。
- direct send 若绕过 Codex Desktop 的 automation review / lifecycle，可能缺少 one-shot cleanup，但也需要新的 delivery log。
- 若直接删除 heartbeat fallback，可能让无人值守在能力不可用或 UI 不刷新时完全断链。
- 若把 direct thread dispatch 写成新的总控派发层，会重复 VAD 冗余问题；transport 必须保持在 delivery adapter 层。

## 开放问题

1. 总控 Stage 0 是否能在当前 Codex 版本发现稳定 thread send API？
2. direct send 使用的目标 id 是 thread id、session id、window id，还是 app-server live handle？
3. 目标窗口正在执行时，总控 Desktop host thread-send 工具是调用 `turn/steer`、排队、返回 busy / not-steerable，还是 interrupt？
4. direct send 成功判据是 API 返回成功、thread/read 出现 turn / steering item，还是目标窗口 UI 可见并继续处理？
5. heartbeat fallback 默认是否允许自动启用，还是必须由总控显式 `--transport heartbeat`？

## 需要确认

用户已明确目标：默认从 1 分钟 heartbeat 投递切换为直接线程投递。

Design 建议不再额外等待用户确认，交给总控接收后先做 Stage 0 capability probe；如果 Stage 0 发现 direct thread dispatch 在当前 Codex 环境不可用或不能刷新 UI，再回总控裁决是否临时保留 heartbeat default。
