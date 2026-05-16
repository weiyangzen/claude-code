# FILE `.github/workflows/sweep.yml` 研究文档

## 场景与职责
该工作流是 issue 生命周期治理主流程，负责定期执行“打 stale 标签 + 按超时自动关单”。它连接 `issue-lifecycle.ts` 策略与实际 issue 操作。

## 功能点目的
- 自动化维护 issue 池健康度，减少长期无进展工单。
- 用统一生命周期策略（label/day/reason/nudge）驱动关单行为。
- 通过高赞阈值保护社区关注度高的问题。

## 具体技术实现（关键流程/数据结构/协议/命令）
- 触发器：
  - `schedule: 0 10,22 * * *`
  - `workflow_dispatch`
- 权限：`issues: write`。
- 并发：`concurrency.group=daily-issue-sweep`，防止并发重入。
- 执行流程：
  1. `checkout`
  2. `setup-bun`
  3. `bun run scripts/sweep.ts`
- 脚本关键实现（`scripts/sweep.ts`）：
  - 读取 `scripts/issue-lifecycle.ts` 中 `lifecycle[]` 与 `STALE_UPVOTE_THRESHOLD=10`。
  - `markStale`：扫描 open issues，跳过 PR/locked/有 assignee/已 stale/autoclose/高赞 issue，给其余 issue 打 `stale`。
  - `closeExpired`：按每个 lifecycle label 拉取 issue，读取 events 找最近 labeled 时间；若超过阈值且之后无 human 评论，则评论并关闭（`state_reason=not_planned`）。
- 环境变量：`GITHUB_TOKEN`、`GITHUB_REPOSITORY_OWNER`、`GITHUB_REPOSITORY_NAME`。

## 关键代码路径与文件引用
- 工作流入口：`.github/workflows/sweep.yml`
- 执行脚本：`scripts/sweep.ts`
- 策略源：`scripts/issue-lifecycle.ts`
- 关联提醒流：`.github/workflows/issue-lifecycle-comment.yml`
- 关联恢复流：`.github/workflows/remove-autoclose-label.yml`
- 关联终态流：`.github/workflows/lock-closed-issues.yml`

## 依赖与外部交互
- 依赖：Bun、GitHub REST API。
- 凭据：`GITHUB_TOKEN`。
- 外部交互：issue 列表、事件、评论、标签、状态更新。
- 测试现状：无专门自动化测试，主要靠 dry-run 能力（脚本支持）与线上观察。

## 风险、边界与改进建议
- 风险 1：分页上限固定（每段最多 10 页），仓库规模扩大后可能漏处理。
- 风险 2：workflow 未暴露 dry-run 开关，线上调试需改命令或本地执行。
- 风险 3：与 triage/remove-autoclose 的并发标签变更存在竞争窗口。
- 风险 4：关闭判断依赖 issue events/comments 时序，极端并发下可能出现边界误判。
- 建议：
  - 增加 workflow_dispatch 输入 `dry_run`。
  - 将扫描页数、阈值、批处理上限配置化。
  - 为关键决策输出结构化日志（label 时间、评论时间、跳过原因）。
