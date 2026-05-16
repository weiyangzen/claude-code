# DIR 研究：plugins/plugin-dev/skills/plugin-structure/references

## 场景与职责

`plugins/plugin-dev/skills/plugin-structure/references` 是 `plugin-structure` 技能的“深水区知识层”，以两份参考文档承载核心协议细节：

1. `manifest-reference.md`：定义 `plugin.json` 字段语义、路径规则、验证规则与示例（`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:1-552`）。
2. `component-patterns.md`：定义组件生命周期与分层组织模式（`plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:1-567`）。

在整体插件开发链路中，它的职责不是执行代码，而是提供“可被流程与校验工具复用的规范契约”：

1. 上游调用方
- `plugin-structure` README 将该目录声明为 References 层，作为按需加载的详细说明（`plugins/plugin-dev/skills/plugin-structure/README.md:30-49,85-93`）。
- `plugin-structure` SKILL 在主文档末尾引导读取 `references/` 以做深入实现（`plugins/plugin-dev/skills/plugin-structure/SKILL.md:476`）。
- `/plugin-dev:create-plugin` 在 Phase 2 强制加载 `plugin-structure`，因此间接依赖本目录规范进行组件规划与结构落地（`plugins/plugin-dev/commands/create-plugin.md:48-52,116-149,371-374`）。

2. 下游被调用方
- `plugin-validator` 的校验项（manifest、路径、组件结构）直接消费本目录定义的规则（`plugins/plugin-dev/agents/plugin-validator.md:56-123`）。
- `skill-reviewer` 在审查技能时检查 `references/` 的组织质量与引用可达性（`plugins/plugin-dev/agents/skill-reviewer.md:73-84,99-105`）。
- `plugin-structure/examples` 的 minimal/standard/advanced 模板本质上是本目录规范的实例化（`plugins/plugin-dev/skills/plugin-structure/examples/minimal-plugin.md:64-67`，`plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md:131-142`）。

3. 与仓库级文档的关系
- 仓库 `plugins/README.md` 给出统一插件骨架（manifest 位于 `.claude-plugin/plugin.json`），本目录给出更细字段与路径约束，形成“总纲 + 细则”的文档分层（`plugins/README.md:47-61`，`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`）。

## 功能点目的

### 1) `manifest-reference.md` 的目标

1. 建立 manifest 的硬约束入口
- 明确 `.claude-plugin/plugin.json` 是必需位置，避免“插件存在但不被识别”的失效场景（`manifest-reference.md:7-10`）。

2. 定义核心元数据字段
- 规范 `name/version/description` 的格式、语义与示例，约束插件命名与版本治理（`manifest-reference.md:15-84`）。

3. 定义发布元数据
- 规范 `author/homepage/repository/license/keywords` 的结构与用途，服务分发、归属、可发现性（`manifest-reference.md:85-208`）。

4. 定义组件路径字段
- 说明 `commands/agents/hooks/mcpServers` 的类型与默认值，区分 string / array / inline object（`manifest-reference.md:209-330`）。

5. 定义路径解析协议
- 要求相对路径、`./` 前缀、禁止 `../` 与绝对路径；同时声明默认路径先扫描、再合并自定义路径（`manifest-reference.md:334-371`）。

6. 定义校验与常见错误
- 覆盖语法、字段、组件引用三类校验，并给出错误-修复对照（`manifest-reference.md:373-447`）。

7. 定义复杂度模板
- 给出 minimal/recommended/complete 三档 manifest，降低新建插件设计成本（`manifest-reference.md:449-519`）。

### 2) `component-patterns.md` 的目标

1. 建立运行生命周期心智模型
- 把“发现阶段”和“激活阶段”拆开，解释组件何时被扫描、何时被调用（`component-patterns.md:5-27`）。

2. 提供按组件类型的组织策略
- 为 commands/agents/skills/hooks/scripts 分别提供 flat/categorized/hierarchical 等模式与适用条件（`component-patterns.md:29-447`）。

3. 提供跨组件架构模式
- 给出 shared resources、layered architecture、plugin-within-plugin 模式，解决中大型插件的复用与分层问题（`component-patterns.md:448-535`）。

4. 提供演进型治理建议
- 从命名、可维护性、可扩展性、性能角度给出结构治理建议（`component-patterns.md:537-567`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程

1. 规范加载流程（文档侧）
- 用户触发“插件结构/manifest/组件组织”诉求。
- Claude 加载 `plugin-structure/SKILL.md` 的核心规则。
- 需要细节时下钻到本目录 `references/*.md`。
- 再回到 `create-plugin` 流程执行目录创建与组件实现。
- 证据：`plugins/plugin-dev/skills/plugin-structure/README.md:85-93`，`plugins/plugin-dev/commands/create-plugin.md:48-52,116-164`。

2. 运行时发现流程（协议侧）
- 先读 `.claude-plugin/plugin.json`。
- 扫描默认目录（commands/agents/skills/hooks/.mcp）。
- 追加扫描 manifest 自定义路径。
- 合并注册组件，冲突时报错。
- 证据：`manifest-reference.md:352-371`，`component-patterns.md:11-17`。

3. 激活流程（执行侧）
- Command：slash command 命中后执行。
- Agent：按任务能力匹配并选择。
- Skill：按 description 语义匹配加载。
- Hook：事件触发后按 matcher 执行。
- MCP：按工具能力路由到 server。
- 证据：`component-patterns.md:23-27`。

### B. 核心数据结构与协议

1. Manifest 字段协议
- `name` 正则：`/^[a-z][a-z0-9]*(-[a-z0-9]+)*$/`（`manifest-reference.md:33-36`）。
- `version` 语义版本（`manifest-reference.md:44-63`）。
- `commands/agents`：string 或 string[]（`manifest-reference.md:213-257`）。
- `hooks/mcpServers`：路径字符串或 inline 对象（`manifest-reference.md:261-330`）。

2. 路径协议
- 必须相对路径且以 `./` 开头；禁止 `../` 和绝对路径（`manifest-reference.md:338-350`）。
- 运行命令/脚本建议通过 `${CLAUDE_PLUGIN_ROOT}` 引用（`manifest-reference.md:283-285,318-321`；`component-patterns.md:470-473`）。

3. 生命周期协议
- 注册在初始化阶段完成，不是持续热发现；后续按组件类型激活（`component-patterns.md:17,23-27`）。

4. 组织模式协议
- 命令分层目录需要 manifest 显式列出路径，嵌套目录不会被自动递归发现（`component-patterns.md:93-121`）。
- hook 事件拆分文件模式仅是组织建议，文档明确 Claude Code 不支持直接文件引用占位，需要 build script 合并（`component-patterns.md:321-349`）。

### C. 关键命令与脚本语义（来源于参考文档示例）

1. Hook 命令
- 典型命令形式：`bash ${CLAUDE_PLUGIN_ROOT}/scripts/validate.sh`（`manifest-reference.md:283-285`）。

2. 共享库引入
- 典型 shell 复用方式：`source "${CLAUDE_PLUGIN_ROOT}/lib/test-utils.sh"`（`component-patterns.md:470-473`）。

3. 构建步骤
- 当 hooks 配置按事件拆分时，需要外部 build 脚本做合并（`component-patterns.md:348-349`）。

### D. 配置、测试、脚本、文档上下文

1. 配置
- 本目录核心输出是配置契约：`plugin.json` 字段、默认路径、解析顺序、校验规则（`manifest-reference.md:11-371`）。

2. 测试
- 本目录没有独立测试文件；实际校验由外部 `plugin-validator` 和组件技能中的校验脚本承担（`plugin-validator.md:56-123`，`create-plugin.md:233-260`）。

3. 脚本
- 本目录没有落地脚本文件；`component-patterns.md` 仅提供脚本组织与命令样式模式（`component-patterns.md:381-447,470-474`）。

4. 文档
- 本目录是 `plugin-structure` 的 references 子层，与 `SKILL.md`（核心）和 `examples/`（模板）形成渐进披露闭环（`plugin-structure/README.md:15-49,85-93`）。

## 关键代码路径与文件引用

### 目标目录（被研究对象）

1. `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md`
2. `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md`

### 上游调用方（谁依赖该目录）

1. `plugins/plugin-dev/skills/plugin-structure/README.md:30-49`
- 显式列出两份 reference 文档与作用。

2. `plugins/plugin-dev/skills/plugin-structure/SKILL.md:476`
- 主技能将深入阅读导向 `references/`。

3. `plugins/plugin-dev/commands/create-plugin.md:48-52,116-149`
- 流程强依赖 `plugin-structure` 规则来做目录/manifest 设计。

4. `plugins/plugin-dev/README.md:95-113`
- 在 toolkit 总览中把本目录作为 plugin-structure 的细节资源层。

### 下游被调用方（谁消费这些规则）

1. `plugins/plugin-dev/agents/plugin-validator.md:56-123`
- 校验清单直接映射 manifest、路径、组件规则。

2. `plugins/plugin-dev/agents/skill-reviewer.md:73-84,99-105`
- 评审技能是否正确使用 references 并保持引用可达。

3. `plugins/plugin-dev/skills/plugin-structure/examples/minimal-plugin.md:64-67`
4. `plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md:576-580`
5. `plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md:701-706`
- 三套示例结构对应本目录提出的分层模式与资源组织策略。

### 横向一致性文档

1. `plugins/README.md:47-61`
- 给出仓库级结构基线。

2. `plugins/plugin-dev/skills/mcp-integration/SKILL.md:23-59`
- 与本目录 `mcpServers` 规则形成互补：一个讲插件结构入口，一个讲 MCP 配置细节。

3. `plugins/plugin-dev/skills/skill-development/SKILL.md:331-340,608-615`
- 把 `references/` 作为技能分层原则，并将 `plugin-structure` 作为组织样例。

## 依赖与外部交互

### 1) 本地依赖

1. 依赖 `plugin-structure` 主技能触发与导航（`plugin-structure/SKILL.md:1-5,476`）。
2. 依赖 `create-plugin` 工作流在 Phase 2/4 实际落地规范（`create-plugin.md:48-52,116-149`）。
3. 依赖 `plugin-validator` 执行验收闭环（`plugin-validator.md:56-123`）。

### 2) 运行时/协议交互

1. 与 Claude Code 插件发现机制交互：manifest -> 目录扫描 -> 注册（`manifest-reference.md:352-371`，`component-patterns.md:11-17`）。
2. 与 Hook 协议交互：事件、matcher、command/prompt hook 结构（`manifest-reference.md:261-297`，`component-patterns.md:291-379`）。
3. 与 MCP 协议交互：`mcpServers` 路径或 inline 配置（`manifest-reference.md:298-330`）。

### 3) 外部标准/生态交互

1. SemVer：版本管理语义（`manifest-reference.md:44-63`）。
2. SPDX：license 标准标识（`manifest-reference.md:166-180`）。
3. URL 元数据：homepage/repository 的发布可追溯性（`manifest-reference.md:115-163`）。

### 4) 测试与脚本交互边界

1. 本目录不含可执行脚本与测试；其“可验证性”依赖外部 validator 与人工审查。
2. 文档中包含脚本与配置片段，但无本目录内 CI 自动校验机制。

## 风险、边界与改进建议

1. 风险：文档规则与运行时实现可能漂移
- 本目录是规范层，不是可执行层；若运行时行为变化，文档可能过时。
- 建议：增加针对 `manifest-reference.md` 示例的轻量 schema 校验脚本，并纳入 CI。

2. 风险：命令嵌套目录规则易误读
- `component-patterns.md` 强调命令嵌套目录不会自动发现（`93-121`），但主技能里“commands 目录自动加载”描述较概括（`plugin-structure/SKILL.md:112-115`）。
- 建议：在 `SKILL.md` commands 段补一句“默认不递归子目录”。

3. 风险：event-based hooks 示例含“不可直接使用”的占位语法
- 文档给出 `${file:...}` 组合写法，同时注明 Claude Code 不支持（`component-patterns.md:339-349`）。
- 建议：补一个最小 build 脚本示例，避免读者停留在概念层。

4. 风险：参考文档中的规模阈值是经验值而非硬约束
- 如 hooks inline `<50 lines`、MCP inline `<20 lines`（`manifest-reference.md:293-330`）属于建议，可能被误解为强规则。
- 建议：在对应段落补充 “heuristic, not enforced” 标注。

5. 风险：目录缺少“参考文档回归测试”
- 当前没有自动检查引用文件路径、示例 JSON 有效性、正则/字段说明一致性。
- 建议：新增 `scripts/validate-references.sh`，至少执行：
  - Markdown 链接与路径可达检查；
  - 代码块 JSON 语法抽样校验；
  - 关键规则（manifest 路径、`./` 前缀）的一致性 grep 检查。

6. 边界结论
- 该目录核心价值是“协议定义与组织模式沉淀”，而不是执行能力。
- 其质量上限取决于：
  - 与 `plugin-validator`/`create-plugin` 的规则同步速度；
  - 与 `examples/` 的一致性；
  - 是否具备最小自动化文档校验能力。
