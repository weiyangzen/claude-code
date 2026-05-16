# DIR `plugins/hookify/skills` 研究文档

## 场景与职责

`plugins/hookify/skills` 是 Hookify 插件的“规则写作规范层”，当前目录只包含一个技能：
- `plugins/hookify/skills/writing-rules/SKILL.md`

它不直接执行 Hook，也不直接读写运行时输入；核心职责是给命令型 agent 提供统一规则 DSL（frontmatter + message body）与写作约束，减少 `/hookify`、`/hookify:list`、`/hookify:configure` 在规则语法上的漂移。

在插件链路中的位置：
- 上游调用方：命令文档显式要求先加载 `hookify:writing-rules`（`plugins/hookify/commands/hookify.md:9`、`plugins/hookify/commands/list.md:8`、`plugins/hookify/commands/configure.md:8`）。
- 下游消费者：技能指导生成的 `.claude/hookify.*.local.md` 最终由 Hook 运行时加载和评估（`plugins/hookify/core/config_loader.py:198`、`plugins/hookify/core/rule_engine.py:35`）。
- 插件元信息上下文：Hookify 在插件总览中声明该 skill（`plugins/README.md:22`）。

## 功能点目的

### 1) 统一规则文件契约
技能定义了规则文件必须采用 markdown + YAML frontmatter，并固定命名为 `.claude/hookify.{rule-name}.local.md`（`plugins/hookify/skills/writing-rules/SKILL.md:11`）。

### 2) 统一字段语义与可选能力
技能明确 `name/enabled/event/action/pattern/conditions` 的语义和可选性，避免命令生成出引擎不可消费的字段（`plugins/hookify/skills/writing-rules/SKILL.md:29`、`plugins/hookify/skills/writing-rules/SKILL.md:48`、`plugins/hookify/skills/writing-rules/SKILL.md:64`）。

### 3) 把用户自然语言约束映射为可匹配条件
技能为 `bash/file/stop/prompt/all` 事件提供示例与模式建议，帮助把“不要做 X”转为可执行规则（`plugins/hookify/skills/writing-rules/SKILL.md:127`）。

### 4) 规范告警文案质量
技能定义 message body 的写作建议（检测内容、风险解释、替代方案），提高触发后提示可执行性（`plugins/hookify/skills/writing-rules/SKILL.md:100`）。

### 5) 降低规则调试门槛
技能内置 regex 编写与本地测试建议（`python3 -c ...`），让用户和命令在生成规则后可快速自检（`plugins/hookify/skills/writing-rules/SKILL.md:254`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 技能调用协议

技能本身是文档驱动协议，不是 Python 模块。命令通过 `allowed-tools` 里的 `Skill` 工具声明可加载技能，再在正文第一步要求“先加载 hookify:writing-rules”：
- `/hookify`：`plugins/hookify/commands/hookify.md:4`, `plugins/hookify/commands/hookify.md:9`
- `/hookify:list`：`plugins/hookify/commands/list.md:3`, `plugins/hookify/commands/list.md:8`
- `/hookify:configure`：`plugins/hookify/commands/configure.md:3`, `plugins/hookify/commands/configure.md:8`

这使技能成为命令执行前的“软约束 schema”。

### B. 规则 DSL 到运行时数据结构的映射

技能定义的 frontmatter 字段，与运行时 `Rule/Condition` 一一对应：
- `Rule`：`name/enabled/event/pattern/conditions/action/message`（`plugins/hookify/core/config_loader.py:33`）
- `Condition`：`field/operator/pattern`（`plugins/hookify/core/config_loader.py:15`）

映射细节：
1. 简单写法（`pattern`）：
   - 由 `Rule.from_dict` 自动转为单条 `Condition`（`plugins/hookify/core/config_loader.py:56`）。
   - `event=bash` 映射 `field=command`；`event=file` 映射 `field=new_text`（`plugins/hookify/core/config_loader.py:61`）。
2. 高级写法（`conditions`）：
   - 直接解析为 `Condition` 列表（`plugins/hookify/core/config_loader.py:50`）。
3. 文本 body：
   - 由 `extract_frontmatter` 拆分并写入 `Rule.message`（`plugins/hookify/core/config_loader.py:87`, `plugins/hookify/core/config_loader.py:103`, `plugins/hookify/core/config_loader.py:83`）。

### C. 关键执行流程（技能 -> 命令 -> 运行时）

1. 用户触发 `/hookify` 系列命令。  
2. 命令先读取技能规范（命令文档要求）。  
3. 命令收集行为约束（参数或会话分析 agent）并生成 `.claude/hookify.*.local.md`（`plugins/hookify/commands/hookify.md:84`, `plugins/hookify/commands/hookify.md:128`；agent 参考 `plugins/hookify/agents/conversation-analyzer.md:11`）。  
4. Hook 事件触发后，`hooks.json` 调用 Python 执行器（`plugins/hookify/hooks/hooks.json:4`）。  
5. 执行器按事件加载规则并交给引擎求值（`plugins/hookify/hooks/pretooluse.py:52`, `plugins/hookify/core/rule_engine.py:35`）。  
6. 命中后输出协议化 JSON：`warn` 仅 `systemMessage`，`block` 在 Tool 事件返回 `permissionDecision=deny`，Stop 事件返回 `decision=block`（`plugins/hookify/core/rule_engine.py:60`）。

### D. 关键协议与命令/字段语义

1. 事件类型：
   - 技能声明 `bash/file/stop/prompt/all`（`plugins/hookify/skills/writing-rules/SKILL.md:41`）。
   - hook 执行器将 `tool_name` 归一化为 `bash/file` 事件过滤（`plugins/hookify/hooks/pretooluse.py:41`, `plugins/hookify/hooks/posttooluse.py:36`）。
2. 条件运算符：
   - 技能列出 `regex_match/contains/equals/not_contains/starts_with/ends_with`（`plugins/hookify/skills/writing-rules/SKILL.md:89`）。
   - 引擎按字符串语义执行上述操作符（`plugins/hookify/core/rule_engine.py:166`）。
3. 字段提取：
   - `command/file_path/new_text/old_text/content/transcript/user_prompt` 等由 `_extract_field` 决定来源（`plugins/hookify/core/rule_engine.py:182`）。

### E. 测试、脚本、文档现状

1. `skills` 目录内无脚本、无测试，仅 `SKILL.md` 文档资产。  
2. Hookify 插件整体无 `tests/` 目录；当前仅 `core/*.py` 保留 `__main__` 手工测试片段（`plugins/hookify/core/config_loader.py:277`, `plugins/hookify/core/rule_engine.py:276`）。  
3. 用户文档侧，README 与 help 命令共同描述同一规则契约（`plugins/hookify/README.md:71`, `plugins/hookify/commands/help.md:26`）。

## 关键代码路径与文件引用

目标目录：
- `plugins/hookify/skills/writing-rules/SKILL.md`

直接调用方（命令层）：
- `plugins/hookify/commands/hookify.md`
- `plugins/hookify/commands/list.md`
- `plugins/hookify/commands/configure.md`
- `plugins/hookify/commands/help.md`

下游运行时（消费规则）：
- `plugins/hookify/hooks/hooks.json`
- `plugins/hookify/hooks/pretooluse.py`
- `plugins/hookify/hooks/posttooluse.py`
- `plugins/hookify/hooks/stop.py`
- `plugins/hookify/hooks/userpromptsubmit.py`
- `plugins/hookify/core/config_loader.py`
- `plugins/hookify/core/rule_engine.py`

协同上下文：
- `plugins/hookify/agents/conversation-analyzer.md`
- `plugins/hookify/examples/dangerous-rm.local.md`
- `plugins/hookify/examples/console-log-warning.local.md`
- `plugins/hookify/examples/sensitive-files-warning.local.md`
- `plugins/hookify/examples/require-tests-stop.local.md`
- `plugins/hookify/README.md`
- `plugins/README.md`
- `plugins/hookify/.claude-plugin/plugin.json`

## 依赖与外部交互

### 1) 文件系统
- 规则产物落在“当前项目目录” `.claude/` 下（`plugins/hookify/commands/hookify.md:128`）。
- 运行时扫描模式固定为 `.claude/hookify.*.local.md`（`plugins/hookify/core/config_loader.py:210`）。

### 2) 运行环境
- Hook 执行依赖 `python3` 命令（`plugins/hookify/hooks/hooks.json:9`）。
- 依赖 `CLAUDE_PLUGIN_ROOT` 注入 `sys.path`，确保 `hookify.core` 可导入（`plugins/hookify/hooks/pretooluse.py:14`）。

### 3) Claude Code 工具协议
- 命令层依赖 `Skill/Task/AskUserQuestion/Write/Edit/Glob/Read` 等工具能力（例如 `plugins/hookify/commands/hookify.md:4`）。
- 规则结果需符合 Hook JSON 协议（deny/block/systemMessage）才能影响执行控制（`plugins/hookify/core/rule_engine.py:66`）。

### 4) 外部交互边界
- 无网络调用、无第三方 Python 依赖（README 声明 stdlib-only，`plugins/hookify/README.md:298`）。
- 会读取 `transcript_path` 指向的本地会话文件（Stop 规则，`plugins/hookify/core/rule_engine.py:207`）。

## 风险、边界与改进建议

### 1) `file + pattern` 简写在 Write 场景可能漏报
- 现状：技能示例大量使用 `event: file + pattern`（`plugins/hookify/skills/writing-rules/SKILL.md:151`）；运行时会把简写映射到 `field=new_text`（`plugins/hookify/core/config_loader.py:64`）。
- 风险：`Write` 工具主要字段是 `content`，`new_text` 分支读取 `new_string`，可能导致匹配为空（`plugins/hookify/core/rule_engine.py:235`, `plugins/hookify/core/rule_engine.py:239`）。
- 建议：  
  1. 在技能文档中补充“Write 场景优先使用 `conditions` + `field: content`”。  
  2. 或在 `Rule.from_dict` 对 `event=file` 默认改为 `content|new_text` 兼容策略。

### 2) `not_contains` 示例把 regex OR 当作普通子串
- 现状：技能与示例常写 `pattern: npm test|pytest|cargo test`（`plugins/hookify/examples/require-tests-stop.local.md:9`）。
- 风险：`not_contains` 在引擎中是普通字符串包含判断，不是正则 OR（`plugins/hookify/core/rule_engine.py:172`），会造成 Stop 规则误判。
- 建议：  
  1. 技能里明确 `not_contains` 不支持 regex。  
  2. 推荐改为 `operator: regex_match` + 反向逻辑（或新增 `not_regex_match`）。

### 3) 命令文本与 agent 调用示例存在轻微不一致
- 现状：`/hookify` 文本写“launch conversation-analyzer agent”，但示例 Task payload 用 `subagent_type: general-purpose`（`plugins/hookify/commands/hookify.md:25`, `plugins/hookify/commands/hookify.md:33`）。
- 风险：实现者可能无法稳定调用到目标 agent 配置（`plugins/hookify/agents/conversation-analyzer.md:2`）。
- 建议：统一为显式 `conversation-analyzer` 调用约定并给出标准 payload。

### 4) frontmatter 解析器是手写 YAML 子集
- 现状：`extract_frontmatter` 为自实现 parser（`plugins/hookify/core/config_loader.py:105`）。
- 风险：对复杂 YAML（嵌套结构、特殊字符、注释位置）容错有限，技能文档又鼓励用户手工写规则，容易触发解析偏差。
- 建议：  
  1. 在技能里明确“仅支持的 YAML 子集”。  
  2. 新增轻量校验命令或 lint 脚本，先校验再加载。

### 5) 缺少自动化回归测试
- 现状：`skills` 目录无测试，Hookify 插件仅有 `__main__` 手工示例（`plugins/hookify/core/config_loader.py:277`, `plugins/hookify/core/rule_engine.py:276`）。
- 风险：技能文档更新与引擎语义变更容易失配，问题通常在运行时才暴露。
- 建议：新增最小自动化用例矩阵：
  1. `event=file` 下 Write/Edit/MultiEdit 字段映射。  
  2. `not_contains` 与 regex 行为。  
  3. `block/warn` 输出协议在 PreToolUse/Stop 两种事件的差异。

