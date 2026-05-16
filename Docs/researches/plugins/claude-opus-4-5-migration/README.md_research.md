# FILE `plugins/claude-opus-4-5-migration/README.md` 研究文档

## 场景与职责

`plugins/claude-opus-4-5-migration/README.md` 是该插件的人类入口文档，职责不是执行迁移，而是把“何时触发、触发后做什么、去哪里看更细规则”压缩成最短说明。

它在系统中的定位是“能力声明层”，上接插件发现链路，下接 skill 执行协议：

1. 上游调用方（让该 README 有意义的入口）
- 仓库根 README 把用户导向插件目录总览：`README.md:48-50`。
- 插件总览表声明该插件提供迁移 skill：`plugins/README.md:16`。
- marketplace 注册插件源目录，决定可安装性与可发现性：`.claude-plugin/marketplace.json:18-27`。

2. 下游被调用方（README 描述的能力实际由谁实现）
- 插件元数据：`plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:1-9`。
- 核心迁移规则：`plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:1-105`。
- 参考细则：
  - `.../references/effort.md:1-70`
  - `.../references/prompt-snippets.md:1-106`

3. README 自身承担的最小职责
- 说明迁移目标范围（Sonnet 4.x / Opus 4.1 -> Opus 4.5）：`plugins/claude-opus-4-5-migration/README.md:3`。
- 给出可触发的自然语言示例：`plugins/claude-opus-4-5-migration/README.md:11-13`。
- 给出外部最佳实践链接：`plugins/claude-opus-4-5-migration/README.md:17`。

结论：该 README 不是“迁移引擎”，而是“触发契约 + 导航入口”。

## 功能点目的

### 1) 插件能力一句话定位
- 内容：`Migrate your code and prompts from Sonnet 4.x and Opus 4.1 to Opus 4.5.`（`README.md:3`）
- 目的：将复杂迁移任务压缩为明确升级意图，降低首次使用决策成本。

### 2) Overview 对“自动化范围”的声明
- 内容强调处理 model strings、beta headers 与配置细节（`README.md:7`）。
- 目的：让用户预期“不是单纯替换模型名”，而是包含协议兼容项（例如 beta 头处理）。
- 对应落实位置：
  - model 映射表：`SKILL.md:31-47`
  - 不兼容 beta 清理：`SKILL.md:23-29`

### 3) Usage 触发样例
- 触发语句：`"Migrate my codebase to Opus 4.5"`（`README.md:11-13`）。
- 目的：提供可直接复制的最小提示，降低用户与 skill 触发器之间的语义偏差。
- 与 skill frontmatter 的一致性：`SKILL.md:2-4` 的 description 明确“用户希望更新到 Opus 4.5 时使用”。

### 4) Learn More 外部文档跳转
- 指向 Claude 4 prompting best practices（`README.md:17`）。
- 目的：把“迁移后 prompt 优化”交给官方文档，README 自身保持精简。

### 5) Authors 元信息公开
- 作者信息：`README.md:19-21`；与 `plugin.json` 作者一致（`plugin.json:5-8`）。
- 目的：提供责任归属与维护沟通线索。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（从 README 触发到实际迁移）

1. 插件发现
- marketplace 将插件名映射到本地源码目录：`.claude-plugin/marketplace.json:18-27`。
- 插件目录满足标准结构（含 `.claude-plugin/plugin.json`、`skills/`、`README.md`）：`plugins/README.md:49-61`。

2. skill 发现与装载
- Claude Code 自动扫描 `skills/`，识别包含 `SKILL.md` 的子目录：`plugins/plugin-dev/skills/skill-development/SKILL.md:269-276`。
- 常驻元数据是 `name + description`；正文按触发加载：`.../skill-development/SKILL.md:79-84`。

3. 用户触发
- 用户使用 README 提供的自然语言意图进行请求（`README.md:11-13`）。
- 触发后加载迁移 skill 协议（`SKILL.md:1-4`）。

4. 执行迁移工作流（由 SKILL.md 规定）
- 搜索代码库内模型串/API 调用：`SKILL.md:12`。
- 按平台映射替换目标模型串：`SKILL.md:31-47`。
- 移除不支持 beta：`context-1m-2025-08-07`（`SKILL.md:25-29`）。
- 输出变更总结及后续支持提示：`SKILL.md:16-17`。

5. 问题驱动的二次 prompt 修复（可选）
- 仅在用户明确反馈问题时启用 `prompt-snippets`：`SKILL.md:52`、`prompt-snippets.md:3`。
- 五类问题域：工具过触发、过度工程、代码探索不足、前端设计质量、thinking 词敏感（`SKILL.md:60-99`）。

### B. 关键数据结构

1. 插件元数据（JSON）
- 文件：`plugin.json:1-9`
- 字段：`name`、`version`、`description`、`author`。
- 用途：插件识别与展示，不承载执行逻辑。

2. 技能协议（Markdown + YAML frontmatter）
- frontmatter：`name`、`description`（`SKILL.md:1-4`）。
- body：迁移流程、模型映射表、条件化修复策略（`SKILL.md:10-105`）。

3. 映射表结构（逻辑层）
- `Platform -> TargetModel`：4 平台目标 ID（`SKILL.md:33-38`）。
- `SourceModel -> SourceIDs`：3 类来源模型（`SKILL.md:42-47`）。
- 排除规则：Haiku 不迁移（`SKILL.md:48`）。

4. 条件策略结构（issue -> action）
- 每个问题域包含 `Apply if` 条件 + 修复指令/片段来源（`SKILL.md:64,79,85,89,97`，`prompt-snippets.md:5-106`）。

### C. 协议与命令层特征

1. 该插件没有自定义 slash command
- 入口是自然语言触发 skill，而不是 `/xxx` 命令。
- 目录下无 `commands/`、`agents/`、`hooks/`。

2. 该插件没有执行脚本
- `find plugins/claude-opus-4-5-migration -maxdepth 4 -type f` 仅 5 个文件（README、plugin.json、SKILL、2 个 references）。
- 无 `.sh/.py/.ts` 执行体，迁移动作由 Claude 在用户代码库执行。

3. API 参数与协议点
- 不支持项：`context-1m-2025-08-07`（`SKILL.md:25-29`）。
- 可选性能项：`output_config.effort` + `effort-2025-11-24` beta（`effort.md:17-56`）。
- thinking 判定条件：是否存在 `thinking` 参数（`prompt-snippets.md:79-85`）。

## 关键代码路径与文件引用

### 目标对象（本次 FILE）
- `plugins/claude-opus-4-5-migration/README.md:1-21`

### 同插件直接依赖
- `plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:1-9`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:1-105`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/effort.md:1-70`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/prompt-snippets.md:1-106`

### 发现链路与索引文档（调用方）
- `README.md:48-50`
- `plugins/README.md:13-27`
- `.claude-plugin/marketplace.json:18-27`

### skill 机制说明（运行上下文）
- `plugins/plugin-dev/skills/skill-development/SKILL.md:79-84`
- `plugins/plugin-dev/skills/skill-development/SKILL.md:269-276`

### 研究治理相关（配置/脚本）
- checklist：`Docs/researches/blueprint_checklist.md:160-164`
- 每日 todo 生成脚本：`.ops/generate_daily_research_todo.sh:1-42`

### 测试与脚本现状
- 插件目录内无测试文件（`tests/`/`test/`）与执行脚本；属于文档驱动 skill 插件。

## 依赖与外部交互

### 1) 内部依赖
- 对插件系统的依赖：需要 Claude Code 的插件发现与 skill 调度机制。
- 对文档一致性的依赖：README 的能力声明必须与 SKILL/references 同步。
- 对 marketplace 的依赖：条目决定安装/分发时能否找到该插件。

### 2) 外部交互
- 外部文档：README 跳转到 `platform.claude.com` 的 prompting guide（`README.md:17`）。
- 外部模型平台：迁移后的模型 ID 覆盖 Anthropic API、AWS Bedrock、Google Vertex、Azure（`SKILL.md:35-38`）。
- 外部 API 协议耦合：`effort-2025-11-24` beta 与 `output_config.effort`（`effort.md:17-53`）。

### 3) 与用户代码库的交互
- 插件自身不直接调用 API；它驱动 Claude 去修改“用户工程内模型配置与提示词文本”。
- 改写对象通常包括 SDK 调用参数、模型 ID 字符串、beta headers 与 prompt 内容。

### 4) 配置/测试/脚本/文档依赖结论
- 配置：`plugin.json` + `SKILL.md` frontmatter。
- 测试：无内建自动化验证。
- 脚本：无执行脚本，行为依赖运行时模型遵循度。
- 文档：README 为入口，SKILL/references 为执行规范主体。

## 风险、边界与改进建议

### 风险

1. 规则冲突风险（effort 默认策略）
- `SKILL.md:15` 写“迁移时添加 effort=high”，但 `SKILL.md:105` 又写“仅用户请求时配置 effort”。
- 这会导致执行者出现不一致行为，README 的“automates... configuration details”（`README.md:7`）也会被不同解释。

2. 时效性风险（硬编码版本）
- 模型 ID 与 beta 标识均为日期版本（如 `20251101`、`effort-2025-11-24`）。
- 上游平台版本变更后，README 声明仍可能看似正确但实际迁移目标已过期。

3. 误替换风险
- 当前规范偏字符串替换，若用户仓库中存在注释/文档样例/历史配置，可能误改非运行路径。

4. 无回归保障风险
- 插件内无脚本化检查，无法自动验证“替换是否完整且未伤及无关文本”。

5. 覆盖边界风险
- 未定义自定义 model alias、企业网关代理、多环境动态拼接模型 ID 的迁移策略。

### 边界

1. 该 README 不执行迁移，只描述迁移能力。
2. 该插件不是代码执行插件，不包含命令、agent、hook 或脚本。
3. 迁移质量依赖运行时模型遵循文档程度与人工审阅。

### 改进建议

1. 消除 effort 语义冲突
- 在 `SKILL.md` 明确单一策略：
  - 方案 A：默认添加 effort；
  - 方案 B：默认不改，用户请求才改。
- 保持 README 与 SKILL 语义一致。

2. 引入“先审计后改写”流程
- 先输出命中清单（文件、行、旧值->新值）再执行批量替换，降低误改风险。

3. 增加最小验证脚本
- 在插件内新增轻量 `scripts/`（例如基于 `rg` 的迁移前后快照对比）以支持复核。

4. 增补版本维护元信息
- 在 README 或 SKILL 加“最后验证日期/目标模型版本”，并纳入发布检查。

5. 明确非覆盖场景
- 在 README 增加“不处理 alias/router 场景”的边界说明，并给出人工排查建议。
