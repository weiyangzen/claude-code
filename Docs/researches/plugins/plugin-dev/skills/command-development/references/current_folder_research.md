# plugins/plugin-dev/skills/command-development/references 目录研究（DIR）

## 场景与职责

`plugins/plugin-dev/skills/command-development/references` 是 `command-development` 技能的“深度规范层”，承接核心 `SKILL.md` 的渐进披露设计，把高频但细节密集的规则拆分为 7 份专题文档：

1. `frontmatter-reference.md`：命令元数据字段规范（`description / allowed-tools / model / argument-hint / disable-model-invocation`）与校验清单（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:22-463`）。
2. `plugin-features-reference.md`：插件命令发现机制、`${CLAUDE_PLUGIN_ROOT}`、与 agent/skill/hook 协同模式（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:13-609`）。
3. `interactive-commands.md`：`AskUserQuestion` 交互协议与问答流设计（`plugins/plugin-dev/skills/command-development/references/interactive-commands.md:32-920`）。
4. `advanced-workflows.md`：多命令编排、状态持久化、锁与恢复机制（`plugins/plugin-dev/skills/command-development/references/advanced-workflows.md:279-722`）。
5. `testing-strategies.md`：7 层测试策略与自动化脚本模板（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:9-700`）。
6. `documentation-patterns.md`：命令内联文档、README、版本/变更与维护注释模式（`plugins/plugin-dev/skills/command-development/references/documentation-patterns.md:9-739`）。
7. `marketplace-considerations.md`：分发兼容性、命名冲突规避、版本兼容与发布维护（`plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md:9-904`）。

在插件全局工作流中的职责定位：

1. 被 `command-development/README.md` 明确登记为 References 层资产（`plugins/plugin-dev/skills/command-development/README.md:43-111`）。
2. 被 `command-development/SKILL.md` 显式引用用于补充语法和插件特性细节（`plugins/plugin-dev/skills/command-development/SKILL.md:326,645,715,832-834`）。
3. 在 `/plugin-dev:create-plugin` 的“命令实现阶段”被间接消费：先加载 `command-development` 技能，再按这些参考规范写命令（`plugins/plugin-dev/commands/create-plugin.md:157-191`）。
4. 被 `plugin-validator` 与 `skill-reviewer` 的质检流程间接覆盖：前者校验命令文件字段和结构，后者检查 references 质量与资源组织（`plugins/plugin-dev/agents/plugin-validator.md:76-84`，`plugins/plugin-dev/agents/skill-reviewer.md:73-84`）。

## 功能点目的

### 1) 把“命令定义”变成可验证契约

`frontmatter-reference.md` 的目标是把 frontmatter 从“经验写法”变成“字段契约”：

1. 明确字段类型/默认值/适用场景（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:24-323`）。
2. 提供从 minimal 到 complex 的分层样例（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:324-414`）。
3. 给出常见错误与发布前检查项（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:416-463`）。

### 2) 把“插件命令”从单文件提示扩展为组件化编排

`plugin-features-reference.md` 的目标是定义插件场景下命令的工程化模式：

1. 自动发现与命名空间组织（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:15-53`）。
2. `${CLAUDE_PLUGIN_ROOT}` 的可移植路径协议（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:75-220`）。
3. 与 `Task`（agent）、skill 引用、hook 事件协同的模式（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:330-437`）。
4. 输入/资源/输出/错误校验模板（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:439-561`）。

### 3) 把“交互命令”从自由问答收敛为结构化协议

`interactive-commands.md` 的目标是为复杂选项采集建立统一问答协议：

1. 标准参数结构：`questions[]/question/header/options/multiSelect`（`plugins/plugin-dev/skills/command-development/references/interactive-commands.md:34-60`）。
2. 交互设计约束：每题 2-4 选项、每次 1-4 问题、自动 Other（`plugins/plugin-dev/skills/command-development/references/interactive-commands.md:62-67,235-240`）。
3. 典型交互模式：确认、多问卷、条件分支、迭代采集、验证循环（`plugins/plugin-dev/skills/command-development/references/interactive-commands.md:269-577`）。

### 4) 把“一次性命令”升级为可恢复工作流

`advanced-workflows.md` 的目标是支撑跨命令、跨阶段的可靠执行：

1. `.claude/*.local.md` 状态持久化（`plugins/plugin-dev/skills/command-development/references/advanced-workflows.md:281-330`）。
2. 锁文件防并发与 flag 协调（`plugins/plugin-dev/skills/command-development/references/advanced-workflows.md:363-448`）。
3. 失败回滚与 checkpoint 恢复（`plugins/plugin-dev/skills/command-development/references/advanced-workflows.md:550-602`）。

### 5) 把“测试建议”前置到命令设计期

`testing-strategies.md` 的目标是让命令在发布前具备最低可靠性门槛：

1. 从结构校验到集成联调的 7 级测试面（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:11-339`）。
2. 提供可复用脚本模板（`validate-command.sh`、`validate-frontmatter.sh`、`test-commands.sh`）与 pre-commit/CI 示例（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:34-452`）。
3. 补齐 UX、UAT、调试清单（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:545-700`）。

### 6) 把“文档”纳入命令生命周期

`documentation-patterns.md` 的目标是让命令具备自解释与可维护性：

1. 命令内嵌用途、参数、示例、故障排查、变更记录模板（`plugins/plugin-dev/skills/command-development/references/documentation-patterns.md:11-115,510-599`）。
2. README 配套模板（安装、配置、故障排查、支持渠道）（`plugins/plugin-dev/skills/command-development/references/documentation-patterns.md:601-687`）。
3. 文档发布检查清单（`plugins/plugin-dev/skills/command-development/references/documentation-patterns.md:722-739`）。

### 7) 把“本地可用”扩展到“可市场分发”

`marketplace-considerations.md` 的目标是覆盖未知用户与异构环境：

1. 跨平台、依赖检测、降级策略（`plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md:11-163`）。
2. 分发规范：命名空间、配置化、版本兼容与弃用策略（`plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md:294-470`）。
3. 发布治理：预发布清单、Beta 机制、更新通知（`plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md:726-904`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（目录级）

1. 触发与加载：用户请求创建/改造命令 -> `command-development` 被触发（`plugins/plugin-dev/skills/command-development/SKILL.md:1-5`，`plugins/plugin-dev/commands/create-plugin.md:157-191`）。
2. 规范下钻：核心规则在 `SKILL.md`，细节按需进入 `references/*.md`（`plugins/plugin-dev/skills/command-development/README.md:94-111`）。
3. 实现落地：按参考文档生成命令 Markdown（frontmatter + 指令体 + 校验/交互/状态）。
4. 质量门禁：结合测试策略、文档清单、分发清单进行发布前验收（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:592-635`，`plugins/plugin-dev/skills/command-development/references/documentation-patterns.md:722-737`，`plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md:728-769`）。

### B. 关键数据结构与协议

1. Frontmatter 协议（命令元数据）
   - 字段：`description`、`allowed-tools`、`model`、`argument-hint`、`disable-model-invocation`（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:24-323`）。
   - 特别点：`allowed-tools` 接受字符串或字符串数组；默认可继承会话权限（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:62-86`）。

2. AskUserQuestion 交互协议
   - 结构：`questions` 数组，每项含 `question/header/options/multiSelect`（`plugins/plugin-dev/skills/command-development/references/interactive-commands.md:34-60`）。
   - 约束：header 最长 12 字符、选项建议 2-4 个、单次调用 1-4 问（`plugins/plugin-dev/skills/command-development/references/interactive-commands.md:41,65-67,467-470`）。

3. 工作流状态协议
   - 载体：`.claude/plugin-name-workflow.local.md` 或 `.claude/deployment-state.local.md`（`plugins/plugin-dev/skills/command-development/references/advanced-workflows.md:286-313,663`）。
   - 字段示例：`workflow/stage/environment/commit/tests_passed/build_complete`（`plugins/plugin-dev/skills/command-development/references/advanced-workflows.md:289-297`）。
   - 并发控制：`.claude/deployment.lock`；恢复控制：`deployment-checkpoints.log`（`plugins/plugin-dev/skills/command-development/references/advanced-workflows.md:418-446,588-601`）。

4. 测试脚本协议（模板级）
   - 结构校验：`validate-command.sh`（frontmatter 标记数、扩展名、空文件检查）（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:34-71`）。
   - 字段校验：`validate-frontmatter.sh`（`model` 合法值、`allowed-tools` 存在性、描述长度）（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:82-124`）。
   - 聚合执行：`test-commands.sh` + pre-commit + GitHub Actions（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:343-452`）。

5. 发布/维护协议（文档与版本）
   - 版本变更注释与迁移说明模板（`plugins/plugin-dev/skills/command-development/references/documentation-patterns.md:512-555`）。
   - 兼容检查与弃用警告模板（`plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md:408-470`）。
   - 更新通知模板（`plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md:846-876`）。

### C. 关键命令模式（目录内定义）

1. 配置驱动命令：通过 `@${CLAUDE_PLUGIN_ROOT}/config/*.json` 注入配置执行（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:223-241`）。
2. 模板驱动命令：通过 `templates/*.md` 统一输出结构（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:245-263`）。
3. 交互向导命令：`AskUserQuestion` 多轮采集 + 条件追问（`plugins/plugin-dev/skills/command-development/references/interactive-commands.md:149-197,339-380`）。
4. 流水线命令：多脚本串行/并行 + 状态更新（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:267-288`，`plugins/plugin-dev/skills/command-development/references/advanced-workflows.md:244-277`）。
5. 可恢复命令：checkpoint + rollback + resume（`plugins/plugin-dev/skills/command-development/references/advanced-workflows.md:550-602`）。

## 关键代码路径与文件引用

### 1) 目标目录（本次研究对象）

1. `plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md`
2. `plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md`
3. `plugins/plugin-dev/skills/command-development/references/interactive-commands.md`
4. `plugins/plugin-dev/skills/command-development/references/advanced-workflows.md`
5. `plugins/plugin-dev/skills/command-development/references/testing-strategies.md`
6. `plugins/plugin-dev/skills/command-development/references/documentation-patterns.md`
7. `plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md`

### 2) 调用方（谁依赖这些 references）

1. `plugins/plugin-dev/skills/command-development/README.md`
   - 把 7 份 references 作为渐进披露主体（`plugins/plugin-dev/skills/command-development/README.md:43-111`）。
2. `plugins/plugin-dev/skills/command-development/SKILL.md`
   - 在 Bash/插件特性/收尾导航处显式指向 references（`plugins/plugin-dev/skills/command-development/SKILL.md:326,645,715,832-834`）。
3. `plugins/plugin-dev/commands/create-plugin.md`
   - Phase 5 要求加载 `command-development`，实际实现时会落到 references 细则（`plugins/plugin-dev/commands/create-plugin.md:157-191`）。
4. `plugins/plugin-dev/agents/skill-reviewer.md`
   - 评估技能是否正确利用 `references/`（`plugins/plugin-dev/agents/skill-reviewer.md:73-84,141-145`）。

### 3) 被调用方/相关上下文依赖（references 依赖谁）

1. 插件结构基线：
   - `plugins/README.md:49-61`（标准插件结构）
   - `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`（manifest 必须在 `.claude-plugin/plugin.json`）
   - `plugins/plugin-dev/skills/plugin-structure/references/component-patterns.md:11-16`（启动时先读 manifest 再发现组件）
2. 命令验证上下文：
   - `plugins/plugin-dev/agents/plugin-validator.md:76-84`（命令 frontmatter 检查规则）
3. Toolkit 全局定位：
   - `plugins/plugin-dev/README.md:7-17,135-152`（command-development 在 7 技能体系中的角色）

### 4) 配置/测试/脚本/文档相关路径

1. 参考文档内定义的配置与状态路径：
   - `.claude/plugin-name.local.md`（交互配置、分发配置）  
     见 `plugins/plugin-dev/skills/command-development/references/interactive-commands.md:124-142`、`plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md:362-405`。
   - `.claude/deployment-state.local.md`、`.claude/deployment.lock`、`.claude/deployment-checkpoints.log`  
     见 `plugins/plugin-dev/skills/command-development/references/advanced-workflows.md:81-120,418-446,588-601`。
2. 参考文档内给出的测试脚本与流水线路径（模板）：
   - `validate-command.sh`、`validate-frontmatter.sh`、`test-commands.sh`（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:34-386`）
   - `.git/hooks/pre-commit`、`.github/workflows/test-commands.yml`（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:388-452`）
3. 参考文档内给出的文档配套路径（模板）：
   - 命令 README 模板及支持链接（`plugins/plugin-dev/skills/command-development/references/documentation-patterns.md:601-687`）。

## 依赖与外部交互

### 1) 运行时工具/协议依赖

1. Claude Code 工具依赖：
   - `AskUserQuestion`（交互问答）
   - `Task`（命令触发 agent）
   - `Skill`（命令提示触发技能知识）
   - `Read/Write/Bash`（文件与 shell 交互）  
   证据：`plugins/plugin-dev/skills/command-development/references/interactive-commands.md:13-67`，`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:334-384`。
2. 命令 DSL 依赖：
   - frontmatter、`@file`、`!\`bash\``、`$1/$ARGUMENTS` 的语义由 command-development 主技能与 references 协同定义（`plugins/plugin-dev/skills/command-development/SKILL.md:195-327`，`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:22-323`）。

### 2) 系统命令与外部工具交互

1. 常见 shell/CLI：`git`, `gh`, `npm`, `node`, `kubectl`, `helm`, `test`, `grep`, `find`, `wc`（散见于 7 份 references 的 `allowed-tools` 与 `!\`` 片段）。
2. 发布/运维交互：`/plugin update plugin-name`、Issue 链接、Release Notes URL（`plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md:441-443,870-873`，`plugins/plugin-dev/skills/command-development/references/documentation-patterns.md:684-686`）。

### 3) 目录级边界（非常关键）

1. 本目录是“规范资产”，不是运行时代码模块；不会被编译，也没有自带可执行 `scripts/`。
2. `testing-strategies.md` 中脚本与 CI 都是模板文本，并未在目录内落地为真实脚本文件（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:343-452`）。
3. 因为是模式库，示例命令需按目标项目权限模型与工具链做裁剪后才能投入生产。

## 风险、边界与改进建议

### 风险 1（高）：插件 manifest 路径示例与官方结构存在冲突

现状：

1. `plugin-features-reference.md` 使用 `plugin-name/plugin.json` 作为 manifest 位置（`plugins/plugin-dev/skills/command-development/references/plugin-features-reference.md:20-25`）。
2. 插件结构规范强调必须是 `.claude-plugin/plugin.json`（`plugins/README.md:51-55`，`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:7-10`）。

影响：按 references 示例创建插件时可能放错 manifest 路径，导致插件无法被正确识别。

建议：统一 references 中所有目录树示例到 `.claude-plugin/plugin.json`，并在文档中加入“旧示例兼容说明”。

### 风险 2（高）：`allowed-tools` 规范与验证策略不一致

现状：

1. `frontmatter-reference.md` 明确允许 `allowed-tools` 为字符串或数组（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:62-86`）。
2. `plugin-validator.md` 规则描述偏向“应为数组”（`plugins/plugin-dev/agents/plugin-validator.md:82`）。

影响：合法命令可能在质量校验中被误报，导致开发流程摩擦。

建议：对齐 validator 规则为“字符串或数组均合法”，并在测试模板中加入双格式样例。

### 风险 3（中）：权限最小化原则与样例默认值存在张力

现状：

1. 多处样例使用 `Bash(*)`（如 `plugin-features-reference.md:129,154,230,274,317,528`；`marketplace-considerations.md:18,73`）。
2. 同目录又强调“尽量收敛权限”（`plugins/plugin-dev/skills/command-development/references/frontmatter-reference.md:124-127`）。

影响：用户直接复制样例易保留过宽权限。

建议：将示例默认改为命令前缀白名单（如 `Bash(git:*)`、`Bash(node:*)`），并把 `Bash(*)` 限定为“无法预先约束命令集”的特例。

### 风险 4（中）：测试策略“模板化”但缺少同目录可执行基线

现状：`testing-strategies.md` 提供了完整脚本模板，但目录中没有真实 `scripts/validate-*.sh` 产物（`plugins/plugin-dev/skills/command-development/references/testing-strategies.md:343-452`）。

影响：规范执行依赖人工复制粘贴，难形成稳定回归。

建议：在 `command-development/scripts/` 落地最小可执行版本，并让 references 改为“调用真实脚本 + 解释实现”。

### 风险 5（中）：README 状态段与 references 完成度不一致

现状：`command-development/README.md` 仍把 advanced/testing/documentation/marketplace 标为 “in progress”（`plugins/plugin-dev/skills/command-development/README.md:250-255`），但对应参考文档已存在且较完整。

影响：读者对目录成熟度判断偏低，降低采用意愿。

建议：更新 README 状态为“已提供，持续迭代”，并标注最近校验日期。

### 风险 6（低）：文档模板中的版本/时间戳存在历史锚点，易过期

现状：多个模板内固定了示例版本与日期（如 2025 年时间戳、版本号），若被直接复用可能造成维护噪音（`plugins/plugin-dev/skills/command-development/references/documentation-patterns.md:516-549`，`plugins/plugin-dev/skills/command-development/references/marketplace-considerations.md:418-427,855-863`）。

影响：真实项目文档可能携带无关历史信息，影响可信度。

建议：把固定值改为占位符（`${VERSION}`、`${DATE}`）并在 checklist 增加“替换示例占位符”项。

### 边界说明

1. 本目录仅定义“命令开发方法论与协议样式”，不提供执行引擎实现。
2. 文档中的 shell 片段、路径和插件组件名称主要是示例，不保证直接复制即可在任意仓库运行。
3. 可靠落地需同时满足：插件目录正确、工具权限正确、外部命令可用、相关组件（agents/skills/hooks/scripts）存在。
