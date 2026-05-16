# plugins/plugin-dev/skills/command-development/examples 目录研究（DIR）

## 场景与职责

`plugins/plugin-dev/skills/command-development/examples` 是 `plugin-dev` 中 `command-development` 技能的“样例层”，职责不是直接执行代码，而是给 Claude/开发者提供可复制的命令模板与模式库。

该目录当前包含两个核心样例文档：

1. `simple-commands.md`：面向项目级/个人级命令的基础模式（10 个示例）。
2. `plugin-commands.md`：面向插件命令的进阶模式（10 个示例，覆盖脚本、模板、agent、skill、多组件编排）。

它在上下文中的定位：

1. 上游调用方（文档级调用）
- `command-development/SKILL.md` 将 `examples/` 标注为命令模式样例入口（`plugins/plugin-dev/skills/command-development/SKILL.md:834`）。
- `command-development/README.md` 将两个样例文件列为“Examples”核心资产（`plugins/plugin-dev/skills/command-development/README.md:64,72,108-109`）。
- `/plugin-dev:create-plugin` 在组件实现阶段要求先加载 `command-development` skill，间接使 examples 成为命令落地参考（`plugins/plugin-dev/commands/create-plugin.md:153,159,183`）。

2. 下游被调用方（运行时/使用方）
- 插件作者或项目开发者将样例改写为真实命令文件（例如 `.claude/commands/*.md` 或 `plugin/commands/*.md`）。
- Claude Code 在命令执行时消费这些协议元素（frontmatter、`$1`、`$ARGUMENTS`、`@file`、`!\`bash\``）。

## 功能点目的

### 1) 提供“从简单到复杂”的命令模板集合

`simple-commands.md` 通过 10 个命令样例覆盖常见任务：代码审查、安全审查、测试、文档生成、Git 状态、部署、对比、快速修复、研究、代码讲解（`plugins/plugin-dev/skills/command-development/examples/simple-commands.md:7,44,87,118,163,196,234,279,315,360`）。

目标是让开发者快速落地最常见的 slash command，不必从空白 prompt 开始。

### 2) 提供插件特有能力的标准化写法

`plugin-commands.md` 的 10 个模式覆盖：脚本驱动、模板驱动、多脚本流水线、配置驱动部署、agent/skill 集成、输入校验、环境感知（`plugins/plugin-dev/skills/command-development/examples/plugin-commands.md:20,51,91,128,171,212,249,289,346,391`）。

目标是把 `${CLAUDE_PLUGIN_ROOT}`、插件资源路径、组件协同等插件特性写成可复用模式。

### 3) 把“命令接口”文档化

两个样例文件都大量展示：

1. `description`
2. `argument-hint`
3. `allowed-tools`
4. `model`

形成“命令即接口”的约定（如 `simple-commands.md:14,94,95,287`；`plugin-commands.md:29-30,60-62,355-356,400-401`）。

### 4) 提供反模式提示与实战建议

`plugin-commands.md` 内置了开发提示与常见错误（路径硬编码、遗漏 `allowed-tools`、未做输入验证等），用于减少复制样例后的落地失败（`plugins/plugin-dev/skills/command-development/examples/plugin-commands.md:483-552`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（从触发到落地）

1. 触发阶段
- 用户使用 `/plugin-dev:create-plugin` 或请求“创建命令”。
- `create-plugin` 在 Phase 5 明确要求加载 `command-development` skill（`plugins/plugin-dev/commands/create-plugin.md:153-191`）。

2. 知识加载阶段
- `SKILL.md` 提供核心规则，`README.md` 提供渐进披露导航，examples 被作为可直接套用的模板库（`plugins/plugin-dev/skills/command-development/README.md:94-109`，`plugins/plugin-dev/skills/command-development/SKILL.md:832-834`）。

3. 命令文件生成阶段
- 以 Markdown 命令文件为载体，填充 frontmatter + 指令正文。
- 使用 examples 中对应模式（简单命令或插件命令）进行改造。

4. 执行阶段
- Claude Code 根据 frontmatter 权限和模型配置执行命令。
- `@path` 注入文件内容，`!\`cmd\`` 注入命令输出，`$1/$ARGUMENTS` 替换命令参数。

### B. 数据结构与协议

1. 命令文件结构协议（Markdown + YAML frontmatter）
- 字段定义来自 `frontmatter-reference.md`：`description`、`allowed-tools`、`model`、`argument-hint`、`disable-model-invocation`（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:24,60,130,196,270`）。

2. 参数替换协议
- 位置参数：`$1/$2/...`
- 全量参数：`$ARGUMENTS`
- examples 中对应场景广泛覆盖（`simple-commands.md:94,241,286,322`；`plugin-commands.md:29,60,137,180,355,400`）。

3. 文件注入协议
- `@file-path` 把文件内容作为上下文输入（`simple-commands.md:128,244,370`；`plugin-commands.md:102,188,301`）。

4. Bash 上下文协议
- `!\`command\`` 在执行前收集动态上下文。
- 由 `allowed-tools` 管控可执行命令范围（`simple-commands.md:170-174`，`plugin-commands.md:61,138,181,356,401`）。

5. 插件路径协议
- `${CLAUDE_PLUGIN_ROOT}` 作为插件内资源定位锚点（`plugin-commands.md:35,69,104,144,183,306,360,405`）。
- 详细行为在 `plugin-features-reference.md` 说明（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:75-99,109-164,567`）。

6. 交互协议（本目录未直接示例，但属于上下文依赖）
- `AskUserQuestion` 参数结构与 `multiSelect` 规则在 `interactive-commands.md` 定义（`plugins/plugin-dev/skills/command-development/references/interactive-commands.md:32-64,269-333`）。

### C. 关键命令模式

1. 基础模式（simple）
- Read-only 分析、Git 上下文、单参数/多参数、快速模型（haiku）、文件对比、上下文聚合（`simple-commands.md:410-493`）。

2. 插件模式（plugin）
- 脚本执行、配置加载、模板驱动、agent 调度、skill 引导、输入/资源校验、环境差异化执行（`plugin-commands.md:437-481`）。

3. 测试模式（依赖 references）
- 7 级测试策略（结构、字段、手工、参数、文件引用、bash、集成）+ 自动化脚本模板（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:9-343`）。

## 关键代码路径与文件引用

### 目标目录（被研究对象）

1. `plugins/plugin-dev/skills/command-development/examples/simple-commands.md`
- 通用命令样例库；覆盖 frontmatter 与常见任务模板。

2. `plugins/plugin-dev/skills/command-development/examples/plugin-commands.md`
- 插件命令样例库；覆盖 `${CLAUDE_PLUGIN_ROOT}`、组件协同与校验模式。

### 上游调用与上下文依赖

1. `plugins/plugin-dev/skills/command-development/SKILL.md`
- 对 examples 的正式引用与规则主文档（`326,645,832-834`）。

2. `plugins/plugin-dev/skills/command-development/README.md`
- 渐进披露索引，明确 examples 在技能结构中的层级（`21-72,94-109`）。

3. `plugins/plugin-dev/commands/create-plugin.md`
- 工作流命令，Phase 5 要求加载 `command-development` skill 并实现 commands（`153-191`）。

4. `plugins/plugin-dev/README.md`
- `/plugin-dev:create-plugin` 作为总入口（`21-49`）。

### 被调用规范与补充文档（examples 的“下游规范依赖”）

1. `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md`
- frontmatter 字段规范。

2. `plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md`
- 插件命令发现、`${CLAUDE_PLUGIN_ROOT}`、组件集成、校验模式。

3. `plugins/plugin-dev/skills/command-development/references/interactive-commands.md`
- AskUserQuestion 协议。

4. `plugins/plugin-dev/skills/command-development/references/testing-strategies.md`
- 命令测试分层与自动化模板。

### 相关配置/脚本/清单路径

1. `Docs/researches/blueprint_checklist.md:79`
- 当前研究对象在总 checklist 的条目。

2. `.ops/generate_daily_research_todo.sh`
- 从 checklist 计算 pending/done 并生成 `Docs/researches/todos_YYYYMMDD.md`（`.ops/generate_daily_research_todo.sh:5-7,15-18,41-42`）。

3. `.ops/generate_research_blueprint_checklist.sh`
- 维护全量研究蓝图清单（`.ops/generate_research_blueprint_checklist.sh:5,66,70-73`）。

## 依赖与外部交互

### 1) 依赖类型

1. 文档依赖
- examples 依赖 `SKILL.md` 与 `references/*.md` 提供规则与边界。

2. 工具协议依赖
- `allowed-tools` 绑定 Claude Code 工具能力（Read/Grep/Bash/Task/Skill/AskUserQuestion）。

3. shell 生态依赖（样例中出现）
- `git`、`gh`、`npm`、`kubectl`、`node`、`bash`、`grep`、`test`。

4. 插件结构依赖
- `${CLAUDE_PLUGIN_ROOT}` 所指向的插件目录结构需真实存在（`scripts/`、`config/`、`templates/` 等）。

### 2) 外部交互方式

1. 与文件系统交互
- 读取目标代码文件（`@...`），读取插件模板/配置（`@${CLAUDE_PLUGIN_ROOT}/...`）。

2. 与系统命令交互
- 通过 `!\`...\`` 运行命令收集上下文并反馈给模型。

3. 与 Claude Code 子系统交互
- 通过自然语言提示触发 agent/skill；交互式流程可调用 AskUserQuestion（在 references 层定义）。

### 3) 测试与验证现状

1. 本目录无可执行测试脚本或 CI 配置文件。
2. 测试能力主要以 `testing-strategies.md` 的模板与流程存在，属于“规范建议”而非“自动执行资产”。

## 风险、边界与改进建议

### 风险 1（高）：样例中的 `Bash(*)` 容易被直接复制到生产命令

现状：多个插件样例使用 `allowed-tools: Bash(*)`（`plugin-commands.md:61,138,356,401,530`）。

影响：权限面过大，增加误执行和安全风险。

建议：
1. 默认改为白名单模式（如 `Bash(node:*)`、`Bash(git:*)`）。
2. 仅在确有必要时保留 `Bash(*)`，并在样例中写明判据。

### 风险 2（高）：样例资源路径多为“约定存在”，缺少前置校验

现状：大量示例直接引用 `${CLAUDE_PLUGIN_ROOT}/scripts|config|templates`。

影响：用户复制后若资源不存在，命令会失败。

建议：
1. 在更多样例中内置 `test -f/-d` 校验（目前仅部分示例体现）。
2. 增加“最小可运行插件骨架”链接或文件树示例。

### 风险 3（中）：examples 未直接覆盖 AskUserQuestion 交互命令

现状：交互协议在 `references/interactive-commands.md`，但 `examples/` 下没有同级互动命令示例。

影响：用户需要跨文件拼装，学习路径变长。

建议：新增 `interactive-plugin-commands.md`（至少包含确认、多选、条件分支三个完整例子）。

### 风险 4（中）：规范与验证存在潜在漂移

现状：`frontmatter-reference` 允许 `allowed-tools` 为字符串或数组；`plugin-validator` 当前描述偏向“数组”（`plugins/plugin-dev/agents/plugin-validator.md:76-84`）。

影响：可能出现“样例合法但校验误报”的体验。

建议：统一 validator 与规范文本，并补充对应测试样例。

### 风险 5（低）：本目录缺少可执行回归脚本

现状：测试策略存在于文档，不在本目录落地脚本。

影响：examples 更新后难以自动回归验证可用性。

建议：
1. 在 `command-development/scripts/` 增加 `validate-example-frontmatter.sh` 与 `lint-command-examples.sh`。
2. 在 CI 中增加对 examples 的结构检查（frontmatter、字段合法性、危险权限扫描）。

### 边界说明

1. 本目录是“样例知识库”，不是运行时模块；不会被编译或单元测试直接执行。
2. 示例中的命令片段是模式模板，不保证在任何仓库即插即用。
3. 真实可用性受插件实际目录结构、工具安装、权限策略和 Claude Code 版本行为共同影响。
