# Repository Residue Cleanup Requirement Design

- Design Key：`REPOSITORY-RESIDUE-CLEANUP-2026-05-31`
- 类型：bug / current-mainline-risk / requirement-candidate
- 状态：ready-for-workspace
- 证据状态：已完成只读代码事实检查；已先行完成可安全清理和最小修复；等待总控接收复核
- 维护窗口：AlembicDesign
- 建议接收窗口：AlembicWorkspace
- 日期：2026-05-31

## 用户目标

多个子仓库根目录出现错误的多余文件夹和内容，例如 `Alembic/.asd`、`Alembic/.cursor/skills`、`AlembicPlugin/.asd`、真实项目里的 `.agents/skills`。目标不是把这些目录简单加入 ignore，而是找出哪里把运行态 / 投影态内容写进源码仓库，清理当前污染，并建立可重复验证的边界守卫。

## 完成定义

- 已识别当前 workspace 配置内的源码仓库残留目录。
- 已区分合法产品资产与污染物：例如 `AlembicPlugin/.agents/plugins/marketplace.json` 是 tracked marketplace 资产，不应误删。
- 已清理未被 git 跟踪的运行态 / 投影态污染物。
- 已补 workspace 级 residue 检查，后续总控验证能发现 `.asd/`、`.cursor/skills`、`.agents/skills` 等残留。
- 已补至少一个产品根因修复，避免 Alembic 源码仓库继续因 Logger 初始化生成 `.asd/logs`。
- 已留下验证命令和结果，方便总控接收。

## 代码事实

1. 当前残留复现：
   - `node scripts/check-repository-residue.mjs` 在修复前发现 5 个残留：
     - `Alembic/.asd`
     - `Alembic/.cursor/skills`
     - `AlembicPlugin/.asd`
     - `BiliDili/.agents/skills`
     - `BiliDili/.agents/.DS_Store`
   - 这些路径均未被对应仓库 git 跟踪，可安全作为本地生成污染物清理。

2. 合法例外：
   - `AlembicPlugin/.agents/plugins/marketplace.json` 被 git 跟踪，来自 Plugin marketplace 打包资产。
   - `codex-lark-remote/.agents/plugins/marketplace.json` 同样是 tracked marketplace 资产，且不属于 Alembic workspace 配置内的本轮治理对象。
   - 因此 `.agents` 不能整体一刀切删除；应只处理 `.agents/skills` 这类运行时 skill projection 残留，或未跟踪的本地噪音。

3. 根因链路：
   - `AlembicCore/src/infrastructure/logging/Logger.ts` 原逻辑在 `config.file.path || './.asd/logs'` 下直接 `mkdirSync`，没有调用 `PathGuard.assertProjectWriteSafe`。
   - `DatabaseConnection` 已有排除项目保护：Alembic 源码仓库会把 DB 重定向到 `/tmp/alembic-dev/alembic.db`，说明源码仓库不应落 `.asd` 是既有设计。
   - `SetupService` 已拒绝在 Alembic 自身源码仓库做 standard setup，提示使用 ghost；这也说明 `.asd` 出现在源码仓库根下违反运行态边界。
   - `HnswVectorAdapter` 当前已经通过 `PathGuard` 创建 context index；本轮看到的空 `context/index` 更像历史残留或旧链路产物。
   - `ProjectSkillDelivery` 的 `.agents/skills` 是 Codex project skill runtime export，代码要求授权；真实项目 `BiliDili/.agents/skills` 没有当前计划授权，按 workspace 污染处理。

## 已完成修复

- 新增 `codex-control-workspace/scripts/check-repository-residue.mjs`：
  - 默认只读扫描 workspace 配置内 repositories。
  - 识别 `.asd`、`.cursor/skills`、`.agents/skills`、`.agents/.DS_Store`。
  - 跳过不存在的 repo，不误删 tracked 文件。
  - `--fix` 只清理未被 git 跟踪的残留，并删除因此变空的 `.cursor` / `.agents` 父目录。
  - 支持 `allowedRepositoryResiduePaths` 和 repo 局部 `allowedResiduePaths` 作为显式例外。

- 验证接入：
  - `verify-control-center.mjs` 默认加入 `repository residue` 检查。
  - `--with-script-tests` 加入 `check-repository-residue.test.mjs`。
  - `scripts/README.md` 登记新脚本和测试。

- 产品根因修复：
  - `AlembicCore/src/infrastructure/logging/Logger.ts` 在文件日志路径位于源码 projectRoot 内、且 `PathGuard` 判定不允许项目写入时，将日志重定向到 `/tmp/alembic-dev/logs`。
  - 同步修复 `Alembic/vendor/AlembicCore/src/infrastructure/logging/Logger.ts` 和 `AlembicPlugin/vendor/AlembicCore/src/infrastructure/logging/Logger.ts`。
  - 新增 `AlembicCore/test/LoggerRuntimeBoundary.test.ts` 覆盖 excluded Alembic source repo 不创建 `.asd`。

## 已清理残留

- `Alembic/.asd`
- `Alembic/.cursor/skills`，并清理空 `.cursor`
- `AlembicPlugin/.asd`
- `BiliDili/.agents/skills`
- `BiliDili/.agents/.DS_Store`，并清理空 `.agents`

## 验证结果

- `node scripts/check-repository-residue.mjs --fix`
  - 修复前 5 个 residue，修复后 blocking 0。
- `node scripts/check-repository-residue.mjs`
  - `Residue entries: 0`
  - `Blocking entries: 0`
- `node --test scripts/check-repository-residue.test.mjs`
  - 2 tests passed。
- `node scripts/check-script-docs.mjs`
  - passed。
- `npm test -- test/LoggerRuntimeBoundary.test.ts` in `AlembicCore`
  - 1 test file passed，1 test passed。
- `node scripts/verify-control-center.mjs --with-script-tests`
  - residue check / script docs / current plan sync / current layout / test boundary / TODO board / git diff whitespace passed。
  - 整体验证未通过，失败来自既有当前计划状态：缺失 `pcvm-workspace-plan-generation-2026-05-30.md` 链接、当前计划缺 `总控决策记录`、dispatch coverage / task package 校验失败，以及 `workspace-control.test` 读取真实当前计划后继承该失败。该失败不是本轮 residue 守卫或 Logger 修复引入。

## 建议总控下一步

1. 接收 `REPOSITORY-RESIDUE-CLEANUP-2026-05-31` 为 bugfix / workspace hygiene 项。
2. 复核本轮改动涉及的仓库状态：`codex-control-workspace`、`AlembicCore`、`Alembic`、`AlembicPlugin`、`AlembicDesign`。
3. 决定是否把 residue 检查作为后续总控常规验证门禁保留；Design 推荐保留默认检查。
4. 若后续确实要允许某个项目使用 `.agents/skills`，必须由当前计划显式写入 `allowedResiduePaths` 或等价授权，不应靠本地残留静默存在。

## 仍需确认的问题

- 总控是否接受 Design 本轮先行修复产品根因，还是需要拆成正式验收任务由对应源码窗口复核提交。
- `BiliDili` 作为真实测试项目，后续是否允许任何 `.agents/skills` runtime projection；Design 推荐默认不允许，除非当前测试计划显式授权。
