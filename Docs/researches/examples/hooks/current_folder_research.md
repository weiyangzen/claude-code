# DIR `examples/hooks` 研究文档

## 场景与职责

`examples/hooks` 当前仅包含一个示例脚本 `bash_command_validator_example.py`，定位是 Claude Code Hook 的最小可运行参考实现，不是仓库主流程中的自动执行模块。

该目录的职责是：
- 演示如何把外部脚本接入 `PreToolUse` 事件。
- 演示如何读取 Hook stdin JSON、按规则做命令校验、并用 exit code 决定“放行/阻断”。
- 给团队提供“命令规范前置治理”的可复制模板（把低效 Bash 用法在执行前拦住）。

调用关系：
1. 调用方
- 终端用户或管理员在 Claude Code 配置中声明 `PreToolUse` + `matcher: "Bash"`，并通过 `command` 指向该脚本。
2. 被调用方
- Claude Code 的 Hook 运行器在 Bash 工具执行前调用该脚本。
3. 被影响对象
- Bash 工具请求（`tool_input.command`）会被该脚本检查并可能被 `exit 2` 阻断。

仓库边界：
- 仓库内没有 CI、workflow、或业务脚本直接调用 `examples/hooks/*`；它是文档型示例资产。

## 功能点目的

### 1) Bash 命令规范校验
目标是把常见“可替代但低效”的命令用法提前拦截：
- `grep ...`（无管道）提示改用 `rg`。
- `find <path> -name ...` 提示改用 `rg --files` 组合。

### 2) Hook 协议示例化
脚本同时承担“协议样板”作用，展示：
- stdin 输入读取与 JSON 解析。
- 仅在 `tool_name == "Bash"` 时生效。
- 用 stderr 回传提示，用 `exit 2` 表示阻断。

### 3) Python 版实现模板
相比 shell 版示例，此文件提供 Python 实现路径（`json/re/sys`），便于扩展更复杂规则逻辑。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 关键流程

1. 读取输入
- `json.load(sys.stdin)` 解析 hook 输入。
- 解析失败时 stderr 输出错误并 `exit 1`。

2. 事件过滤
- 当 `tool_name != "Bash"` 时直接 `exit 0`（不干预其它工具）。

3. 提取命令
- 从 `tool_input.command` 取待校验命令。
- 命令为空则放行。

4. 规则匹配
- 遍历 `_VALIDATION_RULES`，通过 `re.search(pattern, command)` 收集命中项。

5. 输出与决策
- 若有命中：逐条 stderr 输出 `• message`，并 `exit 2` 阻断。
- 未命中：`exit 0`。

### B. 数据结构

脚本用一个常量列表表达规则：
- 类型：`list[tuple[str, str]]`
- 结构：`(regex_pattern, human_message)`

当前两条规则：
1. `^grep\b(?!.*\|)`
- 含义：命令以 `grep` 起始且后续不含 `|` 才触发。
2. `^find\s+\S+\s+-name\b`
- 含义：命令以 `find <单个参数> -name` 形态触发。

### C. 协议与返回语义

1. Hook 注册配置（脚本 docstring 内给了示例）
- 事件：`PreToolUse`
- 匹配器：`Bash`
- 命令：`python3 /path/to/claude-code/examples/hooks/bash_command_validator_example.py`

2. 输入协议（运行时 JSON）
- 关键字段：`tool_name`、`tool_input.command`
- 该字段结构与 hook-development 的 `test-hook.sh --create-sample PreToolUse` 输出一致（同样包含 `hook_event_name/tool_name/tool_input`）。

3. 退出码语义
- `0`：放行。
- `1`：输入异常（示例中是 JSON 解析失败）。
- `2`：阻断当前工具调用（策略拒绝）。

### D. 命令级实测（本次研究）

执行命令：
- `printf '{...}' | python3 examples/hooks/bash_command_validator_example.py`

实测结果：
1. `grep foo bar.txt` -> `exit 2`，stderr 提示改用 `rg`。
2. `find . -name "*.py"` -> `exit 2`，stderr 提示改用 `rg --files`。
3. `ls -la` -> `exit 0`。
4. 非法 JSON -> `exit 1`，stderr 输出 JSON 解析错误。
5. `grep foo bar.txt | head` -> `exit 0`（被负向前瞻放过）。
6. `sudo grep foo bar.txt` -> `exit 0`（规则未覆盖前缀）。
7. `find . -type f -name "*.py"` -> `exit 0`（规则要求 `-name` 紧跟第二段参数）。

## 关键代码路径与文件引用

### 目标目录
- `examples/hooks/bash_command_validator_example.py`

### 调用方与配置上下文
- `examples/hooks/bash_command_validator_example.py`（文件头 docstring 内嵌 `PreToolUse` 配置示例）
- `plugins/security-guidance/hooks/hooks.json`（仓库内生产化 `PreToolUse` 配置参考，`matcher` + `type: command`）
- `plugins/hookify/hooks/hooks.json`（多事件 hook 配置样式，包含 `timeout` 字段）

### 被调用方与实现对照
- `plugins/security-guidance/hooks/security_reminder_hook.py`（Python Hook，读取 stdin JSON，并在策略命中时输出系统消息）
- `plugins/plugin-dev/skills/hook-development/examples/validate-bash.sh`（同类 Bash 校验 Hook，使用 `exit 2` + `permissionDecision` JSON）
- `plugins/hookify/hooks/pretooluse.py`（PreToolUse 执行器，展示通用输入读取和统一输出 JSON）
- `plugins/hookify/core/rule_engine.py`（规则引擎对 `PreToolUse` 的 `permissionDecision: deny` 输出结构）

### 测试/校验脚本上下文
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`
  - 可生成 `PreToolUse` 样例输入并解读 exit code。
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`
  - 校验 `hooks.json` 结构、事件名、类型、timeout 与硬编码路径。
- `plugins/plugin-dev/skills/hook-development/scripts/README.md`
  - 提供 hook 编写->测试->配置验证的工作流。

### 文档与演进上下文
- `plugins/README.md`（官方插件目录中对 Hook 能力的总览）
- `CHANGELOG.md`（Hook 行为演进：`exit code 2` 展示、`additionalContext`、`updatedInput`、PreToolUse 可修改输入）

### 研究流程脚本（与目标目录非运行时耦合）
- `.ops/generate_research_blueprint_checklist.sh`
- `.ops/generate_daily_research_todo.sh`
- `Docs/researches/blueprint_checklist.md`

## 依赖与外部交互

### 本地依赖
- 解释器：`python3`
- Python 标准库：`json`、`re`、`sys`

### 运行时交互
- 输入通道：stdin JSON（由 Claude Code Hook 运行器提供）。
- 输出通道：stderr（人类提示/阻断原因），stdout 基本未使用。
- 决策通道：进程退出码（0/1/2）。

### 配置依赖
- 需要用户在 hooks 配置中显式注册该脚本，否则不会执行。
- 示例命令使用绝对路径占位符 `/path/to/...`，迁移到真实环境时必须替换。

### 外部资源
- 文件注释引用官方文档：`https://docs.anthropic.com/en/docs/claude-code/hooks`
- 脚本本身无网络访问、无第三方包依赖。

### 测试现状
- 该目录没有专属自动化测试。
- 可借助 `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh` 做离线回归测试。

## 风险、边界与改进建议

### 风险

1. 规则覆盖不足（可绕过）
- `sudo grep`、`command grep ...`、`find . -type f -name ...` 等常见变体不会触发。
- `grep ... | ...` 被设计为放行，但在“强制统一用 rg”策略下会形成漏拦。

2. 正则语义容易与策略目标不一致
- 当前模式是“只拦最窄写法”，更像演示而非生产策略。

3. 可移植性风险
- docstring 示例命令写死绝对路径，不符合插件中常见的 `${CLAUDE_PLUGIN_ROOT}` 可移植做法。

4. 自动化保障缺失
- 无 CI 测试，规则修改后易出现静默回归。

### 边界

1. 该目录是示例，不承担企业级策略完备性。
2. 仅处理 `Bash` 工具，不处理 `Edit/Write/MultiEdit` 等其他工具。
3. 不处理复杂 shell AST，也不执行命令重写（仅阻断+提示）。

### 改进建议

1. 增加回归测试资产
- 在 `examples/hooks/` 下增加 `testcases/*.json` 与一个最小测试脚本，覆盖 safe/block/error/edge 场景并断言 exit code。

2. 提升规则表达能力
- 由简单 regex 升级为分词/AST 级判断（至少先用 `shlex` 做 token 解析），减少误判与漏判。

3. 统一输出结构
- 参考 `hookify`/`plugin-dev` 示例，在阻断时输出结构化 JSON（含 `hookSpecificOutput.permissionDecision` 和 `systemMessage`），提升跨客户端一致性。

4. 改进配置示例
- 在示例中增加 `${CLAUDE_PLUGIN_ROOT}` 风格写法，避免用户直接复制后路径失效。

5. 文档补充
- 为该目录补 `README.md`，给出“安装位置、最小 hooks 配置、验证命令、故障排查”四步指引。

