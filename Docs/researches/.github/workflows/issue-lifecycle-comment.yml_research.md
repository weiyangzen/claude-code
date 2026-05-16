# FILE `.github/workflows/issue-lifecycle-comment.yml` 研究文档

## 场景与职责
该工作流在 issue 被打上标签时触发，用于发布生命周期提醒评论（例如 `needs-info`、`stale`、`autoclose` 的倒计时说明）。它是 lifecycle 策略的用户沟通层。

## 功能点目的
- 把“标签即将触发自动关闭”的规则显式通知给 issue 作者。
- 将提醒文案从 workflow 迁移到统一策略源（`scripts/issue-lifecycle.ts`），降低文案分散。
- 通过自动评论减少维护者重复解释成本。

## 具体技术实现（关键流程/数据结构/协议/命令）
- 触发器：`issues: labeled`。
- 权限：`issues: write`。
- 执行流程：
  1. `checkout`
  2. `setup-bun`
  3. `bun run scripts/lifecycle-comment.ts`
- 环境变量注入：
  - `GITHUB_TOKEN`
  - `LABEL`（`github.event.label.name`）
  - `ISSUE_NUMBER`
- 脚本逻辑（`scripts/lifecycle-comment.ts`）：
  - 读取 `scripts/issue-lifecycle.ts` 中 `lifecycle[]`。
  - 若当前标签不在生命周期配置内，直接跳过。
  - 生成 `nudge + timeout` 文案并调用 `POST /issues/{n}/comments`。

## 关键代码路径与文件引用
- 工作流入口：`.github/workflows/issue-lifecycle-comment.yml`
- 执行脚本：`scripts/lifecycle-comment.ts`
- 生命周期策略源：`scripts/issue-lifecycle.ts`
- 上游标签生产者：`.github/workflows/claude-issue-triage.yml`、人工维护操作
- 关联收敛器：`.github/workflows/sweep.yml`

## 依赖与外部交互
- 依赖：Bun runtime、GitHub REST API。
- 凭据：`GITHUB_TOKEN`。
- 外部交互：向 issue timeline 写评论。
- 测试现状：无自动化测试，行为依赖线上标签事件验证。

## 风险、边界与改进建议
- 风险 1：所有 `labeled` 事件都会拉起 job；非生命周期标签会产生额外 runner 开销。
- 风险 2：若 label 快速加删加，可能产生重复提醒评论。
- 边界：仅负责“提醒评论”，不负责关单与移除标签。
- 建议：
  - 在 workflow 级增加标签白名单条件，减少无效触发。
  - 评论增加去重 marker，避免重复提醒。
  - 为 lifecycle-comment 增加 dry-run/单元测试入口，验证文案和标签映射。
