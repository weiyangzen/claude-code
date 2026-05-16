# FILE `.github/workflows/non-write-users-check.yml` 研究文档

## 场景与职责
该工作流是 `.github/**` 变更的安全审计守卫：当 PR 修改 workflow 并新增/修改 `allowed_non_write_users` 时，自动发布风险提示评论。

## 功能点目的
- 提醒审阅者关注 Claude Action 的触发权限扩大风险。
- 对高风险配置变更建立统一提示模板，降低漏审概率。
- 避免同一 PR 重复刷屏（通过 marker 注释去重）。

## 具体技术实现（关键流程/数据结构/协议/命令）
- 触发器：`pull_request`，路径过滤 `.github/**`。
- 权限：`contents: read`、`pull-requests: write`。
- 执行流程（纯 shell + gh）：
  1. `gh pr diff` 获取 PR diff。
  2. 用 `grep '^diff --git a/\.github/.*\.ya?ml'` 判断是否改了 workflow yaml。
  3. 用 `grep '^+.*allowed_non_write_users'` 查新增行。
  4. `gh pr view --json comments` 检查是否已有 `<!-- non-write-users-check -->`。
  5. 无历史提示时 `gh pr comment` 发布固定警示文案。
- 幂等机制：HTML 注释 marker 防重复评论。

## 关键代码路径与文件引用
- 工作流入口：`.github/workflows/non-write-users-check.yml`
- 重点被审对象：`.github/workflows/claude-issue-triage.yml`、`.github/workflows/claude-dedupe-issues.yml` 等 Claude Action 工作流
- 关联策略文档：`.claude/commands/*`（决定触发后可执行工具边界）

## 依赖与外部交互
- 依赖：GitHub CLI `gh`、`grep`。
- 凭据：`GITHUB_TOKEN`（通过 `GH_TOKEN` 注入）。
- 外部交互：GitHub Pull Request API（diff/view/comment）。
- 测试现状：无自动化测试，依赖 PR 真实触发验证。

## 风险、边界与改进建议
- 风险 1：当前为“提醒型”控制，不会阻断合并。
- 风险 2：仅匹配新增行，复杂改写（如多行 YAML 结构）可能漏报。
- 风险 3：路径过滤是 `.github/**`，会在非 workflow 文件变更时也启动 job。
- 边界：仅检测 `allowed_non_write_users` 字段，不评估 prompt/tool 权限是否扩大。
- 建议：
  - 升级为可配置阻断检查（required status + override 机制）。
  - 改为 YAML AST 解析，减少 grep 误判/漏判。
  - 将扫描范围收窄到 `.github/workflows/**/*.yml`。
