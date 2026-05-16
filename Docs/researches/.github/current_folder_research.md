# .github 目录研究

## 场景与职责

`.github/` 在本仓库承担 GitHub 平台集成层职责，覆盖三类场景：

1. 问题入口标准化：通过 Issue Forms 约束用户提交信息质量，并自动附加基础标签。
2. 问题流转自动化：通过 GitHub Actions 把 issue/comment/schedule 事件路由到 Claude Code Action、Bun 脚本、`gh` 命令与 GitHub REST API。
3. 治理与风控：通过自动 stale/auto-close/lock、非写权限用户检查、事件埋点与跨仓库 dispatch 维持仓库健康与运营可观测性。

从调用关系看，`.github` 不是孤立配置目录，而是仓库自动化编排入口：

- 调用方：GitHub Events（issues、issue_comment、schedule、workflow_dispatch、pull_request）。
- 被调用方：
  - `scripts/*.ts`（Bun 运行）
  - `actions/github-script@v7`
  - `anthropics/claude-code-action@v1`
  - `.claude/commands/*.md`（通过 Claude slash command prompt 间接触发）
  - GitHub/Statsig API 与 `gh` CLI。

## 功能点目的

### 1) Issue 表单层（`.github/ISSUE_TEMPLATE/*.yml`）

- `bug_report.yml`：收集复现步骤、版本、平台、终端等可诊断字段，降低 triage 往返成本。
- `model_behavior.yml`：区分“模型行为偏差”与“程序崩溃类 bug”，强调越权编辑、忽略指令等行为问题。
- `feature_request.yml`：要求问题陈述与场景价值，便于产品优先级判断。
- `documentation.yml`：聚焦文档缺失/错误/不清晰问题。
- `config.yml`：禁用空白 issue，提供 Discord/官方文档/快速开始/排障链接，减少无效问题。

### 2) 自动化工作流层（`.github/workflows/*.yml`）

- 交互触发：
  - `claude.yml`：当文本包含 `@claude` 时运行 Claude Code Action。
  - `claude-issue-triage.yml`：新 issue 或人工评论触发 `/triage-issue`。
  - `claude-dedupe-issues.yml`：新 issue 或手工 dispatch 触发 `/dedupe`。
- 生命周期维护：
  - `sweep.yml`：定时标记 stale + 关闭超时 lifecycle issue。
  - `issue-lifecycle-comment.yml`：给被贴 lifecycle 标签的 issue 自动发提醒评论。
  - `remove-autoclose-label.yml`：用户新评论后移除 `autoclose`。
  - `lock-closed-issues.yml`：关闭后 7 天无活动则锁帖。
- 去重与追补：
  - `auto-close-duplicates.yml`：3 天后自动关闭“疑似重复且无异议”的 issue。
  - `backfill-duplicate-comments.yml`：对历史 issue 触发 dedupe 回填。
- 运维与治理：
  - `log-issue-events.yml`：issue opened/closed 事件写入 Statsig。
  - `issue-opened-dispatch.yml`：将新 issue URL 转发到目标仓库 `repository_dispatch`。
  - `non-write-users-check.yml`：PR 修改 `.github/**` 且引入 `allowed_non_write_users` 时自动提醒安全风险。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. Dedupe 主链路（AI 判重 -> 评论 -> 延迟自动关闭）

1. 触发：`claude-dedupe-issues.yml` 在 `issues.opened` 或 `workflow_dispatch` 执行。
2. 动作：`anthropics/claude-code-action@v1` 使用 prompt：
   - `/dedupe <owner/repo>/issues/<num>`
3. 命令协议（`.claude/commands/dedupe.md`）：只允许 `./scripts/gh.sh` + `./scripts/comment-on-duplicates.sh`。
4. 评论协议（`scripts/comment-on-duplicates.sh`）：
   - 固定模板 `Found N possible duplicate issues:`
   - 最多 3 个候选 issue
   - 声明“3天后自动关闭”，并允许用户通过评论或 👎 阻止自动关闭
5. 延迟关闭（`scripts/auto-close-duplicates.ts`）：
   - 扫描 open issue（创建时间 >3 天）
   - 找“Bot 且包含 `Found` + `possible duplicate`”的评论
   - 要求：该评论时间 >3 天、之后无新评论、原作者未对该评论点 👎
   - 从评论正文提取目标重复 issue 号（`#123` 或 issue URL）
   - PATCH 关闭 issue，`state_reason=duplicate`，并加 `duplicate` label + 关闭说明评论。

### B. Triage 主链路（分类/生命周期标签自动化）

1. 触发：`claude-issue-triage.yml` 在 `issues.opened` 与 `issue_comment.created` 触发，且过滤 bot 评论与 PR 评论场景。
2. 并发控制：按 issue 号并发组 `issue-triage-<num>`，新事件到来会取消旧任务。
3. 动作：`anthropics/claude-code-action@v1` prompt `/triage-issue REPO: ... ISSUE_NUMBER: ... EVENT: ...`。
4. triage 协议（`.claude/commands/triage-issue.md`）：
   - 必须先读 label 列表，禁止发明标签
   - 新 issue 场景可加分类标签 + 生命周期标签（`needs-repro`/`needs-info`）
   - 评论场景重点移除生命周期标签（如用户补充信息后）
   - 禁止发评论，只能编辑标签
5. 标签执行（`scripts/edit-issue-labels.sh`）：
   - 用 `gh label list` 过滤不存在标签
   - 最终调用 `gh issue edit <num> --add-label/--remove-label`。

### C. 生命周期关闭链路（规则中心 + 周期执行）

1. 规则中心（`scripts/issue-lifecycle.ts`）：
   - `lifecycle[]`：`invalid(3d)`, `needs-repro(7d)`, `needs-info(7d)`, `stale(14d)`, `autoclose(14d)`
   - `STALE_UPVOTE_THRESHOLD=10`（高赞 issue 不自动 stale/close）
2. 定时扫库（`scripts/sweep.ts`，由 `sweep.yml` 运行）：
   - `markStale`：按 `updated_at` 升序扫描 open issue，对低活跃且低赞 issue 打 `stale`
   - `closeExpired`：按 lifecycle 标签扫描，基于 issue events 找到标签打上时间，超时则评论后关闭（`state_reason=not_planned`）
   - 安全兜底：若标签后出现非 Bot 评论则跳过关闭
3. 标签提示（`scripts/lifecycle-comment.ts`，由 `issue-lifecycle-comment.yml` 触发）：
   - 当 issue 被贴 lifecycle 标签时，自动发“将在 X 天后关闭”的提醒。
4. 活跃恢复（`remove-autoclose-label.yml`）：
   - 有人类评论即尝试移除 `autoclose`。

### D. 观测与外部路由

- Statsig：
  - `log-issue-events.yml` 上报 issue opened/closed 事件。
  - `claude-dedupe-issues.yml` 在工作流末尾上报 `github_duplicate_comment_added`。
- 跨仓库 dispatch：
  - `issue-opened-dispatch.yml` 通过 `gh api repos/<target>/dispatches` 转发 issue URL。

### E. 运行协议与命令约束

- Bun 工作流统一命令：`bun run scripts/<name>.ts`
- `scripts/gh.sh` 是受限包装器：仅允许 `issue view/list`、`search issues`、`label list`，并限制 flags（`--comments --state --limit --label`）。
- Secrets/Env 以 workflow `env:` 注入，关键变量包括：
  - `GITHUB_TOKEN`
  - `ANTHROPIC_API_KEY`
  - `STATSIG_API_KEY`
  - `ISSUE_OPENED_DISPATCH_TARGET_REPO`
  - `ISSUE_OPENED_DISPATCH_TOKEN`。

## 关键代码路径与文件引用

### 目录与入口

- `.github/workflows/claude-dedupe-issues.yml`
- `.github/workflows/claude-issue-triage.yml`
- `.github/workflows/claude.yml`
- `.github/workflows/sweep.yml`
- `.github/workflows/issue-lifecycle-comment.yml`
- `.github/workflows/auto-close-duplicates.yml`
- `.github/workflows/backfill-duplicate-comments.yml`
- `.github/workflows/remove-autoclose-label.yml`
- `.github/workflows/lock-closed-issues.yml`
- `.github/workflows/log-issue-events.yml`
- `.github/workflows/issue-opened-dispatch.yml`
- `.github/workflows/non-write-users-check.yml`
- `.github/ISSUE_TEMPLATE/bug_report.yml`
- `.github/ISSUE_TEMPLATE/model_behavior.yml`
- `.github/ISSUE_TEMPLATE/feature_request.yml`
- `.github/ISSUE_TEMPLATE/documentation.yml`
- `.github/ISSUE_TEMPLATE/config.yml`

### 关键下游实现

- `.claude/commands/dedupe.md`
- `.claude/commands/triage-issue.md`
- `scripts/gh.sh`
- `scripts/comment-on-duplicates.sh`
- `scripts/edit-issue-labels.sh`
- `scripts/auto-close-duplicates.ts`
- `scripts/backfill-duplicate-comments.ts`
- `scripts/issue-lifecycle.ts`
- `scripts/sweep.ts`
- `scripts/lifecycle-comment.ts`

### 配套流程（研究系统）

- `.ops/research_guard.sh`（生成任务提示，要求研究并勾选 checklist）
- `.ops/generate_research_blueprint_checklist.sh`
- `.ops/generate_daily_research_todo.sh`

## 依赖与外部交互

### GitHub 平台能力

- GitHub Actions runners（`ubuntu-latest`）
- GitHub Actions marketplace actions：
  - `actions/checkout@v4`
  - `actions/github-script@v7`
  - `oven-sh/setup-bun@v2`
  - `anthropics/claude-code-action@v1`
- GitHub REST API（通过 `fetch`/`gh api`/`gh issue`）

### 外部服务

- Anthropic API（Claude Code Action 运行所需）
- Statsig 事件上报 API（`https://events.statsigapi.net/v1/log_event`）

### 认证与配置边界

- 依赖 GitHub Secrets 注入；缺失时部分 workflow 会降级跳过（如 dedupe 的 Statsig 日志步骤）。
- `non-write-users-check.yml` 作为治理补丁，提醒 `allowed_non_write_users` 变更带来的触发面扩大。
- 仓库另有安全提醒 Hook（`plugins/security-guidance/hooks/security_reminder_hook.py`）对 `.github/workflows/*.yml` 编辑注入风险给出提示。

## 风险、边界与改进建议

### 主要风险

1. 判重/自动关闭逻辑对评论文案耦合较强
- `auto-close-duplicates.ts` 通过字符串 `Found` + `possible duplicate` 识别 dedupe 评论；若文案变更会失效。

2. 部分脚本硬编码仓库名，复用性与正确性受限
- `scripts/comment-on-duplicates.sh` 与 `scripts/backfill-duplicate-comments.ts` 固定 `anthropics/claude-code`，在 fork/多仓场景会偏离当前仓库上下文。

3. 风险防护偏“提示型”而非“阻断型”
- `non-write-users-check.yml` 仅留言提醒，不会 fail PR。

4. 数据上报容错与质量控制有限
- Statsig 上报失败不会阻断流程；有利于稳态但可能掩盖长期观测缺口。

5. 自动化策略存在误伤边界
- `sweep.ts` 与 `lock-closed-issues.yml` 基于时间阈值批处理，尽管已有高赞与人工评论兜底，仍可能与真实处理节奏错位。

### 边界条件

- 无专门测试目录或 workflow 单测，当前自动化主要依赖线上运行行为验证。
- 多数脚本有分页上限（如 page<=10/page<=20/page<=200），极端大仓库可能出现扫描盲区。
- `issue-opened-dispatch.yml` 调用失败默认吞错退出，目标仓库不可达时缺少强告警。

### 改进建议

1. 将 dedupe 评论识别从“文案匹配”升级为“结构化标记”
- 在 dedupe 评论加入隐藏标识（例如 HTML 注释标签），`auto-close-duplicates.ts` 按标识识别。

2. 去除硬编码仓库名
- 统一改为读取 `GITHUB_REPOSITORY` / `GH_REPO`，与 workflow 触发仓库保持一致。

3. 为高风险配置引入可选阻断策略
- 对 `allowed_non_write_users` 变更增加可配置 fail 模式（例如仅允许安全团队标签豁免）。

4. 增加 dry-run 与回放测试资产
- 为 `sweep.ts`、`auto-close-duplicates.ts` 增加 fixture 级别的 API 响应回放测试，覆盖分页、反应表态、标签时间计算等关键分支。

5. 加强可观测性
- 对 dispatch/Statsig 失败增加统一指标或告警（至少输出可机器采集的错误计数日志）。
