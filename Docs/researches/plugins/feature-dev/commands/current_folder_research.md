# plugins/feature-dev/commands 目录研究（DIR）

## 场景与职责

`plugins/feature-dev/commands` 是 `feature-dev` 插件的“命令协议层”，当前仅包含一个命令定义文件 `feature-dev.md`，用于把新功能开发过程固化为 7 阶段的人机协作流程（澄清 -> 探索 -> 设计 -> 实施 -> 复核）。

目录清单：
- `plugins/feature-dev/commands/feature-dev.md`

在插件体系中的职责位置：
1. 插件由 marketplace 注册并指向 `./plugins/feature-dev`：`.claude-plugin/marketplace.json:62-70`。
2. 插件元信息由 `.claude-plugin/plugin.json` 提供：`plugins/feature-dev/.claude-plugin/plugin.json:1-9`。
3. Claude Code 组件发现机制会扫描插件 `commands/` 目录中的 `.md`：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-345`。
4. 本目录命令作为 `/feature-dev` 的实际执行协议入口：`plugins/feature-dev/commands/feature-dev.md:1-125`。

职责边界：
- 本目录负责“流程编排与约束”，不包含可执行脚本/源码。
- 具体探索、架构、评审能力由 `agents/` 被调用方承担，而非本目录直接实现。

## 功能点目的

### 1) 把“直接开写”改造成阶段化开发
- 目的：降低需求不清、架构返工、实现偏离现有模式的风险。
- 机制：命令拆分 Discovery / Exploration / Clarification / Architecture / Implementation / Quality Review / Summary 七阶段：`plugins/feature-dev/commands/feature-dev.md:20-124`。

### 2) 强制“先理解再行动”
- 目的：避免在不了解现有代码时改动。
- 机制：要求先理解现有模式，并在 agent 返回后阅读关键文件：`plugins/feature-dev/commands/feature-dev.md:13-15,52`。

### 3) 在关键节点引入用户决策门
- 目的：把歧义消解和方案选择前置，减少错误实现。
- 机制：
  - Phase 3 必须先提问并等待回答：`plugins/feature-dev/commands/feature-dev.md:61-69`。
  - Phase 4 必须询问用户选方案：`plugins/feature-dev/commands/feature-dev.md:80-82`。
  - Phase 5 未获批准不得开始编码：`plugins/feature-dev/commands/feature-dev.md:89-93`。
  - Phase 6 评审后先问修复策略：`plugins/feature-dev/commands/feature-dev.md:108-109`。

### 4) 使用多 agent 并行提升分析覆盖
- 目的：让探索、设计、审查具备多视角并发输入。
- 机制：
  - Phase 2：2-3 个 `code-explorer`：`plugins/feature-dev/commands/feature-dev.md:41-44`。
  - Phase 4：2-3 个 `code-architect`：`plugins/feature-dev/commands/feature-dev.md:78`。
  - Phase 6：3 个 `code-reviewer`：`plugins/feature-dev/commands/feature-dev.md:106`。

### 5) 支持命令参数注入与持续进度跟踪
- 目的：提升入口效率，并避免流程失控。
- 机制：
  - 使用 `$ARGUMENTS` 接收用户命令参数：`plugins/feature-dev/commands/feature-dev.md:24`。
  - `$ARGUMENTS` 是命令系统的动态参数机制：`plugins/plugin-dev/skills/command-development/README.md:138-140`。
  - 要求全程使用 TodoWrite：`plugins/feature-dev/commands/feature-dev.md:16`。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（7 阶段状态机）

1. Discovery
- 输入源：`Initial request: $ARGUMENTS`：`plugins/feature-dev/commands/feature-dev.md:24`。
- 产出：问题定义、约束与用户确认：`plugins/feature-dev/commands/feature-dev.md:26-33`。

2. Codebase Exploration
- 启动多 `code-explorer` 并行，覆盖不同角度并返回关键文件清单：`plugins/feature-dev/commands/feature-dev.md:41-50`。
- 汇总后强制回读这些文件，形成深上下文：`plugins/feature-dev/commands/feature-dev.md:52-53`。

3. Clarifying Questions
- 将边界、异常、兼容、性能等歧义显式列出并等待答复：`plugins/feature-dev/commands/feature-dev.md:64-68`。
- 用户给“你决定”时仍要给建议并获取明确确认：`plugins/feature-dev/commands/feature-dev.md:69`。

4. Architecture Design
- 并行生成不同取向方案（最小改动/清晰架构/务实平衡）：`plugins/feature-dev/commands/feature-dev.md:78-80`。
- 输出对比和推荐，等待用户选型：`plugins/feature-dev/commands/feature-dev.md:80-82`。

5. Implementation
- 显式人审门：无批准不可实施：`plugins/feature-dev/commands/feature-dev.md:89-93`。
- 按选定架构落地并持续更新 todo：`plugins/feature-dev/commands/feature-dev.md:94-97`。

6. Quality Review
- 三路并行评审后合并高风险问题：`plugins/feature-dev/commands/feature-dev.md:106-108`。
- 修不修、何时修由用户决策：`plugins/feature-dev/commands/feature-dev.md:108-109`。

7. Summary
- 关闭 todo，输出变更摘要、决策和后续建议：`plugins/feature-dev/commands/feature-dev.md:118-123`。

### B. 数据结构与会话状态（隐式）

本命令没有显式 JSON schema，但通过阶段约束形成隐式状态槽位：
- `initialRequest`：由 `$ARGUMENTS` 注入。
- `codebaseFindings`：Phase 2 汇总结果与关键文件列表。
- `clarifications`：Phase 3 用户回答集合。
- `architectureOptions` + `selectedApproach`：Phase 4 产物与用户选型。
- `implementationChanges`：Phase 5 代码改动。
- `reviewFindings` + `userDecision`：Phase 6 审查结果与处置策略。
- `finalSummary`：Phase 7 交付信息。

以上是从协议文本推导出的执行态；核心证据位于阶段动作定义：`plugins/feature-dev/commands/feature-dev.md:20-124`。

### C. 协议机制（Prompt Contract）

1. frontmatter 协议
- `description` 提供命令说明，`argument-hint` 提供参数提示：`plugins/feature-dev/commands/feature-dev.md:1-4`。

2. 命令与 agent 的调用契约
- 本命令在文本中定义“Launch X agents”来触发子代理阶段：`plugins/feature-dev/commands/feature-dev.md:41,78,106`。
- 插件命令能力文档明确：命令可通过 Task tool 启动插件 agents：`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:334-356`。

3. 人在回路（Human-in-the-loop）协议
- 多个阶段含“必须等待用户输入/批准”的硬门控，确保关键决策外显：`plugins/feature-dev/commands/feature-dev.md:67,81-82,89,108-109`。

### D. 命令与工具交互特征

- 该命令文件不包含 `!` 内联 shell 命令，也不直接声明外部 API 调用；属于“纯编排提示词”实现。
- 执行动作依赖 Claude Code 运行时工具与 agent 子任务机制。
- 相比 `code-review` 命令在 frontmatter 显式限制 `allowed-tools`（`plugins/code-review/commands/code-review.md:2`），`feature-dev.md` 未做工具白名单约束：`plugins/feature-dev/commands/feature-dev.md:1-4`。

## 关键代码路径与文件引用

### 目标目录（研究对象）
- `plugins/feature-dev/commands/feature-dev.md:1-125`

### 上游调用方与装配配置
- 插件注册入口：`.claude-plugin/marketplace.json:62-70`
- 插件元数据：`plugins/feature-dev/.claude-plugin/plugin.json:1-9`
- 自动发现机制（commands 扫描）：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-345`
- 插件总览说明：`plugins/README.md:20,49-61`
- 插件用户文档入口：`plugins/feature-dev/README.md:19-35`

### 下游被调用方
- 探索代理：`plugins/feature-dev/agents/code-explorer.md:1-51`
- 架构代理：`plugins/feature-dev/agents/code-architect.md:1-34`
- 评审代理：`plugins/feature-dev/agents/code-reviewer.md:1-46`
- 命令内调用点：`plugins/feature-dev/commands/feature-dev.md:41-44,78-79,106-107`

### 文档一致性与行为承诺
- 7 阶段用户说明：`plugins/feature-dev/README.md:35-220`
- agent 触发时机说明：`plugins/feature-dev/README.md:261-307`
- 质量评审输出说明：`plugins/feature-dev/README.md:309-313`

### 测试/脚本上下文
- `plugins/feature-dev/commands` 目录无测试、无脚本文件。
- `plugins/feature-dev` 全目录仅包含 manifest/README/agents/commands 六个 Markdown/JSON 资产，未提供自动化回归脚本。

## 依赖与外部交互

### 1) 内部依赖
1. 依赖 marketplace 注册和 source 路径正确，插件才能被发现：`.claude-plugin/marketplace.json:62-70`。
2. 依赖 `commands/*.md` 自动发现机制，`feature-dev.md` 才能映射为 slash command：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:341-345`。
3. 依赖 `agents/` 中三个同名 agent 可被成功启动：`plugins/feature-dev/commands/feature-dev.md:41,78,106` 与 `plugins/feature-dev/agents/*.md`。
4. 依赖 README 描述与命令协议一致，避免用户预期偏差：`plugins/feature-dev/README.md:35-220`。

### 2) 外部交互
1. 用户交互：至少三处强制等待用户输入（澄清、选型、评审处置）。
2. 版本控制上下文：`code-reviewer` 默认审查 `git diff`，间接把 Git 状态作为输入：`plugins/feature-dev/agents/code-reviewer.md:13`。
3. 网络与系统工具面：三个 agent 都包含 `WebFetch/WebSearch/BashOutput/KillShell` 等工具授权：`plugins/feature-dev/agents/code-explorer.md:4`、`plugins/feature-dev/agents/code-architect.md:4`、`plugins/feature-dev/agents/code-reviewer.md:4`。

### 3) 配置依赖
- 本插件 manifest 只提供元数据，不定义额外 commands/agents 自定义路径，意味着依赖默认目录约定：`plugins/feature-dev/.claude-plugin/plugin.json:1-9`。
- 命令参数展示依赖 `argument-hint` 字段：`plugins/feature-dev/commands/feature-dev.md:3`。

## 风险、边界与改进建议

### 风险

1. 编排是提示词协议，缺乏机器强校验
- 7 阶段顺序和门控依赖模型遵守文本，缺少可执行状态机约束，可能出现阶段跳过或执行不一致。

2. 工具权限边界偏宽
- 命令未设置 `allowed-tools`，下游 agent 默认工具集较大，存在“研究任务却调用高权限工具”的误用面。证据：`plugins/feature-dev/commands/feature-dev.md:1-4` 与 `plugins/feature-dev/agents/*.md:4`。

3. 文档与评审阈值口径不一致
- `code-reviewer` 规定只报 `>=80`：`plugins/feature-dev/agents/code-reviewer.md:33`。
- README 输出分档却写到 50-74：`plugins/feature-dev/README.md:310-312`。

4. 并行 agent 成本与时延风险
- 三个阶段都要求并行多 agent，复杂仓库下 token 成本和响应时间较高；README 也提示可能变慢：`plugins/feature-dev/README.md:371-379`。

5. 回归验证缺口
- 目录无测试/脚本，命令协议改动后缺少自动 smoke-check。

### 边界

1. 本目录只定义 `/feature-dev` 的流程协议，不实现具体业务代码。
2. 本目录不承担 agent 内部策略（探索、架构、评审细节由 `agents/` 定义）。
3. 本目录不实现插件加载器本身，仅消费插件系统目录约定与运行时能力。

### 改进建议

1. 给 `/feature-dev` 增加最小工具白名单
- 在 frontmatter 增加 `allowed-tools`，并按阶段收敛权限，降低误操作面。

2. 引入“阶段产物模板”
- 为 Phase 2/3/4/6 定义固定输出结构（如关键文件清单、问题清单、方案对比表、评审问题表），降低汇总歧义。

3. 统一 README 与 agent 的评审阈值口径
- 二者统一为同一置信度分档，避免用户预期和实际行为不一致。

4. 增加命令协议的回归脚本
- 为 `plugins/feature-dev` 增加轻量检查脚本，至少校验：阶段标题完整、关键 gate 文案存在、引用的 agent 名称均可解析。

5. 增加大任务降级策略
- 当仓库较大或任务较小，允许阶段性减少并行 agent 数量（例如 2/2/2 或 1/1/1），在质量与成本间可配置平衡。
