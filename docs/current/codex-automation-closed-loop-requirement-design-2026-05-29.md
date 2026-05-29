# Codex Automation Closed Loop 需求设计

Design Key：CODEX-AUTOMATION-CLOSED-LOOP-2026-05-29
日期：2026-05-29
状态：ready-for-workspace
维护窗口：AlembicDesign
总控：AlembicWorkspace
原始计划：[codex-automation-closed-loop-original-plan-2026-05-29.md](codex-automation-closed-loop-original-plan-2026-05-29.md)
规则统一补充：[codex-automation-closed-loop-rule-unification-2026-05-29.md](codex-automation-closed-loop-rule-unification-2026-05-29.md)

## 定位

本需求是 Codex 内部自动化闭环优化，不是新增产品功能，也不是替换总控。

它要把当前 VAD / automation prompt / 子窗口回填 / 总控验收收敛为一条精简闭环：

```text
总控 1 -> N 子窗口：target-specific dispatch
子窗口 N -> 1 总控：compact result envelope
总控 1 -> 1 裁决：pull evidence + next decision
```

`AlembicAgent 完成了` 只是人工最小示范，说明总控可以用少量信号触发推理和证据拉取。完整自动化不是一句话，而是“总控已知上下文下的精简 target-specific prompt + 子窗口结构化结果 + 总控聚合验收”。

## 最终目标

建立 `CodexAutomationClosedLoop`：

- 总控按 current plan / task package 生成每个目标窗口的精简 dispatch packet。
- 自动化 prompt 针对目标窗口和任务，不再照搬人工通用复制提示词。
- VAD 与总控派发拆开：总控决定目标窗口、提示词、证据和验收；VAD 只负责把 prompt 投递到目标窗口，必要时 keep live，并按 return route 回跳总控。
- 子窗口只处理自己的目标任务，输出 compact result envelope。
- 总控聚合 N 个子窗口结果，并主动 pull commit / diff / command output / runtime JSON / reports / screenshots。
- 总控独立裁决 `accepted / needs-rework / blocked / next-wave / stop`。
- 手动复制提示词继续保持通用、稳妥、自包含；automation prompt 走精简目标化路径。

## 已确认决策

- VAD 核心接口收敛为“给我目标窗口、提示词、回跳策略，我负责投递”。
- keep live 是必要投递保障，但不是调度逻辑；它只保证投递期间机器和会话可继续工作。
- 总控知道目标窗口和任务内容，因此 target-specific prompt 由总控派发层生成。
- 子窗口回调总控应保持简单，只需带 delivery / correlation 信息唤醒总控；任务证据和验收由总控自行 pull review。
- VAD 普通路径轻量执行，不默认跑 full preflight / audit / verify，不在 VAD 内部复刻 current plan / TODO 派发。
- `用完即弃`、payload 轻量匹配、thread local-only、mode gate 和 skill ref 属于快递信封基础能力；它们常驻但不等于任务完成。

## 当前问题

### 人工提示词与自动化提示词混用

人工提示词为了方便复制粘贴，必须通用、自包含、能唤醒任意目标窗口，所以会重复：

- 读取 AGENTS。
- 读取 workspace index / current plan。
- 声明窗口定位。
- 回填格式。
- 禁止事项。

这对人工路径是合理的，因为用户可能把同一段发给不同窗口。

但 automation 中，总控已经知道目标窗口、任务、控制文档、task id、dispatch group 和 evidence target。如果仍然复制人工提示词，就会产生大量冗余，并让子窗口把精力花在重复定位，而不是执行任务和回填证据。

### 子窗口回总控过度结构化

当前 VAD 路径要求子窗口理解 `claim / finish / chain-next / record-stop / controller-return / target-courier`。这些对真实无人值守 smoke 和多窗口批次有价值，但对普通 Codex 内部自动化任务太重。

### 子窗口承担了过多“下一跳”责任

普通闭环下，子窗口只应完成任务并回总控；下一波派发、是否验收、是否返工、是否需要测试，都应由总控决定。`target-courier` 应该是显式 opt-in，而不是普通闭环默认。

## 精简闭环模型

### 1. Controller Dispatch Packet

总控派发层生成面向单个目标窗口的 packet。它属于总控派发逻辑，不属于 VAD 投递逻辑：

```ts
type ControllerDispatchPacket = {
  targetWindow: string;
  taskId: string;
  dispatchGroup?: string;
  controlDoc: string;
  taskPackageAnchor?: string;
  objective: string;
  allowedActions: string[];
  forbiddenActions: string[];
  evidenceRequired: string[];
  backfillTarget: string;
  stopConditions: string[];
  contextPolicy: "assumed-current" | "refresh-if-missing" | "force-refresh";
  prompt: string;
};
```

关键变化：

- 目标窗口不需要从通用提示词里猜自己是谁。
- 目标任务不需要从整份 current plan 里搜索。
- prompt 只携带当前任务所需的上下文和证据入口。
- `contextPolicy` 决定是否需要刷新 AGENTS / current plan，而不是每次都强制重复。

### 1.5 VAD Delivery Envelope

VAD 接收的是总控已经写好的投递信封：

```ts
type VadDeliveryEnvelope = {
  deliveryId: string;
  targetWindow: string;
  prompt: string;
  returnRoute: "controller" | "none";
  oneShot: true;
  keepLive?: boolean;
  controllerCorrelationId?: string;
};
```

VAD 不解析 `objective / forbiddenActions / evidenceRequired`，不从 current plan 解析派发表，不生成 target-specific prompt。它只根据 `targetWindow` 找到目标 thread，把 `prompt` 送过去，并按 `returnRoute` 唤醒总控。

`keepLive` 只表示投递期间需要保活，不改变任务目标、窗口选择或回跳策略。

### 2. Target-Specific Prompt

automation prompt 应当短、具体、面向目标：

```text
继续当前窗口任务：AlembicAgent / PCVM-W6B-AGENT-STAGE-NODE-IDENTITY。

目标：实现 canonical stage node identity passthrough。
任务来源：<controlDoc>#<taskPackageAnchor>
边界：只改 AlembicAgent；不跑 full cold-start；不派 AlembicTest。
证据：提交 hash、targeted test 命令和结果、关键 diff / 测试断言、遗留风险。
完成后返回 Result Envelope；总控会 pull evidence 并验收。
```

它不需要复制人工提示词里的完整通用说明。若上下文缺失或目标窗口不确定，才按 `contextPolicy` 刷新：

- `assumed-current`：已由 automation runtime 确认目标窗口和 task id；只需读取任务锚点。
- `refresh-if-missing`：如果本窗口无法确认任务 / 仓库边界，再读取 AGENTS 和 current plan。
- `force-refresh`：风险任务、首次登记、规则变更、跨仓库边界、删除 / 迁移 / 发布任务，强制读取 AGENTS / current plan。

说明：这不是废除 AGENTS 规则，而是把“每条 automation prompt 都长篇强制读 AGENTS”改为可验证的上下文策略。总控若要正式实施，需要同步治理当前 AGENTS / VAD skill 的提示词硬规则。

### 3. Target Execution

目标窗口执行原则：

- 只处理 `targetWindow` 与本窗口一致的任务。
- 只处理一个明确 `taskId`。
- 不领取其它窗口任务。
- 不决定下一波。
- 不创建普通下一跳。
- 需要阻塞时返回 blocked result envelope。

目标窗口可以用本窗口内部推理能力判断如何执行任务；不需要把总控全部规则复制进 prompt。

### 4. Compact Result Envelope

子窗口返回给总控的不是一句“完成了”，而是最小可验收结果包：

```ts
type TargetResultEnvelope = {
  targetWindow: string;
  taskId: string;
  dispatchGroup?: string;
  status: "completed" | "blocked" | "needs-review";
  changedRepos: string[];
  commits?: string[];
  evidenceRefs: string[];
  verificationSummary: string[];
  riskSummary: string[];
  nextSuggestion?: string;
};
```

自然语言可以很短，但必须能映射到这个 envelope：

```text
AlembicAgent / PCVM-W6B-AGENT-STAGE-NODE-IDENTITY completed.
commit: <hash>
tests: <command> PASS
evidence: <backfill path or summary>
risk: no full cold-start
```

截图里的 `AlembicAgent 完成了` 是最小人工示范；在自动化里应升级为 “窗口 + taskId + status + evidence refs”。

### 5. Controller Aggregation

总控知道本轮是：

- 单窗口任务：收到一个 result envelope 后进入 pull review。
- 多窗口 group：等待 N 个 result envelope，或任何一个 blocked 触发 review。
- 上下游依赖：producer result accepted 后再 dispatch consumer。

默认聚合策略：

```text
controller-last
```

即所有目标回到总控，由总控决定下一步。

### 6. Controller Pull Review

总控不信任 result envelope 本身为完成事实，只把它当证据索引：

- 读 commit。
- 看 diff。
- 跑或复核验证命令。
- 读 runtime JSON / report / screenshot。
- 对照 current plan 完成定义。
- 决定 accepted / needs-rework / blocked / next-wave / stop。

## 保留内容

必须保留：

- 总控最终裁决权。
- target window / task id / dispatch group。
- role guard：子窗口不能跨窗口。
- evidence refs：commit / diff / command / report / runtime JSON。
- local-only thread registry。
- mode enabled / disabled。
- controller-last 作为普通批次默认。
- manual copy prompt 作为人工 fallback。

## 抛弃 / 降级内容

建议从普通自动化闭环中抛弃或降级：

- 每条 automation prompt 都复制完整人工通用提示词。
- 每条 automation prompt 都强制长篇读取 AGENTS / workspace index / current status。
- 子窗口默认创建 target-courier 下一跳。
- 子窗口默认理解复杂 controller-return prompt。
- 把 `record-stop` 等一次性 wakeup 清理细节暴露为任务主语义。
- 把 `agentNext` 当作总控裁决。
- 把一句“完成了”当作完整完成事实。

## 手动路径与自动化路径分层

| 路径 | 目标 | Prompt 形态 | 上下文策略 |
| --- | --- | --- | --- |
| 手动复制 | 用户方便复制到任意窗口。 | 通用、自包含、稳妥、较长。 | 默认强制读取 AGENTS / current plan。 |
| Codex automation | 总控已知目标和任务，精准投递。 | target-specific、短、只含任务锚点和证据要求。 | `contextPolicy` 控制是否刷新。 |
| 高风险 automation | 删除、迁移、发布、跨仓库边界。 | target-specific + 明确刷新规则。 | `force-refresh`。 |
| 多 IDE / 多 LLM | 不同 host 能力不一。 | host-agnostic task envelope。 | 由 delivery channel 决定。 |

## 与现有 VAD 的关系

本需求不要求删除 VAD，而是重排默认路径：

- 普通 Codex 自动化：`controller dispatch -> target result -> controller review`。
- 多窗口无人值守：保留 dispatch group，但默认 `controller-last`。
- target-courier：仅用于显式 smoke / 特殊链式任务。
- VAD runtime：继续承担 queue / registry / group / mode / automation run 记录。
- prompt：由总控派发层生成 target-specific prompt；VAD 只附加极薄 delivery envelope。
- VAD / 总控派发拆层：target-specific prompt 由总控派发层生成；VAD 只做 delivery envelope，不再做一遍 send-eligible / current plan / TODO 派发逻辑。
- happy path：配置完成后的普通 VAD 只走 `controller builds prompt -> VAD delivers -> target executes -> controller return -> controller review`，重量级 preflight / audit / full verify 只作为失败诊断或高风险任务能力。
- keep live：只作为投递可靠性保障，不进入总控派发决策。
- 快递信封基础能力：`用完即弃`、payload/claim 轻量匹配、mode disabled 停止续投递、thread id local-only、skill ref 是每次投递的基础语义，不属于重量诊断；但它们也不等于任务完成或总控裁决。

## 与前置 Design 的关系

- `AUTOMATION-FLOW-STATE-2026-05-29`：现状研究输入。
- `CHILD-WINDOW-COMPLETION-SIGNAL-2026-05-29`：降级为本需求的一个子场景，不再单独作为首要实现需求。
- `MULTI-LLM-DISPATCH-OPTIMIZATION-2026-05-29`：后续可扩展为 host-agnostic task envelope；本需求先聚焦 Codex 内部闭环。
- `codex-automation-closed-loop-rule-unification-2026-05-29.md`：本需求的规则治理补充，专门处理 VAD 冗余、manual / automation prompt 分层、`contextPolicy`、子仓库 `automationProfile` 和 managed AGENTS 接入卡拆分。

## 分阶段候选

| 阶段 | 目标 | 完成条件 |
| --- | --- | --- |
| Stage 0 | 总控代码事实复核 | 查清现有 VAD prompt 生成、target skill、controller skill、current plan prompt 和 runtime 状态。 |
| Stage 1 | 定义 dispatch packet / result envelope | 明确字段、最小 backfill、controller aggregation 规则。 |
| Stage 2 | prompt 分层 | 手动通用提示词保留；automation prompt 改为 target-specific；加入 `contextPolicy`。 |
| Stage 3 | controller-last 默认化 | 普通自动化不让子窗口 target-courier；下一步由总控决定。 |
| Stage 4 | runtime status 简化 | 清楚区分 wakeup cleanup、task result、controller review 和 manual sendability。 |
| Stage 5 | targeted smoke | 用一个真实 Codex 单窗口任务验证 target-specific prompt + result envelope；再用 1*N / N*1 group 验证。 |

## 完成定义

- 人工提示词和 automation prompt 分层明确。
- automation prompt 可针对目标窗口和任务生成，不再复制通用长提示。
- VAD 不再承载总控派发逻辑：不解析 current plan / TODO，不判断 `待启动`，不生成 target-specific prompt，只投递总控给出的 prompt 并处理回跳。
- 配置完成后的 VAD 普通工作流保持简单快捷，不默认运行全量 preflight / audit / verify，也不把复杂诊断变成每次 heartbeat 的分叉。
- `用完即弃` 被定义为快递信封基础能力，只表示 heartbeat 已消费；不会被误当成 result envelope、任务完成或验收事实。
- 子窗口返回 compact result envelope，不只是一句完成，也不承担总控决策。
- 总控能聚合单窗口 / 多窗口结果并 pull evidence 验收。
- 普通路径默认 controller-last。
- target-courier 变为显式 opt-in。
- AGENTS / current plan 刷新由 `contextPolicy` 控制；高风险任务仍可强制刷新。
- 现有 VAD 安全边界不被削弱：thread id local-only、role guard、mode gate、evidence review 仍保留。

## 总控接收建议

Design 建议总控接收为新的 automation flow optimization 主需求。接收后先做 Stage 0，不直接改脚本：

- 读取 `visible-dispatch.mjs` 的 prompt 生成与 finish-chain。
- 读取 VAD target / controller skill。
- 对比当前人工可复制提示词和 automation payload。
- 识别哪些 AGENTS / current plan 读取要求是硬规则，哪些只是 prompt 冗余。
- 识别哪些逻辑属于总控派发层，哪些才属于 VAD 投递层；尤其确认 VAD 不再复刻 current plan / TODO 派发。
- 识别哪些检查属于 happy path 必需，哪些只能作为失败诊断 / Agent 可调用能力。
- 识别哪些是“快递信封基础能力”，例如 `用完即弃` 和 payload/claim 轻量匹配，必须常驻但不能扩大为重流程。
- 输出一份 rule governance 判断：将总控派发层、VAD 投递层、快递信封基础能力、轻量 happy path 和重量诊断能力分开。

## 总控接手后裁决项

- 普通路径默认使用 `refresh-if-missing` 还是允许部分低风险场景使用 `assumed-current`。
- 哪些任务必须 `force-refresh` 读取 AGENTS / current plan。
- `target-courier` 作为显式 opt-in 的具体允许场景。
- `CHILD-WINDOW-COMPLETION-SIGNAL-2026-05-29` 作为本需求 result signal 子场景吸收后的账本处理方式。
- runtime 是否需要新增 `manualSendableButAutomationClean` 这类状态，避免 `post-run-audit` 对手动主线产生误读。
