# plugins/commit-commands 目录研究（DIR）

## 场景与职责

`plugins/commit-commands` 是一个“Git 工作流自动化”插件，目标是把高频但分散的 Git/GitHub CLI 操作封装为 3 个 slash command：`/commit`、`/commit-push-pr`、`/clean_gone`。

- 插件在仓库级 marketplace 中注册为可发现项：`.claude-plugin/marketplace.json:40-49`
- 插件在总览中被声明为“commit/push/PR 创建”能力：`plugins/README.md:18`
- 插件元信息由 `.claude-plugin/plugin.json` 提供：`plugins/commit-commands/.claude-plugin/plugin.json:1-9`

目录结构是典型“声明式插件”：
- 元数据：`plugins/commit-commands/.claude-plugin/plugin.json`
- 命令协议：`plugins/commit-commands/commands/*.md`
- 用户文档：`plugins/commit-commands/README.md`

该目录不包含可执行源码（如 `.ts/.js/.py`）或目录内脚本/测试；核心行为由 Markdown 命令提示词驱动。

## 功能点目的

### 1) `/commit`

目的：用单命令完成“变更理解 + 提交”，减少手工写 commit message 与多步 Git 操作。

- 限定工具为 `git add/status/commit`：`plugins/commit-commands/commands/commit.md:2`
- 预注入上下文包括 `git status`、`git diff HEAD`、当前分支、最近 10 条提交：`plugins/commit-commands/commands/commit.md:8-11`
- 强约束“单次消息内完成工具调用，不输出额外文本”：`plugins/commit-commands/commands/commit.md:15-17`

### 2) `/commit-push-pr`

目的：把“建分支（如在 main）-> 提交 -> 推送 -> 创建 PR”串成一次操作，降低上下文切换。

- 限定工具包含 `git checkout --branch`、`git add/status/commit/push`、`gh pr create`：`plugins/commit-commands/commands/commit-push-pr.md:2`
- 任务顺序明确要求 4 步完成：`plugins/commit-commands/commands/commit-push-pr.md:16-20`
- 同样要求在单条消息内完成所有工具调用：`plugins/commit-commands/commands/commit-push-pr.md:20`

### 3) `/clean_gone`

目的：清理远端已删除但本地仍残留的分支，并处理相关 worktree。

- 先列出分支识别 `[gone]`：`plugins/commit-commands/commands/clean_gone.md:11-17`
- 再查 `git worktree list`：`plugins/commit-commands/commands/clean_gone.md:19-23`
- 最后通过管道脚本循环删除 worktree 与分支：`plugins/commit-commands/commands/clean_gone.md:25-40`

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用链）

1. 插件发现与注册  
   marketplace 条目 `source: "./plugins/commit-commands"` 指向本目录，作为上游发现入口：`.claude-plugin/marketplace.json:47`

2. 命令发现  
   插件结构遵循 `commands/` 自动发现约定（插件目录规范）：`plugins/README.md:49-61`

3. 命令执行  
   用户触发 `/commit`、`/commit-push-pr`、`/clean_gone` 后，Claude 按命令 Markdown（含 frontmatter 和正文约束）执行对应 shell 工具链。

### B. 数据结构与协议

1. 插件元数据（JSON）  
   `name/description/version/author` 四类字段定义插件标识与展示信息：`plugins/commit-commands/.claude-plugin/plugin.json:2-8`

2. 命令协议（Markdown + YAML frontmatter）  
   - `description`：命令说明（用于帮助展示语义）  
   - `allowed-tools`：命令级工具白名单（仅在 `/commit` 与 `/commit-push-pr` 声明）  
   参考命令 frontmatter 规范：`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-99`

3. 动态上下文注入协议  
   `!` 反引号命令（如 `!`git status``）在执行前注入当前仓库上下文，用于让模型基于即时状态决策：`plugins/commit-commands/commands/commit.md:8-11`、`plugins/commit-commands/commands/commit-push-pr.md:8-10`

### C. 关键命令与行为

1. `/commit`  
   - 读取工作区与提交历史上下文  
   - 允许 `git add` + `git commit` 落地单次提交  
   - 强制“只发工具调用消息”减少多轮交互噪音

2. `/commit-push-pr`  
   - 在 `main` 上时创建新分支（由指令规定）  
   - 提交后推送到 `origin`  
   - 调用 `gh pr create` 打开 PR

3. `/clean_gone`  
   - 通过 `git branch -v | grep '\[gone\]'` 过滤候选分支：`plugins/commit-commands/commands/clean_gone.md:29`
   - `sed` 去掉 `+/*/空格` 前缀并由 `awk` 取分支名：`plugins/commit-commands/commands/clean_gone.md:29`
   - 用 `git worktree list` 匹配关联 worktree 并 `git worktree remove --force`：`plugins/commit-commands/commands/clean_gone.md:32-35`
   - 最后 `git branch -D` 强制删分支：`plugins/commit-commands/commands/clean_gone.md:39`

## 关键代码路径与文件引用

### 目录内核心路径

- 插件元信息：`plugins/commit-commands/.claude-plugin/plugin.json:1-9`
- 命令 `/commit`：`plugins/commit-commands/commands/commit.md:1-17`
- 命令 `/commit-push-pr`：`plugins/commit-commands/commands/commit-push-pr.md:1-20`
- 命令 `/clean_gone`：`plugins/commit-commands/commands/clean_gone.md:1-53`
- 使用文档：`plugins/commit-commands/README.md:1-225`

### 上游调用方（谁发现/触发它）

- 仓库级 marketplace 注册：`.claude-plugin/marketplace.json:40-49`
- 插件目录总览声明：`plugins/README.md:18`
- 根 README 的插件入口说明：`README.md:48-50`

### 下游被调用方（它会调用谁）

- Git CLI：`git add/status/commit/checkout/push/branch/worktree`（见命令正文与 `allowed-tools`）
- GitHub CLI：`gh pr create`（`/commit-push-pr`）
- Shell 工具链：`grep/sed/awk/while`（`/clean_gone` 管道脚本）

### 配置、测试、脚本、文档覆盖

- 配置：frontmatter（`description`、`allowed-tools`）+ `.claude-plugin/plugin.json`
- 测试：目录内无 `test`/`spec` 文件（声明式插件，未见自动化回归入口）
- 脚本：目录内无独立 `scripts/*.sh`，清理逻辑内联在 `clean_gone.md`
- 文档：`plugins/commit-commands/README.md` 对用法、需求与故障排查有完整描述（如 `gh auth login`、`git fetch --prune`）：`plugins/commit-commands/README.md:195-210`

## 依赖与外部交互

### 运行时依赖

1. Claude Code 插件系统（负责插件/命令发现与执行）
2. 本地 Git 环境（所有命令都依赖）
3. GitHub CLI 与认证（`/commit-push-pr` 依赖）：`plugins/commit-commands/README.md:86-88,182-183,195-202`
4. 仓库需存在 `origin` 远端（README 明确要求）：`plugins/commit-commands/README.md:88`

### 与仓库状态的交互

1. 读取：`git status`、`git diff HEAD`、`git log --oneline -10`  
2. 写入：`git add`、`git commit`、`git push`、`git branch -D`  
3. 远程交互：`gh pr create` 与 GitHub API（由 gh CLI 间接完成）

### 与其他目录/命令关系

仓库根还存在一个同名行为的本地命令文件 `.claude/commands/commit-push-pr.md`，与插件版本几乎完全一致（仅空行差异），形成“双源定义”：
- `.claude/commands/commit-push-pr.md:1-19`
- `plugins/commit-commands/commands/commit-push-pr.md:1-20`

## 风险、边界与改进建议

### 风险与边界

1. `README` 描述与命令实现存在能力漂移风险  
   README 声称 `/commit` 会“避免提交 secrets、附加 attribution、遵循 conventional commit”，但 `commit.md` 本身未写入硬性检查或过滤命令，只给出高层任务描述：  
   - 文档描述：`plugins/commit-commands/README.md:41-45`  
   - 命令实现：`plugins/commit-commands/commands/commit.md:15-17`

2. `/clean_gone` 未声明 `allowed-tools`  
   与另两个命令不同，`clean_gone.md` frontmatter 只有 `description`，工具权限继承会话默认，最小权限边界不清晰：`plugins/commit-commands/commands/clean_gone.md:1-3`

3. `clean_gone` 对分支名解析与匹配较脆弱  
   依赖 `grep/sed/awk` 文本解析 `git branch -v` 输出格式；若分支名包含正则特殊字符，`grep "\\[$branch\\]"` 匹配可能误判：`plugins/commit-commands/commands/clean_gone.md:29-33`

4. 强制删除策略风险  
   使用 `git branch -D` 强制删除本地分支，若 `[gone]` 分支包含仅本地有效提交，存在误删风险：`plugins/commit-commands/commands/clean_gone.md:39`

5. 双源命令定义的维护成本  
   `.claude/commands/commit-push-pr.md` 与插件内同名命令并存，未来任一方更新都可能导致行为分叉。

6. 缺少自动化测试/回归校验  
   目录内无脚本化测试，prompt 变更后的行为一致性主要靠人工验证。

### 改进建议

1. 为 `/clean_gone` 增加最小 `allowed-tools` 白名单  
   例如限制到 `Bash(git branch:*), Bash(git worktree:*), Bash(git rev-parse:*), Bash(grep:*), Bash(sed:*), Bash(awk:*)` 或改写为纯 Git 子命令流程以减少外壳工具依赖。

2. 把 README 承诺改为“可验证约束”  
   若要求 secrets 保护，可在命令中显式加入 `.env`/凭证文件排除检查逻辑；否则应弱化 README 的“保证式”表述。

3. 降低文本解析脆弱性  
   优先使用更结构化的 Git 命令输出（如 `for-each-ref`）替代 `branch -v | grep | sed | awk` 管道。

4. 将 `.claude/commands/commit-push-pr.md` 与插件版本统一为单一来源  
   可改为保留一份并通过文档/生成流程同步，避免长期漂移。

5. 增加轻量回归脚本  
   至少校验：frontmatter 可解析、`allowed-tools` 覆盖齐全、命令关键步骤未丢失（commit/push/pr、gone 清理）。
