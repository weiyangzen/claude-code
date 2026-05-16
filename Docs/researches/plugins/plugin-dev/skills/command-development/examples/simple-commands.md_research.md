# plugins/plugin-dev/skills/command-development/examples/simple-commands.md 研究

## 场景与职责

`plugins/plugin-dev/skills/command-development/examples/simple-commands.md` 是 `command-development` 技能中的“通用命令样例集”，主要面向 `.claude/commands/*.md` 这类项目/个人命令，不绑定插件私有资源。

它在体系中的职责：

1. 通过 10 个基础样例说明“命令应写给 Claude，而不是写给用户”的写法基线（`simple-commands.md:5`）。
2. 提供 frontmatter、参数、文件引用、bash 注入的最小落地模板（`simple-commands.md:11-504`）。
3. 作为 command-development skill 的 examples 层资产，由 README 与 SKILL 文档间接引用（`plugins/plugin-dev/skills/command-development/README.md:60-111`；`plugins/plugin-dev/skills/command-development/SKILL.md:832-834`）。

同时，它与 `/plugin-dev:create-plugin` 的 Command 实施步骤形成上下游关系：创建命令时可直接复用此文件中的结构化模板（`plugins/plugin-dev/commands/create-plugin.md:153-190`）。

## 功能点目的

该文件通过 10 个例子覆盖高频命令类型：

1. `review.md`：仓库代码质量审查（`simple-commands.md:7-40`）。
2. `security-review.md`：安全漏洞审查，含严重级别输出要求（`simple-commands.md:44-83`）。
3. `test-file.md`：单文件测试执行与结果解读（`simple-commands.md:87-114`）。
4. `document.md`：文件文档生成（`simple-commands.md:118-159`）。
5. `git-status.md`：Git 状态聚合（`simple-commands.md:163-192`）。
6. `deploy.md`：部署流程模板（`simple-commands.md:196-230`）。
7. `compare-files.md`：双文件差异与影响评估（`simple-commands.md:234-275`）。
8. `quick-fix.md`：快速修复模式，使用 `haiku`（`simple-commands.md:279-311`）。
9. `research.md`：主题研究与改进建议（`simple-commands.md:315-356`）。
10. `explain.md`：代码讲解模板（`simple-commands.md:360-406`）。

末尾 `Key Patterns` 抽象出 7 个可复用构件（只读分析、Git 操作、参数模式、快速模型、上下文聚合等），并给出写作建议（`simple-commands.md:410-504`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程

该文件隐含的命令实现流程如下：

1. 定义命令文件路径（`.claude/commands/*.md`），并使用 Markdown + YAML frontmatter 描述行为。
2. 声明能力边界：通过 `allowed-tools` 约束工具访问（如 `Read`、`Grep`、`Bash(git:*)`、`Bash(kubectl:*)`，见 `simple-commands.md:14,51,95,170,204,485`）。
3. 参数绑定：
- 位置参数 `$1/$2`（`simple-commands.md:98,207,244,370,444,456`）。
- 全量参数 `$ARGUMENTS`（`simple-commands.md:290,326`）。
4. 上下文注入：
- 文件注入：`@file`（`simple-commands.md:128,244,370,476`）。
- 命令注入：`!\`cmd\``（`simple-commands.md:100,175-181,212,431,488`）。
5. 输出约束：每个样例明确期望输出结构（例如审查分类、风险等级、部署步骤、比较报告、讲解层次）。

### 2) 数据结构与协议

样例采用一致的“命令协议模板”：

1. `File`：目标命令文件名。
2. frontmatter（可选）：`description`、`allowed-tools`、`model`、`argument-hint`。
3. 正文指令：以任务清单、分节要求、输出格式约束组成。
4. `Usage`：命令调用示例。

这些字段和约束由 `frontmatter-reference.md` 提供权威语义：

1. frontmatter 全字段均为可选（`frontmatter-reference.md:20`）。
2. `allowed-tools` 可字符串/数组并支持 `Bash(prefix:*)`（`frontmatter-reference.md:60-100`）。
3. `model` 建议按复杂度匹配（`frontmatter-reference.md:130-194`）。
4. `argument-hint` 用于参数接口可读性（`frontmatter-reference.md:196-268`）。

### 3) 关键命令协议片段

1. 最小权限协议：
- 审查类命令常用 `Read/Grep` 或 `Bash(git:*)`（`simple-commands.md:14,51,170,416-429`）。

2. 参数接口协议：
- 单参数：`argument-hint: [target]` + `$1`（`simple-commands.md:437-445`）。
- 多参数：`argument-hint: [source] [target] [options]` + `$1/$2/$3`（`simple-commands.md:449-457`）。
- 自由文本：`$ARGUMENTS`（`simple-commands.md:290,326`）。

3. 模型选择协议：
- 快速任务示例指定 `haiku`（`simple-commands.md:287,465-466`）。
- 复杂分析示例可用 `sonnet`（`simple-commands.md:52,323`）。

4. 上下文聚合协议：
- 把 `git status` 和文件内容合并为统一输入上下文（`simple-commands.md:481-492`）。

## 关键代码路径与文件引用

### 目标文件

1. `plugins/plugin-dev/skills/command-development/examples/simple-commands.md`
- 通用命令样例与模式总结（`1-504`）。

### 上游调用与文档入口

1. `plugins/plugin-dev/skills/command-development/README.md`
- 声明 simple-commands 是 10 个完整通用样例（`64-70,107-109`）。
2. `plugins/plugin-dev/skills/command-development/SKILL.md`
- 将 examples 目录作为模式示例出口（`832-834`）。
3. `plugins/plugin-dev/commands/create-plugin.md`
- 在 Commands 实施步骤要求遵循 command-development 技能规则（`183-190`）。
4. `plugins/plugin-dev/README.md`
- 对外说明 Command Development 覆盖 frontmatter、参数、bash 与组织方式（`135-152`）。

### 下游规范与辅助文档

1. `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md`
- 字段定义与校验项（`20-105,130-323,445-453`）。
2. `plugins/plugin-dev/skills/command-development/references/testing-strategies.md`
- 结构验证、参数测试、文件引用测试、bash 测试与集成测试（`11-343,454-630`）。
3. `plugins/plugin-dev/skills/command-development/references/documentation-patterns.md`
- 命令内嵌注释和可维护文档模式（`11-71,96-166`）。
4. `plugins/plugin-dev/skills/command-development/references/interactive-commands.md`
- 需要结构化问答时的 AskUserQuestion 规范（`13-67,269-333,462-541`）。

### 配置/脚本/测试关联

1. 该文件不依赖插件私有配置路径（不使用 `${CLAUDE_PLUGIN_ROOT}`）。
2. 依赖外部命令生态：`git`、`npm`、`jest`、`kubectl`（`simple-commands.md:95,170-181,204,212`）。
3. 测试资产不在当前目录，需按 `testing-strategies.md` 补充验证脚本。

## 依赖与外部交互

### 运行时依赖

1. Slash command 运行时：frontmatter 解析、参数替换、`@` 文件注入、`!\`bash\`` 注入。
2. 工具权限系统：`allowed-tools` 是命令行为边界。
3. 模型路由：`model` 字段影响执行成本与能力分配。

### 外部交互

1. 与仓库文件系统交互：读取目标文件（文档化、对比、解释场景）。
2. 与 shell 命令交互：执行 git/test/deploy 上下文命令。
3. 与用户交互：当前样例主要通过参数输入，不含 AskUserQuestion 的结构化问答。

### 与测试/验证体系关系

文件本身是示例文档，不含自动化测试；建议依据 `testing-strategies.md` 的 7 层策略做回归（语法、字段、参数、文件引用、bash、集成、UX），并在 CI 或 pre-commit 中运行命令校验脚本（`testing-strategies.md:11-443,545-630`）。

## 风险、边界与改进建议

### 风险 1（中）：参数缺失与非法输入处理覆盖不足

现状：多数样例直接消费 `$1/$2/$ARGUMENTS`，未展示前置验证（如 `simple-commands.md:98,207,244,290,326,370`）。

影响：用户漏填参数时，命令输出可能不稳定或误导。

建议：增加“参数校验版”样例，至少展示 `test -n`/枚举校验模板。

### 风险 2（中）：部署示例含确认语句但未使用结构化交互工具

现状：`deploy.md` 以自然语言询问 `Proceed with deployment? (yes/no)`（`simple-commands.md:224`）。

影响：在自动化/多轮对话中，确认逻辑可重复性较弱。

建议：给出 AskUserQuestion 版本并设置 `disable-model-invocation` 示例（参照 `interactive-commands.md:269-333` 与 `frontmatter-reference.md:270-323`）。

### 风险 3（中）：`git-status` 示例包含 `git fetch`，存在额外副作用和网络依赖

现状：`Remote Status` 步骤执行 `git fetch && git status -sb`（`simple-commands.md:181`）。

影响：命令从“纯观察”变为“会更新远程引用”的操作，且依赖网络可达性。

建议：提供只读替代（例如仅 `git status -sb`）并把 `fetch` 设为可选分支。

### 风险 4（低）：工具权限示例与“最小权限”原则可进一步对齐

现状：多处做得较好（`Read/Grep`、`Bash(git:*)`），但缺少“为何选择该权限”的注释。

建议：在复杂命令里增加一行注释说明权限理由，便于后续审计。

### 风险 5（低）：缺少与插件场景的桥接说明

现状：本文件偏通用命令，未说明何时应切换到 `plugin-commands.md`（后者包含 `${CLAUDE_PLUGIN_ROOT}` 与 agent/skill 协同）。

建议：在文件开头或结尾新增“适用边界”段，明确“通用命令 vs 插件命令”的选择标准。

### 边界说明

1. 本文件是“通用模板集合”，不直接绑定任何真实业务目录或脚本。
2. 示例可复制但不可保证即插即用，需按仓库工具链与权限策略调整。
3. 对高风险命令（部署、批量修改）应补充人工确认和测试回归后再投入日常使用。
