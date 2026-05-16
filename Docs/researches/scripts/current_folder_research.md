# DIR `scripts` 研究文档

## 场景与职责

`scripts/` 是仓库 GitHub Issue 治理自动化的执行层，承接两类入口：

1. Claude Code slash-command 入口（`.claude/commands/*.md`）：
- `/triage-issue` 通过 `scripts/gh.sh` + `scripts/edit-issue-labels.sh` 进行只读分析与标签变更。
- `/dedupe` 通过 `scripts/gh.sh` + `scripts/comment-on-duplicates.sh` 搜索并写入重复 issue 提示评论。

2. GitHub Actions 定时/事件入口（`.github/workflows/*.yml`）：
- `sweep.ts` 负责 stale 打标与生命周期超时关闭。
- `lifecycle-comment.ts` 在生命周期标签被打上时自动留言。
- `auto-close-duplicates.ts` 根据“重复候选评论+静默期”自动关单。
- `backfill-duplicate-comments.ts` 触发 dedupe workflow 回填历史 issue 的重复检测评论。

从职责边界看，`scripts/` 不负责模型推理本身，而负责“可执行治理动作”：读写 issue、读写标签、发评论、关单、调 workflow dispatch。

## 功能点目的

1. `gh.sh`
- 目的：作为 `gh` CLI 安全包装器，限制允许的子命令和参数，避免任意 GitHub 命令执行。

2. `edit-issue-labels.sh`
- 目的：仅对仓库真实存在的标签做 add/remove，降低 triage 误打标签风险。

3. `comment-on-duplicates.sh`
- 目的：把 dedupe 结果标准化成固定格式评论，给 issue 作者 3 天申诉窗口。

4. `issue-lifecycle.ts`
- 目的：集中维护生命周期策略常量（标签、超时天数、提示文案、upvote 阈值）。

5. `lifecycle-comment.ts`
- 目的：当 issue 被加生命周期标签后，自动发“超时关闭预告”评论。

6. `sweep.ts`
- 目的：定时执行两阶段治理：
- 阶段 A：对长期不活跃 issue 打 `stale`。
- 阶段 B：对生命周期标签超时且无人类反馈的 issue 自动关闭。

7. `auto-close-duplicates.ts`
- 目的：对已有“possible duplicate”提示且 3 天无异议的 issue 自动按 duplicate 关闭。

8. `backfill-duplicate-comments.ts`
- 目的：批量触发 `claude-dedupe-issues.yml`，为历史 issue 回填重复检测评论。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 命令受限协议（`gh.sh`）

`scripts/gh.sh` 做了三层白名单约束：

1. 子命令白名单：仅允许
- `issue view`
- `issue list`
- `search issues`
- `label list`

2. 参数白名单：仅允许
- `--comments`
- `--state`
- `--limit`
- `--label`

3. 查询防逃逸
- `search issues` 的 query 禁止包含 `repo:`/`org:`/`user:`，避免跨仓搜索绕过。
- 仓库作用域强制来自 `GH_REPO` 或 `GITHUB_REPOSITORY`（`owner/repo` 格式）。

该协议直接被 `.claude/commands/triage-issue.md` 与 `.claude/commands/dedupe.md` 当作唯一 GitHub 读取入口使用。

### 2) 生命周期核心数据结构（`issue-lifecycle.ts`）

`lifecycle` 是 `as const` 的只读策略数组，每项结构：
- `label`: 生命周期标签名
- `days`: 超时天数
- `reason`: 关闭说明理由
- `nudge`: 预警评论文案

当前策略：
- `invalid`（3d）
- `needs-repro`（7d）
- `needs-info`（7d）
- `stale`（14d）
- `autoclose`（14d）

并定义 `STALE_UPVOTE_THRESHOLD = 10`，用于保护高关注 issue 不被自动 stale/close。

### 3) 生命周期执行流程（`sweep.ts` + `lifecycle-comment.ts`）

1. `sweep.ts` 主流程
- `markStale(owner, repo)`：
- 分页拉取 open issues（按 `updated asc`），跳过 PR/locked/有 assignee 的 issue。
- 当遇到 `updated_at > stale cutoff` 直接 `return`（利用升序提前结束）。
- 排除已有 `stale/autoclose` 与高赞 issue（`+1 >= 10`）。
- 其余 issue 加 `stale` 标签。

- `closeExpired(owner, repo)`：
- 遍历 `lifecycle` 中每个标签。
- 拉取带该标签的 open issues。
- 查 issue events，取“该标签最近一次 labeled 时间”。
- 若 labeled 时间超过阈值，再检查 labeled 后评论；存在非 Bot 评论则跳过。
- 否则先发关闭说明评论，再 PATCH 关闭（`state_reason: not_planned`）。

2. `lifecycle-comment.ts`
- 依赖环境变量 `GITHUB_REPOSITORY/LABEL/ISSUE_NUMBER`。
- 在 `issues.labeled` 事件中查找 `lifecycle` 对应项并 POST 评论：
- `entry.nudge + "This issue will be closed automatically ..."`

### 4) 重复 issue 自动化流程（dedupe/comment/close/backfill）

1. 首次评论
- Claude slash-command `/dedupe` 最终调用：
- `./scripts/comment-on-duplicates.sh --base-issue <N> --potential-duplicates <...>`
- 该脚本校验 base 与重复候选 issue 存在性，并写入统一模板评论（最多 3 条候选）。

2. 3 天后自动关闭（`auto-close-duplicates.ts`）
- 拉取 open issues（创建超过 3 天）。
- 对每个 issue 找“Bot 发布且文案含 `Found` + `possible duplicate`”的评论。
- 若最近重复评论已超过 3 天，且之后无新评论，且作者未对该评论点 `-1`，则关闭 issue：
- PATCH issue：`state=closed, state_reason=duplicate, labels=['duplicate']`
- 再追加自动关闭说明评论。

3. 历史回填（`backfill-duplicate-comments.ts`）
- 按 issue 编号区间扫描 issue（默认 `1..4049`）。
- 若无重复检测评论，则调用 GitHub Workflow Dispatch API 触发：
- `claude-dedupe-issues.yml`，input 为 `issue_number`。
- 默认 `DRY_RUN=true`，实际触发需 `DRY_RUN=false`。

### 5) 关键命令与 API 端点

1. 本地命令
- `gh issue view/list/search`
- `gh label list`
- `gh issue edit/comment`
- `bun run scripts/*.ts`（CI 内）

2. GitHub REST API（`fetch`）
- `POST /repos/{owner}/{repo}/issues/{n}/comments`
- `POST /repos/{owner}/{repo}/issues/{n}/labels`
- `PATCH /repos/{owner}/{repo}/issues/{n}`
- `GET /repos/{owner}/{repo}/issues?...`
- `GET /repos/{owner}/{repo}/issues/{n}/events`
- `GET /repos/{owner}/{repo}/issues/comments/{comment_id}/reactions`
- `POST /repos/{owner}/{repo}/actions/workflows/claude-dedupe-issues.yml/dispatches`

## 关键代码路径与文件引用

1. 脚本目录核心文件
- `scripts/gh.sh`
- `scripts/edit-issue-labels.sh`
- `scripts/comment-on-duplicates.sh`
- `scripts/issue-lifecycle.ts`
- `scripts/lifecycle-comment.ts`
- `scripts/sweep.ts`
- `scripts/auto-close-duplicates.ts`
- `scripts/backfill-duplicate-comments.ts`

2. 直接调用方（命令层）
- `.claude/commands/triage-issue.md` -> `scripts/gh.sh` + `scripts/edit-issue-labels.sh`
- `.claude/commands/dedupe.md` -> `scripts/gh.sh` + `scripts/comment-on-duplicates.sh`

3. 直接调用方（Workflow 层）
- `.github/workflows/claude-issue-triage.yml`（触发 `/triage-issue`）
- `.github/workflows/claude-dedupe-issues.yml`（触发 `/dedupe`）
- `.github/workflows/sweep.yml` -> `bun run scripts/sweep.ts`
- `.github/workflows/issue-lifecycle-comment.yml` -> `bun run scripts/lifecycle-comment.ts`
- `.github/workflows/auto-close-duplicates.yml` -> `bun run scripts/auto-close-duplicates.ts`
- `.github/workflows/backfill-duplicate-comments.yml` -> `bun run scripts/backfill-duplicate-comments.ts`

4. 关联治理链路（非直接调用但同域协作）
- `.github/workflows/remove-autoclose-label.yml`：issue 新评论后自动移除 `autoclose`。
- `.github/workflows/lock-closed-issues.yml`：关闭 7 天后自动加锁。
- `.github/ISSUE_TEMPLATE/*.yml`：决定 issue 输入质量，影响 triage/lifecycle 误判率。

## 依赖与外部交互

1. 运行依赖
- Shell：`bash`, `grep`, `sed`, `tr`
- GitHub CLI：`gh`
- JS Runtime：`bun`（CI 中通过 `oven-sh/setup-bun@v2`）

2. 凭据与环境变量
- 必需：`GITHUB_TOKEN`
- 仓库作用域：`GITHUB_REPOSITORY` 或 `GITHUB_REPOSITORY_OWNER` + `GITHUB_REPOSITORY_NAME` 或 `GH_REPO`
- 生命周期评论输入：`LABEL`, `ISSUE_NUMBER`
- backfill 控制：`DRY_RUN`, `MAX_ISSUE_NUMBER`, `MIN_ISSUE_NUMBER`

3. 外部平台交互
- GitHub REST API（issue/comment/label/events/reactions/workflow-dispatch）
- GitHub Actions 运行器与权限模型（`issues: write`、`actions: write` 等）

4. 测试与验证现状
- 目录内无单元测试或集成测试（未见 `test`/`spec`）。
- 当前可见质量门禁以运行时校验为主（参数校验、白名单、workflow 权限）。
- 本地环境缺少 `bun`/`gh` 时无法做端到端 dry-run；仅可做 Bash 语法检查。

## 风险、边界与改进建议

1. 仓库目标硬编码风险
- `comment-on-duplicates.sh` 与 `backfill-duplicate-comments.ts` 固定 `anthropics/claude-code`。
- 建议：统一改为读取 `GH_REPO`/`GITHUB_REPOSITORY`，或由 workflow inputs 注入。

2. 配置漂移风险（文档/工作流与脚本不一致）
- `backfill-duplicate-comments.yml` 暴露 `days_back` 输入，但脚本未消费 `DAYS_BACK`，实际按 issue 编号区间工作。
- 建议：删除无效输入，或在脚本中实现按时间窗口过滤，避免运维误判。

3. 误关单风险（重复目标提取规则过宽）
- `auto-close-duplicates.ts` 使用 `/#(\d+)/` 优先匹配，可能提取到评论中的非“重复目标 issue”编号。
- 建议：改为解析“候选列表结构”或限定 URL 前缀来源，必要时只接受仓库 issue URL。

4. 分页/流量控制边界
- 多处默认每页 100 且页数上限固定（10/20/200），超大仓库可能漏扫。
- 无 rate-limit 感知退避（429/secondary rate limit）与重试。
- 建议：增加 Link-header 翻页、指数退避、可配置扫描窗口。

5. 文案耦合风险
- 自动关闭逻辑依赖评论文本包含 `"Found"` 与 `"possible duplicate"`，属于脆弱字符串协议。
- 建议：在评论中加入机器可读 marker（如 HTML 注释 `<!-- dedupe-bot:v1 -->`）并按 marker 检测。

6. 生命周期并发一致性边界
- `sweep.ts` 通过“labeled 后是否有 human comment”做安全兜底，但 triage/remove-autoclose workflow 也在并行改标签。
- 建议：为关键操作增加幂等检查与审计日志字段（例如把触发依据写入 comment metadata）。

7. 测试覆盖不足
- 关键治理脚本均无回归测试，策略变更容易在生产仓库直接暴露。
- 建议：新增 `scripts/tests/`，至少覆盖：
- `gh.sh` 参数白名单
- 生命周期 cutoff 计算
- duplicate comment 解析与反例
- backfill 的区间/dry-run行为
