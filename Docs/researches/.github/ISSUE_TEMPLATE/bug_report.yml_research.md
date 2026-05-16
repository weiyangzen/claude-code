# FILE `.github/ISSUE_TEMPLATE/bug_report.yml` 研究文档

## 场景与职责

`bug_report.yml` 是仓库在 GitHub Issue 创建入口的缺陷采集模板，面向 Claude Code 运行异常、错误日志、回归问题等场景。该文件本身不执行逻辑，但它决定了新建 issue 的：

- 默认分类信号：标题前缀 `[BUG] `、默认标签 `bug`（`.github/ISSUE_TEMPLATE/bug_report.yml:3-5`）。
- 最低可诊断信息：实际行为、期望行为、复现步骤、版本与环境字段（`:30-175`）。
- 下游治理触发质量：triage 是否会追加 `needs-repro` / `needs-info`，以及重复检测与生命周期策略是否能有效工作（`.claude/commands/triage-issue.md:40-49`，`scripts/sweep.ts:95-149`）。

## 功能点目的

1. 入口分流与防重复
- 通过顶部 markdown 与 preflight 复选项要求用户先查重、确认使用最新版本、单问题单提单（`.github/ISSUE_TEMPLATE/bug_report.yml:7-29`）。
- 目的：减少重复 issue 和“多问题混单”。

2. 形成可调试的缺陷画像
- 强制采集 `actual`、`expected`、`reproduction`、`regression`、`version`、`platform`、`os`、`terminal`（`:30-175`）。
- 目的：让 triage 与工程排障第一轮就具备关键上下文。

3. 服务生命周期标签判断
- triage 指令中明确将 bug 类 issue 作为 `needs-repro` / `needs-info` 的主要适用对象（`.claude/commands/triage-issue.md:41-46,68`）。
- 本模板字段就是减少这两类标签误用或漏用的主要输入基础。

## 具体技术实现（关键流程/数据结构/协议/命令）

1. Issue Form 结构实现
- 顶层键：`name/description/title/labels/body`（`.github/ISSUE_TEMPLATE/bug_report.yml:1-6`）。
- `body[]` 中混合 `markdown/checkboxes/textarea/dropdown/input` 五种控件。
- 每个字段依赖 `id`（例如 `actual`, `reproduction`, `version`）和 `validations.required` 实现最小必填约束。

2. 关键字段设计
- `error_output` 使用 `render: shell`，提升日志可读性（`:51-61`）。
- `model/platform/os/terminal` 采用下拉约束，减少自由文本歧义（`:85-175`）。
- `regression` 为必填，但 `working_version` 可选（`:98-117`），表示“回归判断”和“精确回归版本”被解耦。

3. 端到端流程
1) 用户在 `issues/new/choose` 选择 Bug Report，提交后 GitHub 自动带上 `bug` 标签。
2) `issues.opened` 同时触发：
- `claude-issue-triage.yml` -> `/triage-issue ...` -> `scripts/gh.sh` + `scripts/edit-issue-labels.sh`（`.github/workflows/claude-issue-triage.yml:1-38`，`.claude/commands/triage-issue.md:27-52`）。
- `claude-dedupe-issues.yml` -> `/dedupe ...`（`.github/workflows/claude-dedupe-issues.yml:1-35`）。
3) 后续 `sweep.ts` 按生命周期规则做 stale/close（`scripts/issue-lifecycle.ts:3-34`，`scripts/sweep.ts:95-149`）。

4. 协议/命令收敛
- triage/dedupe 命令被限制在 `gh.sh` 白名单子命令中（`scripts/gh.sh:23-96`）。
- 标签变更必须经过 `edit-issue-labels.sh` 的 label 存在性过滤（`scripts/edit-issue-labels.sh:47-80`）。

## 关键代码路径与文件引用

- 模板定义：`.github/ISSUE_TEMPLATE/bug_report.yml`
- 模板分发总控：`.github/ISSUE_TEMPLATE/config.yml`
- 新 issue triage：`.github/workflows/claude-issue-triage.yml` -> `.claude/commands/triage-issue.md`
- 标签写入：`scripts/edit-issue-labels.sh`
- 只读查询封装：`scripts/gh.sh`
- 重复检测链路：`.github/workflows/claude-dedupe-issues.yml` -> `.claude/commands/dedupe.md` -> `scripts/comment-on-duplicates.sh`
- 生命周期与自动关闭：`scripts/issue-lifecycle.ts`, `scripts/lifecycle-comment.ts`, `scripts/sweep.ts`, `.github/workflows/sweep.yml`
- 自动关闭重复 issue：`scripts/auto-close-duplicates.ts`, `.github/workflows/auto-close-duplicates.yml`
- 用户入口文档：`README.md:52-55`

## 依赖与外部交互

1. 平台依赖
- GitHub Issue Forms 渲染和必填校验。
- GitHub Actions 事件：`issues.opened`, `issue_comment.created`, `schedule`。

2. 仓库内部依赖
- Claude Code Action：`anthropics/claude-code-action@v1`（triage/dedupe workflow 使用）。
- CLI/runtime：`gh`, `bun`, shell 脚本执行环境。

3. 外部服务
- GitHub REST API（issue/comment/label/events/reactions）。
- Statsig 事件上报（`.github/workflows/claude-dedupe-issues.yml:36-83`, `.github/workflows/log-issue-events.yml:1-40`）。
- 模板内外链：npm versions、GitHub issue 搜索。

4. 测试/校验现状
- 仓库内未看到对 `bug_report.yml` 的专门 schema lint 或表单回归测试；主要依赖线上创建 issue 时暴露错误。

## 风险、边界与改进建议

1. 风险：条件必填未表达
- 现状：`working_version` 没有随 “regression=Yes” 自动变必填（Issue Form 本身也不支持复杂条件）。
- 影响：回归问题可能缺少关键版本定位信息。
- 建议：在 triage 提示词中增加“若 regression=Yes 且缺 working_version，优先补充 needs-info”规则。

2. 风险：日志字段可能包含敏感信息
- 现状：`error_output` 鼓励粘贴日志，但无敏感信息提醒。
- 影响：token/path/内部地址泄露风险。
- 建议：在模板文案加入“提交前脱敏”提醒，或增加专门 preflight 复选项。

3. 风险：环境选项演进滞后
- 现状：`terminal` 下拉是静态列表（`.github/ISSUE_TEMPLATE/bug_report.yml:154-173`）。
- 影响：新终端生态出现后会落入 `Other`，降低统计质量。
- 建议：定期基于 issue 数据补充选项，并在 triage 中保留 `Other` 的语义归类规则。

4. 边界：模板提高输入质量，但不保证可复现
- 即使字段齐全，仍可能缺少最小工程样例；是否可复现最终依赖人工复核与二次沟通。

5. 建议：增加“最小复现资产”引导
- 在 `reproduction` 描述中加入固定提示：优先提供可运行最小仓库或最小文件片段，减少来回追问成本。
