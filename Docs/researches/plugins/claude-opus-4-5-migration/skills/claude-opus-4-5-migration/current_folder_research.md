# plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration 目录研究（DIR）

## 场景与职责

该目录是插件 `claude-opus-4-5-migration` 的核心执行知识体，属于“纯 Skill 驱动”实现：

- 目录职责：定义从 Sonnet 4.0 / Sonnet 4.5 / Opus 4.1 到 Opus 4.5 的迁移策略与操作顺序（`plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:1-105`）。
- 调用方（入口上下文）：
  - marketplace 注册项把插件暴露给 Claude Code（`.claude-plugin/marketplace.json:18-27`）。
  - 插件 README 给出用户触发语句（`plugins/claude-opus-4-5-migration/README.md:9-13`）。
  - 插件总览表声明该插件提供 skill `claude-opus-4-5-migration`（`plugins/README.md:13-17`）。
- 被调用方（目录内依赖）：
  - `references/effort.md`：effort 参数与 beta 接入说明（`.../references/effort.md:1-70`）。
  - `references/prompt-snippets.md`：按问题类型注入的提示词片段（`.../references/prompt-snippets.md:1-106`）。

该目录不包含可执行脚本、测试代码、命令定义或 agent 配置，全部能力通过 Markdown 规范驱动。

## 功能点目的

1. 统一模型串迁移
- 将 1P / Bedrock / Vertex / Azure 的旧模型 ID 替换为 Opus 4.5 目标 ID（`SKILL.md:31-47`）。

2. 清理不兼容配置
- 移除 `context-1m-2025-08-07` beta 头，并保留解释注释（`SKILL.md:23-29`）。

3. 规范输出与回访
- 要求迁移后总结修改项，并输出固定回访语句以便后续 prompt 调整（`SKILL.md:16-17`）。

4. 按需处理行为差异
- 仅在用户明确请求或报告问题时，才应用工具过触发、过度工程、代码探索、前端设计、thinking 敏感性等修复（`SKILL.md:50-99`，`prompt-snippets.md:3-106`）。

5. effort 参数策略参考
- 提供 `output_config.effort` 配置及 `effort-2025-11-24` beta 依赖（`effort.md:17-56`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

目录定义的是“人工/代理执行流程规范”，不是可执行程序：

1. 在目标代码库检索模型字符串与 API 调用（`SKILL.md:12`）。
2. 按平台映射替换为 Opus 4.5 模型串（`SKILL.md:31-47`）。
3. 移除 `context-1m-2025-08-07` 并写注释（`SKILL.md:23-29`）。
4. 处理 effort 参数（`SKILL.md:15` + `SKILL.md:105` + `effort.md:3-4`，存在策略冲突，见风险章节）。
5. 用户若报告问题，再按症状选择 `prompt-snippets.md` 对应片段并“结构化嵌入”原 prompt（`SKILL.md:52-59`，`prompt-snippets.md:99-106`）。

### 数据结构

1. Skill frontmatter
- `name`/`description` 作为触发与语义匹配配置（`SKILL.md:1-4`）。

2. 模型映射表
- `Platform -> Opus 4.5 Model String`（`SKILL.md:33-38`）。
- `Source Model -> 各平台旧 ID`（`SKILL.md:42-46`）。

3. 策略片段目录
- `prompt-snippets.md` 以“问题-触发条件-片段/替换规则”三段式组织（`prompt-snippets.md:5-106`）。

### 协议与参数

1. 模型协议字段
- `model` 字段在不同平台使用不同命名格式（`SKILL.md:35-38`）。

2. beta 与输出参数
- SDK 方案使用 `betas` + `output_config.effort`（`effort.md:21-42`）。
- Raw API 方案使用 `anthropic-beta` + `output_config.effort`（`effort.md:46-55`）。

3. thinking 判定协议
- 仅当请求内存在 `thinking` 对象时视为启用扩展思考（`prompt-snippets.md:79-85`）。

### 命令与脚本

- 该目录本身没有脚本或命令实现（`find plugins/claude-opus-4-5-migration -maxdepth 4 -type f` 仅返回 5 个文档/配置文件）。
- 研究流程相关命令来自仓库运维脚本（如 `.ops/generate_daily_research_todo.sh`），并非插件运行时依赖（`.ops/generate_daily_research_todo.sh:1-42`）。

## 关键代码路径与文件引用

核心路径：

- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:1-105`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/effort.md:1-70`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/prompt-snippets.md:1-106`

上游入口与配置：

- `.claude-plugin/marketplace.json:18-27`
- `plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:1-9`
- `plugins/claude-opus-4-5-migration/README.md:1-21`
- `plugins/README.md:13-17`

研究流程关联：

- `Docs/researches/blueprint_checklist.md:30-34`
- `.ops/generate_daily_research_todo.sh:1-42`

## 依赖与外部交互

内部依赖：

1. 目录内文档依赖
- `SKILL.md` 显式引用 `references/effort.md` 与 `references/prompt-snippets.md`（`SKILL.md:15,79,85,91,103-105`）。

2. 插件发现/装载依赖
- 通过 marketplace 的 `source` 指向插件根，再由插件内 `skills/` 目录提供能力（`.claude-plugin/marketplace.json:25`，`plugins/README.md:49-60`）。

外部交互：

1. 模型平台语义
- 迁移目标覆盖 Anthropic 1P、AWS Bedrock、Google Vertex、Azure AI Foundry（`SKILL.md:35-38`）。

2. API 协议字段
- 与 Claude API 请求体的 `model`、`betas` / `anthropic-beta`、`output_config`、`thinking` 字段直接耦合（`effort.md:21-55`，`prompt-snippets.md:79-85`）。

3. 外部文档链接
- README 外链到 Anthropic prompt best practices（`plugins/claude-opus-4-5-migration/README.md:17`）。

测试与质量保障现状：

- 当前无自动化测试、无示例输入输出样本、无校验脚本；质量主要依赖执行者遵循文字规范。

## 风险、边界与改进建议

主要风险：

1. effort 默认策略冲突
- `SKILL.md` 工作流第 4 步要求“添加 effort=high”（`SKILL.md:15`）。
- 同文件末尾又写“仅在用户请求时配置 effort”（`SKILL.md:105`）。
- `effort.md` 也写“迁移时添加 high”（`effort.md:3-4`）。
- 风险：执行者行为不一致，可能造成不必要参数变更或漏改。

2. 版本时效风险
- 模型串与 beta 标识为硬编码日期（如 `20251101`、`effort-2025-11-24`，见 `SKILL.md:35-38`、`effort.md:17`）。
- 风险：上游版本更新后，该 skill 会快速过时。

3. 文本替换误伤风险
- 当前仅给出“查找并替换”原则，没有规范化边界（例如注释/示例代码/文档中的模型串是否替换）。
- 风险：可能误改非运行配置，或遗漏变体拼写。

4. 无测试回归保护
- 没有 smoke test、示例仓库、或差异校验脚本。
- 风险：迁移结果难以批量验证，尤其在多平台混合配置仓库中。

边界：

1. 明确不迁移 Haiku（`SKILL.md:48`）。
2. prompt 修复默认不执行，仅在用户反馈问题时执行（`SKILL.md:52-53`）。
3. 该目录仅提供规范，不直接执行代码修改。

改进建议：

1. 统一 effort 策略
- 在 `SKILL.md` 与 `effort.md` 明确唯一默认行为（例如“默认不改 effort，除非用户请求”或相反）。

2. 增加可维护版本策略
- 在文档中增加“版本来源与更新时间”段，或通过集中配置表维护模型 ID 与 beta 标识。

3. 补充迁移边界规则
- 新增“只改可执行配置文件，不改注释/README/历史日志”的默认策略与例外清单。

4. 增加最小验证资产
- 提供 `before/after` 示例与校验脚本（例如检查是否残留旧模型串、是否误改 Haiku）。

5. 将 prompt 片段模块化
- 为每类问题提供“短版/长版”片段和插入位置建议模板，降低执行分歧。
