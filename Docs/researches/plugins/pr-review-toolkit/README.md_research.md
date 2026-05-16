# plugins/pr-review-toolkit/README.md 研究

## 场景与职责

`plugins/pr-review-toolkit/README.md` 是 `pr-review-toolkit` 插件的用户侧主文档，职责不是执行审查，而是定义“如何使用插件能力”的交互契约。

1. 对外能力说明入口：概述插件由 6 个专项 agent 组成，覆盖评论、测试、错误处理、类型设计、通用代码质量、代码简化（`plugins/pr-review-toolkit/README.md:1-8`）。
2. 用户操作手册：给出触发语句、适用时机、组合使用和排障建议（`plugins/pr-review-toolkit/README.md:11-295`）。
3. 与命令协议对齐层：README 描述的能力需落到 `/pr-review-toolkit:review-pr` 与 6 个 agent 协议文件，否则会出现“文档承诺与实际行为偏差”（`plugins/pr-review-toolkit/commands/review-pr.md:1-189`，`plugins/pr-review-toolkit/agents/*.md`）。

从插件生态看，它是被市场与插件索引“发现”的文档资产：

- marketplace 将插件注册到 `./plugins/pr-review-toolkit`（`.claude-plugin/marketplace.json:117-125`）。
- 插件总览将该目录公开为 PR 审查工具并展示命令/agent 列表（`plugins/README.md:25`）。

## 功能点目的

README 里的功能点可归为 5 组目的。

1. 能力分工（6 个 agent）
- 通过“每个 agent 一个质量维度”减少泛化审查噪声：
  - `comment-analyzer`（注释准确性）
  - `pr-test-analyzer`（测试覆盖）
  - `silent-failure-hunter`（静默失败/错误处理）
  - `type-design-analyzer`（类型与不变量）
  - `code-reviewer`（通用规范与缺陷）
  - `code-simplifier`（可读性与简化）
  （`plugins/pr-review-toolkit/README.md:11-138`）

2. 触发与编排
- 提供自然语言触发示例，降低用户记忆命令参数的门槛（`plugins/pr-review-toolkit/README.md:140-180`）。
- 支持“单点检查”和“全量检查”两种审查策略（`plugins/pr-review-toolkit/README.md:157-170`）。

3. 工作流嵌入
- README 把 agent 插入典型 PR 生命周期：提交前、建 PR 前、评审后、反馈修复后复检（`plugins/pr-review-toolkit/README.md:220-295`）。

4. 结果消费方式
- 强调结构化输出（问题定位、原因、改进建议、按严重性排序），目的是把审查结果直接变成行动清单（`plugins/pr-review-toolkit/README.md:211-219`）。

5. 运维与排障
- 提供触发失败与范围错误两类常见问题的排障建议（`plugins/pr-review-toolkit/README.md:263-282`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

README 本身是文档协议，不是脚本实现。其“技术实现”体现在与命令/agent 协议的一致性。

### 1) 关键流程

1. 安装与发现
- 用户通过 `/plugins` 安装该插件（`plugins/pr-review-toolkit/README.md:181-191`）。
- 插件元数据由 manifest 声明（`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`），并由 marketplace 聚合（`.claude-plugin/marketplace.json:117-125`）。

2. 触发路径
- README 给出两类入口：
  - 自然语言请求触发特定 agent（`plugins/pr-review-toolkit/README.md:144-155`）。
  - slash command 入口 `/pr-review-toolkit:review-pr`（由命令文件定义，`plugins/pr-review-toolkit/commands/review-pr.md:93-113`）。

3. 执行与聚合
- `review-pr` 命令协议定义：识别变更、映射审查面、串行/并行调度、统一汇总（`plugins/pr-review-toolkit/commands/review-pr.md:15-88`）。
- README 中“并行/串行”建议与命令协议一一对应（`plugins/pr-review-toolkit/README.md:241-253` 对 `review-pr.md:47-56,109-113`）。

### 2) 关键数据结构（协议结构）

1. 命令 frontmatter 结构
- `description`、`argument-hint`、`allowed-tools`（`plugins/pr-review-toolkit/commands/review-pr.md:1-5`）。
- 可选参数域：`comments/tests/errors/types/code/simplify/all`（`plugins/pr-review-toolkit/commands/review-pr.md:20-29`）。

2. agent frontmatter 结构
- 每个 agent 至少包含 `name/description/model`，并通过正文定义输出格式和审查规则：
  - `comment-analyzer`（`plugins/pr-review-toolkit/agents/comment-analyzer.md:1-70`）
  - `pr-test-analyzer`（`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:1-69`）
  - `silent-failure-hunter`（`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:1-130`）
  - `type-design-analyzer`（`plugins/pr-review-toolkit/agents/type-design-analyzer.md:1-110`）
  - `code-reviewer`（`plugins/pr-review-toolkit/agents/code-reviewer.md:1-47`）
  - `code-simplifier`（`plugins/pr-review-toolkit/agents/code-simplifier.md:1-83`）

3. 输出协议（README 与 agent 的映射）
- README 声称“结构化输出 + 严重度优先” （`plugins/pr-review-toolkit/README.md:211-219`）。
- 实际 agent 协议中落地为：
  - `code-reviewer`：0-100 置信度，仅报告 >=80（`plugins/pr-review-toolkit/agents/code-reviewer.md:22-33`）。
  - `pr-test-analyzer`：建议项 1-10 关键度（`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:27-48`）。
  - `silent-failure-hunter`：`CRITICAL/HIGH/MEDIUM` 严重级（`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:103-109`）。
  - `type-design-analyzer`：四维度 1-10 评分（`plugins/pr-review-toolkit/agents/type-design-analyzer.md:24-47,58-69`）。

### 3) 关键命令与外部命令调用

`review-pr` 文档协议明确依赖命令：

- `git diff --name-only`：识别改动文件（`plugins/pr-review-toolkit/commands/review-pr.md:31`）。
- `gh pr view`：读取 PR 上下文（`plugins/pr-review-toolkit/commands/review-pr.md:32`）。
- `Task` 工具：调度子 agent（`plugins/pr-review-toolkit/commands/review-pr.md:4`）。

## 关键代码路径与文件引用

### 目标对象

- `plugins/pr-review-toolkit/README.md:1-313`

### 调用方（上游）

1. 插件索引文档（目录导航）
- `plugins/README.md:25`

2. marketplace 配置（安装与发现入口）
- `.claude-plugin/marketplace.json:117-125`

3. 研究流程与清单系统（非运行时）
- `Docs/researches/blueprint_checklist.md:279-287`
- `Docs/researches/todos_20260320.md:14-21`

### 被调用方（下游/语义落地）

1. 插件元数据
- `plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`

2. 命令编排协议
- `plugins/pr-review-toolkit/commands/review-pr.md:1-189`

3. 六个专项 agent 协议
- `plugins/pr-review-toolkit/agents/comment-analyzer.md:1-70`
- `plugins/pr-review-toolkit/agents/pr-test-analyzer.md:1-69`
- `plugins/pr-review-toolkit/agents/silent-failure-hunter.md:1-130`
- `plugins/pr-review-toolkit/agents/type-design-analyzer.md:1-110`
- `plugins/pr-review-toolkit/agents/code-reviewer.md:1-47`
- `plugins/pr-review-toolkit/agents/code-simplifier.md:1-83`

### 配置、测试、脚本、文档上下文

1. 配置
- 插件配置：`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`
- 命令配置（frontmatter）：`plugins/pr-review-toolkit/commands/review-pr.md:1-5`
- 市场配置：`.claude-plugin/marketplace.json:117-125`

2. 测试
- `plugins/pr-review-toolkit` 目录仅包含 `README.md`、`commands/*.md`、`agents/*.md`、`plugin.json`，无 `*.test.* / *.spec.* / __tests__`。

3. 脚本
- 目录下无 `*.sh/*.py/*.js/*.ts` 可执行脚本；流程依赖命令协议与 Task 调度，不是脚本化实现。

4. 文档
- 目标 README 是用户文档主入口（`plugins/pr-review-toolkit/README.md:1-313`）。
- `commands/review-pr.md` 是“执行协议文档”（`plugins/pr-review-toolkit/commands/review-pr.md:1-189`）。
- `agents/*.md` 是“角色协议文档”（见上文路径）。

## 依赖与外部交互

1. Claude Code 插件机制
- 依赖 `.claude-plugin` manifest + marketplace source 完成插件发现与安装（`plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`，`.claude-plugin/marketplace.json:117-125`）。

2. Claude 工具能力
- `review-pr` frontmatter 允许 `Bash/Glob/Grep/Read/Task`（`plugins/pr-review-toolkit/commands/review-pr.md:4`），说明插件通过宿主工具执行检索与 agent 调度。

3. 外部命令与平台
- 依赖本地 Git 与 GitHub CLI（`git diff --name-only`, `gh pr view`）获取改动与 PR 上下文（`plugins/pr-review-toolkit/commands/review-pr.md:31-33`）。

4. 项目规范文档依赖
- 多个 agent 明确依赖 `CLAUDE.md` 规则进行审查（`plugins/pr-review-toolkit/agents/code-reviewer.md:8,16`；`plugins/pr-review-toolkit/agents/pr-test-analyzer.md:62`；`plugins/pr-review-toolkit/agents/silent-failure-hunter.md:123-126`；`plugins/pr-review-toolkit/agents/code-simplifier.md:44`）。
- 当前仓库未检出 `CLAUDE.md`，属于运行时上下文依赖而非仓库内硬文件。

## 风险、边界与改进建议

### 风险

1. 文档承诺与协议细节存在轻微漂移
- README 声称“所有 agent 都提供 confidence scoring”（`plugins/pr-review-toolkit/README.md:195-209`），但实际只有部分 agent 有明确数值评分；`comment-analyzer` 输出模板不含显式数值分（`plugins/pr-review-toolkit/agents/comment-analyzer.md:48-67`）。

2. 作者信息双源不一致
- README/manifest 作者为 Daisy（`plugins/pr-review-toolkit/README.md:307-309`，`plugins/pr-review-toolkit/.claude-plugin/plugin.json:6-7`），marketplace 条目作者为 Anthropic（`.claude-plugin/marketplace.json:120-123`）。

3. 无自动化测试或脚本校验
- 该插件完全由 markdown 协议驱动，缺少测试与 lint 容易导致提示词/frontmatter 漂移无人发现。

4. 外部依赖失败降级未在 README 明确
- README 介绍了触发方式与排障，但未说明 `gh pr view` 不可用时的降级策略。

### 边界

1. README 仅定义用户协作协议，不直接执行审查逻辑。
2. 实际审查质量取决于命令编排是否遵循 `review-pr.md` 和 agent 提示词质量。
3. 插件不提供强约束执行器；属于“软协议”体系。

### 改进建议

1. 在 README 增加“执行前置条件”
- 明确 `gh` 安装/登录要求，以及缺失时仅基于 `git diff` 的降级行为。

2. 在 README 增加“能力映射表”
- 将 README 中 6 个能力点逐条映射到 `commands/review-pr.md` 与对应 `agents/*.md`，便于维护一致性。

3. 引入文档一致性检查
- 在 CI 增加检查：
  - README 中 agent 列表与 `agents/` 文件名一致
  - README 中命令参数与 `review-pr.md` 一致
  - 作者/版本在 manifest 与 marketplace 对齐

4. 为无评分 agent 补充明确分级语义
- 至少在 README 中注明哪些 agent 使用数值评分、哪些使用分级/定性，避免用户误解“统一评分”。
