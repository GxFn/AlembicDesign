# Codex Automation Closed Loop 规则统一补充设计

Design Key：CODEX-AUTOMATION-CLOSED-LOOP-2026-05-29
补充主题：VAD redundancy removal / rule unification / child repository automation profile
日期：2026-05-29
状态：design-addendum / ready-for-workspace
维护窗口：AlembicDesign
总控：AlembicWorkspace

## 结论先行

当前 VAD 冗余的核心不是“规则太多”，而是三类内容混在了同一个 prompt 里：

1. 人工复制提示词需要的通用自包含说明。
2. 子窗口必须永久遵守的硬边界。
3. 某一次 automation 任务需要的目标信息。

正确收敛不是删除规则，而是分层：

```text
Hard Rules：AGENTS / child stop card，常驻，不每跳重述
Operating Steps：skill / reference，按需加载，不塞进 prompt
Controller Dispatch：目标窗口、任务内容、提示词、证据要求，由总控派发层生成
VAD Delivery Envelope：targetWindow + prompt + returnRoute + oneShot，本地投递信封
Runtime State：automation_id + thread registry + mode，本地状态
```

自动化 prompt 由总控派发层生成，VAD 只做“提示词透传 + 投递元数据 + 回跳地址”，不再承担“人工复制提示词 + 命令手册 + 总控计划摘要”的混合职责。

进一步校准：VAD 不应该实现第二套总控派发逻辑。VAD 只需要知道目标窗口、要投递的 prompt 和回跳路径；不负责从 current plan 解析任务、不决定发送给谁、不生成任务提示词、不判断 TODO 顺序、不解析 objective / evidence / forbidden actions。总控派发层把这些都决定好后，把一封已经写好的信交给 VAD。

新增主路径约束：配置完成后的 VAD 工作流必须保持单线、快速、低分叉。重量级 preflight、全量 audit、完整规则复核、registry 深检查、workspace 全量 verify 等能力只作为 Agent 可调用工具，或在失败 / 不一致 / 高风险场景使用，不进入每次 heartbeat 的默认路径。

正常路径应收敛为：

```text
controller builds prompt -> VAD delivers -> target executes -> VAD/controller-return wakes controller -> controller pull review
```

不应收敛成：

```text
start/resume -> preflight -> audit -> verify -> status diagnose -> prompt rebuild -> claim -> ...
```

## 最终确认目标

VAD 的产品化目标不是成为第二个总控，而是成为轻量投递层：

```ts
deliver({
  targetWindow,
  prompt,
  returnRoute,
  keepLive,
  oneShot,
  correlationId
})
```

已确认原则：

- 总控写信和验收；VAD 送信、保活、用完即弃、失败回报、必要时回跳。
- 总控知道目标窗口和任务内容，可以生成定向 prompt。
- 子窗口回调总控只需要简单唤醒和 correlation 信息；总控自行 pull evidence。
- keep live 是投递可靠性保障，不参与任务拆分、窗口选择或验收裁决。
- VAD 主路径轻量执行，重校验只作为失败诊断 / Agent 可调用能力。

## 快递信封能力分层

VAD 应先被理解成“快递信封”，再理解成“自动执行系统”。信封基础能力不是重量逻辑；它们是投递可靠性的最小语义。

### 1. 信封基础能力

这些能力每次 heartbeat 都需要具备，但应该很轻，不展开成复杂检查：

- **目标标识**：`currentWindow / deliveryId / returnRoute`，可带总控 correlation id。
- **提示词透传**：VAD 接收总控已经生成好的 prompt，并尽量原样投递；不重写任务目标和验收要求。
- **回跳地址**：VAD 记录完成后应该唤醒总控还是只停止；普通路径只回总控。
- **必要保活**：投递期间按需 keep live，防止无人值守任务因睡眠中断；保活失败只报告可靠性风险，不改变派发策略。
- **投递目标**：真实 thread id 只在本地 runtime 使用，不进入 prompt、tracked 文档或回填正文。
- **一次性消费**：heartbeat 被目标窗口接收后要能 `record-stop` 或等价标记，避免同一封信反复唤醒。
- **轻量身份校验**：payload 的 `currentWindow` 必须等于当前窗口；delivery id / correlation id 只做投递对齐，不做任务语义判断。
- **轻量模式门禁**：mode disabled 时不继续链式投递。
- **技能指针**：prompt 只给 target/controller skill 引用，不复制命令手册。
- **失败升级口**：发现缺字段、错窗口、错 task、无真实 thread 或 mode 冲突时停下，进入诊断能力。

其中 `用完即弃` 属于信封基础能力：它只表示“这次唤醒已被消费”，不表示任务完成、不表示证据通过，也不触发下一波裁决。

### 2. 最小闭环能力

这些能力是一次自动化任务从派发到验收必须有的最小闭环，但其中任务内容由总控派发层生产，VAD 只承载和投递：

- **Controller prompt**：总控已经写好的目标窗口 prompt，包含任务目标、边界、证据和 result envelope 要求。
- **Delivery envelope**：VAD 只记录目标窗口、提示词、一次性消费、回跳地址和本地 delivery id。
- **Task execution**：目标窗口在本仓库职责内执行任务。
- **Result envelope**：返回 `status / changedRepos / commits / evidenceRefs / verificationSummary / riskSummary`。
- **Controller pull review**：总控根据 result envelope 拉原始证据并裁决。
- **Controller-last**：普通路径由总控决定下一步，不让子窗口默认续投递。

最小闭环允许失败，但失败也应以 result envelope 或明确 stop reason 返回，而不是自动进入大规模诊断。

### 3. 重量能力

这些能力保留给 Agent 调用，或在失败 / 高风险时使用：

- 全量 `preflight`。
- `audit-automation` / `post-run-audit`。
- registry deep audit / stale thread 修复。
- workspace docs verify / full verify。
- current plan 全文重解析。
- 强制刷新父级 AGENTS、current plan、目标仓库 AGENTS。
- 复杂 Markdown 推断 controller dispatch packet。
- 从 current plan / TODO 表自动解析派发窗口和任务顺序。
- 在 VAD 内部生成 target-specific prompt。
- target-courier 链式投递。
- TestWindow / real project 多条件边界检查。
- 回填与 commit / diff / test evidence 冲突调查。

结论：普通 VAD 主路径 = 信封基础能力 + 最小闭环能力；重量能力只在失败升级、高风险 profile 或总控明确要求时进入。

## VAD 与总控派发边界

### 总控派发层负责

- 读取 current plan / TODO / Backlog。
- 判断用户目标、阶段顺序、窗口覆盖、producer / consumer 依赖。
- 选择目标窗口。
- 生成目标窗口 prompt。
- 决定 result envelope 字段和验收证据。
- 决定是否需要 TestWindow、是否高风险、是否必须 force-refresh。
- 收到结果后 pull evidence 并裁决下一步。

### VAD 投递层负责

- 接收总控给出的 `targetWindow`。
- 接收总控给出的 prompt，不解析任务语义。
- 接收总控给出的 keep live 需求，只做运行保障。
- 根据 registry 找到目标 Codex thread。
- 创建 heartbeat / automation。
- 记录一次性投递、消费和停止状态。
- 根据总控给出的 return route 唤醒总控。
- 在投递失败、thread 缺失、payload 冲突时提供诊断入口。

### VAD 不负责

- 不从 current plan 里解析任务行。
- 不决定哪些窗口 `待启动`。
- 不根据 TODO 排优先级。
- 不生成 target-specific task prompt。
- 不理解或校验 objective / forbiddenActions / evidenceRequired。
- 不根据子窗口回填裁决 accepted / rework / next-wave。
- 不默认 target-courier 续投递。

因此，当前 `enqueueFromPlan` / `buildTaskPrompt` 中的“读取计划、拼任务提示词、按 send-eligible row 生成任务”应被视为历史桥接或总控派发模块逻辑，不应继续作为 VAD 核心职责。

## 已观察到的真实冗余

### 1. 子仓库接入卡已经要求 VAD 轻量，但脚本仍输出长人工提示

子仓库 `AGENTS.md` 的 managed access card 已写：

```text
VAD heartbeat 提示词只承载动态变量、规则名和 skill 指向；不得把提示词当成完整命令手册。
```

但 `codex-control-workspace/scripts/visible-dispatch.mjs` 的 `buildTaskPrompt(task)` 仍输出：

```text
继续当前窗口任务：<window>

先读取 AGENTS.md、workspace index、current plan，以及目标仓库 AGENTS.md。
先明确声明当前窗口定位和本轮仓库职责。
再按照文档领取并完成分配给你所在窗口的任务。
如果任务包较大...
完成后回填...
自动化补充...
```

这说明规则目标已经变轻，但实现仍保留了人工提示词结构。

### 2. 父级分派规则没有区分 manual prompt 和 automation prompt

父级 `AGENTS.md` 和 `window-dispatch.md` 当前要求所有 `待启动 / 执行中` 窗口的任务包和可复制提示词都包含：

- 读取 workspace `AGENTS.md`。
- 读取 current plan。
- 读取目标仓库 `AGENTS.md`。
- 明确窗口定位。

这对人工复制提示词是必要的，但对 VAD heartbeat 不应机械套用。VAD 自身只需要 registry、targetWindow、deliveryId、returnRoute 和一次性消费状态；任务语义由总控 prompt 承载。

需要把规则拆成：

- Manual dispatch preconditions。
- Automation heartbeat preconditions。

### 3. VAD queue 试图承担派发任务模型

当前 `enqueueFromPlan` 生成 task 时主要保存：

- `taskId`
- `targetWindow`
- `controlDoc`
- `promptRef: "可复制提示词"`
- `taskSummary`
- `groupId`
- `returnPolicy`

这些字段看似不够完整，但真正的问题不是“VAD queue 缺少更多派发字段”，而是 VAD queue 不应该承载派发任务模型。`objective / allowed / forbidden / verification / evidenceRequired` 应由总控派发层写进 prompt 或 controller dispatch packet；VAD 只存 delivery envelope。

因此不要把解决方向设成“给 VAD queue 加完整 task packet”。正确方向是：让总控派发层生成完整 prompt，VAD 透传 prompt。

### 4. 子窗口回填和 runtime cleanup 语义混在一起

`record-stop --reason "target received"` 只是一次性 wakeup 已消费，不是任务完成证据。当前 skill 已经写清，但 prompt 仍容易让执行窗口把 wakeup cleanup、finish、result evidence 混成一个流程。

需要把这三件事拆开：

- Wakeup cleanup：防止同一 heartbeat 重复打扰。
- Task result：子窗口完成 / 阻塞 / 待验收。
- Controller review：总控拉证据并裁决。

### 5. target-courier 历史价值存在，但不应是普通闭环默认

`target-courier` 适合 smoke 和特殊链式测试，但普通 Codex 内部闭环应默认：

```text
controller-last
```

否则子窗口容易承担“给下一个窗口发什么”的调度责任，曾经也确实暴露过 Plugin 越权调起 AlembicTest 的问题。

## 主路径与能力支持

### VAD Happy Path

配置完整、registry 正常、目标窗口清晰时，VAD 默认只做以下动作：

1. 总控派发层生成目标窗口 prompt。
2. VAD 接收 `targetWindow / prompt / returnRoute / oneShot`。
3. VAD 创建 heartbeat 并投递到目标窗口。
4. 目标窗口执行任务并返回 result envelope。
5. VAD 或 controller-return 唤醒总控。
6. 总控 pull evidence 并裁决下一步。

这条路径不默认运行：

- 全量 `preflight`。
- 全量 `post-run-audit`。
- workspace docs verify。
- registry deep audit。
- current plan 全文重解析。
- AGENTS / current plan 每跳强制长读。
- 在 VAD 内部重建 target prompt。
- 在 VAD 内部解析 TODO / current plan 派发表。
- 自动创建下一目标窗口 heartbeat。

### Capability Support

复杂能力保留，但作为工具箱：

- `preflight`：用于首次安装、配置变更、registry 缺失、thread id 异常。
- `audit-automation` / `post-run-audit`：用于回跳冲突、automation 残留、证据异常、用户要求清场。
- keep live 异常诊断：用于保活启动 / 停止失败，不作为派发分支。
- docs verify / workspace verify：用于规则治理、脚本改动、归档前验收，不作为普通 heartbeat 前置。
- `force-refresh`：用于删除、迁移、发布、跨仓库、TestWindow、真实项目、高风险或上下文不确定任务。
- full current plan scan：用于总控派发层诊断任务包缺失、task id 不匹配、目标窗口不清；不作为 VAD 投递层默认动作。

不应放进重量能力的基础项：

- `record-stop` / 一次性消费标记。
- payload 与 claim 的轻量匹配。
- mode disabled 停止续投递。
- thread id local-only。
- skill ref。

这些属于快递信封的基本语义，必须轻量常驻。

### 失败触发条件

只有命中以下条件才进入重量逻辑：

- payload 的 `currentWindow` 与目标窗口不一致。
- payload 缺少 `currentWindow / deliveryId / returnRoute / prompt`。
- prompt 缺失目标、禁止事项或证据要求。
- registry 没有真实 thread id，或 thread id 疑似占位。
- mode disabled 但仍尝试续跳。
- heartbeat 创建失败或重复触发。
- 子窗口回填与 commit / diff / test evidence 冲突。
- 当前任务命中高风险 profile。
- 用户或总控明确要求 audit / full verify。

这样 VAD 主路径不分叉，复杂能力仍可被 Agent 在需要时调用。

## 规则分层方案

### A. Hard Rules

位置：

- 父级 `AGENTS.md`
- 子仓库本地 `AGENTS.md` 的 stop card
- managed Workspace 接入卡中的 VAD 最小门禁

保留内容：

- 自动化不改变窗口职责。
- 子窗口只能处理自己的 `targetWindow / taskId`。
- claim mismatch 必须停。
- thread id local-only。
- `AlembicTest` 下一跳总控拥有。
- evidence 不足不得验收。
- 总控 pull review 才是完成事实。
- mode disabled 不得继续链式投递。

不应放在每次 heartbeat prompt 中长篇重复。

### B. Operating Steps

位置：

- `skills/dev/visible-automation-dispatch-target/SKILL.md`
- `skills/dev/visible-automation-dispatch-controller/SKILL.md`
- `references/visible-automation-dispatch.md`

承载：

- claim / finish / group-status / controller-tick / record-stop 命令细节。
- 异常分支。
- 如何删除一次性 wakeup。
- 如何处理 finish JSON。

prompt 只引用 skill 和规则名，不复制操作手册。

### C. Controller Dispatch Packet

位置：

- 当前计划 task package。
- 总控派发模块。
- 可选脚本可读 block。

承载：

```ts
type ControllerDispatchPacket = {
  taskId: string;
  targetWindow: string;
  controlDoc: string;
  taskPackageAnchor?: string;
  objective: string;
  allowedActions: string[];
  forbiddenActions: string[];
  verificationCommands: string[];
  evidenceRequired: string[];
  backfillTarget?: string;
  riskLevel: "low" | "normal" | "high";
  contextPolicy: "assumed-current" | "refresh-if-missing" | "force-refresh";
  promptProfile: "codex-target-compact" | "codex-target-high-risk";
  prompt: string;
};
```

说明：这是总控派发层的数据，不是 VAD 投递层的数据。VAD 可以保存 opaque `deliveryRef` 或 prompt 文本，但不解析 `objective / forbiddenActions / evidenceRequired`。

### D. VAD Delivery Envelope

位置：

- `.workspace-local/visible-dispatch/*.json`
- heartbeat payload metadata

承载：

```ts
type VadDeliveryEnvelope = {
  deliveryId: string;
  targetWindow: string;
  prompt: string;
  returnRoute: "controller" | "none";
  oneShot: true;
  keepLive?: boolean;
  controllerCorrelationId?: string;
  localAutomationId?: string;
  status: "queued" | "armed" | "delivered" | "consumed" | "failed";
};
```

承载内容：

- 目标窗口。
- 总控给出的 prompt。
- 回跳地址。
- keep live 需求。
- 一次性消费状态。
- 本地 automation id。
- 本地 thread readiness。

这些是本地运行态，不写 tracked docs。

## Prompt Profile

### 1. `manual-universal`

用途：人工复制粘贴。

特点：

- 自包含。
- 要求读取 `AGENTS.md`、index、current plan、目标仓库 AGENTS。
- 适合用户把同一段发给任意窗口。
- 保留现有通用提示词风格。

### 2. `codex-target-compact`

用途：普通 Codex automation。

建议 prompt：

```text
继续当前窗口任务：{targetWindow} / {taskId}

目标：{objective}
任务锚点：{controlDoc}{taskPackageAnchor}
边界：{allowedActions}; 禁止：{forbiddenActions}
验证：{verificationCommands}
回填：返回 TargetResultEnvelope，包含 status、commits、evidenceRefs、verificationSummary、riskSummary。

自动化补充：
- currentWindow: {targetWindow}
- taskId: {taskId}
- controlDoc: {controlDoc}
- contextPolicy: refresh-if-missing
- skill: {targetSkillPath}
```

这段 prompt 由总控派发层生成。VAD 只接收并投递，不根据 current plan 自行生成。

不重复：

- 完整 AGENTS 阅读清单。
- current status。
- 子 agent 泛化说明。
- 完整 finish / record-stop 命令手册。

### 3. `codex-target-high-risk`

用途：

- 删除。
- 迁移。
- 发布。
- 跨仓库边界。
- TestWindow / real project。
- 用户确认门禁相关任务。

特点：

- 仍然 target-specific。
- 但 `contextPolicy=force-refresh`。
- prompt 明确要求刷新父级 / 子仓库 AGENTS 和 current plan。

### 4. `codex-controller-return-compact`

用途：最后窗口回跳总控。

建议 prompt：

```text
继续总控验收：dispatchGroup {dispatchGroup}

最近完成：{lastCompletedTarget} / {lastTaskId}
计划：{controlPlan}
动作：读取 group evidence，pull 原始证据，裁决 accepted / rework / blocked / next-wave / stop。

自动化补充：
- dispatchGroup: {dispatchGroup}
- lastCompletedTarget: {lastCompletedTarget}
- lastTaskId: {lastTaskId}
- controlPlan: {controlPlan}
- contextPolicy: refresh-if-missing
- skill: {controllerSkillPath}
```

## Context Policy

`contextPolicy` 是解决“要不要每次强制读 AGENTS”的关键。

它不是让普通路径产生多套分叉流程，而是一个失败升级阈值。正常 heartbeat 仍走同一条 fast path；只有上下文缺失、目标不确定或任务高风险时，才升级到刷新 / 审计 / 诊断能力。

### `assumed-current`

适用：

- 同一窗口连续 VAD heartbeat。
- registry / preflight 已确认 cwd、windowName、thread。
- 总控生成的 prompt 完整。
- 低风险单仓库任务。

行为：

- 不在 prompt 强制长读 AGENTS。
- 子窗口如发现上下文缺失、task mismatch、cwd mismatch，再刷新。

### `refresh-if-missing`

适用：

- 默认普通 Codex automation。
- 子窗口可能换过上下文。
- 总控生成的 prompt 完整但需要保险。

行为：

- prompt 不长篇要求强读。
- target skill / access card 要求：无法确认 window / task / repo 时，先读 AGENTS / current plan；确认无误后执行。

### `force-refresh`

适用：

- 高风险任务。
- 首次登记。
- AGENTS / skill / prompt profile 刚变更。
- 跨仓库 / 删除 / 迁移 / 发布 / TestWindow / real project。

行为：

- prompt 明确要求刷新父级 AGENTS、current plan 和目标仓库 AGENTS。

## 子仓库配置方案

### 当前配置事实

`codex-control-workspace/.workspace-local/workspace.config.json` 已提供 Alembic 实例化窗口：

- `Alembic`
- `AlembicCore`
- `AlembicAgent`
- `AlembicDashboard`
- `AlembicPlugin`
- `AlembicDesign`
- `AlembicTest`
- `BiliDili`

每个产品子仓库 `AGENTS.md` 由 `control-workspace-install.mjs write-agents` 写入 managed access card。

当前安装脚本的 `normalizedRepositories` 只保留 `windowName / path / mode / role / managedAgents` 等基础字段；如果后续直接在 JSON 里增加 `automationProfile`，脚本默认不会自动消费这些字段。因此配置优化不能只写配置文件，还要让以下读取点同时认识 profile：

- workspace config 归一化。
- VAD enqueue / prompt builder。
- managed AGENTS access card 生成。
- 可能的 `prompts` / `status` 诊断输出。

### 推荐配置层级

`automationProfile` 建议按两层处理：

1. tracked `workspace.config.json` 放通用 profile 名称和默认结构，例如 `product-repo`、`design-window`、`test-window`、`real-project-protected`。
2. ignored `.workspace-local/workspace.config.json` 放 Alembic 实例化窗口的实际覆盖，例如 `AlembicAgent` 默认 `codex-target-compact`，`AlembicTest` 默认 `codex-target-high-risk`。

这样不会把 Alembic 专属窗口策略硬编码进通用 control workspace，也不会要求所有安装实例都接受同一套窗口名。

### 建议新增 automation profile

不一定第一阶段实现，但建议作为配置设计目标：

```json
{
  "windowName": "AlembicAgent",
  "automationProfile": {
    "defaultPromptProfile": "codex-target-compact",
    "defaultContextPolicy": "refresh-if-missing",
    "allowedReturnPolicies": ["controller-last"],
    "targetCourierAllowed": false,
    "forceRefreshFor": ["delete", "migration", "release", "cross-repository", "test-window"],
    "resultEnvelope": "commit-tests-risk",
    "ruleRefs": [
      "visible-automation-dispatch-target",
      "local-repository-stop-card"
    ]
  }
}
```

`AlembicTest` 应覆盖：

```json
{
  "windowName": "AlembicTest",
  "automationProfile": {
    "defaultPromptProfile": "codex-target-high-risk",
    "defaultContextPolicy": "force-refresh",
    "allowedReturnPolicies": ["controller-last"],
    "targetCourierAllowed": false,
    "requiresTestBoundary": true
  }
}
```

`BiliDili` 默认不进入 automation dispatch target。

### 子仓库 AGENTS managed access card 调整

当前 access card 的 `领取 workspace 任务时` 是通用入口。建议拆成两段：

#### Manual Workspace Task

保留当前强读规则：

```text
人工复制提示词 / 普通 workspace 任务：
1. 读本仓库 AGENTS。
2. 读父级 AGENTS。
3. 读 workspace index / status / current plan。
4. 明确窗口定位。
```

#### VAD Compact Heartbeat

新增紧凑规则：

```text
VAD compact heartbeat：
1. 如果 payload 有 currentWindow / deliveryId / returnRoute，先确认 currentWindow 等于本窗口。
2. 收到 heartbeat 后做一次性消费标记，避免重复唤醒。
3. 执行 prompt 中的任务说明；VAD payload 不替代 prompt，也不追加总控派发逻辑。
4. 如果 prompt 要求 force-refresh 或本窗口无法确认边界，再刷新父级 AGENTS、current plan、目标仓库 AGENTS。
5. 子窗口只返回 TargetResultEnvelope，不决定下一波。
```

这样子仓库配置仍保留硬边界，但不会要求每个 heartbeat prompt 都复制人工路径。

## VAD 数据模型调整候选

### 当前

```ts
DispatchTask {
  taskId;
  targetWindow;
  status;
  controlDoc;
  promptRef: "可复制提示词";
  taskSummary;
  groupId;
  returnPolicy;
}
```

### 建议

```ts
VadDelivery {
  deliveryId;
  targetWindow;
  status;
  prompt;
  returnRoute;
  oneShot;
  controllerCorrelationId;
  localAutomationId;
  createdAt;
  consumedAt;
  failedReason;
}
```

第一阶段目标不是让 VAD 解析更完整的 current plan，而是让 VAD 接收总控派发层已经生成好的 prompt。没有 prompt 时，VAD 应停止并回报总控派发缺失，不自行回退生成 `manual-universal`。

## Result Envelope

目标窗口回总控的结果必须比“一句完成了”多一点，但比完整回填文档轻：

```ts
type TargetResultEnvelope = {
  targetWindow: string;
  taskId: string;
  status: "completed" | "blocked" | "needs-review";
  changedRepos: string[];
  commits: string[];
  evidenceRefs: string[];
  verificationSummary: string[];
  riskSummary: string[];
  nextSuggestion?: string;
};
```

总控再根据 `evidenceRefs` pull 原始证据。

## 普通闭环默认策略

默认：

```text
controller-last
```

含义：

- 总控一次发给 N 个子窗口。
- 子窗口各自返回 result envelope。
- 非最后窗口不创建下一跳。
- 最后窗口可以创建 controller-return，但只作为唤醒总控的 delivery。
- 总控决定验收 / 返工 / 下一波。

`target-courier`：

- 只允许显式 opt-in。
- 适合 smoke / 特殊链式测试。
- 不作为普通任务默认。

## 冗余移除清单

### 从 VAD target heartbeat prompt 移除

- 通用“先读取 AGENTS.md、index、current plan、目标仓库 AGENTS.md”长句。
- 通用“先明确声明窗口定位”长句。
- 通用“如果任务包较大，可开启子 agent”长句。
- 通用“完成后回填：完成范围、提交 hash...”长句。
- 完整 command manual。

### 移到总控派发层

- objective。
- allowed / forbidden。
- verification commands。
- evidenceRequired。
- backfill target。
- stop conditions。
- target-specific prompt 生成。
- current plan / TODO 解析。

### 移到 skill

- claim / finish 命令。
- record-stop 命令。
- finish JSON 分支。
- controller-return 处理步骤。

### 保留在 AGENTS

- 硬停止卡。
- role guard。
- TestWindow 边界。
- thread id local-only。
- evidence review 不是 verdict。

## 迁移阶段候选

### Stage 0：规则审计

总控复核：

- `AGENTS.md` 中哪些“读取 AGENTS / current plan”属于 manual dispatch hard rule。
- 哪些可以拆成 automation compact context policy。
- 子仓库 managed access card 是否允许拆分 manual / VAD compact。
- 哪些诊断 / 校验只能作为失败场景工具，不能进入 VAD happy path。
- 哪些属于“快递信封基础能力”，必须保留在每次投递中但保持轻量。

完成条件：形成 rule governance 结论，不改脚本。

### Stage 1：配置 profile 草案

在配置层设计 `automationProfile`，先不改行为：

- default prompt profile。
- context policy。
- allowed return policy。
- TestWindow / real project override。

完成条件：总控确认配置字段和子仓库边界。

### Stage 2：VAD transport API 最小化

VAD 输入改为 delivery request：

- targetWindow。
- prompt。
- returnRoute。
- keepLive。
- oneShot。
- controllerCorrelationId。

完成条件：VAD 不再需要解析 current plan 或 task packet；缺 prompt 时停止并回报总控派发层。

### Stage 3：总控派发 prompt builder

把 target-specific prompt builder 从 VAD 核心职责中拆出，归入总控派发层：

- `manual-universal` 不动。
- 总控派发层生成 `codex-target-compact`。
- 高风险任务使用 `codex-target-high-risk`。
- VAD 只包装极薄 delivery metadata，不重写 prompt。
- 默认 prompt 不触发全量 preflight / audit / verify；只暴露必要 skill ref 和失败升级条件。

完成条件：prompt 明显短，VAD 只负责送达和回跳。

### Stage 4：managed AGENTS access card split

更新 `control-workspace-install.mjs` managed block：

- Manual Workspace Task。
- VAD Compact Heartbeat。
- VAD hard stop conditions。

完成条件：子仓库配置更清楚，不再和 compact prompt 冲突。

### Stage 5：真实 smoke

先单窗口：

- `AlembicAgent` target-specific prompt。
- result envelope。
- 总控 pull evidence。

再多窗口：

- 1*N controller fan-out。
- N*1 controller-last return。
- 无 target-courier。

完成条件：证明 prompt 压缩后没有降低安全和证据质量。

## 验收指标

- VAD target prompt 行数减少。
- prompt 中重复 AGENTS / current plan 文案减少。
- 配置完成后的普通 VAD 路径只有一条 happy path，不默认进入 preflight / audit / verify 分叉。
- keep live 作为投递保障可用，但不会引入派发分支。
- `用完即弃`、payload/claim 轻量匹配、thread id local-only 等信封基础能力保留，但不被误写成任务完成或总控裁决。
- VAD 不再复制总控派发逻辑：不解析 current plan、不判定 send-eligible、不生成任务 prompt、不理解 objective / boundary / evidence。
- target task 的 objective / boundary / evidence 由总控生成的 prompt 直接承载。
- 子窗口不再需要理解 target-courier 作为普通路径。
- result envelope 一次通过率不下降。
- 总控 pull review 证据完整。
- 没有出现跨窗口 claim、TestWindow 越权或 raw thread id 泄漏。

## 给总控的建议

不要直接“删 prompt 文本”。先做 Stage 0 rule governance：

1. 承认 manual dispatch 和 VAD automation 是两种入口。
2. 承认总控派发层和 VAD 投递层是两层，不能在 VAD 里再做一遍总控派发。
3. 在父级规则中把“可复制提示词必须强读 AGENTS”限定为 manual / generic prompt。
4. 在 VAD 规则中允许 compact heartbeat 只做 delivery envelope。
5. 子仓库 managed access card 拆成 manual task 与 VAD compact heartbeat 两段。
6. 明确 VAD happy path 不默认跑重校验；把 preflight / audit / full verify 保留为可调用能力和失败诊断。
7. 然后把 `buildTaskPrompt` 拆出 VAD 核心，改为总控派发层生成 prompt、VAD 透传。

## 总控接手后裁决项

- 普通路径默认使用 `refresh-if-missing`，还是允许低风险场景使用 `assumed-current`。
- `target-courier` 作为显式 opt-in 的具体允许场景。
- 将 `enqueueFromPlan` / `buildTaskPrompt` 中的总控派发职责迁出 VAD 核心的实现方案。
- 子仓库 managed AGENTS access card 由安装脚本统一升级的具体 diff。
