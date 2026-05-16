# plugins/commit-commands/README.md 研究

## 场景与职责

`plugins/commit-commands/README.md` 是 `commit-commands` 插件的用户文档入口，职责是把该插件的 3 个命令能力、适用场景、依赖前提和排障路径解释给使用者，而不是直接执行 Git 操作（`plugins/commit-commands/README.md:1-225`）。

在仓库/插件体系中的位置：
- 根文档仅负责把用户导向插件总览，不展开单插件细节（`README.md:48-50`）。
- `plugins/README.md` 将该插件暴露为“Git workflow automation”，并列出 3 个命令（`plugins/README.md:18`）。
- marketplace 把插件目录注册为可发现组件（`.claude-plugin/marketplace.json:40-49`）。
- 插件元信息来自 `.claude-plugin/plugin.json`，README 承接“人类可读行为说明”（`plugins/commit-commands/.claude-plugin/plugin.json:1-9`）。

与同目录文件的职责边界：
- `README.md`：说明“应当做什么、何时用、依赖什么、失败怎么排查”（文档层）。
- `commands/*.md`：定义“实际怎么执行”的命令协议（执行层）。
- `.claude-plugin/plugin.json`：插件标识、版本、作者（配置层）。

## 功能点目的

### 1) `/commit`
目的：把“分析改动 + 生成提交信息 + 提交”收敛成单命令，减少频繁手动 `git add/commit` 的切换成本。
- README 描述了从状态分析到提交落地的 6 步过程（`plugins/commit-commands/README.md:15-22`）。
- 目标体验是“在开发节奏中快速提交，仍保留人工复核”（`plugins/commit-commands/README.md:134-139`）。

### 2) `/commit-push-pr`
目的：把“建分支（main 上）-> 提交 -> 推送 -> 创建 PR”串成一次 workflow，降低漏步骤概率。
- README 给出完整链路与预期输出（PR URL）（`plugins/commit-commands/README.md:51-57`）。
- 面向“准备提 PR”这一阶段性节点，而不是日常小步提交（`plugins/commit-commands/README.md:140-145`）。

### 3) `/clean_gone`
目的：维护本地仓库卫生，自动移除远端已删分支及其关联 worktree。
- README 把“识别 gone -> 处理 worktree -> 删除分支”写成明确流程（`plugins/commit-commands/README.md:94-99`）。
- 使用时机聚焦在“PR 合并后周期性清理”（`plugins/commit-commands/README.md:123-126,147-151`）。

### 4) 支撑性内容（安装/前提/排障）
README 同时承担操作约束说明：
- 运行前提：Git、`gh`、远端仓库（`plugins/commit-commands/README.md:86-89,179-183`）。
- 故障排查：无变更可提交、`gh` 认证失败、`[gone]` 不出现时先 `git fetch --prune`（`plugins/commit-commands/README.md:187-210`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

README 本身是“实现意图说明”；真正执行协议在 `commands/*.md`。二者合并后可还原具体实现。

### A. 命令协议载体
3 个命令都使用 Markdown + YAML frontmatter：
- `description` 用于命令语义说明（`plugins/commit-commands/commands/commit.md:3`，`plugins/commit-commands/commands/commit-push-pr.md:3`，`plugins/commit-commands/commands/clean_gone.md:2`）。
- `allowed-tools` 用于限制工具白名单（`commit.md:2`，`commit-push-pr.md:2`）。frontmatter 语义与格式约束见参考规范（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-127`）。

### B. 关键执行流程
1. `/commit`（`plugins/commit-commands/commands/commit.md:6-17`）
- 先注入运行时上下文：`git status`、`git diff HEAD`、当前分支、最近提交。
- 再要求同一条消息内完成 `git add` + `git commit`。

2. `/commit-push-pr`（`plugins/commit-commands/commands/commit-push-pr.md:6-20`）
- 注入当前仓库状态。
- 规定顺序：必要时建分支 -> 提交 -> 推送 -> `gh pr create`。
- 强约束“单消息内完成全部工具调用”。

3. `/clean_gone`（`plugins/commit-commands/commands/clean_gone.md:11-40`）
- 通过 `git branch -v` 查 `[gone]`。
- 读取 `git worktree list`。
- 用 `grep/sed/awk/while` 管道批量移除 worktree 并 `git branch -D` 删除本地分支。

### C. 数据结构（隐式）
该插件没有独立代码结构体，数据结构通过提示词协议隐式存在：
- `GitSnapshot`：`status/diff/current_branch/recent_commits`（来自 `!\`...\`` 注入，`commit.md:8-11`）。
- `PRWorkflowPlan`：`need_new_branch/commit_message/push/pr_create`（来自 `commit-push-pr.md:16-19`）。
- `GoneBranchItem`：`branch_name/worktree_path/is_removed`（来自 `clean_gone.md:29-39`）。

### D. 关键协议细节
- 插件发现协议：命令位于 `commands/` 会被自动发现（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-345`，`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:214-237,356-371`）。
- 命名协议：`commit-push-pr.md -> /commit-push-pr`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:307-310`）。
- Bash 上下文注入协议：命令可用 `!\`cmd\`` 将输出注入提示上下文（`plugins/plugin-dev/skills/command-development/README.md:117-128,192-202`）。

## 关键代码路径与文件引用

目标对象：
- `plugins/commit-commands/README.md:1-225`

上游调用方（谁把用户导向它）：
1. 根入口：`README.md:48-50`
2. 插件总览：`plugins/README.md:13-19`
3. marketplace 注册：`.claude-plugin/marketplace.json:40-49`

下游被调用方（README 所描述能力由谁实现）：
1. 插件元信息：`plugins/commit-commands/.claude-plugin/plugin.json:1-9`
2. `/commit`：`plugins/commit-commands/commands/commit.md:1-17`
3. `/commit-push-pr`：`plugins/commit-commands/commands/commit-push-pr.md:1-20`
4. `/clean_gone`：`plugins/commit-commands/commands/clean_gone.md:1-53`

配置、测试、脚本、文档上下文覆盖：
- 配置：`plugin.json` + command frontmatter（`allowed-tools`/`description`）。
- 测试：`plugins/commit-commands` 目录未发现 `test/spec` 文件（仅 5 个文件：README、manifest、3 个 command）。
- 脚本：目录内无独立 `scripts/`；`clean_gone` 的脚本逻辑直接内联在命令 markdown（`clean_gone.md:25-40`）。
- 文档：形成 `README.md -> plugins/README.md -> plugins/commit-commands/README.md` 的三级文档链路。

补充关键上下文：
- 仓库内还存在 `.claude/commands/commit-push-pr.md` 的同名命令定义，与插件版几乎一致（`.claude/commands/commit-push-pr.md:1-19`），属于并行来源。

## 依赖与外部交互

### 内部依赖
1. 插件加载链路：`.claude-plugin/marketplace.json` 的 `source` 指向插件目录（`.claude-plugin/marketplace.json:47`）。
2. 自动发现链路：依赖 `commands/` 默认扫描与命令文件命名规则（`plugin-structure/SKILL.md:343-345,307-310`）。
3. 命令权限链路：依赖 frontmatter `allowed-tools` 约束是否与正文调用一致（`commit.md:2,8-11`，`commit-push-pr.md:2,8-10`）。

### 外部交互
1. Git 本地写操作：`git add/commit/checkout/push/branch -D/worktree remove`（见三份命令文件）。
2. GitHub 远程交互：`gh pr create`（`commit-push-pr.md:2,19`）。
3. Shell 文本处理工具：`grep/sed/awk`（`clean_gone.md:29-33`）。
4. 环境前提：`gh` 需安装并认证，仓库需有 `origin`（`plugins/commit-commands/README.md:86-89,195-202`）。

## 风险、边界与改进建议

### 风险
1. README 承诺与命令约束存在漂移
- README 声称 `/commit` 会遵循 conventional commits、规避 secrets、附带 attribution（`plugins/commit-commands/README.md:41-45`），但 `commit.md` 未给出显式检查步骤（`commit.md:15-17`）。

2. `allowed-tools` 与上下文注入命令不完全匹配
- `/commit` frontmatter 允许 `git add/status/commit`，但上下文里还执行 `git diff`、`git branch`、`git log`（`commit.md:2,8-11`）。
- `/commit-push-pr` 允许 `checkout/add/status/push/commit/gh pr create`，但上下文还有 `git diff`、`git branch`（`commit-push-pr.md:2,8-10`）。

3. `/clean_gone` 删除策略偏激进
- 使用 `git branch -D` 强制删除，若本地分支仍有未合并价值提交，存在误删风险（`clean_gone.md:39`）。

4. `/clean_gone` 文本解析脆弱
- 依赖 `git branch -v | grep | sed | awk` 与正则匹配，特殊分支名或输出格式变化时可能误判（`clean_gone.md:29-33`）。

5. 双源命令定义导致维护分叉风险
- `.claude/commands/commit-push-pr.md` 与插件命令并存，后续更新容易不同步。

6. 无自动化回归
- 插件目录没有专门测试/校验脚本，行为变化主要靠人工验证。

### 边界
1. 本插件是“提示词协议层”，并非可编译执行代码库。
2. 成功率高度依赖运行环境状态（Git 仓库状态、远端权限、`gh auth`）。
3. README 只描述意图，不直接保证执行器一定按文案细节实现。

### 改进建议
1. 对齐 `allowed-tools` 与实际上下文命令
- 为 `/commit` 补充 `Bash(git diff:*)`、`Bash(git branch:*)`、`Bash(git log:*)`；
- 为 `/commit-push-pr` 补充 `Bash(git diff:*)`、`Bash(git branch:*)`。

2. 将 README 的“保证式表述”改成“约束+前提”
- 要么在命令里补显式检查（secrets/attribution/conventional 格式），要么在 README 弱化为“建议行为”。

3. 降低 `/clean_gone` 误删风险
- 改为“两阶段删除”：先 `git branch -d`，失败再提示并确认是否 `-D`。

4. 用结构化 Git 输出替代文本管道
- 优先 `git for-each-ref` 等稳定字段接口，减少 `grep/sed/awk` 解析脆弱性。

5. 统一 `commit-push-pr` 单一来源
- 明确由插件命令或 `.claude/commands` 二选一作为权威定义。

6. 增加最小回归校验
- 至少校验 frontmatter 可解析、`allowed-tools` 覆盖一致、三条命令关键步骤完整。
