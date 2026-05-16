# plugins/pr-review-toolkit/commands/review-pr.md 研究

## 场景与职责

`plugins/pr-review-toolkit/commands/review-pr.md` 是 `pr-review-toolkit` 插件的主命令协议文件，职责是把用户的 `/pr-review-toolkit:review-pr` 请求编排为“多 agent 协同审查 + 统一汇总输出”的流程，而不是直接执行静态分析算法。

它在系统中的位置是：
1. marketplace 注册插件来源目录 `./plugins/pr-review-toolkit`（`.claude-plugin/marketplace.json:117-125`）。
2. 插件元信息由 `plugins/pr-review-toolkit/.claude-plugin/plugin.json` 提供（`name/version/description/author`，见 `plugin.json:1-9`）。
3. Claude Code 自动扫描 `commands/*.md` 建立 slash command（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-345`）。
4. 文件名 `review-pr.md` 对应命令名 `review-pr`，与插件名拼接形成 `/pr-review-toolkit:review-pr`（`SKILL.md:307-309`，`plugins/README.md:25`）。

职责边界：
- 负责评审范围判定、子 agent 选择、并发策略与结果汇总（`review-pr.md:15-88`）。
- 不负责具体规则实现；具体审查逻辑在 `agents/*.md`。
- 不直接包含可执行脚本、测试代码或 CI 逻辑，属于“提示词协议层”。

## 功能点目的

### 1. 提供统一 PR 质量评审入口
- 通过一个命令聚合评论、测试、错误处理、类型、通用代码质量、简化六类能力（`review-pr.md:22-29`）。
- 支持默认全量评审（`all`）和按方面定向评审（`review-pr.md:18,22-29,99-107`）。

### 2. 基于改动上下文做动态路由
- 先通过 `git diff --name-only` 和 `gh pr view` 获取变更与 PR 上下文（`review-pr.md:30-33`）。
- 再按改动特征决定触发哪些 agent，减少无关扫描（`review-pr.md:37-43`）。

### 3. 分离“问题发现”与“代码抛光”阶段
- `code-simplifier` 被定义为“After passing review”阶段（`review-pr.md:43`），强调先修复缺陷、再做简化。

### 4. 提供串行/并行两种执行模式
- 串行强调可读性和交互性（`review-pr.md:47-51`）。
- 并行强调吞吐与速度（`review-pr.md:52-56,109-113`）。

### 5. 标准化跨 agent 输出
- 定义统一汇总桶：`Critical Issues / Important Issues / Suggestions / Positive Observations`（`review-pr.md:59-64`）。
- 给出固定行动顺序模板，直接转化为修复计划（`review-pr.md:67-88`）。

### 6. 嵌入开发工作流节点
- 覆盖提交前、建 PR 前、反馈修复后三个场景（`review-pr.md:156-181`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. Frontmatter 协议
`review-pr.md` 通过 frontmatter 定义命令元协议：
- `description`：命令语义描述（`review-pr.md:2`）。
- `argument-hint`：参数提示为 `[review-aspects]`（`review-pr.md:3`）。
- `allowed-tools`：`Bash/Glob/Grep/Read/Task`（`review-pr.md:4`）。

含义：
- `Task` 允许调用下游 agent。
- `Bash` 支持执行 `git/gh` 上下文命令。
- `Read/Grep/Glob` 支持按需读取代码与文件。

### B. 执行流程（命令状态机）
1. 读取输入参数 `$ARGUMENTS`（`review-pr.md:11`）。
2. 识别评审范围，默认 `all`（`review-pr.md:15-29`）。
3. 执行 `git diff --name-only` 获取 changed files（`review-pr.md:31`）。
4. 执行 `gh pr view` 获取 PR 视图信息（`review-pr.md:32`）。
5. 根据改动类型映射 agent（`review-pr.md:37-43`）。
6. 按串行或并行模式调度 agent（`review-pr.md:45-56,109-113`）。
7. 汇总多 agent 结果并输出结构化行动计划（`review-pr.md:57-88`）。

### C. 隐含数据结构（从协议反推）
1. `ReviewAspects`
- 值域：`comments | tests | errors | types | code | simplify | all`（`review-pr.md:22-29`）。
- 扩展模式词：`parallel`（示例中出现，`review-pr.md:109-113`）。

2. `ChangedContext`
- `changedFiles`: `git diff --name-only` 产物（`review-pr.md:31`）。
- `prContext`: `gh pr view` 产物（`review-pr.md:32`）。

3. `ActivatedAgents`
- 固定映射表（`review-pr.md:38-43`）：
  - always: `code-reviewer`
  - test changes: `pr-test-analyzer`
  - comment/doc changes: `comment-analyzer`
  - error handling changes: `silent-failure-hunter`
  - type changes: `type-design-analyzer`
  - post-review polish: `code-simplifier`

4. `AggregatedFindings`
- 统一输出桶（`review-pr.md:60-64,71-82`）。

5. `ActionPlan`
- 固定四步修复顺序（`review-pr.md:83-87`）。

### D. 下游 agent 协议差异（聚合关键点）
`review-pr` 的核心技术难点是“异构输出归一化”。下游 agent 的评分/分级体系并不一致：
- `code-reviewer`：0-100 且仅报告 >=80（`agents/code-reviewer.md:22-33`）。
- `pr-test-analyzer`：1-10 关键度（`agents/pr-test-analyzer.md:27-48`）。
- `type-design-analyzer`：四维 1-10 评分（`agents/type-design-analyzer.md:24-47,58-69`）。
- `silent-failure-hunter`：`CRITICAL/HIGH/MEDIUM`（`agents/silent-failure-hunter.md:103-109`）。
- `comment-analyzer` 与 `code-simplifier`更偏结构化文本建议（`agents/comment-analyzer.md:48-67`，`agents/code-simplifier.md:38-83`）。

因此，`review-pr.md` 的 `Aggregate Results` 和 `Provide Action Plan` 段实际上承担“统一汇总协议”的作用（`review-pr.md:57-88`）。

## 关键代码路径与文件引用

### 目标对象
- `plugins/pr-review-toolkit/commands/review-pr.md:1-189`

### 调用方（上游入口/装配）
- `.claude-plugin/marketplace.json:117-125`（marketplace 注册）
- `plugins/pr-review-toolkit/.claude-plugin/plugin.json:1-9`（插件 manifest）
- `plugins/README.md:25`（插件目录索引，暴露命令）
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:307-309,343-345`（命名约定与自动发现机制）

### 被调用方（下游 agent）
- `plugins/pr-review-toolkit/agents/comment-analyzer.md:1-70`
- `plugins/pr-review-toolkit/agents/pr-test-analyzer.md:1-69`
- `plugins/pr-review-toolkit/agents/silent-failure-hunter.md:1-130`
- `plugins/pr-review-toolkit/agents/type-design-analyzer.md:1-110`
- `plugins/pr-review-toolkit/agents/code-reviewer.md:1-47`
- `plugins/pr-review-toolkit/agents/code-simplifier.md:1-83`

### 关联文档
- `plugins/pr-review-toolkit/README.md:1-313`（用户指南与使用场景）
- `plugins/README.md:1-77`（官方插件目录总览）

### 测试/脚本实物证据
- `plugins/pr-review-toolkit/commands/` 当前仅 `review-pr.md`，无 `*.test.*`/`*.spec.*`。
- `plugins/pr-review-toolkit/` 无 `.sh/.py/.ts/.js` 执行脚本；该命令依赖运行时工具而非本地脚本实现。

## 依赖与外部交互

### 内部依赖
1. 插件发现机制依赖标准目录结构与 manifest（`plugin-structure/SKILL.md:341-349`）。
2. 命令中引用的 agent 名称必须与 `agents/*.md` frontmatter `name` 对齐。
3. 多个 agent 明确依赖 `CLAUDE.md` 规范作为审查准绳（如 `code-reviewer.md:8,16`、`pr-test-analyzer.md:62`、`silent-failure-hunter.md:123-126`、`code-simplifier.md:44`）。

### 外部交互
1. `git diff --name-only`：读取本地仓库变更（`review-pr.md:31`）。
2. `gh pr view`：读取 GitHub PR 上下文（`review-pr.md:32`），依赖 `gh` 可执行、登录态与网络。
3. `Task`：向多个审查 agent 分派任务（`review-pr.md:4,45-56`）。

### 配置、测试、脚本、文档依赖结论
- 配置：命令 frontmatter + plugin manifest + agent frontmatter。
- 测试：目标命令无自动化测试保障，属于“协议即行为”。
- 脚本：无专用执行脚本，行为依赖模型遵循提示词流程。
- 文档：`README.md` 与命令示例耦合较紧，必须保持一致，否则用户感知能力与实际编排会偏离。

## 风险、边界与改进建议

### 风险
1. **参数语义非形式化**
- `all parallel` 仅以示例表达（`review-pr.md:109-113`），没有明确语法或冲突处理规则。

2. **外部命令失败未定义降级路径**
- `gh pr view` 失败（未登录/离线/非 PR 分支）时没有显式 fallback 说明（`review-pr.md:30-33`）。

3. **异构评分归一化缺口**
- 下游 agent 分数体系不统一，命令层虽有汇总模板，但未给出明确映射策略，可能导致不同执行轮次聚合风格波动。

4. **协议层缺少自动回归**
- 当前是 Markdown 流程定义，缺少机器可执行校验，重构文案时容易破坏可执行一致性。

5. **文档漂移风险**
- `review-pr.md`、`README.md`、`plugins/README.md` 同时描述能力点，缺少一致性检查。

### 边界
1. 本文件不实现具体审查算法，只定义审查编排协议。
2. 本文件不直接发 GitHub 评论、不运行测试命令，也不修改业务代码。
3. 它依赖 Claude Code 运行时对命令 markdown 与 `Task` 语义的正确解释。

### 改进建议
1. 增加命令协议 lint
- 校验 `commands/review-pr.md` 引用的 agent 是否全部存在。
- 校验 frontmatter 字段完整性与类型合法性。

2. 明确参数语法
- 在命令中显式写成 `<aspects...> [parallel]`，并定义冲突输入处理（例如 `all` 与其他 aspect 同时出现时的优先级）。

3. 定义统一分级映射表
- 例如把 `0-100 / 1-10 / 枚举严重级` 统一映射到 `Critical/Important/Suggestion`，减少汇总歧义。

4. 定义 `gh` 降级策略
- `gh pr view` 失败时自动退化为仅基于 `git diff` 的本地审查，并在摘要中声明“PR 元信息缺失”。

5. 增加文档一致性检查脚本
- 自动比对 `README` 与 `review-pr` 的方面枚举、命令示例、agent 列表，减少长期漂移。
