# plugins/feature-dev/agents 目录研究（DIR）

## 场景与职责

`plugins/feature-dev/agents` 是 `feature-dev` 插件的“子代理能力层”，承接 `/feature-dev` 主命令在不同阶段的并行分析与审查任务。目录内包含 3 个 agent 定义文件：

- `code-explorer.md`：代码库特性探索与调用链追踪（`plugins/feature-dev/agents/code-explorer.md:1-51`）
- `code-architect.md`：架构方案设计与实施蓝图输出（`plugins/feature-dev/agents/code-architect.md:1-34`）
- `code-reviewer.md`：实现后高置信质量审查（`plugins/feature-dev/agents/code-reviewer.md:1-46`）

在流程编排中，它们由 `/feature-dev` 命令按阶段拉起：
- Phase 2：2-3 个 `code-explorer` 并行（`plugins/feature-dev/commands/feature-dev.md:36-54`）
- Phase 4：2-3 个 `code-architect` 并行（`plugins/feature-dev/commands/feature-dev.md:73-82`）
- Phase 6：3 个 `code-reviewer` 并行（`plugins/feature-dev/commands/feature-dev.md:101-109`）

插件外层上下文：
- marketplace 注册 `feature-dev` 插件并指向 `./plugins/feature-dev`（`.claude-plugin/marketplace.json:62-70`）
- Claude Code 对插件 `agents/` 下 `.md` 文件进行自动发现（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-346`）
- 插件总览文档对该目录能力有摘要说明（`plugins/README.md:20`）

结论：该目录不是“实现业务逻辑”的源码目录，而是工作流中的“专家角色协议目录”，通过 frontmatter + 提示词定义 agent 职责边界、工具权限与输出合同。

## 功能点目的

### 1) `code-explorer`：建立改动前的事实基线

目的：在动手设计/编码前，系统性回答“现有实现怎么工作”。其提示词要求从入口追到存储，覆盖调用链、数据流、层次结构、边界与副作用（`plugins/feature-dev/agents/code-explorer.md:11-38`），并强制输出关键文件清单（`plugins/feature-dev/agents/code-explorer.md:49`）。

这直接服务于主命令“先理解再行动”的原则（`plugins/feature-dev/commands/feature-dev.md:13-15`）。

### 2) `code-architect`：把调研结果收敛为可执行方案

目的：基于已有模式做“单一明确决策”，提供落地级蓝图（`plugins/feature-dev/agents/code-architect.md:16-21`），并输出组件设计、实现地图、数据流与分阶段构建顺序（`plugins/feature-dev/agents/code-architect.md:24-33`）。

它与 Phase 4 配合，支持多视角并行产出后由主线程做取舍（`plugins/feature-dev/commands/feature-dev.md:78-82`）。

### 3) `code-reviewer`：在交付前做高信噪比缺陷过滤

目的：在实现后聚焦“真实高风险问题”，减少噪音。其定义了 0-100 置信度并明确仅报告 `>=80` 问题（`plugins/feature-dev/agents/code-reviewer.md:23-34`），默认审查 `git diff` 未暂存改动（`plugins/feature-dev/agents/code-reviewer.md:13`）。

它与 Phase 6 的三路并行评审配合，形成“质量门”（`plugins/feature-dev/commands/feature-dev.md:106-109`）。

### 4) 三类 agent 的协同目标

以阶段职责分离实现“探索 -> 设计 -> 复核”的串联闭环，避免单线程直接编码：
- 先由 explorer 拉齐上下文
- 再由 architect 形成实施蓝图
- 最后 reviewer 过滤高优先级缺陷

该协同策略在插件 README 的 7 阶段说明中与命令协议保持一致（`plugins/feature-dev/README.md:56-66,113-126,176-191`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用方 -> 本目录 -> 被调用方）

1. 插件加载：
- marketplace 条目指向 `./plugins/feature-dev`（`.claude-plugin/marketplace.json:69`）
- Claude Code 读取插件 manifest，随后扫描 `agents/`（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-346`）

2. 命令触发：
- 用户执行 `/feature-dev`，进入 7 阶段协议（`plugins/feature-dev/README.md:19-35`）
- 在 Phase 2/4/6 明确要求启动对应 agent（`plugins/feature-dev/commands/feature-dev.md:41,78,106`）

3. 执行机制：
- 按插件命令能力说明，命令通过 Task 工具触发插件 agent（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:334-356`）
- agent 返回结构化分析结果，主线程汇总并继续流程（`plugins/feature-dev/commands/feature-dev.md:52-53,79-81,107-109`）

### B. 数据结构（agent frontmatter + prompt contract）

每个 agent 文件都采用“YAML frontmatter + 正文提示词”双层结构：

1. frontmatter 元数据（共性）
- `name`: agent 标识（`code-explorer`/`code-architect`/`code-reviewer`）
- `description`: 触发意图与职责摘要
- `tools`: 允许调用的工具集合
- `model: sonnet`
- `color`: yellow/green/red 作为显示分组色

见：
- `plugins/feature-dev/agents/code-explorer.md:1-7`
- `plugins/feature-dev/agents/code-architect.md:1-7`
- `plugins/feature-dev/agents/code-reviewer.md:1-7`

2. 正文提示协议（差异化输出合同）
- `code-explorer`：强调执行流追踪、架构层次、关键文件列表（`plugins/feature-dev/agents/code-explorer.md:14-51`）
- `code-architect`：强调模式提取、架构决策、文件级实施地图（`plugins/feature-dev/agents/code-architect.md:13-34`）
- `code-reviewer`：强调 CLAUDE.md 规范对齐、缺陷识别、置信度阈值过滤（`plugins/feature-dev/agents/code-reviewer.md:17-46`）

### C. 协议与命令约束

1. 协议耦合点（与 `commands/feature-dev.md`）
- 主命令要求 agent 返回“关键文件列表”，并在继续前阅读这些文件（`plugins/feature-dev/commands/feature-dev.md:14,52`）
- 该要求与 `code-explorer` 输出合同完全对齐（`plugins/feature-dev/agents/code-explorer.md:49`）

2. 人机协作停顿点（间接受 agents 输出影响）
- Phase 3 必须先完成问题澄清并等待用户回答（`plugins/feature-dev/commands/feature-dev.md:57-69`）
- Phase 5 必须用户批准后实施（`plugins/feature-dev/commands/feature-dev.md:89-93`）
- Phase 6 评审结果要先询问用户处置策略（`plugins/feature-dev/commands/feature-dev.md:108-109`）

3. 命名与发现
- 插件内 `agents/*.md` 自动发现（`plugins/plugin-dev/skills/agent-development/SKILL.md:299`）
- agent 名称存在插件上下文命名规则（`plugins/plugin-dev/skills/agent-development/SKILL.md:303-306`）

### D. 配置、测试、脚本、文档上下文

1. 配置
- 目录内配置即 3 个 agent 文件 frontmatter
- 上层配置来自 `plugins/feature-dev/.claude-plugin/plugin.json` 与仓库 marketplace（`plugins/feature-dev/.claude-plugin/plugin.json:1-9`，`.claude-plugin/marketplace.json:62-70`）

2. 测试
- 本目录无自动化测试文件（无 `test`/`spec`/`__tests__` 目录）
- `code-reviewer` 仅在提示词层要求关注测试覆盖（`plugins/feature-dev/agents/code-reviewer.md:21`），但未提供 agent 行为回归测试脚本

3. 脚本
- 本目录无可执行脚本；执行行为依赖 Claude Code 的 agent/Task 运行时机制（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:334-356`）

4. 文档
- `plugins/feature-dev/README.md` 对这 3 个 agent 的触发时机和预期输出给出用户级说明（`plugins/feature-dev/README.md:249-314`）

## 关键代码路径与文件引用

### 目标目录核心文件

- `plugins/feature-dev/agents/code-explorer.md`
- `plugins/feature-dev/agents/code-architect.md`
- `plugins/feature-dev/agents/code-reviewer.md`

### 调用方（上游）

1. `plugins/feature-dev/commands/feature-dev.md:41-44,78-79,106-107`
2. `plugins/feature-dev/README.md:61-63,118-121,181-184`
3. `plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:334-356`
4. `plugins/plugin-dev/skills/plugin-structure/SKILL.md:343-346`

### 被调用方（下游工具与环境）

三者 `tools` 声明一致，均可调用：
- `Glob`, `Grep`, `LS`, `Read`, `NotebookRead`, `WebFetch`, `TodoWrite`, `WebSearch`, `KillShell`, `BashOutput`

见：
- `plugins/feature-dev/agents/code-explorer.md:4`
- `plugins/feature-dev/agents/code-architect.md:4`
- `plugins/feature-dev/agents/code-reviewer.md:4`

### 关联配置与文档

- 插件 metadata：`plugins/feature-dev/.claude-plugin/plugin.json:1-9`
- marketplace 注册：`.claude-plugin/marketplace.json:62-70`
- 插件总览：`plugins/README.md:20`
- 本插件主文档：`plugins/feature-dev/README.md:1-412`

## 依赖与外部交互

### 运行时依赖

1. Claude Code 插件组件发现与 agent 执行机制（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-15,23-25`）
2. `/feature-dev` 命令在相应阶段调用这些 agent（`plugins/feature-dev/commands/feature-dev.md:41-44,78-79,106-107`）
3. Git 工作区上下文：`code-reviewer` 默认依赖 `git diff`（`plugins/feature-dev/agents/code-reviewer.md:13`）
4. 项目规范输入：`code-reviewer` 明确依赖 `CLAUDE.md` 规则（`plugins/feature-dev/agents/code-reviewer.md:9,17`）

### 外部交互面

1. 网络交互：全部 agent 开启 `WebSearch` 与 `WebFetch`（三个文件第 4 行）
2. Shell 交互：全部 agent 开启 `BashOutput` 与 `KillShell`（三个文件第 4 行）
3. 文件系统交互：全部 agent 开启 `Read`/`Glob`/`Grep`/`LS`，可进行大范围代码检索（同上）

### 交互约束

主命令要求在关键节点等待用户反馈（澄清、选型、评审处置），这限制了 agents 全自动闭环能力（`plugins/feature-dev/commands/feature-dev.md:66-69,81-82,108-109`）。

## 风险、边界与改进建议

### 风险

1. 工具权限较宽，最小权限不足  
三个 agent 都开放网络与 shell 管理工具；对仅需本地检索的任务来说权限偏大，误用面较广（`plugins/feature-dev/agents/*.md:4`）。

2. 评审口径存在文档不一致  
`code-reviewer` 规定只报 `>=80`（`plugins/feature-dev/agents/code-reviewer.md:33`），而 README 的示例与分档文字更宽（`plugins/feature-dev/README.md:310-312`），容易造成用户预期偏差。

3. agent 级无自动化回归  
目录内仅提示协议，缺少 lint/测试来保证“三个 agent 的命令引用、输出合同、阈值规则”在修改后不漂移。

4. 执行成本风险  
主流程在三个阶段都并行多 agent，易在大仓库下引入较高时延与 token 成本（`plugins/feature-dev/README.md:371-379`）。

### 边界

1. 本目录只定义 agent 角色协议，不直接实现业务代码。
2. 是否触发、触发多少个 agent 由 `commands/feature-dev.md` 决定，而非 agents 自身。
3. 产出质量高度依赖调用时提示内容和主线程汇总能力。

### 改进建议

1. 分级收敛工具权限  
为三类 agent 分别缩减工具集合：explorer/architect 默认去掉 `KillShell`；reviewer 默认去掉网络工具，仅在需要外部参考时显式放开。

2. 对齐 reviewer 阈值文档  
统一 `README` 与 agent 正文中的置信度分档，避免“示例阈值”与“执行阈值”冲突。

3. 增加 prompt 协议回归检查  
新增轻量脚本校验：
- `commands/feature-dev.md` 中提及的 agent 名称必须都存在
- `code-explorer` 必须要求“关键文件列表”
- `code-reviewer` 必须保留 `>=80` 阈值声明（或统一后的新阈值）

4. 增加“轻量模式”调用策略  
针对小改动场景允许降级为“1 explorer + 1 reviewer”，减少并行 agent 数量，降低延迟和消耗。
