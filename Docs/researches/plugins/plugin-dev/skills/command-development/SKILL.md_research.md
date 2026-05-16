# plugins/plugin-dev/skills/command-development/SKILL.md 研究

## 场景与职责

`plugins/plugin-dev/skills/command-development/SKILL.md` 是该技能的核心规范文件，直接承担“触发条件定义 + 命令开发标准 + 插件集成模式”三类职责。

它在 `plugin-dev` 体系中的责任边界：

1. 定义技能何时被加载（frontmatter `description` 的触发短语），覆盖“create slash command / command frontmatter / interactive command / AskUserQuestion”等场景（`plugins/plugin-dev/skills/command-development/SKILL.md:1-5`）。
2. 给出命令开发最小闭环：命令是什么、如何写 frontmatter、如何传参、如何引文件、如何注入 bash、如何组织目录（`plugins/plugin-dev/skills/command-development/SKILL.md:20-526`）。
3. 给出插件化闭环：`${CLAUDE_PLUGIN_ROOT}`、插件命令组织、与 agent/skill/hook 协作、输入与资源校验（`plugins/plugin-dev/skills/command-development/SKILL.md:527-834`）。

在调用链上的位置：

- `plugins/plugin-dev/commands/create-plugin.md` 在 Phase 5 显式要求“Commands: Load command-development skill”（`plugins/plugin-dev/commands/create-plugin.md:157-164,183-191`），本文件即该步骤的主规范源。

## 功能点目的

### 1. 固化“命令写给 Claude，而不是写给用户”的核心原则

该文件在开头以正反例强调命令提示应是执行指令，不是功能说明文案（`plugins/plugin-dev/skills/command-development/SKILL.md:30-53`）。

目的：避免命令内容“描述用户收益”而不“约束模型行为”。

### 2. 统一命令元数据契约

通过 frontmatter 章节规范字段语义与适用场景：

1. `description`（可发现性）
2. `allowed-tools`（权限边界）
3. `model`（复杂度匹配）
4. `argument-hint`（接口可读性）
5. `disable-model-invocation`（手工触发限制）

位置：`plugins/plugin-dev/skills/command-development/SKILL.md:112-194`。

### 3. 统一输入模型与上下文拼装

输入/上下文三件套：

1. 参数替换：`$ARGUMENTS`、`$1/$2/...`（`plugins/plugin-dev/skills/command-development/SKILL.md:195-263`）
2. 文件注入：`@path`（`plugins/plugin-dev/skills/command-development/SKILL.md:265-315`）
3. 运行时上下文注入：`!\`command\``（`plugins/plugin-dev/skills/command-development/SKILL.md:316-327`）

目的：把“静态提示词”变成“上下文感知命令模板”。

### 4. 统一插件级命令设计范式

核心在 `${CLAUDE_PLUGIN_ROOT}` 与“插件多组件协作”：

1. 插件内路径可移植（`plugins/plugin-dev/skills/command-development/SKILL.md:529-573`）
2. 命令自动发现与命名空间组织（`plugins/plugin-dev/skills/command-development/SKILL.md:574-599`）
3. 脚本/模板/配置模式（`plugins/plugin-dev/skills/command-development/SKILL.md:600-645`）
4. agent/skill/hook 协作（`plugins/plugin-dev/skills/command-development/SKILL.md:647-749`）

### 5. 提供可操作的校验与故障处理模式

通过参数验证、文件存在性、插件资源检查、错误处理模板，引导命令在失败时可解释可恢复（`plugins/plugin-dev/skills/command-development/SKILL.md:751-828`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（从触发到落地）

1. 触发
- 技能由 frontmatter 描述中的短语触发（`plugins/plugin-dev/skills/command-development/SKILL.md:1-5`）。

2. 语义建模
- 先确定命令性质与位置（项目/用户/插件）（`plugins/plugin-dev/skills/command-development/SKILL.md:54-73`）。

3. 契约声明
- 选择 frontmatter 字段声明能力边界（`plugins/plugin-dev/skills/command-development/SKILL.md:112-194`）。

4. 输入与上下文注入
- 参数替换 + 文件引用 + bash 执行组合成运行时上下文（`plugins/plugin-dev/skills/command-development/SKILL.md:195-327`）。

5. 结构化组织与复用
- 按命令规模选择扁平目录或 namespaced 目录（`plugins/plugin-dev/skills/command-development/SKILL.md:328-369`）。

6. 插件化扩展
- 使用 `${CLAUDE_PLUGIN_ROOT}` 引入插件脚本、配置、模板，并可串联 agent/skill/hook（`plugins/plugin-dev/skills/command-development/SKILL.md:527-749`）。

7. 校验与错误处理
- 执行输入验证、资源验证、失败分支处理（`plugins/plugin-dev/skills/command-development/SKILL.md:751-828`）。

### B. 数据结构与协议

1. Skill frontmatter 协议
- `name`、`description`、`version`（`plugins/plugin-dev/skills/command-development/SKILL.md:1-5`）。
- `description` 同时承担“触发词枚举”的语义路由职责。

2. 命令 frontmatter 协议
- 字段及语义由本文件概述、并由 `references/frontmatter-reference.md` 详化（`plugins/plugin-dev/skills/command-development/SKILL.md:112-194,832-833`；`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:22-323`）。

3. 参数替换协议
- `$ARGUMENTS`：整段参数
- `$1/$2/$3`：位置参数
- 混合使用：将“固定槽位 + 剩余参数”合并（`plugins/plugin-dev/skills/command-development/SKILL.md:195-263`）。

4. 上下文注入协议
- `@file` 注入文件内容（`plugins/plugin-dev/skills/command-development/SKILL.md:265-315`）。
- `!\`bash\`` 注入命令输出（`plugins/plugin-dev/skills/command-development/SKILL.md:316-327`）。

5. 插件路径协议
- `${CLAUDE_PLUGIN_ROOT}` 作为插件绝对根路径变量，解决“安装位置不确定”问题（`plugins/plugin-dev/skills/command-development/SKILL.md:529-573`）。

### C. 关键命令模式

1. 审查模式（Review）
- 借助 `git diff` 动态收集变更并结构化输出（`plugins/plugin-dev/skills/command-development/SKILL.md:436-453`）。

2. 测试模式（Testing）
- 参数驱动 + `npm` 执行 + 失败分析（`plugins/plugin-dev/skills/command-development/SKILL.md:455-467`）。

3. 工作流模式（Workflow）
- 多步流程串联 `gh`、审查、检查与决策（`plugins/plugin-dev/skills/command-development/SKILL.md:485-500`）。

4. 插件多组件模式
- 脚本分析 + agent 深审 + skill 校验 + 模板报告（`plugins/plugin-dev/skills/command-development/SKILL.md:717-743`）。

## 关键代码路径与文件引用

### 核心文件

1. `plugins/plugin-dev/skills/command-development/SKILL.md`
- 目标研究文件；定义核心原则、协议与模式。

2. `plugins/plugin-dev/skills/command-development/README.md`
- 入口导航，声明结构与渐进披露（`plugins/plugin-dev/skills/command-development/README.md:19-111`）。

### 下游细化文档（被调用方）

1. `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md`
- frontmatter 字段细节、校验清单与错误样例（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:24-455`）。

2. `plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md`
- 插件命令发现、`${CLAUDE_PLUGIN_ROOT}`、多组件协作与校验模板（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:13-606`）。

3. `plugins/plugin-dev/skills/command-development/references/interactive-commands.md`
- AskUserQuestion 交互协议（问题结构、`multiSelect`、多阶段问答）（`plugins/plugin-dev/skills/command-development/references/interactive-commands.md:15-67,149-197,269-920`）。

4. `plugins/plugin-dev/skills/command-development/references/advanced-workflows.md`
- 状态文件、锁、checkpoint、恢复与回滚（`plugins/plugin-dev/skills/command-development/references/advanced-workflows.md:279-723`）。

5. `plugins/plugin-dev/skills/command-development/references/testing-strategies.md`
- 7 级测试体系、脚本模板、pre-commit/CI 建议（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:11-124,291-452,592-702`）。

6. `plugins/plugin-dev/skills/command-development/references/documentation-patterns.md`
- 自文档化注释结构、README 伴随文档、版本迁移说明（`plugins/plugin-dev/skills/command-development/references/documentation-patterns.md:11-71,557-739`）。

7. `plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md`
- 分发兼容性、依赖探测、版本兼容、发布清单与维护策略（`plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md:11-109,345-450,726-904`）。

8. `plugins/plugin-dev/skills/command-development/examples/simple-commands.md`
9. `plugins/plugin-dev/skills/command-development/examples/plugin-commands.md`
- 分别提供通用命令与插件命令模板（`plugins/plugin-dev/skills/command-development/examples/simple-commands.md:7-504`；`plugins/plugin-dev/skills/command-development/examples/plugin-commands.md:20-557`）。

### 调用方（上游）

1. `plugins/plugin-dev/commands/create-plugin.md`
- Phase 5 命令创建时显式要求加载本技能（`plugins/plugin-dev/commands/create-plugin.md:157-164,183-191`）。

2. `plugins/plugin-dev/README.md`
- 对外说明 Command Development 技能触发词与覆盖范围（`plugins/plugin-dev/README.md:135-152`）。

3. `plugins/plugin-dev/skills/skill-development/SKILL.md`
- 将该目录作为技能设计模板参考（`plugins/plugin-dev/skills/skill-development/SKILL.md:608-614`）。

## 依赖与外部交互

### 运行时依赖

1. Claude Code 命令机制
- 依赖 slash command 的 Markdown/frontmatter 解析、参数替换、文件引用与 bash 注入能力。

2. 工具权限模型
- 依赖 `allowed-tools` 协议约束工具使用范围（`plugins/plugin-dev/skills/command-development/SKILL.md:128-146`）。

3. 插件系统语义
- 依赖插件命令自动发现、命名空间显示、`${CLAUDE_PLUGIN_ROOT}` 变量注入（`plugins/plugin-dev/skills/command-development/SKILL.md:529-599`）。

### 外部交互

1. Shell/CLI 交互
- 文档示例涉及 `git`、`gh`、`npm`、`node`、`bash`、`test`、`grep` 等命令（见 `SKILL.md` 与 references/examples 中 `!\`` 片段）。

2. 文件系统交互
- 常见交互对象：`.claude/commands/`、`~/.claude/commands/`、`plugin-name/commands/`、`${CLAUDE_PLUGIN_ROOT}/...`、`.claude/*.local.md`。

3. 组件协作交互
- agent（Task 工具语义）、skill（技能触发语义）、hook（事件触发语义）在命令文本中被编排（`plugins/plugin-dev/skills/command-development/SKILL.md:647-749`）。

## 风险、边界与改进建议

### 风险 1（高）：触发描述覆盖交互场景，但主文未显式承接 interactive 细则

现状：
- frontmatter 描述包含“interactive command / use AskUserQuestion”（`plugins/plugin-dev/skills/command-development/SKILL.md:3`）。
- 主文正文未展开 AskUserQuestion 协议，也未在结尾引用 `interactive-commands.md`。

影响：技能被交互问题触发后，主体内容可能无法立即给出交互参数规范。

建议：在主文新增“Interactive Commands”节，至少链接并摘要 `interactive-commands.md` 的问题结构与 `multiSelect` 约束。

### 风险 2（高）：插件 manifest 路径示例与仓库其他规范不一致

现状：
- 本文件示例使用 `plugin-name/plugin.json`（`plugins/plugin-dev/skills/command-development/SKILL.md:579-586`）。
- 其他规范强调 `.claude-plugin/plugin.json`（`plugins/README.md:51-55`；`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`）。

影响：开发者可能按错误结构组织插件，导致发现/安装异常。

建议：统一示例为 `.claude-plugin/plugin.json`，并在相关 references 同步校正。

### 风险 3（中）：权限最小化原则与示例存在张力

现状：
- 文中强调 `Bash(git:*)` 优于 `Bash(*)`（`plugins/plugin-dev/skills/command-development/SKILL.md:405-410`）。
- 多个模式示例仍使用 `Bash(*)`（`plugins/plugin-dev/skills/command-development/SKILL.md:608,635,811`）。

影响：读者倾向复制宽权限示例，放大安全面。

建议：默认示例改为受限白名单，仅在确有必要时展示 `Bash(*)` 并说明判据。

### 风险 4（中）：与 validator 对 `allowed-tools` 的约束口径不一致

现状：
- 本技能明确 `allowed-tools` 可为字符串或数组（`plugins/plugin-dev/skills/command-development/SKILL.md:131`）。
- `plugin-validator` 文本写“allowed-tools is array if present”（`plugins/plugin-dev/agents/plugin-validator.md:82`）。

影响：可能出现“规范合法但验证误报”。

建议：统一 validator 与规范，接受字符串和数组两种合法形态。

### 风险 5（中）：文档示例含伪流程控制语法，缺少运行时边界说明

现状：
- 文件示例包含 `$IF(...)`、`If ... Otherwise ...` 等描述性控制流（如 `plugins/plugin-dev/skills/command-development/SKILL.md:392-395,765-769`），但未说明其是否为 Claude Code 原生语法。

影响：新手可能误以为可直接按 DSL 执行。

建议：增加“示例语法说明”小节，区分“提示词结构化写法”与“运行时保证语法”。

### 风险 6（低）：缺少对测试/文档/分发参考文档的显式出口

现状：
- 结尾仅引用 frontmatter、plugin-features 与 examples（`plugins/plugin-dev/skills/command-development/SKILL.md:832-834`）。
- 未直接导向 testing/documentation/marketplace/advanced/interactive。

建议：在尾部补充 references 索引矩阵，减少知识孤岛。

### 边界说明

1. `SKILL.md` 是“规范与模式定义”，不是执行器，不保证示例脚本真实存在。
2. 真实效果依赖 Claude Code 运行时、插件结构正确性、以及命令作者实现质量。
3. 文件中很多命令片段属于模板性指导，落地前需结合目标仓库工具链与权限策略做适配。
