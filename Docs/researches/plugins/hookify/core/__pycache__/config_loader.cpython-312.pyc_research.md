# FILE `plugins/hookify/core/__pycache__/config_loader.cpython-312.pyc` 研究文档

## 场景与职责

该文件是 `plugins/hookify/core/config_loader.py` 的 CPython 3.12 字节码缓存，属于 Hookify 的“规则配置解析层”运行产物。

它参与的真实业务场景是：每次 Hook 事件触发时，`hooks/*.py` 调用 `load_rules(event=...)` 从 `.claude/hookify.*.local.md` 读取规则，再交给 `RuleEngine` 做匹配决策。

在职责上，该 `.pyc` 对应源码承担三件事：

1. 定义规则数据模型（`Condition`、`Rule`）。
2. 解析 markdown frontmatter（含 `conditions` 与 legacy `pattern` 兼容）。
3. 批量加载规则并按 `event/enabled` 过滤。

## 功能点目的

1. 规则输入标准化。
- 把松散 frontmatter 字段转换为结构化 `Rule/Condition` 对象。

2. 新旧格式兼容。
- 支持 `conditions` 列表。
- 兼容旧的 `pattern` 单字段，并按事件推断默认匹配字段。

3. 容错加载。
- 单个规则文件损坏不影响其它规则。
- 解析异常写 stderr warning，执行链保持可继续。

4. 运行时性能。
- 通过 `.pyc` 降低每次 hook 进程导入时的源码编译开销。

## 具体技术实现（关键流程/数据结构/协议/命令）

1. 字节码与模块指纹。
- `magic`: `cb0d0d0a`（CPython 3.12）。
- `bitfield`: `0`（timestamp-based）。
- `source_size`: `9690`，与 `config_loader.py` 文件大小一致。
- 顶层 `nested_code`：`Condition`、`Rule`、`extract_frontmatter`、`load_rules`、`load_rule_file`。

2. 数据结构实现。
- `Condition(field, operator, pattern)`，提供 `from_dict`。
- `Rule(name, enabled, event, pattern, conditions, action, tool_matcher, message)`，提供 `from_dict`。
- `Rule.from_dict` 优先使用 `conditions`；若无则将 legacy `pattern` 自动转成单条件。

3. frontmatter 解析流程（`extract_frontmatter`）。
- 要求文本以 `---` 开头；否则返回空 frontmatter。
- 使用 `split('---', 2)` 切分 frontmatter 与 message。
- 手写 YAML 子集解析：支持顶层键值、列表项、列表中的多行字典、布尔 `true/false`。
- 输出 `(frontmatter_dict, message_body)`。

4. 规则加载流程（`load_rules`/`load_rule_file`）。
- 用 `glob('.claude/hookify.*.local.md')` 发现规则文件。
- 每个文件执行 `load_rule_file`：读取文本 -> 解析 frontmatter -> 构建 `Rule`。
- `load_rules(event)` 只保留 `enabled=True` 且事件匹配（`rule.event == event` 或 `all`）。
- 对 `IOError/OSError/PermissionError/ValueError/...` 分类型告警并跳过。

5. 上下游协议关系。
- 上游：Hook 执行器从 stdin 接收 JSON 后，根据工具类型映射出 `event`（`bash/file/stop/prompt`）。
- 下游：`RuleEngine.evaluate_rules(rules, input_data)` 使用本模块输出的规则对象完成最终 block/warn 结果。

6. 本次核验命令（用于字节码与源码一致性）。
- `python3` + `marshal` 读取 `.pyc` 头与代码对象。
- `rg -n "load_rules|RuleEngine" plugins/hookify/hooks` 追踪调用方。

## 关键代码路径与文件引用

- 字节码目标：`plugins/hookify/core/__pycache__/config_loader.cpython-312.pyc`
- 对应源码：`plugins/hookify/core/config_loader.py`
- 直接调用方：
- `plugins/hookify/hooks/pretooluse.py`
- `plugins/hookify/hooks/posttooluse.py`
- `plugins/hookify/hooks/stop.py`
- `plugins/hookify/hooks/userpromptsubmit.py`
- 规则格式文档与示例：
- `plugins/hookify/README.md`
- `plugins/hookify/examples/dangerous-rm.local.md`
- `plugins/hookify/examples/console-log-warning.local.md`
- `plugins/hookify/examples/require-tests-stop.local.md`
- `plugins/hookify/examples/sensitive-files-warning.local.md`
- 引擎消费者：`plugins/hookify/core/rule_engine.py`

## 依赖与外部交互

1. 依赖标准库。
- `os/glob` 做规则发现。
- `dataclasses/typing` 做结构化建模。
- `sys` 输出 warning/error。

2. 文件系统交互。
- 读取项目根目录 `.claude/hookify.*.local.md`。
- 规则文本编码错误、权限错误、路径错误均做降级处理。

3. 与外部运行时交互。
- 不直接执行命令、不联网。
- 通过返回 `Rule` 列表影响 Hook 协议输出结果（由 `rule_engine` 完成最终序列化内容）。

4. 与配置生态交互。
- 与 `hookify:writing-rules` 技能、README 中的 rule format 保持协议一致。

## 风险、边界与改进建议

1. YAML 解析边界有限。
- 当前是手写子集解析器，复杂 YAML（深嵌套、转义、复杂数组）可能误解析。
- 建议：引入严格 schema 校验，至少在失败时给出行号级错误。

2. `pattern` 兼容策略在部分事件存在语义偏差。
- legacy `event != bash/file` 时默认映射到 `content`，而下游未必有该字段，可能导致“规则已加载但不命中”。
- 建议：对 `stop/prompt` 强制提示使用 `conditions` 或提供更合理默认字段。

3. 规则加载顺序不稳定。
- `glob` 结果未排序时，多规则拼接消息顺序可能随文件系统变化。
- 建议：在 `load_rules` 中排序后再加载，保证输出可复现。

4. 字段类型鲁棒性。
- `enabled/action/event` 缺少严格类型约束，配置写错可能静默降级。
- 建议：在 `Rule.from_dict` 增加枚举校验与标准化转换。

5. 测试覆盖缺口。
- 当前主要依赖 `__main__` 手工测试段。
- 建议：新增自动化用例覆盖 frontmatter 解析、错误分支和 legacy 兼容行为。
