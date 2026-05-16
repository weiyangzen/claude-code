# FILE `.github/workflows/auto-close-duplicates.yml` 研究文档

## 场景与职责
该工作流是重复问题治理链路的“延时收敛器”：在 dedupe 机器人先发表评论后，等待观察窗口，再自动关闭满足条件的重复 issue。
它通过定时任务（每天 UTC 09:00）与手动触发执行，属于低频后台治理任务。

## 功能点目的
- 自动减少重复 issue 的长期堆积，降低维护者重复沟通成本。
- 给用户保留 3 天申诉窗口（评论或点踩 `-1`）后再关单，避免即时误关。
- 将重复关单动作标准化为脚本逻辑，而不是人工逐条处理。

## 具体技术实现（关键流程/数据结构/协议/命令）
- 触发器：
  - `schedule: 0 9 * * *`
  - `workflow_dispatch`
- 权限：`contents: read` + `issues: write`，仅允许读仓库并写 issue。
- 执行流程：
  1. `actions/checkout@v4`
  2. `oven-sh/setup-bun@v2`（`bun-version: latest`）
  3. `bun run scripts/auto-close-duplicates.ts`
- 关键脚本逻辑（`scripts/auto-close-duplicates.ts`）：
  - 分页扫描 open issues（最多 20 页，每页 100）。
  - 仅处理创建超过 3 天的 issue。
  - 在评论里寻找 `Found` + `possible duplicate` + `user.type === Bot` 的重复提示评论。
  - 要求最近重复提示评论也超过 3 天，且之后无新增评论。
  - 若 issue 作者未对提示评论点 `-1`，则 `PATCH issue` 为 `state=closed,state_reason=duplicate` 并补自动关闭评论。
- 环境变量：
  - `GITHUB_TOKEN`
  - `GITHUB_REPOSITORY_OWNER`
  - `GITHUB_REPOSITORY_NAME`
  - `STATSIG_API_KEY`（当前脚本未使用）

## 关键代码路径与文件引用
- 工作流入口：`.github/workflows/auto-close-duplicates.yml`
- 执行脚本：`scripts/auto-close-duplicates.ts`
- 上游来源（重复评论生产者）：
  - `.github/workflows/claude-dedupe-issues.yml`
  - `.claude/commands/dedupe.md`
  - `scripts/comment-on-duplicates.sh`
- 同域治理协作：
  - `.github/workflows/backfill-duplicate-comments.yml`（补历史 dedupe 评论）

## 依赖与外部交互
- 运行依赖：GitHub Actions `ubuntu-latest`、Bun。
- 平台依赖：GitHub Issues REST API（读 issue/comments/reactions、写 issue/comments）。
- 凭据依赖：`secrets.GITHUB_TOKEN`。
- 配置依赖：评论文本协议与 dedupe 评论模板保持一致。
- 测试现状：仓库内未见该 workflow 的自动化测试；主要依赖线上日志观察。

## 风险、边界与改进建议
- 风险 1：文本协议脆弱。脚本靠字符串 `Found`/`possible duplicate` 判断，改文案会导致失效。
- 风险 2：重复目标提取过宽。`/#(\d+)/` 可能提取到非目标编号。
- 风险 3：分页上限固定（20 页），超大仓库可能漏扫。
- 风险 4：`STATSIG_API_KEY` 在该 workflow 中冗余，存在配置漂移。
- 建议：
  - 在重复评论里加机器标记（如 HTML 注释）替代纯文本匹配。
  - 用结构化链接解析替代正则首命中。
  - 将扫描页数与时间窗口参数化。
  - 清理未使用环境变量，避免误导运维。
