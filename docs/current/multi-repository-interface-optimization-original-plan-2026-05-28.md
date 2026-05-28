# Multi Repository Interface Optimization 原始计划

Design Key：MULTI-REPOSITORY-INTERFACE-OPTIMIZATION-2026-05-28
日期：2026-05-28
状态：ready-for-workspace / maintenance requirement
维护窗口：AlembicDesign
总控：AlembicWorkspace
来源：用户新增非业务维护需求

## 用户原始目标

在现有 AlembicWorkspace 架构下新增一个需求，内容是多仓库之间的接口优化。该需求不新增流程架构，不重写自动化机制，只利用当前 Design handoff、总控 TODO、VAD 自动化领取和各仓库窗口回填验收的既有模式。

用户希望这类需求后续可以成为“不需要大量指导、可无人值守推进”的维护型任务来源，例如各仓库之间的接口优化。

## 判断类型

- `new-requirement`
- `maintenance`
- `cross-repository-interface`
- `automation-eligible-candidate`

## 目标候选

建立一条维护型需求主线：系统性发现并收敛 Alembic 系列仓库之间的真实接口漂移、类型不一致、client / server contract 不匹配、package export/import 边界问题、runtime artifact 同步问题和测试 fixture contract 偏差。

## 使用现有机制

该需求使用当前机制承接：

```text
AlembicDesign 需求设计
-> AlembicWorkspace 总控接收 / 入账
-> 总控按现有 TODO / VAD 机制拆任务包
-> 目标仓库窗口 claim / 修复 / 回填
-> 总控独立验收
```

不新增 dispatcher、状态机、自动化协议或新窗口职责。

## 初步范围

- `AlembicCore` contract 与 `Alembic` / `AlembicPlugin` / `AlembicAgent` 消费方对齐。
- `Alembic` HTTP route / resident response 与 `AlembicDashboard` API client 对齐。
- `AlembicPlugin` source、runtime package、skill / MCP export 与 `Alembic` / Codex host 消费方对齐。
- `AlembicAgent` provider / AI contract 与 `Alembic` internal AI 调用方对齐。
- `AlembicTest` fixture、probe、report schema 与真实产品 contract 对齐，避免测试证据偏离产品接口。

## 非目标

- 不设计新的自动化流程架构。
- 不新增业务功能。
- 不删除、迁移、降级或替换产品能力。
- 不为未来可能需要创建空 interface、空 provider、空 adapter。
- 不做大规模重构、抽象、拆仓或合仓。
- 不把没有失败证据或没有真实消费方的“接口美化”当作可执行任务。

## 建议下一步

交给总控接收为维护型需求候选。总控接收后先做代码事实调研和接口面清单，再从真实失败证据或明确 producer / consumer 漂移中拆出第一批任务包。
