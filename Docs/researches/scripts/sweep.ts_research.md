# FILE `scripts/sweep.ts` 研究文档

## 场景与职责

`scripts/sweep.ts` 是仓库 issue 生命周期治理的主执行器，负责定时打 `stale` 标签并关闭超时生命周期 issue。

该脚本对应 `.github/workflows/sweep.yml`，按 cron 每日两次执行，也支持手动 dispatch。

## 功能点目的

1. 扫描不活跃 open issue，按规则自动打 `stale`。
2. 处理生命周期标签超时 issue（`invalid/needs-repro/needs-info/stale/autoclose`）。
3. 关闭前检查是否有人类评论作为反证，避免误关。
4. 对高赞 issue（`+1 >= STALE_UPVOTE_THRESHOLD`）进行保护。
5. 支持 `--dry-run`，用于演练和策略验证。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 全局配置与通用请求

- 从 `issue-lifecycle.ts` 导入 `lifecycle` 和 `STALE_UPVOTE_THRESHOLD`。
- `NEW_ISSUE` 固定指向 `https://github.com/anthropics/claude-code/issues/new/choose`。
- `githubRequest<T>()` 封装 REST 调用，要求 `GITHUB_TOKEN`，`404` 时返回空对象，其余非 2xx 抛错。

### 2) 阶段 A：`markStale(owner, repo)`

1. 读取 `stale` 策略天数，计算 `cutoff`。
2. 分页拉取 open issues（`sort=updated&direction=asc`，最多 10 页）。
3. 对每个 issue 依次过滤：
   - 跳过 PR（`pull_request`）。
   - 跳过 locked issue。
   - 跳过有 assignee 的 issue。
   - 若 `updated_at > cutoff`，利用“升序”提前结束整个扫描。
   - 跳过已有 `stale` 或 `autoclose` 标签。
   - 跳过高赞（`+1 >= 10`）issue。
4. 命中候选后：
   - dry-run：仅打印。
   - 非 dry-run：`POST /issues/{n}/labels` 加 `stale`。

### 3) 阶段 B：`closeExpired(owner, repo)`

对 `lifecycle` 中每个标签逐一执行：

1. 计算该 label 的超时 `cutoff`。
2. 分页查询带该 label 的 open issues（最多 10 页）。
3. 过滤 PR / locked / 高赞 issue。
4. 拉取 issue events，找到该 label 最近一次 `labeled` 时间 `labeledAt`。
5. 若 `labeledAt` 不存在或未超时则跳过。
6. 拉取 `since=labeledAt` 的评论，若存在非 Bot 评论则跳过。
7. 满足关闭条件：
   - 先 `POST /issues/{n}/comments` 写关闭说明。
   - 再 `PATCH /issues/{n}` 为 `state=closed`、`state_reason=not_planned`。

### 4) 主入口与运行协议

- 必需环境变量：`GITHUB_REPOSITORY_OWNER`、`GITHUB_REPOSITORY_NAME`、`GITHUB_TOKEN`。
- 执行顺序固定：先 `markStale`，后 `closeExpired`。
- 输出最终汇总：标记数量与关闭数量。

## 关键代码路径与文件引用

- 主实现：`scripts/sweep.ts`
- 生命周期策略：`scripts/issue-lifecycle.ts`
- 提前提醒脚本：`scripts/lifecycle-comment.ts`
- 调用 workflow：`.github/workflows/sweep.yml`
- 活动恢复协同：
  - `.github/workflows/claude-issue-triage.yml`（评论后可移除生命周期标签）
  - `.github/workflows/remove-autoclose-label.yml`

关键函数：

- `githubRequest()`：统一 API 访问层。
- `markStale()`：不活跃打标。
- `closeExpired()`：生命周期超时关闭。
- `CLOSE_MESSAGE()`：关闭评论模板生成。

## 依赖与外部交互

1. 运行依赖
- Bun 运行时。
- GitHub Actions 定时调度环境。

2. 环境变量
- `GITHUB_TOKEN`。
- `GITHUB_REPOSITORY_OWNER`、`GITHUB_REPOSITORY_NAME`。

3. 外部 API
- `GET /repos/{owner}/{repo}/issues?...`
- `POST /repos/{owner}/{repo}/issues/{issue_number}/labels`
- `GET /repos/{owner}/{repo}/issues/{issue_number}/events`
- `GET /repos/{owner}/{repo}/issues/{issue_number}/comments?since=...`
- `POST /repos/{owner}/{repo}/issues/{issue_number}/comments`
- `PATCH /repos/{owner}/{repo}/issues/{issue_number}`

4. 权限需求
- workflow 声明 `issues: write`。

## 风险、边界与改进建议

1. 分页上限可能漏扫
- issue 列表和 label 列表都限制 10 页，仓库规模增大后会遗漏。
- 建议改为基于 Link Header 的完整分页或配置化页上限。

2. 事件/评论分页不完整
- `events?per_page=100`、`comments?per_page=100` 未翻页。
- 高活跃 issue 可能拿不到真正最近 `labeledAt` 或最新人类评论，存在误关风险。

3. 404 处理语义模糊
- `githubRequest` 把任意 404 转为空对象，可能掩盖错误路径。
- 建议仅在已知可容忍接口上做 404 容错，其他场景保留失败。

4. 关闭文案与仓库地址硬编码
- `NEW_ISSUE` 固定指向 `anthropics/claude-code`，fork 场景会跳到上游仓库。
- 建议改为使用当前 `owner/repo` 动态拼接。

5. 热度保护粒度有限
- 仅基于 `+1` 数量，不考虑近期评论热度或订阅情况。
- 建议叠加最近活动信号，减少高价值问题被 stale。

6. 并发流程一致性
- triage workflow 和 remove-autoclose workflow 也会并发修改标签。
- 建议增加幂等检查与更细粒度日志，便于排查竞态。

7. 生命周期标签覆盖面
- `closeExpired` 会遍历 `lifecycle` 全量标签（包括 `stale`、`invalid`、`autoclose`）。
- 建议在配置中增加 `closable: boolean` 显式开关，避免策略误配置造成大规模自动关单。
