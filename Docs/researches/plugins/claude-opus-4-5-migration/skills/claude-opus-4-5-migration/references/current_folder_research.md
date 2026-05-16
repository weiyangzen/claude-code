# plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references 目录研究（DIR）

## 场景与职责

该目录是 skill `claude-opus-4-5-migration` 的“细节知识层”，用于承接 `SKILL.md` 主流程之外的参数说明与问题修复片段。

目录角色：
- 作为 `SKILL.md` 的被调用参考：
  - 迁移流程第 4 步引用 `references/effort.md`（`plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:15`）。
  - 问题修复章节多处要求从 `references/prompt-snippets.md` 取片段（`.../SKILL.md:79,85,91,103`）。
  - `SKILL.md` 末尾再次指向两份 references（`.../SKILL.md:103-105`）。
- 作为插件能力链路中的中间层：用户自然语言触发迁移意图（`plugins/claude-opus-4-5-migration/README.md:11-13`），skill 触发后按需读取 references。
- 在插件系统中属于“按需加载资源”：metadata 常驻、SKILL 主体按触发加载、references 按需要加载（`plugins/plugin-dev/skills/skill-development/SKILL.md:271-277`）。

目录边界：
- 当前目录仅有 2 个文档文件：`effort.md`、`prompt-snippets.md`。
- 不包含可执行脚本、测试用例、命令定义或配置清单（插件目录文件总数为 5，均为文档或元数据）。

## 功能点目的

### 1) `effort.md`：迁移后推理力度参数指南

核心目的：
- 定义 `output_config.effort` 的语义（影响 thinking、文本响应、函数调用 token 消耗）（`references/effort.md:7`）。
- 给出 `high/medium/low` 三档选择建议（`.../effort.md:9-14`）。
- 提供 Python / TypeScript SDK 与 Raw API 的可复制配置模板（`.../effort.md:19-56`）。
- 强调与 thinking budget 独立（`.../effort.md:58-64`）。

定位：这是“参数与协议接入文档”，不是迁移执行逻辑。

### 2) `prompt-snippets.md`：问题驱动的提示词修复库

核心目的：
- 明确默认策略：**默认只改模型串，不默认改 prompt**（`references/prompt-snippets.md:3`）。
- 覆盖 5 类已知差异问题并给出修复材料：
  1. 工具过触发（`.../prompt-snippets.md:5-19`）
  2. 过度工程（`.../prompt-snippets.md:20-33`）
  3. 代码探索不足（`.../prompt-snippets.md:35-45`）
  4. 前端设计质量（`.../prompt-snippets.md:47-73`）
  5. thinking 词敏感（`.../prompt-snippets.md:75-98`）
- 给出集成规范：片段需要与原 prompt 结构融合、使用 XML 标签、保持原有功能内容（`.../prompt-snippets.md:99-106`）。

定位：这是“症状 -> 干预片段”知识库，用于二次调优。

### 3) 与主 Skill 的协同目的

- `SKILL.md` 负责迁移主流程和触发条件；references 负责提供细粒度材料。
- 这种拆分符合“progressive disclosure”原则：核心流程保持精简，细节按需读取（`plugins/plugin-dev/README.md:17`，`plugins/plugin-dev/README.md:264-269`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

该目录本身不执行代码，实际是“文档驱动决策流”：

1. 迁移触发后，执行者先按 `SKILL.md` 完成模型串与 beta 处理（`.../SKILL.md:12-17,23-47`）。
2. 若涉及 effort 参数，读取 `effort.md` 获取字段与 beta 的写法（`.../SKILL.md:15,105` + `references/effort.md:17-56`）。
3. 若用户报告特定行为问题，读取 `prompt-snippets.md` 选取对应片段并按结构化规则插入（`.../SKILL.md:52-59,79-99` + `references/prompt-snippets.md:99-106`）。
4. 输出修改摘要，标注模型串变更与 prompt 修改点（`references/prompt-snippets.md:106`）。

### 数据结构

1. 表格型策略数据
- `effort.md`：`Effort -> Use Case` 表（`references/effort.md:9-14`）。
- `prompt-snippets.md`：多处 `Before -> After` 替换对照表（`.../prompt-snippets.md:13-19,91-98`）。

2. 协议片段模板
- SDK/Raw API 请求模板以结构化字段示例呈现（`references/effort.md:19-56`）。
- thinking 启用条件使用 JSON 片段表述（`.../prompt-snippets.md:79-85`）。

3. 条件化规则结构
- 每个问题段落都包含 `Problem`、`When to add/apply`、`Solution/Snippet` 三元结构（`references/prompt-snippets.md:7-11,22-27,37-42,49-55,77-90`）。

### 协议

本目录内容直接约束 Claude API 请求字段：
- `model`（示例模型串）（`references/effort.md:22,35,48`）。
- `betas`（SDK）与 `anthropic-beta`（Raw API）（`.../effort.md:24,37,50`）。
- `output_config.effort`（`.../effort.md:25-27,38-40,51-53`）。
- `thinking` 对象是否存在用于判定“扩展思考已启用”（`references/prompt-snippets.md:79-85`）。

并与主 skill 的平台模型映射直接联动（Anthropic / Bedrock / Vertex / Azure）（`.../SKILL.md:33-38`）。

### 命令

- 该目录无任何可执行命令、脚本入口、hook 或 agent 定义。
- 迁移动作由触发 skill 的 Claude 执行，不由本目录提供执行器。

## 关键代码路径与文件引用

目录内核心文件：
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/effort.md:1-70`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/prompt-snippets.md:1-106`

直接调用方（上游）：
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:15,52-59,79,85,91,103-105`

插件级上下文：
- `plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:1-9`
- `plugins/claude-opus-4-5-migration/README.md:1-21`
- `plugins/README.md:16`
- `.claude-plugin/marketplace.json:18-27`
- `README.md:48-50`

规范上下文（references 的设计语义）：
- `plugins/plugin-dev/README.md:17,264-269`
- `plugins/plugin-dev/skills/skill-development/SKILL.md:61-66,271-277`

## 依赖与外部交互

### 内部依赖

1. 对 `SKILL.md` 的强依赖
- references 不独立触发，必须通过 `SKILL.md` 的流程与条件被消费。

2. 对插件装载链路的依赖
- marketplace 条目指向插件 source（`.claude-plugin/marketplace.json:25`）。
- 插件通过 `plugin.json` 提供元信息（`plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:2-8`）。

3. 对“按需加载”机制的依赖
- references 默认不常驻上下文，依赖运行时按需读取机制（`plugins/plugin-dev/skills/skill-development/SKILL.md:274-277`）。

### 外部交互

1. 模型平台与 API 协议耦合
- 文档中的模型串、beta 头和请求字段需要与 Anthropic/云平台接口保持一致（`references/effort.md:17-56`，`.../SKILL.md:33-38`）。

2. 外链文档依赖
- 插件 README 引用外部 prompting 指南作为补充实践依据（`plugins/claude-opus-4-5-migration/README.md:17`）。

### 测试、脚本、配置现状

- 配置：本目录无独立配置文件；配置语义由文档中字段示例表达。
- 测试：无测试样例与自动化校验。
- 脚本：无 `scripts/`、无执行脚本。

## 风险、边界与改进建议

### 风险

1. effort 策略冲突风险
- `SKILL.md` 第 4 步要求“添加 effort=high”（`.../SKILL.md:15`）。
- 同文件末尾又写“only if user requests it”（`.../SKILL.md:105`）。
- `effort.md` 也倾向默认添加 high（`references/effort.md:3-4`）。
- 风险结果：不同执行者可能做出相反迁移行为。

2. 版本时效风险
- 模型串与 beta 名称是日期硬编码（如 `20251101`、`effort-2025-11-24`），上游版本变更后会迅速老化（`.../SKILL.md:35-38`，`references/effort.md:17`）。

3. prompt 误改风险
- `prompt-snippets` 含长段强约束文本，若不按“仅问题触发”原则应用，可能改变原系统提示词行为边界（`references/prompt-snippets.md:3,99-105`）。

4. 关键词替换语义漂移风险
- thinking 敏感修复建议替换“think”词族（`.../prompt-snippets.md:89-98`），若做机械全局替换，可能误伤原语义或文档内容。

5. 与其它技能的规则漂移风险
- 前端美学片段与 `frontend-design` 插件目标高度重合，长期并行维护易发生规范不一致（`references/prompt-snippets.md:47-73`，`plugins/README.md:21`）。

### 边界

1. 默认迁移边界
- 默认动作是模型迁移，不自动注入全部提示词片段（`references/prompt-snippets.md:3`）。

2. 能力边界
- 本目录只提供知识与模板，不直接执行代码改写。

3. 模型边界
- 主 skill 明确不迁移 Haiku（`.../SKILL.md:48`）；references 不扩展该边界。

### 改进建议

1. 统一 effort 默认策略
- 在 `SKILL.md` 与 `effort.md` 保持同一条默认规则，避免“总是加”与“按需加”并存。

2. 增加可维护映射源
- 将模型串和 beta 标识抽为集中配置或版本表，降低文档多处硬编码维护成本。

3. 增补最小验证资产
- 增加 `before/after` 示例与检查脚本（如残留旧模型串、误改 Haiku 检查），为文档规则提供可执行校验。

4. 强化插入策略模板
- 为每类 snippet 增加“短版/完整版 + 推荐插入位置”模板，减少人工整合差异。

5. 与 `frontend-design` 建立联动约束
- 将重叠前端规则改为“引用单一来源”或维护同步策略，降低双份规范漂移。
