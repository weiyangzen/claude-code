# FILE `plugins/hookify/hooks/__pycache__/stop.cpython-312.pyc` 研究文档

## 场景与职责

`stop.cpython-312.pyc` 是 `plugins/hookify/hooks/stop.py` 的字节码缓存，服务于 `Stop` 生命周期事件。

当代理尝试结束任务时，Hook Runtime 会调用 `stop.py`；该 `.pyc` 缓存用于快速进入“停止前规则校验”流程，支持会话收尾质量门禁。

## 功能点目的

1. 缓存 `Stop` 执行器，减少每次停止事件触发时的编译开销。
2. 固化固定事件加载逻辑：`load_rules(event='stop')`。
3. 通过统一引擎输出 Stop 协议（例如 `decision: block` 与 `reason`）。
4. 延续统一容错策略：异常时输出错误消息并 `exit 0`。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 字节码头与对象元数据（实测）

- 文件：`plugins/hookify/hooks/__pycache__/stop.cpython-312.pyc`
- `magic`: `cb0d0d0a`
- `flags`: `0`
- `timestamp`: `1773905087`（UTC `2026-03-19T07:24:47Z`）
- `src_size`: `1557`（与 `stop.py` 一致）
- `co_filename`: `plugins/hookify/hooks/stop.py`
- `main` 首行：30

### 2) 关键反汇编流程（main）

该缓存中的 `main` 逻辑呈现为：

1. `json.load(sys.stdin)` 读取停止事件输入。
2. 直接调用 `load_rules(event='stop')`（不依赖 `tool_name`）。
3. `RuleEngine().evaluate_rules(rules, input_data)`。
4. 输出 JSON 到 stdout。
5. `except` 输出 `Hookify error`。
6. `finally` 始终 `sys.exit(0)`。

### 3) Stop 协议链路

当规则命中 `action: block` 且 `hook_event_name` 为 `Stop` 时，引擎返回：

- `decision: "block"`
- `reason: <组合消息>`
- `systemMessage: <组合消息>`

此外，Stop 规则可经 `field: transcript` 触发全文读取 `input_data.transcript_path`，这是停止场景的关键外部 I/O。

### 4) 研究命令

- `file plugins/hookify/hooks/__pycache__/stop.cpython-312.pyc`
- `python3` + `struct/marshal/dis` 解析头部与 `main` 指令
- `nl -ba plugins/hookify/hooks/stop.py`

## 关键代码路径与文件引用

- `plugins/hookify/hooks/__pycache__/stop.cpython-312.pyc`（目标缓存）
- `plugins/hookify/hooks/stop.py:30-59`（缓存对应源码）
- `plugins/hookify/hooks/hooks.json:26-35`（Stop 事件注册）
- `plugins/hookify/core/config_loader.py:198-241`（`event='stop'` 规则筛选）
- `plugins/hookify/core/rule_engine.py:66-71`（Stop block 协议输出）
- `plugins/hookify/core/rule_engine.py:205-225`（`reason/transcript` 字段提取与文件读取）
- `plugins/hookify/examples/require-tests-stop.local.md:1-22`（Stop 规则样例）

## 依赖与外部交互

1. 调用方：Hook Runtime 的 `Stop` 事件调度。
2. 输入：stdin JSON（含 `reason`、`transcript_path` 等）。
3. 下游：`load_rules()` 与 `RuleEngine`。
4. 文件系统：读取 `.claude/hookify.*.local.md`，以及可能读取 transcript 文件。
5. 输出：stdout JSON（含 `decision/reason/systemMessage` 或 `{}`）。
6. 运行环境：`CLAUDE_PLUGIN_ROOT` 和 CPython 3.12 缓存机制。

## 风险、边界与改进建议

1. Stop 场景对 transcript 有 I/O 依赖，长会话或大文件会放大评估延迟。
2. 示例规则常写 `not_contains: npm test|pytest|cargo test`，但 `not_contains` 为字面包含，非正则 OR，易产生误判。
3. 统一 `exit 0` 让停止门禁在异常场景退化为放行，影响“必须通过再停止”的约束能力。
4. 当前 `.pyc` 为时戳缓存，不适合跨版本跨环境复用；应以源码为审计主源。
5. 建议在 Stop 路径增加 transcript 读取缓存、结构化错误输出、可选严格模式（stop fail-closed）。
