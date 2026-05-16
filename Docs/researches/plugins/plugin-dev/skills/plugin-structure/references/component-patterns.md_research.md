# plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md 研究

## 场景与职责
`component-patterns.md` 是 `plugin-structure` 技能的高级组织参考层，职责不是定义“字段语法”，而是定义“结构演进策略”。在 `plugin-structure` 的分层里，它位于 core `SKILL.md` 之后、examples 之前，承接“原则 -> 组织模式 -> 可落地布局”的中间层：

1. 对上游的职责（被谁调用）
- `plugin-structure/README.md` 将其列为两份参考文档之一，定位为 Advanced organization patterns，覆盖 lifecycle、命令/agent/skill/hook/script 组织与可扩展模式（`plugins/plugin-dev/skills/plugin-structure/README.md:30-49`）。
- `plugin-structure/SKILL.md` 末尾明确将深入阅读导向 `references/`，在结构设计场景下触发本文件（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:476`）。
- `/plugin-dev:create-plugin` 在 Phase 2 强制加载 `plugin-structure`，实际进行组件规划时会间接消费本文件中的组织建议（`plugins/plugin-dev/commands/create-plugin.md:44-63,373-374`）。

2. 对下游的职责（影响谁）
- 影响 `examples/minimal-plugin.md`、`standard-plugin.md`、`advanced-plugin.md` 的结构选型，尤其是多目录命令、分层 hooks、共享库与 layered architecture（`plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md:7-99,131-142`）。
- 为 `plugin-validator` 的“结构正确性”检查提供模式背景，例如目录位置、frontmatter 要求、hooks 文件组织（`plugins/plugin-dev/agents/plugin-validator.md:67-123`）。

3. 在同目录中的角色边界
- `manifest-reference.md` 负责 `plugin.json` 字段契约与路径规则；
- `component-patterns.md` 负责组件目录与资源布局模式。
两者共同构成“配置语义 + 结构实践”的完整规范面。

## 功能点目的
本文件按组件类型给出“何时使用什么结构”的决策框架，主要功能点如下：

1. 生命周期模型（Discovery / Activation）
- 目的：先建立 Claude Code 组件注册与触发的时序心智模型，避免把“注册时机”和“执行时机”混淆。
- 内容：启动时扫描 manifest 与组件、注册并初始化 hooks/MCP；运行时按 command/agent/skill/hook/mcp 类型触发（`component-patterns.md:5-27`）。

2. Commands 组织模式
- 目的：在命令数量增长时，提供从 flat 到 categorized 到 hierarchical 的升级路径。
- 内容：
  - flat：5-15 个命令时保持单目录（`component-patterns.md:31-53`）；
  - categorized：通过 manifest `commands` 数组声明多目录（`component-patterns.md:54-92`）；
  - hierarchical：嵌套目录不自动发现，需要显式列出每个子目录（`component-patterns.md:93-121`）。

3. Agents / Skills 组织模式
- 目的：在能力增长后按角色、能力域、流程阶段进行解耦。
- 内容：
  - agents：role-based / capability-based / workflow-based（`component-patterns.md:133-185`）；
  - skills：topic/tool/workflow 三种组织法（`component-patterns.md:186-260`），并补充 rich resources 结构（`component-patterns.md:261-290`）。

4. Hooks 与 Scripts 组织模式
- 目的：降低配置复杂度与维护成本，避免 hooks.json 与脚本目录失控。
- 内容：
  - hooks：monolithic、event-based、purpose-based（`component-patterns.md:291-380`）；
  - scripts：flat、categorized、language-based（`component-patterns.md:381-447`）。

5. 跨组件架构模式
- 目的：处理中大型插件中的复用与分层问题。
- 内容：shared resources、layered architecture、plugin-within-plugin（`component-patterns.md:448-535`）。

6. 可维护性与性能建议
- 目的：给出增长期插件的治理原则，避免后期重构成本过高。
- 内容：命名一致性、组织可扩展性、目录深度与 discovery 性能关系（`component-patterns.md:537-567`）。

## 具体技术实现（关键流程/数据结构/协议/命令）
### 1) 关键流程
1. 发现阶段流程（注册期）
- Claude Code 启动后：扫描已启用插件 -> 读取 `.claude-plugin/plugin.json` -> 扫描默认与自定义路径 -> 解析 frontmatter/JSON -> 注册组件 -> 初始化 hooks/MCP（`component-patterns.md:9-17`）。
- 技术要点：组件注册发生于初始化阶段，不是持续热发现（`component-patterns.md:17`）。

2. 激活阶段流程（运行期）
- Command：slash 命令查表后执行；
- Agent：任务到达后按能力选择；
- Skill：语义匹配 `description` 时加载；
- Hook：事件触发按 matcher 执行；
- MCP：工具调用按 server capability 转发（`component-patterns.md:23-27`）。

3. 结构升级流程（规模演进）
- 小规模插件：flat；
- 中规模插件：按功能分目录并在 manifest 显式声明；
- 大规模插件：采用 layered + shared resources + plugin-within-plugin 组合（`component-patterns.md:54-121,448-535`）。

### 2) 关键数据结构与协议
1. Manifest 与目录的映射协议（本文件示例）
- 多目录命令配置：
```json
{
  "commands": [
    "./commands",
    "./admin-commands",
    "./workflow-commands"
  ]
}
```
- 用途：把目录结构决策显式化，绕开默认发现对嵌套目录的限制（`component-patterns.md:72-121`）。

2. Hook 配置组织协议
- 单文件模式：所有事件放在 `hooks/hooks.json`（`component-patterns.md:293-320`）。
- 事件拆分模式：示例使用 `${file:...}` 占位符组装，但文档明确 Claude Code 不支持直接文件引用，需构建脚本先合并为最终 `hooks.json`（`component-patterns.md:321-349`）。

3. 共享资源协议
- 跨组件共享脚本/库通过 `${CLAUDE_PLUGIN_ROOT}` 绝对定位插件根目录，避免 cwd 依赖：
```bash
source "${CLAUDE_PLUGIN_ROOT}/lib/test-utils.sh"
run_tests
```
（`component-patterns.md:469-474`）。

4. 架构分层协议
- `commands` 作为用户交互层，`agents` 作为编排层，`skills` 作为知识层，`lib/core|integrations|utils` 作为实现层（`component-patterns.md:481-500`）。

### 3) 关键命令/脚本模式
本文件本身是文档，不直接执行命令；但给出可执行模式约束：
- Hook/脚本引用应使用 `${CLAUDE_PLUGIN_ROOT}`（`component-patterns.md:470-473`）；
- 事件拆分 hooks 必须借助“构建合并脚本”产物化（`component-patterns.md:348-349`）。

### 4) 配置、测试、脚本、文档实现关系
1. 配置
- 通过 manifest 的 `commands` 等字段把目录模式落地为可发现配置（与 `manifest-reference.md` 对齐，`component-patterns.md:72-121`）。

2. 测试
- 本文件没有测试脚本；验证依赖外部流程：`create-plugin` 的验证阶段与 `plugin-validator`（`plugins/plugin-dev/commands/create-plugin.md:233-260`，`plugins/plugin-dev/agents/plugin-validator.md:49-142`）。

3. 脚本
- 本文件给出 scripts 的目录治理模式（`component-patterns.md:381-447`），但不提供实际脚本实现。

4. 文档
- 与 `SKILL.md`、`manifest-reference.md`、`examples/` 构成 progressive disclosure：核心概念 -> 深度规范 -> 模板化实例（`plugins/plugin-dev/skills/plugin-structure/README.md:85-93`）。

## 关键代码路径与文件引用
1. 被研究对象
- `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:1-567`

2. 直接文档调用方
- `plugins/plugin-dev/skills/plugin-structure/README.md:30-49,85-93`
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:339-356,476`
- `plugins/plugin-dev/README.md:95-113`

3. 业务流程调用方
- `plugins/plugin-dev/commands/create-plugin.md:44-63,116-149,233-260,373-374`

4. 规则消费方
- `plugins/plugin-dev/agents/plugin-validator.md:49-123`
- `plugins/plugin-dev/skills/plugin-structure/examples/minimal-plugin.md:7-13`
- `plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md:7-35`
- `plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md:7-99,131-142`

5. 互补规范文件
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:209-371`（字段类型与解析顺序）
- `plugins/plugin-dev/skills/plugin-structure/SKILL.md:20-44,86-107`（核心层规则）

## 依赖与外部交互
1. 仓库内依赖
- 强依赖 `plugin-structure` 主技能触发与导航（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:2-4,476`）。
- 强依赖 `manifest-reference.md` 提供字段语义；本文件专注结构策略而非字段定义。
- 间接依赖 `create-plugin` 与 `plugin-validator` 把文档规则转化为执行与校验动作。

2. 运行时协议交互
- 与 Claude Code 的组件发现机制交互：manifest 读取 + 组件扫描 + 注册（`component-patterns.md:11-17`）。
- 与 Hook 事件协议交互：PreToolUse/PostToolUse/Stop 等事件下的组织与脚本结构（`component-patterns.md:291-374`）。
- 与 Skill 触发协议交互：描述匹配触发（`component-patterns.md:25`）。

3. 外部交互
- 本文件不直接调用第三方 API 或网络服务；外部交互来自其推荐的 MCP/Hook 架构在实际插件中的执行。

4. 测试与脚本依赖边界
- 无独立测试、无构建脚本；
- “event-based hooks 需要 build script”的要求仅以说明形式出现，未提供标准实现模板（`component-patterns.md:348-349`）。

## 风险、边界与改进建议
1. 风险：文档建议与运行时能力存在错配窗口
- 典型例子是 `${file:...}` 拆分 hooks 的写法本身不可直接运行，若读者忽略注释会产生配置失败（`component-patterns.md:339-349`）。
- 建议：在该段落增加“可直接复制的最小合并脚本”与产物示例，降低误用概率。

2. 风险：规模阈值为经验值，缺乏可验证标准
- 如“5-10 scripts / 10+ hooks / 20+ commands”等均是启发式阈值，不是硬约束（`component-patterns.md:44-47,350-353,421-424`）。
- 建议：补一个“决策矩阵”章节，按团队人数、变更频率、组件耦合度给出可量化选择标准。

3. 风险：性能建议缺少可观测基线
- 文档建议“避免深层嵌套、减少 custom path”以优化 discovery（`component-patterns.md:563-567`），但没有基准测试方法。
- 建议：在 references 增补 `benchmark-discovery.md`，定义样本规模与测量步骤。

4. 边界：本文件不定义 manifest 字段合法性
- 字段类型、路径合法性、错误样例在 `manifest-reference.md`，本文件只给“组织模式”与“适用场景”。
- 建议：在开头显式加入“字段规范请看 manifest-reference”的 cross-link，减少读者误解。

5. 边界：本文件不提供自动化校验器
- 校验职责由 `plugin-validator` 和各技能脚本承担，不在本文件内实现。
- 建议：新增一个轻量脚本检查器，对示例目录树和 JSON 片段做一致性 smoke-test（例如检查 hierarchical commands 是否都出现在 manifest `commands` 数组中）。
