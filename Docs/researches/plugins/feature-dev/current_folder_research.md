# plugins/feature-dev 目录研究（DIR）

## 场景与职责

`plugins/feature-dev` 是一个面向“新功能开发全过程编排”的声明式插件，核心入口为 `/feature-dev` 命令，目标是把“需求澄清 -> 代码理解 -> 架构决策 -> 实施 -> 复核”固化为可重复执行的 7 阶段流程，减少直接开写导致的返工。

- 插件在仓库级 marketplace 中注册，来源目录即 `./plugins/feature-dev`：`.claude-plugin/marketplace.json:62-70`
- 插件总览将其标记为“7-phase feature development workflow”：`plugins/README.md:20`
- 插件元信息由本目录 manifest 提供：`plugins/feature-dev/.claude-plugin/plugin.json:1-9`

职责分层：
1. `commands/feature-dev.md`：定义阶段化执行协议、关键约束（先澄清后实现、需要用户确认等）。
2. `agents/*.md`：定义 3 类子代理能力边界（探索、架构、评审）及输出合同。
3. `README.md`：对外说明流程、触发方式、最佳实践与故障排查。
4. `.claude-plugin/plugin.json`：插件身份、版本、作者信息。

本目录没有可执行源码（`.ts/.js/.py`）、独立脚本或自动化测试，运行行为主要由 Markdown 提示协议驱动。

## 功能点目的

### 1) 通过 7 阶段降低功能开发不确定性

`README` 明确流程目标是“先理解后实现”，覆盖 Discovery、Exploration、Clarification、Architecture、Implementation、Review、Summary：`plugins/feature-dev/README.md:35-220`。

具体目的：
- 防止需求歧义直接进入编码：`plugins/feature-dev/README.md:85-111`
- 防止不了解现有代码就改动：`plugins/feature-dev/README.md:56-65`
- 在实现前完成多方案比较与用户选型：`plugins/feature-dev/README.md:113-156`

### 2) 用专业化 agent 做并行分工

流程中明确在关键阶段并行调用不同 agent：
- 阶段 2：2-3 个 `code-explorer`：`plugins/feature-dev/README.md:61-70`
- 阶段 4：2-3 个 `code-architect`：`plugins/feature-dev/README.md:118-125`
- 阶段 6：3 个 `code-reviewer`：`plugins/feature-dev/README.md:181-191`

目的不是“多模型炫技”，而是将探索、设计、评审职责隔离，降低单线程思考遗漏。

### 3) 把“停顿点”制度化，避免过早编码

命令协议里有两处硬性停顿：
- Phase 3 先提完澄清问题并等待回答：`plugins/feature-dev/commands/feature-dev.md:57-69`
- Phase 5 没有明确用户批准不得实现：`plugins/feature-dev/commands/feature-dev.md:85-93`

这两处是流程质量的核心控制点。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用链）

1. 发现与加载
- Claude Code 插件系统从 marketplace 读取 `feature-dev` 条目并映射到目录：`.claude-plugin/marketplace.json:62-70`
- 目录结构符合标准插件约定（`.claude-plugin/`、`commands/`、`agents/`）：`plugins/README.md:49-61`

2. 命令触发
- 用户输入 `/feature-dev [可选描述]`，命中 `commands/feature-dev.md`：`plugins/feature-dev/README.md:19-33`
- frontmatter 提供命令描述与参数提示：`plugins/feature-dev/commands/feature-dev.md:1-4`

3. 流程编排
- 主命令按 7 阶段顺序执行：`plugins/feature-dev/commands/feature-dev.md:20-124`
- 在探索/架构/评审阶段并发调用对应 agent 并汇总结果：`plugins/feature-dev/commands/feature-dev.md:36-54,73-82,101-109`

4. 人机决策闭环
- 命令要求在关键节点等待用户输入（澄清答案、架构选择、是否修复评审问题）：`plugins/feature-dev/commands/feature-dev.md:66-69,81-82,108-109`

### B. 数据结构与协议

1. 插件元数据（JSON manifest）
- 字段包括 `name/version/description/author`：`plugins/feature-dev/.claude-plugin/plugin.json:2-8`
- 作用是插件识别、展示与发布元信息。

2. 命令协议（Markdown + YAML frontmatter）
- `description`：命令功能说明
- `argument-hint`：参数提示（可选功能描述）
- `$ARGUMENTS`：将用户输入参数注入 Discovery 阶段上下文：`plugins/feature-dev/commands/feature-dev.md:24`

3. agent 协议（Markdown + YAML frontmatter）
- 公共字段：`name`、`description`、`tools`、`model`、`color`：`plugins/feature-dev/agents/code-explorer.md:1-7`（其余两个文件同构）
- 三类 agent 的“输出合同”分别约束探索报告、架构蓝图、评审结论格式：
  - 探索：入口/调用链/关键文件清单：`plugins/feature-dev/agents/code-explorer.md:41-51`
  - 架构：组件设计/实现地图/构建顺序：`plugins/feature-dev/agents/code-architect.md:24-33`
  - 评审：仅报告高置信问题（>=80）：`plugins/feature-dev/agents/code-reviewer.md:23-44`

### C. 关键命令与行为约束

1. `/feature-dev` 命令
- 强调先读代码再行动：`plugins/feature-dev/commands/feature-dev.md:13-15`
- 强制 TodoWrite 跟踪阶段进度：`plugins/feature-dev/commands/feature-dev.md:16`
- 在实现前需要显式用户批准：`plugins/feature-dev/commands/feature-dev.md:89-93`

2. `code-explorer` agent
- 目标是“从入口追踪到存储”的完整实现理解：`plugins/feature-dev/agents/code-explorer.md:11-25`

3. `code-architect` agent
- 要求做“单一、明确”的架构决策，不鼓励含糊备选：`plugins/feature-dev/agents/code-architect.md:16-18,34`

4. `code-reviewer` agent
- 默认审查 `git diff` 未暂存变更：`plugins/feature-dev/agents/code-reviewer.md:11-14`
- 采用置信度门限过滤低价值问题：`plugins/feature-dev/agents/code-reviewer.md:23-33`

## 关键代码路径与文件引用

### 目录内核心路径

- 插件 manifest：`plugins/feature-dev/.claude-plugin/plugin.json:1-9`
- 主命令协议：`plugins/feature-dev/commands/feature-dev.md:1-125`
- 探索 agent：`plugins/feature-dev/agents/code-explorer.md:1-51`
- 架构 agent：`plugins/feature-dev/agents/code-architect.md:1-34`
- 评审 agent：`plugins/feature-dev/agents/code-reviewer.md:1-46`
- 用户文档：`plugins/feature-dev/README.md:1-412`

### 调用方（上游）

1. marketplace 插件发现：`.claude-plugin/marketplace.json:62-70`
2. 插件总览索引入口：`plugins/README.md:20`
3. 用户通过 `/feature-dev` slash command 触发：`plugins/feature-dev/README.md:19-33`

### 被调用方（下游）

1. 子代理能力：`code-explorer`、`code-architect`、`code-reviewer`（由 Phase 2/4/6 触发）
2. 工具层：三个 agent 均声明可用 `Glob/Grep/LS/Read/NotebookRead/WebFetch/TodoWrite/WebSearch/KillShell/BashOutput`：
   - `plugins/feature-dev/agents/code-explorer.md:4`
   - `plugins/feature-dev/agents/code-architect.md:4`
   - `plugins/feature-dev/agents/code-reviewer.md:4`
3. 代码规范输入：`code-reviewer` 显式依赖 `CLAUDE.md` 规则：`plugins/feature-dev/agents/code-reviewer.md:9,17`

### 配置、测试、脚本、文档覆盖

1. 配置
- 插件级：`.claude-plugin/plugin.json`
- 命令级：`commands/feature-dev.md` frontmatter
- agent 级：`agents/*.md` frontmatter（工具白名单、模型、角色）

2. 测试
- 目录内无 `test/spec/__tests__` 文件；未提供自动化回归机制。

3. 脚本
- 目录内无 `scripts/` 或可执行脚本；流程执行由 prompt 协议驱动。

4. 文档
- `README.md` 详细描述了阶段、触发方式、适用边界、故障排查：`plugins/feature-dev/README.md:35-405`
- 全局插件文档对其能力有摘要声明：`plugins/README.md:20`

## 依赖与外部交互

### 运行时依赖

1. Claude Code 插件系统（命令/agent 自动发现与执行）
2. 子代理运行能力（命令要求并行拉起 2-3/3 个 agent）
3. Git 仓库上下文（评审 agent 默认使用 `git diff`）：`plugins/feature-dev/agents/code-reviewer.md:13`
4. 项目规范文件（`CLAUDE.md`）作为评审准则输入：`plugins/feature-dev/agents/code-reviewer.md:9,17`

### 外部交互面

1. 网络检索能力
- 三个 agent frontmatter 都包含 `WebSearch`、`WebFetch`，意味着在需要时可与外部网页交互。

2. Shell/系统交互
- 三个 agent frontmatter 都声明 `BashOutput` 与 `KillShell`，可读取 shell 输出并管理命令生命周期。

3. 用户交互
- 多阶段显式等待用户确认/回答，是流程正确推进的前置条件：`plugins/feature-dev/commands/feature-dev.md:66-69,81-82,92`

## 风险、边界与改进建议

### 风险与边界

1. 文档与执行标准存在轻微不一致
- README 把 `code-reviewer` 输出分为“Critical 75-100 / Important 50-74”：`plugins/feature-dev/README.md:310-312`
- 但 `code-reviewer` agent 明确“只报告 >=80”：`plugins/feature-dev/agents/code-reviewer.md:33`
- 影响：用户对评审输出阈值的预期可能偏差。

2. 作者元信息命名不一致
- 插件内 manifest 为 `Sid Bidasaria`：`plugins/feature-dev/.claude-plugin/plugin.json:6`
- marketplace 为 `Siddharth Bidasaria`：`.claude-plugin/marketplace.json:66`
- 影响：审计/归属系统若按字符串精确匹配，可能产生重复身份。

3. 流程依赖用户响应，自动化程度受限
- 两个强停顿点（Phase 3、Phase 5）决定了其不适合完全无人值守场景。

4. 无目录内测试与回归脚本
- 该插件是提示协议型实现，变更后缺少自动化守护，容易产生提示词漂移问题。

5. 执行成本边界
- 阶段 2/4/6 都是并行 agent 设计，在大型仓库下可能带来较高延迟与 token 成本（README 已提示“agents may be slow”）：`plugins/feature-dev/README.md:371-379`

### 改进建议

1. 对齐评审阈值文档
- 建议统一 README 与 `code-reviewer` 的阈值语义（统一为 >=80，或调整 agent 文案以匹配文档分档）。

2. 补充轻量回归检查
- 增加脚本验证关键不变量：7 阶段完整性、强停顿点存在、agent 名称与命令中引用一致。

3. 增加最小权限收敛策略
- 当前 agent 工具集合较宽（含 WebSearch/WebFetch/KillShell）；可考虑按阶段提供更窄工具配置，降低误用面。

4. 提供“快速路径”模式
- 在保持 Phase 3/5 安全阈值下，为简单特性增加 3-4 阶段精简流，减少轻量需求的流程成本。

5. 元信息一致性治理
- 将 `.claude-plugin/plugin.json` 与 marketplace 的作者展示名统一，避免后续发布治理歧义。
