# DIR `plugins/hookify` 研究文档

## 场景与职责

`plugins/hookify` 是一个“规则驱动 Hook 框架”插件，目标是把用户的自然语言偏好/约束，沉淀为项目级 `.claude/hookify.*.local.md` 规则文件，并在 Claude Code Hook 事件中动态生效。

核心职责分为三层：
- 规则生成与管理层：通过 `/hookify`、`/hookify:list`、`/hookify:configure`、`/hookify:help` 指导或执行规则创建/查看/启停。
- 规则加载与判定层：`core/config_loader.py` 解析 frontmatter + message，`core/rule_engine.py` 执行条件匹配与动作决策。
- 事件执行层：`hooks/*.py` 挂在 `PreToolUse/PostToolUse/Stop/UserPromptSubmit`，读取 stdin JSON，调用规则引擎并返回协议 JSON。

在仓库中的上下文定位：
- 市场入口：`.claude-plugin/marketplace.json` 将 `hookify` 注册为可安装插件。
- 插件清单入口：`plugins/README.md` 将其定义为命令+agent+skill 的组合插件。
- 同类参考：`plugins/security-guidance`、`plugins/ralph-wiggum` 等插件也通过 `hooks/hooks.json` 注册事件脚本。

## 功能点目的

主要功能点及设计意图：
- `/hookify`（`commands/hookify.md`）
  - 目的：把“避免某类行为”的口头约束转成可执行规则文件。
  - 模式：
    - 有参数：根据显式要求建规则。
    - 无参数：调用 `conversation-analyzer` agent 分析会话中的反复问题后建规则。
- `/hookify:list`（`commands/list.md`）
  - 目的：枚举 `.claude/hookify.*.local.md` 的启用状态、事件、模式，降低规则可见性成本。
- `/hookify:configure`（`commands/configure.md`）
  - 目的：通过交互切换 `enabled: true/false`，让规则可快速启停。
- `/hookify:help`（`commands/help.md`）
  - 目的：统一解释规则格式、事件类型、常见排障。
- `agents/conversation-analyzer.md`
  - 目的：将“用户纠正/不满信号”结构化为可匹配模式（工具类型、正则、严重级别）。
- `skills/writing-rules/SKILL.md`
  - 目的：标准化规则写法（frontmatter 字段、operators、字段映射、命名与测试建议）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 运行流程

1. Claude Code 在插件加载后读取 `hooks/hooks.json`。
2. 事件触发时执行对应脚本（`pretooluse.py`/`posttooluse.py`/`stop.py`/`userpromptsubmit.py`）。
3. 脚本从 stdin 读取 hook input JSON，按工具推断 event（`bash`/`file`/`stop`/`prompt`）。
4. `load_rules(event=...)` 扫描当前工作目录下 `.claude/hookify.*.local.md`。
5. `extract_frontmatter` + `Rule.from_dict` 生成规则对象。
6. `RuleEngine.evaluate_rules` 逐条匹配 conditions，汇总 warning/blocking。
7. 输出 JSON 到 stdout，由 Claude Code 解释为允许/拒绝/系统消息。

### 2) 核心数据结构

`core/config_loader.py`：
- `Condition`
  - `field`：如 `command`、`file_path`、`new_text`、`transcript`、`user_prompt`。
  - `operator`：`regex_match`/`contains`/`equals`/`not_contains`/`starts_with`/`ends_with`。
  - `pattern`：匹配文本或正则。
- `Rule`
  - `name`、`enabled`、`event`、`action`、`tool_matcher`、`message`。
  - 兼容 `pattern`（simple）与 `conditions`（advanced）两种写法。
  - `pattern` 会被转换为单条件；`event=bash` 映射到 `field=command`，`event=file` 映射到 `field=new_text`，其它 event 映射到 `field=content`。

### 3) 匹配与决策逻辑

`core/rule_engine.py`：
- 使用 `compile_regex()` + `lru_cache(maxsize=128)` 缓存正则编译。
- `_rule_matches` 采用“全部条件 AND”语义。
- `_extract_field` 针对工具类型提取字段：
  - Bash：`tool_input.command`
  - Write/Edit：`content/new_string/old_string/file_path`
  - MultiEdit：拼接 edits 的 `new_string`
  - Stop：可读 `reason` 和 `transcript_path` 指向的会话文件
  - UserPromptSubmit：`user_prompt`
- `evaluate_rules` 汇总结果：
  - 有 `action=block` 命中：
    - Stop 返回 `{decision: "block", reason, systemMessage}`
    - Pre/PostToolUse 返回 `{hookSpecificOutput: {permissionDecision: "deny"}, systemMessage}`
  - 仅 warning：返回 `{systemMessage}`
  - 无命中：返回 `{}`

### 4) Hook I/O 协议实现要点

输入：Claude Code 注入的 stdin JSON（包含 `hook_event_name`、`tool_name`、`tool_input`、`transcript_path` 等）。

输出：
- 标准 JSON 输出到 stdout。
- 脚本异常时仍 `sys.exit(0)`，并输出错误 systemMessage，避免“因 Hook 自身失败阻断工具”。

路径与导入：
- 脚本依赖 `CLAUDE_PLUGIN_ROOT` 动态注入 `sys.path`（插件目录及其父目录），保证 `from hookify.core...` 可导入。

### 5) 关键命令/配置

- 插件元信息：`.claude-plugin/plugin.json`
- 事件绑定：`hooks/hooks.json`
- 用户规则文件模式：`.claude/hookify.*.local.md`
- 手工验证（仓库内通用脚本）：
  - `bash plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh --create-sample PreToolUse`
  - `bash plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh <hooks.json>`

### 6) 研究过程中的实测结论

- Python 语法检查通过：
  - `python3 -m py_compile plugins/hookify/hooks/*.py plugins/hookify/core/*.py`
- PreToolUse 端到端验证（临时目录 + 临时规则）显示可返回 deny 协议：
  - 输出示例：`{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"deny"}, ...}`
- 发现 Stop 示例规则逻辑偏差：
  - `examples/require-tests-stop.local.md` 使用 `not_contains` + `pattern: npm test|pytest|cargo test`，实际是“子串不包含字面量 `npm test|pytest|cargo test`”，会误判并触发阻断。

## 关键代码路径与文件引用

入口与注册：
- `plugins/hookify/.claude-plugin/plugin.json`
- `plugins/hookify/hooks/hooks.json`
- `.claude-plugin/marketplace.json`
- `plugins/README.md`

Hook 执行器：
- `plugins/hookify/hooks/pretooluse.py`
- `plugins/hookify/hooks/posttooluse.py`
- `plugins/hookify/hooks/stop.py`
- `plugins/hookify/hooks/userpromptsubmit.py`

规则核心：
- `plugins/hookify/core/config_loader.py`
- `plugins/hookify/core/rule_engine.py`

规则生产与管理接口：
- `plugins/hookify/commands/hookify.md`
- `plugins/hookify/commands/list.md`
- `plugins/hookify/commands/configure.md`
- `plugins/hookify/commands/help.md`

规则生成协作者：
- `plugins/hookify/agents/conversation-analyzer.md`
- `plugins/hookify/skills/writing-rules/SKILL.md`

示例规则：
- `plugins/hookify/examples/dangerous-rm.local.md`
- `plugins/hookify/examples/console-log-warning.local.md`
- `plugins/hookify/examples/sensitive-files-warning.local.md`
- `plugins/hookify/examples/require-tests-stop.local.md`

外部参考（测试/规范）：
- `plugins/plugin-dev/skills/hook-development/scripts/README.md`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`
- `CHANGELOG.md`（hooks 协议能力演进）

## 依赖与外部交互

运行时依赖：
- Python 3.7+（README 声明）
- Python 标准库：`json/os/sys/re/glob/dataclasses/functools` 等
- Claude Code 注入环境变量：
  - `CLAUDE_PLUGIN_ROOT`（导入路径构造）
  - Hook input JSON 中的 `transcript_path`、`tool_input`、`user_prompt` 等字段

文件系统交互：
- 读取规则：`.claude/hookify.*.local.md`
- Stop 事件可能读取 transcript 文件（`transcript_path`）

与 Claude Code Hook 协议交互：
- 输入通道：stdin JSON
- 输出通道：stdout JSON（`systemMessage`/`decision`/`hookSpecificOutput.permissionDecision`）
- 错误策略：Hook 自身异常默认降级为“放行+提示”，不抛非零退出码

与仓库其它模块关系：
- `matchers/`、`utils/` 当前为空，占位目录，暂无运行时逻辑。
- `plugin-dev` 的 hook-development 脚本可做通用测试/校验，但其 schema 假设与当前插件 `hooks/hooks.json` 结构不完全一致。

## 风险、边界与改进建议

1. Stop/Prompt 的 simple `pattern` 映射存在可用性缺口。
- 现状：`Rule.from_dict` 对非 bash/file 事件把 `pattern` 映射到 `field=content`；`_extract_field` 对 Stop/UserPromptSubmit 不提供 `content` 字段。
- 结果：`event: stop|prompt` 且只写 `pattern` 的规则通常不生效。
- 建议：
  - `event=stop` 默认映射到 `transcript` 或 `reason`。
  - `event=prompt` 默认映射到 `user_prompt`。
  - 在加载阶段对不支持字段给显式告警。

2. `not_contains` 与“多候选模式”语义易误用，示例已出现误判。
- 现状：`not_contains` 是纯子串判断，不支持 `|` 正则语义。
- 结果：`pattern: npm test|pytest|cargo test` 不会按“任一命中”理解，可能导致错误阻断。
- 建议：
  - 新增 `not_regex_match` operator，或
  - 在文档和示例中改成 `regex_match` + 反向逻辑组合。

3. `PostToolUse` 的 block 输出语义不清晰。
- 现状：`evaluate_rules` 对 `PostToolUse` 也返回 `permissionDecision: deny`。
- 风险：Post 阶段通常已执行完成，deny 的实际约束力可能有限/依赖宿主实现。
- 建议：Post 事件统一降级为 warning/systemMessage，或明确文档说明其行为。

4. Frontmatter 解析器是手写 YAML 子集，鲁棒性边界明显。
- 现状：`extract_frontmatter` 通过字符串缩进与分隔符手工解析，支持有限。
- 风险：复杂 YAML（嵌套、转义、冒号/逗号边界）易出现静默解析偏差。
- 建议：使用标准 YAML 解析库，或至少补充解析单测并增强错误提示。

5. Hook 配置校验链路不统一。
- 现状：仓库中多个插件（含 hookify）使用 `{"description":...,"hooks":{...}}` 结构；`validate-hook-schema.sh` 当前按“顶层事件键”遍历并强制 matcher，直接用于 hookify 会报错。
- 建议：
  - 校验脚本兼容插件当前 schema（顶层 `hooks` 容器）。
  - 或在脚本 README 明确“适用 schema 版本/转换方式”。

6. 自动化测试缺失。
- 现状：`plugins/hookify` 无专属测试目录或 CI 测试用例。
- 风险：规则语义回归（尤其 event/field 映射、operator 行为）难以及时发现。
- 建议：
  - 增加最小测试集：frontmatter 解析、operator 行为、每类 event 的匹配与输出协议。
  - 引入基于 `test-hook.sh` 的 smoke 测试样例。

7. 工作目录依赖隐式。
- 现状：`load_rules` 固定相对路径 `.claude/hookify.*.local.md`。
- 风险：若宿主进程 cwd 非项目根，规则会“看起来存在但不生效”。
- 建议：优先使用 hook input 的 `cwd` 或 `CLAUDE_PROJECT_DIR` 作为规则根目录。
