# FILE `plugins/hookify/hooks/__pycache__/posttooluse.cpython-312.pyc` 研究文档

## 场景与职责

`posttooluse.cpython-312.pyc` 是 `plugins/hookify/hooks/posttooluse.py` 在 CPython 3.12 下的字节码缓存。

它不直接定义业务规则，而是在 `PostToolUse` 事件触发时为 Python 导入与执行提供缓存代码对象，减少脚本冷启动开销，保障 `hooks.json` 中该事件 `timeout: 10` 的时间预算。

## 功能点目的

1. 缓存 `PostToolUse` 执行器的编译结果，降低重复解析 `.py` 的成本。
2. 保留模块级代码对象与 `main()` 的可执行快照，支持快速加载。
3. 通过 `timestamp-based pyc` 机制与源码时间戳/大小联动，控制缓存失效。
4. 在 Hookify 规则链路中承载“事后规则评估”入口逻辑（事件映射 -> 规则加载 -> 规则判定 -> JSON 输出）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 字节码头与源码映射（实测）

- 文件：`plugins/hookify/hooks/__pycache__/posttooluse.cpython-312.pyc`
- `magic`: `cb0d0d0a`（CPython 3.12）
- `flags`: `0`（timestamp-based）
- `timestamp`: `1773905087`（UTC `2026-03-19T07:24:47Z`）
- `src_size`: `1776`（与 `plugins/hookify/hooks/posttooluse.py` 文件大小一致）
- `co_filename`: `plugins/hookify/hooks/posttooluse.py`
- `main` 代码对象首行：30

### 2) 关键反汇编流程（main）

反汇编显示该缓存执行流程与源码一致：

1. `json.load(sys.stdin)` 读取 Hook 输入。
2. 从 `input_data.get("tool_name", "")` 读取工具名。
3. `Bash -> bash`，`Edit/Write/MultiEdit -> file`。
4. `load_rules(event=event)` 筛选规则。
5. `RuleEngine().evaluate_rules(rules, input_data)` 生成决策。
6. `print(json.dumps(result), file=sys.stdout)` 输出协议 JSON。
7. 异常分支输出 `{"systemMessage": "Hookify error: ..."}`。
8. `finally` 固定 `sys.exit(0)`（fail-open）。

### 3) 协议落点

该 `.pyc` 缓存的是执行器实现，实际协议由 `rule_engine` 决定：

- 命中 `block` 且 `hook_event_name` 为 `PostToolUse`：输出
  - `hookSpecificOutput.hookEventName = "PostToolUse"`
  - `hookSpecificOutput.permissionDecision = "deny"`
  - `systemMessage`
- 仅 `warn`：输出 `systemMessage`
- 无命中：输出 `{}`

### 4) 研究命令

- `file plugins/hookify/hooks/__pycache__/posttooluse.cpython-312.pyc`
- `python3` + `struct/marshal/dis` 解析头部与 `main` 字节码
- `nl -ba plugins/hookify/hooks/posttooluse.py`

## 关键代码路径与文件引用

- `plugins/hookify/hooks/__pycache__/posttooluse.cpython-312.pyc`（目标缓存文件）
- `plugins/hookify/hooks/posttooluse.py:30-66`（缓存对应源码入口）
- `plugins/hookify/hooks/hooks.json:15-24`（PostToolUse 命令路由与 10 秒超时）
- `plugins/hookify/core/config_loader.py:198-241`（规则发现与 event 过滤）
- `plugins/hookify/core/rule_engine.py:35-94`（block/warn 合并与协议输出）
- `plugins/hookify/core/rule_engine.py:182-254`（字段抽取逻辑）
- `plugins/hookify/commands/help.md:18-25`（事件行为文档说明）

## 依赖与外部交互

1. 解释器依赖：CPython 3.12 字节码格式（文件名后缀 `cpython-312`）。
2. 调用方：Claude Code Hook Runtime，通过 `hooks.json` 调起 `python3 .../posttooluse.py`。
3. 输入：stdin JSON（含 `hook_event_name/tool_name/tool_input` 等）。
4. 输出：stdout JSON（`systemMessage` 或 `hookSpecificOutput`）。
5. 下游模块：`hookify.core.config_loader`、`hookify.core.rule_engine`。
6. 规则文件：运行目录下 `.claude/hookify.*.local.md`。

## 风险、边界与改进建议

1. `PostToolUse` 阶段的 `permissionDecision: deny` 是事后语义，无法回滚已执行副作用，容易被误解为“强拦截”。
2. `.pyc` 为运行时缓存，纳入版本库会产生跨环境噪声；仓库内 `plugins/hookify/.gitignore` 已声明应忽略 `__pycache__/`。
3. 当前执行器始终 `exit 0`，导入失败/运行异常都会放行，安全规则在故障场景可能失效。
4. 事件映射仅覆盖 `Bash/Edit/Write/MultiEdit`，其它工具会走 `event=None`，仅命中 `event: all` 规则。
5. 建议将 pre/post 公共逻辑抽成统一 runner，减少两份字节码行为漂移风险。
