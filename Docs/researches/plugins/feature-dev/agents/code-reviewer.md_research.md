# plugins/feature-dev/agents/code-reviewer.md 研究

## 场景与职责

`code-reviewer` 是 `feature-dev` 的交付前质量门代理，主要在 Phase 6 并行执行，用于发现高优先级缺陷并过滤低价值噪声。

流程定位：
- 调用方在 Phase 6 启动 3 个 reviewer，分别关注“简洁性/DRY/优雅性、功能正确性、项目约定与抽象一致性”（`plugins/feature-dev/commands/feature-dev.md:101-109`，`plugins/feature-dev/README.md:176-191`）。
- review 后主线程汇总并询问用户处理策略（立即修复/稍后修复/按现状推进）。

## 功能点目的

1. 高置信筛选
- 该 agent 定义 0-100 置信度体系，并硬性约束“只报告 >=80 问题”（`plugins/feature-dev/agents/code-reviewer.md:23-34`），核心目标是减少误报。

2. 统一评审维度
- 明确三大检查面：项目规范合规、真实功能缺陷、重要代码质量问题（`plugins/feature-dev/agents/code-reviewer.md:15-22`）。

3. 默认审查最近改动
- 默认范围是未暂存 `git diff`，满足“开发完立刻复核”的高频场景（`plugins/feature-dev/agents/code-reviewer.md:11-13`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) frontmatter 与能力声明
- `name: code-reviewer`（`plugins/feature-dev/agents/code-reviewer.md:2`）
- `description` 声明缺陷与规范评审职责（`plugins/feature-dev/agents/code-reviewer.md:3`）
- `tools` 包含代码检索、web、shell、todo 能力（`plugins/feature-dev/agents/code-reviewer.md:4`）
- `model: sonnet`、`color: red`（`plugins/feature-dev/agents/code-reviewer.md:5-6`）

### 2) 审查协议
- 输入范围协议：默认未暂存 diff，可被用户指定文件范围覆盖。
- 分析协议：
  - 规范合规（以 `CLAUDE.md` 或同类规则为基准）
  - Bug 检测（逻辑、空值、竞争、内存、安全、性能）
  - 代码质量（重复、关键错误处理缺失、可访问性、测试覆盖）
- 输出协议：高置信问题分组（Critical/Important），每条含置信度、文件行号、依据与修复建议（`plugins/feature-dev/agents/code-reviewer.md:35-45`）。

### 3) 评分数据结构（逻辑）
- 离散锚点：0/25/50/75/100。
- 报告阈值：`>=80`。
- 该结构使多个 reviewer 可在统一标尺下合并结果，便于主线程排序和决策。

### 4) 配置/测试/脚本/文档上下文
- 配置：agent 自动发现依赖插件目录规范与 `plugin.json`（`plugins/feature-dev/.claude-plugin/plugin.json:1-9`）。
- 测试：未发现 reviewer 规则或阈值的自动化回归测试。
- 脚本：无专用脚本；默认 diff 范围依赖 git 工作区状态。
- 文档：README 对 reviewer 的定位与阈值有说明，但存在与 agent 定义不完全一致的问题（见风险项）。

## 关键代码路径与文件引用

- agent 定义：`plugins/feature-dev/agents/code-reviewer.md:1-46`
- 命令调用（Phase 6）：`plugins/feature-dev/commands/feature-dev.md:101-109`
- README 流程说明（Phase 6）：`plugins/feature-dev/README.md:176-208`
- README agent 小节：`plugins/feature-dev/README.md:295-314`
- 插件注册：`.claude-plugin/marketplace.json:62-70`
- 插件发现机制：`plugins/plugin-dev/skills/plugin-structure/SKILL.md:341-346`
- 命令触发 agent 机制：`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:334-356`

## 依赖与外部交互

1. 内部依赖
- 依赖可读的 git 变更集（默认 unstaged diff）。
- 依赖项目规范文件；该仓库根目录未发现 `CLAUDE.md`，意味着执行时常需退化到“等价规范”来源。

2. 工具与外部交互
- 代码读取与搜索：`Read/Grep/Glob/LS`。
- 外部参考：`WebSearch/WebFetch`（评估安全/框架最佳实践时可用）。
- Shell：`BashOutput/KillShell`（可读取 git 状态、运行命令）。

3. 用户交互
- reviewer 不直接定稿，主线程必须向用户呈现问题并确认处置策略后再进入下一步。

## 风险、边界与改进建议

1. 风险
- 文档阈值不一致：agent 要求仅报告 `>=80`，而 README 输出示例包含 `50-74` 的 Important 区间（`plugins/feature-dev/README.md:309-312`），可能造成使用预期偏差。
- 规范来源不稳定：若 `CLAUDE.md` 缺失或分散在多处，合规判断会漂移。
- 工具权限冗余：review 常见场景不必使用 `KillShell` 与 web 工具，存在超配。

2. 边界
- 该 agent 只“发现并建议修复”，不自动实施修复。
- 默认只覆盖未暂存改动；若风险在已暂存或历史代码，需显式扩大范围。

3. 改进建议
- 对齐阈值定义：统一 README 与 agent 的置信度分档和报告阈值。
- 增加“范围回显”要求：在输出首段固定打印评审范围（unstaged/staged/specific files），避免误解。
- 增加轻量契约测试：校验 `>=80` 阈值、输出字段完整性、严重级别分组存在。
- 将 reviewer 分为“本地快速模式（禁 web）”与“深度模式（可 web）”，减少默认权限面和时延。
