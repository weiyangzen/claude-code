# FILE `plugins/feature-dev/commands/feature-dev.md` 研究文档

## 场景与职责

`plugins/feature-dev/commands/feature-dev.md` 是 `feature-dev` 插件的核心命令协议文件，直接定义 `/feature-dev` 的执行行为。它不是业务代码实现，而是“阶段化编排协议”，把一次功能开发任务拆成可控的人机协作流程（`plugins/feature-dev/commands/feature-dev.md:1-125`）。

在插件体系中的职责位置：

1. 插件注册与装载入口
- marketplace 将 `feature-dev` 指向 `./plugins/feature-dev`（`.claude-plugin/marketplace.json:62-70`）。
- 插件元信息由 `plugins/feature-dev/.claude-plugin/plugin.json` 提供（`plugins/feature-dev/.claude-plugin/plugin.json:2-8`）。

2. 命令发现与路由
- Claude Code 自动扫描 `commands/*.md` 并注册命令（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-345`）。
- `feature-dev.md` frontmatter 暴露命令描述和参数提示，形成 `/feature-dev [Optional feature description]` 的交互入口（`plugins/feature-dev/commands/feature-dev.md:2-4`，`plugins/feature-dev/README.md:19-33`）。

3. 上下游协作边界
- 上游调用方：用户 slash command、插件市场注册、插件文档承诺（`plugins/feature-dev/README.md:17-33`，`plugins/README.md:20`）。
- 下游被调用方：`code-explorer` / `code-architect` / `code-reviewer` 三类 agent（`plugins/feature-dev/commands/feature-dev.md:41,78,106`；`plugins/feature-dev/agents/*.md`）。

职责边界：
- 负责“流程协议、阶段门控、任务编排”。
- 不负责“具体业务实现、自动化测试脚本、运行时代码执行器”。

## 功能点目的

1. 强制“先理解后实现”
- 核心原则要求先理解代码，再做设计和实现（`plugins/feature-dev/commands/feature-dev.md:13-15`）。
- 避免在未知上下文下直接编码，降低返工概率。

2. 将需求不确定性前置消解
- Phase 3 明确要求识别边界条件、异常处理、兼容性与性能问题，并等待用户回答（`plugins/feature-dev/commands/feature-dev.md:63-69`）。
- 目标是让架构设计基于清晰需求，而不是隐式假设。

3. 通过并行 agent 增加分析覆盖度
- Phase 2 并行 `2-3` 个 `code-explorer`（`plugins/feature-dev/commands/feature-dev.md:41-44`）。
- Phase 4 并行 `2-3` 个 `code-architect`（`plugins/feature-dev/commands/feature-dev.md:78-80`）。
- Phase 6 并行 `3` 个 `code-reviewer`（`plugins/feature-dev/commands/feature-dev.md:106-108`）。

4. 通过“人审门”控制高风险决策
- 关键节点必须停下来让用户确认：
  - 澄清后再设计（`plugins/feature-dev/commands/feature-dev.md:67`）
  - 方案比选后用户选型（`plugins/feature-dev/commands/feature-dev.md:80-82`）
  - 未批准不得实现（`plugins/feature-dev/commands/feature-dev.md:89-93`）
  - 评审后由用户决定修复策略（`plugins/feature-dev/commands/feature-dev.md:108-109`）

5. 持续进度可见性
- 全流程要求使用 TodoWrite 跟踪（`plugins/feature-dev/commands/feature-dev.md:16`，`118-119`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) Frontmatter 协议

`feature-dev.md` frontmatter 目前包含：
- `description`：命令用途说明（`plugins/feature-dev/commands/feature-dev.md:2`）。
- `argument-hint`：参数提示（`plugins/feature-dev/commands/feature-dev.md:3`）。

结合命令开发参考：
- `argument-hint` 用于文档化参数和自动补全（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:196-207`）。
- 未声明 `allowed-tools` 时，会继承会话权限（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-67`）。

### 2) 7 阶段流程状态机（文本协议形态）

命令主体实现的是“协议化状态机”：

1. Discovery（`20-33`）
- 输入：`$ARGUMENTS`（`24`），该变量是命令系统的动态参数机制（`plugins/plugin-dev/skills/command-development/README.md:125,138-140`）。
- 输出：问题定义、范围、约束、用户确认。

2. Codebase Exploration（`36-54`）
- fan-out：并行启动多个 `code-explorer`，并要求每个 agent 提供关键文件列表（`41-44`）。
- fan-in：主流程回读 agent 提供文件后再汇总（`52-53`）。

3. Clarifying Questions（`57-70`）
- 显式收敛歧义，强制等待用户输入（`66-67`）。

4. Architecture Design（`73-82`）
- 并行多方案生产，主流程负责比较与推荐，再交用户决策（`78-81`）。

5. Implementation（`85-98`）
- 以“用户批准”为硬门控（`89-93`）。

6. Quality Review（`101-110`）
- 并行评审 + 合并高严重问题 + 用户决策修复策略（`106-109`）。

7. Summary（`113-123`）
- 关闭 todo 并汇总改动与决策（`118-123`）。

### 3) 隐式数据结构（协议上下文）

该命令不定义显式 JSON schema，但存在稳定的隐式上下文结构：

- `initial_request`：来自 `$ARGUMENTS`。
- `exploration_findings`：Phase 2 聚合结果、关键文件列表。
- `clarifications`：Phase 3 的问答集合。
- `architecture_options` / `selected_approach`：Phase 4 方案与选型。
- `implementation_delta`：Phase 5 变更结果。
- `review_findings` / `disposition`：Phase 6 评审结论与处置选择。
- `final_summary`：Phase 7 最终输出。

这些状态槽位由阶段动作文本决定（`plugins/feature-dev/commands/feature-dev.md:20-123`）。

### 4) 命令与 agent 协议耦合

- 命令通过自然语言协议触发 agent（`plugins/feature-dev/commands/feature-dev.md:41,78,106`）。
- 插件命令能力文档说明命令可通过 Task tool 调用插件 agents（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:334-356`）。
- 三个 agent 的职责合同分别覆盖“探索、架构、评审”：
  - `code-explorer` 强调调用链追踪和关键文件清单（`plugins/feature-dev/agents/code-explorer.md:16-25,41-50`）。
  - `code-architect` 强调可执行架构蓝图与实施地图（`plugins/feature-dev/agents/code-architect.md:13-21,24-33`）。
  - `code-reviewer` 强调高置信过滤（仅报告 `>=80`）和 `git diff` 默认审查范围（`plugins/feature-dev/agents/code-reviewer.md:13,23-33`）。

### 5) 命令形态特征

- 该文件没有 `!` 内联 Bash 执行，也没有外部 API 直连语句，属于“纯提示协议编排”。
- 与 `plugins/code-review/commands/code-review.md` 这种显式 `allowed-tools` 命令相比，`feature-dev.md` 当前更依赖运行时默认权限（`plugins/code-review/commands/code-review.md:1-3` vs `plugins/feature-dev/commands/feature-dev.md:1-4`）。

## 关键代码路径与文件引用

### A. 目标对象
- `plugins/feature-dev/commands/feature-dev.md:1-125`

### B. 调用方（上游）
- marketplace 注册：`.claude-plugin/marketplace.json:62-70`
- 插件清单文档：`plugins/README.md:20`
- 插件用户入口文档：`plugins/feature-dev/README.md:19-35`
- 自动发现机制：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:341-345`

### C. 被调用方（下游）
- `plugins/feature-dev/agents/code-explorer.md:1-51`
- `plugins/feature-dev/agents/code-architect.md:1-34`
- `plugins/feature-dev/agents/code-reviewer.md:1-46`
- 命令中触发点：`plugins/feature-dev/commands/feature-dev.md:41,78,106`

### D. 配置路径
- 插件元数据：`plugins/feature-dev/.claude-plugin/plugin.json:1-9`
- 命令 frontmatter：`plugins/feature-dev/commands/feature-dev.md:1-4`
- 命令开发规范参考：
  - `plugins/plugin-dev/skills/command-development/README.md:117-127,150-156,233-239`
  - `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:60-67,196-207`

### E. 测试、脚本、文档
- 目录实况（`find plugins/feature-dev -maxdepth 3 -type f`）：仅 6 个文件，分别是 manifest、README、3 个 agents、1 个 command，无测试文件与脚本文件。
- 文档：
  - 流程说明：`plugins/feature-dev/README.md:35-247`
  - 使用边界与故障处理：`plugins/feature-dev/README.md:341-404`

## 依赖与外部交互

### 1) 内部依赖

1. 插件发现链路依赖
- 需要 marketplace 中 `source` 正确指向目录（`.claude-plugin/marketplace.json:62-70`）。
- 需要命令自动发现扫描 `commands/`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:341-345`）。

2. agent 可用性依赖
- 命令文本显式依赖 3 个 agent 名称及职责（`plugins/feature-dev/commands/feature-dev.md:41,78,106`）。
- agents 文件缺失或语义漂移会直接影响命令执行质量。

3. 文档一致性依赖
- README 对 7 阶段行为的承诺应与命令协议保持一致（`plugins/feature-dev/README.md:56-65,117-125,162-168,180-191`）。

### 2) 外部交互

1. 用户交互
- 至少三次硬性交互等待：澄清答案、架构选型、评审处置（`plugins/feature-dev/commands/feature-dev.md:67,81,108`）。

2. 代码仓库交互（间接）
- 虽然命令本身不直接执行 Git，但下游 `code-reviewer` 默认审查 `git diff`（`plugins/feature-dev/agents/code-reviewer.md:13`），因此依赖 Git 工作区状态。

3. 工具面与潜在外部资源
- 三个 agent 统一声明了 `WebFetch/WebSearch/BashOutput/KillShell` 等工具能力（`plugins/feature-dev/agents/code-explorer.md:4`，`plugins/feature-dev/agents/code-architect.md:4`，`plugins/feature-dev/agents/code-reviewer.md:4`），具备外部查询和系统命令交互能力。

## 风险、边界与改进建议

### 风险

1. 协议执行主要靠提示词约束，缺少机器可验证门控
- 阶段顺序和“必须等待”语义是文本约束，不是硬状态机，存在被跳过的可能。

2. 权限收敛不足
- `feature-dev.md` 未使用 `allowed-tools`（`plugins/feature-dev/commands/feature-dev.md:1-4`），而命令开发规范明确推荐最小化工具权限（`plugins/plugin-dev/skills/command-development/README.md:237-239`，`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:109-127`）。

3. 评审阈值口径不一致
- README 输出示例含 50-74 分档（`plugins/feature-dev/README.md:309-312`），但 `code-reviewer` 明确仅报告 `>=80`（`plugins/feature-dev/agents/code-reviewer.md:33`），会造成用户预期偏差。

4. 多轮并行 agent 的成本/时延风险
- Phase 2/4/6 均要求并行 agent（`plugins/feature-dev/commands/feature-dev.md:41,78,106`），README 也提示大型仓库会较慢（`plugins/feature-dev/README.md:371-379`）。

5. 元数据一致性风险
- 作者名在 marketplace 与插件 manifest 存在写法差异（`Siddharth Bidasaria` vs `Sid Bidasaria`，对比 `.claude-plugin/marketplace.json:66` 与 `plugins/feature-dev/.claude-plugin/plugin.json:6`）。

### 边界

1. 本文件只定义流程，不实现业务功能。
2. 本文件不负责 agent 内部算法，只负责“何时调谁、输出要什么”。
3. 本插件目录当前无专用测试与脚本，因此对协议回归的保障主要靠人工执行与文档一致性检查。

### 改进建议

1. 为 `/feature-dev` 增加最小 `allowed-tools`
- 即使该命令偏编排，也应声明最小必要工具集合，降低误调用高权限工具的风险。

2. 为关键阶段定义结构化产物模板
- 例如强制 Phase 3 输出“问题编号/原因/影响面”，Phase 4 输出“方案-代价-风险矩阵”，减少执行偏差。

3. 统一 README 与 agent 的置信度语义
- 建议统一为“仅报告 `>=80`”，或同步调整 agent 文案，避免用户误判 review 覆盖范围。

4. 增加轻量自动校验脚本
- 校验项可包括：7 个阶段标题存在、关键 gate 文案存在、引用 agent 名称可解析、README 与命令关键约束一致。

5. 引入并行度降级策略
- 提供“轻量模式”（如 1/1/1 或 2/1/2）以适配小任务和大仓库，平衡质量与成本。
