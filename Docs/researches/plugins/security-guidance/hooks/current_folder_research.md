# plugins/security-guidance/hooks 研究

## 场景与职责

`plugins/security-guidance/hooks` 是 `security-guidance` 插件的执行入口目录，职责是把 Claude Code 的 `PreToolUse` 事件绑定到安全提醒脚本，并在写文件类工具执行前做轻量安全拦截。

- 上游调用方：Claude Code Hook 运行时（按插件 hooks 配置触发 `PreToolUse`）
- 本目录内对象：
  - `hooks.json`：事件注册与工具匹配
  - `security_reminder_hook.py`：规则匹配、状态去重、阻断/放行
- 下游交互：
  - `python3` 解释器
  - 本地状态文件 `~/.claude/security_warnings_state_<session_id>.json`
  - stderr（提示文案）+ 退出码（阻断/放行）

该目录定位是“写入前安全提醒层”，不是完整静态安全扫描器。

## 功能点目的

1. 只拦截写入相关工具
- `hooks.json` 通过 `matcher: "Edit|Write|MultiEdit"` 限定触发范围，避免影响 Bash/Read 等无关工具。

2. 在高风险模式出现时阻断当前工具调用
- 命中规则后输出提醒到 stderr，并 `sys.exit(2)`（PreToolUse 阻断语义）。

3. 同会话去重
- 对同一 `session_id` 的同一 `file_path-ruleName` 只提醒一次，降低重复打扰。

4. 支持临时关闭
- 通过 `ENABLE_SECURITY_REMINDER=0` 直接放行。

5. 低成本清理历史状态
- 以 10% 概率执行状态文件清理，删除 30 天前记录。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 关键流程

1. Claude Code 读取插件 hooks 配置（本插件未在 `plugin.json` 覆写 hooks 路径，按默认 `./hooks/hooks.json`）。
2. `PreToolUse` 事件发生且工具名匹配 `Edit|Write|MultiEdit` 时，执行命令：
   - `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/security_reminder_hook.py`
3. Hook 从 stdin 读取 JSON，提取 `session_id`、`tool_name`、`tool_input`。
4. 仅对 `Edit/Write/MultiEdit` 提取待检查内容：
   - `Write` -> `tool_input.content`
   - `Edit` -> `tool_input.new_string`
   - `MultiEdit` -> `" ".join(edits[*].new_string)`
5. 规则命中后生成去重键 `"<file_path>-<ruleName>"`：
   - 首次命中：写入状态 -> stderr 输出提醒 -> `exit 2`
   - 已提醒过：`exit 0`
6. 未命中或不适用场景：`exit 0`。

### 2) 数据结构

核心规则表 `SECURITY_PATTERNS`（list[dict]），字段包括：

- `ruleName`：规则标识
- `path_check`：路径型匹配（当前用于 `.github/workflows/*.yml|*.yaml`）
- `substrings`：内容子串匹配（如 `eval(`、`dangerouslySetInnerHTML`、`os.system`）
- `reminder`：命中文案

实现特征：顺序扫描、首条命中即返回；因此规则顺序会影响输出结果。

### 3) 协议与返回码

- 输入协议：command hook 从 stdin 接收 JSON，关键字段含 `session_id`、`hook_event_name`、`tool_name`、`tool_input`。
- 退出码语义：
  - `0`：放行/成功
  - `2`：阻断（PreToolUse 常用阻断码）
  - 其他：异常（通常不作为可预期阻断）

### 4) 实测命令（本次研究）

在仓库根目录直接执行：

```bash
python3 plugins/security-guidance/hooks/security_reminder_hook.py < test-input.json
```

本次构造 `Write + eval(` 输入并验证结果：

- 第一次命中：`exit=2`，stderr 输出 eval 风险提醒
- 同会话同输入第二次：`exit=0`（去重生效）
- `ENABLE_SECURITY_REMINDER=0`：`exit=0`
- 状态文件会落到 `~/.claude/security_warnings_state_<session_id>.json`

附：`plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh` 对该 `hooks.json` 直接校验会失败（exit 5），原因是该脚本按“顶层事件键”遍历，而插件实际使用 wrapper 格式（`{"description":...,"hooks":{...}}`）。

## 关键代码路径与文件引用

- `plugins/security-guidance/hooks/hooks.json:1`
- `plugins/security-guidance/hooks/hooks.json:4`
- `plugins/security-guidance/hooks/hooks.json:9`
- `plugins/security-guidance/hooks/hooks.json:12`
- `plugins/security-guidance/hooks/security_reminder_hook.py:31`
- `plugins/security-guidance/hooks/security_reminder_hook.py:129`
- `plugins/security-guidance/hooks/security_reminder_hook.py:183`
- `plugins/security-guidance/hooks/security_reminder_hook.py:202`
- `plugins/security-guidance/hooks/security_reminder_hook.py:217`
- `plugins/security-guidance/hooks/security_reminder_hook.py:220`
- `plugins/security-guidance/hooks/security_reminder_hook.py:273`
- `plugins/security-guidance/.claude-plugin/plugin.json:1`
- `plugins/README.md:27`
- `plugins/plugin-dev/skills/plugin-structure/references/manifest-reference.md:262`
- `plugins/plugin-dev/skills/hook-development/SKILL.md:62`
- `plugins/plugin-dev/skills/hook-development/SKILL.md:300`
- `plugins/plugin-dev/skills/hook-development/SKILL.md:324`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:30`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:205`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:43`

## 依赖与外部交互

### 运行依赖

- `python3`
- Python 标准库：`json`、`os`、`random`、`sys`、`datetime`

### 配置与环境变量

- `CLAUDE_PLUGIN_ROOT`：`hooks.json` 中用于拼接脚本路径
- `ENABLE_SECURITY_REMINDER`：插件开关（`0` 关闭）

### 文件系统与进程交互

- stdin：接收 hook input JSON
- `~/.claude/security_warnings_state_<session_id>.json`：会话去重状态
- `/tmp/security-warnings-log.txt`：异常调试日志（仅 debug_log 分支）
- 无网络调用

### 测试/脚本/文档上下文

- 本目录无专属单元测试、无 README。
- 可复用 `plugin-dev` 工具链：
  - `scripts/test-hook.sh`（生成样例输入、执行并解释退出码）
  - `scripts/hook-linter.sh`（通用脚本质量检查）
  - `scripts/validate-hook-schema.sh`（当前与 wrapper 格式存在兼容问题）
- 规范文档由 `hook-development/SKILL.md` 与 `manifest-reference.md` 提供。

## 风险、边界与改进建议

1. 子串匹配易误报与漏报
- 风险：注释/字符串常量也会触发，变形写法可能绕过。
- 建议：引入最小语法感知（正则边界或轻量 AST）并按语言拆分规则。

2. 首条命中即返回
- 风险：一次编辑中的多类风险只提示一条，修复迭代变慢。
- 建议：收集多命中后合并输出（可限制上限）。

3. 去重键粒度较粗（`file_path-ruleName`）
- 风险：同文件后续新增同类风险也不再提醒。
- 建议：键中加入内容摘要或时间窗口。

4. 异常分支偏 fail-open
- 风险：JSON 解析失败/状态异常时直接放行，保护静默失效。
- 建议：增加可观测告警，并支持可选 fail-closed 模式。

5. 建议文案存在仓库耦合
- 风险：`execFileNoThrow` 路径示例并不存在于当前仓库，易误导。
- 建议：替换为仓库无关建议，或按项目能力动态提示。

6. 调试日志策略较弱
- 风险：固定写 `/tmp/security-warnings-log.txt`，无轮转与开关。
- 建议：添加显式 debug 开关和日志大小/保留策略。

7. 工具链一致性问题
- 风险：官方示例文档推荐的 schema 校验脚本无法直接校验该目录 `hooks.json`（wrapper 格式）。
- 建议：扩展 `validate-hook-schema.sh` 支持插件 wrapper，或提供 wrapper 预处理步骤。
