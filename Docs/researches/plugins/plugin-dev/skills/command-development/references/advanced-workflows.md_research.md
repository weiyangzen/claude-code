# advanced-workflows.md 研究

## 场景与职责

`plugins/plugin-dev/skills/command-development/references/advanced-workflows.md` 是 command-development references 中的“工作流编排层”。它解决单条命令无法可靠承载的场景：多阶段执行、跨命令状态共享、中断恢复、失败回滚、并发互斥。

它在体系中的定位：

1. 对 `SKILL.md` 的命令基础能力做“跨命令扩展”。
2. 与 `plugin-settings` 的 `.local.md` 配置模式互补（状态持久化）。
3. 与 `plugin-features-reference.md` 的多脚本模式形成上下游：后者强调组件协同，本文件强调流程可靠性。

## 功能点目的

1. 多步骤可视化执行（`11-170`）
- 用编号步骤和决策点提升复杂命令可理解性。

2. 跨命令状态延续（`64-127,281-360`）
- 将上下文写入 `.claude/*.local.md`，让 `/deploy-init -> /deploy-test -> /deploy-build` 连续可恢复。

3. 组合编排（`171-277`）
- 支持链式命令、流水线命令、并行验证。

4. 协调与并发控制（`361-448`）
- flag 文件跨命令通信，lock 文件避免并发部署。

5. 容错恢复（`517-602`）
- 失败时给出恢复选项，支持 rollback 与 checkpoint resume。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 工作流状态数据结构

状态文件示例（`286-313`）采用 YAML frontmatter + markdown body：

1. 前台字段：`workflow/stage/environment/branch/commit/tests_passed/build_complete`。
2. 文本区承载“已完成/待完成步骤”给人类阅读。

这与 `plugin-settings` 的 frontmatter 解析方法一致：
- `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh:37-58`

### 2) 关键流程协议

1. 初始化：创建状态并记录分支、commit、时间戳（`74-104,639-666`）。
2. 阶段推进：后续命令读取状态并更新 stage（`106-120,668-706`）。
3. 完成清理：删除状态文件（`708-720`）。

### 3) 并发与协调协议

1. 互斥锁：`.claude/deployment.lock`（`418-446`）。
2. 信号文件：`.claude/feature-complete.flag` 驱动后续命令行为（`373-401`）。
3. Checkpoint 日志：`.claude/deployment-checkpoints.log` 按阶段追加（`588-601`）。

### 4) 错误恢复协议

1. Graceful Failure：失败后给用户选择继续策略（`530-548`）。
2. Rollback：失败触发 `rollback.sh` 回退（`557-574`）。
3. Resume：按最后 checkpoint 恢复（`579-602`）。

## 关键代码路径与文件引用

核心文件：

1. `plugins/plugin-dev/skills/command-development/references/advanced-workflows.md:1-722`

关联调用方：

1. `plugins/plugin-dev/skills/command-development/README.md:103`
2. `Docs/researches/plugins/plugin-dev/skills/command-development/references/current_folder_research.md:51-55,101-103,120-121`
3. `plugins/plugin-dev/skills/plugin-settings/current_folder_research.md:193`（引用该文状态模式）

关键被调用方/支撑文件：

1. `plugins/plugin-dev/skills/plugin-settings/SKILL.md:11-19,60-171`
2. `plugins/plugin-dev/skills/plugin-settings/scripts/parse-frontmatter.sh:37-58`
3. `plugins/plugin-dev/commands/create-plugin.md:221-225`（settings 文件创建流程）

## 依赖与外部交互

1. 依赖工具：`Read/Write/Bash`，并在示例中调用 `git/gh/npm`。
2. 依赖本地文件系统协议：`.claude/*.local.md`、`.lock`、`.flag`、`.log`。
3. 依赖外部脚本占位：`deploy.sh`、`rollback.sh`、`validate.sh`、`build.sh`（文档示例中出现，但仓库并未提供这些脚本实现）。
4. 与用户交互依赖明确 decision points（yes/no、恢复选项）。

## 风险、边界与改进建议

### 风险

1. 锁文件没有 TTL/owner 元数据，异常退出后容易残留僵尸锁。
2. 状态写入示例未落实“原子写”，与文档建议 `Atomic updates`（`624`）存在落差。
3. `checkpoint` 追加日志缺少去重/一致性校验，恢复点可能失真。
4. 多数恢复/回滚依赖外部脚本占位，直接复制模板会“看起来可恢复、实际不可恢复”。

### 边界

1. 本文件提供流程模式，不提供统一状态机库。
2. 不负责 hook 生命周期与 session 重载问题。

### 改进建议

1. 提供官方 workflow helper 脚本：原子写状态、锁管理、checkpoint 读写。
2. lock 文件建议写入 `pid/session/start_time` 并支持强制接管策略。
3. 增加“状态 schema 校验”步骤，避免手工编辑破坏恢复。
4. 给 rollback/checkpoint 增加最小可运行示例脚本，降低模板误用风险。
