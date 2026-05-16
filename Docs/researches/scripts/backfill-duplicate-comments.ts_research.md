# FILE `scripts/backfill-duplicate-comments.ts` 研究文档

## 场景与职责

`scripts/backfill-duplicate-comments.ts` 用于“历史回填”重复检测，不直接判断重复，而是批量触发已有的 dedupe workflow，让旧 issue 也补上 `possible duplicate` 评论。

它是一次性/运维型脚本，默认安全模式（`DRY_RUN=true`），主要由 `.github/workflows/backfill-duplicate-comments.yml` 手动触发。

## 功能点目的

1. 扫描指定 issue 编号区间（默认 `#1` 到 `<#4050`）。
2. 识别哪些 issue 还没有 dedupe bot 评论。
3. 对缺失评论的 issue 调用 Workflow Dispatch API 触发 `claude-dedupe-issues.yml`。
4. 支持 dry-run，仅打印计划动作不执行。
5. 控制触发节奏（每次 dispatch 后 sleep 1 秒）避免瞬时打满 Actions。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 核心结构与调用封装

- `GitHubIssue` / `GitHubComment` 接口定义了脚本真正依赖的字段。
- `githubRequest<T>()` 封装 GitHub REST 请求，统一鉴权与异常处理：
  - `Authorization: Bearer <token>`
  - `Accept: application/vnd.github.v3+json`
  - `User-Agent: backfill-duplicate-comments-script`

### 2) 工作流触发协议

`triggerDedupeWorkflow()` 调用：

- `POST /repos/{owner}/{repo}/actions/workflows/claude-dedupe-issues.yml/dispatches`
- Body:
  - `ref: 'main'`
  - `inputs.issue_number: '<issue number>'`

当 `dryRun=true` 时不发送请求，仅打印 `[DRY RUN] Would trigger...`。

### 3) 扫描与筛选主流程

`backfillDuplicateComments()` 主要逻辑：

1. 校验 `GITHUB_TOKEN`；缺失时抛出带 usage 的错误信息。
2. 读取控制参数：
   - `DRY_RUN`（默认 true）
   - `MAX_ISSUE_NUMBER`（默认 4050）
   - `MIN_ISSUE_NUMBER`（默认 1）
3. 固定仓库 `owner="anthropics"`、`repo="claude-code"`。
4. 分页拉取 `state=all` issues（`sort=created&direction=desc`，每页 100，最多 200 页）。
5. 过滤 issue 编号仅保留 `[minIssueNumber, maxIssueNumber)`。
6. 针对每个 issue 拉取评论，检查是否已存在 dedupe 评论（`Found` + `possible duplicate` + `user.type === "Bot"`）。
7. 若不存在 dedupe 评论，则触发 `claude-dedupe-issues.yml`（或 dry-run 打印）。
8. 每个候选处理后等待 1 秒。

## 关键代码路径与文件引用

- 主实现：`scripts/backfill-duplicate-comments.ts`
- 被触发 workflow：`.github/workflows/claude-dedupe-issues.yml`
- 调用入口 workflow：`.github/workflows/backfill-duplicate-comments.yml`
- dedupe command 协议：`.claude/commands/dedupe.md`
- 与评论格式耦合脚本：`scripts/comment-on-duplicates.sh`

关键函数：

- `githubRequest()`：API 层。
- `triggerDedupeWorkflow()`：dispatch 层。
- `backfillDuplicateComments()`：分页、筛选、节流、统计。

## 依赖与外部交互

1. 运行依赖
- Bun 运行时。
- GitHub Actions（常见执行环境）或本地具备网络访问环境。

2. 环境变量
- 必需：`GITHUB_TOKEN`（需 `actions:write` + issues 读取能力）。
- 可选：`DRY_RUN`、`MAX_ISSUE_NUMBER`、`MIN_ISSUE_NUMBER`。

3. 外部 API
- `GET /repos/{owner}/{repo}/issues?state=all...`
- `GET /repos/{owner}/{repo}/issues/{issue_number}/comments`
- `POST /repos/{owner}/{repo}/actions/workflows/claude-dedupe-issues.yml/dispatches`

4. 与上游/下游协作
- 上游：运维手动 dispatch `backfill-duplicate-comments.yml`。
- 下游：每次 dispatch 会拉起 `claude-dedupe-issues.yml`，再通过 `/dedupe` 链路决定是否发重复评论。

## 风险、边界与改进建议

1. 配置漂移
- workflow 暴露了 `days_back` (`DAYS_BACK`)，脚本完全未读取，实际是按 issue 编号区间运行。
- 建议统一参数语义：要么删除 `days_back`，要么实现按时间窗口过滤。

2. 仓库硬编码
- `owner/repo` 固定为 `anthropics/claude-code`，不随运行上下文变化。
- 建议改为读取 `GITHUB_REPOSITORY` 或 workflow input。

3. 文本协议耦合
- “是否已有 dedupe 评论”依赖文本关键词，易受文案修改影响。
- 建议引入机器 marker 或评论 metadata。

4. PR 混入风险
- `/issues` 接口返回 PR，脚本未显式排除 `pull_request`。
- 建议过滤 PR，避免对 PR 编号触发 issue dedupe 流程。

5. 触发风暴风险
- 大区间 + `DRY_RUN=false` 会触发大量 workflow，1 秒间隔仍可能消耗 Actions 配额。
- 建议增加每次执行上限、断点续跑、失败重试与速率控制。

6. 分页与停止条件复杂
- 当前通过“当前页最老 issue 编号”做启发式停止，行为依赖 issue 编号与创建时间单调关系。
- 建议补充明确终止条件日志，并增加单元测试覆盖边界页。

7. 幂等性与可观测性不足
- 仅依靠“有没有 dedupe 评论”判断幂等，且无外部审计记录。
- 建议输出结构化结果（JSON summary）并落地 artifact，便于复盘。
