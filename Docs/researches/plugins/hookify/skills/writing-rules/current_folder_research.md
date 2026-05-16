# DIR `plugins/hookify/skills/writing-rules` 研究文档

## 场景与职责

`plugins/hookify/skills/writing-rules` 目录当前只有一个资产：`SKILL.md`。它不是运行时代码，而是 Hookify 规则 DSL 的“作者指南/执行约束”，用于在 Claude Code 的命令执行前统一规则写法。

它在整体链路中的职责是：
- 给命令层提供规则语法基线：`/hookify`、`/hookify:list`、`/hookify:configure` 都显式要求先加载该 skill，再读写 `.claude/hookify.*.local.md` 规则文件。
- 把用户自然语言偏好转成可执行前置约束：定义 `event/action/pattern/conditions` 等字段语义、命名规范、regex 写法和消息体写法。
- 作为运行时引擎的上游契约：skill 约定的 frontmatter 最终由 `core/config_loader.py` 解析为 `Rule/Condition`，再由 `core/rule_engine.py` 执行匹配与阻断/告警决策。

## 功能点目的

`SKILL.md` 的核心功能点与目的如下：

1. 规则文件契约定义  
- 规定规则文件使用 markdown + YAML frontmatter，路径为 `.claude/hookify.{rule-name}.local.md`，实现“用户项目本地规则、即时生效”的约束。

2. 字段与语义标准化  
- 定义 `name/enabled/event/action/pattern/conditions` 的用途，降低不同命令/代理生成规则时的歧义。

3. 事件与字段映射指导  
- 给出 `bash/file/stop/prompt/all` 事件与可用字段（如 `command`、`file_path`、`new_text`、`user_prompt`）以及常用模式，帮助从意图转为可匹配条件。

4. 文案质量规范  
- 明确 message body 应包含“检测内容 + 风险原因 + 替代建议”，避免只给机械警告。

5. 调试与维护流程  
- 给出 regex 本地验证命令、命名规范、启停规则流程（`enabled: false` / 删除文件），降低维护成本。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程（从 skill 到运行时）

1. 用户执行 `/hookify` 系列命令。  
2. 命令文档要求先加载 `hookify:writing-rules`，据此组织提问、生成规则文件。  
3. 规则文件落地到项目根 `.claude/hookify.*.local.md`。  
4. Hook 事件触发时，`hooks/hooks.json` 调用 Python 执行器（PreToolUse/PostToolUse/Stop/UserPromptSubmit）。  
5. 执行器调用 `load_rules(event=...)` 读取规则文件，`extract_frontmatter` 手工解析 frontmatter，`Rule.from_dict` 转成 `Rule/Condition`。  
6. `RuleEngine.evaluate_rules` 执行条件匹配并返回协议 JSON：warning 返回 `systemMessage`；blocking 在工具事件返回 `permissionDecision: deny`，在 Stop 事件返回 `decision: block`。

### 2) 关键数据结构与 DSL 映射

- 规则 DSL（skill 定义）：
  - `name`: 规则唯一标识
  - `enabled`: 启停开关
  - `event`: `bash|file|stop|prompt|all`
  - `action`: `warn|block`
  - `pattern` 或 `conditions[]`
  - markdown body（触发提示内容）
- 运行时数据结构（core）：
  - `Rule(name, enabled, event, pattern, conditions, action, tool_matcher, message)`
  - `Condition(field, operator, pattern)`
- 简写与高级写法：
  - 仅 `pattern` 时，`Rule.from_dict` 会自动转为单条 `Condition`；`event=bash` 映射 `field=command`，`event=file` 映射 `field=new_text`，其他事件映射 `field=content`。
  - `conditions` 时，按显式 `field/operator/pattern` 列表执行，规则命中语义是“所有条件 AND”。

### 3) 协议与命令

- Hook 输入通道：stdin JSON（包含 `hook_event_name/tool_name/tool_input/transcript_path/user_prompt` 等）。
- Hook 输出协议：
  - 命中 `warn`：`{"systemMessage": "..."}`
  - 命中 `block` 且事件为 Pre/PostToolUse：`hookSpecificOutput.permissionDecision = "deny"`
  - 命中 `block` 且事件为 Stop：`decision = "block"`
- 技能中给出的实用命令：
  - regex 自测：`python3 -c "import re; print(re.search(r'your_pattern', 'test text'))"`
- 目录内测试/脚本现状：
  - `plugins/hookify/skills/writing-rules` 本身无脚本与测试；验证主要依赖命令链路与运行时 hook 触发。

## 关键代码路径与文件引用

目标对象：
- `plugins/hookify/skills/writing-rules/SKILL.md`

直接调用方（命令层）：
- `plugins/hookify/commands/hookify.md`（要求优先加载 skill，并指导生成规则文件）
- `plugins/hookify/commands/list.md`（要求优先加载 skill，用于读取/展示规则）
- `plugins/hookify/commands/configure.md`（要求优先加载 skill，用于启停规则）

下游被调用方（运行时消费规则）：
- `plugins/hookify/core/config_loader.py`（规则文件扫描、frontmatter 解析、Rule/Condition 构建）
- `plugins/hookify/core/rule_engine.py`（条件匹配、warn/block 决策、输出协议拼装）
- `plugins/hookify/hooks/hooks.json`（Hook 事件与 Python 执行器注册）
- `plugins/hookify/hooks/pretooluse.py`
- `plugins/hookify/hooks/posttooluse.py`
- `plugins/hookify/hooks/stop.py`
- `plugins/hookify/hooks/userpromptsubmit.py`

配置与文档上下文：
- `plugins/hookify/README.md`（对用户公开的规则格式/示例/排障说明）
- `plugins/hookify/agents/conversation-analyzer.md`（无参数 `/hookify` 时的行为提取上游）
- `plugins/hookify/examples/*.local.md`（规则样例）
- `plugins/hookify/.claude-plugin/plugin.json`（插件元信息）
- `.claude-plugin/marketplace.json`（插件市场入口）
- `plugins/README.md`（插件目录总览中对 hookify skill 的声明）

研究流程相关脚本（本次任务依赖）：
- `.ops/research_guard.sh`（第 N 行 checklist 驱动研究任务模板）
- `.ops/generate_daily_research_todo.sh`（按 `blueprint_checklist.md` 生成当日待办）

## 依赖与外部交互

1. 运行时依赖  
- Python 3（hook 执行器由 `python3` 调起）。  
- 标准库（`json/os/sys/re/glob/dataclasses/functools` 等），无第三方依赖。  

2. 文件系统依赖  
- 规则文件固定扫描路径：项目当前工作目录下 `.claude/hookify.*.local.md`。  
- Stop 规则可读取 `transcript_path` 指向的会话文本。  

3. 环境变量/宿主协议依赖  
- `CLAUDE_PLUGIN_ROOT` 用于注入 `sys.path`，保证 `hookify.core` 可导入。  
- 依赖 Claude Code Hook 输入输出 JSON 协议字段的一致性。  

4. 与外部系统交互边界  
- 不发起网络请求。  
- 不依赖数据库或外部服务。  
- 主要与 Claude Code 宿主的 hook 事件生命周期交互。  

## 风险、边界与改进建议

1. `event: stop|prompt` + 简写 `pattern` 存在失效风险  
- `Rule.from_dict` 对非 bash/file 简写映射到 `field=content`，但 `_extract_field` 对 Stop/Prompt 并未稳定提供 `content` 字段，导致命中率不可预期。  
- 建议：为 `stop` 默认映射 `transcript` 或 `reason`，为 `prompt` 默认映射 `user_prompt`，并在 skill 文档明确说明。

2. `event: file` 简写默认 `new_text` 对 `Write` 场景不稳  
- `Write` 常用字段是 `content`，skill 示例大量用 `pattern` 简写；实际可能在不同工具下漏匹配。  
- 建议：skill 增补“Write 场景优先 conditions + field: content”，或在 core 中将 file 简写改为多字段兜底策略。

3. `not_contains` 与 regex 语义易混淆  
- 文档/示例常出现 `pattern: npm test|pytest|cargo test`；但 `not_contains` 是普通子串判断，不是 regex OR。  
- 建议：新增 `not_regex_match` 操作符，或在 skill 中强制区分“子串”和“正则”场景。

4. skill 文档与运行时能力存在漂移风险  
- skill 是文档契约，运行时是 Python 实现，当前缺少自动一致性检查。  
- 建议：补充“规则样例 -> 引擎结果”的自动化回归测试，覆盖 bash/file/stop/prompt 四类事件。

5. frontmatter 为手写 YAML 子集解析  
- 复杂 YAML（特殊字符、深层嵌套）可能被静默错误解析。  
- 建议：在 skill 中限定支持的 YAML 子集并提供 lint 校验命令；中期可替换为标准 YAML 解析器。

6. 调用约定文档存在轻微不一致  
- `/hookify` 文档提到 `conversation-analyzer`，示例 payload 却是 `subagent_type: general-purpose`。  
- 建议：统一命令文档中的 agent 调用模板，减少实现者歧义。

