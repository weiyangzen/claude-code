# FILE `plugins/security-guidance/hooks/security_reminder_hook.py` 研究文档

## 场景与职责

`security_reminder_hook.py` 是 `security-guidance` 插件的核心执行脚本。它在 `PreToolUse` 事件中接收 Claude Code 的工具调用上下文，识别高风险编辑模式，并在必要时阻断本次写操作。

职责边界：

- 做“写入前提醒与阻断”，不做完整静态分析
- 覆盖少量高价值安全模式（当前 9 类）
- 在同一会话内对同一风险去重，降低重复打扰

该脚本是策略执行层；其上游调度来自 `hooks/hooks.json`，下游交互主要是本地文件和 stderr/退出码。

## 功能点目的

1. 风险模式识别
- 对文件路径与编辑内容做轻量匹配，发现可能的命令注入、XSS、动态代码执行、反序列化风险等。

2. 在工具执行前阻断
- 命中风险时通过 `stderr` 输出提醒，并以 `sys.exit(2)` 阻断当前 `PreToolUse` 对应工具调用。

3. 会话内去重提醒
- 对同一 `session_id`、同一 `file_path`、同一规则只提示一次，避免反复中断同一修复流程。

4. 兼顾可用性的 fail-open
- 解析失败、状态文件异常等场景默认放行（`exit 0`），避免 hook 自身故障阻塞开发。

5. 低成本状态维护
- 采用本地 JSON 状态文件保存去重集合，并以概率触发过期清理。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 规则结构：`SECURITY_PATTERNS`

脚本使用 `list[dict]` 描述规则，字段包括：

- `ruleName`：规则标识
- `path_check`：路径型匹配函数（仅 GitHub Actions workflow 规则）
- `substrings`：内容子串匹配列表
- `reminder`：命中后输出的提醒文本

当前规则覆盖 9 类场景：

- `.github/workflows/*.yml|*.yaml` 路径风险提示
- `child_process.exec / exec( / execSync(`
- `new Function`
- `eval(`
- `dangerouslySetInnerHTML`
- `document.write`
- `.innerHTML =`
- `pickle`
- `os.system / from os import system`

匹配策略为顺序扫描、首条命中即返回；因此规则顺序决定最终提示。

### 2) 输入提取与协议适配

入口 `main()` 从 stdin 读取 JSON，并提取：

- `session_id`
- `tool_name`
- `tool_input`

仅当 `tool_name` 属于 `Edit/Write/MultiEdit` 时继续处理。内容提取逻辑由 `extract_content_from_input()` 按工具分支：

- `Write` -> `tool_input["content"]`
- `Edit` -> `tool_input["new_string"]`
- `MultiEdit` -> 拼接 `tool_input["edits"][*]["new_string"]`

### 3) 命中判定流程

`check_patterns(file_path, content)` 先做路径匹配，再做子串匹配：

1. `file_path` 去前导 `/` 后用于 `path_check`
2. 若路径规则命中，直接返回 `(ruleName, reminder)`
3. 否则遍历 `substrings` 在 `content` 中做包含匹配
4. 命中即返回；无命中返回 `(None, None)`

### 4) 去重与状态文件

- 状态文件路径：`~/.claude/security_warnings_state_<session_id>.json`
- 状态内容：已提示的 `warning_key` 列表（内存中转为 `set`）
- 去重键：`<file_path>-<ruleName>`

流程：

1. 命中后读取当前会话状态（`load_state`）
2. 未出现过则写入状态（`save_state`）
3. 首次命中打印提醒并 `exit 2`
4. 已提醒过则直接放行

### 5) 生命周期维护

- `ENABLE_SECURITY_REMINDER=0`：直接短路 `exit 0`
- `cleanup_old_state_files()`：10% 概率触发，删除 `~/.claude` 下 30 天前状态文件
- `debug_log()`：异常时尝试写 `/tmp/security-warnings-log.txt`，但任何日志错误都会吞掉

### 6) 退出码语义

- `0`：放行
- `2`：阻断（PreToolUse 约定阻断码）

脚本在 JSON 解析错误、无文件路径、非目标工具、状态 I/O 异常等情形均倾向返回 `0`，体现 fail-open 策略。

## 关键代码路径与文件引用

- `plugins/security-guidance/hooks/security_reminder_hook.py:31-126`（规则定义 `SECURITY_PATTERNS`）
- `plugins/security-guidance/hooks/security_reminder_hook.py:129-131`（状态文件路径函数）
- `plugins/security-guidance/hooks/security_reminder_hook.py:134-157`（历史状态清理）
- `plugins/security-guidance/hooks/security_reminder_hook.py:159-180`（状态读写）
- `plugins/security-guidance/hooks/security_reminder_hook.py:183-199`（规则匹配入口）
- `plugins/security-guidance/hooks/security_reminder_hook.py:202-214`（工具输入内容提取）
- `plugins/security-guidance/hooks/security_reminder_hook.py:217-276`（主流程与退出码）
- `plugins/security-guidance/hooks/hooks.json:4-13`（上游事件绑定与 matcher）
- `plugins/plugin-dev/skills/hook-development/SKILL.md:296-317`（hook stdin 协议与退出码说明）
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:30-43`（PreToolUse 样例输入结构）
- `examples/hooks/bash_command_validator_example.py:56-79`（同类 command hook 阻断示例）

## 依赖与外部交互

### 运行依赖

- 解释器：`python3`
- 标准库：`json`、`os`、`random`、`sys`、`datetime`

### 输入输出与文件系统

- 输入：Claude Code hook runtime 通过 stdin 传入 JSON
- 输出：
  - 命中时向 `stderr` 输出提醒文本
  - 通过退出码返回阻断/放行决策
- 本地文件：
  - 读写 `~/.claude/security_warnings_state_<session_id>.json`
  - 异常日志 `/tmp/security-warnings-log.txt`

### 环境变量

- `ENABLE_SECURITY_REMINDER`：安全提醒全局开关
- `CLAUDE_PLUGIN_ROOT`：由上游 `hooks.json` 在 command 中使用（本脚本本身不读取）

### 外部系统交互

- 无网络请求、无子进程调用、无第三方包依赖

## 风险、边界与改进建议

1. 匹配机制偏粗糙，误报和漏报并存
- 风险：纯子串匹配会误伤注释/文档文本；变形写法可能绕过。
- 建议：按语言增加边界匹配或 AST 级检测，至少先对高误报规则做精细化。

2. 首条命中即返回，信息覆盖不足
- 风险：一次编辑中存在多类风险时，用户一次只能看到一条提醒。
- 建议：支持多命中聚合输出，并限制最大输出条数。

3. 去重键粒度仅 `file_path-ruleName`
- 风险：同文件新增同类风险不会再次提示，可能静默放行。
- 建议：在去重键加入内容摘要（如 hash）或时间窗口。

4. fail-open 在安全场景可能过宽
- 风险：JSON 异常、状态损坏时直接放行，保护能力下降且不可见。
- 建议：增加可观测告警；引入可选“严格模式”（异常时 fail-closed）。

5. 调试日志策略缺乏治理
- 风险：固定写 `/tmp/security-warnings-log.txt`，无开关和轮转，长期可能堆积。
- 建议：使用显式 `DEBUG` 环境变量控制，并设置大小上限/轮转。

6. 文案存在仓库耦合示例
- 风险：`execFileNoThrow` 提示路径并非当前仓库必然存在，可能误导。
- 建议：改为与仓库无关的安全替代建议，或动态探测项目内可用替代。

7. 缺少自动化测试
- 风险：规则调整易引入回归（误拦截或漏拦截）。
- 建议：补充最小回归集（规则命中、未命中、去重、开关、异常输入）。

