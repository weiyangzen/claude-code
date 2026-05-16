# plugins/commit-commands/commands/commit-push-pr.md 研究

## 场景与职责

`plugins/commit-commands/commands/commit-push-pr.md` 是 `commit-commands` 插件里的“发布前一键链路命令”，职责是把“提交 -> 推送 -> 创建 PR”整合成单次命令协议（`plugins/commit-commands/commands/commit-push-pr.md:1-20`）。

职责分层如下：

- 上游调用方
- marketplace 注册 `commit-commands` 插件来源（`.claude-plugin/marketplace.json:40-48`）。
- 插件启用后按命令命名约定自动暴露 `/commit-push-pr`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:307-310,341-345`）。
- 终端用户在“准备发起 PR”阶段触发命令（`plugins/commit-commands/README.md:47-57,140-145`）。
- 下游被调用方
- Git：`checkout --branch`、`add`、`commit`、`push`（`plugins/commit-commands/commands/commit-push-pr.md:2,16-18`）。
- GitHub CLI：`gh pr create`（`plugins/commit-commands/commands/commit-push-pr.md:2,19`）。

它不是插件总说明文档，用户预期与前置要求主要写在 `plugins/commit-commands/README.md`（`plugins/commit-commands/README.md:49-57,77-89,195-202`）。

## 功能点目的

1. 减少上下文切换  
把多条 Git/GitHub CLI 操作收敛为一次命令触发，降低手工串联成本（`plugins/commit-commands/README.md:7,49-57`）。

2. 规范“从改动到 PR”的最小闭环  
协议固定为“必要时建分支 -> 单次提交 -> 推送 -> 建 PR”，避免漏步骤（`plugins/commit-commands/commands/commit-push-pr.md:16-19`）。

3. 强制原子执行风格  
要求在单条消息中完成所有工具调用，避免执行中途被额外交互打断（`plugins/commit-commands/commands/commit-push-pr.md:20`）。

4. 提供即时报表上下文  
先注入 `status/diff/current branch`，让提交消息和 PR 行为基于当前仓库状态生成（`plugins/commit-commands/commands/commit-push-pr.md:8-10`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) Frontmatter 配置

- `allowed-tools` 白名单限定为：
- `Bash(git checkout --branch:*)`
- `Bash(git add:*)`
- `Bash(git status:*)`
- `Bash(git push:*)`
- `Bash(git commit:*)`
- `Bash(gh pr create:*)`  
（`plugins/commit-commands/commands/commit-push-pr.md:2`）
- `description` 为命令摘要（`plugins/commit-commands/commands/commit-push-pr.md:3`）。

`allowed-tools` 的字段语义是“覆盖/收窄可调用工具集合”，用于最小权限执行（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-67,124-127`）。

### 2) 上下文注入与执行协议

- 使用 `!`command`` 内联注入运行时上下文（`plugins/commit-commands/commands/commit-push-pr.md:8-10`），这属于命令系统支持的 bash 输出注入机制（`plugins/plugin-dev/skills/command-development/README.md:145-147`）。
- 任务协议要求 4 个动作必须全部完成，且“单消息、仅工具调用、无额外文本”（`plugins/commit-commands/commands/commit-push-pr.md:16-20`）。

### 3) 关键流程

1. 分支决策  
仅在当前位于 `main` 时创建新分支（`plugins/commit-commands/commands/commit-push-pr.md:16`）。

2. 提交创建  
对当前改动生成单次 commit（`plugins/commit-commands/commands/commit-push-pr.md:17`）。

3. 远端同步  
将当前分支推送到 `origin`（`plugins/commit-commands/commands/commit-push-pr.md:18`）。

4. PR 创建  
调用 `gh pr create` 完成 Pull Request（`plugins/commit-commands/commands/commit-push-pr.md:19`）。

### 4) 隐式数据结构（协议层）

- `WorkingTreeSnapshot`
- 字段语义：`status`、`diff`、`current_branch`。
- 来源：3 条上下文注入命令（`plugins/commit-commands/commands/commit-push-pr.md:8-10`）。

- `BranchActionPlan`
- 字段语义：`need_new_branch`、`branch_name`、`push_remote`。
- 来源：第 16、18 条步骤定义（`plugins/commit-commands/commands/commit-push-pr.md:16,18`）。

- `PRCreationIntent`
- 字段语义：`commit_message`、`pr_title/body`、`target_repo`（由 `gh` 上下文推导）。
- 来源：第 17、19 条步骤定义（`plugins/commit-commands/commands/commit-push-pr.md:17,19`）。

## 关键代码路径与文件引用

### 命令实现主路径

- `plugins/commit-commands/commands/commit-push-pr.md:1-20`  
定义工具白名单、上下文输入与执行约束。

### 调用方（上游）路径

- `.claude-plugin/marketplace.json:40-48`  
将插件暴露给插件系统。
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:307-310,341-345`  
定义命令命名映射和自动扫描流程。
- `plugins/commit-commands/README.md:47-57,140-145`  
定义此命令在用户工作流中的定位。

### 被调用方（下游）路径

- `plugins/commit-commands/commands/commit-push-pr.md:2,16-19`  
实际触达的 Git/GitHub CLI 命令面。

### 配置/文档/测试/脚本路径

- 配置
- `plugins/commit-commands/.claude-plugin/plugin.json:1-9`（插件元数据）。
- 文档
- `plugins/commit-commands/README.md:77-89,195-202`（特性、依赖、故障排查）。
- 测试
- `plugins/commit-commands` 下未提供该命令专属自动化测试。
- 脚本
- 无独立脚本文件；流程逻辑直接写在命令 Markdown（`plugins/commit-commands/commands/commit-push-pr.md:14-20`）。
- 并行来源
- `.claude/commands/commit-push-pr.md:1-19` 存在同名近同构命令，可能形成双源维护。

## 依赖与外部交互

### 内部依赖

- 插件发现链与命令扫描机制（`.claude-plugin/marketplace.json:40-48`，`plugins/plugin-dev/skills/plugin-structure/SKILL.md:341-345`）。
- frontmatter 工具控制机制（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-67`）。

### 外部交互

- 本地 Git 写操作：分支创建、提交、推送（`plugins/commit-commands/commands/commit-push-pr.md:2,16-18`）。
- GitHub API 交互：`gh pr create` 需要网络与认证（`plugins/commit-commands/commands/commit-push-pr.md:2,19`，`plugins/commit-commands/README.md:86-89,195-202`）。
- 远端仓库约束：默认假设 remote 名为 `origin`（`plugins/commit-commands/commands/commit-push-pr.md:18`，`plugins/commit-commands/README.md:88`）。

## 风险、边界与改进建议

1. 工具白名单与上下文命令不一致  
正文使用了 `git diff HEAD` 与 `git branch --show-current` 作为上下文输入，但 `allowed-tools` 未包含这两个子命令，存在权限配置与协议文本不一致风险（`plugins/commit-commands/commands/commit-push-pr.md:2,9-10`）。

2. 分支策略过于单一  
仅在 `main` 上分支，未覆盖 `master`/`trunk`/自定义默认分支场景（`plugins/commit-commands/commands/commit-push-pr.md:16`）。

3. 缺少失败补偿路径  
若“推送成功但 PR 创建失败”，当前协议未定义回滚或补救步骤，可能留下半完成状态（`plugins/commit-commands/commands/commit-push-pr.md:18-20`）。

4. 单提交策略对复杂改动不友好  
命令强制单次 commit，无法表达多逻辑块拆分提交，影响审查粒度（`plugins/commit-commands/commands/commit-push-pr.md:17`）。

5. 多来源漂移风险  
仓库同时存在 `.claude/commands/commit-push-pr.md` 与插件版，更新时容易行为分叉（`.claude/commands/commit-push-pr.md:1-19`，`plugins/commit-commands/commands/commit-push-pr.md:1-20`）。

6. 文档/变更日志可能已出现实现偏差  
`CHANGELOG.md` 记录过“自动把 PR URL 发 Slack（经 MCP）”的能力，但当前命令文件未体现对应动作（`CHANGELOG.md:904`，`plugins/commit-commands/commands/commit-push-pr.md:16-20`）。

改进建议：

1. 补齐 `allowed-tools` 与上下文命令一致性  
加入 `Bash(git diff:*)` 与 `Bash(git branch:*)`，避免权限策略与协议文本冲突。

2. 默认分支策略参数化  
基于 `git symbolic-ref refs/remotes/origin/HEAD` 或配置项判定默认分支，不硬编码 `main`。

3. 增加失败分支与补偿指令  
明确 `gh pr create` 失败时输出后续操作建议（如重试、指定 `--base/--head`、补充权限检查）。

4. 统一命令单一来源  
在 `.claude/commands` 与 `plugins/commit-commands/commands` 中保留一个主版本，另一个由脚本同步或显式废弃。
