# FILE `plugins/hookify/commands/hookify.md` 研究文档

## 场景与职责

`/hookify` 是 Hookify 插件的主入口，负责把“自然语言约束”转成可执行规则文件 `.claude/hookify.{name}.local.md`。它承担的是规则生产，不直接做 hook 执行。

职责边界：
- 上游：用户显式给出约束（有参）或要求自动分析会话（无参）。
- 中游：命令通过 `Skill + Task + AskUserQuestion + Write` 完成“分析 -> 选择 -> 生成”。
- 下游：生成规则由 `hooks/*.py` 在各事件触发时装载，`RuleEngine` 判定 warn/block。

关键定位：
- 命令定义：`plugins/hookify/commands/hookify.md:1`
- 会话分析 agent：`plugins/hookify/agents/conversation-analyzer.md:1`
- 运行时加载与执行：`plugins/hookify/core/config_loader.py:198`、`plugins/hookify/core/rule_engine.py:35`

## 功能点目的

1. 降低规则创建门槛
- 用户无需手写 frontmatter，通过对话就能得到规则文件（`plugins/hookify/commands/hookify.md:82-102`）。

2. 支持双入口创建
- 有参数：围绕 `$ARGUMENTS` 精准建规则。
- 无参数：通过会话分析发现可治理问题（`plugins/hookify/commands/hookify.md:19-27`）。

3. 在落盘前完成用户确认
- 通过 AskUserQuestion 让用户选择“建哪些规则、warn 还是 block、具体 pattern”（`plugins/hookify/commands/hookify.md:62-80`）。

4. 保证规则写入正确位置
- 强调写入“当前工作目录 `.claude/`”，避免误写到插件目录导致规则不生效（`plugins/hookify/commands/hookify.md:128-137`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

1. 命令协议与权限面
- frontmatter 声明 `argument-hint` 与工具集合：`Read/Write/AskUserQuestion/Task/Grep/TodoWrite/Skill`（`plugins/hookify/commands/hookify.md:3-4`）。
- 首步要求加载 `hookify:writing-rules`，确保输出 DSL 与运行时契约一致（`plugins/hookify/commands/hookify.md:9`，`plugins/hookify/skills/writing-rules/SKILL.md:13-27`）。

2. 行为信息收集流程
- 有参路径：使用 `$ARGUMENTS`，并补充最近对话上下文（`plugins/hookify/commands/hookify.md:19-23`）。
- 无参路径：触发 conversation analysis（`plugins/hookify/commands/hookify.md:24-27`），示例里通过 Task 工具传结构化 prompt（`plugins/hookify/commands/hookify.md:30-58`）。

3. 交互式确认流程
- Question 1：选择要 hookify 的行为（多选，最多 4 项）。
- Question 2：每项选择 `warn` 或 `block`。
- Question 3：确认/修订触发 pattern。
- 该设计把“检测建议”与“治理策略”分离，避免直接自动落盘（`plugins/hookify/commands/hookify.md:62-80`）。

4. 规则文件生成协议
- 简单规则使用 `pattern`（`plugins/hookify/commands/hookify.md:91-103`）。
- 复杂规则使用 `conditions[]`（`plugins/hookify/commands/hookify.md:108-124`）。
- 核心字段与 runtime 映射：
  - `name/enabled/event/pattern/conditions/action/message`
  - 由 `Rule.from_dict` 转为 `Rule/Condition`（`plugins/hookify/core/config_loader.py:45-84`）。

5. 落盘与生效链路
- 要求先确保 `.claude/` 存在，再写 `.claude/hookify.{name}.local.md`（`plugins/hookify/commands/hookify.md:132-137`）。
- 规则生效机制：每次 hook 触发时调用 `load_rules(event=...)` 动态扫描 `.claude/hookify.*.local.md`（`plugins/hookify/core/config_loader.py:209-227`），无需重启。

6. warn/block 执行协议
- `RuleEngine` 对命中规则按 `action` 分类：
  - `block` + `PreToolUse/PostToolUse` -> `permissionDecision: deny`
  - `block` + `Stop` -> `decision: block`
  - `warn` -> `systemMessage`
- 语义来源：`plugins/hookify/core/rule_engine.py:60-91`

## 关键代码路径与文件引用

- 主命令：`plugins/hookify/commands/hookify.md`
- 会话分析 agent：`plugins/hookify/agents/conversation-analyzer.md`
- 规则语法 skill：`plugins/hookify/skills/writing-rules/SKILL.md`
- 规则加载：`plugins/hookify/core/config_loader.py`
- 规则求值：`plugins/hookify/core/rule_engine.py`
- Hook 注册：`plugins/hookify/hooks/hooks.json`
- Hook 执行器：
  - `plugins/hookify/hooks/pretooluse.py`
  - `plugins/hookify/hooks/posttooluse.py`
  - `plugins/hookify/hooks/stop.py`
  - `plugins/hookify/hooks/userpromptsubmit.py`
- 用户文档：`plugins/hookify/README.md:39-69`

测试/脚本上下文：
- 命令层无自动测试。
- 运行层只有 `config_loader.py` 与 `rule_engine.py` 中 `__main__` 手工验证段，不覆盖 `/hookify` 交互流程。

## 依赖与外部交互

1. 工具依赖
- `Skill`：加载规则 DSL 指南。
- `Task`：会话分析子任务。
- `AskUserQuestion`：规则确认与参数化。
- `Write`：写规则文件。
- `TodoWrite`：流程追踪（`plugins/hookify/commands/hookify.md:231`）。

2. 文件系统依赖
- 强依赖当前工作目录 `.claude/` 路径；与插件目录隔离（`plugins/hookify/commands/hookify.md:128-137`）。

3. 运行时耦合
- 与 `hooks.json` 的 4 类事件强耦合；`event` 字段需对应运行时过滤逻辑（`plugins/hookify/hooks/hooks.json:4-47`，`plugins/hookify/core/config_loader.py:219-223`）。

4. 外部交互范围
- 不依赖网络 API；外部交互主要是 Claude Code 工具协议和本地文件读写。

## 风险、边界与改进建议

1. agent 标识语义不一致
- 现状：文本写“Launch the conversation-analyzer agent”，但 Task 示例为 `subagent_type: general-purpose`（`plugins/hookify/commands/hookify.md:25` 与 `plugins/hookify/commands/hookify.md:33`）。
- 风险：分析质量和输出结构不稳定。
- 建议：统一为专用 agent 标识，或在 prompt 中强制 JSON schema 输出。

2. `event=stop/prompt` 的 simple pattern 易失效
- 现状：`pattern` 在 `Rule.from_dict` 中对非 bash/file 映射到 `field=content`（`plugins/hookify/core/config_loader.py:61-68`）。
- 风险：Stop/Prompt 事件常用字段是 `transcript/user_prompt`，导致规则可能不触发。
- 建议：命令对 stop/prompt 默认输出 `conditions`，字段改为 `transcript` 或 `user_prompt`。

3. `event=file` + simple pattern 对 Write 覆盖不足
- 现状：simple file pattern 映射到 `new_text`；Write 的关键内容通常在 `content`（`plugins/hookify/core/rule_engine.py:235-240`）。
- 风险：规则在 Write 场景漏报。
- 建议：生成 file 规则时优先 `conditions`，同时匹配 `new_text` 与 `content`。

4. 规则命名与冲突控制不足
- 风险：同名规则文件或语义重复规则会造成提示噪音/冲突。
- 建议：落盘前做名称去重检查，并提示“合并到现有规则/创建新规则”二选一。

5. 缺少闭环测试
- 风险：命令文案变动后可能破坏 “分析 -> 生成 -> hook 生效” 链路。
- 建议：增加最小 E2E smoke（mock AskUserQuestion + 生成规则 + 喂入 hook JSON 验证 warn/block 输出）。
