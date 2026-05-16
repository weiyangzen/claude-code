# DIR `plugins/hookify/utils` 研究文档

## 场景与职责

`plugins/hookify/utils` 在当前仓库中是 Hookify 插件的“预留工具层目录”，目录内仅有一个空文件 `__init__.py`（0 bytes），没有任何可执行实现。

从运行链角度看，它当前不承担实际逻辑；Hookify 的真实执行路径是：

1. `hooks/hooks.json` 注册 4 个事件脚本（PreToolUse/PostToolUse/Stop/UserPromptSubmit）。
2. `hooks/*.py` 读取 stdin JSON，加载规则并调用引擎。
3. `core/config_loader.py` 负责规则发现与解析。
4. `core/rule_engine.py` 负责条件匹配、动作决策与协议输出。

因此，`utils` 当前职责是“包结构占位 + 未来公共逻辑承载点”，而不是运行期主干模块。

## 功能点目的

虽然目录本身为空，但结合上下文可以明确它的设计目的：

1. 维持插件分层结构完整性。
- `hookify` 目录按 `commands/agents/skills/hooks/core/matchers/utils` 分层组织，`utils` 与 `matchers` 一样属于可扩展边界。

2. 为跨脚本复用预留落点。
- 目前 `hooks/pretooluse.py`、`posttooluse.py`、`stop.py`、`userpromptsubmit.py` 有大量重复逻辑（路径注入、stdin 解析、异常兜底、JSON 输出）。
- `utils` 可承接这些重复逻辑，减少维护分叉。

3. 为后续演进提供低风险迁移路径。
- 命令层、技能层、README 已稳定面向 `.claude/hookify.*.local.md` 规则格式；把公共代码沉到 `utils` 可在不改用户配置协议的前提下迭代内部实现。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 当前关键流程（`utils` 的上下游）

1. Hook 注册层：`plugins/hookify/hooks/hooks.json`。
- 4 个事件统一通过 `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/*.py` 执行，`timeout` 均为 10 秒。

2. Hook 执行层：`plugins/hookify/hooks/*.py`。
- 从 `CLAUDE_PLUGIN_ROOT` 推导并注入 `sys.path`。
- 读取 stdin 输入 JSON。
- 根据 `tool_name` 进行事件归类（如 `Bash -> bash`，`Edit/Write/MultiEdit -> file`）。
- 调用 `load_rules(event=...)` 与 `RuleEngine.evaluate_rules(...)`。
- 统一输出 JSON，并在异常时兜底 `systemMessage`，最终 `exit 0`。

3. 规则加载层：`plugins/hookify/core/config_loader.py`。
- 扫描 `.claude/hookify.*.local.md`。
- 解析 frontmatter 生成 `Rule/Condition`。
- 过滤 `enabled: true` 与 `event` 匹配项。

4. 规则执行层：`plugins/hookify/core/rule_engine.py`。
- 条件运算符：`regex_match`、`contains`、`equals`、`not_contains`、`starts_with`、`ends_with`。
- 支持 `tool_matcher` 工具匹配和字段提取（`command`、`file_path`、`new_text`、`transcript`、`user_prompt` 等）。
- `block` 优先于 `warn`。

结论：`utils` 当前不在执行路径上；其“技术实现”体现在尚未被落地的抽象空间。

### B. 关键数据结构（`utils` 未来潜在服务对象）

1. `Condition`（`core/config_loader.py`）
- 字段：`field/operator/pattern`。

2. `Rule`（`core/config_loader.py`）
- 字段：`name/enabled/event/pattern/conditions/action/tool_matcher/message`。
- 兼容 legacy `pattern` 与新式 `conditions`。

3. Hook 输入/输出协议（`hooks/*.py` + `rule_engine.py`）
- 输入：stdin JSON（含 `hook_event_name`、`tool_name`、`tool_input`、`user_prompt`、`reason`、`transcript_path` 等）。
- 输出：
  - Tool 事件阻断：`hookSpecificOutput.permissionDecision = "deny"`
  - Stop 阻断：`decision = "block"`
  - 提示：`systemMessage`

### C. 相关命令与脚本（验证链路）

1. 用户规则生产与管理命令：
- `/hookify`（创建规则文件）
- `/hookify:list`（枚举规则）
- `/hookify:configure`（启停规则）
- `/hookify:help`（说明机制）

2. 规则文件规范：
- `plugins/hookify/skills/writing-rules/SKILL.md` 定义规则字段、事件、操作符与命名约定。

3. 测试/校验脚本（来自 plugin-dev 工具链，hookify 自身未内建 tests）：
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/hook-linter.sh`

## 关键代码路径与文件引用

### 目标目录

- `plugins/hookify/utils/__init__.py`（空文件，0 bytes）

### 直接上下游（调用方/被调用方）

- `plugins/hookify/hooks/hooks.json:3`
- `plugins/hookify/hooks/pretooluse.py:12`
- `plugins/hookify/hooks/pretooluse.py:35`
- `plugins/hookify/hooks/posttooluse.py:12`
- `plugins/hookify/hooks/stop.py:12`
- `plugins/hookify/hooks/userpromptsubmit.py:12`
- `plugins/hookify/core/config_loader.py:15`
- `plugins/hookify/core/config_loader.py:198`
- `plugins/hookify/core/rule_engine.py:27`
- `plugins/hookify/core/rule_engine.py:35`

### 配置、文档、样例、脚本

- `plugins/hookify/.claude-plugin/plugin.json:2`
- `plugins/hookify/README.md:7`
- `plugins/hookify/commands/hookify.md:84`
- `plugins/hookify/commands/list.md:16`
- `plugins/hookify/commands/configure.md:18`
- `plugins/hookify/commands/help.md:24`
- `plugins/hookify/skills/writing-rules/SKILL.md:11`
- `plugins/hookify/examples/dangerous-rm.local.md:1`
- `plugins/hookify/examples/require-tests-stop.local.md:1`
- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh:41`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh:30`

## 依赖与外部交互

1. 运行时依赖
- Python 3（hook 命令均用 `python3` 触发）。
- Python 标准库（`os/sys/json/glob/re/dataclasses/functools`）。
- 环境变量：`CLAUDE_PLUGIN_ROOT`（导入路径注入）。

2. 文件系统交互
- 读取项目目录规则：`.claude/hookify.*.local.md`。
- Stop 场景可读取 `transcript_path` 对应文件。
- 输出通过 stdout JSON 返回给 Hook runtime。

3. 外部系统交互
- 无网络调用、无数据库调用、无第三方 SDK 依赖。
- 与 Claude Hook runtime 的交互为本地进程协议（stdin/stdout + exit code）。

4. 测试现状
- `plugins/hookify` 下未发现 `tests/` 或 `test/` 自动化测试目录。
- `core/*.py` 仅保留 `if __name__ == '__main__'` 的手工演示片段。

## 风险、边界与改进建议

1. 空目录导致公共逻辑分散
- 现状：4 个 hook 脚本重复实现路径注入、输入读取、错误兜底。
- 风险：未来改动时易出现行为不一致（例如某个脚本漏同步异常处理）。
- 建议：在 `plugins/hookify/utils/` 新增公共模块（如 `runtime.py`），沉淀以下函数：
  - `bootstrap_python_path()`
  - `detect_event(input_data)`
  - `safe_read_stdin_json()`
  - `safe_emit_json(result)`

2. 事件映射规则分布在多个脚本
- 现状：PreToolUse/PostToolUse 各自维护 `tool_name -> event` 逻辑。
- 风险：工具类型扩展时可能出现某脚本支持、某脚本不支持。
- 建议：把映射表统一到 `utils` 常量并复用。

3. 校验脚本与 hookify 配置格式存在潜在偏差
- `validate-hook-schema.sh` 默认要求每个 hook 条目有 `matcher`，而 `hookify/hooks/hooks.json` 使用的是插件 wrapper 结构且未显式设置 matcher。
- 建议：
  - 要么给 hookify 的每个条目补显式 `matcher`；
  - 要么增强校验脚本对该格式的兼容。

4. 缺少自动化回归验证
- 现状：无 hookify 专属单测，公共行为主要靠手工验证。
- 建议：
  - 使用 `test-hook.sh --create-sample` 建立最小 smoke test；
  - 若后续把公共逻辑迁移到 `utils`，同步补充 `pytest` 级单测，覆盖事件映射、异常兜底、协议输出。

5. 边界说明不足
- 现状：`utils/__init__.py` 无任何注释，难以看出“占位而非遗漏”。
- 建议：至少补一段模块注释，声明当前状态和未来用途，降低维护歧义。
