# FILE `.claude/commands/commit-push-pr.md` 研究文档

## 场景与职责

该文件是仓库级自定义 slash command 定义，命令名映射为 `/commit-push-pr`（文件名到命令名映射规则见 `plugins/plugin-dev/skills/plugin-structure/SKILL.md:307-310`）。其职责是把“本地代码改动 -> 提交 -> 推送 -> 建 PR”压缩成一次命令执行策略，而不是实现脚本本身。

它处于“提示词编排层”，通过 frontmatter 和正文约束驱动 Claude 调用 Git/GitHub CLI：

- 权限边界：`allowed-tools` 仅允许 `git` 与 `gh pr create`（`.claude/commands/commit-push-pr.md:2`）
- 上下文注入：执行前注入 `git status`、`git diff HEAD`、`git branch --show-current`（`:8-10`）
- 目标流程：分支检查、提交、推送、创建 PR（`:15-19`）

该命令没有对应专用 workflow 调度，主要用于交互式开发阶段；同时仓库内存在插件同名实现 `plugins/commit-commands/commands/commit-push-pr.md`，两者内容等价（仅空行差异）。

## 功能点目的

1. 降低开发者在“准备提 PR”时的工具切换成本
- 将 `git checkout --branch`、`git add/commit/push`、`gh pr create` 串成一步，避免漏步骤。

2. 提供最小权限执行面
- 通过 `allowed-tools` 把执行面限制在必要命令，避免命令执行漂移到无关工具。

3. 强制一次性完成关键链路
- 文本明确要求“单条消息内完成全部工具调用”（`.claude/commands/commit-push-pr.md:19`），目的是减少中途对话分叉导致的状态不一致。

4. 复用命令生态规范
- 使用标准 frontmatter 字段（`description`、`allowed-tools`），符合命令规范文档定义（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:24-129`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 命令协议与权限收敛

文件采用 Markdown + YAML frontmatter 协议：

```yaml
allowed-tools: Bash(git checkout --branch:*), Bash(git add:*), Bash(git status:*), Bash(git push:*), Bash(git commit:*), Bash(gh pr create:*)
description: Commit, push, and open a PR
```

关键点：

- `allowed-tools` 使用 Bash 子命令前缀过滤，限制可执行命令集合。
- 根据 frontmatter 规范，未声明的工具不可用，减少误执行面。

### 2) 运行时上下文注入

正文的 `!\`...\`` 块会在命令执行前读取实时仓库状态：

- `git status`
- `git diff HEAD`
- `git branch --show-current`

这让模型能基于当前工作区状态决定是否要先切分支、如何组织 commit。

### 3) 主流程编排（声明式，不含脚本）

正文定义的步骤是：

1. 若在 `main`，创建新分支（`:15`）
2. 创建一个 commit（`:16`）
3. 推送到 `origin`（`:17`）
4. 执行 `gh pr create`（`:18`）
5. 必须在一个响应中完成（`:19`）

该文件不包含分支命名策略、PR 模板参数、失败重试逻辑；这些均依赖模型即时决策和 CLI 默认行为。

### 4) 同源实现与文档承诺关系

- 同源命令：`plugins/commit-commands/commands/commit-push-pr.md:1-20`
- 用户文档：`plugins/commit-commands/README.md:47-89,140-146,195-203`

README 描述了“分析分支全部 commit、生成更完整 PR 描述”等能力；而 `.claude/commands/commit-push-pr.md` 文本只强制了 4 个操作步骤，未显式要求 PR 描述结构，属于“行为承诺 > 命令硬约束”的关系。

### 5) 无显式数据结构，依赖外部命令状态机

该文件本身没有 JSON/schema 数据结构。真实状态由 Git/GitHub CLI 提供：

- Git 工作区状态（untracked/staged/unstaged）
- 当前分支与远端跟踪关系
- `gh` 认证态和仓库上下文

## 关键代码路径与文件引用

- 目标文件（命令定义）
- `.claude/commands/commit-push-pr.md:1-19`

- 同源实现（插件）
- `plugins/commit-commands/commands/commit-push-pr.md:1-20`

- 用户侧行为说明
- `plugins/commit-commands/README.md:47-89`

- 命令 frontmatter 规范
- `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:7-20`
- `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-129`

- 命令发现与命名映射约定
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:307-310`
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:341-347`

- 自定义命令历史语义（变更记录）
- `CHANGELOG.md:2375`（`.claude/commands` 下 markdown 被发现为 custom slash commands）
- `CHANGELOG.md:1918`（allowed-tools Bash 权限检查修复）

- 测试现状
- 仓库未发现针对该命令的专用自动化测试脚本或 workflow（未检索到对应 `test-*` 或 `spec` 资产）。

## 依赖与外部交互

1. 本地工具依赖
- `git`：分支创建、提交、推送。
- `gh`：`gh pr create`。

2. 环境前提
- 当前目录必须是可写 Git 仓库。
- 需要存在或可配置远端 `origin`（README 也明确了此要求，`plugins/commit-commands/README.md:86-89`）。
- `gh` 需要已登录认证（`plugins/commit-commands/README.md:86-88,195-202`）。

3. 外部交互
- 通过 `gh` 与 GitHub API 交互创建 PR。
- 通过 `git push` 与远端 Git 服务器交互。

4. 权限模型
- 该命令不允许文件编辑工具，不允许网络抓取工具，只允许指定 Bash 命令子集。

## 风险、边界与改进建议

1. 风险：双源定义导致行为漂移
- 现状：`.claude/commands/commit-push-pr.md` 与 `plugins/commit-commands/commands/commit-push-pr.md` 并存。
- 影响：任意一处改动后容易产生行为不一致。
- 建议：收敛为单一来源（例如保留插件版本并由脚本同步到 `.claude/commands`）。

2. 风险：主分支名硬编码为 `main`
- 现状：正文仅写“if on main”。
- 影响：默认分支为 `master` 或其他命名时策略失效。
- 建议：改为“if on default branch”，并用 `git symbolic-ref refs/remotes/origin/HEAD` 推导默认分支。

3. 风险：无失败回滚与重试指引
- 现状：无 push/PR 失败分支流程（例如远端拒绝、认证失效、已有同名分支）。
- 建议：补充异常分支处理指令（最少覆盖 `gh` 未登录、push rejected、空提交）。

4. 风险：与 README 承诺存在粒度差
- 现状：README 声称会生成较完整 PR 描述；命令文本未硬约束。
- 建议：在命令正文明确 PR body 最小模板（Summary/Test Plan/Attribution），减少模型波动。

5. 边界：这是执行策略，不是幂等脚本
- 命令结果依赖当前仓库状态和模型决策，不保证严格可重复。

6. 改进建议（工程化）
- 增加命令静态检查：frontmatter 可解析、关键步骤关键字存在、`allowed-tools` 完整。
- 增加 dry-run 模式命令（只生成将执行的 git/gh 命令，不实际执行），降低误操作成本。
