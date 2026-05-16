# FILE `plugins/hookify/utils/__init__.py` 研究文档

## 场景与职责

`plugins/hookify/utils/__init__.py` 当前是一个 **0 字节空文件**，位于 Hookify 插件的 `utils` 包根。它本身不承载业务逻辑，也不参与规则匹配、规则加载或 Hook 输出协议的运行时计算。

在现有插件结构中，Hookify 的主链路是：

1. Hook 注册：`plugins/hookify/hooks/hooks.json`
2. Hook 执行：`plugins/hookify/hooks/pretooluse.py`、`posttooluse.py`、`stop.py`、`userpromptsubmit.py`
3. 规则加载：`plugins/hookify/core/config_loader.py`
4. 规则求值：`plugins/hookify/core/rule_engine.py`

因此，`utils/__init__.py` 的现实职责是“包占位与扩展边界预留”：

- 作为 `hookify.utils` 包初始化文件，保留后续放置公共函数的空间。
- 与 `core/__init__.py`、`hooks/__init__.py`、`matchers/__init__.py` 一样，体现分层目录的可扩展意图。
- 当前不被任何运行时模块导入，处于未激活状态。

## 功能点目的

虽然目标文件为空，但在架构上仍有三个明确目的：

1. **命名空间稳定性**
- 给未来公共工具模块（例如路径注入、事件归一化、错误输出）提供稳定命名空间：`hookify.utils.*`。

2. **结构分层表达**
- 在仓库目录上提前声明“工具层”存在，避免未来把通用能力继续散落在 `hooks/*.py` 或 `core/*.py`。

3. **低风险演进锚点**
- 当前文件无副作用、无导出、无运行依赖，后续可渐进式引入函数，不会影响现有 Hook 行为。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 目标文件实现现状

- `plugins/hookify/utils/__init__.py`：空文件（0 bytes）。
- 代码层面没有 `import`、没有变量、没有函数、没有副作用。

### 2) 其所在运行链路（用于理解上下文依赖）

#### Hook 事件入口

- `hooks.json` 为四类事件注册命令型 Hook：
  - `PreToolUse` -> `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/pretooluse.py`
  - `PostToolUse` -> `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/posttooluse.py`
  - `Stop` -> `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/stop.py`
  - `UserPromptSubmit` -> `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/userpromptsubmit.py`

#### 执行流程

1. Hook 脚本从 `stdin` 读 JSON 输入。
2. 根据事件或工具类型计算 event（如 `Bash -> bash`，`Edit/Write/MultiEdit -> file`）。
3. 调用 `load_rules(event=...)` 读取 `.claude/hookify.*.local.md`。
4. 使用 `RuleEngine.evaluate_rules` 进行条件求值。
5. 向 `stdout` 输出协议 JSON（block/warn/allow）。
6. 无论异常与否，脚本都 `sys.exit(0)`，避免因为插件错误中断主流程。

### 3) 关键数据结构

在 `core/config_loader.py` 中定义：

- `Condition`
  - `field`、`operator`、`pattern`
- `Rule`
  - `name`、`enabled`、`event`、`pattern`（legacy）
  - `conditions`（新格式）
  - `action`（`warn`/`block`）
  - `tool_matcher`、`message`

`Rule.from_dict` 支持两种规则来源：

- 简单格式：`pattern`（自动推导单条件）
- 高级格式：`conditions`（多条件 AND）

### 4) 协议与命令

#### Hook 输入协议（由 Claude Code 注入）

典型字段包含：
- `hook_event_name`
- `tool_name`
- `tool_input`
- `reason`（Stop）
- `transcript_path`（Stop 场景可读会话转录）
- `user_prompt`（UserPromptSubmit）

#### Hook 输出协议（RuleEngine）

- 阻断（Pre/PostToolUse）：返回 `hookSpecificOutput.permissionDecision = "deny"` + `systemMessage`
- 阻断（Stop）：返回 `decision = "block"` + `reason` + `systemMessage`
- 警告：返回 `systemMessage`
- 无匹配：返回空对象 `{}`

#### 支持的条件操作符

- `regex_match`
- `contains`
- `equals`
- `not_contains`
- `starts_with`
- `ends_with`

### 5) 与目标文件的关系

上述完整链路中未出现对 `hookify.utils` 的导入或调用。当前 `utils/__init__.py` 仅作为“未来可承载公共逻辑”的占位符存在。

## 关键代码路径与文件引用

### 目标对象

- `plugins/hookify/utils/__init__.py`（空文件）

### 上游调用场景（配置与入口）

- `plugins/hookify/hooks/hooks.json`：Hook 事件注册与执行命令
- `plugins/hookify/.claude-plugin/plugin.json`：插件元信息

### 直接执行器（实际运行）

- `plugins/hookify/hooks/pretooluse.py`
- `plugins/hookify/hooks/posttooluse.py`
- `plugins/hookify/hooks/stop.py`
- `plugins/hookify/hooks/userpromptsubmit.py`

### 核心被调用模块

- `plugins/hookify/core/config_loader.py`
  - frontmatter 提取：`extract_frontmatter`
  - 规则扫描：`load_rules`（扫描 `.claude/hookify.*.local.md`）
  - 单文件加载：`load_rule_file`
- `plugins/hookify/core/rule_engine.py`
  - 主求值：`evaluate_rules`
  - 条件求值：`_check_condition`
  - 字段提取：`_extract_field`
  - 正则缓存：`compile_regex`（`lru_cache(maxsize=128)`）

### 规则 DSL、命令、示例文档

- `plugins/hookify/README.md`
- `plugins/hookify/commands/hookify.md`
- `plugins/hookify/commands/list.md`
- `plugins/hookify/commands/configure.md`
- `plugins/hookify/commands/help.md`
- `plugins/hookify/skills/writing-rules/SKILL.md`
- `plugins/hookify/examples/*.local.md`

### 测试/脚本相关（外部工具链）

- `plugins/plugin-dev/skills/hook-development/scripts/validate-hook-schema.sh`
- `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh`

说明：`hookify` 插件目录下未发现专属自动化测试目录（无 `tests/`）。

## 依赖与外部交互

### 代码依赖

- Python 标准库：`os`、`sys`、`json`、`glob`、`re`、`dataclasses`、`functools`
- 当前 `utils/__init__.py` 无任何依赖。

### 环境变量

- `CLAUDE_PLUGIN_ROOT`：Hook 脚本用于动态注入 `sys.path`，保证 `from hookify.core...` 可导入。

### 文件系统交互

- 读取规则：`.claude/hookify.*.local.md`
- 读取转录：`transcript_path`（Stop 规则 `field: transcript`）
- 输出协议：写 `stdout` JSON，警告/错误写 `stderr`（部分路径）

### 与用户配置和文档的交互

- 用户通过 `/hookify` 命令或手工编辑规则文件驱动行为。
- `README`、`help`、`writing-rules` 技能共同定义规则 DSL 使用方式。

### 与脚本工具链交互

- `validate-hook-schema.sh` 可做 hooks.json 静态检查。
- `test-hook.sh` 可构造事件输入并执行 hook 脚本验证输出。

## 风险、边界与改进建议

### 风险与边界

1. `utils` 层未落地，公共逻辑重复
- 4 个 hook 执行脚本都在重复 `CLAUDE_PLUGIN_ROOT` 路径注入、异常包装、固定退出策略。
- 当前重复尚可维护，但演进时易出现行为漂移。

2. `utils/__init__.py` 空实现可读性边界弱
- 对新维护者来说，目录存在但无说明，易误判为“遗漏实现”或“死代码”。

3. 自实现 frontmatter 解析器能力有限
- `extract_frontmatter` 是 YAML 子集解析，不是完整 YAML 语义。
- 复杂 YAML（嵌套、转义、特殊类型）可能出现兼容边界。

4. 校验脚本与仓库实际 hooks 结构存在契约偏差
- `validate-hook-schema.sh` 要求每个事件项有 `matcher`。
- `plugins/hookify/hooks/hooks.json` 当前事件项未显式包含 `matcher`，静态校验可能报错。

5. 自动化测试缺口
- `hookify` 目录未内建测试；主要依赖人工触发或外部脚本验证。

### 改进建议（可渐进实施）

1. 在 `hookify/utils/` 引入运行时公共模块
- 例如 `runtime.py`：封装 `sys.path` 注入、统一错误输出、统一 stdin/stdout JSON 包装。
- 由 `hooks/*.py` 统一调用，减少重复与分叉。

2. 为 `utils/__init__.py` 增加最小文档注释
- 说明其为工具层命名空间入口与预留扩展点，降低认知成本。

3. 增加最小 smoke tests
- 覆盖 `bash/file/stop/prompt` 四类事件的“匹配/不匹配/阻断协议”。
- 可先复用 `plugin-dev` 的 `test-hook.sh` 形成 CI 前置脚本。

4. 对齐 hooks.json 校验规范
- 二选一：为 hook 条目显式补 `matcher`，或升级/调整校验脚本支持 wrapper 结构。

5. 逐步把“规则文件解析与协议定义”抽象成可测试函数
- 先从无副作用纯函数开始（event 映射、输出格式化），便于单测与复用。

