# DIR `plugins/hookify/agents` 研究文档

## 场景与职责

`plugins/hookify/agents` 当前是 Hookify 插件的“会话问题发现层”，目录内仅有一个 agent 定义文件：`plugins/hookify/agents/conversation-analyzer.md`。

该目录职责不是直接执行 Hook，也不直接读写 `.claude/hookify.*.local.md` 规则文件；它负责在“用户没有给出明确规则文本”时，从对话历史中识别可治理行为，并输出结构化问题清单，供 `/hookify` 命令后续转化为规则文件。

在插件上下文中的位置：
- 上游调用方：`/hookify` 命令流程（无参数分支）要求启动该分析 agent。
- 下游消费者：`/hookify` 命令本体根据分析结果发起 AskUserQuestion，并最终创建 `.claude/hookify.{name}.local.md` 文件。
- 运行期关联模块：规则真正生效依赖 `hooks/*.py` + `core/*.py`，而不是 agent 本身。

关键定位参考：
- `plugins/hookify/agents/conversation-analyzer.md:1`
- `plugins/hookify/commands/hookify.md:24`
- `plugins/hookify/hooks/hooks.json:1`
- `plugins/hookify/core/rule_engine.py:35`

## 功能点目的

### 1) 会话问题挖掘（Conversation Mining）
目的：从最近用户消息中提取“可转化为 Hook 规则”的行为约束信号，而不只依赖用户一次性明确描述。

对应定义：
- 显式禁止语句（如 “Don't.../Stop...”）
- 用户不满反馈
- 用户回滚/纠正 Claude 行为
- 重复出现的问题模式

参考：`plugins/hookify/agents/conversation-analyzer.md:20`

### 2) 可执行模式提取（Pattern Extraction）
目的：把自然语言不满/约束，转成可被 Hookify 规则引擎执行的模式元素：
- 工具类型（Bash/Edit/Write/MultiEdit 等）
- 匹配模式（regex 或字面模式）
- 严重级别（high/medium/low）
- 建议 rule 字段（name/event/pattern/message）

参考：
- `plugins/hookify/agents/conversation-analyzer.md:47`
- `plugins/hookify/agents/conversation-analyzer.md:60`
- `plugins/hookify/agents/conversation-analyzer.md:79`

### 3) 为 `/hookify` 规则创建流程提供输入
目的：在 `/hookify` 无参数时，补齐规则候选的来源，降低用户手写 frontmatter 的成本。

调用流程要求：
- `hookify.md` 明确要求“无参数时启动 conversation analysis”。
- 结果进入 AskUserQuestion 阶段，由用户选择要落地的行为与动作（warn/block）。

参考：
- `plugins/hookify/commands/hookify.md:24`
- `plugins/hookify/commands/hookify.md:62`
- `plugins/hookify/commands/hookify.md:84`

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. Agent 定义结构
`conversation-analyzer.md` 是标准 agent 描述文件，frontmatter 定义了运行属性：
- `name: conversation-analyzer`
- `model: inherit`
- `tools: ["Read", "Grep"]`
- 任务说明：会话分析、模式提取、严重级别归类、结构化输出

参考：`plugins/hookify/agents/conversation-analyzer.md:1`

### B. 关键流程（命令到规则）
1. 用户执行 `/hookify`。
2. 若 `$ARGUMENTS` 为空，命令说明要求发起会话分析子任务。
3. 分析结果按结构化问题列表呈现给用户（AskUserQuestion）。
4. 用户确认后，命令生成 `.claude/hookify.{rule-name}.local.md`。
5. 后续工具调用触发 Hook 脚本，动态加载规则并执行 warn/block。

参考：
- `plugins/hookify/commands/hookify.md:17`
- `plugins/hookify/commands/hookify.md:30`
- `plugins/hookify/commands/hookify.md:126`
- `plugins/hookify/hooks/pretooluse.py:35`
- `plugins/hookify/core/config_loader.py:198`

### C. Agent 输出契约（结构化文本模板）
agent 要求输出固定结构，包括：
- `Issue` 标题
- `Severity` / `Tool` / `Pattern` / `Occurrences` / `Context` / `User Reaction`
- `Suggested Rule`（name/event/pattern/message）
- 最终 `Summary`

这是“人可读 + 便于命令后续提问”的半结构化协议，而非 JSON schema。

参考：`plugins/hookify/agents/conversation-analyzer.md:95`

### D. 与规则引擎字段模型的映射关系
agent 推荐的字段最终要能映射到 Hookify Rule 数据结构：
- `event` -> `Rule.event`
- `pattern` -> simple pattern（会被转为 `Condition`）
- `action` -> `Rule.action`
- message body -> `Rule.message`

Rule 结构与条件匹配逻辑由 `core` 实现，不在 agents 目录内：
- `Rule/Condition`：`plugins/hookify/core/config_loader.py:15`
- 规则评估：`plugins/hookify/core/rule_engine.py:35`

### E. 关键命令与协议样式
1. `/hookify` 内部给出的 Task 调用示例（用于会话分析）：
```json
{
  "subagent_type": "general-purpose",
  "description": "Analyze conversation for unwanted behaviors",
  "prompt": "..."
}
```
来源：`plugins/hookify/commands/hookify.md:30`

2. 规则生效协议（由 hooks/core 返回）：
- warning: `{ "systemMessage": "..." }`
- block(Pre/PostToolUse): `hookSpecificOutput.permissionDecision = deny`
- block(Stop): `decision = block`

来源：`plugins/hookify/core/rule_engine.py:60`

### F. 测试/脚本现状（与该目录相关）
- `plugins/hookify/agents` 目录内无自动化测试、无可执行脚本。
- 插件层也未见专属 `tests/`；仅有 `core/*.py` 文件内 `__main__` 手工测试片段。
- 可借助外部通用脚本（`plugins/plugin-dev/skills/hook-development/scripts/*`）做 hook 层面验证，但这不覆盖 agent 文本输出质量。

参考：
- `plugins/hookify/core/config_loader.py:277`
- `plugins/hookify/core/rule_engine.py:276`

## 关键代码路径与文件引用

目标目录：
- `plugins/hookify/agents/conversation-analyzer.md`

直接调用方（上游）：
- `plugins/hookify/commands/hookify.md`

规则生成与管理上下文：
- `plugins/hookify/commands/list.md`
- `plugins/hookify/commands/configure.md`
- `plugins/hookify/commands/help.md`
- `plugins/hookify/skills/writing-rules/SKILL.md`

运行期执行链（被间接依赖）：
- `plugins/hookify/hooks/hooks.json`
- `plugins/hookify/hooks/pretooluse.py`
- `plugins/hookify/hooks/posttooluse.py`
- `plugins/hookify/hooks/stop.py`
- `plugins/hookify/hooks/userpromptsubmit.py`
- `plugins/hookify/core/config_loader.py`
- `plugins/hookify/core/rule_engine.py`

插件注册与文档：
- `plugins/hookify/.claude-plugin/plugin.json`
- `.claude-plugin/marketplace.json`
- `plugins/README.md`
- `plugins/hookify/README.md`

规则样例（用于验证 agent 推荐的可实现性）：
- `plugins/hookify/examples/dangerous-rm.local.md`
- `plugins/hookify/examples/console-log-warning.local.md`
- `plugins/hookify/examples/sensitive-files-warning.local.md`
- `plugins/hookify/examples/require-tests-stop.local.md`

## 依赖与外部交互

### 1) 目录内 agent 的直接依赖
- 工具依赖：`Read`、`Grep`（frontmatter 中声明）。
- 模型依赖：`model: inherit`，跟随调用会话配置。
- 输入依赖：当前会话消息历史（尤其用户消息）。

参考：`plugins/hookify/agents/conversation-analyzer.md:4`

### 2) 与外部模块交互
- 与 `/hookify` 命令交互：作为会话分析子步骤，被命令流程消费。
- 与 `writing-rules` skill 的关系：agent给出行为模式，skill定义规则文件语法与命名规范。
- 与 hooks/runtime 的关系：agent不参与执行，仅影响规则“生成内容质量”。

参考：
- `plugins/hookify/commands/hookify.md:9`
- `plugins/hookify/skills/writing-rules/SKILL.md:11`
- `plugins/hookify/hooks/hooks.json:3`

### 3) 配置与文件系统交互边界
- agent 本身不直接读写项目文件（规则写入由 `/hookify` 命令执行）。
- 规则运行期读取 `.claude/hookify.*.local.md`，因此 agent 推荐必须与该格式兼容。

参考：
- `plugins/hookify/commands/hookify.md:84`
- `plugins/hookify/core/config_loader.py:210`

### 4) 测试与脚本依赖
- 当前目录无测试脚本。
- 研究/流程层脚本位于仓库 `.ops/`，用于 checklist/todo 维护，不影响 agent 运行逻辑。

## 风险、边界与改进建议

1. 调用描述与示例参数存在漂移风险。
- 现状：`hookify.md` 文本要求“启动 conversation-analyzer agent”，但 Task 示例给的是 `subagent_type: general-purpose`。
- 风险：在实现或迁移时，可能没有稳定命中该专用 agent 定义，导致输出风格不一致。
- 建议：在命令文档中显式绑定 agent 名称（若平台支持），或在 prompt 中加入更强约束并附输出 schema。

2. 输出协议是“半结构化文本”，机器可消费性有限。
- 现状：agent 输出模板为 markdown 文本，不是 JSON/YAML 结构。
- 风险：下游若做自动解析，容易因格式漂移失败。
- 建议：新增可选 JSON 输出模式，至少包含 `issues[]` + `suggested_rule` 字段。

3. 严重级别到动作（warn/block）的映射未强约束。
- 现状：agent 有 severity 分类，但最终 action 由后续交互决定。
- 风险：高危行为可能被弱化为 warn。
- 建议：在 `/hookify` 交互里给默认推荐映射（high->block，medium->warn），用户可覆盖。

4. 缺少 agent 层测试与评估基线。
- 现状：无会话样本集、无 golden output、无回归测试。
- 风险：文案调整会引入召回率/误报率退化但难感知。
- 建议：新增最小评测资产：
  - 正例：显式“不要做 X”语句
  - 反例：假设讨论/教学语境
  - 边界例：一次性失误 vs 重复问题

5. “不误报”规则虽有文字说明，但缺乏下游校验闭环。
- 现状：agent 提醒“不要把假设讨论当问题”，但未与规则引擎层做自动防误配检查。
- 建议：在规则生成阶段加一次 pattern lint（例如检测过宽正则、空 pattern、过多 `.*`）。

6. 目录职责边界清晰但可观测性不足。
- 现状：agent 分析结果仅体现在会话，不持久化分析元数据。
- 建议：可选输出“分析摘要文件”到 `.claude/`（如 `hookify.analysis.local.md`），方便审计和迭代规则。

