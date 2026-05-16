# FILE `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/prompt-snippets.md` 研究文档

## 场景与职责

`prompt-snippets.md` 是 Opus 4.5 迁移 skill 的“问题驱动补丁库”，用于在用户报告具体行为问题时，向现有 system prompt 定向注入规则片段，而非默认迁移动作。

在调用链中：
- 直接调用方是 `SKILL.md` 的 Prompt Adjustments 部分与 Reference 跳转（`plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:50-59,79,85,91,103`）。
- 入口仍由插件 README 的迁移意图触发（`plugins/claude-opus-4-5-migration/README.md:11-13`），插件通过 marketplace 条目被发现（`.claude-plugin/marketplace.json:18-26`）。

该文件的职责是把“症状 -> 适用条件 -> 插入片段”结构化，降低迁移后行为偏差修复的随机性。

## 功能点目的

1. 建立默认边界：默认只迁移模型串  
- 文件开头明确“仅在用户显式请求或报告问题时应用片段”（`prompt-snippets.md:3`）。

2. 提供 5 类已知问题修复材料  
- 工具过触发（`prompt-snippets.md:5-19`）  
- 过度工程（`prompt-snippets.md:20-33`）  
- 代码探索不足（`prompt-snippets.md:35-45`）  
- 前端设计质量（`prompt-snippets.md:47-73`）  
- thinking 词敏感（`prompt-snippets.md:75-98`）

3. 约束片段集成方法  
- 要求不要机械追加，应与现有 prompt 结构融合，并使用语义化 XML 标签（`prompt-snippets.md:99-106`）。

4. 保证迁移可追踪性  
- 要求迁移后总结模型串更新与 prompt 修改点（`prompt-snippets.md:106`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程

1. 执行迁移时先按 `SKILL.md` 处理模型串、beta 头等默认步骤（`SKILL.md:12-48`）。  
2. 仅当用户反馈具体问题，进入 `Prompt Adjustments` 分支（`SKILL.md:52-53`）。  
3. 在 `prompt-snippets.md` 中匹配对应症状段落，选取片段并按原 prompt 结构植入（`prompt-snippets.md:99-105` + `SKILL.md:54-59`）。  
4. 输出总结，记录哪些片段被引入（`prompt-snippets.md:106` + `SKILL.md:16`）。

### 数据结构

1. “问题三元组”结构  
- 每一类问题都由 `Problem`、`When to add/apply`、`Solution/Snippet` 组成（`prompt-snippets.md:7-12,22-27,37-42,49-55,77-90`）。

2. 映射替换表  
- 工具过触发与 thinking 敏感使用 `Before -> After` 表表达替换词典（`prompt-snippets.md:13-19,91-98`）。

3. 可粘贴长文本片段  
- 过度工程、代码探索、前端质量通过 fenced code block 表达可注入规则（`prompt-snippets.md:28-33,43-45,55-73`）。

4. thinking 启用判定片段  
- JSON 示例定义“扩展 thinking 已开启”的判定前提（`prompt-snippets.md:79-85`）。

### 协议与命令

- 该文件本身不含可执行命令；“协议”体现在对上游 API 请求语义的判断要求：
  - 仅当请求缺少 `thinking` 参数时，才应用 thinking 词替换策略（`prompt-snippets.md:79-87`）。
- 与主 skill 的协议联动：
  - 仅在用户反馈问题时修改 prompt（`SKILL.md:52-53`）。
  - 插入时匹配现有 XML/结构风格（`SKILL.md:54-59`）。

## 关键代码路径与文件引用

核心文件：
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/prompt-snippets.md:1-106`

直接调用方（上游）：
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:50-59`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:79`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:85`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:91`
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/SKILL.md:103`

同级协作文件：
- `plugins/claude-opus-4-5-migration/skills/claude-opus-4-5-migration/references/effort.md:1-70`（参数配置库，解决的是 cost/perf 参数，不是 prompt 行为）

插件与平台入口：
- `plugins/claude-opus-4-5-migration/README.md:1-21`
- `plugins/claude-opus-4-5-migration/.claude-plugin/plugin.json:1-9`
- `.claude-plugin/marketplace.json:18-26`

潜在规则重叠路径：
- `plugins/README.md:21`（`frontend-design` 插件描述，和本文件前端片段方向重合）

## 依赖与外部交互

### 内部依赖

1. 强依赖 `SKILL.md` 的分支条件  
- 没有 `SKILL.md` 的“问题触发”判断，该文件不应被默认执行。

2. 依赖目标仓库的 prompt 组织方式  
- 是否已有 XML 标签、是否简洁风格，会影响片段裁剪策略（`prompt-snippets.md:101-105`）。

3. 依赖插件技能自动发现机制  
- skills/references 由 Claude Code 按需加载（`plugins/plugin-dev/skills/skill-development/SKILL.md:271-277`）。

### 外部交互

1. 与用户真实问题反馈交互  
- 触发依据是“用户报告了哪类问题”，属于运行时对话信号，不是静态代码信号。

2. 与 API 请求语义交互  
- thinking 敏感修复需要识别是否存在 `thinking` 参数（`prompt-snippets.md:79-87`）。

3. 与前端设计实践外部规范交互  
- 前端片段会与项目现有设计系统、可用动画库、字体策略发生耦合（`prompt-snippets.md:55-73`）。

### 配置/测试/脚本现状

- 配置：无独立配置文件，片段全部内联在 Markdown。
- 测试：无自动测试用例验证“片段注入后行为变化”。
- 脚本：无自动插入脚本，完全依赖迁移执行者人工整合。

## 风险、边界与改进建议

### 风险

1. 过度注入风险  
- 若忽视“默认仅改模型串”的约束（`prompt-snippets.md:3`），可能引入与用户原 prompt 不一致的大量新规则。

2. 语义漂移风险  
- 机械执行关键词替换（如 `think -> consider`）可能改坏原语气或技术术语（`prompt-snippets.md:91-98`）。

3. 上下文膨胀风险  
- 前端片段与过度工程片段篇幅较长（`prompt-snippets.md:28-33,55-73`），可能增加 system prompt token 负担并影响指令优先级。

4. 规范冲突风险  
- 前端片段与独立 `frontend-design` 技能可能长期漂移，导致同类任务得到不同风格指令（`plugins/README.md:21` + `prompt-snippets.md:47-73`）。

5. 触发条件主观风险  
- “用户报告问题”通常是自然语言主观描述，缺乏统一判定标准。

### 边界

1. 该文件不负责模型字符串迁移。  
2. 不定义 API 参数修改（如 `output_config.effort`），那部分由 `effort.md` 负责。  
3. 仅提供建议片段，不保证自动注入或自动验证效果。

### 改进建议

1. 增加“最小注入版”片段  
- 为每类问题提供 short/standard 两版，先用短版降低行为扰动。

2. 增加插入位置模板  
- 例如明确“工具片段放 `<tool_behavior>`，代码探索放 `<coding_guidelines>`”，减少人工拼接差异。

3. 增加注入后检查清单  
- 包含“是否保留原有约束”“是否仅修改相关段落”“是否记录变更摘要”等固定校验项。

4. 建立跨技能单一前端规范源  
- 前端美学段落可引用 `frontend-design` 的统一片段源，避免双份文案长期偏离。

5. 增加示例对照资产  
- 补充 2-3 组真实 `before/after` prompt 片段，示范如何“融合而非追加”。
