# DIR `.claude` 研究报告

## 场景与职责

`.claude` 目录在本仓库承担“项目级 Claude 命令入口”职责，当前仅包含 `commands/`，定义 3 个可被 Claude Code 识别的 slash-command：

- `dedupe.md`：用于 issue 重复问题识别与回帖。
- `triage-issue.md`：用于 issue 分类与生命周期标签维护。
- `commit-push-pr.md`：用于本地 git 一键提交/推送/开 PR。

从运行场景看，前两者由 GitHub Actions 自动触发，第三个主要用于人工交互（本地或 Claude 会话中主动调用）。因此 `.claude` 实际是“GitHub 议题自动化流程 + 开发者 git 工作流”的提示词与工具权限编排层。

## 功能点目的

### 1) `/dedupe`

目标：为新 issue 自动找最多 3 个潜在重复项并发表评论，支撑后续自动关闭重复 issue 流程。

设计意图：
- 限制工具面，仅允许 `./scripts/gh.sh` 与 `./scripts/comment-on-duplicates.sh`。
- 先判定是否应跳过（已关闭、非去重场景、已有重复评论），减少误报。
- 通过多 agent 并行搜索 + 二次过滤，提升召回与准确率平衡。

### 2) `/triage-issue`

目标：对新 issue 或评论事件进行标签治理，不发评论，只改标签。

设计意图：
- 强约束标签来源：必须先拉取现有标签列表，避免“幻觉标签”。
- 区分事件语义：
  - `issues`：做类别与生命周期标签判断。
  - `issue_comment`：只处理生命周期标签增删，不改类别标签。
- 把生命周期标签（`needs-repro`/`needs-info`）和自动关闭策略对齐。

### 3) `/commit-push-pr`

目标：将本地变更串成单次提交并发起 PR，减少手工 git 操作。

设计意图：
- 通过 frontmatter `allowed-tools` 将能力收敛到 `git` 与 `gh pr create`。
- 在 prompt 中强制“一次消息完成全部 tool 调用”，减少中间状态漂移。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 命令声明协议（frontmatter）

`.claude/commands/*.md` 统一使用 frontmatter：
- `description`：命令功能简介。
- `allowed-tools`：允许的工具白名单（例如 `Bash(./scripts/gh.sh:*)`）。

这层配置是执行安全边界的第一道门，约束模型只能调用预设命令集。

### B. `/dedupe` 关键流程

1. `claude-dedupe-issues.yml` 在 `issues.opened` 或手工 `workflow_dispatch` 触发。  
2. 工作流调用 `anthropics/claude-code-action@v1`，prompt 为：
   - `/dedupe owner/repo/issues/<number>`
3. `dedupe.md` 指导模型：
   - 读取 issue / comments。
   - 并行搜索重复项。
   - 过滤候选。
   - 调用 `./scripts/comment-on-duplicates.sh --base-issue ... --potential-duplicates ...` 回帖。
4. 后续由 `auto-close-duplicates.yml` 的 `scripts/auto-close-duplicates.ts` 定时检查：
   - 重复提示评论超过 3 天。
   - 无后续人工评论。
   - 原作者未对该评论点踩（`-1`）。
   - 满足条件则自动关闭为 duplicate 并补充说明评论。

关键命令与规则：
- `scripts/gh.sh`：只允许 `issue view/list`、`search issues`、`label list`，并限制 flags。
- `scripts/comment-on-duplicates.sh`：最多接受 3 个重复 issue，且会校验 issue 是否存在。

### C. `/triage-issue` 关键流程

1. `claude-issue-triage.yml` 在 `issues.opened` 与 `issue_comment.created` 触发。  
2. 工作流 `if` 过滤：
   - 评论事件仅处理非 PR 且非 Bot 评论。
3. 同 issue 采用 `concurrency`（`issue-triage-<number>`）防并发覆盖。  
4. prompt 为：
   - `/triage-issue REPO: <repo> ISSUE_NUMBER: <n> EVENT: <event>`
5. `triage-issue.md` 指导模型：
   - 先读标签全集、issue 正文、评论上下文。
   - `issues` 事件：判断是否 `invalid`、类别标签、是否需要生命周期标签。
   - `issue_comment` 事件：仅做生命周期标签增删。
6. 标签落地通过：
   - `scripts/edit-issue-labels.sh --issue ... --add-label ... --remove-label ...`

### D. 生命周期治理的并行子系统（与 `.claude` 强关联）

虽然脚本不在 `.claude` 下，但与 `/triage-issue` 同域协作：

- `scripts/issue-lifecycle.ts`：生命周期标签单一事实源（label、days、reason、nudge）。
- `scripts/lifecycle-comment.ts`：打上生命周期标签时发提醒评论。
- `scripts/sweep.ts`：按超时关闭长期无响应 issue，并在有人类新评论时跳过。

这使 triage 的标签行为与定时收敛策略闭环。

## 关键代码路径与文件引用

### `.claude` 直接对象
- `.claude/commands/dedupe.md`
- `.claude/commands/triage-issue.md`
- `.claude/commands/commit-push-pr.md`

### 调用方（触发 `.claude` 命令）
- `.github/workflows/claude-dedupe-issues.yml`
- `.github/workflows/claude-issue-triage.yml`

### 被调用方（命令落地脚本）
- `scripts/gh.sh`
- `scripts/comment-on-duplicates.sh`
- `scripts/edit-issue-labels.sh`

### 同域协作与后处理
- `.github/workflows/backfill-duplicate-comments.yml` -> `scripts/backfill-duplicate-comments.ts`
- `.github/workflows/auto-close-duplicates.yml` -> `scripts/auto-close-duplicates.ts`
- `.github/workflows/issue-lifecycle-comment.yml` -> `scripts/lifecycle-comment.ts`
- `.github/workflows/sweep.yml` -> `scripts/sweep.ts`
- `scripts/issue-lifecycle.ts`

### 文档/插件上下文
- `plugins/commit-commands/commands/commit-push-pr.md`（与 `.claude/commands/commit-push-pr.md` 内容基本同构）
- `plugins/commit-commands/README.md`
- `.claude-plugin/marketplace.json`

## 依赖与外部交互

### 运行时依赖
- GitHub Actions：工作流触发与 CI 环境。
- `anthropics/claude-code-action@v1`：执行 slash-command。
- GitHub CLI `gh`：由 shell 脚本用于 issue 查询、标签编辑、评论。
- Bun：运行 TypeScript 自动化脚本。

### 外部系统/API
- GitHub REST API：
  - 由 `gh` CLI 或 `fetch https://api.github.com/...` 调用。
- Statsig Events API：
  - `claude-dedupe-issues.yml`、`log-issue-events.yml` 发送埋点。

### 关键环境变量/密钥
- `GITHUB_TOKEN` / `GH_TOKEN`
- `GH_REPO` / `GITHUB_REPOSITORY`
- `ANTHROPIC_API_KEY`
- `STATSIG_API_KEY`
- `GITHUB_REPOSITORY_OWNER`、`GITHUB_REPOSITORY_NAME`

## 风险、边界与改进建议

### 风险与边界

1. 仓库硬编码风险  
- `scripts/comment-on-duplicates.sh` 固定 `REPO="anthropics/claude-code"`。  
- `scripts/backfill-duplicate-comments.ts` 固定 `owner="anthropics"`、`repo="claude-code"`。  
影响：复用到 fork/其他仓库时会误指向原仓库。

2. 参数与实现不一致风险  
- `backfill-duplicate-comments.yml` 传入 `DAYS_BACK`，但脚本未消费该变量。  
影响：操作者以为可控回溯窗口，实际无效。

3. 提示词与工具声明张力  
- `dedupe.md` 要求“使用 agent 并行”，但 frontmatter 仅声明 Bash 脚本工具。  
影响：不同执行环境下可能出现能力不匹配或行为漂移。

4. 权限面风险  
- triage/dedupe 工作流都启用了 `allowed_non_write_users: "*"`。  
虽然仓库有 `non-write-users-check.yml` 做变更审计，但该配置本身仍提高触发面。

5. 测试覆盖不足  
- 当前未看到针对 `.claude/commands` 与相关脚本的自动化测试（单测/集成测试）入口。  
影响：规则变更后容易在运行时才暴露回归。

### 改进建议

1. 去硬编码与配置化
- 统一改为从 `GH_REPO`/`GITHUB_REPOSITORY` 派生 owner/repo，保留默认值仅用于本地调试。

2. 对齐 workflow 输入与脚本实现
- 为 `backfill-duplicate-comments.ts` 增加 `DAYS_BACK` 过滤逻辑，或移除 workflow 输入避免误导。

3. 明确 `dedupe` 的工具契约
- 若确需 agent 并行，显式在可用工具/执行策略中声明；否则将 prompt 改为纯 Bash 可执行流程。

4. 增加最小可行测试
- shell 脚本：参数校验与错误路径测试（bats/shunit2）。
- TS 脚本：对核心判定函数（如 duplicate 提取、关闭条件）做单测。
- workflow：增加 dry-run 回归检查 job。

5. 降低非写用户触发面
- 将 `allowed_non_write_users` 从 `*` 收敛到白名单或组织成员范围，并保留现有审计工作流。
