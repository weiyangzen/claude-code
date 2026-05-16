# FILE `.github/workflows/lock-closed-issues.yml` 研究文档

## 场景与职责
该工作流定期锁定“关闭后长期无活动”的 issue，减少被重新顶起的历史线程，促使用户新开 issue 描述当前问题。它是生命周期的终态清理器。

## 功能点目的
- 对关闭 7 天以上且未锁定的 issue 自动加锁。
- 在加锁前发布说明评论，给用户明确后续处理路径。
- 防止旧 issue 被反复复活造成维护噪声。

## 具体技术实现（关键流程/数据结构/协议/命令）
- 触发器：
  - `schedule: 0 14 * * *`
  - `workflow_dispatch`
- 权限：`issues: write`。
- 并发：`concurrency.group=lock-threads`，防止重入。
- 实现方式：`actions/github-script@v7` 内联 JS。
- 关键算法：
  - 计算 `sevenDaysAgo`。
  - 分页读取 closed issues（`sort=updated,direction=asc,per_page=100`）。
  - 跳过已锁定与 PR。
  - 若遇到 `updated_at > sevenDaysAgo`，利用升序特性提前终止后续扫描。
  - 对候选先 `createComment`，再 `issues.lock(lock_reason=resolved)`。
  - 统计 `totalLocked` 并输出日志。

## 关键代码路径与文件引用
- 工作流入口：`.github/workflows/lock-closed-issues.yml`
- 执行动作：`actions/github-script@v7`（内联脚本）
- 上游关联：
  - `.github/workflows/sweep.yml`（自动关闭 issue）
  - `.github/workflows/auto-close-duplicates.yml`（duplicate 关闭）
- 下游效果：issue timeline 评论 + lock 状态变更

## 依赖与外部交互
- 依赖：GitHub Actions runtime + Octokit（由 github-script 注入）。
- 外部交互：GitHub Issues API（list/createComment/lock）。
- 凭据：默认 `GITHUB_TOKEN`（workflow 权限范围内）。
- 测试现状：无独立测试，逻辑通过线上定时运行验证。

## 风险、边界与改进建议
- 风险 1：评论后若 lock 失败，下次运行可能再次评论，出现重复提示。
- 风险 2：固定 7 天策略不可配置，策略调整需改代码。
- 风险 3：每次从最旧关闭 issue 扫描，超大仓库下耗时和 API 压力上升。
- 边界：仅处理 closed issue，不处理 open/stale 状态。
- 建议：
  - 对“已发过锁定提示评论”的 issue 做幂等标记。
  - 将天数与 comment 文案参数化。
  - 增加 API 失败重试与速率限制感知。
