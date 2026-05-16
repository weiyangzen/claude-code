# plugins/commit-commands/commands/clean_gone.md 研究

## 场景与职责

`plugins/commit-commands/commands/clean_gone.md` 是 `commit-commands` 插件中的“仓库维护型命令协议”，职责是把“清理远端已删除但本地残留的分支”收敛为可复用的标准流程（`plugins/commit-commands/commands/clean_gone.md:1-53`）。

它在插件体系中的职责边界是：

- 上游发现与装配
- marketplace 将插件注册到 `./plugins/commit-commands`（`.claude-plugin/marketplace.json:40-48`）。
- 插件启用后，命令目录按约定自动扫描，文件名 `clean_gone.md` 映射为 `/clean_gone`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:307-310,341-345`）。
- 下游执行对象
- `git branch -v`、`git worktree list`、`git worktree remove --force`、`git branch -D` 与 shell 管道工具（`plugins/commit-commands/commands/clean_gone.md:14,22,29-39`）。

该文件不负责插件元信息与用户教育文档，这两部分分别由 `plugin.json` 与 `README.md` 承担（`plugins/commit-commands/.claude-plugin/plugin.json:1-9`，`plugins/commit-commands/README.md:90-127`）。

## 功能点目的

1. 识别“远端已删除、本地仍存在”的陈旧分支  
通过 `[gone]` 标记筛选候选对象，避免手工逐个对比（`plugins/commit-commands/commands/clean_gone.md:11-17,29`）。

2. 先处理 worktree 再删分支  
命令显式强调 `+` 前缀分支有 worktree 关联，需要先移除 worktree 再删除分支，避免 git 报错（`plugins/commit-commands/commands/clean_gone.md:17,32-36`）。

3. 批处理清理  
使用循环一次处理所有 `[gone]` 分支，减少重复手工命令（`plugins/commit-commands/commands/clean_gone.md:29-40`）。

4. 给出可追踪反馈  
逐分支输出 `Processing/Removing/Deleting`，让操作者可见清理动作和结果（`plugins/commit-commands/commands/clean_gone.md:30,34,38`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 协议与配置层

- frontmatter 仅定义 `description`，未定义 `allowed-tools`（`plugins/commit-commands/commands/clean_gone.md:1-3`）。
- 与另外两个命令不同，它没有最小工具白名单约束，工具权限依赖会话默认策略（对比 `commit.md` 与 `commit-push-pr.md` 的 `allowed-tools`：`plugins/commit-commands/commands/commit.md:2`，`plugins/commit-commands/commands/commit-push-pr.md:2`）。

### 2) 执行流程（3 阶段）

1. 预览分支状态  
执行 `git branch -v`，让 `[gone]` 候选分支显式可见（`plugins/commit-commands/commands/clean_gone.md:11-15`）。

2. 预览 worktree 映射  
执行 `git worktree list`，为后续删除 worktree 做映射准备（`plugins/commit-commands/commands/clean_gone.md:19-23`）。

3. 批量清理  
管道从 `git branch -v` 解析 `[gone]` 分支名，循环中先删关联 worktree，再强制删分支（`plugins/commit-commands/commands/clean_gone.md:29-40`）。

### 3) 关键命令链

- 分支提取：  
`git branch -v | grep '\[gone\]' | sed 's/^[+* ]//' | awk '{print $1}'`（`plugins/commit-commands/commands/clean_gone.md:29`）。
- worktree 匹配：  
`git worktree list | grep "\\[$branch\\]" | awk '{print $1}'`（`plugins/commit-commands/commands/clean_gone.md:32`）。
- 删除动作：  
`git worktree remove --force "$worktree"` + `git branch -D "$branch"`（`plugins/commit-commands/commands/clean_gone.md:35,39`）。

### 4) 隐式数据结构（协议驱动）

- `GoneBranchCandidate`
- 字段语义：`branch_name`、是否 `[gone]`、是否带 `+` 前缀。
- 来源：`git branch -v` 文本输出（`plugins/commit-commands/commands/clean_gone.md:14,17,29`）。

- `WorktreeBinding`
- 字段语义：`branch_name -> worktree_path`。
- 来源：`git worktree list` + `grep "\\[$branch\\]"`（`plugins/commit-commands/commands/clean_gone.md:22,32`）。

- `CleanupActionResult`
- 字段语义：`worktree_removed`、`branch_deleted`、执行反馈文本。
- 来源：循环内 `echo` 和 git 删除命令结果（`plugins/commit-commands/commands/clean_gone.md:30-39`）。

## 关键代码路径与文件引用

### 命令实现主路径

- `plugins/commit-commands/commands/clean_gone.md:1-53`  
定义 `/clean_gone` 的完整执行协议和命令链。

### 调用方（上游）路径

- `.claude-plugin/marketplace.json:40-48`  
把 `commit-commands` 插件注册为可发现来源。
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:307-310,341-345`  
定义命令文件命名映射与自动发现机制。

### 被调用方（下游）路径

- `plugins/commit-commands/commands/clean_gone.md:14,22,29-39`  
实际调用的 Git 与 shell 文本处理链。

### 配置/文档/测试/脚本路径

- 配置
- `plugins/commit-commands/.claude-plugin/plugin.json:1-9`（插件元信息）。
- 文档
- `plugins/commit-commands/README.md:90-127,147-151,204-210`（命令用途、时机、排障）。
- 测试
- `plugins/commit-commands` 目录内未提供该命令专属自动化测试文件（未见 `test`/`spec` 目录）。
- 脚本
- 无独立脚本文件；清理脚本内联在命令 Markdown（`plugins/commit-commands/commands/clean_gone.md:25-40`）。

## 依赖与外部交互

### 内部依赖

- 依赖插件装配链：marketplace 注册 + 命令目录自动扫描（`.claude-plugin/marketplace.json:40-48`，`plugins/plugin-dev/skills/plugin-structure/SKILL.md:341-345`）。
- 依赖用户文档给出的运维前提：本地分支状态应先与远端同步（`plugins/commit-commands/README.md:204-210`）。

### 外部交互

- Git 本地仓库写操作
- `git worktree remove --force` 直接移除 worktree 目录（`plugins/commit-commands/commands/clean_gone.md:35`）。
- `git branch -D` 强制删除本地分支（`plugins/commit-commands/commands/clean_gone.md:39`）。
- Shell 工具链
- `grep`、`sed`、`awk`、`while read` 参与文本解析与循环控制（`plugins/commit-commands/commands/clean_gone.md:29-40`）。

## 风险、边界与改进建议

1. 权限边界弱化  
`clean_gone.md` 未声明 `allowed-tools`，与同插件其他命令不一致，最小权限策略缺失（`plugins/commit-commands/commands/clean_gone.md:1-3`）。

2. 文本解析脆弱  
逻辑依赖 `git branch -v` 文本格式与正则匹配，分支名包含特殊字符或输出格式变化时可能误判（`plugins/commit-commands/commands/clean_gone.md:29-33`）。

3. 删除策略偏激进  
`git worktree remove --force` 与 `git branch -D` 都是强制操作，存在误删仍有价值本地提交/工作目录的风险（`plugins/commit-commands/commands/clean_gone.md:35,39`）。

4. 对远端状态新鲜度敏感  
若未先 `git fetch --prune`，`[gone]` 视图可能不准确，导致“该删未删”或“状态判断滞后”（`plugins/commit-commands/README.md:204-210`）。

改进建议：

1. 增加最小白名单  
在 frontmatter 补充 `allowed-tools`，只允许本命令实际需要的 `git branch/worktree/rev-parse` 与必要 shell 子命令。

2. 提升分支枚举稳健性  
改用结构化接口（如 `git for-each-ref`）代替 `grep/sed/awk` 组合，减少文本格式耦合。

3. 增加安全阀  
先执行 dry-run（仅打印将删除对象），再二次确认执行强删；或先尝试 `git branch -d`，失败时再升级 `-D`。

4. 增加错误分支处理  
对 `git worktree remove` 失败、权限不足、路径不存在等情况给出分支级失败统计，避免半成功半失败时信息不透明。
