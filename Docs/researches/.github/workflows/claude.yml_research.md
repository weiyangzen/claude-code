# FILE `.github/workflows/claude.yml` 研究文档

## 场景与职责
该工作流是仓库内通用 `@claude` 入口。任何 issue/PR 相关事件中出现 `@claude` 提及时，会触发 Claude Code Action 执行。

## 功能点目的
- 提供统一的人机协作入口，支持在 GitHub 线程内直接调用 Claude。
- 覆盖多事件源（issue 评论、review 评论、issue 标题/正文、review 内容），降低使用门槛。
- 以最小权限（读权限为主）运行，减少默认写操作面。

## 具体技术实现（关键流程/数据结构/协议/命令）
- 触发器：
  - `issue_comment.created`
  - `pull_request_review_comment.created`
  - `issues.opened|assigned`
  - `pull_request_review.submitted`
- 运行条件（job `if`）：
  - 对不同事件分别检查 body/title 中是否包含 `@claude`。
- 权限：`contents: read`、`pull-requests: read`、`issues: read`、`id-token: write`。
- 步骤：
  1. `actions/checkout`（使用 commit SHA 固定到 v4）
  2. `anthropics/claude-code-action@v1`
     - `anthropic_api_key`
     - `claude_args: --model claude-sonnet-4-5-20250929`

## 关键代码路径与文件引用
- 工作流入口：`.github/workflows/claude.yml`
- 文档线索：`README.md`（说明支持在 GitHub 中 `@claude`）
- 关联安全检查：`.github/workflows/non-write-users-check.yml`（审计 `allowed_non_write_users` 变更）
- 同目录其他专用流：
  - `.github/workflows/claude-issue-triage.yml`
  - `.github/workflows/claude-dedupe-issues.yml`

## 依赖与外部交互
- 外部 Action：`anthropics/claude-code-action@v1`。
- 凭据：`ANTHROPIC_API_KEY`。
- 平台依赖：GitHub 事件上下文（评论、review、issue）。
- 运行环境：`ubuntu-latest`。

## 风险、边界与改进建议
- 风险 1：事件覆盖面广，误触发可能增加模型调用成本。
- 风险 2：该工作流未显式限制非写权限用户触发策略，实际行为取决于 action 默认策略。
- 风险 3：模型版本硬编码，升级需改工作流。
- 边界：本流是“mention 入口”，不承担 triage/dedupe 的特定治理语义。
- 建议：
  - 增加最小触发阈值（如排除代码块内 `@claude` 噪声）。
  - 对调用结果增加统一审计标签或日志字段。
  - 评估是否也对 action 版本做 commit pin，进一步降低供应链漂移。
