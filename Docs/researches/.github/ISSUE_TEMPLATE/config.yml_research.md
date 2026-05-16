# FILE `.github/ISSUE_TEMPLATE/config.yml` 研究文档

## 场景与职责

`config.yml` 是 GitHub Issue 模板系统的全局配置入口，职责是控制 issue 创建入口策略，而不是定义具体业务字段。当前文件非常短，但它直接决定了仓库反馈通道的流量分配：

- 禁用空白 issue（`blank_issues_enabled: false`，`.github/ISSUE_TEMPLATE/config.yml:1`）。
- 通过 `contact_links` 提供 Discord、文档、Quickstart、Troubleshooting 四类外部入口（`:2-14`）。

这会影响 `.github/ISSUE_TEMPLATE/*.yml` 的采用率与下游 triage 负载。

## 功能点目的

1. 强制结构化输入
- 禁用 blank issue，迫使用户在 bug/docs/feature/model 预设表单中提交。
- 目的：避免自由格式文本导致 triage 误判和补信息回合增加。

2. 非 issue 场景分流
- `contact_links` 把“求助/学习/常见故障排查”提前导向官方渠道。
- 目的：减少将支持咨询误投为仓库缺陷工单。

3. 降低自动化系统噪音
- 结构化表单比例上升后，`claude-issue-triage.yml` 与 dedupe 流程的输入更稳定（`.github/workflows/claude-issue-triage.yml:1-38`，`.github/workflows/claude-dedupe-issues.yml:1-35`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

1. 数据结构
- 顶层键仅两类：
- `blank_issues_enabled: false`
- `contact_links[]`（每项含 `name/url/about`）

2. 平台协议行为
- GitHub 在 `issues/new/choose` 页面读取该配置。
- 当 `blank_issues_enabled=false` 时，不显示“Open a blank issue”。
- `contact_links` 在模板列表旁以跳转卡片展示，不会创建 issue 实体。

3. 与仓库自动化流程的间接耦合
1) 用户被引导到模板或外部文档。
2) 模板 issue 创建后触发 `issues.opened` workflows：
- `claude-issue-triage.yml`
- `claude-dedupe-issues.yml`
- `issue-opened-dispatch.yml`
- `log-issue-events.yml`
3) 后续生命周期脚本继续处理标签和状态（`scripts/sweep.ts`, `scripts/lifecycle-comment.ts`）。

4. 命令层关系
- `config.yml` 不直接调用脚本，但它通过“入口治理”影响 `.claude/commands/triage-issue.md` 对 issue 正文和标签的可用信息密度。

## 关键代码路径与文件引用

- 全局入口配置：`.github/ISSUE_TEMPLATE/config.yml`
- 同目录具体模板：`.github/ISSUE_TEMPLATE/bug_report.yml`, `documentation.yml`, `feature_request.yml`, `model_behavior.yml`
- 新 issue 自动化消费链：`.github/workflows/claude-issue-triage.yml`, `.github/workflows/claude-dedupe-issues.yml`, `.github/workflows/issue-opened-dispatch.yml`, `.github/workflows/log-issue-events.yml`
- triage 标签执行：`.claude/commands/triage-issue.md`, `scripts/gh.sh`, `scripts/edit-issue-labels.sh`
- 生命周期关闭链：`.github/workflows/sweep.yml`, `scripts/issue-lifecycle.ts`, `scripts/sweep.ts`
- 用户侧入口文档：`README.md:52-59`

## 依赖与外部交互

1. 平台依赖
- GitHub Issue Templates/Forms 前端渲染。

2. 外部链接依赖
- `https://anthropic.com/discord`
- `https://docs.claude.com/en/docs/claude-code`
- `https://docs.claude.com/en/docs/claude-code/quickstart`
- `https://docs.claude.com/en/docs/claude-code/troubleshooting`

3. 与 Actions 的关系
- 本文件不需要 token，不直接触发命令；它通过减少不规范 issue 输入，降低 Actions 与 triage agent 的噪声成本。

4. 测试现状
- 仓库内未见针对 `contact_links` 可达性和配置正确性的自动化检查。

## 风险、边界与改进建议

1. 风险：外链老化
- 现状：链接硬编码在 YAML。
- 影响：站点迁移后会出现失效入口。
- 建议：增加定时 link-check（CI 或 cron）并在失效时自动开 issue。

2. 风险：过强入口约束导致“无模板匹配”场景摩擦
- 现状：blank issue 完全禁用。
- 影响：极端/复合问题可能难以归类，用户体验下降。
- 建议：在四个模板中都提供明确“不适配时选择路径”文案，或新增 `question/support` 模板并标记不进入工程 backlog。

3. 风险：分流渠道与仓库实际支持边界不一致
- 现状：`contact_links` 主要是帮助类入口。
- 影响：若 triage 政策更新但分流文案未同步，仍会引入错投。
- 建议：将 triage policy（例如 invalid 判定）和 contact_links 文案进行季度同步审查。

4. 边界：`config.yml` 只能治理入口，不负责语义质量
- 它能决定“从哪里进”，但不能保证“填得好”。字段质量仍由各具体模板和后续 triage 逻辑承担。
