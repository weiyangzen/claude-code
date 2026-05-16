# DIR `plugins/hookify/commands` 研究文档

## 场景与职责

`plugins/hookify/commands` 是 Hookify 插件的“规则编排入口层”，通过 4 个 slash command 文档把用户意图转成可执行规则文件，或对现有规则做可视化管理：
- `/hookify`：从显式指令或会话分析生成规则。
- `/hookify:list`：枚举现有规则。
- `/hookify:configure`：交互式启停规则。
- `/hookify:help`：说明使用方式与排障方法。

该目录本身不执行 Hook，不直接做规则匹配；它产出的规则文件由运行时 Hook 脚本和规则引擎消费：
- Hook 注册：`plugins/hookify/hooks/hooks.json:2`
- 规则加载：`plugins/hookify/core/config_loader.py:198`
- 规则求值：`plugins/hookify/core/rule_engine.py:35`

在插件总体分层中的位置：
- 上游：用户输入 slash 命令、插件系统发现 `commands/*.md`。
- 下游：写入/编辑 `.claude/hookify.*.local.md`，再由 `hooks/*.py` 在 PreToolUse/PostToolUse/Stop/UserPromptSubmit 事件中动态生效。

关键参考：
- `plugins/README.md:22`
- `plugins/hookify/README.md:39`
- `plugins/hookify/commands/hookify.md:1`
- `plugins/hookify/commands/list.md:1`
- `plugins/hookify/commands/configure.md:1`
- `plugins/hookify/commands/help.md:1`

## 功能点目的

### 1) `/hookify`：把“行为约束”落地为规则文件
目的：将用户偏好（例如“不要再用 rm -rf”）转成 `.claude/hookify.{name}.local.md`，降低手写 frontmatter 成本。

功能目标：
- 支持有参（直接用 `$ARGUMENTS`）与无参（会话分析）两种入口。
- 在创建前做用户确认（AskUserQuestion），让用户选择 warn/block 与触发模式。
- 强制写入项目工作目录 `.claude/`，避免误写入插件目录。

参考：
- `plugins/hookify/commands/hookify.md:19`
- `plugins/hookify/commands/hookify.md:24`
- `plugins/hookify/commands/hookify.md:62`
- `plugins/hookify/commands/hookify.md:128`

### 2) `/hookify:list`：提升规则可观测性
目的：快速查看当前规则总数、启用状态、触发事件与模式，支持用户调参与排障。

功能目标：
- 统一扫描 `.claude/hookify.*.local.md`。
- 读取 frontmatter 核心字段并生成表格摘要。
- 给出“无规则”时的下一步指引。

参考：
- `plugins/hookify/commands/list.md:14`
- `plugins/hookify/commands/list.md:24`
- `plugins/hookify/commands/list.md:62`

### 3) `/hookify:configure`：低风险启停切换
目的：不删除规则文件，通过 `enabled` 切换实现“临时关闭/恢复”，降低误删配置风险。

功能目标：
- 用多选问题收集要切换的规则。
- 批量编辑 `enabled: true/false`。
- 输出变更结果（Enabled/Disabled/Unchanged）。

参考：
- `plugins/hookify/commands/configure.md:35`
- `plugins/hookify/commands/configure.md:73`
- `plugins/hookify/commands/configure.md:92`

### 4) `/hookify:help`：统一知识入口
目的：把命令、事件类型、正则语法、排障建议集中到单一入口，降低学习成本。

参考：
- `plugins/hookify/commands/help.md:10`
- `plugins/hookify/commands/help.md:70`
- `plugins/hookify/commands/help.md:142`

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 命令元信息协议（frontmatter）
目录内 4 个命令均采用命令文档 frontmatter 协议：
- `description`：命令说明。
- `argument-hint`：仅 `/hookify` 定义可选参数提示。
- `allowed-tools`：限制命令执行阶段可调用工具集。

各命令的工具权限边界：
- `/hookify`：`Read/Write/AskUserQuestion/Task/Grep/TodoWrite/Skill`（可创建规则并调用分析 agent）。
- `/hookify:list`：`Glob/Read/Skill`（只读视图）。
- `/hookify:configure`：`Glob/Read/Edit/AskUserQuestion/Skill`（可编辑启用位）。
- `/hookify:help`：`Read`（文档说明型命令）。

参考：
- `plugins/hookify/commands/hookify.md:1`
- `plugins/hookify/commands/list.md:1`
- `plugins/hookify/commands/configure.md:1`
- `plugins/hookify/commands/help.md:1`

### B. `/hookify` 关键流程
1. 预置要求先加载 `hookify:writing-rules` skill，确保规则语法一致。
2. 输入分流：
   - 有参数：用 `$ARGUMENTS` + 最近会话补充上下文。
   - 无参数：通过 Task 启动会话分析流程。
3. 通过 AskUserQuestion 让用户选择行为、动作（warn/block）和匹配模式。
4. 生成规则文件模板（YAML frontmatter + Markdown message）。
5. 确保 `.claude/` 存在后，写入 `.claude/hookify.{name}.local.md`。
6. 明确提示“立即生效，无需重启”。

参考：
- `plugins/hookify/commands/hookify.md:9`
- `plugins/hookify/commands/hookify.md:17`
- `plugins/hookify/commands/hookify.md:30`
- `plugins/hookify/commands/hookify.md:84`
- `plugins/hookify/commands/hookify.md:132`
- `plugins/hookify/commands/hookify.md:154`

### C. `/hookify:list` 关键流程
1. Glob 扫描 `.claude/hookify.*.local.md`。
2. Read 每个文件，提取 `name/enabled/event/pattern` 与消息摘要。
3. 以表格 + 每条规则预览输出。
4. 输出管理提示（如何启停/删除/创建）。

参考：
- `plugins/hookify/commands/list.md:14`
- `plugins/hookify/commands/list.md:19`
- `plugins/hookify/commands/list.md:24`
- `plugins/hookify/commands/list.md:49`

### D. `/hookify:configure` 关键流程
1. Glob 规则文件并读取当前启用状态。
2. AskUserQuestion 多选要切换的规则。
3. Edit 将 `enabled: true` 与 `enabled: false` 互换。
4. 汇总并确认变更立即生效。

参考：
- `plugins/hookify/commands/configure.md:16`
- `plugins/hookify/commands/configure.md:35`
- `plugins/hookify/commands/configure.md:75`
- `plugins/hookify/commands/configure.md:108`

### E. 命令层与运行时数据结构映射
命令层生成的规则 frontmatter 最终映射到 runtime 数据结构：
- `Rule`：`name/enabled/event/pattern/conditions/action/message`
- `Condition`：`field/operator/pattern`

加载链路：`.local.md` -> `extract_frontmatter` -> `Rule.from_dict` -> `RuleEngine.evaluate_rules`。

参考：
- `plugins/hookify/core/config_loader.py:15`
- `plugins/hookify/core/config_loader.py:45`
- `plugins/hookify/core/config_loader.py:87`
- `plugins/hookify/core/config_loader.py:198`
- `plugins/hookify/core/rule_engine.py:35`

### F. Hook 返回协议（命令文档需与之保持一致）
规则命中后，运行时会输出不同协议：
- warn：`{"systemMessage": "..."}`
- block(Pre/PostToolUse)：`hookSpecificOutput.permissionDecision = deny`
- block(Stop)：`decision = block`

这决定了命令文档里 `action: warn|block` 的真实语义边界。

参考：
- `plugins/hookify/core/rule_engine.py:60`
- `plugins/hookify/hooks/pretooluse.py:55`
- `plugins/hookify/hooks/posttooluse.py:48`
- `plugins/hookify/hooks/stop.py:40`

### G. 测试与脚本现状（针对 commands 上下文）
- `plugins/hookify/commands` 目录无自动化测试。
- Hookify 插件内无 `tests/`；仅 `core/*.py` 含 `__main__` 手工测试段。
- 可复用脚本主要是通用 Hook 校验脚本（位于 `plugins/plugin-dev/...`），偏向 `hooks.json` 结构，不覆盖命令文档质量。

参考：
- `plugins/hookify/core/config_loader.py:277`
- `plugins/hookify/core/rule_engine.py:276`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:1`

## 关键代码路径与文件引用

目标目录文件：
- `plugins/hookify/commands/hookify.md`
- `plugins/hookify/commands/list.md`
- `plugins/hookify/commands/configure.md`
- `plugins/hookify/commands/help.md`

直接上游/注册：
- `plugins/hookify/README.md`
- `plugins/README.md`
- `.claude-plugin/marketplace.json`
- `plugins/hookify/.claude-plugin/plugin.json`

直接下游（命令产物的消费方）：
- `plugins/hookify/hooks/hooks.json`
- `plugins/hookify/hooks/pretooluse.py`
- `plugins/hookify/hooks/posttooluse.py`
- `plugins/hookify/hooks/stop.py`
- `plugins/hookify/hooks/userpromptsubmit.py`
- `plugins/hookify/core/config_loader.py`
- `plugins/hookify/core/rule_engine.py`

命令依赖的协同资产：
- `plugins/hookify/agents/conversation-analyzer.md`
- `plugins/hookify/skills/writing-rules/SKILL.md`
- `plugins/hookify/examples/dangerous-rm.local.md`
- `plugins/hookify/examples/console-log-warning.local.md`
- `plugins/hookify/examples/sensitive-files-warning.local.md`
- `plugins/hookify/examples/require-tests-stop.local.md`

## 依赖与外部交互

### 1) 文件系统交互
- 命令侧读写目标是“当前项目目录”的 `.claude/`，不是插件目录。
- 规则扫描/写入模式固定为 `.claude/hookify.*.local.md`。

参考：
- `plugins/hookify/commands/hookify.md:128`
- `plugins/hookify/commands/list.md:16`
- `plugins/hookify/core/config_loader.py:210`

### 2) 工具与子能力依赖
- `/hookify` 依赖 `Task`（会话分析）、`AskUserQuestion`（用户确认）、`Write`（落盘规则）。
- `/hookify:list` 与 `/hookify:configure` 依赖 `Glob` 做规则发现。
- 三个命令都要求先加载 `hookify:writing-rules` skill，降低语法漂移。

参考：
- `plugins/hookify/commands/hookify.md:4`
- `plugins/hookify/commands/hookify.md:9`
- `plugins/hookify/commands/list.md:3`
- `plugins/hookify/commands/configure.md:3`

### 3) 与 Hook 运行时的协议耦合
- 命令文档定义的 `event`、`pattern/conditions`、`action` 需与 `RuleEngine` 字段语义一致。
- 事件映射依赖 hook 执行器对 `tool_name` 的归类：Bash -> `bash`，Edit/Write/MultiEdit -> `file`，Stop/UserPromptSubmit 分别单独加载。

参考：
- `plugins/hookify/hooks/pretooluse.py:42`
- `plugins/hookify/hooks/posttooluse.py:36`
- `plugins/hookify/hooks/stop.py:37`
- `plugins/hookify/hooks/userpromptsubmit.py:37`

### 4) 外部进程/网络依赖
- 命令与运行时实现主要依赖 Python 标准库（`re/glob/json` 等）与本地文件系统。
- 当前链路不依赖外部网络 API；“外部交互”主要是与 Claude Code 的命令/Hook 协议交互。

参考：
- `plugins/hookify/README.md:300`
- `plugins/hookify/hooks/pretooluse.py:1`
- `plugins/hookify/core/config_loader.py:1`

## 风险、边界与改进建议

1. 分析 agent 调用标识存在语义漂移。
- 现状：`/hookify` 文本要求调用 `conversation-analyzer`，但示例 Task payload 使用 `subagent_type: general-purpose`。
- 风险：运行时可能不稳定命中专用 agent 风格，导致分析输出波动。
- 建议：在命令文档中固定 agent 标识（若平台支持），或至少在 prompt 中增加结构化输出约束。
- 参考：`plugins/hookify/commands/hookify.md:25`, `plugins/hookify/commands/hookify.md:33`, `plugins/hookify/agents/conversation-analyzer.md:2`

2. Stop/Prompt 的“简单 pattern”文档与运行时字段不完全一致。
- 现状：`event=stop/prompt` 若仅写 `pattern`，`Rule.from_dict` 会映射到 `field=content`；但 `RuleEngine` 对 stop/prompt 主要支持 `transcript/reason/user_prompt` 字段。
- 风险：用户按命令文档写出的部分规则可能不触发。
- 建议：命令模板对 `stop/prompt` 默认输出 `conditions`，并引导字段为 `transcript` 或 `user_prompt`。
- 参考：`plugins/hookify/core/config_loader.py:61`, `plugins/hookify/core/config_loader.py:67`, `plugins/hookify/core/rule_engine.py:205`, `plugins/hookify/core/rule_engine.py:226`

3. `file + pattern` 对 Write 工具覆盖不足。
- 现状：简单 `pattern` 会映射到 `new_text`；而 Write 场景常见字段为 `content`，`new_text` 取值可能为空。
- 风险：`/hookify` 生成的 file 规则在 Write 场景漏报。
- 建议：命令在生成 `event=file` 规则时优先用 `conditions` 同时覆盖 `new_text` 与 `content`，或运行时修正字段映射。
- 参考：`plugins/hookify/core/config_loader.py:65`, `plugins/hookify/core/rule_engine.py:235`, `plugins/hookify/core/rule_engine.py:239`

4. `configure` 的编辑策略较脆弱。
- 现状：文档仅建议字符串替换 `enabled: true/false`。
- 风险：遇到额外空格、注释、大小写或引号时可能替换失败或误替换。
- 建议：改为 frontmatter 解析后重写目标键，避免文本替换脆弱性。
- 参考：`plugins/hookify/commands/configure.md:77`, `plugins/hookify/commands/configure.md:82`

5. 示例与运算符语义易误用。
- 现状：`not_contains` 是“字面子串不包含”，但示例常写 `npm test|pytest|cargo test`（看起来像 regex OR）。
- 风险：误以为支持 OR 正则，导致 stop 规则行为与预期不一致。
- 建议：命令与示例统一改为 `operator: regex_match` + 负向表达，或拆成多个条件。
- 参考：`plugins/hookify/examples/require-tests-stop.local.md:8`, `plugins/hookify/examples/require-tests-stop.local.md:9`, `plugins/hookify/core/rule_engine.py:172`

6. commands 层缺少自动化回归测试。
- 现状：当前命令行为定义全在 markdown 指令中，无针对“命令-规则-运行时”闭环测试。
- 风险：文档改动导致行为漂移时难以及时发现。
- 建议：新增最小 smoke 套件：
  - 场景 1：`/hookify` 生成规则后，模拟 PreToolUse 输入验证 warn/block。
  - 场景 2：`/hookify:configure` 切换 enabled 后验证加载结果变化。
  - 场景 3：`/hookify:list` 在空目录/多规则目录下输出稳定结构。
- 参考：`plugins/hookify/commands/hookify.md:126`, `plugins/hookify/hooks/pretooluse.py:35`, `plugins/hookify/core/config_loader.py:225`
