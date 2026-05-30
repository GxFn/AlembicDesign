# Workspace Workflow Optimization Analysis

Design Key：WORKSPACE-WORKFLOW-OPTIMIZATION-2026-05-29
日期：2026-05-29
状态：research input / superseded-by-requirement-design
维护窗口：AlembicDesign
总控：AlembicWorkspace
关联参考：CODEX-AUTOMATION-CLOSED-LOOP-2026-05-29
后续正式文档：[workspace-workflow-optimization-original-plan-2026-05-29.md](workspace-workflow-optimization-original-plan-2026-05-29.md)、[workspace-workflow-optimization-requirement-design-2026-05-29.md](workspace-workflow-optimization-requirement-design-2026-05-29.md)

## 判断类型

这是 Workspace 整体工作流优化的 research input，不是执行计划，不直接创建总控 TODO，不修改 workspace 当前状态。

2026-05-29 更新：用户已按 Design 推荐确认继续深化；正式原始计划和需求设计已拆分到后续文档。本文件保留为证据输入和思考底稿。

它参照 VAD 优化的核心方法：

- 先拆开职责层。
- 再定义最小 happy path。
- 把复杂校验、同步和诊断降为工具能力。
- 避免一个机制重复做另一层的判断。

## 核心判断

当前 Workspace 工作流的复杂性不只来自规则多，而是多个层经常混在同一次操作里：

```text
用户目标判断
Design 需求输入
总控接收裁决
TODO / Backlog 账本
目标阶段确认
当前计划 / 任务包
窗口派发 prompt
自动化投递
子窗口 result envelope
总控证据验收
测试交接
脚本同步 / 校验
归档
```

这些层都需要存在，但不应该互相代替。

VAD 的新边界是“总控写信，VAD 送信”。Workspace 整体也应采用类似原则：

```text
总控判断目标和裁决
账本只记录状态
脚本只做机械验证 / 同步
自动化只做投递
测试交流只承载真实测试请求
Design 只产出需求设计和 handoff
```

更整体地看，Workspace 不是单一文档流程，而是一组控制资产共同构成的系统：

```text
AGENTS.md
-> governance skills / references
-> templates
-> scripts
-> .workspace-active current ledgers
-> workspace.config repositories
-> child AGENTS managed access cards
-> child window execution / result envelopes
```

优化目标不是把某一层做得更强，而是让每层只承担自己的职责，避免某一层重复做其它层的工作。

## 控制资产地图

### Parent / Control `AGENTS.md`

应承担：

- 总控身份。
- 最高停止卡。
- 仓库边界。
- 确认门禁。
- 测试与验收硬边界。
- TODO / 分派 / 自动化不可越权规则。
- skill / reference 的触发地图。

不应承担：

- 长命令手册。
- 完整模板。
- 当前 wave 事实。
- 子窗口具体任务。
- 脚本排错细节。

优化方向：`AGENTS.md` 只保留常驻硬边界和地图，不再塞满流程细节；但所有历史防错硬规则不能下沉到 skill 后消失。

### Governance Skill / References

应承担：

- TODO / Backlog 操作细则。
- window dispatch 操作细则。
- testing validation 细则。
- script pipeline 细则。
- workspace ledger / archive 细则。
- control architecture map。
- codex automation loop 操作说明。

不应承担：

- 替代总控判断。
- 独占硬停止规则。
- 写入当前计划事实。

优化方向：skill/reference 是“按场景打开的操作手册”。总控先判断当前场景，再只读对应 reference，避免每次把全部治理规则加载成默认路径。

### Templates

应承担：

- 固定章节。
- 表格形状。
- 脚本可读锚点。
- 少量硬边界提醒。

不应承担：

- 当前计划事实。
- 长 playbook。
- 某次需求的设计内容。
- 总控裁决。

优化方向：模板越小越好，只保证结构契约；复杂说明放 reference，不放模板。

### Scripts

应承担：

- dry-run / check / explicit write。
- 导入 Design handoff inbox。
- 校验 current plan / TODO / task package / test boundary。
- 同步 current plan 到 index/status。
- 归档和摘要生成。
- closed-loop contract packet / envelope / result 操作。

不应承担：

- 判断用户目标。
- 决定 TODO 优先级。
- 选择派发窗口。
- 接受验收事实。
- 改产品源码。

优化方向：脚本按能力分三类：

1. contract scripts：创建 / 检查 packet、envelope、result。
2. ledger scripts：导入、同步、归档账本。
3. diagnostic scripts：失败、清场、归档前验证。

普通 happy path 不默认跑完整 pipeline。

### `.workspace-active` / Current Ledgers

应承担：

- 当前总控计划。
- 当前状态快照。
- 活跃 TODO。
- Design inbox。
- test exchange。
- 当前可派发窗口和提示词。

不应承担：

- 长期规则。
- 产品实现文档。
- 子仓库源码说明。
- 历史全量归档。

优化方向：当前面只放“今天要判断和执行的事实”；长期账本和规则回到 `workspace-ledger` / templates / skills。

### `workspace.config.json` / `.workspace-local/workspace.config.json`

应承担：

- workspace 名称。
- control / design / test / real project window 名称。
- repositories 列表。
- windowName / path / role / managedAgents / mode。
- 本机实例覆盖。

不应承担：

- 当前任务状态。
- 派发计划。
- TODO。
- 验收结论。

优化方向：把子窗口能力配置分成通用 profile 和本机覆盖。比如 automation profile、manual / compact heartbeat 接入卡策略、是否允许 managed AGENTS 写入，都应挂在 repository config 或 profile 体系，而不是散落在脚本和文档里。

### Child Window `AGENTS.md`

应承担：

- 本仓库自己的最高停止卡和实现边界。
- Workspace managed access card。
- 窗口坐标：control workspace、window name、active index/status/current dir。
- VAD / closed-loop 最小门禁。
- 本仓库执行前必须读什么。

不应承担：

- 总控当前 wave 的完整任务书。
- 全局 TODO。
- 其它窗口任务。
- 总控验收裁决。

优化方向：子窗口 AGENTS 的 managed access card 应拆清：

- manual workspace task：人工复制提示词时的强读规则。
- compact automation heartbeat：只确认窗口、delivery/correlation、一次性消费和 prompt。
- local repository stop card：仓库自己的实现硬边界。

### Child Window Result Surface

应承担：

- TargetResultEnvelope。
- commit / diff / test / report / screenshot refs。
- 风险和未验证项。

不应承担：

- 关闭 TODO。
- 判断总控 accepted。
- 创建下一波派发。
- 验证其它窗口。

优化方向：子窗口只回证据索引和风险，总控 pull review 后再更新账本。

## Workspace 分层建议

### 1. Intent / User Goal Layer

负责：

- 判断用户真实目标。
- 判断当前是否新需求、bug、TODO、research、decision、current-mainline-risk、requirement-candidate 或背景信息。
- 判断是否打断当前主线。

不负责：

- 直接写 TODO。
- 直接派发窗口。
- 直接启动 automation。

### 2. Design Intake Layer

负责：

- 保存 Design original plan / requirement design / research / handoff。
- 明确目标、完成定义、阶段候选、风险和总控建议。

不负责：

- 修改 workspace 当前状态。
- 正式写入全局 TODO。
- 派发实现窗口。

### 3. Total-Control Decision Layer

负责：

- 接收或拒收 Design handoff。
- 判断主线影响、优先级、是否入 TODO / Backlog。
- 定义最终目标、完成定义、阶段顺序和第一阻塞点。
- 裁决验收、返工、下一波或停止。

不负责：

- 被脚本输出替代。
- 被子窗口回填替代。
- 被 TODO 自动驱动。

### 4. Ledger Layer

包括：

- Design handoff board / inbox。
- global TODO board。
- current plan。
- current status。
- test exchange。
- workspace index / record map。

负责：

- 记录总控已经裁决或正在跟踪的事实。
- 给脚本和人提供入口。

不负责：

- 自动决定下一步。
- 因为存在 TODO 或 ready handoff 就自动派发。
- 因为某状态表显示完成就自动验收。

### 5. Planning / Dispatch Layer

负责：

- 把已确认目标拆为阶段。
- 把当前阶段拆为 task packages。
- 生成目标窗口 prompt。
- 区分 send / no-send / observe / blocked。

不负责：

- 投递 heartbeat。
- 验收证据。
- 改产品代码。

### 6. Delivery Layer

包括 manual copy prompt 和 Codex Automation Closed Loop。

负责：

- 把总控已经生成的 prompt 送到目标窗口。
- 保活、用完即弃、失败回报、必要时回跳。

不负责：

- 判断目标窗口。
- 解析 current plan。
- 生成 target-specific prompt。
- 判断完成。

### 7. Result / Evidence Layer

负责：

- 子窗口返回 result envelope。
- 保存 commit / diff / command output / runtime JSON / report / screenshot refs。
- 标明风险和未验证项。

不负责：

- 把窗口自述当成验收事实。
- 直接关闭 TODO。

### 8. Review / Acceptance Layer

负责：

- 总控 pull 原始证据。
- 对照完成定义。
- 判断 accepted / rework / blocked / next-wave / stop。
- 更新 TODO / current plan / status / archive。

不负责：

- 被 result envelope 自动触发为完成。
- 被测试窗口推测替代。

### 9. Diagnostic / Governance Layer

包括：

- `verify-control-center`
- `workspace-control`
- `sync-current-plan`
- `import-design-handoffs`
- `check-dispatch-coverage`
- `check-todo-board`
- `check-test-boundary`
- `check-decision-preflight`
- full pipeline / archive scripts

负责：

- 在对应场景做机械校验、同步、导入、诊断和归档辅助。

不负责：

- 成为每次动作默认前置。
- 代替总控判断。

## Happy Path 草案

Workspace 普通路径应尽量像这样：

```text
用户提出目标 / Design handoff
-> 总控接收判断
-> 目标 + 完成定义 + 阶段确认
-> 当前阶段 task packages
-> 总控生成 target prompts
-> manual / automation delivery
-> 子窗口 result envelope
-> 总控 pull evidence review
-> 更新账本 / 下一阶段 / 归档
```

这个路径不默认运行全量 pipeline、全量 TODO 审计、全量脚本同步、TestWindow 交接或归档流程。

## 重量能力触发条件

只有在以下情况进入重量流程：

- 用户目标不清或改变完成定义。
- Design handoff 改变主线、范围、仓库边界或阶段顺序。
- TODO 影响当前派发顺序。
- 当前计划缺少完成定义、任务包、验证命令或窗口覆盖。
- 子窗口 result envelope 证据不足或与原始证据冲突。
- 需要真实项目 / cold-start / Dashboard 手动观察 / runtime monitoring。
- 脚本同步发现 current plan / index / status 不一致。
- 归档前需要证明 TODO、测试交流、回填和状态都已闭合。

## 可能的优化方向

### 0. Workspace Control Asset Map

先建立一份总控控制资产地图，记录每个规则、字段、命令和模板归属：

```ts
type WorkspaceControlAssetMap = {
  hardRules: {
    owner: "AGENTS.md";
    examples: string[];
  };
  procedures: {
    owner: "skills/dev/control-workspace-governance/references";
    trigger: string;
  }[];
  templates: {
    owner: "templates";
    contractSections: string[];
  }[];
  scripts: {
    owner: "scripts";
    class: "contract" | "ledger" | "diagnostic";
    mustNotDecide: string[];
  }[];
  childAccessProfiles: {
    owner: "workspace.config repositories + child AGENTS managed block";
    manualTask: string[];
    compactAutomation: string[];
  }[];
};
```

这一步比直接改模板或脚本更重要。先知道每条规则属于哪一层，才能避免重复和漂移。

### A. Workspace Decision Packet

总控每次正式进入工作前，先形成极小决策包：

```ts
type WorkspaceDecisionPacket = {
  decisionId: string;
  source: "user" | "design-handoff" | "window-result" | "todo" | "test-result";
  classification: string;
  userGoal: string;
  completionDefinition: string;
  currentMainlineImpact: "none" | "parallel" | "interrupt" | "blocks-current";
  nextAction: "intake" | "todo" | "plan" | "dispatch" | "review" | "archive" | "stop";
  evidenceRefs: string[];
};
```

这个 packet 属于总控判断层，不是脚本替代判断。

### B. Ledger Envelope

每个账本只收自己的 envelope：

- Design inbox：需求来源、文档路径、建议。
- TODO board：问题 / 目标、归属、优先级、触发条件。
- Current plan：当前阶段目标、任务包、窗口派发。
- Test exchange：真实测试问题、边界、多条件结论。
- Archive：已验收事实、证据 refs、关闭项。

不要让一个账本承载所有信息。

### C. Fast Path vs Diagnostic Path

总控普通动作只走 fast path：

- 读当前目标。
- 判断下一步。
- 更新一个正确账本。
- 必要时生成一个 dispatch / review / archive 动作。

诊断 path 才运行：

- full verify。
- full sync。
- full import。
- full archive pipeline。
- full test boundary audit。

### D. Prompt Builder 与 Delivery 分离

这与 VAD 结论一致：

- 总控 planning / dispatch layer 生成 prompt。
- delivery layer 只负责送达。
- 子窗口 result layer 只返回证据索引。
- review layer 才裁决。

### E. Script Surface 简化

建议将脚本分为三类：

1. **contract scripts**：只创建 / 检查结构化 packet，例如 closed-loop packet。
2. **ledger scripts**：只导入、同步、归档账本。
3. **diagnostic scripts**：只在失败、归档或治理时运行。

避免一个脚本同时承担“发现目标、决定派发、校验验收、推进状态”。

### F. Child Window Access Profile

建议把子窗口接入配置显式化：

```ts
type ChildWindowAccessProfile = {
  windowName: string;
  repositoryPath: string;
  role: string;
  managedAgents: boolean;
  mode: "external" | "internal";
  manualWorkspaceTask: {
    requireParentAgents: boolean;
    requireActiveIndex: boolean;
    requireCurrentPlan: boolean;
    requireLocalAgents: boolean;
  };
  compactAutomation: {
    acceptsDeliveryEnvelope: boolean;
    requiresCorrelationId: boolean;
    oneShot: true;
    canCreateNextDelivery: false;
  };
  testBoundary?: {
    requiresTestExchange: boolean;
    canRunRealProject: boolean;
  };
};
```

这不是让 config 变成任务状态，而是把“这个窗口如何被 workspace 触达”作为安装契约保存下来。

### G. Current Plan Surface Slimming

当前计划仍是执行主面，但可以分清三种内容：

- **Decision surface**：用户目标、完成定义、第一阻塞点、阶段判断。
- **Dispatch surface**：窗口覆盖、任务包、可复制提示词。
- **Evidence surface**：回填、原始证据 refs、验收结论。

模板应防止这三者互相覆盖。比如回填区不能自动变成验收结论；分派表不能替代目标判断。

## 风险

- 如果拆层太细，会让总控操作变成更多命令；因此必须保留短 happy path。
- 如果只做文档拆层，不改脚本和模板，实际复杂度不会下降。
- 如果弱化 AGENTS hard rules，可能重新出现总控跳判断、信任回填或 TestWindow 滥用。
- 如果把 TODO / Design inbox 自动化得太强，会重新出现“账本驱动总控”的反模式。

## 建议下一步

Design 建议先保留为 research，不立即交给总控执行。

若用户继续确认，应再拆成完整需求：

- Design Key：WORKSPACE-WORKFLOW-OPTIMIZATION-2026-05-29
- 目标：把 Workspace 总控系统按 AGENTS hard rules / skills / templates / scripts / current ledgers / workspace config / child AGENTS / result surfaces 分层，降低日常 happy path 复杂度。
- Stage 0：控制资产地图。复核现有 `AGENTS.md`、governance skills、templates、scripts、current ledgers、workspace config 和 child AGENTS managed card 的所有权。
- Stage 1：定义 WorkspaceDecisionPacket、LedgerEnvelope、ChildWindowAccessProfile。
- Stage 2：梳理脚本 fast path / diagnostic path，避免脚本替代总控判断。
- Stage 3：精简模板和当前状态入口，保证模板只保结构契约。
- Stage 4：升级子窗口 managed access card，区分 manual task 与 compact automation。
- Stage 5：用一个真实 Design handoff -> TODO -> current plan -> dispatch -> review 的小闭环验证。

## 当前不做

- 不修改总控当前状态。
- 不修改 global TODO。
- 不改脚本。
- 不创建执行 wave。
- 不打断当前 PCVM 主线。
