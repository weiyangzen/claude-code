# DIR `.claude/commands` 研究报告

## 场景与职责

`.claude/commands` 是仓库级 slash-command 提示词入口，当前包含 3 个命令文件：

- `dedupe.md`：为新 Issue 执行重复问题识别并回帖。
- `triage-issue.md`：对 Issue 做自动分诊与生命周期标签治理。
- `commit-push-pr.md`：本地开发流中的提交/推送/建 PR 一体化命令。

职责定位不是“业务代码执行层”，而是“Agent 行为编排层”：

1. 通过 frontmatter 的 `allowed-tools` 限制可调用工具。
2. 通过命令正文定义流程顺序、判定条件和输出动作。
3. 把 GitHub Actions 触发事件转译为 Claude 可执行的步骤。

在运行场景上：

- `/dedupe` 与 `/triage-issue` 是 GitHub 工作流自动触发的运维流水线能力。
- `/commit-push-pr` 是人机交互下的开发效率能力（与插件同名命令同构）。

## 功能点目的

### 1) `/dedupe`

目的：降低重复 Issue 对维护成本的影响，在新 Issue 打开后快速识别候选重复项并通知提报者。

预期价值：

- 前置筛掉明显重复。
- 为后续自动关闭重复 Issue 的定时任务提供结构化信号（固定评论模板）。
- 通过“最多 3 个候选”控制误伤与噪音。

### 2) `/triage-issue`

目的：把“Issue 分类”和“生命周期状态”标准化到标签体系，减少人工手动分诊。

预期价值：

- 新 Issue 触发时：统一类型/领域标签并决定是否打 `invalid`、`needs-repro`、`needs-info`。
- 新评论触发时：基于增量对话移除或补充生命周期标签。
- 与 `sweep.ts` 的超时关闭策略形成闭环。

### 3) `/commit-push-pr`

目的：将常见 Git 工作流（commit + push + PR）压缩为一次命令调用，减少上下文切换。

预期价值：

- 对“修改已完成、准备提 PR”的阶段提高吞吐。
- 用最小工具面（`git` + `gh pr create`）约束执行范围。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 命令协议与权限边界

三个命令都采用 Markdown + YAML frontmatter 协议：

- `description`：命令展示描述。
- `allowed-tools`：运行时允许调用的工具白名单（例如 `Bash(./scripts/gh.sh:*)`）。

这意味着 `.claude/commands` 本质上是“声明式执行策略”：

- 命令正文负责流程编排。
- frontmatter 负责权限收敛。

### B. `/dedupe` 关键流程

触发入口：

- `.github/workflows/claude-dedupe-issues.yml`
- 事件：`issues.opened` / `workflow_dispatch`
- 关键 prompt：`/dedupe <owner/repo>/issues/<number>`

执行流程（命令定义层）：

1. 先判断是否应跳过（已关闭、非去重场景、已有重复评论）。
2. 读取目标 issue 并总结。
3. 并行搜索候选重复项（命令文本要求 5 并行搜索策略）。
4. 二次过滤误报。
5. 调用：
   - `./scripts/comment-on-duplicates.sh --base-issue <N> --potential-duplicates <A> <B> <C>`

实现支撑：

- `scripts/gh.sh`：对 `gh` 做严格白名单封装，仅允许：
  - `issue view`
  - `issue list`
  - `search issues`
  - `label list`
- 参数校验要点：
  - 仅接受 `--comments --state --limit --label`
  - `issue view` 必须是数字 issue 编号
  - `search issues` 禁止 `repo:/org:/user:` qualifier，避免越权搜索

下游联动：

- `scripts/comment-on-duplicates.sh` 发布固定格式评论（包含候选链接与 3 天自动关闭提示）。
- `.github/workflows/auto-close-duplicates.yml` + `scripts/auto-close-duplicates.ts` 会读取该评论形态并在满足条件时自动关闭。
- `.github/workflows/backfill-duplicate-comments.yml` + `scripts/backfill-duplicate-comments.ts` 可批量回填旧 issue 的 dedupe 流程触发。

### C. `/triage-issue` 关键流程

触发入口：

- `.github/workflows/claude-issue-triage.yml`
- 事件：`issues.opened` + `issue_comment.created`
- 过滤：评论事件仅处理非 PR 且非 Bot 评论
- 并发控制：`concurrency.group = issue-triage-<issue_number>`
- 关键 prompt：`/triage-issue REPO: ... ISSUE_NUMBER: ... EVENT: ...`

执行流程（按事件分支）：

1. `./scripts/gh.sh label list` 拉取真实标签全集。
2. `./scripts/gh.sh issue view <N>` + `--comments` 获取上下文。
3. 若 `EVENT=issues`：
   - 先判断是否属于 Claude Code 范畴，否则打 `invalid` 并停止。
   - 评估类别标签与重复项。
   - 仅对 bug 评估 `needs-repro` / `needs-info`。
4. 若 `EVENT=issue_comment`：
   - 只处理生命周期标签（如移除 `stale`/`autoclose`、按新信息移除 `needs-*`、必要时新增 `needs-*`）。
   - 不允许改类别标签。
5. 标签落地：
   - `./scripts/edit-issue-labels.sh --issue <N> --add-label ... --remove-label ...`

实现支撑：

- `scripts/edit-issue-labels.sh` 会二次读取仓库标签列表并过滤非法标签，避免“幻觉标签”落地。
- 生命周期策略单一事实源在 `scripts/issue-lifecycle.ts`，被 `sweep.ts` 和 `lifecycle-comment.ts` 共用。

### D. `/commit-push-pr` 关键流程

命令内容为强约束串行操作：

1. 若在 `main` 上则新建分支。
2. 提交当前修改为单个 commit。
3. push 到 `origin`。
4. `gh pr create` 建立 PR。
5. 要求“在单条响应内完成全部工具调用”。

上下文注入：

- 使用 `!\`git status\``、`!\`git diff HEAD\``、`!\`git branch --show-current\`` 在执行前提供仓库状态。

同构关系：

- `plugins/commit-commands/commands/commit-push-pr.md` 与本文件内容等价，说明该能力既存在项目级命令，也以插件形式复用。

### E. 数据结构与文本协议

关键“数据结构”并非数据库模型，而是脚本侧结构化约定：

- `scripts/issue-lifecycle.ts`：`lifecycle[]` 数组（`label/days/reason/nudge`）定义标签超时规则。
- `scripts/auto-close-duplicates.ts`：
  - `GitHubIssue/GitHubComment/GitHubReaction` 接口
  - 通过评论正文正则抽取 duplicate 目标 issue 编号
- `scripts/comment-on-duplicates.sh`：固定评论模板（可被后续自动关闭脚本识别）

这形成了“命令提示词 -> 评论/标签副作用 -> 定时脚本消费”的文本协议链。

## 关键代码路径与文件引用

### 目标目录（被研究对象）

- `.claude/commands/dedupe.md`
- `.claude/commands/triage-issue.md`
- `.claude/commands/commit-push-pr.md`

### 调用方（谁触发这些命令）

- `.github/workflows/claude-dedupe-issues.yml`
- `.github/workflows/claude-issue-triage.yml`

### 被调用方（命令实际调用的脚本）

- `scripts/gh.sh`
- `scripts/comment-on-duplicates.sh`
- `scripts/edit-issue-labels.sh`

### 强相关后处理链路

- `.github/workflows/auto-close-duplicates.yml` -> `scripts/auto-close-duplicates.ts`
- `.github/workflows/backfill-duplicate-comments.yml` -> `scripts/backfill-duplicate-comments.ts`
- `.github/workflows/issue-lifecycle-comment.yml` -> `scripts/lifecycle-comment.ts`
- `.github/workflows/sweep.yml` -> `scripts/sweep.ts`
- `scripts/issue-lifecycle.ts`

### 文档与规范上下文

- `plugins/plugin-dev/skills/command-development/SKILL.md`（命令 frontmatter 与组织方式规范）
- `plugins/commit-commands/commands/commit-push-pr.md`（同名同构命令）
- `plugins/commit-commands/README.md`（用户侧行为说明）
- `CHANGELOG.md`（自定义 slash-command 能力演进记录）

### 测试上下文

- 仓库中未发现针对 `.claude/commands` 或上述脚本的专门测试目录/用例（未检索到 `tests/`、`__tests__/`、`*.spec.*` 针对链路）。

## 依赖与外部交互

### 运行依赖

- GitHub Actions 运行环境
- `anthropics/claude-code-action@v1`
- `gh` CLI（由 shell 脚本调用）
- Bun（执行 TypeScript 自动化脚本）

### 外部 API / 服务

- GitHub API（`gh` 间接调用 + TS 脚本 `fetch https://api.github.com/...` 直接调用）
- Statsig Events API（`claude-dedupe-issues.yml`、`log-issue-events.yml`）

### 关键环境变量

- `GITHUB_TOKEN` / `GH_TOKEN`
- `GH_REPO` / `GITHUB_REPOSITORY`
- `GITHUB_REPOSITORY_OWNER` / `GITHUB_REPOSITORY_NAME`
- `ANTHROPIC_API_KEY`
- `STATSIG_API_KEY`

## 风险、边界与改进建议

### 风险与边界

1. 仓库硬编码与可移植性风险
- `scripts/comment-on-duplicates.sh` 固定 `REPO="anthropics/claude-code"`。
- `scripts/backfill-duplicate-comments.ts` 固定 `owner/repo`。
- `scripts/sweep.ts` 的 `NEW_ISSUE` 链接也固定到主仓库。
- 结果：在 fork/多仓复用时容易写错目标仓库。

2. 参数契约不一致
- `backfill-duplicate-comments.yml` 暴露了 `days_back` 输入，但脚本实际未消费 `DAYS_BACK`。
- 结果：操作员可能误以为支持按时间窗口回填。

3. dedupe 并行代理能力与工具声明耦合偏弱
- `dedupe.md` 文本强调“多 agent 并行”，但 frontmatter 只声明 Bash 脚本工具；执行器行为依赖宿主能力。
- 结果：跨环境运行时可能出现执行策略偏差。

4. 触发面较宽
- triage/dedupe workflow 使用 `allowed_non_write_users: "*"`。
- 仓库虽有 `non-write-users-check.yml` 做审计提醒，但触发范围仍广。

5. 自动关闭依赖文本模式匹配
- 自动关闭脚本基于评论内容关键字与正则抽取 duplicate issue。
- 若评论模板变化或包含多链接/多编号，可能引入误判。

6. 测试覆盖不足
- 缺少对关键判定逻辑（标签增删、duplicate 提取、自动关闭条件）的自动化回归测试。

7. 命令定义重复维护风险
- `.claude/commands/commit-push-pr.md` 与 `plugins/commit-commands/commands/commit-push-pr.md` 同构，可能发生漂移。

### 改进建议

1. 去硬编码配置化
- 统一从 `GH_REPO/GITHUB_REPOSITORY` 派生 owner/repo；仅本地调试时允许默认值。

2. 对齐 workflow 输入与脚本实现
- 为 `backfill-duplicate-comments.ts` 增加 `DAYS_BACK` 过滤，或删掉 workflow 输入避免误导。

3. 固化 dedupe 执行契约
- 明确“并行 agent”是否为强依赖；若是，补充对应可用能力约束；若否，改写为纯脚本可执行流程。

4. 增加最小测试集
- Shell：对 `gh.sh`、`edit-issue-labels.sh`、`comment-on-duplicates.sh` 做参数/错误路径测试。
- TS：对 duplicate 编号抽取和 auto-close 判定函数做单测。
- Workflow：提供 dry-run 级联验证作业。

5. 收敛公开触发面
- 将 `allowed_non_write_users` 从 `*` 收敛到必要白名单。

6. 合并重复命令源
- 采用“单一源文件 + 生成/同步”机制，避免项目命令与插件命令长期漂移。
