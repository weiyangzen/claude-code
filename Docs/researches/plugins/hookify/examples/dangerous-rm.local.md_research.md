# FILE `plugins/hookify/examples/dangerous-rm.local.md` 研究文档

## 场景与职责

`dangerous-rm.local.md` 是 Hookify 的高风险命令防护示例，面向 `Bash` 工具调用场景，对 `rm -rf` 模式进行命中并执行阻断。

该文件位于 `plugins/hookify/examples/`，本质是“规则模板”。真正运行时生效路径是项目根 `.claude/hookify.*.local.md`（`plugins/hookify/core/config_loader.py:210`），因此示例必须复制到 `.claude/` 并符合命名约定后才会被加载。

调用关系：
- 调用方（文档与命令）：`plugins/hookify/README.md`、`plugins/hookify/commands/help.md`、`plugins/hookify/commands/hookify.md`、`plugins/hookify/skills/writing-rules/SKILL.md`。
- 被调用方（执行链）：`hooks/pretooluse.py` / `hooks/posttooluse.py` -> `load_rules(event='bash')` -> `RuleEngine.evaluate_rules(...)`。

## 功能点目的

1. 将“高破坏性命令”从提醒升级为硬性门禁。
- `action: block`（`plugins/hookify/examples/dangerous-rm.local.md:6`）意味着命中后输出 deny 协议，而不是仅展示提示。

2. 以 simple pattern 最短路径定义 bash 规则。
- `event: bash + pattern: rm\s+-rf`（`plugins/hookify/examples/dangerous-rm.local.md:4-5`）让用户无需写 `conditions` 即可启用。

3. 作为 `/hookify` 产物的参考命名与写法。
- 文件名称与 rule 名称都贴近“动作+风险对象”风格（`block-dangerous-rm`），符合写规则 skill 的命名建议。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) Frontmatter 解析与条件推导

规则定义：
- `name: block-dangerous-rm`
- `enabled: true`
- `event: bash`
- `pattern: rm\s+-rf`
- `action: block`

`Rule.from_dict` 对 simple pattern 的推导结果：
- `Condition(field='command', operator='regex_match', pattern='rm\\s+-rf')`

推导依据：`event=bash` 时固定映射到 `field=command`（`plugins/hookify/core/config_loader.py:61-64`）。

### 2) 运行时流程

1. `PreToolUse`/`PostToolUse` hook 脚本从 stdin 读取事件 JSON（`pretooluse.py:39`，`posttooluse.py:34`）。
2. 依据 `tool_name == 'Bash'` 推断为 `event='bash'`（`pretooluse.py:46-47`，`posttooluse.py:39-40`）。
3. `load_rules(event='bash')` 扫描 `.claude/hookify.*.local.md`，仅返回启用规则（`config_loader.py:219-226`）。
4. `RuleEngine._extract_field('command', ...)` 读取 `tool_input.command`（`rule_engine.py:231-233`）。
5. `regex_match`（忽略大小写）匹配成功后，因 `action=block` 进入阻断分支（`rule_engine.py:55-79`）。
6. 对 `PreToolUse/PostToolUse` 返回 `hookSpecificOutput.permissionDecision='deny'`（`rule_engine.py:72-79`）。

### 3) 协议、命令与验证

- Hook 注册命令：`python3 ${CLAUDE_PLUGIN_ROOT}/hooks/pretooluse.py` / `posttooluse.py`（`plugins/hookify/hooks/hooks.json:9,20`）。
- 阻断协议（Pre/Post）：
  - `{"hookSpecificOutput":{"hookEventName":"PreToolUse|PostToolUse","permissionDecision":"deny"},"systemMessage":"..."}`

仓库内实测：对输入 `command: rm -rf /tmp/demo`，返回 deny 协议，且消息体包含规则 markdown 正文，符合设计预期。

## 关键代码路径与文件引用

- 示例规则：`plugins/hookify/examples/dangerous-rm.local.md`
- hook 入口：
  - `plugins/hookify/hooks/hooks.json:3`
  - `plugins/hookify/hooks/pretooluse.py:35`
  - `plugins/hookify/hooks/posttooluse.py:30`
- 配置加载：
  - `plugins/hookify/core/config_loader.py:198`
  - `plugins/hookify/core/config_loader.py:244`
- 规则执行：
  - `plugins/hookify/core/rule_engine.py:35`
  - `plugins/hookify/core/rule_engine.py:60`
  - `plugins/hookify/core/rule_engine.py:231`
- 文档与命令：
  - `plugins/hookify/README.md:75`
  - `plugins/hookify/commands/help.md:79`
  - `plugins/hookify/commands/hookify.md:181`

## 依赖与外部交互

1. 依赖
- Python 标准库与 `hookify.core` 模块。
- `CLAUDE_PLUGIN_ROOT` 环境变量用于动态加路径，保证 `from hookify.core...` 可导入（`pretooluse.py:14-27`）。

2. 外部交互
- 输入：Hook Runtime 注入的 `tool_name/tool_input.command/hook_event_name`。
- 输出：stdout JSON，block 时返回 deny 协议。
- 文件系统：规则来自项目 `.claude/`；示例目录不直接参与扫描。

3. 测试与脚本
- Hookify 自身无独立 `tests/`。
- 可使用 `plugins/plugin-dev/skills/hook-development/scripts/test-hook.sh` 构造 `PreToolUse` 样例进行回归。
- `validate-hook-schema.sh` 用于配置结构检查，但对 hookify 的 wrapper 风格并非完全贴合（可能误报 matcher 缺失）。

## 风险、边界与改进建议

1. 风险：匹配模式较宽，可能误拦截“非执行意图”的文本。
- 示例：命令中作为字符串参数出现 `rm -rf` 也会命中。
- 建议：将模式收紧为命令起始与参数边界，如 `(^|\s)rm\s+-rf(\s|$)`，并考虑排除 `echo`/注释上下文。

2. 边界：Pre 与 Post 都会运行同类规则。
- 影响：若某环境允许 Pre 通过而在 Post 再次匹配，可能产生重复告警/阻断语义不一致。
- 建议：为 block 规则默认只挂 Pre，或增加 `tool_matcher`/事件阶段字段。

3. 风险：示例文件默认位置不可直接生效。
- 建议：增加“复制到 `.claude/hookify.*.local.md`”的一键命令片段，降低误配置。

4. 改进：增加 destructive command 测试矩阵。
- 用例建议：`rm -rf /tmp/x`、`sudo rm -rf /`、`echo "rm -rf"`、`rm -r -f`、`rm -rfv`，并断言是否 deny。
