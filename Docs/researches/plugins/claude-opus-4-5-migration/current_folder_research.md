# plugins/claude-opus-4-5-migration 目录研究（DIR）

## 场景与职责

`plugins/claude-opus-4-5-migration` 是一个“纯 Skill 型”迁移插件：不提供 slash command、agent、hook 或可执行脚本，而是通过 `SKILL.md` 指导 Claude 在用户代码库里完成模型迁移与提示词适配。

目录角色与注册位置：
- 仓库级插件入口：`README.md:48-50`
- 插件总览声明（本插件只暴露 1 个 Skill）：`plugins/README.md:13-17`
- marketplace 注册入口：`.claude-plugin/marketplace.json:18-27`
- 插件自身元数据：`plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:1-9`

目录构成（共 5 个文件）：
- `.claude-plugin/plugin.json`：插件标识/版本/作者元数据。
- `README.md`：用途、触发语句、外链文档。
- `skills/claude-opus-4-5-migration/SKILL.md`：迁移主流程与策略。
- `skills/.../references/effort.md`：`effort` 参数的 beta 接入说明。
- `skills/.../references/prompt-snippets.md`：Opus 4.5 已知行为差异的可选提示词片段。

职责边界：
1. 把 Sonnet 4.0 / Sonnet 4.5 / Opus 4.1 的模型字符串统一迁移到 Opus 4.5。
2. 清理不兼容 beta header（`context-1m-2025-08-07`）。
3. 按需（非默认）注入针对 Opus 4.5 行为差异的提示词修复策略。
4. 明确排除 Haiku 4.5 迁移（防止误改）。

## 功能点目的

### 1) 元数据与市场可发现性
- `plugin.json` 提供标准元信息（`name/version/description/author`），用于插件识别与展示：`plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:2-8`。
- marketplace 条目重复声明版本与作者信息，便于 `/plugin` 市场安装时展示：`.claude-plugin/marketplace.json:18-27`。

目的：确保该能力作为可安装插件被发现，而不是散落文档。

### 2) Skill 触发与单次迁移工作流
- `README` 给出自然语言触发例句：`"Migrate my codebase to Opus 4.5"`（`plugins/claude-opus-4-5-migration/README.md:11-13`）。
- `SKILL.md` frontmatter 描述触发场景：当用户希望更新 codebase/prompt/API call 到 Opus 4.5 时启用：`.../SKILL.md:1-4`。
- 迁移 workflow 固定 6 步：检索 -> 替换模型串 -> 去掉不支持 beta ->（文档建议）effort -> 变更总结 -> 用户后续支持语：`.../SKILL.md:10-17`。

目的：把“模型升级”从一次性口头操作变成可重复执行的结构化流程。

### 3) 跨平台模型字符串映射
- 目标模型串覆盖 4 个平台：Anthropic API / Bedrock / Vertex / Azure：`.../SKILL.md:31-38`。
- 源模型串覆盖 3 类来源（Sonnet 4.0 / Sonnet 4.5 / Opus 4.1）：`.../SKILL.md:40-47`。
- 明确禁止迁移 Haiku：`.../SKILL.md:48`。

目的：减少跨云/跨平台项目的手工遗漏与误替换。

### 4) 提示词适配（按需启用）
- 默认策略是“只改模型串”，提示词修复只在用户明确提出问题时应用：`.../SKILL.md:50-53`、`references/prompt-snippets.md:3`。
- 已定义 5 类问题域：工具过触发、过度工程、代码探索不足、前端设计质量、thinking 词敏感：`.../SKILL.md:60-99`。

目的：控制迁移变更面，避免默认引入行为性 prompt 大改。

### 5) `effort` 参数参考
- `references/effort.md` 说明 `output_config.effort` 及 `effort-2025-11-24` beta 头：`references/effort.md:15-56`。
- 同时给出 Python/TypeScript/Raw API 三种接入模板：`references/effort.md:19-56`。

目的：为“迁移后性能优化”提供可复制配置片段。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（调用链）

1. 插件发现/安装
- 入口由 marketplace `source: ./plugins/claude-opus-4-5-migration` 提供：`.claude-plugin/marketplace.json:25`。
- 插件本地根通过 `.claude-plugin/plugin.json` 识别：`plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:1-9`。

2. Skill 自动发现与触发
- 插件规范中，`skills/` 目录下含 `SKILL.md` 的子目录可被自动发现（通用机制）：`plugins/plugin-dev/skills/skill-development/SKILL.md:269-276`。
- metadata（name + description）常驻，正文按触发加载：`plugins/plugin-dev/skills/skill-development/SKILL.md:79-84`。

3. 迁移执行
- 在用户项目中搜索模型字符串与 API 调用：`plugins/claude-opus-4-5-migration/skills/.../SKILL.md:12`。
- 按平台映射替换目标模型串：`.../SKILL.md:31-47`。
- 清理不兼容 `context-1m-2025-08-07` beta，必要时加注释：`.../SKILL.md:23-29`。
-（文档流程）增加 `effort="high"`：`.../SKILL.md:15` + `references/effort.md:17-56`。
- 输出变更摘要并给出迁移后支持提示语：`.../SKILL.md:16-17`。

4. 问题导向的二次修复
- 若用户反馈具体问题，再定向加载 `prompt-snippets.md` 中对应段落并集成到现有 prompt 结构：`.../SKILL.md:52-59`、`references/prompt-snippets.md:99-106`。

### B. 数据结构与规则模型

1. 插件元数据结构（JSON）
- 字段：`name/version/description/author`。
- 文件：`plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:1-9`。

2. Skill 协议结构（Markdown + YAML frontmatter）
- frontmatter: `name`, `description` 决定触发语义：`plugins/claude-opus-4-5-migration/skills/.../SKILL.md:1-4`。
- body: 迁移步骤、映射表、条件式修复策略：`.../SKILL.md:10-105`。

3. 迁移映射表（逻辑数据结构）
- `Platform -> TargetModelString`（4 平台目标映射）：`.../SKILL.md:33-38`。
- `SourceModel -> per-platform source strings`（3 组来源映射）：`.../SKILL.md:42-47`。
- 排除清单：Haiku 不迁移：`.../SKILL.md:48`。

4. 条件策略（issue -> snippet）
- 每类问题有 `Apply if` 条件 + 对应修复片段来源：`.../SKILL.md:60-99`，`references/prompt-snippets.md:5-98`。

### C. 协议与命令特征

- 该插件无自定义 slash command；入口是自然语言意图触发 skill（README 示例）：`plugins/claude-opus-4-5-migration/README.md:11-13`。
- 运行期实际“命令执行”由 Claude 在用户仓库中完成（搜索、编辑、替换），不是插件目录内脚本驱动。
- 目录中无 `*.sh/*.py/*.ts/*.js` 执行脚本、无测试文件（静态内容插件）。

## 关键代码路径与文件引用

### 目录内核心文件
- `plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:1-9`
- `plugins/claude-opus-4-5-migration/README.md:1-21`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:1-105`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/effort.md:1-70`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/prompt-snippets.md:1-106`

### 上游调用方（谁会让它生效）
- 仓库层插件说明入口：`README.md:48-50`
- 插件列表中对本插件的能力声明：`plugins/README.md:16`
- marketplace 注册条目：`.claude-plugin/marketplace.json:18-27`
- Claude Code 对 skill 的自动发现机制（通用约定）：`plugins/plugin-dev/skills/skill-development/SKILL.md:269-276`

### 下游被调用方（它会影响谁）
- 用户工程中的模型配置与 prompt 文本（由迁移步骤直接改写）：`plugins/claude-opus-4-5-migration/skills/.../SKILL.md:12-17,31-48,50-99`
- Claude API 请求参数结构（`betas`, `output_config.effort`, `thinking`）：
  - `references/effort.md:17-56`
  - `references/prompt-snippets.md:79-85`
- 外部最佳实践文档链接：
  - `plugins/claude-opus-4-5-migration/README.md:17`

### 配置/测试/脚本/文档上下文
- 配置：仅 `plugin.json` 与 `SKILL.md` + `references/*.md`。
- 测试：无目录内自动化测试或回归样例。
- 脚本：无目录内可执行脚本。
- 文档：`README.md` + 2 份 reference 文档构成完整操作说明。

## 依赖与外部交互

### 1) 运行时依赖
- Claude Code 插件加载与 skill 调度机制（目录发现 + frontmatter 触发）。
- 用户项目代码库可写权限（执行替换/编辑）。

### 2) API 协议耦合
- 强耦合 Opus 4.5 目标模型 ID：`claude-opus-4-5-20251101` 及各云厂商格式：`SKILL.md:35-38`。
- 强耦合 beta flag：
  - 不支持项：`context-1m-2025-08-07`（需移除）：`SKILL.md:25-29`
  - 新能力项：`effort-2025-11-24`（需添加才能使用 effort）：`references/effort.md:17,24,37,50`

### 3) 外部资源交互
- 文档跳转到 Anthropic 平台提示词指南：`plugins/claude-opus-4-5-migration/README.md:17`。
- 迁移后代码会调用外部模型服务（Anthropic/Bedrock/Vertex/Azure），但插件本身不直接发请求，只提供改写规范。

### 4) 与其他插件/能力的隐式关联
- `prompt-snippets` 中“前端设计质量”段落与 `frontend-design` 插件目标一致，存在知识重叠：
  - 本插件：`references/prompt-snippets.md:47-73`
  - 前端插件技能定位：`plugins/frontend-design/skills/frontend-design/SKILL.md:1-42`

## 风险、边界与改进建议

### 风险与边界

1. 指令一致性风险：`SKILL.md` 第 4 步写“添加 effort=high”（`SKILL.md:15`），但同文件末尾又写“effort 仅在用户请求时配置”（`SKILL.md:105`）。这会导致执行者对“默认是否改 effort”理解不一致。

2. 版本时效风险：模型串与 beta 标识是硬编码日期（如 `20251101`、`effort-2025-11-24`），一旦平台升级，skill 可能快速过时。

3. 误替换边界：当前规则以字符串替换为主，若用户代码里存在注释、文档示例、多环境并存配置，可能误改非运行路径。

4. 无自动化验证：插件目录无脚本与测试，无法在仓库内做“迁移前后 diff 正确性”回归，只能依赖人工审阅与运行验证。

5. 平台覆盖不完全：映射表覆盖 1P/Bedrock/Vertex/Azure 常见 ID，但未覆盖自定义 model alias 或代理网关场景（例如企业内部 model router）。

### 改进建议

1. 统一 effort 策略
- 在 `SKILL.md` 明确二选一：
  - 默认总是添加 effort；或
  - 默认不改，只有用户要求才改。
- 消除 `SKILL.md:15` 与 `SKILL.md:105` 的语义冲突。

2. 增加“先审计后替换”约束
- 在 workflow 增加“先输出命中清单 + 待替换映射，再批量修改”的步骤，降低误替换风险。

3. 提供最小回归脚本
- 增加 `examples/` 或 `scripts/`（如 grep + snapshot 对比），验证模型串替换与 beta 头处理是否符合预期。

4. 建立版本更新钩子
- 在 README 增补“最近验证日期 + 目标模型版本”字段，并将更新动作纳入发布检查，避免硬编码长期陈旧。

5. 把前端提示词片段抽公共引用
- 当前与 `frontend-design` 技能存在重复，可考虑引用共享片段源，降低多处维护漂移。

