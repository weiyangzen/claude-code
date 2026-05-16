# FILE `plugins/hookify/examples/sensitive-files-warning.local.md` 研究文档

## 场景与职责

`sensitive-files-warning.local.md` 是 Hookify 的敏感文件操作提醒示例，用于在文件编辑相关工具上依据 `file_path` 特征进行安全告警。

该示例强调“路径级风险识别”而非内容检测：只要目标文件路径符合正则（`.env`、`credentials`、`secrets`），即触发警告。

定位关系：
- 调用方：README 与 help/skill 文档将其作为多条件 rules 的入门样例。
- 被调用方：复制到 `.claude/hookify.*.local.md` 后，由 `PreToolUse/PostToolUse` 事件链按 file 规则执行。

## 功能点目的

1. 提前提示“敏感资产被编辑”的操作风险。
- `action: warn`（`plugins/hookify/examples/sensitive-files-warning.local.md:5`）避免打断工作流，但强化安全意识。

2. 展示 `conditions` 形式（相较 simple pattern 更可扩展）。
- 使用 `conditions[].field/operator/pattern`（`sensitive-files-warning.local.md:6-9`）而非顶层 `pattern`，利于后续叠加内容条件。

3. 给用户提供安全治理的基础模板。
- 可快速扩展为“路径 + 内容”双重策略，例如匹配 `API_KEY=` 或私钥头部。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 规则模型

frontmatter 要点：
- `event: file`
- `action: warn`
- `conditions[0] = {field: file_path, operator: regex_match, pattern: \.env$|\.env\.|credentials|secrets}`

因为已显式给出 `conditions`，`Rule.from_dict` 不再走 simple pattern 自动推导路径（`plugins/hookify/core/config_loader.py:50-58`）。

### 2) 匹配执行流程

1. `PreToolUse/PostToolUse` 根据 `tool_name in [Edit, Write, MultiEdit]` 选择 `event='file'`（`pretooluse.py:48-49`，`posttooluse.py:41-42`）。
2. `load_rules(event='file')` 返回启用规则集合（`config_loader.py:219-226`）。
3. `RuleEngine._extract_field('file_path', ...)` 对 `Edit/Write/MultiEdit` 都可读取路径（`rule_engine.py:243-249`）。
4. `regex_match` 使用忽略大小写匹配模式（`rule_engine.py:24,266-269`）。
5. 命中后输出 `systemMessage` 告警文本，不返回 deny。

### 3) 协议与命令

- Hook 命令注册：`python3 ${CLAUDE_PLUGIN_ROOT}/hooks/pretooluse.py`、`posttooluse.py`。
- 输出协议：warn 统一 `{"systemMessage":"..."}`。
- 仓库内实测：`tool_input.file_path = .env.production` 时命中并输出告警，符合预期。

## 关键代码路径与文件引用

- 示例规则：`plugins/hookify/examples/sensitive-files-warning.local.md`
- file 事件入口：
  - `plugins/hookify/hooks/pretooluse.py:41-53`
  - `plugins/hookify/hooks/posttooluse.py:36-46`
- 规则解析：
  - `plugins/hookify/core/config_loader.py:45-84`
  - `plugins/hookify/core/config_loader.py:198-241`
- 字段提取与匹配：
  - `plugins/hookify/core/rule_engine.py:182-254`
  - `plugins/hookify/core/rule_engine.py:256-273`
- 文档上下文：
  - `plugins/hookify/README.md:99`
  - `plugins/hookify/commands/help.md:87`
  - `plugins/hookify/skills/writing-rules/SKILL.md:69`

## 依赖与外部交互

1. 依赖
- Python 标准库，无第三方依赖。
- `CLAUDE_PLUGIN_ROOT` 动态路径注入，保证 `hookify.core` 导入。

2. 外部交互
- 输入：Hook Runtime 提供的 tool 事件 JSON。
- 读取：规则文件从 `.claude/hookify.*.local.md` 动态读取。
- 输出：告警 message 输出到 stdout JSON 的 `systemMessage` 字段。

3. 测试与脚本
- 该规则适合用 `test-hook.sh --create-sample PreToolUse` 生成输入，再替换 `tool_name/tool_input.file_path` 验证。
- `plugins/hookify` 目录当前无自动化用例对“路径正则命中边界”做持续回归。

## 风险、边界与改进建议

1. 风险：仅看路径，不看内容。
- 影响：编辑 `.env.example` 或测试凭证文件也会提示，误报率偏高。
- 建议：增加第二条件，如 `new_text` 包含 `API_KEY|SECRET|TOKEN`，形成高置信告警。

2. 风险：路径模式可能遗漏常见敏感文件。
- 当前模式未覆盖：`.npmrc`、`.pypirc`、`id_rsa`、`*.pem`、云凭证目录等。
- 建议：提供“基础/严格”两档模式，允许项目定制。

3. 边界：忽略大小写匹配可能扩大命中范围。
- 影响：`Secrets.md` 这种非敏感文档名也可能命中 `secrets`。
- 建议：在 regex 中增加路径边界，如 `(^|/)(\.env(\.|$)|credentials|secrets)(/|$)`。

4. 改进：增强规则解释与处置指引。
- 建议在 message 中补“允许继续但请执行以下检查”的标准清单，并引导使用 secrets manager/扫描工具。
