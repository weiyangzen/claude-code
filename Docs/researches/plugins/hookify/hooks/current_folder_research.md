# DIR `plugins/hookify/hooks` 研究文档

## 场景与职责

`plugins/hookify/hooks` 是 Hookify 插件的运行时执行入口层，负责把 Claude Code 的 Hook 事件输入（stdin JSON）转译为统一规则引擎调用，并把结果按 Hook 协议输出到 stdout。

该目录职责可以概括为四件事：

1. 事件绑定：通过 `hooks.json` 把 Claude Code 事件映射到具体 Python 执行器。
2. 输入适配：读取各事件 payload，提取工具类型并映射成 Hookify 内部事件（`bash/file/stop/prompt`）。
3. 规则求值：调用 `hookify.core.config_loader.load_rules` 与 `hookify.core.rule_engine.RuleEngine`。
4. 协议返回：输出 Hook 协议 JSON，并统一降级策略（异常时也 `exit 0`，避免因 hook 自身故障导致流程硬中断）。

上下文定位：

- 上游调用方：Claude Code Hook runtime（通过插件 hooks 配置触发）。
- 同层配置：`plugins/hookify/hooks/hooks.json`。
- 下游被调用方：`plugins/hookify/core/config_loader.py`、`plugins/hookify/core/rule_engine.py`。
- 规则来源：项目根目录 `.claude/hookify.*.local.md`（由 `/hookify` 系列命令生成或手工维护）。

## 功能点目的

### 1) `hooks.json` 统一注册四类事件

目标是将 Hookify 规则能力覆盖在完整用户交互闭环上：

- `PreToolUse`：执行前检查（可 deny）。
- `PostToolUse`：执行后检查（当前实现仍可返回 deny 格式，但语义上更偏提醒）。
- `Stop`：会话结束前检查（可 block stop）。
- `UserPromptSubmit`：用户提交提示词前注入提醒。

对应文件：`plugins/hookify/hooks/hooks.json:1-49`。

### 2) 每个 hook 执行器做“最薄封装”

四个脚本功能几乎一致：

- 设置导入路径（依赖 `CLAUDE_PLUGIN_ROOT`）。
- 从 stdin 读取 JSON。
- 选择 event 过滤条件并 `load_rules(event=...)`。
- 调用 `RuleEngine.evaluate_rules`。
- 输出 JSON 并始终 `exit 0`。

这种设计的目的是把业务逻辑集中在 `core`，让入口脚本保持稳定、易替换。

对应文件：

- `plugins/hookify/hooks/pretooluse.py:1-74`
- `plugins/hookify/hooks/posttooluse.py:1-66`
- `plugins/hookify/hooks/stop.py:1-59`
- `plugins/hookify/hooks/userpromptsubmit.py:1-58`

### 3) 与命令层解耦，支持“规则即时生效”

hooks 目录不负责规则创建，只负责读取 `.claude/hookify.*.local.md`。规则创建/管理由命令层负责：

- `/hookify` 生成规则：`plugins/hookify/commands/hookify.md:84-156`
- `/hookify:list` 枚举规则：`plugins/hookify/commands/list.md:14-60`
- `/hookify:configure` 启停规则：`plugins/hookify/commands/configure.md:14-117`

因此 hooks 层实现“无重启动态生效”，命令层实现“规则生产与维护”。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程（端到端）

1. Claude Code 加载插件并读取 hooks 配置（默认 hooks 路径是 `./hooks/hooks.json`，见插件结构参考 `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`）。
2. 事件触发时执行命令：
   - `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/pretooluse.py`
   - `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/posttooluse.py`
   - `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/stop.py`
   - `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/userpromptsubmit.py`
3. 入口脚本读取 stdin JSON。
4. Pre/Post 根据 `tool_name` 做事件折叠：
   - `Bash -> bash`
   - `Edit/Write/MultiEdit -> file`
   - 其他工具 event 为 `None`（仅匹配 `event: all` 规则）。
5. `load_rules(event=...)` 扫描 `.claude/hookify.*.local.md`，解析 frontmatter，过滤 `enabled` 与 `event`。
6. `RuleEngine.evaluate_rules(rules, input_data)` 逐条执行条件匹配，聚合 warning/block 结果。
7. 输出 JSON 到 stdout，脚本统一 `sys.exit(0)`。

关键位置：

- `plugins/hookify/hooks/hooks.json:4-47`
- `plugins/hookify/hooks/pretooluse.py:39-57`
- `plugins/hookify/hooks/posttooluse.py:34-50`
- `plugins/hookify/hooks/stop.py:34-42`
- `plugins/hookify/hooks/userpromptsubmit.py:34-42`
- `plugins/hookify/core/config_loader.py:198-241`
- `plugins/hookify/core/rule_engine.py:35-94`

### B. 数据结构与匹配语义（由 hooks 调用）

hooks 层本身不定义复杂结构，核心结构由 `core` 提供并被 hooks 消费：

- `Condition(field, operator, pattern)`：单条件。
- `Rule(name, enabled, event, pattern, conditions, action, tool_matcher, message)`：完整规则。
- simple pattern 兼容：`pattern` 会自动转换为 condition。

匹配语义：

- 同一规则内多 conditions 是 AND。
- 命中 block 规则优先。
- Pre/Post 的 block 输出：`hookSpecificOutput.permissionDecision = "deny"`。
- Stop 的 block 输出：`decision = "block"` + `reason`。

关键位置：

- `plugins/hookify/core/config_loader.py:15-84`
- `plugins/hookify/core/rule_engine.py:53-84`
- `plugins/hookify/core/rule_engine.py:96-125`

### C. Hook 协议与命令

1. 输入协议（stdin JSON）

常见字段来自 Claude Hook runtime：

- `hook_event_name`
- `tool_name`
- `tool_input`
- `transcript_path`（Stop）
- `user_prompt`（UserPromptSubmit）

参考示例结构：`examples/hooks/bash_command_validator_example.py:13-27`、`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`（脚本会生成样例输入，README 说明见 `plugins/plugin-dev/skills/hook-development/scripts/README.md:29-61`）。

2. 输出协议（stdout JSON）

- 告警：`{"systemMessage": "..."}`
- Tool 阶段阻断：
  - `{"hookSpecificOutput": {"hookEventName": "PreToolUse|PostToolUse", "permissionDecision": "deny"}, "systemMessage": "..."}`
- Stop 阶段阻断：
  - `{"decision": "block", "reason": "...", "systemMessage": "..."}`
- 无命中：`{}`

3. 错误与退出策略

- import 或运行异常时，脚本输出错误 message，但仍 `exit 0`。
- 设计意图是“hook 故障默认降级放行”，避免把规则系统变成单点故障。

对应：

- `plugins/hookify/hooks/pretooluse.py:25-33`
- `plugins/hookify/hooks/pretooluse.py:61-70`
- `plugins/hookify/hooks/posttooluse.py:21-27`
- `plugins/hookify/hooks/stop.py:21-27`
- `plugins/hookify/hooks/userpromptsubmit.py:21-27`

### D. 与配置/文档/示例的协同关系

- 文档定义了规则格式和事件语义：`plugins/hookify/README.md:71-260`。
- examples 提供规则样板，直接影响 hooks 运行输入：
  - `plugins/hookify/examples/dangerous-rm.local.md`
  - `plugins/hookify/examples/console-log-warning.local.md`
  - `plugins/hookify/examples/sensitive-files-warning.local.md`
  - `plugins/hookify/examples/require-tests-stop.local.md`
- skill 固化编写规范，间接决定 hooks 输入质量：`plugins/hookify/skills/writing-rules/SKILL.md:11-373`。

## 关键代码路径与文件引用

### 目录内主路径

- 事件注册：`plugins/hookify/hooks/hooks.json:1-49`
- PreToolUse 执行器：`plugins/hookify/hooks/pretooluse.py:1-74`
- PostToolUse 执行器：`plugins/hookify/hooks/posttooluse.py:1-66`
- Stop 执行器：`plugins/hookify/hooks/stop.py:1-59`
- UserPromptSubmit 执行器：`plugins/hookify/hooks/userpromptsubmit.py:1-58`

### 直接被调用代码

- 规则加载：`plugins/hookify/core/config_loader.py:198-241`
- 单文件解析：`plugins/hookify/core/config_loader.py:244-275`
- 决策引擎：`plugins/hookify/core/rule_engine.py:35-94`
- 字段提取（Stop/Prompt/Bash/File）：`plugins/hookify/core/rule_engine.py:182-254`

### 上游配置与调用语境

- 插件元信息：`plugins/hookify/.claude-plugin/plugin.json:1-9`
- 插件总览入口：`plugins/README.md:22`
- hooks 默认发现规则：`plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:259-263`
- 命令层如何生产规则：
  - `plugins/hookify/commands/hookify.md:82-156`
  - `plugins/hookify/commands/list.md:14-60`
  - `plugins/hookify/commands/configure.md:14-117`
  - `plugins/hookify/commands/help.md:16-150`

### 测试/脚本/文档支撑

- hooks 开发通用脚本文档：`plugins/plugin-dev/skills/hook-development/scripts/README.md:1-136`
- schema 校验：`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`
- hook 执行测试：`plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`
- hook linter：`plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh`

## 依赖与外部交互

### 1) 运行时依赖

- Python 3（`hooks.json` 中全部使用 `python3` 命令）。
- Python 标准库（`json/os/sys` 等），无第三方包。
- 环境变量：
  - `CLAUDE_PLUGIN_ROOT`（用于构造 import path）。

对应：`plugins/hookify/hooks/hooks.json:9,20,31,42`、`plugins/hookify/hooks/pretooluse.py:14-23`。

### 2) Claude Hook runtime 交互

- 输入：stdin JSON（由宿主注入）。
- 输出：stdout JSON（供宿主解析决策）。
- stderr：core 在解析失败时会写 warning 到 stderr，但 hook 脚本最终仍 0 退出。

对应：

- `plugins/hookify/hooks/pretooluse.py:39,59`
- `plugins/hookify/core/config_loader.py:230-239`
- `plugins/hookify/core/rule_engine.py:215-225,271-273`

### 3) 文件系统交互

- 读取规则：相对 cwd 的 `.claude/hookify.*.local.md`。
- Stop 场景可能读取 `transcript_path` 文件内容。

对应：

- `plugins/hookify/core/config_loader.py:210-212`
- `plugins/hookify/core/rule_engine.py:207-225`

### 4) 与其他插件/工具链交互边界

- hookify hooks 自身未直接依赖其他插件。
- 但仓库提供了 hook 开发通用测试/校验脚本（plugin-dev），可作为研发辅助。
- `plugins/hookify` 目录无独立 `tests/` 自动化测试；当前主要依赖手工测试、示例与命令引导。

## 风险、边界与改进建议

1. Stop/Prompt 的 simple `pattern` 默认映射存在失配风险。
- 现状：`Rule.from_dict` 对非 `bash/file` 事件把 simple pattern 映射到 `content` 字段；而 Stop/Prompt 常见字段是 `transcript/reason/user_prompt`。
- 影响：用户只写 `pattern` 的 `event: stop|prompt` 规则可能不触发。
- 建议：为 `stop/prompt` 指定更合理默认字段，或在加载阶段做显式告警。

2. `PostToolUse` 使用 `permissionDecision: deny` 的行为语义不明确。
- 现状：engine 对 `PreToolUse` 与 `PostToolUse` 共用 deny 输出。
- 风险：Post 阶段工具通常已执行，deny 可能只剩提示价值，容易误导用户“可回滚”。
- 建议：对 Post 统一降级为 warn，或在 README 明确“Post deny 的宿主语义”。

3. 手写 frontmatter 解析器能力边界较窄（位于 core，但直接影响 hooks 可靠性）。
- 风险：复杂 YAML/转义场景可能解析偏差，hooks 侧只会得到“规则未命中/误命中”。
- 建议：改用标准 YAML 解析或补充强约束校验。

4. 当前错误处理偏“静默放行”。
- 现状：入口脚本异常始终 `exit 0`。
- 风险：规则系统失效时不会阻断，适合可用性但降低安全场景确定性。
- 建议：增加可配置故障策略（严格模式下关键事件可 fail-closed，默认仍 fail-open）。

5. 缺少目录级自动化测试。
- 现状：无 `plugins/hookify/tests`；仅 core `__main__` 手工片段与通用测试脚本。
- 建议：新增最小测试集：
  - 事件映射（Pre/Post/Stop/Prompt）
  - 输出协议断言（deny/block/warn）
  - 导入路径构造（`CLAUDE_PLUGIN_ROOT`）
  - 规则读取路径（cwd 与 `CLAUDE_PROJECT_DIR` 差异）

6. cwd 假设是隐含边界。
- 现状：规则路径固定相对 `.claude/`。
- 风险：若 Claude runtime 工作目录不是项目根，规则会“存在但不生效”。
- 建议：优先使用 hook input 的 `cwd` 或环境变量 `CLAUDE_PROJECT_DIR` 作为规则根路径。

