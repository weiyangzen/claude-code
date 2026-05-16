# FILE `.github/workflows/backfill-duplicate-comments.yml` 研究文档

## 场景与职责
该工作流用于“历史 issue 回填 dedupe 评论”。它不直接判断重复，而是批量触发 `claude-dedupe-issues.yml`，让现有 dedupe 流程补齐老 issue 的重复提示。

## 功能点目的
- 补齐旧 issue 中缺失的重复检测评论。
- 通过手动触发方式控制回填节奏，避免常驻任务持续消耗。
- 借助 `dry_run` 先观察候选规模，再决定是否实际触发。

## 具体技术实现（关键流程/数据结构/协议/命令）
- 触发器：仅 `workflow_dispatch`。
- 输入参数：
  - `days_back`（默认 `90`）
  - `dry_run`（`true/false`）
- 权限：`contents: read`、`issues: read`、`actions: write`（需要 dispatch 下游 workflow）。
- 执行流程：
  1. `checkout`
  2. `setup-bun`
  3. `bun run scripts/backfill-duplicate-comments.ts`
- 脚本关键逻辑（`scripts/backfill-duplicate-comments.ts`）：
  - 固定扫描仓库 `anthropics/claude-code`。
  - 按 issue number 区间（`MIN_ISSUE_NUMBER`/`MAX_ISSUE_NUMBER`）分页扫描 issue。
  - 若 issue 评论中不存在 `Found` + `possible duplicate` 的 bot 评论，则视为候选。
  - 候选 issue 触发 `POST /actions/workflows/claude-dedupe-issues.yml/dispatches`，输入 `issue_number`。
  - 默认 `DRY_RUN=true`，非 dry-run 时每次 dispatch 后 sleep 1 秒节流。

## 关键代码路径与文件引用
- 工作流入口：`.github/workflows/backfill-duplicate-comments.yml`
- 执行脚本：`scripts/backfill-duplicate-comments.ts`
- 下游被调用方：`.github/workflows/claude-dedupe-issues.yml`
- dedupe 实际执行链路：`.claude/commands/dedupe.md` -> `scripts/gh.sh` + `scripts/comment-on-duplicates.sh`

## 依赖与外部交互
- GitHub Actions 依赖：Bun、`GITHUB_TOKEN`。
- GitHub API 依赖：
  - 列表/读取 issue 与评论
  - Workflow Dispatch API
- 权限依赖：`actions: write` 是核心权限。
- 文档/配置依赖：依赖 dedupe 评论模板的固定文案作为“已回填”判定条件。

## 风险、边界与改进建议
- 风险 1：`days_back` 输入未被脚本消费，UI 参数与实际行为不一致。
- 风险 2：仓库名硬编码，fork 场景不可复用。
- 风险 3：大规模回填会触发大量 workflow run，仍可能碰到 rate limit。
- 风险 4：仅靠文本匹配判定“已存在 dedupe 评论”，对文案改动敏感。
- 建议：
  - 将输入改为真实生效的 `min_issue_number/max_issue_number` 或真正实现 `days_back`。
  - 支持从 `GITHUB_REPOSITORY` 读取目标仓库。
  - 增加每批次上限与失败重试/退避。
  - 引入 dedupe 评论机器标记，提高判定稳定性。
