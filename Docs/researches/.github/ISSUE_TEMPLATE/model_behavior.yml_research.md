# FILE `.github/ISSUE_TEMPLATE/model_behavior.yml` 研究文档

## 场景与职责

`model_behavior.yml` 针对“模型行为不符合指令或权限预期”的问题单独建模，和传统程序 bug 分离（`.github/ISSUE_TEMPLATE/model_behavior.yml:12-16`）。此模板是仓库治理“安全/行为异常”反馈质量的关键入口，重点采集操作意图与实际执行偏差。

核心职责：

- 统一模型行为问题标识：`[MODEL] ` + `model` 标签（`:3-5`）。
- 强制采集“指令-行为-期望”三联信息与影响面（`:46-85`, `:170-181`）。
- 引导提交者提供权限模式、可复现性、受影响文件、对话片段，为 triage 与后续策略修复提供证据。

## 功能点目的

1. 把行为偏差与运行故障分层
- 模板正文明确“NOT for: crashes/API/install issues”（`.github/ISSUE_TEMPLATE/model_behavior.yml:15`）。
- 目的：把系统错误留给 bug 模板，避免混流。

2. 收集安全相关关键信号
- `behavior_type` 覆盖越权修改、忽略配置、无授权回滚、越界访问等高风险类别（`:28-45`）。
- `permission_mode`、`files_affected` 字段帮助定位是否与权限配置相关（`:86-116`）。

3. 支持可复现与影响评估
- `reproducible`, `reproduction_steps`, `impact`、`version/platform/model` 提供稳定排查维度（`:117-203`）。

4. 控制隐私泄露风险
- preflight 要求确认不含敏感信息（`:23-26`），在行为日志场景下尤其关键。

## 具体技术实现（关键流程/数据结构/协议/命令）

1. 字段结构设计
- 强制字段：`behavior_type`, `what_you_asked`, `what_claude_did`, `expected_behavior`, `permission_mode`, `reproducible`, `model`, `impact`, `version`, `platform`。
- 可选字段：`files_affected`, `reproduction_steps`, `conversation_log`, `additional`。
- `files_affected` 使用 `render: shell`，`conversation_log` 使用 `render: markdown`，分别优化文件列表与对话片段可读性（`:101`, `:166`）。

2. 与 triage 策略的耦合
- triage 指令显式指出“model behavior issues 不应按传统 bug 要求严格复现步骤，示例和模式即可”（`.claude/commands/triage-issue.md:43`）。
- 因此本模板设计强调“实际行为轨迹”和“对话证据”，弱化纯步骤复现依赖。

3. 端到端流程
1) 用户提交模型行为 issue -> 自动加 `model` 标签。
2) `issues.opened` 触发 triage/dedupe/log/dispatch workflows。
3) triage 通过 `gh.sh issue view` 读取正文并打标签，必要时增删 lifecycle 标签（`.claude/commands/triage-issue.md:50-63`）。
4) 后续若长期无活动，`sweep.ts` 仍可基于 lifecycle 标签自动处理（`scripts/sweep.ts:95-149`）。

4. 命令/协议
- 查询操作受 `scripts/gh.sh` 子命令白名单限制（`scripts/gh.sh:29-96`）。
- 标签修改经过 `scripts/edit-issue-labels.sh` 的 label 存在性过滤，防止错误标签污染（`scripts/edit-issue-labels.sh:47-67`）。

## 关键代码路径与文件引用

- 当前模板：`.github/ISSUE_TEMPLATE/model_behavior.yml`
- 对照模板：`.github/ISSUE_TEMPLATE/bug_report.yml`（两者分流边界）
- 模板入口总控：`.github/ISSUE_TEMPLATE/config.yml`
- triage workflow 与指令：`.github/workflows/claude-issue-triage.yml`, `.claude/commands/triage-issue.md`
- dedupe workflow 与指令：`.github/workflows/claude-dedupe-issues.yml`, `.claude/commands/dedupe.md`
- issue/label 命令封装：`scripts/gh.sh`, `scripts/edit-issue-labels.sh`
- 生命周期治理：`scripts/issue-lifecycle.ts`, `scripts/lifecycle-comment.ts`, `scripts/sweep.ts`, `.github/workflows/sweep.yml`
- duplicate 自动关闭链：`scripts/comment-on-duplicates.sh`, `scripts/auto-close-duplicates.ts`

## 依赖与外部交互

1. 平台依赖
- GitHub Issue Forms（下拉、多文本域、必填校验）。

2. 仓库自动化依赖
- GitHub Actions、Claude Code Action、gh CLI、bun。

3. 外部交互
- GitHub REST API 用于 issue、评论、标签、事件读取与写入。
- Statsig issue 事件记录（workflow 层）。

4. 测试现状
- 未见模型行为模板专属的静态校验或字段约束测试（例如敏感字段检查、最小证据检查）。

## 风险、边界与改进建议

1. 风险：行为问题与 bug 的边界仍可能模糊
- 现状：用户常把“模型决策错误”与“CLI 错误输出”混在一起。
- 影响：进入错误模板导致 triage 额外改标签。
- 建议：在模板顶部加入更强的互斥示例，并在 bug 模板内增加“若为行为偏差请改用 MODEL 模板”链接。

2. 风险：敏感信息泄露仍可能发生
- 现状：虽然 preflight 有提醒，但 `conversation_log/files_affected` 可粘贴大量原文。
- 影响：token、内部路径、业务数据泄露风险高于一般 issue。
- 建议：增加独立复选项确认“已脱敏”，并在 placeholder 中给出脱敏示例。

3. 风险：`behavior_type` 枚举会随产品能力演进失配
- 现状：静态枚举无法覆盖新型代理/工具链行为。
- 影响：大量 `Other` 降低统计可用性。
- 建议：结合季度 triage 数据维护枚举，保留向后兼容。

4. 边界：模板不能证明因果
- 提交者描述的是观察到的现象，不等于根因；最终仍需结合会话上下文、版本、环境、策略变更进行工程归因。

5. 建议：增加“是否涉及子代理并行”结构化字段
- 当前仅在 `behavior_type` 中有一项“Subagent behaved unexpectedly”，信息粒度不足。
- 可新增字段记录子代理数量、任务类型、是否并行，以便后续分析行为偏差触发条件。
