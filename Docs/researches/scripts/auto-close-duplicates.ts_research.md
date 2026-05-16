# FILE `scripts/auto-close-duplicates.ts` 研究文档

## 场景与职责

`scripts/auto-close-duplicates.ts` 是重复 issue 治理链路的“最终执行器”。

它不负责识别重复（识别由 `/dedupe` 流程和 `scripts/comment-on-duplicates.sh` 完成），而是根据已有“可能重复”评论、静默期和作者反馈状态，自动把 issue 关闭为 `duplicate`。

典型触发入口是 GitHub Actions 定时任务：`.github/workflows/auto-close-duplicates.yml` 每天执行一次 `bun run scripts/auto-close-duplicates.ts`。

## 功能点目的

1. 对 open issue 进行批量巡检，只关注创建时间已超过 3 天的问题。
2. 识别是否存在 dedupe 机器人评论（文本包含 `Found` + `possible duplicate`，且评论作者类型为 `Bot`）。
3. 确认最近一条重复提示评论距今超过 3 天，且之后无人继续评论。
4. 检查 issue 作者是否对该评论给出 `-1` 反对 reaction；若反对则不自动关闭。
5. 满足条件时，调用 GitHub API 自动关闭 issue，并追加解释性评论。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 数据结构与 HTTP 封装

- 定义了 `GitHubIssue` / `GitHubComment` / `GitHubReaction` 三个接口，用于约束核心字段（`number`、`created_at`、`user.id`、`content` 等）。
- `githubRequest<T>()` 统一封装 `fetch`：
  - Base URL 固定 `https://api.github.com`。
  - Header 含 `Authorization: Bearer <token>`、`Accept: application/vnd.github.v3+json`、`User-Agent: auto-close-duplicates-script`。
  - 非 2xx 直接抛异常，调用方按 issue 粒度捕获。

### 2) 重复目标 issue 编号提取协议

`extractDuplicateIssueNumber(commentBody)` 采用两段匹配：

1. 先匹配 `#123`（`/#(\d+)/`）。
2. 未命中再匹配 GitHub issue URL（`github.com/<owner>/<repo>/issues/123`）。

这是脚本把“评论文案”映射成“关闭目标编号”的关键协议，依赖 dedupe 评论格式稳定。

### 3) 自动关闭主流程

`autoCloseDuplicates()` 的核心步骤：

1. 读取 `GITHUB_TOKEN`；仓库坐标来自 `GITHUB_REPOSITORY_OWNER`、`GITHUB_REPOSITORY_NAME`，缺省回退为 `anthropics/claude-code`。
2. 计算 `threeDaysAgo = now - 3 days`。
3. 分页拉取 open issues（`per_page=100`，最多 20 页），仅保留 `created_at <= threeDaysAgo`。
4. 对每个候选 issue：
   - 拉取 issue comments。
   - 筛选 dedupe bot 评论（文本规则 + `user.type === "Bot"`）。
   - 取最后一条 dedupe 评论；若不足 3 天则跳过。
   - 若 dedupe 评论后有任何新评论，跳过（视为仍在讨论）。
   - 拉取该评论 reactions，若 issue 作者给了 `-1`，跳过。
   - 提取重复目标编号成功后进入关闭动作。
5. 关闭动作 `closeIssueAsDuplicate()`：
   - `PATCH /issues/{n}`：`state=closed`、`state_reason=duplicate`、`labels=['duplicate']`。
   - `POST /issues/{n}/comments`：写入自动关闭说明和复议提示。

### 4) 触发与运行协议

- 运行方式：`bun run scripts/auto-close-duplicates.ts`。
- 调用方：`.github/workflows/auto-close-duplicates.yml`（定时 + 手动触发）。
- 权限需求：workflow 声明 `issues: write`，脚本需 `GITHUB_TOKEN`。

## 关键代码路径与文件引用

- 主实现：`scripts/auto-close-duplicates.ts`
- 重复评论生成上游：`scripts/comment-on-duplicates.sh`
- 自动执行入口：`.github/workflows/auto-close-duplicates.yml`
- dedupe 触发链路（提供上游评论）：
  - `.github/workflows/claude-dedupe-issues.yml`
  - `.claude/commands/dedupe.md`

重点代码段：

- `githubRequest()`：统一 API 调用与错误处理。
- `extractDuplicateIssueNumber()`：把评论文本映射为重复 issue 编号。
- `autoCloseDuplicates()`：批量扫描、条件筛选、审慎关闭。
- `closeIssueAsDuplicate()`：状态变更 + 解释评论。

## 依赖与外部交互

1. 运行依赖
- Bun 运行时（脚本 shebang `#!/usr/bin/env bun`）。
- GitHub Actions Runner（计划任务场景）。

2. 环境变量
- 必需：`GITHUB_TOKEN`。
- 可选：`GITHUB_REPOSITORY_OWNER`、`GITHUB_REPOSITORY_NAME`（缺省回退硬编码仓库）。

3. 外部 API
- `GET /repos/{owner}/{repo}/issues`
- `GET /repos/{owner}/{repo}/issues/{issue_number}/comments`
- `GET /repos/{owner}/{repo}/issues/comments/{comment_id}/reactions`
- `PATCH /repos/{owner}/{repo}/issues/{issue_number}`
- `POST /repos/{owner}/{repo}/issues/{issue_number}/comments`

4. 与其他自动化协作
- 与 `scripts/comment-on-duplicates.sh` 的评论文案存在隐式协议耦合。
- 与 `claude-dedupe-issues.yml` 联动：该 workflow 负责先写“possible duplicate”评论。

## 风险、边界与改进建议

1. 文本协议脆弱
- 当前依赖 `Found` + `possible duplicate` 文本片段识别 dedupe 评论，文案一旦变更会漏判。
- 建议在评论中增加机器可读 marker（例如 `<!-- dedupe:v1 -->`）。

2. 重复编号提取可能误匹配
- `/#(\d+)/` 会优先抓到任意 `#数字`，不一定是候选重复 issue。
- 建议优先解析标准 URL 列表或限定从“候选列表区块”提取。

3. 标签覆写风险
- `PATCH issue` 时传 `labels: ['duplicate']` 可能覆盖原标签集合。
- 建议改为单独调用 label API 增量加 `duplicate`，避免清空上下文标签。

4. PR 混入边界
- `/issues` 列表会包含 PR；脚本未显式跳过 `pull_request`。
- 建议在 issue 结构中读取 `pull_request` 字段并显式 `continue`。

5. 扫描覆盖上限
- 分页上限 20 页（最多 2000 条 open issue），超大仓库会漏扫。
- 建议用 Link Header 分页或可配置页上限。

6. 反对信号识别偏窄
- 仅识别作者 `-1` reaction，不识别“作者留言反对”。
- 建议同时解析 dedupe 评论后的作者文本反馈并视为阻断。

7. 观测性不足
- 工作流传入了 `STATSIG_API_KEY`，脚本未实际使用。
- 建议清理无效环境变量或补充事件上报，避免配置漂移。
