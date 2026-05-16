# FILE `plugins/hookify/core/__pycache__/rule_engine.cpython-312.pyc` 研究文档

## 场景与职责

该文件是 `plugins/hookify/core/rule_engine.py` 的 CPython 3.12 字节码缓存，对应 Hookify 的“规则判定与响应构造层”。

在运行链路中，它位于：

1. 上游：`hooks/*.py` 从 stdin 收到事件 JSON，并通过 `load_rules` 获取候选规则。
2. 本层：`RuleEngine.evaluate_rules` 执行条件匹配、阻断优先级处理、协议输出组装。
3. 下游：Hook 执行器把返回 dict 序列化到 stdout，交给 Claude Hook runtime。

## 功能点目的

1. 统一规则求值入口。
- 一个入口完成多规则扫描与结果合并。

2. 提供跨工具字段抽取能力。
- Bash、Edit/Write、MultiEdit、Stop、UserPromptSubmit 事件都可在统一条件模型下匹配。

3. 保障安全动作优先级。
- `action=block` 永远高于 `warn`。

4. 兼顾性能与容错。
- 正则编译采用 LRU 缓存。
- 非法正则、transcript 读取异常均降级为“不命中 + 告警”，不中断主流程。

## 具体技术实现（关键流程/数据结构/协议/命令）

1. 字节码与模块指纹。
- `magic`: `cb0d0d0a`（CPython 3.12）。
- `bitfield`: `0`（timestamp-based）。
- `source_size`: `10727`，与 `rule_engine.py` 一致。
- 顶层符号包含 `compile_regex`、`RuleEngine`，与源码结构一致。

2. 评估主流程（`evaluate_rules`）。
- 遍历规则并调用 `_rule_matches`。
- 命中后按 `action` 分流到 `blocking_rules` / `warning_rules`。
- 若存在 blocking 规则，立即按事件类型生成阻断输出。
- 否则若有 warning 规则，返回合并 `systemMessage`。
- 全部不命中返回空对象 `{}`。

3. 条件判定流程。
- `_rule_matches` 要求所有条件都命中（AND 语义）。
- `_check_condition` 支持 `regex_match/contains/equals/not_contains/starts_with/ends_with`。
- `_extract_field` 按字段来源分层提取：`tool_input` 直读优先，其次事件特定字段（`reason/transcript/user_prompt`），再按工具类型适配。

4. Stop 与 transcript 处理。
- 当条件字段为 `transcript` 时，读取 `input_data['transcript_path']` 文件全文。
- 文件不存在、权限不足、编码问题都仅写 stderr warning 并返回空字符串。

5. 正则缓存与异常策略。
- `compile_regex` 使用 `@lru_cache(maxsize=128)`，缓存编译后的 pattern。
- `_regex_match` 捕获 `re.error` 并返回 `False`，避免坏规则导致 hook 崩溃。

6. Hook 协议输出格式。
- `Stop` 阻断：`{"decision":"block","reason":"...","systemMessage":"..."}`。
- `PreToolUse/PostToolUse` 阻断：`{"hookSpecificOutput":{"hookEventName":...,"permissionDecision":"deny"},"systemMessage":"..."}`。
- 纯告警：`{"systemMessage":"..."}`。

## 关键代码路径与文件引用

- 字节码目标：`plugins/hookify/core/__pycache__/rule_engine.cpython-312.pyc`
- 对应源码：`plugins/hookify/core/rule_engine.py`
- 规则模型来源：`plugins/hookify/core/config_loader.py`
- Hook 入口调用：
- `plugins/hookify/hooks/pretooluse.py`
- `plugins/hookify/hooks/posttooluse.py`
- `plugins/hookify/hooks/stop.py`
- `plugins/hookify/hooks/userpromptsubmit.py`
- 事件绑定配置：`plugins/hookify/hooks/hooks.json`
- 规则示例/文档：
- `plugins/hookify/README.md`
- `plugins/hookify/examples/require-tests-stop.local.md`
- `plugins/hookify/examples/sensitive-files-warning.local.md`

## 依赖与外部交互

1. 依赖标准库。
- `re`、`functools.lru_cache`、`typing`、`sys`。

2. 与本地文件系统交互。
- 仅在 `transcript` 字段匹配时读取会话 transcript 文件。

3. 与 Hook 运行时协议交互。
- 输入为 hook 事件字典。
- 输出为 JSON 可序列化 dict，供 `hooks/*.py` 输出到 stdout。

4. 与规则生态交互。
- 消费 `config_loader` 生成的 `Rule/Condition`。
- 实际命中行为受规则文件 frontmatter 和工具输入结构共同决定。

## 风险、边界与改进建议

1. `not_contains` 语义易误解。
- 当前是字面子串判断，不是“正则不匹配”；示例若写 `a|b|c` 将被当成整串文本。
- 建议：新增 `not_regex_match` 或在文档中强调该语义。

2. `tool_matcher` 匹配偏严格。
- 仅按 `|` 分隔后做精确匹配，不自动 `strip` 空白，规则书写稍有空格就会失配。
- 建议：预处理空白并补充大小写一致性约束。

3. Post 阶段阻断语义边界。
- `PostToolUse` 返回 `permissionDecision=deny` 时，工具通常已执行完成，更多是告警语义而非真正阻断。
- 建议：区分 pre/post 输出策略，避免用户对“阻断时机”产生误判。

4. transcript 重复读取开销。
- 多条规则同时读 `transcript` 会重复文件 I/O。
- 建议：在一次评估周期内缓存 transcript 内容。

5. 可观测性不足。
- 未知 operator 或字段提取失败时只会“不命中”，缺少结构化诊断。
- 建议：增加 debug 级日志或 `hookify:doctor` 检查命令，帮助快速定位规则失效原因。
