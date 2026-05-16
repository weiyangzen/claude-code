# FILE `plugins/hookify/commands/help.md` 研究文档

## 场景与职责

`/hookify:help` 是 Hookify 的说明型命令，职责是把“规则 DSL + 命令入口 + 排障方法”集中给用户，降低学习和误用成本。它本身不修改规则文件，也不直接参与 hook 判定。

职责边界：
- 上游：用户需要快速了解 Hookify 的能力、命令、规则格式、regex 写法。
- 中游：命令只读（`allowed-tools: ["Read"]`），输出静态帮助信息。
- 下游：用户据此执行 `/hookify`、`/hookify:list`、`/hookify:configure` 或手工编辑 `.claude/hookify.*.local.md`。

关键定位：
- 命令定义：`plugins/hookify/commands/help.md:1`
- 插件总览中的命令暴露：`plugins/hookify/README.md:53-69`、`plugins/README.md:22`

## 功能点目的

1. 统一认知模型
- 解释 Hookify 是“通过规则文件驱动 hooks”，而非手工改 `hooks.json`（`plugins/hookify/commands/help.md:12-25`）。

2. 提供可执行最小示例
- 覆盖 bash/file/stop 场景示例，帮助用户快速上手规则结构（`plugins/hookify/commands/help.md:79-113`）。

3. 传达关键运行语义
- 明确“规则立即生效、无需重启”，减少误判和重复排障（`plugins/hookify/commands/help.md:134-140`）。

4. 建立排障入口
- 给出规则不触发、import 错误、pattern 不匹配的检查步骤（`plugins/hookify/commands/help.md:142-158`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

1. 命令协议
- frontmatter 仅声明 `description` 与 `allowed-tools: ["Read"]`（`plugins/hookify/commands/help.md:2-3`）。
- 该权限模型意味着命令不会产生副作用，定位为“文档内帮助”。

2. 文档结构与技术内容
- Hook 事件映射：PreToolUse/PostToolUse/Stop/UserPromptSubmit（`plugins/hookify/commands/help.md:18-23`），对应注册配置 `plugins/hookify/hooks/hooks.json:4-47`。
- 规则文件协议：frontmatter `name/enabled/event/pattern` + message body（`plugins/hookify/commands/help.md:28-49`）。
- 命令索引：`/hookify`、`/hookify:list`、`/hookify:configure`、`/hookify:help`（`plugins/hookify/commands/help.md:70-76`）。
- 正则提示：给出 Python regex 常用语法和示例（`plugins/hookify/commands/help.md:115-131`）。

3. 与运行时数据结构映射
- 文档中的 `enabled/event/pattern` 最终会进入 `Rule`/`Condition`：
  - 解析：`extract_frontmatter` + `Rule.from_dict`（`plugins/hookify/core/config_loader.py:87-195`、`plugins/hookify/core/config_loader.py:45-84`）。
  - 执行：`RuleEngine.evaluate_rules`（`plugins/hookify/core/rule_engine.py:35-94`）。
- `action: warn|block` 在帮助文档中被强调（`plugins/hookify/commands/help.md:136`），并在引擎里决定 deny/block/systemMessage 输出协议（`plugins/hookify/core/rule_engine.py:60-84`）。

4. 排障命令的可执行性
- 文档推荐 `python3 -c` 验证 regex，和运行时实现一致（`plugins/hookify/commands/help.md:148`，`plugins/hookify/core/rule_engine.py:266-273`）。

## 关键代码路径与文件引用

- 命令本体：`plugins/hookify/commands/help.md`
- 与之对应的用户文档：`plugins/hookify/README.md`
- Hook 事件注册：`plugins/hookify/hooks/hooks.json`
- Hook 执行器：`plugins/hookify/hooks/pretooluse.py`、`posttooluse.py`、`stop.py`、`userpromptsubmit.py`
- 规则解析与执行：`plugins/hookify/core/config_loader.py`、`plugins/hookify/core/rule_engine.py`
- 示例规则来源：`plugins/hookify/examples/*.local.md`

测试/脚本上下文：
- `help.md` 本身无自动测试。
- hookify 仓库内未见独立 `tests/` 覆盖“文档示例是否与运行时一致”。

## 依赖与外部交互

1. 依赖类型
- 依赖本地文档一致性：`help.md` 与 `README.md`、`SKILL.md` 需要长期同步。
- 依赖运行时契约稳定：事件名、字段名、`action` 语义来自 `core` 与 `hooks`。

2. 外部交互
- 无网络调用。
- 通过帮助文档把用户引导到本地文件系统路径 `.claude/hookify.*.local.md`（`plugins/hookify/commands/help.md:138`）。

3. 与其他命令交互
- 引导用户调用 `/hookify` 创建规则、`/hookify:list` 查看、`/hookify:configure` 启停，属于命令层导航中心（`plugins/hookify/commands/help.md:72-75`）。

## 风险、边界与改进建议

1. 文档与实现漂移风险
- 风险：帮助文本可能持续描述旧字段或旧行为，用户按文档操作却不生效。
- 建议：增加“示例规则解析自检”脚本，定期用 `config_loader` 验证示例 frontmatter。

2. 示例覆盖边界不足
- 风险：当前重点是简单 `pattern` 示例，`conditions` 多字段场景在帮助页弱化，用户容易写出低精度规则。
- 建议：增加一段 `conditions` 示例，并标注何时优先用 `conditions`。

3. Read-only 导致无法即时验证项目状态
- 边界：该命令无法直接读取用户项目当前规则并做个性化诊断。
- 建议：可增设 `/hookify:doctor`（读规则+运行校验）作为互补命令。

4. 小型内容质量问题
- 风险：`Getting Started` 序号从 2 跳到 4（`plugins/hookify/commands/help.md:167-173`），虽不影响执行但降低专业感。
- 建议：修正编号并保持示例文件命名与 README 一致。
