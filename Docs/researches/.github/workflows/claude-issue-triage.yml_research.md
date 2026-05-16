# FILE `.github/workflows/claude-issue-triage.yml` 研究文档

## 场景与职责
该工作流负责 issue 自动分拣与生命周期标签治理。它在新 issue 与评论事件触发 Claude `/triage-issue`，只进行标签增删，不做自动评论。

## 功能点目的
- 新 issue 创建时快速补齐分类标签（bug/enhancement/平台标签等）。
- 在评论事件中动态移除或调整 lifecycle 标签（如 `stale/autoclose/needs-info`）。
- 通过同 issue 并发互斥，避免短时间多次触发造成标签抖动。

## 具体技术实现（关键流程/数据结构/协议/命令）
- 触发器：
  - `issues: opened`
  - `issue_comment: created`
- 过滤条件（job `if`）：
  - `issues` 事件总是执行。
  - `issue_comment` 仅处理非 PR 线程、且评论者非 Bot。
- 并发控制：
  - `group: issue-triage-${{ github.event.issue.number }}`
  - `cancel-in-progress: true`
- 核心动作：`anthropics/claude-code-action@v1`
  - prompt：`/triage-issue REPO: ... ISSUE_NUMBER: ... EVENT: ...`
  - `allowed_non_write_users: "*"`
  - 模型参数：`--model claude-opus-4-6`
- triage 规则来源（`.claude/commands/triage-issue.md`）：
  - 只允许 `scripts/gh.sh` 读取与 `scripts/edit-issue-labels.sh` 写标签。
  - 明确禁止发评论。
  - lifecycle 标签仅在证据充分时处理。

## 关键代码路径与文件引用
- 工作流入口：`.github/workflows/claude-issue-triage.yml`
- 命令定义：`.claude/commands/triage-issue.md`
- 工具脚本：`scripts/gh.sh`、`scripts/edit-issue-labels.sh`
- 关联策略：`scripts/issue-lifecycle.ts`（标签语义与超时策略）
- 同域协同：
  - `.github/workflows/remove-autoclose-label.yml`
  - `.github/workflows/sweep.yml`

## 依赖与外部交互
- 依赖 Action：`actions/checkout@v4`、`anthropics/claude-code-action@v1`。
- 平台依赖：GitHub Issues/Labels API（通过 gh CLI 调用）。
- 凭据依赖：`GITHUB_TOKEN`、`ANTHROPIC_API_KEY`。
- 权限：`issues: write` 允许标签变更。

## 风险、边界与改进建议
- 风险 1：评论触发频率高，可能导致重复 triage 成本。
- 风险 2：`allowed_non_write_users: "*"` 需和仓库安全策略一致。
- 风险 3：triage 与 `remove-autoclose-label` 可能同时改标签，存在竞态。
- 边界：该流只做标签，不做评论，不处理 PR 讨论。
- 建议：
  - 对 `issue_comment` 事件增加“实质内容”过滤（例如排除仅 emoji/`+1`）。
  - 为 triage 输出结构化日志（加了哪些标签/删了哪些标签）。
  - 与 `remove-autoclose-label` 做职责收敛，减少重复逻辑。
