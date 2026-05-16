# FILE `plugins/hookify/examples/require-tests-stop.local.md` 研究文档

## 场景与职责

`require-tests-stop.local.md` 是 Hookify 的“会话结束前质量闸门”示例，用于在 `Stop` 事件触发时检查 transcript 中是否出现测试命令痕迹，不满足时阻断结束。

其设计定位是流程治理模板，而非默认启用规则。文件里 `enabled: false`（`plugins/hookify/examples/require-tests-stop.local.md:3`）明确将其作为可选强化策略。

调用关系：
- 调用方：README/帮助文档/写规则 skill 将其作为 stop 事件示例。
- 被调用方：复制到 `.claude/hookify.*.local.md` 并启用后，由 `hooks/stop.py` 固定 `load_rules(event='stop')` 加载并执行。

## 功能点目的

1. 在任务收尾阶段增加“跑测”确认机制。
- 使用 `event: stop + action: block`（`require-tests-stop.local.md:4-5`），目标是把测试执行纳入交付门禁。

2. 展示 `conditions` 结构在 stop 事件上的写法。
- `field: transcript` + `operator: not_contains`（`require-tests-stop.local.md:7-8`）是该示例的核心教学点。

3. 通过默认禁用降低误拦截风险。
- 文案也提示“仅在需要严格执行时启用”（`require-tests-stop.local.md:22`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 规则结构与求值语义

规则配置：
- `conditions[0].field = transcript`
- `conditions[0].operator = not_contains`
- `conditions[0].pattern = npm test|pytest|cargo test`

求值过程：
1. `stop.py` 从 stdin 读入事件 JSON（`plugins/hookify/hooks/stop.py:34`）。
2. `load_rules(event='stop')` 仅加载 stop 规则（`stop.py:37`）。
3. `_extract_field('transcript', ...)` 读取 `input_data.transcript_path` 文件内容（`rule_engine.py:207-213`）。
4. `not_contains` 按“普通子串不包含”执行：`pattern not in field_value`（`rule_engine.py:172-173`）。
5. 命中 block 后若 `hook_event_name == 'Stop'`，返回 `{"decision":"block","reason":"...","systemMessage":"..."}`（`rule_engine.py:66-71`）。

### 2) 关键语义陷阱（实测）

该示例的 `pattern: npm test|pytest|cargo test` 看起来像“正则 OR”，但在 `not_contains` 下不是正则，而是字面字符串。

仓库内实测结果：
- transcript 包含 `pytest` 时，规则仍触发 block（因为不包含完整字面量 `npm test|pytest|cargo test`）。
- transcript 仅当包含整段字面量 `npm test|pytest|cargo test` 时，规则才不触发。

结论：当前示例与其文案“检测任一测试命令”存在行为偏差。

### 3) 命令与协议

- Hook 入口命令：`python3 ${CLAUDE_PLUGIN_ROOT}/hooks/stop.py`（`plugins/hookify/hooks/hooks.json:31`）。
- Stop 阻断协议：`decision=block` + `reason` + `systemMessage`。
- hook 脚本异常时始终 `exit(0)`，避免因 hook 程序错误导致系统级中断（`stop.py:53-55`）。

## 关键代码路径与文件引用

- 示例规则：`plugins/hookify/examples/require-tests-stop.local.md`
- Stop 事件执行：
  - `plugins/hookify/hooks/stop.py:30`
  - `plugins/hookify/hooks/hooks.json:25`
- 规则加载：
  - `plugins/hookify/core/config_loader.py:198`
  - `plugins/hookify/core/config_loader.py:219`
- transcript 读取与条件计算：
  - `plugins/hookify/core/rule_engine.py:157`
  - `plugins/hookify/core/rule_engine.py:207`
  - `plugins/hookify/core/rule_engine.py:172`
- 输出协议：`plugins/hookify/core/rule_engine.py:66-71`
- 文档引用：
  - `plugins/hookify/README.md:180`
  - `plugins/hookify/commands/help.md:107`
  - `plugins/hookify/skills/writing-rules/SKILL.md:217`

## 依赖与外部交互

1. 依赖
- Python 运行环境与 hookify core 模块。
- `CLAUDE_PLUGIN_ROOT` 用于导入路径注入（`stop.py:13-24`）。

2. 外部交互
- 输入：Stop 事件 JSON（关键字段 `hook_event_name/reason/transcript_path`）。
- 文件读取：通过 `transcript_path` 读取会话记录文本。
- 输出：Stop block 协议 JSON。

3. 测试与脚本
- Hookify 无专属自动化测试；本规则建议至少做 transcript 语义回归。
- 可用 `test-hook.sh --create-sample Stop` 生成样例，再替换 `transcript_path` 进行验证。
- 额外可用 `python3 -c` 脚本直连 `RuleEngine` 做条件语义单测。

## 风险、边界与改进建议

1. 关键风险：`not_contains` + `|` 写法与用户直觉不一致。
- 影响：即使执行过 `pytest`，也可能被误判为未跑测并阻断 Stop。
- 建议：
  - 方案 A：新增 `not_regex_match` 操作符；
  - 方案 B：用 `regex_match` + 反向逻辑字段（如 `negate: true`）；
  - 方案 C：拆多规则并支持 OR 组合。

2. 边界：缺失 `transcript_path` 时规则不触发。
- 机制：`field='transcript'` 在无 `transcript_path` 时返回 `None`，条件直接失败。
- 建议：对缺 transcript 的 stop 场景增加显式警告或 fallback 字段（例如 `reason`）。

3. 边界：默认 `enabled: false` 可能导致用户误以为已生效。
- 建议：在 message 或注释中强化“需手动启用”提示；或提供 `/hookify:configure` 快捷启用引导。

4. 改进：提供官方“测试命令检测”稳定模板。
- 建议模板：
  - `conditions: [{field: transcript, operator: regex_match, pattern: '(npm test|pytest|cargo test)'}]`
  - 再配合“未匹配时阻断”的原生机制，避免将 OR 写在 `not_contains` 中。
