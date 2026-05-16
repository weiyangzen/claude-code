# plugins/plugin-dev/skills/command-development/examples/plugin-commands.md 研究

## 场景与职责

`plugins/plugin-dev/skills/command-development/examples/plugin-commands.md` 是 `command-development` 技能的“插件命令样例库”，职责是给插件作者提供可直接改写的命令蓝图，而不是可直接执行的脚本集合。

它在插件开发链路中的位置：

1. 作为示例资产被 `command-development` README 明确列出，属于渐进披露中的 Examples 层（`plugins/plugin-dev/skills/command-development/README.md:60-111`）。
2. 被核心技能文档引用，作为“命令模式样例目录”入口（`plugins/plugin-dev/skills/command-development/SKILL.md:832-834`）。
3. 被 `/plugin-dev:create-plugin` 的 Phase 5 间接消费：该流程要求实现 Commands 时加载 command-development skill（`plugins/plugin-dev/commands/create-plugin.md:153-190`）。

因此，该文件承担“插件命令设计范式的标准样例”职责：强调 `${CLAUDE_PLUGIN_ROOT}`、插件资源路径、组件协同（agent/skill）与输入校验。

## 功能点目的

该文件共 10 个样例，功能意图是按复杂度递进覆盖插件命令常见问题。

1. 脚本驱动基础命令：`commands/analyze.md`，展示参数 + 文件引用 + Node 脚本执行（`plugin-commands.md:20-47`）。
2. 多脚本审计：`commands/full-audit.md`，展示串行执行多个脚本并汇总报告（`plugin-commands.md:51-88`）。
3. 模板驱动生成：`commands/gen-api-docs.md`，把模板与目标文件组合（`plugin-commands.md:91-125`）。
4. 发布流水线：`commands/release.md`，以步骤化方式编排 build/test/package（`plugin-commands.md:128-168`）。
5. 配置驱动部署：`commands/deploy.md`，按环境动态加载配置并补充 git/build 上下文（`plugin-commands.md:171-209`）。
6. Agent 集成：`commands/deep-review.md`，明确通过 Task 工具委派子代理（`plugin-commands.md:212-246`）。
7. Skill 集成：`commands/document-api.md`，通过技能名称注入领域规范（`plugin-commands.md:249-285`）。
8. 多组件编排：`commands/complete-review.md`，组合脚本、agent、skill、模板（`plugin-commands.md:289-343`）。
9. 输入与资源校验：`commands/build-env.md`，先校验再执行并区分失败路径（`plugin-commands.md:346-388`）。
10. 环境感知执行：`commands/run-checks.md`，按 prod/non-prod 选择检查强度（`plugin-commands.md:391-434`）。

此外，文件末尾提供“模式摘要 + 测试建议 + 常见错误”，把样例上升为可复用规则（`plugin-commands.md:437-557`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程

该文件隐含的命令执行流程可拆为：

1. 命令触发：用户输入 `/command-name args`，命中插件命令文件。
2. frontmatter 解析：读取 `description`、`argument-hint`、`allowed-tools`、`model` 等元数据（如 `plugin-commands.md:27-31,58-63,296-300`）。
3. 参数替换：`$1` 等位置参数进入正文（如 `plugin-commands.md:33,65,141,184,302,404`）。
4. 上下文拼装：
- 文件注入：`@${CLAUDE_PLUGIN_ROOT}/...` 或 `@$1`（`plugin-commands.md:103-105,184,302,325,367,406`）。
- 命令注入：`!\`...\`` 执行脚本/CLI（`plugin-commands.md:35,68-74,144-153,186-188,307,359-371,408-418`）。
5. 协同阶段（可选）：
- Agent：通过 Task 工具委派（`plugin-commands.md:224-238`）。
- Skill：通过 skill 名称提示调用（`plugin-commands.md:265-277,317-323`）。
6. 产出阶段：要求模型按结构化模板给出报告/建议（如 `plugin-commands.md:155-160,330-335,422-426`）。

### 2) 数据结构与协议

该文件采用“样例块协议”，每个样例结构一致：

1. `Use case`：场景标识。
2. `File`：建议落地路径（`commands/*.md`）。
3. Markdown 代码块：命令完整内容（含 YAML frontmatter + prompt）。
4. `Key features`：该样例想强调的模式。

其中 frontmatter 字段语义由上游参考文件定义：

1. 所有 frontmatter 字段均可选（`references/frontmatter-reference.md:20`）。
2. `allowed-tools` 支持字符串或数组（`references/frontmatter-reference.md:60-86`）。
3. `model` 值域与适用情境有规范（`references/frontmatter-reference.md:130-194`）。
4. `disable-model-invocation` 是敏感命令约束手段（`references/frontmatter-reference.md:270-323`）。

### 3) 关键命令与协议片段

1. 路径可移植协议：`${CLAUDE_PLUGIN_ROOT}`
- 在脚本、配置、模板、文档引用中贯穿使用（如 `plugin-commands.md:35,68,103,184,234-236,307,325,361,406`）。
- 与 `plugin-features-reference` 的定义一致（`references/plugin-features-reference.md:75-107`）。

2. 组件协同协议：
- Agent 协同：需在插件 `agents/` 下存在定义，Claude 通过 Task 工具拉起（`references/plugin-features-reference.md:332-357`）。
- Skill 协同：需在插件 `skills/` 下存在定义，通过名称提示触发（`references/plugin-features-reference.md:358-383`）。

3. 校验协议：
- 参数白名单校验：`grep -E "^(dev|staging|prod)$"`（`plugin-commands.md:359`）。
- 资源存在性校验：`test -x/-f`（`plugin-commands.md:361-363,477`）。
- 失败分支说明：在 prompt 中定义“校验失败时的解释与修复建议”（`plugin-commands.md:375-379`）。

4. 插件命令发现协议（上下文依赖）：
- 插件 `commands/` 自动发现，子目录形成 namespace（`references/plugin-features-reference.md:15-52`）。

## 关键代码路径与文件引用

### 目标文件

1. `plugins/plugin-dev/skills/command-development/examples/plugin-commands.md`
- 样例主体与模式总结（`1-557`）。

### 直接上游（调用方/入口）

1. `plugins/plugin-dev/skills/command-development/README.md`
- 声明该文件是 Examples 核心资产（`60-111`）。
2. `plugins/plugin-dev/skills/command-development/SKILL.md`
- 在结尾将 examples 作为命令模式入口（`832-834`）。
3. `plugins/plugin-dev/commands/create-plugin.md`
- Phase 5 对 Commands 的实施要求会加载 command-development 技能并生成命令（`153-190`）。
4. `plugins/plugin-dev/README.md`
- 对外说明 Command Development 能力域与触发短语（`135-152`）。

### 关键下游依赖（被调用方/规范）

1. `plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md`
- 自动发现、`${CLAUDE_PLUGIN_ROOT}`、agent/skill/hook 协同、校验模式（`13-52,75-107,330-463`）。
2. `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md`
- frontmatter 字段定义与校验清单（`20-105,130-323,445-453`）。
3. `plugins/plugin-dev/skills/command-development/references/testing-strategies.md`
- 7 级测试策略与脚本模板（`11-343,388-443,592-630`）。
4. `plugins/plugin-dev/skills/command-development/references/interactive-commands.md`
- 交互式命令的 AskUserQuestion 协议（`13-67,269-333,462-541`）。

### 配置/测试/脚本关联

1. 配置路径约定：`${CLAUDE_PLUGIN_ROOT}/config/*.json`（`plugin-commands.md:184,363,367,406`）。
2. 脚本路径约定：`${CLAUDE_PLUGIN_ROOT}/scripts/*.sh|*.js`（`plugin-commands.md:35,68-74,144-153,307,361,369,371,411-418`）。
3. 模板路径约定：`${CLAUDE_PLUGIN_ROOT}/templates/*.md`（`plugin-commands.md:103,325,453`）。
4. 测试建议：跨目录执行、变量展开验证、资源检查（`plugin-commands.md:485-510`）。

## 依赖与外部交互

### 运行时依赖

1. Claude Code slash command 解析能力：frontmatter、参数替换、`@` 文件注入、`!\`bash\`` 注入。
2. 工具权限模型：`allowed-tools` 决定 Read/Bash/Task/Skill 可用面。
3. 插件系统能力：`${CLAUDE_PLUGIN_ROOT}` 注入与 `commands/` 自动发现（`plugin-features-reference.md:15-31,75-85`）。

### 外部交互面

1. Shell 工具链：`node`、`bash`、`git`、`grep`、`test`、`cat`、`ls` 等命令调用（`plugin-commands.md` 多处 `!\`...\``）。
2. 文件系统：读取插件配置、模板、脚本与目标代码文件。
3. 组件总线：
- 与 agent 交互依赖 Task tool（`plugin-commands.md:238`）。
- 与 skill 交互依赖技能名触发（`plugin-commands.md:265,318`）。

### 与测试体系的关系

该文件本身不含可执行测试脚本；测试方法在 `testing-strategies.md` 以规范形式给出，例如结构验证脚本、frontmatter 验证脚本、集成测试场景与 pre-commit/CI 建议（`testing-strategies.md:38-124,291-343,388-443`）。

## 风险、边界与改进建议

### 风险 1（高）：示例资源多为约定路径，仓库中并无同名实体

现状：示例大量引用 `${CLAUDE_PLUGIN_ROOT}/scripts|config|templates|checklists|docs`（如 `plugin-commands.md:234-236,307,325,361-363,406,411-418`），但 `plugins/plugin-dev` 目录并未提供对应运行资产。

影响：读者直接复制后会因缺文件失败。

建议：
1. 增加“最小可运行样例插件结构”节。
2. 每个高风险样例默认附带 `test -f/-x` 预检片段。

### 风险 2（高）：`Bash(*)` 使用频繁，最小权限原则不足

现状：多个样例用 `allowed-tools: Bash(*)`（`plugin-commands.md:61,138,181,356,401,530`）。

影响：扩大命令执行面，安全与可审计性下降。

建议：
1. 默认改为 `Bash(node:*)`、`Bash(git:*)`、`Bash(kubectl:*)` 等白名单。
2. 保留 `Bash(*)` 的场景需附理由和边界说明。

### 风险 3（中）：Agent/Skill 名称是示意值，缺少存在性校验

现状：`code-reviewer`、`code-quality-reviewer`、`coding-standards` 等名称没有配套检查（`plugin-commands.md:224,310,318`）。

影响：实际插件中若名称不一致，会导致协同链路失效。

建议：增加“组件存在性检查清单”（例如在执行前核对 `agents/` 与 `skills/` 目录）。

### 风险 4（中）：发布/部署类命令未展示 `disable-model-invocation`

现状：`release/deploy/build-env` 均可视为高风险操作，但样例未示范手动触发保护。

影响：在自动化调用链中可能触发不应自动执行的动作。

建议：对破坏性或生产影响命令展示 `disable-model-invocation: true`（参考 `frontmatter-reference.md:270-323`）。

### 风险 5（中）：文本式条件分支偏提示词语义，缺少交互协议范式

现状：样例包含 “If validations fail/For production environment”等自然语言分支（`plugin-commands.md:365-379,410-426`），但未结合 AskUserQuestion 的结构化交互。

影响：在需要确认/多选的场景下，执行一致性不如结构化问答。

建议：补充 interactive 插件命令样例，明确何时改用 AskUserQuestion（`interactive-commands.md:13-67,269-333`）。

### 边界说明

1. 本文件是“模式文档”，不保证样例引用的脚本/配置在当前仓库存在。
2. 命令实际效果受插件目录结构、工具安装、权限策略、模型行为共同影响。
3. 样例的主要价值在“结构与协议”，落地前需按目标插件进行适配与验证。
