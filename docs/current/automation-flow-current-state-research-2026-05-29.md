# Automation Flow Current State Research

Design Key：AUTOMATION-FLOW-STATE-2026-05-29
日期：2026-05-29
状态：research / requirement-candidate
维护窗口：AlembicDesign
总控：AlembicWorkspace

## 判断类型

- `research`
- `requirement-candidate`
- `automation-flow-optimization`
- `current-mainline-nonblocking`

## 用户目标

用户指出当前自动化优化不应只看 `CHILD-WINDOW-COMPLETION-SIGNAL-2026-05-29` 这一个点，而应该先把现在自动化的实现和真实使用情况整体梳理出来，再判断后续该优化哪些环节。

本文只做现状盘点和优化候选，不直接替总控入账 TODO，不派发实现窗口，不修改 workspace 当前状态。

## 2026-05-29 用户纠偏

用户进一步明确：本次主要优化的是 Codex 内部闭环，不是一句简单的 `AlembicAgent 完成了`。

截图只是人工最小示范，说明 Agent 具备推理能力，不需要在 automation prompt 中过度提示。真实闭环是：

```text
总控 -> 1 * N 子窗口
子窗口 -> N * 1 总控
```

人工提示词保持通用，是为了方便用户复制粘贴；自动化里总控已经知道给谁发、发什么内容，因此应使用 target-specific prompt，不应强行把人工通用提示词和每跳强制 AGENTS 长读照搬进自动化。

该纠偏已拆为主需求：`CODEX-AUTOMATION-CLOSED-LOOP-2026-05-29`。

## 范围

纳入：

- 总控手动提示词派发。
- Visible Automation Dispatch（VAD）本地运行态、脚本、skill 和文档。
- Codex thread heartbeat automation 作为可见窗口投递通道。
- 子窗口 claim / finish / backfill / controller-return。
- 总控 controller-tick / pull review / accept / stop / post-run-audit。
- 多 LLM / 多 IDE dispatch 需求草案与 child completion signal 草案。

不纳入：

- Alembic 产品 runtime 实现。
- Dashboard UI。
- Lark Remote 接管能力。
- 真实项目测试。
- 新自动化实现方案。

## 当前自动化分层

### 1. 手动总控派发

这是当前最稳定、最常用的人类可控路径：

```text
总控 current plan
-> 发送给 / 可复制提示词
-> 用户或总控把提示词发给目标窗口
-> 子窗口读取 AGENTS / current plan / 目标仓库 AGENTS
-> 子窗口执行任务并回填 evidence
-> 总控读取提交、diff、命令输出、报告、日志后验收
```

特点：

- 可见、简单、可解释。
- 不依赖 thread registry 或 automation runtime。
- 调度成本高，提示词重复多。
- 子窗口回总控通常靠自然语言或文档回填。

### 2. VAD target fan-out

VAD 的原始目标是替代“用户复制提示词到多个 Codex 窗口”的人工搬运，同时保持 Mac Codex 可见窗口。

核心链路：

```text
current plan send-eligible rows
-> visible-dispatch start-plan / resume-plan
-> dispatch queue + dispatch group
-> arm / arm-batch 生成 heartbeat payload
-> Codex automation_update 创建 heartbeat
-> record-arm 记录 automation id
-> 目标窗口收到可见 heartbeat
-> 删除一次性 wakeup + record-stop
-> claim 自己窗口任务
-> 执行任务
-> finish 写 backfill
-> finish-chain 决定 noReturn / target-courier / controller-return
```

VAD 脚本不直接调用 Codex automation API，只生成 payload 和 record 命令；真正的 automation 创建 / 删除由 Codex 工具完成。

### 3. VAD unattended controller

后续增强加入 `dispatchGroup` 和 `controller-last`：

```text
多个目标窗口 heartbeat
-> 每个窗口只 claim / finish 自己任务
-> 非最后窗口 noReturn
-> 最后窗口 returnToController
-> controller-return heartbeat 唤醒总控
-> 总控 review group evidence
-> accept / rework / block / next wave / stop
```

这是为了解决无人值守模式下“多个窗口完成后谁唤醒总控、总控如何继续”的问题。它的安全边界是：自动化只负责投递和回跳，总控仍负责验收和下一步裁决。

### 4. Controller self / long-running mode

`controller-tick`、`start-plan`、`resume-plan`、`stop-plan` 是总控侧的自动化决策入口：

- `start-plan`：当前计划首次无人值守启动。
- `resume-plan`：controller return 或人工中断后的续跑。
- `controller-tick`：只读分类当前 mode / queue / current plan / TODO。
- `post-run-audit`：声明自动化表面清理干净前的结束检查。
- `stop-plan`：关闭后续 finish-chain 和 keep-awake。

当前规则强调 fast path first：正常启动和续跑优先 `start-plan` / `resume-plan`，只有异常时再 audit / preflight / post-run-audit。

## 实现清单

### 文档与需求来源

- `workspace-ledger/requirement-designs/visible-automation-dispatch/original-plan-2026-05-25.md`
- `workspace-ledger/requirement-designs/visible-automation-dispatch/requirement-design-2026-05-25.md`
- `workspace-ledger/requirement-designs/visible-automation-dispatch/code-implementation-dependency-research-2026-05-25.md`
- `workspace-ledger/requirement-designs/visible-automation-dispatch/unattended-controller-requirement-design-2026-05-26.md`
- `workspace-ledger/workspace/archive/2026-05/visible-automation-dispatch/`

### 脚本

- `codex-control-workspace/scripts/visible-dispatch.mjs`
  - mode / keep-awake
  - registry
  - queue
  - dispatch groups
  - arm / arm-batch payload
  - record-arm / record-return / record-stop
  - claim / finish / accept / block
  - controller-tick / group-status / audit-automation / post-run-audit
  - prune-history
- `codex-control-workspace/scripts/workspace-control.mjs`
  - `vad` 聚合入口。
- `codex-control-workspace/scripts/verify-control-center.mjs`
  - 总控文档、dispatch coverage、test boundary、script docs、runtime residue 等总体验证入口。
- `codex-control-workspace/scripts/check-dispatch-coverage.mjs`
  - 检查 current plan 发送窗口和提示词边界。
- `codex-control-workspace/scripts/check-task-packages.mjs`
  - 检查任务包字段。
- `codex-control-workspace/scripts/check-test-boundary.mjs`
  - 防止误把 AlembicTest 当默认测试队列。

### Skill / reference

- `codex-control-workspace/skills/dev/control-workspace-governance/references/visible-automation-dispatch.md`
  - VAD 操作地图。
- `codex-control-workspace/skills/dev/visible-automation-dispatch-target/SKILL.md`
  - 目标窗口收到 heartbeat 后的 claim / finish / wakeup cleanup / next-hop 权限。
- `codex-control-workspace/skills/dev/visible-automation-dispatch-controller/SKILL.md`
  - 总控收到 controller-return 后的 evidence review / next unit / dispatch or stop。

### 本地运行态

VAD runtime 存在于 ignored 目录：

```text
codex-control-workspace/.workspace-local/visible-dispatch/
```

当前文件：

- `state.json`
- `window-registry.json`
- `dispatch-queue.json`
- `automation-runs.json`
- `dispatch-groups.json`
- `keep-awake-control.json`

这些文件保存 raw thread id 和 automation id 等本机状态，不得写入 tracked 文档、GitHub 或回填正文。

## 当前本地状态快照

2026-05-29 读取 `node scripts/visible-dispatch.mjs status --json`：

- mode：`disabled`
- loopEnabled：`false`
- registeredWindows：`7`
- queue tasks：`accepted=35`
- automationRuns：`60`
- dispatchGroups：`22`
- keep-awake：已停止
- stopReason：`user requested automation off; continue task manually`

读取 `controller-tick --compact --json`：

- current plan：`progressive-chain-validation-metrics-wave-6b-agent-stage-node-identity-2026-05-29.md`
- current status：`Wave 6B 已启动 / 发送给 AlembicAgent`
- sendEligibleWindowCount：`1`
- missingSendEligibleWindowCount：`1`
- topAction：`stopped`
- nextAction：`modeDisabled`
- message：automation mode disabled，不 enqueue / arm / select TODO。

读取 `post-run-audit --json`：

- activeTaskCount：`0`
- activeAutomationRunCount：`0`
- issue：当前计划仍有 send-eligible window `AlembicAgent`。

解释：这不代表 VAD runtime 有 active automation 残留，而是说明当前人工计划仍有可发送窗口；`post-run-audit` 更适合“声明自动化运行面清理干净”时使用，不适合把普通手动主线也判成已完全空闲。

## 真实使用历史

### 原始目标

VAD 来自用户要求：去掉总控把提示词手动复制到各 Codex 窗口的搬运过程，同时保持 Codex Mac 前端可见输出。用户不接受完全 headless session 替代可见窗口。

### Wave 1-3

已完成本地 schema、queue、claim、accept、arm payload、record-arm / record-stop / block 等基础能力。脚本仍不直接调用 Codex automation API。

### Wave 4

single-window heartbeat 验证失败：真实 heartbeat 创建后，在观察窗口内未触发目标可见线程 claim / complete。总控后续将该结论收窄为“当时 single-window / 当前可见线程 heartbeat 未触发”，没有外推为 VAD 全局失败。

### Wave 5

新增 `controller-tick`。重点从“盲目继续创建 heartbeat”转为总控先读 mode / queue / current plan / TODO，再判断 stopped / review / arm / wait / enqueue / decision / candidate。

### Wave 7

新增 `finish --chain-next` 和 mode 门禁：

- mode enabled：finish 可生成下一跳 payload。
- mode disabled：只记录完成，不生成下一跳。
- `prune-history` 用于清理旧 terminal runtime 残留。

### Wave 8

heartbeat payload 固定使用 `FREQ=MINUTELY;INTERVAL=1`，并增加 unattended target prompt。窗口 thread id 只写入本地 ignored runtime，输出 redacted 信息。关闭模式后，已 armed payload 醒来也不能继续生成下一跳。

### Wave 9

真实 heartbeat fake TODO smoke 跑通 6 个窗口 / 6 条 automation run，但暴露治理问题：

- `AlembicPlugin` 完成自身任务后越权创建 `AlembicTest` 下一跳。
- `AlembicTest` registry 曾出现 `current-codex-thread` 占位符。

后续修复方向：

- 抽取 VAD target skill。
- 同步 AGENTS role guard。
- 脚本拒绝 placeholder thread id。
- `AlembicTest` 下一跳默认 total-control-owned。

### Unattended Controller Wave 0

加入 dispatch group、controller-last、controller-return 和 controller skill。修复 open group 提前 review、controller-return 重复 / 幂等停止、missing group task、active automation accept guard 和 thread id 输出边界。

### 037 real-loop

使用 `GTODO-2026-05-24-037` 做真实 VAD 关系链测试：

- 三个真实 heartbeat：`AlembicPlugin` / `Alembic` / `AlembicCore`。
- 三窗口只 claim / finish 自己任务。
- 前两个窗口不回跳。
- 最后窗口创建 controller-return。
- 总控回跳后 review group evidence 并 accept。
- 该测试证明 VAD real-loop transport 可用于 037 后续自动化推进，但不证明 037 产品功能已完成。

## 当前使用判断

- 自动化不是默认常开；当前本地是 `mode disabled`。
- 真实主线仍主要依赖手动总控计划和可复制提示词。
- VAD 已经可用于低风险、明确边界的真实 heartbeat 关系链验证。
- VAD 尚未成为“每个普通开发任务都默认走的路径”。
- 对简单单窗口任务，当前 VAD finish / chain-next / record-stop / controller-return 的操作复杂度明显高于人工一句 “某窗口完成了”。
- 对多窗口无人值守批次，结构化 VAD 仍有价值，尤其是 queue、group、role guard、thread id gate 和 active automation guard。

## 主要痛点

### 1. 简单任务回总控过重

现在结构化路径要求子窗口理解 claim、finish、chain-next、record-stop、可能的 controller-return。对单窗口简单任务，这比人工短句重很多。

对应候选：`CHILD-WINDOW-COMPLETION-SIGNAL-2026-05-29`。

### 2. 手动提示词和 VAD payload 缺少统一 envelope

手动复制提示词、VAD heartbeat prompt、多 IDE agent 任务包表达的是同一件事，但目前复用层还不清晰，导致 AGENTS、定位、禁止事项、回填格式反复出现。

对应候选：`MULTI-LLM-DISPATCH-OPTIMIZATION-2026-05-29`。

### 3. controller-last 和 target-courier 双模式增加理解成本

`controller-last` 更符合总控验收逻辑；`target-courier` 适合链式 smoke，但更容易出现跨窗口越权。

建议默认只保留 `controller-last` 作为普通无人值守批次路径，`target-courier` 作为显式测试 / 特殊链路 opt-in。

### 4. post-run-audit 对普通手动主线容易产生误读

当前 `post-run-audit` 会因为 current plan 仍有 send-eligible window 报 issue。这个行为对“自动化运行面是否清理干净”是合理的，但对“现在是否可以继续人工总控任务”容易误解。

建议后续拆出：

- automation run cleanup audit
- current manual plan sendability
- current active VAD risk

### 5. evidence 仍分散在 queue backfill、workspace docs、git、报告和聊天里

VAD queue backfill 不是总控验收事实。总控仍要读取 commit、diff、命令输出、runtime JSON、报告或截图。当前这点规则正确，但会让自动化回填和总控 pull review 之间存在重复字段。

建议后续定义 `Backfill Envelope`：子窗口只提供证据索引和摘要；总控 pull 原始证据。

### 6. thread registry 是高价值但脆弱的本机状态

thread id 必须真实、local-only、可 preflight。历史上已有 placeholder thread id 污染。当前已加 guard，但后续仍需要 registry health / stale diagnosis 更容易读。

### 7. automation wakeup 删除 / record-stop 语义容易混淆

`record-stop --reason "target received"` 只是表示一次性 wakeup 已消费，不代表任务完成。这个规则已经写清，但对执行窗口仍是高认知负担。

### 8. TODO 自动领取边界仍需要更强分类

用户希望未来无人值守可以领取 TODO，但当前规则必须避免低价值小修、小文档、未确认需求、038/039 这种未 ready 项被自动领取。

建议只有满足“已有目标、接口 dossier、验证方式、低风险、无用户确认门禁”的 TODO 才 automation-eligible。

## 优化候选地图

| 候选 | 类型 | 解决问题 | 建议状态 |
| --- | --- | --- | --- |
| `CODEX-AUTOMATION-CLOSED-LOOP-2026-05-29` | automation-flow-optimization | Codex 内部 1*N / N*1 自动化闭环收敛，区分手动通用提示词和 target-specific automation prompt。 | 建议作为主需求交给总控做 Stage 0 规则与代码事实复核。 |
| `CHILD-WINDOW-COMPLETION-SIGNAL-2026-05-29` | automation-flow-optimization subcase | 简单子窗口回总控过重。 | 降级为 `CODEX-AUTOMATION-CLOSED-LOOP-2026-05-29` 的子场景，不建议单独作为首要实现。 |
| `MULTI-LLM-DISPATCH-OPTIMIZATION-2026-05-29` | dispatch-governance | 任务包 / 回填包 host-agnostic，降低提示词重复。 | 继续 Design，先不 ready。 |
| `AUTOMATION-TASK-ENVELOPE-2026-05-29` | requirement-candidate | 把手动提示词、VAD payload、多 IDE agent 任务包统一成 Task Envelope。 | 建议作为下一条完整需求候选。 |
| `VAD-RUNTIME-STATUS-SIMPLIFICATION-2026-05-29` | requirement-candidate | 区分 automation cleanup、manual sendability、active VAD risk，减少误报理解成本。 | 可做轻量 TODO。 |
| `VAD-CONTROLLER-LAST-DEFAULT-2026-05-29` | requirement-candidate | 将 `controller-last` 设为普通批次默认，`target-courier` 改为显式 opt-in。 | 需要总控代码事实确认。 |
| `VAD-REGISTRY-HEALTH-2026-05-29` | requirement-candidate | thread registry 真实 / stale / placeholder / session preflight 更易读。 | 可做低风险工具优化。 |
| `AUTOMATION-ELIGIBLE-TODO-2026-05-29` | requirement-candidate | 定义哪些 TODO 能无人值守领取。 | 需要与总控 TODO policy 对齐。 |

## 推荐收敛路线

### Stage A：先做盘点接收

总控接收本文，只做代码事实补充和 runtime 复核，不直接改脚本。

### Stage B：先拆低风险体验优化

优先选择两条：

1. `CODEX-AUTOMATION-CLOSED-LOOP-2026-05-29`
2. `VAD-RUNTIME-STATUS-SIMPLIFICATION-2026-05-29`

理由：先收敛 Codex 内部闭环和 prompt 分层，再处理状态误读；`CHILD-WINDOW-COMPLETION-SIGNAL-2026-05-29` 作为闭环中的 result signal 子场景吸收。

### Stage C：再做协议统一

把 `MULTI-LLM-DISPATCH-OPTIMIZATION-2026-05-29` 推进为 Task Envelope / Backfill Envelope 需求。它会影响手动提示词、VAD payload、多 IDE agent，所以适合单独设计。

### Stage D：最后治理无人值守 TODO 自动领取

等 envelope 和 status 更清楚后，再定义 `AUTOMATION-ELIGIBLE-TODO-2026-05-29`，避免把未 ready 的需求自动领取。

## 给总控的下一步建议

- 不要把本文直接变成实现 wave。
- 先作为 `research / requirement-candidate` 接收。
- 总控 Stage 0 可以读取：
  - `scripts/visible-dispatch.mjs`
  - `skills/dev/control-workspace-governance/references/visible-automation-dispatch.md`
  - `skills/dev/visible-automation-dispatch-target/SKILL.md`
  - `skills/dev/visible-automation-dispatch-controller/SKILL.md`
  - `.workspace-local/visible-dispatch/*.json`
  - VAD archive 的 Wave 4 / 5 / 7 / 8 / 9 / unattended controller / 037 real-loop 证据。
- Stage 0 输出应回答：
  - 哪些流程必须保留结构化 VAD。
  - 哪些场景可降级为 natural-language signal。
  - 哪些状态输出需要改名或拆分。
  - 哪些字段应进入 Task Envelope / Backfill Envelope。

## 仍需确认

- 用户是否希望把这些优化合成一个 `AUTOMATION-FLOW-OPTIMIZATION` 大需求，还是拆成多个小需求逐个交给总控。
- 第一优先级是否按 Design 推荐：先 completion signal，再 runtime status simplification。
- 简单单窗口任务是否默认允许“不走 VAD finish”，只用短 signal 触发总控 pull review。
- `target-courier` 是否只保留给显式 smoke / 特殊链式任务。
- TODO 自动领取是否必须等 Task Envelope / automation-eligible 分类完成后再启动。
