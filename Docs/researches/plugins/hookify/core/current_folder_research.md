# DIR `plugins/hookify/core` 研究文档

## 场景与职责

`plugins/hookify/core` 是 Hookify 插件的执行内核，位于“规则文件”和“Hook 运行时”之间：

1. 把 `.claude/hookify.*.local.md` 规则文件解析为结构化对象（`Rule`/`Condition`）。
2. 在 Hook 触发时，根据当前事件输入做规则匹配和动作决策（`warn` 或 `block`）。
3. 以 Claude Hook 协议可消费的 JSON 输出结果（例如 `permissionDecision: deny`）。

上下游关系：

- 上游调用方：`plugins/hookify/hooks/pretooluse.py`、`posttooluse.py`、`stop.py`、`userpromptsubmit.py`。
- 下游依赖：项目根目录 `.claude/hookify.*.local.md` 规则文件、`transcript_path` 指向的会话文件（Stop 场景）。
- 配置入口：`plugins/hookify/hooks/hooks.json` 用命令行 `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/*.py` 注入运行。

对应位置：

- `plugins/hookify/core/config_loader.py:15`
- `plugins/hookify/core/rule_engine.py:27`
- `plugins/hookify/hooks/hooks.json:4`
- `plugins/hookify/hooks/pretooluse.py:26`

## 功能点目的

### 1) 规则数据模型标准化

- `Condition` 定义单个匹配条件：`field/operator/pattern`。
- `Rule` 定义规则元数据：`name/enabled/event/action/tool_matcher/message` 与条件列表。
- `Rule.from_dict` 兼容两种规则写法：
  - 简单写法：`pattern`。
  - 高级写法：`conditions` 列表。

对应：

- `plugins/hookify/core/config_loader.py:15`
- `plugins/hookify/core/config_loader.py:32`
- `plugins/hookify/core/config_loader.py:45`

### 2) Frontmatter + Markdown 消息拆分

- `extract_frontmatter` 从 markdown 里抽出 YAML frontmatter 和正文消息。
- 目标是让消息正文直接作为触发后的提示文本，无需额外模板层。

对应：

- `plugins/hookify/core/config_loader.py:87`
- `plugins/hookify/core/config_loader.py:103`

### 3) 规则装载与按事件过滤

- `load_rules(event=...)` 扫描 `.claude/hookify.*.local.md`。
- 仅返回 `enabled: true`，并按 `event`（或 `all`）过滤。

对应：

- `plugins/hookify/core/config_loader.py:198`
- `plugins/hookify/core/config_loader.py:210`
- `plugins/hookify/core/config_loader.py:225`

### 4) 规则匹配与阻断/告警协议输出

- `RuleEngine.evaluate_rules` 聚合所有命中规则。
- `action == block` 优先于 `warn`。
- 根据 Hook 事件类型输出不同协议：
  - `Stop`：`{"decision":"block","reason":"..."}`
  - `PreToolUse/PostToolUse`：`{"hookSpecificOutput":{"permissionDecision":"deny"}}`
  - 其他事件：仅 `systemMessage`

对应：

- `plugins/hookify/core/rule_engine.py:35`
- `plugins/hookify/core/rule_engine.py:60`
- `plugins/hookify/core/rule_engine.py:66`
- `plugins/hookify/core/rule_engine.py:72`

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. 运行链路（端到端）

1. Claude Hook 触发事件（如 `PreToolUse`）。
2. `hooks.json` 执行 Python 命令，例如：
   - `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/pretooluse.py`
3. hook 脚本从 `stdin` 读取 JSON 输入，映射 `event`（`bash/file/stop/prompt`）。
4. 调用 `load_rules(event=...)` 装载生效规则。
5. 调用 `RuleEngine.evaluate_rules(rules, input_data)` 计算输出。
6. hook 脚本将结果 JSON 输出到 `stdout`，并始终 `exit 0`（不因脚本异常导致硬失败）。

对应：

- `plugins/hookify/hooks/hooks.json:4`
- `plugins/hookify/hooks/pretooluse.py:39`
- `plugins/hookify/hooks/pretooluse.py:52`
- `plugins/hookify/hooks/pretooluse.py:56`
- `plugins/hookify/hooks/pretooluse.py:70`

### B. 数据结构与兼容策略

`Rule` 字段（核心）：

- `name`: 规则名。
- `enabled`: 启停。
- `event`: `bash/file/stop/prompt/all`。
- `pattern`: legacy 简写。
- `conditions`: 高级条件列表。
- `action`: `warn/block`。
- `tool_matcher`: 工具名白名单（`Bash`、`Edit|Write`、`*`）。
- `message`: frontmatter 之后的 markdown 文本。

兼容逻辑：如果未提供 `conditions` 且存在 `pattern`，按 `event` 推断默认匹配字段：

- `bash -> command`
- `file -> new_text`
- 其他 -> `content`

对应：

- `plugins/hookify/core/config_loader.py:35`
- `plugins/hookify/core/config_loader.py:57`
- `plugins/hookify/core/config_loader.py:62`
- `plugins/hookify/core/config_loader.py:65`

### C. 条件运算符与字段抽取

支持操作符：

- `regex_match`（正则，忽略大小写）
- `contains`
- `equals`
- `not_contains`
- `starts_with`
- `ends_with`

字段来源：

- 通用：优先 `tool_input[field]`。
- Stop 场景：`reason`、`transcript`（通过 `transcript_path` 读文件）。
- UserPromptSubmit 场景：`user_prompt`。
- 工具特化：
  - `Bash.command`
  - `Write/Edit/MultiEdit` 的 `new_text/old_text/content/file_path`

对应：

- `plugins/hookify/core/rule_engine.py:166`
- `plugins/hookify/core/rule_engine.py:182`
- `plugins/hookify/core/rule_engine.py:205`
- `plugins/hookify/core/rule_engine.py:209`
- `plugins/hookify/core/rule_engine.py:231`
- `plugins/hookify/core/rule_engine.py:246`

### D. 协议与命令细节

Hook 命令注册：

- `PreToolUse`: `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/pretooluse.py`
- `PostToolUse`: `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/posttooluse.py`
- `Stop`: `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/stop.py`
- `UserPromptSubmit`: `python3 ${CLAUDE_PLUGIN_ROOT}/hooks/userpromptsubmit.py`

阻断协议关键字段：

- `hookSpecificOutput.permissionDecision = "deny"`（Tool 事件）
- `decision = "block"`（Stop 事件）
- `systemMessage`（统一提示信息）

对应：

- `plugins/hookify/hooks/hooks.json:9`
- `plugins/hookify/hooks/hooks.json:20`
- `plugins/hookify/hooks/hooks.json:31`
- `plugins/hookify/hooks/hooks.json:42`
- `plugins/hookify/core/rule_engine.py:74`
- `plugins/hookify/core/rule_engine.py:68`

## 关键代码路径与文件引用

核心实现：

- `plugins/hookify/core/config_loader.py:15`（`Condition`/`Rule` 数据模型）
- `plugins/hookify/core/config_loader.py:87`（frontmatter 解析）
- `plugins/hookify/core/config_loader.py:198`（目录扫描与规则加载）
- `plugins/hookify/core/config_loader.py:244`（单文件读取）
- `plugins/hookify/core/rule_engine.py:14`（正则缓存 `lru_cache`）
- `plugins/hookify/core/rule_engine.py:35`（规则求值主流程）
- `plugins/hookify/core/rule_engine.py:96`（单条规则匹配）
- `plugins/hookify/core/rule_engine.py:182`（字段抽取）

直接调用方：

- `plugins/hookify/hooks/pretooluse.py:26`
- `plugins/hookify/hooks/posttooluse.py:22`
- `plugins/hookify/hooks/stop.py:22`
- `plugins/hookify/hooks/userpromptsubmit.py:22`
- `plugins/hookify/hooks/hooks.json:4`

规则来源与用户接口：

- `plugins/hookify/commands/hookify.md:84`（创建 `.claude/hookify.{name}.local.md`）
- `plugins/hookify/commands/list.md:16`（规则文件发现模式）
- `plugins/hookify/commands/configure.md:18`（配置入口）
- `plugins/hookify/README.md:71`（规则格式与事件说明）
- `plugins/hookify/examples/require-tests-stop.local.md:1`（Stop 场景样例）

测试与脚本现状：

- 无 `plugins/hookify/tests` 目录。
- `core` 中仅有 `__main__` 手工测试段，不属于自动化测试。
- 运行入口脚本为 `plugins/hookify/hooks/*.py`（而非专门测试脚本）。

对应：

- `plugins/hookify/core/config_loader.py:277`
- `plugins/hookify/core/rule_engine.py:276`

## 依赖与外部交互

### 1) 语言与库依赖

仅 Python 标准库：`glob`、`os`、`re`、`json`、`sys`、`dataclasses`、`functools.lru_cache`。

对应：

- `plugins/hookify/core/config_loader.py:7`
- `plugins/hookify/core/rule_engine.py:4`

### 2) 环境依赖

- 依赖 `CLAUDE_PLUGIN_ROOT` 注入 `sys.path`，保障 `from hookify.core...` 可导入。
- Hook 脚本通过 `stdin` 接收事件 payload，`stdout` 输出 JSON。

对应：

- `plugins/hookify/hooks/pretooluse.py:14`
- `plugins/hookify/hooks/pretooluse.py:39`
- `plugins/hookify/hooks/pretooluse.py:59`

### 3) 文件系统交互

- 读取规则：`.claude/hookify.*.local.md`（相对当前工作目录）。
- Stop 条件可读取 `transcript_path` 指向文件全文。

对应：

- `plugins/hookify/core/config_loader.py:210`
- `plugins/hookify/core/rule_engine.py:209`

### 4) 文档/命令交互

- `/hookify` 命令负责生成规则文件，`core` 负责执行期解析/匹配。
- `/hookify:list` 与 `/hookify:configure` 面向用户展示和启停规则，间接影响 `core` 加载结果。

对应：

- `plugins/hookify/commands/hookify.md:84`
- `plugins/hookify/commands/list.md:14`
- `plugins/hookify/commands/configure.md:16`

## 风险、边界与改进建议

### 1) frontmatter 解析器是手写 YAML 子集，兼容边界较窄

风险：

- 对复杂 YAML（嵌套、转义、类型推断、注释位置）支持有限。
- 规则内容稍复杂时，可能“静默解析偏差”而非显式失败。

证据：

- `plugins/hookify/core/config_loader.py:105`
- `plugins/hookify/core/config_loader.py:184`

建议：

- 方案 A：引入严格 YAML 解析并加 schema 校验。
- 方案 B：保留轻量实现但增加字段级验证与错误定位（行号/字段）。

### 2) `not_contains` 是字符串包含而非正则，和示例易产生语义错配

风险：

- `not_contains` 走 `pattern not in field_value`，不是 regex。
- 示例 `pattern: npm test|pytest|cargo test`（带 `|`）在 `not_contains` 下会当作字面字符串，导致 Stop 规则几乎总命中（若启用）。

证据：

- `plugins/hookify/core/rule_engine.py:172`
- `plugins/hookify/examples/require-tests-stop.local.md:9`

建议：

- 新增 `not_regex_match` 运算符，或让 `not_contains` 明确仅接受字面文本并在文档中限制示例写法。

### 3) 事件映射和字段抽取存在覆盖盲区

风险：

- `Pre/Post` 仅把 `Bash` 映射为 `bash`，`Edit/Write/MultiEdit` 映射为 `file`，其他工具默认 `event=None`（加载全部规则），可能带来误触发。
- `extract_field` 对未覆盖工具返回 `None`，规则会直接不匹配，导致“看似已配置但不生效”。

证据：

- `plugins/hookify/hooks/pretooluse.py:46`
- `plugins/hookify/hooks/pretooluse.py:49`
- `plugins/hookify/core/rule_engine.py:254`

建议：

- 增加工具-事件映射表与未知工具告警。
- 为常见工具补全字段适配（或提供统一 JSONPath 风格字段访问）。

### 4) 错误处理偏“吞错继续”，诊断信号不足

风险：

- `load_rules`/hook 主流程多处捕获大范围异常并继续，避免中断是优点，但也可能掩盖配置错误。

证据：

- `plugins/hookify/core/config_loader.py:228`
- `plugins/hookify/core/config_loader.py:236`
- `plugins/hookify/hooks/pretooluse.py:61`

建议：

- 增加可选 debug 模式（结构化错误详情、命中规则列表、解析耗时）。
- 在 `/hookify:list` 或独立 `doctor` 命令暴露“规则可解析性检查”。

### 5) 自动化测试缺位

风险：

- 当前只有 `__main__` 手工 smoke，缺少回归测试，解析器和协议输出容易在修改时退化。

证据：

- `plugins/hookify/core/config_loader.py:277`
- `plugins/hookify/core/rule_engine.py:276`

建议：

- 最小测试集优先级：
  1. frontmatter 解析表驱动用例（simple/conditions/quoted/malformed）。
  2. 运算符语义测试（尤其 `not_contains` 与 regex 系运算符）。
  3. 各事件输出协议快照测试（Pre/Post/Stop/UserPromptSubmit）。

