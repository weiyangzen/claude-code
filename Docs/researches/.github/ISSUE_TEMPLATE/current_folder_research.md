# DIR `.github/ISSUE_TEMPLATE` 研究文档

## 场景与职责

`.github/ISSUE_TEMPLATE` 是 GitHub Issues 的“入口契约层”。它不直接执行代码，但决定了用户提单时的结构化输入、默认标签与分流路径；随后这些输入会被仓库内自动化流程消费。

本目录承担三类职责：

1. 入口治理
- 通过 `config.yml` 关闭空白 issue（`blank_issues_enabled: false`），强制用户从预设表单进入。
- 提供 `contact_links`，将求助/文档问题优先分流到 Discord 与官方文档，减少无效 issue。

2. 结构化采集
- 通过 4 个 issue form（bug/feature/docs/model）采集固定字段（`id`）和必填项（`required: true`）。
- 在 issue 创建时自动附加分类标签（`bug`/`enhancement`/`documentation`/`model`）。

3. 下游自动化的“输入标准化”
- 新 issue 打开后会触发 triage、dedupe、统计、跨仓 dispatch、生命周期治理等 workflow。
- 这些流程虽不直接解析 YAML 字段 `id`，但依赖表单输出的正文结构和默认标签做分类与自动处理。

## 功能点目的

### 1) `bug_report.yml`
目的：收集可复现缺陷所需的最小排障信息，降低 `needs-repro` / `needs-info` 生命周期标签出现概率。

关键点：
- 默认标题前缀 `[BUG] `、默认标签 `bug`。
- 强制字段覆盖“实际行为 / 期望行为 / 复现步骤 / 回归判断 / 版本 / 平台 / OS / 终端”。
- `error_output` 使用 `render: shell`，便于日志粘贴与可读性。

### 2) `feature_request.yml`
目的：把“需求背景”与“方案设想”分离，提升需求讨论质量。

关键点：
- 默认标题前缀 `[FEATURE] `、默认标签 `enhancement`。
- 强制 `problem` + `solution` + `priority` + `category`，避免只给结论不给场景。

### 3) `documentation.yml`
目的：将文档问题显式分类（缺失/错误/歧义/坏链），支持文档维护优先级排序。

关键点：
- 默认标题前缀 `[DOCS] `、默认标签 `documentation`。
- 强制 `doc_type`、`section`、`issue`、`suggested`、`impact`，可直接用于文档修复工单。

### 4) `model_behavior.yml`
目的：将“模型行为不符合预期”与“程序缺陷/安装问题”分离，减少误分流。

关键点：
- 默认标题前缀 `[MODEL] `、默认标签 `model`。
- 强制采集“用户请求 vs 实际行为 vs 期望行为”，包含权限模式、可复现性、影响级别。

### 5) `config.yml`
目的：把不适合走 issue 的请求导向外部渠道，降低仓库 triage 压力。

关键点：
- 禁止空白 issue。
- `contact_links` 指向 Discord、文档、Quickstart、Troubleshooting。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程（端到端）

1. 用户进入 `https://github.com/<owner>/<repo>/issues/new/choose`（仓库内 `scripts/sweep.ts` 的关闭提示也指向该入口）。
2. GitHub 根据 `.github/ISSUE_TEMPLATE/config.yml` 展示可选表单并禁用空白 issue。
3. 用户提交表单后，GitHub 生成 issue：
- 标题应用模板前缀（如 `[BUG] `）。
- 自动打上模板定义标签（如 `bug`）。
- 表单字段渲染为 markdown 小节写入 issue body。
4. `issues.opened` 触发多个 workflow：
- `claude-issue-triage.yml`：调用 `anthropics/claude-code-action@v1` 执行 `/triage-issue ...`。
- `claude-dedupe-issues.yml`：调用 `/dedupe ...` 搜索重复问题并发评论。
- `issue-opened-dispatch.yml`：向目标仓库发 repository dispatch。
- `log-issue-events.yml`：上报 Statsig 事件。
5. 后续生命周期治理：
- `issue-lifecycle-comment.yml`、`sweep.yml`、`remove-autoclose-label.yml`、`lock-closed-issues.yml`、`auto-close-duplicates.yml` 按标签和活跃度执行提醒/关闭/锁定。

### 2) 数据结构

#### A. Issue Form 文档结构（YAML）
通用结构：
- 顶层：`name` / `description` / `title` / `labels` / `body`。
- `body[]` 项：`type`（`markdown|checkboxes|textarea|dropdown|input`）+ `id` + `attributes` + `validations`。
- 必填约束：`validations.required: true`。

#### B. 目录内已定义的标签与标题前缀
- `bug_report.yml` -> `[BUG] ` + `bug`
- `feature_request.yml` -> `[FEATURE] ` + `enhancement`
- `documentation.yml` -> `[DOCS] ` + `documentation`
- `model_behavior.yml` -> `[MODEL] ` + `model`

#### C. 生命周期配置（下游脚本）
虽然不在本目录，但直接影响模板价值兑现：
- `scripts/issue-lifecycle.ts` 定义 `invalid(3d) / needs-repro(7d) / needs-info(7d) / stale(14d) / autoclose(14d)`。
- triage 与 sweep 流程据此执行提醒与超时关闭。

### 3) 协议与命令

#### A. GitHub 事件协议
- `issues.opened`：主要消费表单产出的正文与默认标签。
- `issue_comment.created`：用于去除 `autoclose/stale` 或重新 triage。
- `schedule/workflow_dispatch`：做批量去重、清理、关闭。

#### B. 仓库内关键命令链
- triage：`/triage-issue ...` -> `.claude/commands/triage-issue.md` -> `./scripts/gh.sh` + `./scripts/edit-issue-labels.sh`
- dedupe：`/dedupe ...` -> `.claude/commands/dedupe.md` -> `./scripts/gh.sh` + `./scripts/comment-on-duplicates.sh`
- lifecycle sweep：`bun run scripts/sweep.ts`
- duplicate autoclose：`bun run scripts/auto-close-duplicates.ts`

#### C. 安全收敛
- `scripts/gh.sh` 白名单限定仅允许 `issue view/list`, `search issues`, `label list` 及有限 flags，避免 triage agent 过权访问。

## 关键代码路径与文件引用

### 1) 目标目录（被研究对象）
- `.github/ISSUE_TEMPLATE/config.yml`
- `.github/ISSUE_TEMPLATE/bug_report.yml`
- `.github/ISSUE_TEMPLATE/feature_request.yml`
- `.github/ISSUE_TEMPLATE/documentation.yml`
- `.github/ISSUE_TEMPLATE/model_behavior.yml`

### 2) 调用方（谁消费这个目录产出）
- `.github/workflows/claude-issue-triage.yml`
- `.github/workflows/claude-dedupe-issues.yml`
- `.github/workflows/issue-opened-dispatch.yml`
- `.github/workflows/log-issue-events.yml`
- `.github/workflows/issue-lifecycle-comment.yml`
- `.github/workflows/sweep.yml`
- `.github/workflows/auto-close-duplicates.yml`
- `.github/workflows/remove-autoclose-label.yml`
- `.github/workflows/lock-closed-issues.yml`

### 3) 被调用方（workflow 继续下钻）
- `.claude/commands/triage-issue.md`
- `.claude/commands/dedupe.md`
- `scripts/gh.sh`
- `scripts/edit-issue-labels.sh`
- `scripts/comment-on-duplicates.sh`
- `scripts/lifecycle-comment.ts`
- `scripts/issue-lifecycle.ts`
- `scripts/sweep.ts`
- `scripts/auto-close-duplicates.ts`
- `scripts/backfill-duplicate-comments.ts`

### 4) 相关文档与入口
- `README.md`（官方文档入口）
- `scripts/sweep.ts` 中 `NEW_ISSUE`（关闭后重开入口链接）

## 依赖与外部交互

### 1) 平台依赖
- GitHub Issue Forms（由 GitHub 平台解释 `.github/ISSUE_TEMPLATE/*.yml`）。
- GitHub Actions 事件系统（`issues`, `issue_comment`, `schedule`, `workflow_dispatch`）。

### 2) 仓库运行时依赖
- `anthropics/claude-code-action@v1`（triage/dedupe 执行器）。
- `gh` CLI（dispatch、issue/label 操作）。
- `bun`（执行 `scripts/*.ts`）。

### 3) 外部服务交互
- GitHub REST API（issue、comment、label、events、reactions）。
- Statsig 事件上报（`log-issue-events.yml`, `claude-dedupe-issues.yml`）。
- 模板内外链：npm versions、Discord、Claude 文档站。

### 4) 配置/密钥
- `GITHUB_TOKEN`
- `ANTHROPIC_API_KEY`
- `STATSIG_API_KEY`
- `ISSUE_OPENED_DISPATCH_TOKEN` / `ISSUE_OPENED_DISPATCH_TARGET_REPO`

## 风险、边界与改进建议

1. 风险：模板字段变更缺少自动校验
- 现状：未发现针对 issue form 的 schema lint/test 流程。
- 影响：字段拼写或结构错误通常在运行期（用户提交时）才暴露。
- 建议：增加 CI 校验（最少做 YAML 语法 + 关键字段存在性检查）。

2. 风险：标题前缀与分类标签可能漂移
- 现状：模板定义了 `[BUG]/[FEATURE]/[DOCS]/[MODEL]` 和默认标签；triage 又会二次判断并打标。
- 影响：若模板语义与 triage 规则偏离，可能出现标签不一致。
- 建议：在 triage 指令中显式定义“保留模板主类别标签”的策略，或增加冲突检测。

3. 风险：模型行为与普通 bug 边界仍有交叉
- 现状：`model_behavior.yml` 与 `bug_report.yml` 均可能承载“异常行为”。
- 影响：同类问题分散到不同标签，影响聚类与优先级管理。
- 建议：增加互斥引导（例如在 bug 模板中增加跳转提示），并在 triage 中统一二次归类规则。

4. 风险：模板中的外链域名存在不一致历史风险
- 现状：仓库 README 与模板内文档链接域名并不完全一致（`code.claude.com` 与 `docs.claude.com` 体系并存）。
- 影响：后续站点迁移时容易出现部分过期链接。
- 建议：集中维护文档基地址（单一变量源），由脚本生成模板链接。

5. 边界：本目录只负责“输入定义”，不负责执行
- 说明：真正的业务动作（打标、评论、关闭、锁定、上报）发生在 workflow 与 scripts 层；因此该目录优化的核心价值是“输入质量”和“分流精度”，不是执行逻辑本身。

6. 改进建议：为关键模板补“最小可行动信息”质量门槛
- 示例：在 bug/model 模板增加“最小复现仓库或最小样例片段”选填引导，以及“无日志时说明原因”的提示语。
- 预期：减少 triage 回合数，提高首次响应可处理度。
