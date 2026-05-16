# FILE `plugins/hookify/examples/console-log-warning.local.md` 研究文档

## 场景与职责

`console-log-warning.local.md` 是 Hookify 的“文件编辑类质量提醒”示例规则，目标是在 Claude 进行 `Edit/Write/MultiEdit` 时，检测新增内容中的 `console.log(` 并给出提示，但不阻断执行。

该文件属于 `plugins/hookify/examples/` 示例目录，职责是模板与教学，不是运行时直接扫描目标。运行时只扫描项目工作目录下 `.claude/hookify.*.local.md`（`plugins/hookify/core/config_loader.py:210`）。

调用关系定位：
- 调用方（文档/命令层）：`plugins/hookify/README.md`、`plugins/hookify/commands/help.md`、`plugins/hookify/skills/writing-rules/SKILL.md` 将其作为可复制样例。
- 被调用方（运行链路）：复制到 `.claude/hookify.console-log-warning.local.md` 后，被 `hooks/pretooluse.py` 或 `hooks/posttooluse.py` 经 `load_rules -> RuleEngine` 消费。

## 功能点目的

1. 以最低侵入方式提醒调试语句残留风险。
- `action: warn` 明确告警而非 deny/block（`plugins/hookify/examples/console-log-warning.local.md:6`）。

2. 使用简写模式降低规则编写门槛。
- 通过 `event: file + pattern: console\.log\(`（`plugins/hookify/examples/console-log-warning.local.md:4-5`）触发自动条件映射，无需手写 `conditions`。

3. 作为规则 DSL 的“file 事件最小可用范式”。
- 对应 writing-rules skill 中 file 事件示例，帮助用户快速扩展为更复杂条件（文件路径 + 内容双条件）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) Frontmatter 到 Rule 的映射

文件原始配置：
- `name: warn-console-log`
- `enabled: true`
- `event: file`
- `pattern: console\.log\(`
- `action: warn`

解析后由 `Rule.from_dict` 生成：
- `conditions = [Condition(field='new_text', operator='regex_match', pattern='console\\.log\\(')]`

映射依据：`event=file` 的 simple pattern 会映射到 `field=new_text`（`plugins/hookify/core/config_loader.py:56-73`）。

### 2) 运行时关键流程

1. Claude 触发 `PreToolUse` 或 `PostToolUse`，Python hook 脚本读取 stdin JSON（`plugins/hookify/hooks/pretooluse.py:39`，`posttooluse.py:34`）。
2. 脚本按 `tool_name` 推断事件：`Edit/Write/MultiEdit -> file`（`pretooluse.py:48-49`，`posttooluse.py:41-42`）。
3. `load_rules(event='file')` 只加载启用且事件匹配规则（`plugins/hookify/core/config_loader.py:219-226`）。
4. `RuleEngine._extract_field('new_text', ...)` 在 `Edit/Write` 读取 `new_string`，在 `MultiEdit` 聚合 `edits[].new_string`（`plugins/hookify/core/rule_engine.py:235-253`）。
5. `regex_match` 使用 `re.IGNORECASE` + LRU 缓存编译（`plugins/hookify/core/rule_engine.py:14-25,266-269`）。
6. 命中后因 `action=warn` 返回 `{"systemMessage": ...}`，不下发 deny（`plugins/hookify/core/rule_engine.py:86-91`）。

### 3) 协议与命令

- Hook 注册命令：`python3 ${CLAUDE_PLUGIN_ROOT}/hooks/pretooluse.py` / `posttooluse.py`（`plugins/hookify/hooks/hooks.json:9,20`）。
- Hook 输出协议：warn 仅 `systemMessage`。
- 本次验证（仓库内实测）对 `Edit` 输入 `new_string: console.log("x")`，返回仅含 `systemMessage`，符合 warn 语义。

## 关键代码路径与文件引用

- 示例规则本体：`plugins/hookify/examples/console-log-warning.local.md`
- 规则扫描与解析：
  - `plugins/hookify/core/config_loader.py:198`
  - `plugins/hookify/core/config_loader.py:244`
- 条件匹配与协议输出：
  - `plugins/hookify/core/rule_engine.py:35`
  - `plugins/hookify/core/rule_engine.py:144`
  - `plugins/hookify/core/rule_engine.py:182`
- Hook 事件入口：
  - `plugins/hookify/hooks/pretooluse.py:35`
  - `plugins/hookify/hooks/posttooluse.py:30`
  - `plugins/hookify/hooks/hooks.json:3`
- 配置与写法文档：
  - `plugins/hookify/README.md:71`
  - `plugins/hookify/skills/writing-rules/SKILL.md:145`
  - `plugins/hookify/commands/help.md:95`

## 依赖与外部交互

1. 依赖
- Python 标准库：`json/os/sys/re/glob/dataclasses/functools`。
- 环境变量：`CLAUDE_PLUGIN_ROOT` 用于注入 `sys.path`（`pretooluse.py:14-23`）。

2. 外部交互
- 输入：Claude Hook Runtime 的 stdin JSON（包含 `hook_event_name/tool_name/tool_input`）。
- 输出：stdout JSON（warn 场景仅 `systemMessage`）。
- 文件系统：只从 `.claude/hookify.*.local.md` 读取规则，不读 `plugins/hookify/examples/`。

3. 测试与脚本上下文
- `plugins/hookify` 目录无专属自动化 tests。
- 可借 `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh` 生成 `PreToolUse` 样例并驱动 hook 脚本。
- `validate-hook-schema.sh` 可校验 hooks JSON，但其默认“顶层事件键 + matcher 必填”假设与 hookify 的 wrapper 结构不完全一致，使用时需注意。

## 风险、边界与改进建议

1. 风险：示例文件易被误当成“已激活规则”。
- 原因：运行时不扫描 `plugins/hookify/examples/*`。
- 建议：在示例顶部增加“必须复制到 `.claude/hookify.*.local.md` 才生效”的说明。

2. 边界：当前规则只看 `new_text`，不看文件路径。
- 影响：`docs/*.md`、测试样例中的 `console.log` 也会告警。
- 建议：升级为 `conditions` 双条件：`file_path` 限定源码后缀 + `new_text` 匹配 console.log。

3. 边界：大小写不敏感匹配。
- 影响：`Console.Log(` 这类非常规写法也会触发。
- 建议：如需大小写严格控制，新增 `regex_match_case_sensitive` 或在模式层加约束。

4. 改进：补齐回归测试。
- 建议新增最小用例：`Edit`、`Write`、`MultiEdit` 三类输入下的 warn 输出断言，防止字段映射回归。
