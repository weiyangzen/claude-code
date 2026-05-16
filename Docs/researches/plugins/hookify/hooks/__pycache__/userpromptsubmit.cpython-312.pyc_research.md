# FILE `plugins/hookify/hooks/__pycache__/userpromptsubmit.cpython-312.pyc` 研究文档

## 场景与职责

`userpromptsubmit.cpython-312.pyc` 是 `plugins/hookify/hooks/userpromptsubmit.py` 的缓存字节码，面向 `UserPromptSubmit` 事件。

它支撑“用户提示词提交时”的规则检查入口，让 Hookify 能在工具执行前、会话输入侧执行 prompt 规则匹配。

## 功能点目的

1. 缓存 Prompt 执行器，降低用户每次发起请求时的 Python 启动成本。
2. 固化事件筛选：`load_rules(event='prompt')`。
3. 承载 prompt 字段规则判定链路（通过引擎读取 `user_prompt`）。
4. 保持统一容错行为：异常不阻断系统主流程。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 字节码头信息（实测）

- 文件：`plugins/hookify/hooks/__pycache__/userpromptsubmit.cpython-312.pyc`
- `magic`: `cb0d0d0a`
- `flags`: `0`
- `timestamp`: `1773905087`（UTC `2026-03-19T07:24:47Z`）
- `src_size`: `1543`（与 `userpromptsubmit.py` 一致）
- `co_filename`: `plugins/hookify/hooks/userpromptsubmit.py`
- `main` 首行：30

### 2) 主流程反汇编摘要

反汇编对应流程如下：

1. `json.load(sys.stdin)` 获取 `UserPromptSubmit` 输入。
2. 固定执行 `load_rules(event='prompt')`。
3. 调用 `RuleEngine.evaluate_rules(...)`。
4. 将结果 `json.dumps(...)` 输出到 stdout。
5. 异常输出 `{"systemMessage": "Hookify error: ..."}`。
6. `finally` 固定 `sys.exit(0)`。

### 3) Prompt 协议与字段

- prompt 文本由 `rule_engine._extract_field()` 在 `field == 'user_prompt'` 时从 `input_data.user_prompt` 读取。
- 对于 `UserPromptSubmit`，引擎未定义单独的强 block 协议分支；即使匹配 `action: block`，当前实现主要回传 `systemMessage`。

### 4) 研究命令

- `file plugins/hookify/hooks/__pycache__/userpromptsubmit.cpython-312.pyc`
- `python3` + `marshal/dis` 读取 `co_consts/co_names/main` 指令
- `nl -ba plugins/hookify/hooks/userpromptsubmit.py`

## 关键代码路径与文件引用

- `plugins/hookify/hooks/__pycache__/userpromptsubmit.cpython-312.pyc`（目标缓存）
- `plugins/hookify/hooks/userpromptsubmit.py:30-58`（缓存对应源码）
- `plugins/hookify/hooks/hooks.json:37-46`（UserPromptSubmit 路由）
- `plugins/hookify/core/config_loader.py:198-241`（prompt 规则加载）
- `plugins/hookify/core/rule_engine.py:80-84`（非 Stop/Pre/Post block 的回退输出）
- `plugins/hookify/core/rule_engine.py:226-228`（`user_prompt` 字段抽取）
- `plugins/hookify/commands/help.md:19-24`（事件说明）

## 依赖与外部交互

1. 运行时：CPython 3.12 字节码加载。
2. 调用方：Hook Runtime 的 `UserPromptSubmit` 生命周期事件。
3. 输入：stdin JSON（关键字段 `user_prompt`）。
4. 输出：stdout JSON（通常为 `systemMessage` 或 `{}`）。
5. 规则源：`.claude/hookify.*.local.md` 中 `event: prompt` 条目。
6. 内部依赖：`load_rules` 和 `RuleEngine`。

## 风险、边界与改进建议

1. `action: block` 在 Prompt 事件上的语义不完整，现状更接近“提醒”而非“禁止提交”。
2. 执行器异常依旧 `exit 0`，会导致 prompt 规则链路在故障时静默失效。
3. `.pyc` 缓存仅反映某次编译快照，审计与变更评审必须以 `.py` 源码为准。
4. 规则匹配高度依赖 `user_prompt` 文本；若运行时字段变更，会直接导致规则失效。
5. 建议增加 Prompt 专用阻断协议分支和显式兼容层（字段别名、版本检查、诊断日志）。
