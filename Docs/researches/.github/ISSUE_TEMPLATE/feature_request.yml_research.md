# FILE `.github/ISSUE_TEMPLATE/feature_request.yml` 研究文档

## 场景与职责

`feature_request.yml` 承担产品需求收集入口，目标是把“想要什么”转换为“为什么需要 + 预期交互 + 优先级 + 分类”四类结构化输入。模板输出将直接影响需求 triage、重复检测和后续路线图讨论。

职责边界：

- 定义默认元信息：`[FEATURE] ` 标题前缀、`enhancement` 默认标签（`.github/ISSUE_TEMPLATE/feature_request.yml:3-5`）。
- 约束提案质量：问题陈述、方案描述、优先级、分类必填（`:25-104`）。
- 与 bug/model/docs 模板区分，降低“缺陷误报为需求”与“需求误报为缺陷”的概率。

## 功能点目的

1. 强制先讲问题再讲方案
- `problem` 与 `solution` 均为必填，并在说明中强调先描述痛点（`:25-55`）。
- 目的：避免仅给出命令/参数建议而无真实业务场景。

2. 收集优先级与功能域
- `priority`（Critical/High/Medium/Low）+ `category`（CLI/TUI/File/API/MCP/性能/配置/SDK/文档）必填（`:73-103`）。
- 目的：把需求输入映射到维护者可执行的排序维度。

3. 控制 issue 粒度与去重
- preflight 要求“已搜索现有请求、单 issue 单功能”（`:15-24`）。
- 目的：减少重复、避免超大范围需求单影响可推进性。

## 具体技术实现（关键流程/数据结构/协议/命令）

1. 表单结构
- `body[]` 包含：
- `checkboxes preflight`（必选）
- `textarea problem/solution`（必填）
- `textarea alternatives/use_case/additional`（可选）
- `dropdown priority/category`（必填）

2. 数据协议与消费方式
- Issue Form 最终渲染为 markdown 正文，而非机器可直接反序列化 JSON。
- triage 流程使用 `gh issue view` 读取文本，再按语义与标签决策（`scripts/gh.sh:29-96`, `.claude/commands/triage-issue.md:35-39`）。
- triage 指令明确“生命周期标签仅用于 bug”，因此 enhancement issue 原则上不应被 `needs-repro/needs-info` 标记（`.claude/commands/triage-issue.md:44-46,68`）。

3. 端到端执行流程
1) 用户提交 feature request -> 自动打 `enhancement`。
2) `issues.opened` 触发：
- triage (`claude-issue-triage.yml`)
- dedupe (`claude-dedupe-issues.yml`)
- logging (`log-issue-events.yml`)
- dispatch (`issue-opened-dispatch.yml`)
3) 若后续出现“重复建议且无人反馈”，duplicate 系列脚本可能加注释或自动关闭（`scripts/comment-on-duplicates.sh`, `scripts/auto-close-duplicates.ts`）。

4. 命令与标签一致性机制
- `scripts/edit-issue-labels.sh` 会先拉取仓库 label 列表再执行 add/remove（`scripts/edit-issue-labels.sh:47-80`），防止模板默认标签与仓库真实标签不一致时硬失败。

## 关键代码路径与文件引用

- 当前模板：`.github/ISSUE_TEMPLATE/feature_request.yml`
- 全局配置：`.github/ISSUE_TEMPLATE/config.yml`
- triage workflow：`.github/workflows/claude-issue-triage.yml`
- triage 指令：`.claude/commands/triage-issue.md`
- dedupe workflow：`.github/workflows/claude-dedupe-issues.yml`
- dedupe 指令：`.claude/commands/dedupe.md`
- issue 查询封装：`scripts/gh.sh`
- 标签编辑脚本：`scripts/edit-issue-labels.sh`
- duplicate 评论与自动关闭：`scripts/comment-on-duplicates.sh`, `scripts/auto-close-duplicates.ts`
- 生命周期规则（间接相关）：`scripts/issue-lifecycle.ts`, `scripts/sweep.ts`

## 依赖与外部交互

1. 平台依赖
- GitHub Issue Form 的 dropdown/checkbox/textarea 渲染及必填检查。

2. 仓库依赖
- GitHub Actions + Claude Code Action + gh/bun 工具链。

3. 外部交互
- preflight 中查重链接依赖 GitHub issue search（`:20`）。
- 若 workflow 启用统计，issue 事件会上报 Statsig（`.github/workflows/log-issue-events.yml`）。

4. 测试与验证
- 未见 feature 模板的自动化质量门禁（例如字段语义检查、下拉值漂移检查）。

## 风险、边界与改进建议

1. 风险：优先级字段主观性高
- 现状：优先级完全由提单人选择。
- 影响：Critical/High 可能被过度使用，影响排序信号。
- 建议：triage 侧增加“优先级重写/补充标签”策略，如 `priority-user-high` 与 `priority-maintainer` 分离。

2. 风险：`category` 选项随产品演进可能过时
- 现状：类别是静态枚举。
- 影响：新能力被频繁归为 `Other`，降低统计可用性。
- 建议：按季度统计 `Other` 高频词并回写分类枚举。

3. 风险：需求和 bug 仍可能混单
- 现状：用户可能在 feature 模板中描述当前行为错误。
- 影响：triage 需要二次改标签，增加处理成本。
- 建议：在模板顶部增加“若当前行为与预期不符请改用 Bug Report”的显式跳转提示。

4. 边界：模板只保证信息结构，不保证方案可行性
- `solution` 字段提高讨论效率，但不能替代架构评估与实现成本分析。

5. 建议：新增“成功判据”字段
- 增加可选字段要求写出验收标准（如 CLI 输出、交互行为），有助于后续实现与测试闭环。
