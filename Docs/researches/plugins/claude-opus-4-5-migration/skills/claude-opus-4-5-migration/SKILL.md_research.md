# FILE `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md` 研究文档

## 场景与职责

`plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md` 是该插件的核心执行规范文件，负责把“从 Sonnet 4.0 / Sonnet 4.5 / Opus 4.1 迁移到 Opus 4.5”拆解为可执行步骤与约束，而不是提供可执行代码。

它在链路中的职责分层如下：

1. 上游调用方（谁让这个 skill 被发现和触发）
- Marketplace 注册将插件源目录暴露给 Claude Code：`.claude-plugin/marketplace.json:18-27`。
- 插件总览声明该插件的核心内容是 skill `claude-opus-4-5-migration`：`plugins/README.md:16`。
- 插件 README 提供用户触发语句（自然语言入口）：`plugins/claude-opus-4-5-migration/README.md:11-13`。
- Claude Code 对 `skills/` 目录自动发现 `SKILL.md`（机制说明）：`plugins/plugin-dev/skills/skill-development/SKILL.md:269-276`。

2. 下游被调用方（SKILL 执行时会引用什么）
- 参数配置细则：`plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/effort.md:1-70`。
- Prompt 修复片段库：`plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/prompt-snippets.md:1-106`。
- 用户代码仓库中的模型配置、API 调用与提示词文本（由 workflow 第 1 步搜索并修改）：`plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:12-17`。

3. 文件边界
- 插件目录仅 5 个文件，无 `commands/`、`agents/`、`hooks/`、`scripts/`（实测）：
  - `plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json`
  - `plugins/claude-opus-4-5-migration/README.md`
  - `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md`
  - `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/effort.md`
  - `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/prompt-snippets.md`

结论：该 SKILL 是“迁移策略协议层”，并不直接执行 API 调用或脚本；执行落地依赖 Claude 在用户仓库内按文本规范实施。

## 功能点目的

### 1. Frontmatter 触发语义
- `name` 与 `description` 定义 skill 名称与触发意图：`SKILL.md:1-4`。
- 目的：让 Claude 在“升级到 Opus 4.5”语义下精准加载该技能，而非误触其他模型迁移方案。

### 2. 六步迁移工作流
- 步骤包含：搜索、替换模型、移除不支持 beta、添加 effort、总结变更、给出后续支持提示：`SKILL.md:12-17`。
- 目的：将迁移控制在可审计闭环内（改前发现 -> 改动实施 -> 改后汇总）。

### 3. 多平台模型 ID 映射
- 目标模型串覆盖 Anthropic/AWS Bedrock/Google Vertex/Azure：`SKILL.md:33-38`。
- 来源模型串覆盖 Sonnet 4.0、Sonnet 4.5、Opus 4.1：`SKILL.md:42-47`。
- 目的：减少跨云平台迁移时的 ID 格式错误。

### 4. 兼容性约束
- 明确移除 `context-1m-2025-08-07` beta header 并保留注释：`SKILL.md:25-29`。
- 明确 Haiku 4.5 不在迁移范围：`SKILL.md:48`。
- 目的：避免“误升级不兼容配置”和“误改不同产品线模型”。

### 5. 问题驱动的 Prompt 调整
- 默认只改模型串；仅在用户明确请求或反馈问题时才启用 prompt 修复：`SKILL.md:52`。
- 五类问题域：工具过触发、过度工程、代码探索不足、前端设计质量、thinking 词敏感：`SKILL.md:60-99`。
- 目的：控制迁移变更面，避免一次迁移引入大量行为变化。

### 6. 外部细则解耦
- `effort` 配置放入 `references/effort.md`：`SKILL.md:15,105`。
- 各类 prompt 片段放入 `references/prompt-snippets.md`：`SKILL.md:79,85,91,103`。
- 目的：保持 SKILL 主体简洁，细节按需加载（符合 progressive disclosure 思路）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（执行视角）

1. 触发与加载
- Claude Code 先加载 metadata（name/description），在匹配用户请求后加载 SKILL body，再按需读取 references：`plugins/plugin-dev/skills/skill-development/SKILL.md:79-84,269-276`。

2. 基线迁移
- 在用户仓库检索模型串/API 调用：`SKILL.md:12`。
- 按平台映射替换目标模型 ID：`SKILL.md:33-47`。
- 清理不支持 beta header：`SKILL.md:25-29`。
- 输出完整变更摘要并提醒后续可继续调优：`SKILL.md:16-17`。

3. 条件化二次修复
- 当且仅当用户反馈具体问题，再从 `prompt-snippets.md` 注入对应片段：`SKILL.md:52,79,85,91,103`。
- 注入时要求“按原 prompt 结构融合”，而不是简单末尾追加：`SKILL.md:54-58`、`prompt-snippets.md:101-106`。

### B. 关键数据结构

1. Skill 元数据结构（YAML frontmatter）
- 字段：`name`、`description`。
- 作用：用于技能发现与触发匹配（执行入口）。

2. 模型迁移映射表（Markdown table）
- `Platform -> TargetModelString`：`SKILL.md:33-38`。
- `SourceModel -> {Anthropic,AWS,Vertex}`：`SKILL.md:42-47`。
- 排除规则：`Do NOT migrate Haiku`：`SKILL.md:48`。

3. 问题修复矩阵（条件 -> 动作）
- 每类问题提供 `Apply if` 条件和替换策略/片段：`SKILL.md:64-99`。
- 细化定义在 `prompt-snippets.md`（含 before/after 表和可嵌入片段）：`prompt-snippets.md:13-98`。

### C. 协议与参数

1. 模型 ID 协议
- Anthropic: `claude-opus-4-5-20251101`
- Bedrock: `anthropic.claude-opus-4-5-20251101-v1:0`
- Vertex: `claude-opus-4-5@20251101`
- Azure: `claude-opus-4-5-20251101`
- 来源：`SKILL.md:35-38`。

2. effort 参数协议（beta）
- 需要 beta 标识 `effort-2025-11-24`：`effort.md:17`。
- SDK 方式：`betas` + `output_config.effort`：`effort.md:21-42`。
- Raw API 方式：`anthropic-beta` + `output_config.effort`：`effort.md:47-55`。

3. thinking 判定协议
- 仅当请求中存在 `thinking` 参数时，扩展 thinking 才启用：`prompt-snippets.md:79-85`。
- 无 `thinking` 参数时，若出现“think”相关问题，建议词汇替换：`SKILL.md:95-99`、`prompt-snippets.md:87-97`。

### D. 落地命令（实操建议）

`SKILL.md` 没有内置脚本命令，实操通常依赖通用检索/替换命令。可复用的最小命令集：

```bash
# 1) 扫描模型串与相关参数
rg -n "claude-(sonnet|opus|haiku)-4|anthropic\.claude-(sonnet|opus)-4|claude-(sonnet|opus)-4@|context-1m-2025-08-07|effort-2025-11-24|output_config|thinking"

# 2) 聚焦 API 调用位置（示例：messages.create / model 字段）
rg -n "messages\.create|model\s*[:=]|anthropic-beta|betas"

# 3) 变更后复扫，确认旧 ID 已清理且 Haiku 未误改
rg -n "claude-sonnet-4-20250514|claude-sonnet-4-5-20250929|claude-opus-4-1-20250422"
```

说明：以上命令属于执行建议，不是插件内置脚本。

## 关键代码路径与文件引用

### 目标文件
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:1-105`

### 直接依赖（被调用方）
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/effort.md:1-70`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/prompt-snippets.md:1-106`

### 触发与发现路径（调用方）
- `.claude-plugin/marketplace.json:18-27`
- `plugins/README.md:16`
- `plugins/claude-opus-4-5-migration/README.md:7-13`
- `plugins/plugin-dev/skills/skill-development/SKILL.md:269-276`

### 插件配置文件
- `plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:1-9`

### 文档与治理关联
- 根入口：`README.md:48-50`
- 研究清单（本任务项）：`Docs/researches/blueprint_checklist.md:160-164`
- todo 生成脚本：`.ops/generate_daily_research_todo.sh:1-42`

### 测试/脚本现状
- 插件内无测试文件、无执行脚本（仅文档与配置文件）。

## 依赖与外部交互

### 1) 调用方依赖
- 依赖 marketplace 注册正确映射插件目录，否则 skill 不可被安装发现：`.claude-plugin/marketplace.json:25`。
- 依赖 skill frontmatter 命名与用户意图匹配，否则触发概率下降：`SKILL.md:2-4`。

### 2) 被调用方依赖
- `references/effort.md` 提供 effort 参数的实现细节（SDK/Raw API）：`effort.md:17-55`。
- `references/prompt-snippets.md` 提供 5 类问题的可注入片段：`prompt-snippets.md:5-106`。

### 3) 配置依赖
- 插件级配置：`plugin.json`（name/version/description/author）用于元信息管理：`plugin.json:2-8`。
- 迁移级配置：模型 ID、beta header、output_config、thinking 参数规则分别分布在 `SKILL.md` 与 references。

### 4) 测试依赖
- 当前无内建自动化测试或回归脚本；验证依赖人工复扫、diff 审阅与运行时试调用。

### 5) 脚本依赖
- 插件本身无脚本依赖。
- 本研究流程依赖仓库治理脚本 `.ops/generate_daily_research_todo.sh` 重新生成当天待办。

### 6) 外部交互
- 面向外部平台协议：Anthropic API / AWS Bedrock / Google Vertex / Azure 的模型 ID 格式：`SKILL.md:35-38`。
- 外部文档跳转：Claude 4 提示词最佳实践链接在插件 README：`plugins/claude-opus-4-5-migration/README.md:17`。

## 风险、边界与改进建议

### 风险

1. 规则歧义风险（effort 是否默认改）
- workflow 第 4 步写“添加 effort=high”：`SKILL.md:15`。
- 参考段落又写“effort 仅用户请求时配置”：`SKILL.md:105`。
- 二者会导致执行策略不一致。

2. 版本时效风险
- 目标模型 ID 与 beta 标识均是固定日期版本：`SKILL.md:35-38`、`effort.md:17`。
- 上游版本更新后，skill 可能过时且静默失效。

3. 误修改风险
- 规范偏文本替换，若缺少“运行路径筛选”，可能改到示例文档/注释/历史记录。

4. 覆盖不足风险
- 来源映射表未列 Azure 的旧模型串（仅给目标 Azure 串）：`SKILL.md:38,42-47`。
- 在 Azure 场景下可能出现“目标有定义、来源未穷举”的漏迁移。

5. 无自动回归风险
- 没有测试或脚本验证“是否清理完旧模型串、是否误改 Haiku、是否保留必要参数”。

### 边界

1. 该 skill 是文档协议，不直接发起 API 请求。
2. 该插件不提供命令/agent/hook；全部行为由 skill 指导 Claude 在用户仓库内执行。
3. 默认策略是“先模型迁移，后按需 prompt 调整”，并非一次性全面重写提示词。

### 改进建议

1. 统一 effort 策略
- 在 `SKILL.md` 明确单一规则：`默认添加` 或 `仅用户请求添加`，避免歧义。

2. 增加“迁移前审计清单”步骤
- 在正式改写前输出命中清单（文件、旧值、平台归属），供用户确认后再批量替换。

3. 完善 Azure 来源映射
- 在 `Source Model Strings to Replace` 增加 Azure 旧 ID 样式，补齐跨平台对称性。

4. 增加轻量验证脚本
- 新增可选 `scripts/verify-migration.sh`（仅 `rg` 级别）自动检查旧模型残留、Haiku 误改、beta 头状态。

5. 加入版本维护注记
- 在 SKILL 或 README 增加“最后验证日期/适配模型版本”，降低文档老化风险。
