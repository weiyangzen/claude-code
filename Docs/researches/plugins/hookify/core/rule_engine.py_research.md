# FILE `plugins/hookify/core/rule_engine.py` 研究文档

## 场景与职责

`rule_engine.py` 是 Hookify 的规则判定层。  
它接收 `config_loader` 产出的 `Rule` 列表与 hook 事件输入 JSON，完成“条件匹配 -> 动作优先级处理 -> Hook 协议结果构造”。

在整体链路中的职责定位：

1. 上游：`hooks/*.py` 读取 stdin 后调用 `RuleEngine.evaluate_rules(...)`。
2. 本层：规则匹配、字段抽取、运算符求值、block/warn 合并。
3. 下游：向 hook 执行器返回 dict，由执行器序列化为 stdout JSON 给 Claude Hook runtime。

## 功能点目的

1. 统一规则求值入口。
- `evaluate_rules` 扫描全部候选规则，收集 `blocking_rules` 与 `warning_rules`。

2. 提供可扩展的条件运算符机制。
- `_check_condition` 支持 `regex_match/contains/equals/not_contains/starts_with/ends_with`。

3. 实现跨事件的字段抽取适配。
- `_extract_field` 兼容 `Bash`、`Write/Edit`、`MultiEdit`、`Stop`、`UserPromptSubmit` 常见字段。

4. 按 Hook 事件输出不同协议格式。
- `Stop` 阻断：`decision=block` + `reason`
- `PreToolUse/PostToolUse` 阻断：`hookSpecificOutput.permissionDecision=deny`
- 其他场景：仅 `systemMessage`

5. 性能优化。
- `compile_regex` 使用 `@lru_cache(maxsize=128)`，降低重复 pattern 编译成本。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 主流程：`evaluate_rules`

输入：

- `rules: List[Rule]`
- `input_data: Dict[str, Any]`（来自 hook stdin JSON，常见字段有 `hook_event_name/tool_name/tool_input`）

流程：

1. 逐条调用 `_rule_matches(rule, input_data)`。
2. 按 `rule.action` 放入 `blocking_rules` 或 `warning_rules`。
3. 若有 `blocking_rules`，优先返回阻断响应（高于 warn）。
4. 若仅有 `warning_rules`，返回合并后的 `systemMessage`。
5. 无命中返回空对象 `{}`。

消息合并策略：

- 每条规则消息前缀 `**[rule.name]**`
- 多条消息以双换行拼接，保留来源可追踪性。

### 2) 规则匹配：`_rule_matches`

判定步骤：

1. 读取 `tool_name` 与 `tool_input`。
2. 若规则声明 `tool_matcher`，先调用 `_matches_tool` 过滤。
3. 若规则无条件（`conditions` 为空）直接返回 `False`。
4. 遍历条件并执行 `_check_condition`，全部满足才命中（AND 语义）。

`_matches_tool` 语义：

- `*` 匹配所有工具。
- 否则按 `|` 分割，执行 `tool_name in patterns` 的精确匹配（非正则、无 trim）。

### 3) 条件判定：`_check_condition`

1. 调用 `_extract_field` 获取待匹配文本。
2. 若字段不存在返回 `False`。
3. 根据 `operator` 分派运算：
   - `regex_match`: `_regex_match(pattern, text)`
   - `contains`: `pattern in text`
   - `equals`: `pattern == text`
   - `not_contains`: `pattern not in text`
   - `starts_with`: `text.startswith(pattern)`
   - `ends_with`: `text.endswith(pattern)`
4. 未知 operator 返回 `False`。

### 4) 字段抽取：`_extract_field`

字段来源优先级：

1. 优先直接读 `tool_input[field]`（存在则转字符串返回）。
2. 若有 `input_data`，处理事件字段：
   - `reason` -> `input_data['reason']`
   - `transcript` -> 读取 `input_data['transcript_path']` 文件全文
   - `user_prompt` -> `input_data['user_prompt']`
3. 按工具类型特化：
   - `Bash.command`
   - `Write/Edit` 的 `content/new_string/old_string/file_path`
   - `MultiEdit` 的 `file_path` 与全部 edits `new_string` 拼接

异常处理：

- transcript 文件读取失败会写 stderr warning 并返回空字符串，避免中断 hook 流程。

### 5) 正则匹配与缓存

`compile_regex(pattern)` 使用全局 LRU 缓存，固定 `re.IGNORECASE` 编译。  
`_regex_match` 捕获 `re.error`，输出错误后返回 `False`，防止坏规则导致进程异常。

### 6) 协议输出细节

阻断输出依据 `input_data['hook_event_name']`：

- `Stop`：`{"decision":"block","reason":"...","systemMessage":"..."}`
- `PreToolUse|PostToolUse`：`{"hookSpecificOutput":{"hookEventName":..., "permissionDecision":"deny"},"systemMessage":"..."}`
- 其他事件：`{"systemMessage":"..."}`

这决定了宿主 Claude Hook runtime 在不同事件阶段如何解释规则结果。

## 关键代码路径与文件引用

- 核心实现：
  - `plugins/hookify/core/rule_engine.py:14-25` (`compile_regex`)
  - `plugins/hookify/core/rule_engine.py:35-94` (`evaluate_rules`)
  - `plugins/hookify/core/rule_engine.py:96-125` (`_rule_matches`)
  - `plugins/hookify/core/rule_engine.py:127-143` (`_matches_tool`)
  - `plugins/hookify/core/rule_engine.py:144-180` (`_check_condition`)
  - `plugins/hookify/core/rule_engine.py:182-254` (`_extract_field`)
  - `plugins/hookify/core/rule_engine.py:256-273` (`_regex_match`)

- 输入来源调用方：
  - `plugins/hookify/hooks/pretooluse.py:55-59`
  - `plugins/hookify/hooks/posttooluse.py:48-52`
  - `plugins/hookify/hooks/stop.py:40-44`
  - `plugins/hookify/hooks/userpromptsubmit.py:40-44`

- 上游规则模型：
  - `plugins/hookify/core/config_loader.py:15-84`
  - `plugins/hookify/core/config_loader.py:198-241`

- 规则文档与示例：
  - `plugins/hookify/README.md:93-260`
  - `plugins/hookify/examples/*.local.md`

## 依赖与外部交互

1. 代码依赖：
- Python 标准库 `re/sys/functools.lru_cache/typing`
- 规则模型来自 `hookify.core.config_loader` 的 `Rule/Condition`

2. 文件系统交互：
- 仅在字段为 `transcript` 时读取 `transcript_path` 指向文件
- 读取错误写 stderr，不抛出到主流程

3. 协议交互：
- 输入：hook 执行器传入的事件字典（从 stdin JSON 解码）
- 输出：可 JSON 序列化 dict（由 hook 脚本写 stdout）

4. 运行时行为边界：
- 该文件不负责进程退出码，进程 fail-open 策略由 `hooks/*.py` 统一控制（最终 `exit 0`）

## 风险、边界与改进建议

1. `not_contains` 是字面包含，不是正则否定。
- 风险：示例里若写 `pattern: npm test|pytest|cargo test`，会被当作整段字面文本，语义偏离预期。
- 建议：新增 `not_regex_match`，或在文档中明确 `not_contains` 仅用于字面字符串。

2. `tool_matcher` 分割后不做空白清理。
- 风险：`"Edit | Write"` 会生成 `"Edit "` 与 `" Write"`，无法命中真实工具名。
- 建议：`matcher.split('|')` 后做 `strip()`，并支持大小写标准化。

3. `PreToolUse` 与 `PostToolUse` 共用 deny 输出。
- 风险：Post 阶段工具已执行，`permissionDecision=deny` 在语义上可能无法阻止既成操作，易让规则作者误判效果。
- 建议：区分 pre/post 行为，post 统一退化为提示或增加明确“事后告警”协议字段。

4. transcript 读取存在重复 I/O。
- 风险：多规则多条件都匹配 `transcript` 时会重复读取同一文件，增加 stop 阶段延迟。
- 建议：在一次 `evaluate_rules` 内缓存 transcript 内容（按 path 缓存）。

5. 正则默认忽略大小写且不可配置。
- 风险：某些规则需要大小写敏感匹配时无法表达。
- 建议：为条件增加可选 flags（如 `regex_flags`），或约定 `operator` 变体（`regex_match_cs`）。

6. 未知 operator 静默失败。
- 风险：规则拼写错误时只会“不命中”，排障成本高。
- 建议：记录结构化警告（含 rule/condition 索引），并在 `hookify:list/doctor` 中展示。

7. 自动化测试缺位。
- 建议优先补齐：
  - 运算符语义测试（尤其 `not_contains` 与 regex 错误）
  - 各事件阻断协议测试（Stop/Pre/Post）
  - transcript 文件读取异常分支测试
