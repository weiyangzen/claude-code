# FILE `plugins/feature-dev/README.md` 研究文档

## 场景与职责

`plugins/feature-dev/README.md` 是 `feature-dev` 插件的人类使用协议文档，负责把“如何触发、何时停下来问用户、何时并行用子代理、何时才允许实现”的流程讲清楚。

1. 在仓库文档链路中的位置（调用方）
- 根文档把插件入口指向 `plugins/README.md`（`README.md:48-50`）。
- 插件总览把 `feature-dev` 暴露为 7 阶段工作流（`plugins/README.md:20`）。
- marketplace 把 `feature-dev` 名称映射到目录 `./plugins/feature-dev`（`.claude-plugin/marketplace.json:62-70`）。

2. 对下游执行层的职责（被调用方）
- README 声明 `/feature-dev` 作为用户入口（`plugins/feature-dev/README.md:19-33`）。
- README 的 7 阶段说明对应实际执行协议 `commands/feature-dev.md`（`plugins/feature-dev/commands/feature-dev.md:20-124`）。
- README 的 3 个 agent 说明对应 `agents/code-explorer.md`、`agents/code-architect.md`、`agents/code-reviewer.md`（`plugins/feature-dev/README.md:249-314`）。

3. 文档层角色边界
- 该文件本身不执行代码，而是用户与命令协议之间的“行为合同”。
- 真正可执行逻辑在命令文件与 agent 文件中；README 负责对齐用户预期、降低误用概率。

## 功能点目的

1. 提供统一入口与交互预期
- 通过 `/feature-dev [可选描述]` 触发（`plugins/feature-dev/README.md:23-33`）。
- 目标是避免“直接开写”，改为先理解代码再实现（`plugins/feature-dev/README.md:7,11-17`）。

2. 固化 7 阶段研发流程
- Discovery：明确问题与约束（`plugins/feature-dev/README.md:37-46`）。
- Codebase Exploration：并行探索现有实现（`plugins/feature-dev/README.md:56-70`）。
- Clarifying Questions：补齐歧义并等待回答（`plugins/feature-dev/README.md:85-99`）。
- Architecture Design：多方案对比并让用户选型（`plugins/feature-dev/README.md:113-126`）。
- Implementation：必须获批后才编码（`plugins/feature-dev/README.md:158-174`）。
- Quality Review：并行评审并由用户决定处置策略（`plugins/feature-dev/README.md:176-191`）。
- Summary：输出结果、决策与后续建议（`plugins/feature-dev/README.md:210-220`）。

3. 定义子代理能力分工
- `code-explorer`：追调用链与架构层次（`plugins/feature-dev/README.md:251-272`）。
- `code-architect`：做架构蓝图与实施地图（`plugins/feature-dev/README.md:273-294`）。
- `code-reviewer`：按高置信问题输出修复建议（`plugins/feature-dev/README.md:295-314`）。

4. 明确适用边界
- 适合复杂新功能，不适合单行修复/紧急 hotfix（`plugins/feature-dev/README.md:349-362`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 插件到命令的加载协议

1. marketplace 注册：`feature-dev` -> `./plugins/feature-dev`（`.claude-plugin/marketplace.json:62-70`）。
2. 运行时按插件约定自动发现组件：读取 `.claude-plugin/plugin.json`，扫描 `commands/` 与 `agents/`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-346`）。
3. 本插件 manifest 提供元数据（`plugins/feature-dev/.claude-plugin/plugin.json:1-9`）。
4. `/feature-dev` 命令来自 `commands/feature-dev.md` frontmatter 与正文协议（`plugins/feature-dev/commands/feature-dev.md:1-4`）。

### 2) 命令协议（7 阶段状态机）

`commands/feature-dev.md` 等价于一个“带人工闸门的阶段状态机”：

1. 输入注入
- `Initial request: $ARGUMENTS` 把 slash command 参数注入流程（`plugins/feature-dev/commands/feature-dev.md:24`）。

2. 并行探索
- Phase 2 明确要求并行启动 2-3 个 `code-explorer`，并要求返回 5-10 个关键文件（`plugins/feature-dev/commands/feature-dev.md:41-45`）。
- 主线程必须回读这些文件后再总结（`plugins/feature-dev/commands/feature-dev.md:52-53`）。

3. 人工确认闸门
- Phase 3：必须提问并等待用户回答（`plugins/feature-dev/commands/feature-dev.md:66-69`）。
- Phase 4：给推荐但仍需用户选方案（`plugins/feature-dev/commands/feature-dev.md:80-82`）。
- Phase 5：未获批准不得实现（`plugins/feature-dev/commands/feature-dev.md:89-93`）。
- Phase 6：评审后先问修不修（`plugins/feature-dev/commands/feature-dev.md:108-109`）。

4. 全程任务跟踪
- 命令要求全程使用 TodoWrite（`plugins/feature-dev/commands/feature-dev.md:16`），Phase 7 要求收口（`plugins/feature-dev/commands/feature-dev.md:118-123`）。

### 3) 子代理协议与数据结构

1. agent 声明结构
- 三个 agent 使用同构 frontmatter：`name/description/tools/model/color`（例如 `plugins/feature-dev/agents/code-explorer.md:1-7`）。

2. `code-explorer` 输出合同
- 强制输出入口点、调用流、依赖、关键文件列表（`plugins/feature-dev/agents/code-explorer.md:41-50`）。

3. `code-architect` 输出合同
- 要求给出单一明确架构决策、组件设计、实施地图、构建顺序（`plugins/feature-dev/agents/code-architect.md:24-34`）。

4. `code-reviewer` 评审合同
- 默认审查 `git diff`（`plugins/feature-dev/agents/code-reviewer.md:13`）。
- 使用 0-100 置信度并要求仅报告 `>=80`（`plugins/feature-dev/agents/code-reviewer.md:23-34`）。
- 输出需含文件行号、依据与修复建议（`plugins/feature-dev/agents/code-reviewer.md:37-43`）。

### 4) 命令与文档的一致性

README 中阶段动作与命令协议基本一致：
- Phase 2/4/6 的并行 agent 数量与职责对齐（`plugins/feature-dev/README.md:61-63,118-121,181-184` 对应 `plugins/feature-dev/commands/feature-dev.md:41,78,106`）。
- “等待用户输入/批准”的关键门控也对齐（`plugins/feature-dev/README.md:98,125,163,187` 对应 `plugins/feature-dev/commands/feature-dev.md:67,81,89,108`）。

## 关键代码路径与文件引用

1. 目标对象
- `plugins/feature-dev/README.md`

2. 直接执行协议
- `plugins/feature-dev/commands/feature-dev.md`

3. 被调用子代理
- `plugins/feature-dev/agents/code-explorer.md`
- `plugins/feature-dev/agents/code-architect.md`
- `plugins/feature-dev/agents/code-reviewer.md`

4. 插件配置与注册
- `plugins/feature-dev/.claude-plugin/plugin.json`
- `.claude-plugin/marketplace.json`

5. 上游索引文档
- `README.md`
- `plugins/README.md`

6. 运行机制参考（组件自动发现）
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md`

7. 约束对照样例（工具白名单）
- `plugins/code-review/commands/code-review.md`
- `plugins/plugin-dev/skills/command-development/README.md`

8. 测试/脚本上下文
- `plugins/feature-dev` 目录仅含 manifest + README + commands + agents，无专用测试与脚本文件（目录扫描结果：`plugins/feature-dev/*`）。

## 依赖与外部交互

1. Claude Code 插件运行时依赖
- 依赖插件自动发现机制加载命令与 agent（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:341-346`）。

2. Git 工作区依赖
- `code-reviewer` 默认依赖 `git diff` 作为输入（`plugins/feature-dev/agents/code-reviewer.md:13`）。
- README 也将 Git repository 作为使用前提（`plugins/feature-dev/README.md:365-367`）。

3. 项目规范依赖
- `code-reviewer` 以 `CLAUDE.md` 或等价规范作为主要合规依据（`plugins/feature-dev/agents/code-reviewer.md:9,17`）。
- 当前仓库未检出 `CLAUDE.md` 文件（`rg --files | rg 'CLAUDE\.md$'` 无结果），因此该插件在本仓库更依赖“等价规范”或 agent 自主判断。

4. 外部工具与网络交互面
- 三个 agent 工具集合包含 `WebFetch/WebSearch/BashOutput/KillShell`（`plugins/feature-dev/agents/code-explorer.md:4`，其余两个同构）。
- 这意味着流程并非纯本地静态分析，可产生网络访问与 shell 侧效应。

5. 文档与外部站点交互
- README 引导到 Claude Code 语境和命令实践，不直接调用外部 API，但插件总入口链接到官方文档站点（`plugins/README.md:9,45`）。

## 风险、边界与改进建议

1. 风险：评审阈值描述不一致
- README 说输出分档为 `75-100` 与 `50-74`（`plugins/feature-dev/README.md:310-312`）。
- `code-reviewer` 明确“仅报告 `>=80`”（`plugins/feature-dev/agents/code-reviewer.md:33`）。
- 建议：统一阈值口径，避免用户预期“会看到 50-79 问题”但实际看不到。

2. 风险：命令缺少 `allowed-tools` 限制
- `/feature-dev` frontmatter 仅有 `description` 与 `argument-hint`（`plugins/feature-dev/commands/feature-dev.md:1-4`）。
- 对照 `code-review` 已使用显式工具白名单（`plugins/code-review/commands/code-review.md:1-3`）。
- 建议：为 `/feature-dev` 增加最小必要工具白名单，降低误调用高权限工具的风险。

3. 风险：多并行 agent 带来的时延与成本
- Phase 2/4/6 都要求并行多 agent（`plugins/feature-dev/commands/feature-dev.md:41,78,106`）。
- README 已提示大型仓库会变慢（`plugins/feature-dev/README.md:371-379`）。
- 建议：加入“快速模式/深度模式”开关，根据仓库规模和任务紧急度调节 agent 数量。

4. 风险：审查规范输入可能缺失
- 流程假设有 `CLAUDE.md` 规则来源（`plugins/feature-dev/agents/code-reviewer.md:17`）。
- 在无 `CLAUDE.md` 的仓库，评审标准易漂移。
- 建议：README 增加“无 CLAUDE.md 时使用何种替代规范”的明确说明（例如 `README`/lint 配置/语言风格指南）。

5. 风险：元数据作者命名不一致
- 插件 manifest 写 `Sid Bidasaria`（`plugins/feature-dev/.claude-plugin/plugin.json:6`），marketplace 写 `Siddharth Bidasaria`（`.claude-plugin/marketplace.json:66`）。
- 建议：统一作者命名，避免审计或归属自动化时出现重复实体。

6. 测试与可验证性边界
- 该插件是“提示协议+文档编排”资产，缺少专门自动化测试脚本来校验“README/command/agent 一致性”。
- 建议新增轻量校验脚本：
  - 检查 README 声明的阶段与 `commands/feature-dev.md` 是否一致；
  - 检查 README 声明的 agent 名称是否都存在于 `agents/`；
  - 检查阈值、并行数量等关键参数是否一致。
