# plugins/plugin-dev/skills/command-development 目录研究（DIR）

## 场景与职责

`plugins/plugin-dev/skills/command-development` 是 `plugin-dev` 插件中负责“命令开发规范化”的技能目录，核心职责是把 Claude Code slash command 从零散提示词提升为可复用、可验证、可分发的工程资产。

该目录在 `plugin-dev` 全局体系中的位置：

1. 作为 7 大技能之一，对应“Command Development”能力域（`plugins/plugin-dev/README.md:7-15,135-152`）。
2. 在 `/plugin-dev:create-plugin` 工作流的 Phase 5 被显式加载，用于指导命令文件落地（`plugins/plugin-dev/commands/create-plugin.md:157-164,183-191`）。
3. 其规范会被 `plugin-validator` 间接消费（用于校验 `commands/**/*.md` 的 frontmatter 和内容完整性，`plugins/plugin-dev/agents/plugin-validator.md:76-84`）。

目录内部职责分层：

1. 核心规范层：`SKILL.md`
- 定义 slash command 的基本语义、frontmatter 字段、参数替换、文件引用、bash 注入、插件组件集成（`plugins/plugin-dev/skills/command-development/SKILL.md:22-327,527-834`）。

2. 渐进披露层：`references/*.md`
- 拆分出 frontmatter 细则、交互式命令、高级工作流、测试策略、文档模式、市场化分发规范（`plugins/plugin-dev/skills/command-development/README.md:43-111`）。

3. 可执行样例层：`examples/*.md`
- 给出通用命令与插件命令的完整模板，覆盖 agent/skill/hook/脚本/配置等组合场景（`plugins/plugin-dev/skills/command-development/examples/simple-commands.md:7-504`，`plugins/plugin-dev/skills/command-development/examples/plugin-commands.md:20-557`）。

## 功能点目的

### 1) 统一“命令是给 Claude 的指令”这一核心心智

`SKILL.md` 明确命令内容应写给模型执行而非写给用户说明，避免“描述性文案”导致执行意图不完整（`plugins/plugin-dev/skills/command-development/SKILL.md:30-53`）。

### 2) 统一命令元数据与权限边界

通过 frontmatter 字段规范：
- `description`：帮助 `/help` 可发现（`plugins/plugin-dev/skills/command-development/SKILL.md:114-127`）。
- `allowed-tools`：工具权限最小化（`plugins/plugin-dev/skills/command-development/SKILL.md:128-146`）。
- `model`：按复杂度选择 `haiku/sonnet/opus`（`plugins/plugin-dev/skills/command-development/SKILL.md:147-163`）。
- `argument-hint`：命令接口文档化（`plugins/plugin-dev/skills/command-development/SKILL.md:164-180`）。
- `disable-model-invocation`：限制仅手工触发（`plugins/plugin-dev/skills/command-development/SKILL.md:181-194`）。

补充细则由 `frontmatter-reference.md` 给出（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:24-323`）。

### 3) 统一命令输入模型

支持三类输入：
1. 文本参数：`$ARGUMENTS`、`$1/$2/...`（`plugins/plugin-dev/skills/command-development/SKILL.md:195-263`）。
2. 文件注入：`@path`（`plugins/plugin-dev/skills/command-development/SKILL.md:265-315`）。
3. 运行时上下文：`!\`bash\``（`plugins/plugin-dev/skills/command-development/SKILL.md:316-327`）。

### 4) 把插件化场景变成标准模式

通过 `${CLAUDE_PLUGIN_ROOT}` 解决路径可移植、跨安装一致性问题（`plugins/plugin-dev/skills/command-development/SKILL.md:529-573`，`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:75-220`）。

### 5) 支持多组件编排（agent/skill/hook）

命令可：
- 触发 agent（Task tool 语义）（`plugins/plugin-dev/skills/command-development/SKILL.md:651-679`，`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:332-357`）。
- 借助 skill 进行领域规范化输出（`plugins/plugin-dev/skills/command-development/SKILL.md:680-706`）。
- 与 hooks 协同（状态准备与事件联动）（`plugins/plugin-dev/skills/command-development/SKILL.md:707-716`，`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:385-407`）。

### 6) 从“可用”提升到“可发布”

目录内引用文档覆盖测试、文档、市场分发：
- 测试分层（结构、字段、手工、集成、性能、UX）：`testing-strategies.md`（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:9-702`）。
- 自文档化模式：`documentation-patterns.md`（`plugins/plugin-dev/skills/command-development/references/documentation-patterns.md:9-739`）。
- 市场分发与兼容性：`marketplace-considerations.md`（`plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md:9-904`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程：从触发到执行

1. 触发与装载
- 技能触发词由 `SKILL.md` frontmatter `description` 定义（`plugins/plugin-dev/skills/command-development/SKILL.md:1-5`）。
- 编排入口 `/plugin-dev:create-plugin` 在命令实现阶段显式加载本技能（`plugins/plugin-dev/commands/create-plugin.md:157-164,183-191`）。

2. 命令文件生成
- 以 Markdown 为载体，frontmatter 可选（`plugins/plugin-dev/skills/command-development/SKILL.md:74-110`）。
- 通过 `argument-hint` 暴露参数契约，通过 prompt body 定义执行指令（`plugins/plugin-dev/skills/command-development/SKILL.md:164-180,195-263`）。

3. 执行上下文拼装
- `@file` 在执行前注入文件内容（`plugins/plugin-dev/skills/command-development/SKILL.md:267-315`）。
- `!\`cmd\`` 执行 shell 命令注入动态上下文（`plugins/plugin-dev/skills/command-development/SKILL.md:316-327`）。

4. 插件能力联动
- `${CLAUDE_PLUGIN_ROOT}` 解析插件内绝对路径（`plugins/plugin-dev/skills/command-development/SKILL.md:529-573`）。
- 命令通过文本指令触发 Task/Skill 协作（`plugins/plugin-dev/skills/command-development/SKILL.md:651-706`）。

### B. 关键数据结构与协议

1. 命令 frontmatter 协议
- 核心字段：`description / allowed-tools / model / argument-hint / disable-model-invocation`（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:22-323`）。
- `allowed-tools` 支持字符串与数组格式，且默认继承会话权限（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:62-86,64`）。
- `model` 值域被限定为 `sonnet/opus/haiku`（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:135-148`）。

2. 交互式问答协议（AskUserQuestion）
- 问题结构：`question/header/options/multiSelect`（`plugins/plugin-dev/skills/command-development/references/interactive-commands.md:34-60`）。
- 设计约束：选项通常 2-4 个、每次 1-4 个问题，支持自动 `Other`（`plugins/plugin-dev/skills/command-development/references/interactive-commands.md:63-67,235-240`）。

3. 工作流状态协议
- 建议把跨命令状态存入 `.claude/*.local.md`，frontmatter 承载阶段、环境、commit、标志位（`plugins/plugin-dev/skills/command-development/references/advanced-workflows.md:281-313`）。
- 支持锁文件与 checkpoint 恢复（`plugins/plugin-dev/skills/command-development/references/advanced-workflows.md:404-448,579-602`）。

### C. 关键命令模式（目录内定义）

1. 基础模式
- 代码审查、测试执行、文档生成、Git 状态、部署等（`plugins/plugin-dev/skills/command-development/examples/simple-commands.md:7-230`）。

2. 插件模式
- 脚本驱动、模板驱动、多脚本流水线、环境感知、资源校验（`plugins/plugin-dev/skills/command-development/examples/plugin-commands.md:20-209,346-434`）。

3. 交互模式
- 确认型、条件分支型、迭代采集型、上下文感知型（`plugins/plugin-dev/skills/command-development/references/interactive-commands.md:269-379,381-425,624-666`）。

4. 测试模式
- 7 级测试金字塔 + 自动化脚本模板 + CI/pre-commit 模板（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:11-124,291-452`）。

## 关键代码路径与文件引用

### 目录主文件（目标对象）

1. `plugins/plugin-dev/skills/command-development/SKILL.md`
- 核心规范与插件集成入口。

2. `plugins/plugin-dev/skills/command-development/README.md`
- 技能结构、使用方式、渐进披露入口。

3. `plugins/plugin-dev/skills/command-development/examples/simple-commands.md`
- 通用命令样例库（10 个）。

4. `plugins/plugin-dev/skills/command-development/examples/plugin-commands.md`
- 插件命令样例库（10 个，含 agent/skill/hook 协作）。

5. `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md`
- frontmatter 字段规范源。

6. `plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md`
- 插件路径、组件联动、校验模式。

7. `plugins/plugin-dev/skills/command-development/references/interactive-commands.md`
- AskUserQuestion 协议与交互流程。

8. `plugins/plugin-dev/skills/command-development/references/advanced-workflows.md`
- 多命令工作流、状态管理、恢复机制。

9. `plugins/plugin-dev/skills/command-development/references/testing-strategies.md`
- 测试分层、脚本模板、CI 策略。

10. `plugins/plugin-dev/skills/command-development/references/documentation-patterns.md`
- 自文档化、错误信息、README 模板。

11. `plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md`
- 分发兼容性、命名冲突、可运维性。

### 上下游依赖（调用方 / 被调用方）

1. 调用方：`plugins/plugin-dev/commands/create-plugin.md`
- 在 Phase 5 显式要求加载 command-development skill（`157-164,183-191`）。

2. 调用方：`plugins/plugin-dev/README.md`
- 将 command-development 定义为 toolkit 核心能力并给出触发词（`135-152`）。

3. 被调用方：`plugins/plugin-dev/agents/plugin-validator.md`
- 对命令文件进行前置质量校验（`76-84`）。

4. 相关规范：`plugins/plugin-dev/skills/plugin-structure/SKILL.md`
- 对命令目录组织与自动发现给出基础约束（`110-135`）。

5. 元规范：`plugins/plugin-dev/skills/skill-development/SKILL.md`
- 将 command-development 作为“最佳实践样例”引用（`608-614`）。

## 依赖与外部交互

### 运行时工具依赖

1. Claude Code 工具接口
- `Read/Write/Grep/Glob/Bash`：命令和样例频繁使用。
- `AskUserQuestion`：交互命令的核心机制（`plugins/plugin-dev/skills/command-development/references/interactive-commands.md:13-67`）。
- `Task`：命令触发 agent（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:334-357`）。
- `Skill`：命令提示中引用技能以触发知识加载（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:358-384`）。

2. Shell/CLI 生态依赖（按样例）
- `git`, `gh`, `npm`, `kubectl`, `helm`, `node`, `bash`, `test`, `grep`（见各 references/examples 的 `allowed-tools` 与 `!\`` 命令片段）。

### 文件系统与配置交互

1. 命令目录
- 项目级 `.claude/commands/`、用户级 `~/.claude/commands/`、插件级 `plugin-name/commands/`（`plugins/plugin-dev/skills/command-development/SKILL.md:54-73`）。

2. 插件资源路径协议
- `${CLAUDE_PLUGIN_ROOT}` 用于脚本、模板、配置、文档定位（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:75-220`）。

3. 状态与配置文件
- `.claude/*.local.md` 在高级工作流中承担跨命令状态共享（`plugins/plugin-dev/skills/command-development/references/advanced-workflows.md:281-331`）。

### 测试与发布交互

1. 本目录没有实际可执行 `scripts/` 子目录，测试能力以参考文档中的脚本模板形式提供（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:34-124,341-452`）。
2. 市场化建议覆盖跨平台、依赖探测、版本兼容、beta 流程与更新提示（`plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md:11-109,408-479,726-876`）。

## 风险、边界与改进建议

### 风险 1（高）：跨技能规范存在不一致（manifest 路径）

现状：
1. command-development 的插件目录示例使用 `plugin-name/plugin.json`（`plugins/plugin-dev/skills/command-development/SKILL.md:579-586`）。
2. plugin 体系文档与 plugin-structure 技能强调 `.claude-plugin/plugin.json`（`plugins/README.md:51-60`，`plugins/plugin-dev/skills/plugin-structure/SKILL.md:110-135`）。

影响：开发者可能按错误位置放置 manifest，导致自动发现/安装行为不一致。

建议：统一为 `.claude-plugin/plugin.json`，并在 command-development 示例内修正目录树。

### 风险 2（高）：`allowed-tools` 校验规则与规范定义冲突

现状：
1. frontmatter 规范允许 `allowed-tools` 为字符串或数组（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:62-86`）。
2. plugin-validator 只强调“若存在则应是数组”（`plugins/plugin-dev/agents/plugin-validator.md:82`）。

影响：合法的字符串格式可能在验证阶段被误报。

建议：让 validator 与规范对齐，支持字符串与数组两种合法形态。

### 风险 3（中）：README 状态信息与当前内容漂移

现状：
- README 把 advanced/testing/documentation/marketplace 标注为“in progress”（`plugins/plugin-dev/skills/command-development/README.md:250-255`），但这些 reference 文件已存在且内容完整（`plugins/plugin-dev/skills/command-development/references/*.md`）。

影响：读者会误判目录成熟度，降低信任。

建议：更新状态段，改为“已提供 + 后续迭代方向”。

### 风险 4（中）：部分样例过度放宽 Bash 权限

现状：
- 多处样例使用 `allowed-tools: Bash(*)`（例如 `plugins/plugin-dev/skills/command-development/examples/plugin-commands.md:61,138,356,401`）。
- 同目录规范又强调应尽量最小化权限（`plugins/plugin-dev/skills/command-development/SKILL.md:405-410`，`references/frontmatter-reference.md:124-127`）。

影响：新手复制样例时易引入过宽权限。

建议：默认改为带命令前缀白名单的 `Bash(xxx:*)`，并在“何时必须 Bash(*)”给出明确判据。

### 风险 5（中）：`plugin-validator` 文件尾部疑似残留文本

现状：
- `plugins/plugin-dev/agents/plugin-validator.md` 在主要内容后出现与上下文不一致的收尾句（`184` 行）。

影响：可能误导 agent 输出或干扰 prompt 边界。

建议：清理残留文本，确保 agent 提示词单一闭环。

### 风险 6（低）：目录偏“规范文档”，缺少可直接执行的回归脚本资产

现状：
- 本目录主要是 `SKILL + references + examples`，无 `scripts/` 目录。
- `testing-strategies.md` 提供了 `validate-command.sh / validate-frontmatter.sh / test-commands.sh` 模板，但未作为项目内真实脚本落地（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:34-124,343-386`）。

影响：测试实践易停留在“文档建议”，难以形成标准化回归。

建议：新增 `scripts/` 并把模板脚本产品化，接入 pre-commit/CI 示例中提到的路径约定。

### 边界说明

1. 该目录本质是“方法论与模板层”，不直接执行业务逻辑；其有效性依赖 Claude Code 运行时实现与工具权限体系。
2. 目录中大量 `!\`` 与流程片段是示例 DSL，不保证可直接复制执行到所有项目；需要按项目工具链裁剪。
3. 分发级可靠性（跨平台、兼容性、依赖探测）目前主要靠文档约束而非强制校验。
