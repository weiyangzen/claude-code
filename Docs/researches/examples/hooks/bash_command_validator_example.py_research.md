# FILE `examples/hooks/bash_command_validator_example.py` 研究文档

## 场景与职责

`examples/hooks/bash_command_validator_example.py` 是一个 Claude Code `PreToolUse` 命令型 Hook 示例脚本，作用边界非常清晰：

1. 仅在 `Bash` 工具执行前触发（通过 hooks 配置中的 `matcher: "Bash"`）。
2. 从 Hook stdin 读取 JSON 输入，提取 `tool_input.command`。
3. 对命令字符串进行正则规则校验。
4. 命中规则时输出可读提示到 stderr，并通过 `exit 2` 阻断本次工具调用。
5. 未命中规则时 `exit 0` 放行。

该文件在仓库中定位是“可复制的最小参考实现”，不是仓库内任何业务流程或 CI 的直接运行组件。通过全文检索，仓库内没有其他代码文件直接 import/exec 这个脚本；它由使用者在 hooks 配置中显式注册后，由 Claude Code Hook 运行器调用。

## 功能点目的

### 1. 提供 Bash 命令治理样板
当前实现把“建议用 `rg` 替代低效查找命令”前置到执行前：

- 拦截 `grep`（无管道场景）并提示使用 `rg`。
- 拦截 `find <path> -name ...` 并提示使用 `rg --files` 组合。

核心目标不是安全防护，而是效率与工具规范统一。

### 2. 演示 Hook 最小协议
该示例把 PreToolUse 命令 Hook 的关键协议信息浓缩在一个文件中：

- 输入：stdin JSON（`tool_name`、`tool_input.command`）。
- 输出：stderr 人类可读提示（阻断时）。
- 控制：退出码 `0/1/2`。

### 3. 作为 Python 版本的扩展基线
相比 shell 示例（如 `plugins/plugin-dev/skills/hook-development/examples/validate-bash.sh`），此文件展示了 Python 写法（`json` + `re`），便于后续扩展复杂规则或复用到插件脚本中。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1. 关键流程

1. 解析输入
- `json.load(sys.stdin)` 读取 Hook 输入。
- JSON 解析失败时：stderr 输出错误并 `sys.exit(1)`。

2. 工具过滤
- `tool_name != "Bash"` 直接 `sys.exit(0)`，避免影响非 Bash 工具。

3. 命令提取
- `tool_input = input_data.get("tool_input", {})`
- `command = tool_input.get("command", "")`
- 命令为空直接放行。

4. 规则匹配
- `_validate_command(command)` 遍历 `_VALIDATION_RULES`。
- 每条规则用 `re.search(pattern, command)` 判定并收集命中消息。

5. 阻断输出
- 若有命中：逐条输出 `• message` 到 stderr，`sys.exit(2)` 阻断。
- 无命中：函数结束，进程返回 0（放行）。

### 2. 数据结构与规则设计

规则存储结构：
- 常量 `_VALIDATION_RULES: list[tuple[str, str]]`
- 元组结构：`(regex_pattern, human_message)`

当前规则：
1. `^grep\b(?!.*\|)`
- 含义：命令以 `grep` 起始，且命令行中不存在 `|`。
- 结果：`grep foo file` 会拦截，`grep foo file | head` 不拦截。

2. `^find\s+\S+\s+-name\b`
- 含义：匹配 `find <单段路径> -name` 紧邻写法。
- 结果：`find . -name "*.py"` 会拦截；`find . -type f -name "*.py"` 不会命中。

### 3. Hook 协议与退出码

仓库内 hook 开发文档与脚本上下文（`plugins/plugin-dev/skills/hook-development/SKILL.md`、`scripts/test-hook.sh`）与该示例行为一致：

- Hook 输入通过 stdin 提供 JSON。
- PreToolUse 关键字段含 `tool_name` 与 `tool_input`。
- `exit 0` 表示放行；`exit 2` 用于阻断；其他非 0 常见于错误状态。

该示例采用 stderr 承载阻断提示文本，属于“文本提示型阻断”；未输出结构化 JSON（例如 `permissionDecision`）。

### 4. 本次仓内实测命令

在仓库根目录执行：

```bash
printf '{"tool_name":"Bash","tool_input":{"command":"grep foo bar.txt"}}' | \
  python3 examples/hooks/bash_command_validator_example.py
# exit=2, stderr: Use 'rg' ...

printf '{"tool_name":"Bash","tool_input":{"command":"find . -name \"*.py\""}}' | \
  python3 examples/hooks/bash_command_validator_example.py
# exit=2, stderr: Use 'rg --files ...'

printf '{"tool_name":"Bash","tool_input":{"command":"grep foo bar.txt | head"}}' | \
  python3 examples/hooks/bash_command_validator_example.py
# exit=0

printf '{bad json}' | python3 examples/hooks/bash_command_validator_example.py
# exit=1, stderr: Invalid JSON input
```

## 关键代码路径与文件引用

### 1. 目标文件（核心实现）
- `examples/hooks/bash_command_validator_example.py:1-83`
  - 规则定义：`35-45`
  - 校验函数：`48-53`
  - 主流程：`56-79`
  - 配置示例（docstring）：`13-27`

### 2. 调用方/配置路径（上下文）
- `examples/hooks/bash_command_validator_example.py:13-27`
  - 内嵌 `hooks` JSON 片段，展示 `PreToolUse` + `matcher: "Bash"` + `type: "command"` 的注册方式。
- `plugins/security-guidance/hooks/hooks.json:1-14`
  - 仓库内真实 `PreToolUse` command hook 配置样式（可作为调用配置参考）。
- `plugins/hookify/hooks/hooks.json:1-47`
  - 展示多事件 hooks 结构与 `timeout` 字段。

### 3. 协议/脚本/文档依赖（被调用关系与工程约束）
- `plugins/plugin-dev/skills/hook-development/SKILL.md:294-320`
  - 退出码语义、Hook stdin 输入字段说明。
- `plugins/plugin-dev/skills/hook-development/SKILL.md:497-518`
  - 匹配 hooks 并行执行语义（设计上不要依赖顺序）。
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:22-95,180-208`
  - 生成 PreToolUse 样例输入并判读 `exit code` 的测试工具。
- `plugins/plugin-dev/skills/hook-development/examples/validate-bash.sh:1-41`
  - Bash 版本的同类验证示例，体现同样的 `exit 2` 阻断路径。

### 4. 与 settings 的关联约束
- `examples/settings/README.md:3-8,18,27`
  - settings 层级、可禁用用户 hooks、sandbox 仅作用于 Bash 工具等注意事项。
- `examples/settings/settings-strict.json:13`
  - `allowManagedHooksOnly: true` 会限制用户/项目自定义 hooks 使用范围。

### 5. 测试现状
- 仓库中未发现针对 `examples/hooks/bash_command_validator_example.py` 的专门单元测试或 CI 测试。
- 可复用 `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh` 做离线回归测试。

## 依赖与外部交互

### 1. 运行时依赖
- Python 解释器：`python3`
- 标准库：`json`、`re`、`sys`
- 无第三方依赖包。

### 2. 输入/输出交互
- 输入：stdin JSON（由 Claude Code hook runtime 提供）。
- 输出：
  - 阻断提示写入 stderr。
  - stdout 默认不写入。
- 决策：通过进程退出码控制放行/阻断。

### 3. 配置交互
- 该文件不会自动生效，必须在 hooks 配置中注册命令路径。
- 文件头示例使用绝对路径占位符 `/path/to/...`，实际部署需替换为真实路径（更推荐 `${CLAUDE_PLUGIN_ROOT}` 风格用于可移植性）。

### 4. 外部文档交互
- 文件头注释引用了 hooks 官方文档链接：`https://docs.anthropic.com/en/docs/claude-code/hooks`。
- 脚本本身不发起网络请求。

## 风险、边界与改进建议

### 风险

1. 规则覆盖面较窄，存在常见漏拦
- 例如 `sudo grep ...`、`command grep ...`、`find . -type f -name ...` 不命中当前规则。

2. 正则匹配易产生策略与直觉偏差
- `grep` 只因存在管道就放行，未必符合“统一改用 rg”的组织策略目标。

3. 输出协议非结构化
- 当前阻断仅输出文本，不含结构化字段（如 `permissionDecision`），跨客户端展示一致性与机器可消费性较弱。

4. 路径示例可移植性不足
- 文档示例是硬编码绝对路径，容易在复制后直接失效。

### 边界

1. 这是示例脚本，不是生产级完整策略引擎。
2. 仅处理 `Bash` 工具，不覆盖 `Read/Write/Edit/MultiEdit`。
3. 仅做字符串级规则判断，不解析 shell AST，也不自动改写命令。

### 改进建议

1. 增强规则鲁棒性
- 覆盖前缀场景（`sudo`/`env`/`command`），并支持更灵活的 `find` 参数顺序。

2. 引入结构化阻断输出
- 参考 `validate-bash.sh` 风格，返回统一 JSON 字段（如 `hookSpecificOutput.permissionDecision` + `systemMessage`），保留人类可读消息。

3. 增加样例级回归测试
- 在 `examples/hooks/` 补充最小测试脚本和固定输入样例，断言 exit code 与输出，避免规则调整回归。

4. 优化配置示例
- 在注释中增加 `${CLAUDE_PLUGIN_ROOT}` 写法示例，减少路径硬编码问题。

5. 明确策略定位
- 若定位“效率建议”，可改为提示不阻断；若定位“强制规范”，需扩大规则覆盖并标注例外清单。
