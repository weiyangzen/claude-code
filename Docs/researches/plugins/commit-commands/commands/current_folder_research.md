# plugins/commit-commands/commands 目录研究（DIR）

## 场景与职责

`plugins/commit-commands/commands` 是 `commit-commands` 插件的“命令协议层”，包含 3 个 slash command 协议文件：
- `plugins/commit-commands/commands/commit.md`
- `plugins/commit-commands/commands/commit-push-pr.md`
- `plugins/commit-commands/commands/clean_gone.md`

该目录不承载可执行源码（如 TS/Python），而是通过 Markdown + frontmatter 约束 Claude 在运行时执行 Git/GitHub CLI 工作流。

调用方（上游入口）链路：
1. marketplace 注册插件目录：`.claude-plugin/marketplace.json:40-48`。
2. 插件元信息由 `plugin.json` 提供：`plugins/commit-commands/.claude-plugin/plugin.json:1-9`。
3. 插件命令按 `commands/*.md` 自动发现：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:112-115,343-345`。
4. 文件名映射为命令名（如 `commit-push-pr.md -> /commit-push-pr`）：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:307-310`。

被调用方（下游执行）链路：
- `/commit`：调用 `git status/diff/log/add/commit`（`commit.md:2,8-11,15-17`）。
- `/commit-push-pr`：调用 `git checkout --branch/add/status/commit/push` 与 `gh pr create`（`commit-push-pr.md:2,16-20`）。
- `/clean_gone`：调用 `git branch -v`、`git worktree list/remove --force`、`git branch -D`，并使用 `grep/sed/awk/while` 管道（`clean_gone.md:14,22,29-40`）。

职责边界：
- 目录内无独立 `scripts/`、无测试文件；行为正确性主要依赖 prompt contract 与工具调用结果。
- 对外部系统（GitHub、远端仓库）的可用性假设在命令层显式或隐式存在。

## 功能点目的

### 1) `/commit`
目的：把“分析改动 + 起提交信息 + 执行提交”收敛成一次命令。
- 通过内联上下文注入当前状态：`git status`、`git diff HEAD`、当前分支、最近 10 条提交（`commit.md:8-11`）。
- 以最小权限白名单限制为提交所需 git 子命令（`commit.md:2`）。
- 强制单条消息完成工具调用，减少中间对话噪音（`commit.md:17`）。

### 2) `/commit-push-pr`
目的：把“分支准备 -> 提交 -> 推送 -> 开 PR”串成单次执行。
- 在主分支上自动切出新分支（`commit-push-pr.md:16`）。
- 完成 commit + push + `gh pr create`（`commit-push-pr.md:17-19`）。
- 限定单消息完成，降低中途遗漏步骤概率（`commit-push-pr.md:20`）。

### 3) `/clean_gone`
目的：清理远端已删除但本地残留的分支（含 worktree 分支）。
- 先识别 `[gone]` 分支，再处理关联 worktree，最后删本地分支（`clean_gone.md:11-40`）。
- 兼顾普通分支和带 `+` 标记的 worktree 分支（`clean_gone.md:17,29`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 协议结构
1. 文件协议：Markdown 命令 + YAML frontmatter。
2. frontmatter 关键字段：
- `description`：命令说明（`commit.md:3`, `commit-push-pr.md:3`, `clean_gone.md:2`）。
- `allowed-tools`：工具白名单（`commit.md:2`, `commit-push-pr.md:2`）。
3. `allowed-tools` 默认可继承会话权限（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-65`）。

### B. 关键流程
1. `/commit` 流程（3 段）：
- 注入 Git 快照（status/diff/branch/log）。
- 基于快照形成单次提交意图。
- 在同一响应中执行 `git add` + `git commit`（`commit.md:15-17`）。

2. `/commit-push-pr` 流程（4+1 步）：
- 若在 `main` 则创建分支（`commit-push-pr.md:16`）。
- 生成并创建 commit（`commit-push-pr.md:17`）。
- 推送至 `origin`（`commit-push-pr.md:18`）。
- 用 `gh pr create` 建 PR（`commit-push-pr.md:19`）。
- 约束：必须在单消息中完成所有工具调用（`commit-push-pr.md:20`）。

3. `/clean_gone` 流程（列表 -> 解析 -> 删除）：
- `git branch -v` 人类可读检查（`clean_gone.md:14`）。
- `git worktree list` 获取 worktree 映射（`clean_gone.md:22`）。
- 管道提取 `[gone]` 分支名：`grep '\[gone\]' | sed 's/^[+* ]//' | awk '{print $1}'`（`clean_gone.md:29`）。
- 逐分支尝试删除关联 worktree，再 `git branch -D`（`clean_gone.md:32-39`）。

### C. 隐含数据结构（命令协议层）
1. `GitSnapshot`：`status`, `diff`, `current_branch`, `recent_commits`（来自 `!\`...\`` 注入）。
2. `CommitPlan`：是否建新分支、提交信息、是否推送与开 PR。
3. `GoneBranchCandidate`：`branch_name`, `has_worktree`, `worktree_path`, `deleted`。

### D. 关键命令与外部协议
- Git：`git add/status/diff/log/checkout/push/branch/worktree`。
- GitHub CLI：`gh pr create`。
- Shell 文本处理：`grep/sed/awk`。
- 运行期工具约束：frontmatter 白名单 + 会话权限继承机制。

## 关键代码路径与文件引用

目标目录与核心文件：
- `plugins/commit-commands/commands/commit.md:1-17`
- `plugins/commit-commands/commands/commit-push-pr.md:1-20`
- `plugins/commit-commands/commands/clean_gone.md:1-53`

上游调用/装配路径：
- `.claude-plugin/marketplace.json:40-48`（插件注册与 source 路径）。
- `plugins/commit-commands/.claude-plugin/plugin.json:1-9`（插件元信息）。
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:112-115,343-345`（`commands/*.md` 自动发现）。
- `plugins/README.md:18,55-61`（仓库级插件说明与标准结构）。

下游被调用路径：
- Git/GitHub 命令集合见 `commit.md:2,8-11`、`commit-push-pr.md:2,8-10,16-20`、`clean_gone.md:14,22,29-39`。

文档与语义来源：
- `plugins/commit-commands/README.md:11-225`（用途、需求、故障排查）。
- `plugins/plugin-dev/skills/command-development/SKILL.md:128-146,316-327`（`allowed-tools` 与 `!\`bash\`` 上下文注入语义）。
- `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-67`（frontmatter 字段语义）。

相关重复定义路径：
- `.claude/commands/commit-push-pr.md:1-19`（与插件命令近乎同构，仅存在空行差异）。

测试与脚本现状：
- `plugins/commit-commands/` 仅 5 个文件（`plugin.json`、`README.md`、3 个命令文件），无独立 `scripts/` 与测试文件。

## 依赖与外部交互

内部依赖：
1. 插件发现链路（marketplace + plugin manifest + commands 自动扫描）。
2. 命令 frontmatter 解析与权限控制机制。
3. 命令文档正文对执行步骤的强约束（尤其“单消息完成”）。

外部依赖：
1. Git CLI 可用且仓库状态正常。
2. `/commit-push-pr` 依赖 `gh` 登录态与 `origin` 远端：`plugins/commit-commands/README.md:86-89,182-183,195-202`。
3. `/clean_gone` 依赖远端追踪状态新鲜；README 建议 `git fetch --prune`：`plugins/commit-commands/README.md:204-210`。

外部交互面：
- 本地仓库写操作：`git add/commit/branch/worktree remove/branch -D`。
- 远端交互：`git push`, `gh pr create`。

配置项：
- 命令级 frontmatter：`description`、`allowed-tools`。
- 插件级元信息：`name/description/version/author`（`plugin.json`）。

## 风险、边界与改进建议

风险：
1. `/clean_gone` 未声明 `allowed-tools`（中）
- 当前继承会话权限，权限边界弱于另外两个命令（`clean_gone.md:1-3`）。

2. `clean_gone` 文本解析脆弱（中）
- 分支提取依赖 `git branch -v` 文本格式与正则匹配；复杂分支名可能误匹配 worktree（`clean_gone.md:29-33`）。

3. 强制删除风险（高）
- `git branch -D` 无合并保护，若本地仍有有价值提交存在误删风险（`clean_gone.md:39`）。

4. 文档承诺与实现约束不完全对齐（中）
- README 声称 `/commit` 包含 secrets 规避、conventional commit、attribution，但命令正文未给出可验证硬约束（`plugins/commit-commands/README.md:41-45` vs `commit.md:15-17`）。

5. 命令双源定义漂移（中）
- `.claude/commands/commit-push-pr.md` 与插件版本并存，后续易产生行为分叉。

6. 版本说明漂移（低到中）
- `CHANGELOG.md:904` 提到 `/commit-push-pr` 与 Slack/MCP 的联动，但当前两个 `commit-push-pr` 命令文件均未体现该流程，可能来自其他实现分支或历史版本。

边界：
1. 本目录是“提示协议层”，不负责安装、认证、权限申请与 UI。
2. 无独立单元测试/集成测试载体，传统测试覆盖指标不适用。
3. 命令质量高度依赖运行时模型遵循度与环境状态（Git/gh）。

改进建议：
1. 给 `/clean_gone` 增加最小化 `allowed-tools` 白名单，和其余命令保持一致的最小权限策略。
2. 把 `clean_gone` 的分支解析改为更稳健接口（例如 `git for-each-ref` + 明确字段），减少 `grep/sed/awk` 解析歧义。
3. 将 `git branch -D` 改为两阶段策略（先 `-d`，失败再二次确认 `-D`）并输出风险提示。
4. 明确统一 `/commit-push-pr` 的单一来源（插件目录或 `.claude/commands` 二选一）。
5. 增加最小回归校验脚本（frontmatter 语法、关键步骤存在性、`allowed-tools` 完整性），可参考 `plugins/plugin-dev/skills/command-development/references/testing-strategies.md:11-124` 的结构化检查思路。
