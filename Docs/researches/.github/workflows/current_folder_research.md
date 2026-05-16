# DIR 研究：`.github/workflows`

## 场景与职责
`.github/workflows` 是本仓库在 GitHub 平台上的自动化编排层，核心职责是把「Issue/PR 事件、定时任务、手动触发」路由到 Claude Code Agent、Bun 脚本、GitHub Script 与外部系统（Statsig、跨仓库 dispatch），实现仓库治理闭环。

从职责分层看，该目录承担 4 类能力：

1. 人机协同入口：`@claude` 唤起与 Issue 自动分拣。
2. 生命周期治理：标记 `stale`、自动关单、锁定已关闭主题、评论触发取消 `autoclose`。
3. 重复问题治理：重复检测、回填历史 issue 的重复评论、延时自动关重。
4. 观测与联动：事件上报 Statsig、向目标仓库发送 `repository_dispatch`。

工作流总览（12 个）：

| 工作流 | 触发 | 主要动作 | 下游 |
|---|---|---|---|
| `claude.yml` | `issue_comment`/`pull_request_review_comment`/`issues`/`pull_request_review` | `@claude` 命中后执行 Claude Action | `anthropics/claude-code-action@v1` |
| `claude-issue-triage.yml` | `issues.opened`、`issue_comment.created` | 运行 `/triage-issue` 只改标签 | `.claude/commands/triage-issue.md` -> `scripts/gh.sh`/`scripts/edit-issue-labels.sh` |
| `claude-dedupe-issues.yml` | `issues.opened`、`workflow_dispatch` | 运行 `/dedupe` + Statsig 事件上报 | `.claude/commands/dedupe.md` -> `scripts/gh.sh`/`scripts/comment-on-duplicates.sh` |
| `auto-close-duplicates.yml` | `cron` + 手动 | 关闭满足条件的重复 issue | `scripts/auto-close-duplicates.ts` |
| `backfill-duplicate-comments.yml` | 手动 | 扫历史 issue 并触发 dedupe workflow_dispatch | `scripts/backfill-duplicate-comments.ts` |
| `issue-lifecycle-comment.yml` | `issues.labeled` | 生命周期标签落地提醒评论 | `scripts/lifecycle-comment.ts` |
| `sweep.yml` | `cron` + 手动 | 标记 stale + 按生命周期关闭 | `scripts/sweep.ts` -> `scripts/issue-lifecycle.ts` |
| `remove-autoclose-label.yml` | `issue_comment.created` | 活跃后移除 `autoclose` | `actions/github-script@v7` |
| `lock-closed-issues.yml` | `cron` + 手动 | 关闭 7 天后锁帖 | `actions/github-script@v7` |
| `log-issue-events.yml` | `issues.opened/closed` | 上报 issue 事件到 Statsig | `curl https://events.statsigapi.net/v1/log_event` |
| `issue-opened-dispatch.yml` | `issues.opened` | 跨仓库 `repository_dispatch` | `gh api repos/{repo}/dispatches` |
| `non-write-users-check.yml` | `.github/**` 相关 PR | 检测 `allowed_non_write_users` 变更并评论告警 | `gh pr diff/view/comment` |

## 功能点目的
### 1) Claude 交互触发
- `claude.yml` 用于通用 `@claude` 响应，覆盖 issue 评论、PR 评论、review 内容与 issue 标题/正文 mention（`.github/workflows/claude.yml:3-37`）。
- 目标是让仓库维护者与贡献者无需本地环境也可在 GitHub 线程内直接调用 Claude。

### 2) 自动分拣与标签治理
- `claude-issue-triage.yml` 在新 issue 与非 Bot 评论时运行，使用 issue 维度并发组避免同一 issue 并行冲突（`.github/workflows/claude-issue-triage.yml:12-17`）。
- `/triage-issue` 强约束「只改标签不发评论」（`.claude/commands/triage-issue.md:8,55-63`），降低噪音并防止自动回复失控。

### 3) 重复问题处理链路
- `claude-dedupe-issues.yml` 在 issue 新建或手工输入 issue_number 时触发（`.github/workflows/claude-dedupe-issues.yml:3-12`）。
- `/dedupe` 指令通过受限工具查重并发表评论（`.claude/commands/dedupe.md:1-27`）。
- `auto-close-duplicates.yml` 延迟 3 天后收敛重复单（`scripts/auto-close-duplicates.ts:112-115,189-214`），防止误杀。
- `backfill-duplicate-comments.yml` 用于历史数据补齐（`.github/workflows/backfill-duplicate-comments.yml:1-44`）。

### 4) 生命周期自动化
- `issue-lifecycle-comment.yml` 在标签被打上时通知作者倒计时（`.github/workflows/issue-lifecycle-comment.yml:3-27`）。
- `sweep.yml` 按生命周期配置自动打 `stale` 与关单（`.github/workflows/sweep.yml:3-31`，`scripts/issue-lifecycle.ts:3-34`）。
- `remove-autoclose-label.yml` 作为活动恢复兜底，出现新人工评论时移除 `autoclose`（`.github/workflows/remove-autoclose-label.yml:13-32`）。
- `lock-closed-issues.yml` 对关闭且 7 天无活动的 issue 自动加评论并 lock（`.github/workflows/lock-closed-issues.yml:23-92`）。

### 5) 观测与跨仓联动
- `log-issue-events.yml` 将 issue 事件写入 Statsig（`.github/workflows/log-issue-events.yml:25-40`）。
- `claude-dedupe-issues.yml` 也会在 dedupe 执行后记录事件（`.github/workflows/claude-dedupe-issues.yml:36-83`）。
- `issue-opened-dispatch.yml` 将新 issue 派发到目标仓库（`.github/workflows/issue-opened-dispatch.yml:24-27`）。

## 具体技术实现（关键流程/数据结构/协议/命令）
### A. 触发与并发控制
- 事件触发：`issues`、`issue_comment`、`pull_request_review_comment`、`pull_request_review`、`schedule`、`workflow_dispatch`。
- 并发：
  - triage 使用 `issue-triage-${{ github.event.issue.number }}`，同 issue 新事件会 `cancel-in-progress: true`（`.github/workflows/claude-issue-triage.yml:15-17`）。
  - sweep/lock 分别使用 `daily-issue-sweep`、`lock-threads` 防止重入（`.github/workflows/sweep.yml:11-13`，`.github/workflows/lock-closed-issues.yml:12-13`）。

### B. Claude Action 协议层
- 统一调用：`anthropics/claude-code-action@v1`。
- triage prompt：`/triage-issue REPO: ... ISSUE_NUMBER: ... EVENT: ...`（`.github/workflows/claude-issue-triage.yml:35`）。
- dedupe prompt：`/dedupe owner/repo/issues/{number}`（`.github/workflows/claude-dedupe-issues.yml:32`）。
- `allowed_non_write_users: "*"` 出现在 triage 与 dedupe 两条流（`.github/workflows/claude-issue-triage.yml:34`，`.github/workflows/claude-dedupe-issues.yml:31`）。

### C. 脚本的数据结构与判定逻辑
1. `scripts/issue-lifecycle.ts`
- 常量 `lifecycle[]` 是生命周期单一事实源，元素结构：`{ label, days, reason, nudge }`（`scripts/issue-lifecycle.ts:3-34`）。
- `STALE_UPVOTE_THRESHOLD=10` 参与 sweep 跳过高关注 issue（`scripts/issue-lifecycle.ts:38`）。

2. `scripts/sweep.ts`
- `markStale`：按 `updated_at asc` 拉取 open issues，遇到新于 cutoff 的 issue 立即停止扫描（`scripts/sweep.ts:54-67`）。
- `closeExpired`：按生命周期标签遍历，读取 issue events 找最近一次 label 时间，再检查 label 之后是否有人类评论（`scripts/sweep.ts:115-133`）。
- 关闭动作：先评论再 `PATCH state=closed, state_reason=not_planned`（`scripts/sweep.ts:144-146`）。

3. `scripts/lifecycle-comment.ts`
- 从 `LABEL` 查 `lifecycle[]` 映射，若不在配置中则直接退出（`scripts/lifecycle-comment.ts:19-23`）。
- 通过 GitHub REST `POST /repos/{repo}/issues/{num}/comments` 发提醒（`scripts/lifecycle-comment.ts:34-45`）。

4. `scripts/backfill-duplicate-comments.ts`
- 以 issue number 范围扫描，过滤无“possible duplicate” bot 评论的 issue（`scripts/backfill-duplicate-comments.ts:160-178`）。
- 对候选 issue 调 `POST /actions/workflows/claude-dedupe-issues.yml/dispatches`（`scripts/backfill-duplicate-comments.ts:59-69`）。

5. `scripts/auto-close-duplicates.ts`
- 候选条件：
  - issue 创建超过 3 天（`scripts/auto-close-duplicates.ts:112-134`）。
  - 最近重复评论也超过 3 天（`189-201`）。
  - 之后无人评论（`203-214`）。
  - issue 作者未对该评论点 👎（`228-241`）。
- 收敛动作：`PATCH issues/{num}` 设 `state_reason=duplicate` + 补评论（`73-95`）。

### D. 受限命令包装
- `scripts/gh.sh` 限制可调用子命令与 flags，仅允许 `issue view/list`, `search issues`, `label list`，并禁止查询注入 `repo:/org:/user:`（`scripts/gh.sh:29-35,76-83`）。
- triage 的 label 写入由 `scripts/edit-issue-labels.sh` 完成，先拉取真实标签表再过滤，避免写入不存在 label（`scripts/edit-issue-labels.sh:47-66`）。
- dedupe 的评论写入由 `scripts/comment-on-duplicates.sh` 完成，最多 3 个候选并做 issue 存在性校验（`scripts/comment-on-duplicates.sh:51-75`）。

### E. 外部协议与命令
- GitHub API（REST）
  - `fetch https://api.github.com/...`：脚本层通用请求。
  - `gh api repos/{target}/dispatches`：跨仓 dispatch（`.github/workflows/issue-opened-dispatch.yml:24-27`）。
- Statsig API
  - `POST https://events.statsigapi.net/v1/log_event`，payload 为 `events[]`（`.github/workflows/claude-dedupe-issues.yml:50-74`，`.github/workflows/log-issue-events.yml:25-40`）。

## 关键代码路径与文件引用
### 主路径（调用链）
1. `issues.opened` -> `claude-issue-triage.yml` -> `/triage-issue` -> `scripts/gh.sh` + `scripts/edit-issue-labels.sh`
2. `issues.opened` -> `claude-dedupe-issues.yml` -> `/dedupe` -> `scripts/gh.sh` + `scripts/comment-on-duplicates.sh`
3. `workflow_dispatch(backfill)` -> `scripts/backfill-duplicate-comments.ts` -> dispatch `claude-dedupe-issues.yml`
4. `schedule/workflow_dispatch(sweep)` -> `scripts/sweep.ts` -> `scripts/issue-lifecycle.ts`
5. `issues.labeled` -> `issue-lifecycle-comment.yml` -> `scripts/lifecycle-comment.ts` -> `scripts/issue-lifecycle.ts`
6. `issue_comment.created` -> `remove-autoclose-label.yml` -> `actions/github-script`
7. `schedule/workflow_dispatch(lock)` -> `lock-closed-issues.yml` -> `actions/github-script`
8. `issues.opened` -> `issue-opened-dispatch.yml` -> `gh api repos/{target}/dispatches`
9. `issues.opened/closed` -> `log-issue-events.yml` -> Statsig
10. `schedule/workflow_dispatch` -> `auto-close-duplicates.yml` -> `scripts/auto-close-duplicates.ts`

### 关键文件（含上下文）
- 工作流目录：
  - `.github/workflows/claude.yml`
  - `.github/workflows/claude-issue-triage.yml`
  - `.github/workflows/claude-dedupe-issues.yml`
  - `.github/workflows/auto-close-duplicates.yml`
  - `.github/workflows/backfill-duplicate-comments.yml`
  - `.github/workflows/sweep.yml`
  - `.github/workflows/issue-lifecycle-comment.yml`
  - `.github/workflows/remove-autoclose-label.yml`
  - `.github/workflows/lock-closed-issues.yml`
  - `.github/workflows/log-issue-events.yml`
  - `.github/workflows/issue-opened-dispatch.yml`
  - `.github/workflows/non-write-users-check.yml`
- Claude 命令定义：
  - `.claude/commands/triage-issue.md`
  - `.claude/commands/dedupe.md`
- 脚本：
  - `scripts/sweep.ts`
  - `scripts/issue-lifecycle.ts`
  - `scripts/lifecycle-comment.ts`
  - `scripts/auto-close-duplicates.ts`
  - `scripts/backfill-duplicate-comments.ts`
  - `scripts/gh.sh`
  - `scripts/edit-issue-labels.sh`
  - `scripts/comment-on-duplicates.sh`
- 上游文档/模板：
  - `README.md`（`@claude` 入口描述）
  - `.github/ISSUE_TEMPLATE/*.yml`（初始标签与内容结构，影响 triage 输入质量）

## 依赖与外部交互
### GitHub 平台能力
- GitHub Actions runner: `ubuntu-latest`。
- 权限粒度：`issues:write/read`, `pull-requests:write/read`, `actions:write`, `id-token:write`, `contents:read`。
- Secrets：
  - `ANTHROPIC_API_KEY`
  - `GITHUB_TOKEN`
  - `STATSIG_API_KEY`
  - `ISSUE_OPENED_DISPATCH_TARGET_REPO`
  - `ISSUE_OPENED_DISPATCH_TOKEN`

### 第三方 Actions
- `anthropics/claude-code-action@v1`
- `actions/checkout@v4`（仅 `claude.yml` 使用 commit pin）
- `oven-sh/setup-bun@v2`
- `actions/github-script@v7`

### 外部服务
- GitHub REST API (`api.github.com`)。
- Statsig Events API (`events.statsigapi.net`)。
- 跨仓 repository dispatch（目标仓库由 secret 指定）。

### 测试与验证现状
- 仓库中未发现针对这些 workflow/脚本的自动化测试用例（无专门 `test/spec` 覆盖 `.github/workflows` 与 `scripts/*` 逻辑）。
- 当前可靠性主要依赖：运行时日志 + 手动触发 `workflow_dispatch` 验证。

## 风险、边界与改进建议
### 风险与边界
1. 安全面
- `allowed_non_write_users: "*"` 放大了可触发面，若后续 prompt/tool 权限扩大，可能引入滥用风险（已由 `non-write-users-check.yml` 做弱提醒，但非阻断）。
- 多数 Action 版本未 pin 到 commit SHA（`@v4/@v7/@v2/@v1`），存在供应链漂移风险。

2. 逻辑一致性
- `backfill-duplicate-comments.yml` 暴露 `days_back` 输入，但 `scripts/backfill-duplicate-comments.ts` 实际未使用该变量；脚本采用 issue number 范围策略（`MIN_ISSUE_NUMBER/MAX_ISSUE_NUMBER`）。存在“界面参数与真实行为不一致”。
- `auto-close-duplicates.yml` 传入 `STATSIG_API_KEY`，但 `scripts/auto-close-duplicates.ts` 未消费该变量，属于冗余配置。
- `log-issue-events.yml` 监听 `opened,closed`，但事件名固定为 `github_issue_created` 且时间字段取 `created_at`，对 closed 事件语义不准确。

3. 可靠性
- `issue-opened-dispatch.yml` 捕获失败后 `exit 0`，下游故障会被静默吞掉，易造成“看起来成功，实际上未联动”。
- `remove-autoclose-label.yml` 与 triage comment 逻辑都可能移除生命周期标签，存在功能重叠与潜在竞态。

4. 可维护性
- 仓库名硬编码：`scripts/backfill-duplicate-comments.ts`、`scripts/comment-on-duplicates.sh` 固定 `anthropics/claude-code`，复用到 fork 仓库时行为不一致。
- `schedule` 使用 UTC cron，注释含 Pacific 说明，若维护者按本地时区理解可能误判执行窗口。

### 改进建议（按优先级）
1. 高优先级
- 将 `non-write-users-check.yml` 从“提醒”升级为可选阻断策略（例如通过 required check + label override）。
- 为关键第三方 Action 改为 commit SHA pin，至少覆盖 `anthropics/claude-code-action`、`setup-bun`、`github-script`。
- 修复 `backfill` 参数语义：要么脚本真正实现 `days_back` 过滤，要么 workflow 输入改成 `min_issue_number/max_issue_number`。

2. 中优先级
- 拆分 `log-issue-events.yml` 的 opened/closed 事件名与时间字段，保证统计语义一致。
- 为 `issue-opened-dispatch.yml` 添加失败告警（例如 workflow warning 注释或 retry/backoff），避免静默失败。
- 清理无效 env（如 `auto-close-duplicates.yml` 的 `STATSIG_API_KEY`）。

3. 低优先级
- 为 `scripts/*` 增加最小化 dry-run 集成测试（可通过 `workflow_dispatch` + mock token/fixture issue）。
- 对 `remove-autoclose-label` 与 triage 的标签移除职责做单一化，减少重复逻辑。

