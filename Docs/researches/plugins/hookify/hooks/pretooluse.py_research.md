# FILE `plugins/hookify/hooks/pretooluse.py` 研究文档

## 场景与职责

`pretooluse.py` 是 Hookify 在 `PreToolUse` 事件上的执行器。它在工具真正执行前接收 Hook 输入，按工具类型筛选规则并调用规则引擎，最终输出允许/拒绝或提醒信息。

该脚本是 Hookify 的“前置防护入口”，决定危险 Bash 或文件写入行为是否被及时拦截。

## 功能点目的

1. 运行前事件接入。
- 对接 `hooks.json` 中的 `PreToolUse` command hook。

2. 工具类型折叠。
- 将 Claude 工具名折叠为 Hookify 内部事件：`Bash -> bash`，`Edit/Write/MultiEdit -> file`。

3. 动态规则加载。
- 按事件调用 `load_rules(event=...)`，读取 `.claude/hookify.*.local.md`。

4. 决策输出。
- 调用 `RuleEngine.evaluate_rules(...)` 输出协议 JSON（包含 `permissionDecision: deny` 或 `systemMessage`）。

5. 失败放行（fail-open）。
- 任意异常下仍输出 JSON 并 `exit 0`，避免 hook 崩溃阻塞主流程。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 导入路径注入

脚本先读取 `CLAUDE_PLUGIN_ROOT`，并向 `sys.path` 注入：

- `parent_dir = dirname(CLAUDE_PLUGIN_ROOT)`（用于 `import hookify.*`）
- `CLAUDE_PLUGIN_ROOT` 本身（兼容同目录脚本访问）

实现：`plugins/hookify/hooks/pretooluse.py:14-23`。

### 2) 核心执行流程

1. `json.load(sys.stdin)` 读取 hook 输入（`pretooluse.py:39`）。
2. 读取 `tool_name` 并做事件映射（`pretooluse.py:43-49`）。
3. `rules = load_rules(event=event)`（`pretooluse.py:52`）。
4. `RuleEngine().evaluate_rules(rules, input_data)`（`pretooluse.py:55-56`）。
5. `print(json.dumps(result))` 输出 JSON（`pretooluse.py:59`）。
6. `finally` 固定 `sys.exit(0)`（`pretooluse.py:68-70`）。

### 3) 输入输出协议

- 输入：Claude Hook runtime 注入 JSON，关键字段包括 `hook_event_name/tool_name/tool_input`。
- 输出：规则引擎构造的 dict（序列化 JSON）。
  - 阻断时（`hook_event_name=PreToolUse`）会输出：
    - `hookSpecificOutput.hookEventName = "PreToolUse"`
    - `hookSpecificOutput.permissionDecision = "deny"`
    - `systemMessage`
  - 无命中时输出 `{}`。

协议实现位于引擎：`plugins/hookify/core/rule_engine.py:65-94`。

### 4) 错误处理

- 导入失败：输出 `{"systemMessage":"Hookify import error: ..."}`，立即 `exit 0`（`pretooluse.py:25-33`）。
- 运行失败：输出 `{"systemMessage":"Hookify error: ..."}`，最终 `exit 0`（`pretooluse.py:61-70`）。

## 关键代码路径与文件引用

- 目标文件：`plugins/hookify/hooks/pretooluse.py:1-74`
- 路由配置：`plugins/hookify/hooks/hooks.json:4-13`
- 规则加载：`plugins/hookify/core/config_loader.py:198-241`
- 决策引擎：`plugins/hookify/core/rule_engine.py:35-94`
- 字段抽取（Bash/File）：`plugins/hookify/core/rule_engine.py:230-253`
- 用户规则来源：
  - `plugins/hookify/README.md:71-128`
  - `plugins/hookify/commands/hookify.md:84-156`

## 依赖与外部交互

1. Python 标准库：`os/sys/json`。
2. 内部模块：`hookify.core.config_loader`、`hookify.core.rule_engine`。
3. 环境变量：`CLAUDE_PLUGIN_ROOT`（路径注入关键依赖）。
4. 外部输入：stdin JSON（Claude Hook runtime）。
5. 文件系统间接依赖：`load_rules` 扫描 `.claude/hookify.*.local.md`。

## 风险、边界与改进建议

1. 事件映射依赖精确工具名。
- 风险：如果运行时工具名变化（大小写或新工具），`event=None` 只会命中 `event: all` 规则。
- 建议：统一做大小写归一化并支持可配置映射表。

2. 全局异常吞掉细节。
- 风险：脚本层捕获 `Exception` 后仅返回字符串，定位复杂问题较慢。
- 建议：追加结构化错误字段（如 `errorType`、`stage`）并可选 debug 日志。

3. fail-open 策略在强安全场景不够严格。
- 风险：导入失败或解析异常时仍放行敏感操作。
- 建议：引入可配置严格模式（仅对 PreToolUse 支持 fail-closed）。

4. 代码重复。
- 现状：与 `posttooluse.py` 大量重复（路径注入、读取、输出、异常处理）。
- 建议：抽取公共运行器函数，降低后续维护成本和行为漂移风险。
