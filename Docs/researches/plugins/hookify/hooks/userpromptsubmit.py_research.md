# FILE `plugins/hookify/hooks/userpromptsubmit.py` 研究文档

## 场景与职责

`userpromptsubmit.py` 是 Hookify 在 `UserPromptSubmit` 事件上的执行器，用户每次提交新提示词时触发，用于在“任务开始前”做提示词级规则检查与提醒。

它承担的是会话输入侧治理能力（prompt guardrail），而不是工具调用治理。

## 功能点目的

1. 接入 `UserPromptSubmit` 生命周期事件。
2. 仅加载 `event='prompt'` 的规则，提升匹配准确性。
3. 利用统一规则引擎输出提示信息，复用同一套规则 DSL。
4. 保持 fail-open，确保提示词检查失败不阻断系统主流程。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 路由与导入

- `hooks.json` 将 `UserPromptSubmit` 路由到本脚本（`hooks.json:37-46`）。
- 启动后先注入 `sys.path`（`userpromptsubmit.py:13-19`），再导入 `load_rules/RuleEngine`（`userpromptsubmit.py:22-23`）。

### 2) 执行流程

1. `input_data = json.load(sys.stdin)`（`userpromptsubmit.py:34`）。
2. 固定加载 prompt 规则：`load_rules(event='prompt')`（`userpromptsubmit.py:37`）。
3. `RuleEngine.evaluate_rules(rules, input_data)`（`userpromptsubmit.py:40-41`）。
4. 输出 JSON（`userpromptsubmit.py:44`）。
5. `finally` 强制 `exit 0`（`userpromptsubmit.py:52-54`）。

### 3) Prompt 字段抽取

规则引擎在 `_extract_field` 中支持 `field == 'user_prompt'`，从 `input_data.user_prompt` 读取文本（`rule_engine.py:226-228`）。

因此 prompt 规则可以基于用户原始输入做 regex/contains 等条件匹配。

### 4) 协议语义

引擎对 `hook_event_name` 非 `Stop/PreToolUse/PostToolUse` 的 block 分支仅返回 `systemMessage`（`rule_engine.py:80-84`）。

这意味着本事件即便规则 `action: block`，当前实现也主要体现为“提醒信息”，不输出显式 `decision:block`。

## 关键代码路径与文件引用

- 目标文件：`plugins/hookify/hooks/userpromptsubmit.py:1-58`
- 路由配置：`plugins/hookify/hooks/hooks.json:37-46`
- Prompt 字段读取：`plugins/hookify/core/rule_engine.py:226-228`
- 通用决策流程：`plugins/hookify/core/rule_engine.py:35-94`
- 规则来源与事件说明：`plugins/hookify/README.md:122-128,255-257`

## 依赖与外部交互

1. Python 依赖：`os/sys/json`。
2. 内部依赖：`hookify.core.config_loader` 与 `hookify.core.rule_engine`。
3. 环境依赖：`CLAUDE_PLUGIN_ROOT`。
4. 输入交互：stdin JSON 的 `user_prompt` 字段。
5. 输出交互：stdout JSON `systemMessage`（以及可能空对象 `{}`）。

## 风险、边界与改进建议

1. `action: block` 在 UserPromptSubmit 的语义不闭环。
- 风险：规则作者会认为可“阻止用户提示词提交”，但当前仅回传消息。
- 建议：为 `UserPromptSubmit` 增加明确 block 协议分支，或在文档中声明“当前仅提醒”。

2. 无 prompt 专属错误上下文。
- 风险：异常只返回通用 `Hookify error`，难快速判断是输入解析、规则加载还是条件匹配问题。
- 建议：返回结构化错误元数据（例如 `event: UserPromptSubmit`、`stage: load_rules`）。

3. 与其他执行器重复代码较多。
- 风险：维护时容易出现单脚本行为漂移。
- 建议：抽公共函数统一处理 stdin、路径注入和异常模板。

4. 规则可见性问题。
- 风险：用户可能不知道 prompt 规则是按 `.claude/hookify.*.local.md` 动态加载。
- 建议：在 `/hookify:help` 增加 prompt 事件专门段落与可复现示例。
