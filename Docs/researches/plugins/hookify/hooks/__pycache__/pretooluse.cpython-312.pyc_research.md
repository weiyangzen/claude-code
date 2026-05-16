# FILE `plugins/hookify/hooks/__pycache__/pretooluse.cpython-312.pyc` 研究文档

## 场景与职责

`pretooluse.cpython-312.pyc` 是 `plugins/hookify/hooks/pretooluse.py` 的 CPython 3.12 缓存产物，对应 `PreToolUse` 事件执行器。

它处于“工具执行前”防护链路，承担规则前置判定入口的字节码承载职责：在进程启动后快速进入规则评估逻辑，避免导入编译损耗占用拦截时机。

## 功能点目的

1. 为 PreToolUse 执行器提供可复用字节码，提升事件触发时响应速度。
2. 在缓存中固化 `tool_name -> event` 映射逻辑（`Bash/bash`、`Edit|Write|MultiEdit/file`）。
3. 承载 fail-open 执行模型（异常仍输出 JSON 且 `exit 0`）。
4. 保障与 `hooks.json` 注册命令一致的运行路径可快速命中缓存。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 字节码头信息（实测）

- 文件：`plugins/hookify/hooks/__pycache__/pretooluse.cpython-312.pyc`
- `magic`: `cb0d0d0a`
- `flags`: `0`（timestamp-based pyc）
- `timestamp`: `1773905087`（UTC `2026-03-19T07:24:47Z`）
- `src_size`: `2205`（与 `pretooluse.py` 一致）
- `co_filename`: `plugins/hookify/hooks/pretooluse.py`
- `main` 首行：35

### 2) 关键字节码流程（main）

反汇编结果可还原出以下关键路径：

1. `json.load(sys.stdin)` 读取 Hook 输入。
2. 读取 `tool_name` 并映射 `event`：
   - `Bash` -> `bash`
   - `Edit|Write|MultiEdit` -> `file`
3. `load_rules(event=event)` 从 `.claude` 加载规则。
4. 创建 `RuleEngine` 并调用 `evaluate_rules(...)`。
5. 输出 `json.dumps(result)` 到 `stdout`。
6. `except Exception` 输出 `Hookify error`。
7. `finally` 固定 `sys.exit(0)`。

### 3) 协议行为

本文件不直接定义协议字段，但缓存的执行器会驱动引擎产出如下结果：

- `PreToolUse` 阻断：`hookSpecificOutput.permissionDecision = "deny"`
- 警告：`systemMessage`
- 无匹配：`{}`

对应协议装配位于 `rule_engine.py`。

### 4) 研究命令

- `file plugins/hookify/hooks/__pycache__/pretooluse.cpython-312.pyc`
- `python3` + `marshal/dis` 提取 `co_filename`、`main` 指令序列
- `nl -ba plugins/hookify/hooks/pretooluse.py`

## 关键代码路径与文件引用

- `plugins/hookify/hooks/__pycache__/pretooluse.cpython-312.pyc`（目标缓存）
- `plugins/hookify/hooks/pretooluse.py:35-74`（执行器源码）
- `plugins/hookify/hooks/hooks.json:4-13`（PreToolUse 命令注册）
- `plugins/hookify/core/config_loader.py:209-226`（规则扫描和事件过滤）
- `plugins/hookify/core/rule_engine.py:49-79`（PreToolUse block 输出格式）
- `plugins/hookify/core/rule_engine.py:231-253`（Bash/File 字段提取）
- `plugins/hookify/README.md:122-128`（事件类型说明）

## 依赖与外部交互

1. 运行时依赖：CPython 3.12 导入缓存机制。
2. 环境变量：`CLAUDE_PLUGIN_ROOT`（源码里用于注入 `sys.path`，缓存执行结果依赖该路径）。
3. 外部输入：Claude Hook runtime 的 stdin JSON。
4. 外部输出：stdout JSON，供 Hook runtime 解释。
5. 文件系统依赖：`.claude/hookify.*.local.md` 规则文件。
6. 下游依赖：`load_rules()` 与 `RuleEngine.evaluate_rules()`。

## 风险、边界与改进建议

1. 映射边界：`tool_name` 非已知值时 `event=None`，只会命中 `event: all`；可能造成规则漏检。
2. 安全边界：fail-open 设计在执行器故障时默认放行，不适合强约束场景。
3. 可观测性：当前异常路径仅返回 `systemMessage` 文本，不利于自动化诊断。
4. 缓存一致性：`.pyc` 基于时间戳与大小校验，若跨环境复制缓存可能出现不可预测重编译。
5. 建议增加严格模式（可选 fail-closed）并输出结构化错误字段（`event/stage/error_type`）。
