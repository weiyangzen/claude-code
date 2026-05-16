# plugins/claude-opus-4-5-migration/skills 目录研究（DIR）

## 场景与职责

`plugins/claude-opus-4-5-migration/skills` 是该插件的能力核心目录，承载一个可被 Claude Code 自动发现并触发的迁移技能：`claude-opus-4-5-migration`。

该目录在插件体系中的职责分层如下：
- 仓库/插件层对外宣告“这是一个 Skill 型插件”：
  - 根 README 的插件入口：`README.md:48-50`
  - 插件总览表中的能力声明：`plugins/README.md:16`
  - marketplace 注册此插件源路径：`.claude-plugin/marketplace.json:18-27`
- `skills/` 目录承接插件内部能力实现，遵循“子目录含 `SKILL.md` 即可被发现”的技能约定：`plugins/plugin-dev/skills/skill-development/SKILL.md:269-276`
- 本目录实际只包含 1 个技能实现与 2 个参考文档：
  - `skills/claude-opus-4-5-migration/SKILL.md`
  - `skills/claude-opus-4-5-migration/references/effort.md`
  - `skills/claude-opus-4-5-migration/references/prompt-snippets.md`

从业务目标看，该目录不是“执行脚本目录”，而是“迁移策略与规则目录”：通过文本化流程约束 Claude 在用户代码库里执行模型迁移、参数迁移和提示词修复。

## 功能点目的

### 1. 技能触发语义与迁移边界定义
- `SKILL.md` frontmatter 用 `name/description` 定义触发语义与适用场景：`plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:1-4`
- 明确支持从 Sonnet 4.0、Sonnet 4.5、Opus 4.1 升级到 Opus 4.5，并明确“不可迁移 Haiku 4.5”：`.../SKILL.md:8,48`

目的：让迁移行为具备可预期的触发条件与风险边界，避免“见到 Claude 模型串就一律替换”。

### 2. 一次性迁移主流程标准化
- 技能定义了固定 6 步流程（检索、替换、去 beta、effort、总结、回访提示）：`.../SKILL.md:10-17`

目的：把迁移操作从临时人工动作变成可复用 SOP，降低遗漏模型串/遗漏 beta 头的概率。

### 3. 跨平台模型字符串映射
- 明确给出 Opus 4.5 在 1P/Bedrock/Vertex/Azure 的目标模型串：`.../SKILL.md:33-38`
- 明确给出 Sonnet 4.0 / Sonnet 4.5 / Opus 4.1 三组来源模型串：`.../SKILL.md:42-47`

目的：在多云部署或多 SDK 共存代码库中，统一迁移口径，减少平台间不一致。

### 4. 参数与行为差异适配
- `references/effort.md` 给出 `effort` 参数目的、等级选择、以及 Python/TS/Raw API 配置模板：`.../references/effort.md:1-70`
- `references/prompt-snippets.md` 给出 5 类已知行为问题（工具过触发、过度工程、探索不足、前端质量、thinking 词敏感）及条件化修复片段：`.../references/prompt-snippets.md:5-106`

目的：把“迁移后行为调优”从经验口传沉淀为结构化知识，并保持“按需启用”而非默认大改提示词。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

1. 发现与触发
- Claude Code 自动扫描插件 `skills/` 目录，发现含 `SKILL.md` 的子目录并常驻加载技能元信息：`plugins/plugin-dev/skills/skill-development/SKILL.md:271-276`
- 用户通过自然语言迁移意图触发（插件 README 示例）：`plugins/claude-opus-4-5-migration/README.md:11-13`

2. 迁移执行
- 搜索代码库内模型字符串与 API 调用：`.../SKILL.md:12`
- 按平台映射将旧模型替换为 Opus 4.5：`.../SKILL.md:31-47`
- 删除不支持的 `context-1m-2025-08-07` beta header，并留下注释说明：`.../SKILL.md:23-29`
- 根据策略处理 effort：
  - 工作流步骤写的是“迁移时添加 `effort=high`”：`.../SKILL.md:15`
  - 参考区写的是“仅在用户请求时配置 effort”：`.../SKILL.md:105`
- 输出迁移摘要并提示用户后续可继续调参：`.../SKILL.md:16-17`

3. 问题驱动的二次修复
- 默认仅改模型串；只有用户明确请求或报告问题时才引入 prompt snippets：`.../SKILL.md:50-53`、`.../references/prompt-snippets.md:3`
- 注入时需与原 prompt 结构融合，并建议使用 XML 标签组织新增规则：`.../SKILL.md:54-59`、`.../references/prompt-snippets.md:99-106`

### 数据结构

1. 技能元数据（YAML frontmatter）
- 字段：`name`、`description`，用于触发判定：`.../SKILL.md:1-4`

2. 迁移映射表（Markdown 表驱动）
- 目标模型表：`Platform -> Opus 4.5 Model String`：`.../SKILL.md:33-38`
- 来源模型表：`Source Model -> 各平台旧模型串`：`.../SKILL.md:42-47`
- 排除规则：Haiku 不迁移：`.../SKILL.md:48`

3. API 参数片段
- `effort` 通过 `output_config.effort` 提供，同时要求 beta `effort-2025-11-24`：`.../references/effort.md:17,24-27,37-40,50-53`
- thinking 敏感性判断依赖请求里是否存在 `thinking` 字段：`.../references/prompt-snippets.md:79-85`

### 协议与命令

- 插件协议：目录 + `SKILL.md` 的声明式协议，无目录内可执行代码。
- API 协议关键字段：`model`、`betas`/`anthropic-beta`、`output_config`、`thinking`：`.../references/effort.md:21-55`、`.../references/prompt-snippets.md:79-85`
- 可操作命令层面：该目录无 `sh/py/ts/js` 脚本；测试建议来自通用规范（`cc --plugin-dir /path/to/plugin` 人工触发验证）：`plugins/plugin-dev/skills/skill-development/SKILL.md:284-292`

## 关键代码路径与文件引用

### 目录内核心路径
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:1-105`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/effort.md:1-70`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/prompt-snippets.md:1-106`

### 上游调用方/触发入口
- 根仓库插件入口说明：`README.md:48-50`
- 插件目录总览中的技能声明：`plugins/README.md:16`
- marketplace 插件注册与 source 路径：`.claude-plugin/marketplace.json:18-27`
- 技能自动发现机制：`plugins/plugin-dev/skills/skill-development/SKILL.md:271-276`

### 下游被调用方/影响面
- 用户代码库中的模型配置与 API 调用参数（被迁移动作直接改写）：`.../SKILL.md:12-17,31-48`
- 用户系统提示词内容（仅在问题驱动场景下按片段修复）：`.../SKILL.md:50-99`、`.../references/prompt-snippets.md:5-106`
- 外部模型服务协议（Anthropic/Bedrock/Vertex/Azure 的模型 ID 与参数格式）：`.../SKILL.md:33-47`、`.../references/effort.md:21-55`

### 配置/测试/脚本/文档上下文
- 配置：
  - 插件元数据：`plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:1-9`
  - 插件说明文档：`plugins/claude-opus-4-5-migration/README.md:1-21`
- 测试：目录内无自动化测试；参考做法是以 `--plugin-dir` 进行安装后人工触发验证：`plugins/plugin-dev/skills/skill-development/SKILL.md:284-292`
- 脚本：目录内无执行脚本，迁移由 Claude 按文字流程直接执行。
- 文档：`SKILL.md` 负责流程主规约，`references/*.md` 承载细分知识。

## 依赖与外部交互

### 1. 平台与版本依赖
- 模型 ID 强耦合到日期版号（如 `claude-opus-4-5-20251101`）：`.../SKILL.md:35-38`
- 来源模型 ID 也为固定版号（`20250514/20250929/20250422`）：`.../SKILL.md:44-46`
- effort 依赖 beta 标识 `effort-2025-11-24`：`.../references/effort.md:17`

### 2. 运行时依赖
- 依赖 Claude Code 技能调度机制按描述触发技能：`plugins/plugin-dev/skills/skill-development/SKILL.md:271-276`
- 依赖用户仓库可读写，以执行搜索、替换、提示词编辑。

### 3. 外部交互
- 目录本身不直接发网络请求，但其迁移规则直接作用于外部模型调用配置。
- 插件 README 外链指向 Anthropic 提示词最佳实践：`plugins/claude-opus-4-5-migration/README.md:17`

### 4. 与其他能力的关系
- `prompt-snippets.md` 内“前端设计质量”片段与独立 `frontend-design` 插件目标存在语义重叠（均用于提升前端输出质量），未来可能出现规则漂移。

## 风险、边界与改进建议

### 风险与边界

1. `effort` 策略存在内部冲突
- 工作流第 4 步写“添加 `effort=high`”：`.../SKILL.md:15`
- 参考章节写“仅在用户请求时配置 effort”：`.../SKILL.md:105`
- 会导致不同执行者迁移行为不一致。

2. 硬编码版本漂移风险
- 模型串与 beta 头采用固定日期 ID：`.../SKILL.md:35-47`、`.../references/effort.md:17`
- 当平台发布新版本后，技能可能“可执行但非最新”。

3. 字符串替换误伤风险
- 当前规约以字符串命中为中心，未要求“先列命中清单再确认”的防呆流程。
- 在注释/文档/废弃配置并存仓库中，存在误改非运行路径风险。

4. 无自动化回归保障
- 目录无测试样例与脚本，难以自动验证迁移完整性与正确性。

5. 覆盖边界有限
- 映射表覆盖主流平台格式，但不覆盖企业自定义别名、网关路由字段或二次封装 SDK。

### 改进建议

1. 明确统一 effort 默认策略
- 在 `SKILL.md` 中统一为“默认添加”或“仅按需添加”其一，并删除冲突描述。

2. 增加“审计-确认-替换”三段式流程
- 先输出命中结果与替换计划，再执行变更并输出 diff 摘要，降低误替换概率。

3. 增加最小验证资产
- 在技能目录补充 `examples/`（迁移前后样例）或 `scripts/`（校验命中与替换完整性），用于快速回归。

4. 建立版本更新提示机制
- 在 README 或 SKILL 中增加“目标模型串最后验证日期”，并在发布流程中校验模型 ID 时效性。

5. 抽离可复用提示词片段
- 将与其他插件重叠的片段抽成共享 reference，减少多点维护导致的不一致。
