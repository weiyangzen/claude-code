# plugins/feature-dev/agents/code-architect.md 研究

## 场景与职责

`code-architect` 是 `feature-dev` 插件在“架构设计阶段（Phase 4）”的专用子代理定义文件。它不直接改业务代码，而是把前置探索结果收敛成可落地的实施蓝图，供主线程和用户做选型决策。

职责边界来自两层约束：
- agent 自身系统提示词要求“做明确单一架构决策并给出完整实施图”（`plugins/feature-dev/agents/code-architect.md:16-34`）。
- 调用方 `/feature-dev` 命令要求在 Phase 4 并行拉起 2-3 个 `code-architect`，分别从“最小改动/清晰架构/务实平衡”角度设计方案，再由主线程汇总推荐并询问用户选择（`plugins/feature-dev/commands/feature-dev.md:73-82`，`plugins/feature-dev/README.md:113-156`）。

## 功能点目的

1. 将“理解代码”转换为“实施蓝图”
- 在 Phase 2/3 完成现状理解和需求澄清后，`code-architect` 负责定义目标架构与落地路径，避免直接编码导致反复返工。

2. 输出可执行而非泛泛建议
- 输出合同要求文件级改动地图、组件责任、数据流、分阶段构建序列与关键非功能项（测试、性能、安全、错误处理），目标是让实现阶段可按图施工（`plugins/feature-dev/agents/code-architect.md:24-33`）。

3. 支持并行方案对比
- 单个 `code-architect` 强调“选择并承诺一种方案”（`plugins/feature-dev/agents/code-architect.md:17,34`），而工作流层面通过并行多个实例实现方案空间覆盖。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 文件结构与运行时元数据
该文件采用“frontmatter + 提示词正文”协议：
- `name: code-architect`：agent 标识（`plugins/feature-dev/agents/code-architect.md:2`）
- `description`：能力摘要（`plugins/feature-dev/agents/code-architect.md:3`）
- `tools`：允许调用的工具集合（`plugins/feature-dev/agents/code-architect.md:4`）
- `model: sonnet`、`color: green`：模型与显示属性（`plugins/feature-dev/agents/code-architect.md:5-6`）

### 2) 调用协议与主流程衔接
- `feature-dev` 命令在 Phase 4 明确要求并行调用 `code-architect`（`plugins/feature-dev/commands/feature-dev.md:78`）。
- 插件机制层面，命令触发 agent 由 Task 工具承载（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:332-356`）。
- agent 文件发现依赖插件自动发现机制：启用插件后扫描 `agents/*.md`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:341-346`）。

### 3) 输出数据合同（逻辑数据结构）
虽然不是 JSON schema，但正文定义了稳定输出槽位，可视为“半结构化结果协议”：
- Patterns & Conventions Found
- Architecture Decision
- Component Design
- Implementation Map
- Data Flow
- Build Sequence
- Critical Details

这些槽位直接对应后续 Phase 5 的实施输入与用户决策输入。

### 4) 配置/测试/脚本/文档上下文
- 配置：插件最小 manifest 在 `plugins/feature-dev/.claude-plugin/plugin.json`，通过默认目录自动发现 agent（`plugins/feature-dev/.claude-plugin/plugin.json:1-9`）。
- 测试：仓库中无该 agent 的专门自动化测试或契约测试文件。
- 脚本：该 agent 无独立脚本，完全由 Claude Code 插件运行时加载执行。
- 文档：`plugins/feature-dev/README.md` 对该 agent 的触发时机与期望输出有用户文档说明（`plugins/feature-dev/README.md:273-294`）。

## 关键代码路径与文件引用

- agent 定义：`plugins/feature-dev/agents/code-architect.md:1-34`
- 调用方命令协议：`plugins/feature-dev/commands/feature-dev.md:73-82`
- 插件用户文档（Phase 4 与 agent 说明）：`plugins/feature-dev/README.md:113-156`、`plugins/feature-dev/README.md:273-294`
- 插件注册与发现：`.claude-plugin/marketplace.json:62-70`、`plugins/feature-dev/.claude-plugin/plugin.json:1-9`
- 运行机制参考：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-356`、`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:332-356`

## 依赖与外部交互

1. 内部依赖
- 依赖上游 Phase 2/3 的探索结果与澄清结果，否则架构输入不完整。
- 依赖主线程在多方案间做综合判断并与用户确认。

2. 工具依赖
- `Read/Grep/Glob/LS` 支撑代码模式提取。
- `TodoWrite` 允许与主流程任务跟踪对齐。
- `WebSearch/WebFetch/BashOutput/KillShell` 提供外部信息与 shell 能力（`plugins/feature-dev/agents/code-architect.md:4`）。

3. 外部交互
- 通过 `/feature-dev` 与用户发生两次关键交互：方案推荐、方案选择（`plugins/feature-dev/commands/feature-dev.md:80-82`）。

## 风险、边界与改进建议

1. 风险
- 权限面偏宽：架构任务通常以本仓分析为主，但默认开放网络与 shell 管理工具，存在超额能力面。
- 结果一致性风险：多个并行 architect 各自“单方案承诺”，可能在术语与评估维度上不一致，增加汇总难度。

2. 边界
- `code-architect` 只负责“设计蓝图”，不负责最终代码落地与缺陷闭环。
- 最终路线选择由主线程与用户共同决策，不由该 agent 单独决定。

3. 改进建议
- 增加“输出骨架模板”约束：对七个输出槽位规定固定顺序与最小字段，便于并行结果横向比较。
- 对 Phase 4 增加“比较矩阵”要求（复杂度/改动面/回滚成本/测试成本），降低主线程汇总负担。
- 引入轻量契约测试（例如 grep 校验关键段落标题存在），防止后续维护时输出协议漂移。
