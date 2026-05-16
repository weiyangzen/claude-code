# plugins/feature-dev/agents/code-explorer.md 研究

## 场景与职责

`code-explorer` 是 `feature-dev` 工作流的前置分析代理，主要服务 Phase 2（Codebase Exploration）。其职责是把“要做什么功能”先映射到“现有系统如何工作”，形成可追溯的事实基线。

在调用链中：
- 上游：`/feature-dev` 在 Phase 2 并行启动 2-3 个 `code-explorer`，每个聚焦不同角度（相似特性、架构层、UI/测试扩展点等）（`plugins/feature-dev/commands/feature-dev.md:36-54`，`plugins/feature-dev/README.md:56-83`）。
- 下游：主线程读取 explorer 返回的关键文件清单，继续 Phase 3 需求澄清与 Phase 4 架构设计（`plugins/feature-dev/commands/feature-dev.md:52-53`）。

## 功能点目的

1. 建立“入口到落盘”的完整认知链
- 该 agent 的核心使命是从入口点追踪到数据存储，穿透所有抽象层（`plugins/feature-dev/agents/code-explorer.md:11-13`）。

2. 降低架构与实现阶段的不确定性
- 通过“调用链 + 数据变换 + 依赖关系 + 状态副作用”输出，减少后续阶段对既有系统的误判（`plugins/feature-dev/agents/code-explorer.md:21-37`）。

3. 产出“必读文件列表”作为团队对齐资产
- 输出合同强制给出最关键文件列表（`plugins/feature-dev/agents/code-explorer.md:49`），为后续讨论提供共同语境。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) frontmatter 协议
- `name: code-explorer`（`plugins/feature-dev/agents/code-explorer.md:2`）
- `description` 定义分析深度和用途（`plugins/feature-dev/agents/code-explorer.md:3`）
- `tools` 提供读仓、搜索、外部检索与 shell 能力（`plugins/feature-dev/agents/code-explorer.md:4`）
- `model: sonnet`、`color: yellow`（`plugins/feature-dev/agents/code-explorer.md:5-6`）

### 2) 分析流程协议
正文定义四步流程：
1. Feature Discovery（入口、边界、配置）
2. Code Flow Tracing（调用链、数据变换、依赖、副作用）
3. Architecture Analysis（分层、模式、接口、横切关注点）
4. Implementation Details（算法、错误处理、性能、技术债）

对应位置：`plugins/feature-dev/agents/code-explorer.md:16-37`。

### 3) 与命令层的契合点
- 命令层明确要求 explorer 给出 5-10 个关键文件，且主线程必须读取这些文件（`plugins/feature-dev/commands/feature-dev.md:44,52`）。
- 这与 explorer 的输出合同“essential files list”完全一致（`plugins/feature-dev/agents/code-explorer.md:49`）。

### 4) 配置/测试/脚本/文档上下文
- 配置：`plugins/feature-dev/.claude-plugin/plugin.json` 采用最小配置，依赖默认目录发现 agents。
- 测试：未发现该 agent 的自动化触发测试或输出契约测试。
- 脚本：该 agent 不包含本地脚本，执行由 Claude Code 运行时托管。
- 文档：README 对其目的、触发时机、输出项有显式说明（`plugins/feature-dev/README.md:251-272`）。

## 关键代码路径与文件引用

- agent 文件：`plugins/feature-dev/agents/code-explorer.md:1-51`
- 命令调用：`plugins/feature-dev/commands/feature-dev.md:36-54`
- 7 阶段文档（Phase 2）：`plugins/feature-dev/README.md:56-83`
- agent 文档说明：`plugins/feature-dev/README.md:251-272`
- 插件清单注册：`.claude-plugin/marketplace.json:62-70`
- 自动发现机制：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-346`
- Task 调用机制参考：`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:332-356`

## 依赖与外部交互

1. 内部依赖
- 强依赖仓库结构可检索性；若目标项目分层混乱或命名不一致，追踪成本显著上升。
- 强依赖主线程后续“读关键文件并消化”，否则探索结果无法转化为设计输入。

2. 工具与外部交互
- 本地代码交互：`Read/Grep/Glob/LS`。
- 外部信息交互：`WebSearch/WebFetch` 可用于框架/协议补充。
- Shell 交互：`BashOutput/KillShell`。

3. 协作交互
- 该 agent 的输出主要面向主线程，不直接决定实现；其价值体现在为后续澄清问题与架构决策提供证据链。

## 风险、边界与改进建议

1. 风险
- 输出深度波动：若提示过宽，容易产出“面广但浅”的扫描结果，影响后续可用性。
- 文件清单质量风险：若关键文件遗漏，主线程会在后续阶段基于不完整上下文决策。
- 权限过宽风险：探索任务通常无需 `KillShell`，可考虑收敛。

2. 边界
- `code-explorer` 只解释“现状”，不负责提出最终架构结论。
- 不承担实现与修复责任，结果是“输入材料”而非“交付代码”。

3. 改进建议
- 在输出合同中新增“Top 5 必读 + Why”字段，提升文件清单可执行性。
- 要求每条执行流至少包含一个入口函数和一个终点持久化/输出节点，避免空泛流程图。
- 为 Phase 2 增加“覆盖率自检项”（是否覆盖 API/UI/配置/存储四类入口），减少遗漏。
