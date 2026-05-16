# plugins/commit-commands/commands/commit.md 研究

## 场景与职责

`plugins/commit-commands/commands/commit.md` 是 `commit-commands` 插件中最基础的日常提交命令协议，职责是把“分析改动 + 生成提交信息 + 执行提交”压缩为单次 `/commit` 执行（`plugins/commit-commands/commands/commit.md:1-17`）。

它在系统中的位置是：

- 上游调用方
- `commit-commands` 插件由 marketplace 注册并发现（`.claude-plugin/marketplace.json:40-48`）。
- 命令目录自动扫描把 `commit.md` 暴露为 `/commit`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:307-310,341-345`）。
- 用户在日常开发中触发该命令（`plugins/commit-commands/README.md:11-22,134-139`）。
- 下游被调用方
- Git 本地操作：`git add`、`git commit`，并读取 `git status/diff/branch/log` 上下文（`plugins/commit-commands/commands/commit.md:2,8-11,15-17`）。

该文件不负责插件级介绍和安装说明，这些由 `plugins/commit-commands/README.md` 承担（`plugins/commit-commands/README.md:5-10,128-183`）。

## 功能点目的

1. 快速完成小步提交  
把“查看状态、拟定 message、stage、commit”合成一次命令，减少手工命令切换（`plugins/commit-commands/README.md:7,15-22`）。

2. 基于仓库历史自动拟合提交风格  
命令注入最近 10 条提交作为上下文，目的是让 commit message 更贴近仓库既有习惯（`plugins/commit-commands/commands/commit.md:11`）。

3. 降低中间对话噪音  
协议要求“单条消息完成 stage+commit 且仅输出工具调用”，保证执行路径简洁（`plugins/commit-commands/commands/commit.md:17`）。

4. 维持最小命令面  
frontmatter 将可执行工具面压缩为 `git add/status/commit`，目标是限制无关操作（`plugins/commit-commands/commands/commit.md:2`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) Frontmatter 配置

- `allowed-tools: Bash(git add:*), Bash(git status:*), Bash(git commit:*)`（`plugins/commit-commands/commands/commit.md:2`）。
- `description: Create a git commit`（`plugins/commit-commands/commands/commit.md:3`）。

`allowed-tools` 是命令级权限收敛字段，缺省时会继承会话权限（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-67`）。

### 2) 上下文注入机制

命令正文通过 `!`command`` 注入四类运行时数据：

- `git status`
- `git diff HEAD`
- `git branch --show-current`
- `git log --oneline -10`  
（`plugins/commit-commands/commands/commit.md:8-11`）

这属于命令系统提供的“bash 执行并回填输出”模式（`plugins/plugin-dev/skills/command-development/README.md:145-147`）。

### 3) 执行协议

- 核心目标：基于上述上下文创建“单个 commit”（`plugins/commit-commands/commands/commit.md:15`）。
- 执行约束：一次消息里完成 stage 和 commit，不允许额外文本（`plugins/commit-commands/commands/commit.md:17`）。

### 4) 隐式数据结构（协议层）

- `GitSnapshot`
- 字段语义：`status`、`diff`、`current_branch`、`recent_commits`。
- 来源：4 条上下文注入命令（`plugins/commit-commands/commands/commit.md:8-11`）。

- `CommitIntent`
- 字段语义：`summary`、`scope`、`message`（由模型根据 `GitSnapshot` 推导）。
- 来源：任务语句“Based on the above changes”与提交动作（`plugins/commit-commands/commands/commit.md:15`）。

- `StagePlan`
- 字段语义：待 stage 文件集合与执行顺序。
- 来源：单消息 stage+commit 约束（`plugins/commit-commands/commands/commit.md:17`）。

## 关键代码路径与文件引用

### 命令实现主路径

- `plugins/commit-commands/commands/commit.md:1-17`  
定义 `/commit` 的权限、上下文输入、执行约束。

### 调用方（上游）路径

- `.claude-plugin/marketplace.json:40-48`  
注册插件来源。
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:307-310,341-345`  
定义命令命名与发现机制。
- `plugins/commit-commands/README.md:11-22,134-139`  
定义命令目标与使用场景。

### 被调用方（下游）路径

- `plugins/commit-commands/commands/commit.md:2,8-11,15-17`  
调用 git 上下文命令和写操作命令。

### 配置/文档/测试/脚本路径

- 配置
- `plugins/commit-commands/.claude-plugin/plugin.json:1-9`（插件元信息）。
- 文档
- `plugins/commit-commands/README.md:41-46,187-194`（功能宣称、常见问题）。
- 测试
- 插件目录未提供该命令专属自动化测试用例。
- 脚本
- 无独立脚本文件；逻辑全部内联于命令 Markdown（`plugins/commit-commands/commands/commit.md:6-17`）。

## 依赖与外部交互

### 内部依赖

- 插件注册与发现机制（`.claude-plugin/marketplace.json:40-48`，`plugins/plugin-dev/skills/plugin-structure/SKILL.md:341-345`）。
- frontmatter 约束语义（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-67,124-127`）。

### 外部交互

- 仅依赖本地 Git 仓库环境：
- 读取状态：`status/diff/branch/log`（`plugins/commit-commands/commands/commit.md:8-11`）。
- 写入提交：`git add` + `git commit`（`plugins/commit-commands/commands/commit.md:2,17`）。

与 `/commit-push-pr` 不同，该命令不直接依赖 GitHub 网络调用与 `gh` CLI。

## 风险、边界与改进建议

1. 权限声明与上下文命令存在不一致  
`allowed-tools` 未包含 `git diff`、`git branch`、`git log`，但正文使用了这些命令进行上下文注入（`plugins/commit-commands/commands/commit.md:2,9-11`）。

2. README 承诺与协议细节存在落差  
README 声称会规避 secrets、遵循 conventional commit、附带 attribution（`plugins/commit-commands/README.md:41-46`），但命令正文未提供显式校验/过滤步骤（`plugins/commit-commands/commands/commit.md:15-17`）。

3. stage 范围缺乏显式策略  
协议只要求“stage and create commit”，未声明应全量 `git add .` 还是按改动类型筛选，容易在复杂工作区误纳入不期望文件（`plugins/commit-commands/commands/commit.md:17`）。

4. 失败分支未结构化  
未定义“无改动可提交”“commit message 冲突”“hook 失败”等失败路径，运行行为依赖模型临场处理（`plugins/commit-commands/README.md:187-194`）。

改进建议：

1. 补齐白名单  
在 `allowed-tools` 中加入 `Bash(git diff:*)`、`Bash(git branch:*)`、`Bash(git log:*)`，保证声明与实际一致。

2. 显式化提交边界  
将“stage 策略”写入协议（例如仅 stage tracked changes，或先列文件再执行 commit）。

3. 增加轻量安全检查  
在执行 commit 前加入可审计检查（如敏感文件模式提示、空提交提前退出）。

4. 增加参数化能力  
可新增 `argument-hint` 支持自定义 commit message 或 scope，减少纯自动生成在关键提交上的误差。
