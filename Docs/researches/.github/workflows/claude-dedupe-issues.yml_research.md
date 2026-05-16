# FILE `.github/workflows/claude-dedupe-issues.yml` 研究文档

## 场景与职责
该工作流是“重复问题识别”的核心入口：在 issue 新开时自动运行 dedupe，也支持手动指定 issue 触发。它调用 Claude Code Action 执行 `/dedupe`，并在末尾上报 Statsig 事件。

## 功能点目的
- 新 issue 到达后尽早给出重复候选，减少并行讨论线程。
- 提供 `workflow_dispatch` 以支撑历史回填和人工重跑。
- 上报 dedupe 执行事件，为运营观测提供数据。

## 具体技术实现（关键流程/数据结构/协议/命令）
- 触发器：
  - `issues: opened`
  - `workflow_dispatch`（输入 `issue_number`）
- 权限：`contents: read`、`issues: write`。
- 步骤 1（核心）：`anthropics/claude-code-action@v1`
  - `prompt: /dedupe <repo>/issues/<number>`
  - `allowed_non_write_users: "*"`
  - `claude_args: --model claude-sonnet-4-5-20250929`
  - `GH_TOKEN` 与 `github_token` 均使用 `secrets.GITHUB_TOKEN`
- dedupe 命令约束（`.claude/commands/dedupe.md`）：
  - 允许工具仅 `scripts/gh.sh` 与 `scripts/comment-on-duplicates.sh`
  - 先读 issue，再并行搜索重复，再发布最多 3 条候选评论
- 步骤 2（观测）：`if: always()` 的 Statsig 上报 shell
  - `jq -n` 组装 payload，事件名固定 `github_duplicate_comment_added`
  - `curl POST https://events.statsigapi.net/v1/log_event`
  - `STATSIG_API_KEY` 缺失时直接跳过，不阻塞 workflow

## 关键代码路径与文件引用
- 工作流入口：`.github/workflows/claude-dedupe-issues.yml`
- Claude 命令定义：`.claude/commands/dedupe.md`
- 工具脚本：`scripts/gh.sh`、`scripts/comment-on-duplicates.sh`
- 回填调用方：`.github/workflows/backfill-duplicate-comments.yml`、`scripts/backfill-duplicate-comments.ts`
- 下游收敛器：`.github/workflows/auto-close-duplicates.yml`、`scripts/auto-close-duplicates.ts`

## 依赖与外部交互
- 第三方 Action：`anthropics/claude-code-action@v1`。
- 平台交互：GitHub Issues API（由 gh/脚本触发）、Statsig Events API。
- 凭据：`ANTHROPIC_API_KEY`、`GITHUB_TOKEN`、`STATSIG_API_KEY`。
- Runner 依赖：`ubuntu-latest` 自带 `jq/curl`（该步骤依赖这两者）。

## 风险、边界与改进建议
- 风险 1：`allowed_non_write_users: "*"` 提高了触发面，需持续配合安全审计。
- 风险 2：Statsig 事件在 `always()` 下执行，可能记录“执行了 dedupe 工作流”而非“确实添加评论”。
- 风险 3：事件名固定为 `github_duplicate_comment_added`，但未校验评论是否实际写入。
- 风险 4：模型版本硬编码，升级需要改 workflow。
- 建议：
  - 让 dedupe 脚本输出结构化结果（是否已评论/候选数），Statsig 基于结果上报。
  - 对 `allowed_non_write_users` 变更配合强制审查门禁。
  - 增加失败分类（模型失败、GitHub API 失败、无重复候选）。
