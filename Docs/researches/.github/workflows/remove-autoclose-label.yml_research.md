# FILE `.github/workflows/remove-autoclose-label.yml` 研究文档

## 场景与职责
该工作流是 lifecycle 自动化的“活跃恢复兜底”：当带 `autoclose` 标签的 open issue 出现新评论时，自动移除 `autoclose`，防止刚恢复讨论就被定时关闭。

## 功能点目的
- 以最小动作体现“有人继续参与=不应自动关单”。
- 与 `sweep.yml` 的自动关闭规则形成反向保护。
- 降低维护者手工移除 `autoclose` 的工作量。

## 具体技术实现（关键流程/数据结构/协议/命令）
- 触发器：`issue_comment.created`。
- 权限：`issues: write`。
- job 级条件：
  - issue 必须 `open`
  - issue 标签中包含 `autoclose`
  - 评论者不是 `github-actions[bot]`
- 执行动作：`actions/github-script@v7`
  - 调用 `github.rest.issues.removeLabel(name='autoclose')`
  - 对 `404` 特判为“已移除/不存在”，视作正常
- 行为特征：无 checkout，执行链路短。

## 关键代码路径与文件引用
- 工作流入口：`.github/workflows/remove-autoclose-label.yml`
- 生命周期策略源：`scripts/issue-lifecycle.ts`（定义 `autoclose` 标签）
- 关联自动关闭器：`.github/workflows/sweep.yml`、`scripts/sweep.ts`
- 关联 triage 流：`.github/workflows/claude-issue-triage.yml`（评论事件也会处理 lifecycle 标签）

## 依赖与外部交互
- 依赖：`actions/github-script@v7`。
- 外部交互：GitHub Issues API（remove label）。
- 凭据：默认 `GITHUB_TOKEN` + `issues:write` 权限。
- 测试现状：无显式自动化测试。

## 风险、边界与改进建议
- 风险 1：只排除了 `github-actions[bot]`，其他 bot 评论也会触发移除，可能带来误恢复。
- 风险 2：和 triage comment 逻辑存在职责重叠，可能出现竞态与重复日志。
- 风险 3：仅关注 `autoclose`，不处理 `stale`，策略分散在多个工作流。
- 边界：仅改标签，不发评论、不关单。
- 建议：
  - 将“有效人类活动”判定统一到单一流程（triage 或该工作流二选一）。
  - 扩展 bot 过滤（`user.type != Bot`）以减少噪声。
  - 增加审计字段（为什么移除、由谁触发）。
