# AlembicDesign Workspace Handoff Board

更新日期：2026-05-28
维护窗口：AlembicDesign
接收窗口：AlembicWorkspace

## 定位

本文件是 AlembicDesign 自己维护的正式交接清单。Design 完成原始计划、需求设计、用户确认和 TODO / Backlog 挂载建议后，把条目登记到这里；AlembicWorkspace 通过 `scripts/import-design-handoffs.mjs` 自动发现并生成总控收件箱。

本清单不是 workspace 全局 TODO，也不是执行计划。总控脚本只能自动发现、校验和汇总；是否正式入账、是否打断当前主线、是否进入目标阶段确认或 wave，仍由 AlembicWorkspace 裁决。

## 状态枚举

- `draft`：Design 仍在整理，暂不交给总控。
- `ready-for-workspace`：Design 已完成交接准备，等待总控接收。
- `accepted-by-workspace`：总控已正式接收并转入 workspace 账本。
- `needs-design`：总控或用户要求 Design 继续补充。
- `paused`：用户或 Design 暂停。
- `archived`：已归档或不再需要总控接收。

## Handoff 清单

| ID | 状态 | 标题 | 原始计划 | 需求设计 | Handoff | 用户确认 | 当前主线关系 | 建议 TODO | 优先级 | 下一步 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| PCVM-2026-05-25 | ready-for-workspace | Progressive Chain Validation Metrics | [original-plan](progressive-chain-validation-metrics-original-plan-2026-05-25.md) | [requirement-design](progressive-chain-validation-metrics-requirement-design-2026-05-25.md) | 未单独生成；需求设计已包含交接信息 | 用户已确认 | 下一主线候选；不打断当前 LLM 输入优化 Test-08 | 建议并入 `GTODO-2026-05-25-003`，当前主线完成后由总控做代码事实调研和目标阶段确认 | P0-after-current | 总控接收评审；正式入账后补代码事实调研，不直接派发实现窗口 |
| ARTIFACT-DRAWER-2026-05-25 | accepted-by-workspace | Timeline Artifact Recipe Drawer Optimization | [original-plan](timeline-artifact-recipe-drawer-optimization-original-plan-2026-05-25.md) | [requirement-design](timeline-artifact-recipe-drawer-optimization-requirement-design-2026-05-25.md) | 未单独生成；需求设计已包含交接信息 | 用户已确认：复用既有 Recipe evolution 双层抽屉、只做窄屏适配、保留通用样式、需要返回按钮、希望立即启动 | 独立 Dashboard UI 优化候选；不混入 PCV metrics 或当前 LLM 输入优化 Wave 5；用户认为可并行 | 已转入 `GTODO-2026-05-25-004` 和 `docs/workspace/current/artifact-drawer-parallel-dispatch-2026-05-25.md` | P0-now-if-no-conflict | 总控已接收并发送给 `AlembicDashboard`；Design 无需继续补充 |
| KNOWLEDGE-EVOLUTION-TODOS-2026-05-26 | draft | Knowledge Evolution TODOs Sequence Index | [original-plan](knowledge-evolution-todos-original-plan-2026-05-26.md) | [requirement-design](knowledge-evolution-todos-requirement-design-2026-05-26.md) | 未单独生成 | 用户纠偏：不是只整理自动化领取字段，而是按 037/038/039 顺序逐个讨论需求；037 已完成；2026-05-28 新增 AI mock 删除插队需求 | 顺序讨论索引；不直接交给总控执行；当前建议顺序为 AI mock 删除 -> 037 收敛 -> 038 -> 039 | 仅作为连续需求索引；当前可交给总控的是 `AI-MOCK-REMOVAL-2026-05-28`，038/039 暂不 ready | P1-sequence-index | 不由总控直接接收执行；总控应优先评审 `AI-MOCK-REMOVAL-2026-05-28` |
| AI-MOCK-REMOVAL-2026-05-28 | ready-for-workspace | AI Mock Removal | [original-plan](ai-mock-removal-original-plan-2026-05-28.md) | [requirement-design](ai-mock-removal-requirement-design-2026-05-28.md) | 未单独生成；需求设计已包含交接信息 | 用户已确认按 Design 推荐：删除产品 runtime AI mock，覆盖 AlembicAgent 产品导出 / registry / factory fallback；test-local fake / fixture 可保留但必须隔离；历史数据 cleanup 由 Stage 0 查证；Dashboard mock UI/API 优先直接删除 | 037 收敛、038、039 之前的前置清理；避免后续 runtime / cold-start / evolution 测试继续被 mock path 污染 | 建议总控接收为新的最前置需求，先做跨仓库代码事实调研和删除边界确认 | P0-before-037-convergence | 可交给总控领取；总控先做 Stage 0 代码事实调研和目标阶段确认，再决定 deletion wave |
| INTENT-KNOWLEDGE-CONSOLIDATION-2026-05-28 | ready-for-workspace | Intent Knowledge Evolution Baseline | [original-plan](intent-knowledge-consolidation-original-plan-2026-05-28.md) | [requirement-design](intent-knowledge-consolidation-requirement-design-2026-05-28.md) | 未单独生成；需求设计已包含交接信息 | 用户确认命名使用 `IntentKnowledgeEvolutionBaseline`，先把该收敛需求设计完整 | intent knowledge route 已归档后的基线收敛项；排在 `AI-MOCK-REMOVAL-2026-05-28` 之后，用于固定 038/039 可消费契约、不可继承假设和风险归口 | 建议总控在 AI mock 删除边界确认后接收；先做轻量代码事实 / 证据复核，再写入 baseline | P1-after-ai-mock-removal | 等 AI mock 删除需求被总控接收并确认边界后领取；不要跳过 baseline 直接派 038/039 |
| MULTI-REPOSITORY-INTERFACE-OPTIMIZATION-2026-05-28 | ready-for-workspace | Multi Repository Interface Optimization | [original-plan](multi-repository-interface-optimization-original-plan-2026-05-28.md) | [requirement-design](multi-repository-interface-optimization-requirement-design-2026-05-28.md) | 未单独生成；需求设计已包含交接信息 | 用户确认：不用新增流程架构，使用现有机制新增一个需求，内容是多仓库接口优化 | 非业务维护型需求；不打断 AI mock 删除、baseline、038/039；适合作为后续自动化 backlog 候选 | 建议总控接收后先做 Stage 0 producer / consumer 接口事实清单，再选择有失败证据的最小接口修复任务 | P2-maintenance-backlog | 总控接收评审；先做代码事实清单，不直接派实现；可在主线间隙或自动化模式中领取低风险任务 |
| INTENT-RECOGNITION-2026-05-26 | archived | Intent Recognition Episode Continuity | [original-plan](intent-recognition-episode-continuity-original-plan-2026-05-26.md) | [requirement-design](intent-recognition-episode-continuity-requirement-design-2026-05-26.md) | 未单独生成；需求设计已包含交接信息 | 用户已确认：不升级旧逻辑、prime 不等联网 AI、Codex 不强制传 intent、低置信度可不强注入、本地千问可作快速 refinement、多会话连续 | `GTODO-2026-05-24-037` 已由总控完成归档 | 已归档，不再作为当前新交接项 | P1-first-in-sequence | 无需 Design 继续补充；后续只通过 037 收敛需求复用事实 |
| INTENT-KNOWLEDGE-2026-05-26 | archived | Plugin Intent Knowledge Route | [original-plan](plugin-intent-knowledge-route-original-plan-2026-05-26.md) | [requirement-design](plugin-intent-knowledge-route-requirement-design-2026-05-26.md) | 未单独生成；需求设计已包含交接信息 | 用户已确认：不是替代识别目标，识别与消费都要做；prime 交付包需汇总意图、搜索、向量、关联关系，并保留 Recipe 源路径 | `GTODO-2026-05-24-037` 已由总控完成归档 | 已归档，不再作为当前新交接项 | P1-after-recognition | 无需 Design 继续补充；后续只通过 037 收敛需求复用事实 |

## 使用规则

- `ready-for-workspace` 条目必须有原始计划、需求设计、用户确认、当前主线关系、建议 TODO 和下一步。
- Handoff 文档可选；如果需求设计已经包含足够的交接信息，可以在 `Handoff` 列说明原因。
- 清单中可以出现多个 ready 条目，但总控只能按当前主线、优先级、依赖和目标阶段确认逐个接收。
- Design 不直接修改 `docs/workspace/current/global-todo-board.md`，也不直接创建 wave。
