# FILE `plugins/hookify/agents/conversation-analyzer.md` 研究文档

## 场景与职责

`plugins/hookify/agents/conversation-analyzer.md` 是 Hookify 插件在“无参 `/hookify`”场景下的会话分析规范文件。它不直接执行 Hook，也不直接读写规则文件；职责是把会话中的用户不满/纠正信号提炼为“可落地为 Hook 规则”的候选项。

该文件在插件链路里的角色是“分析输入层”：
- 上游调用场景：`/hookify` 命令在 `$ARGUMENTS` 为空时，要求启动会话分析流程（`plugins/hookify/commands/hookify.md:24-31`）。
- 下游消费场景：分析结果被 `/hookify` 继续用于 `AskUserQuestion` 决策，再写入 `.claude/hookify.{name}.local.md`（`plugins/hookify/commands/hookify.md:62-156`）。
- 运行时生效链路：真正执行 warn/block 的是 `hooks/*.py -> core/config_loader.py -> core/rule_engine.py`，而不是 agent 文件本身（`plugins/hookify/hooks/hooks.json:3-47`，`plugins/hookify/core/rule_engine.py:35-94`）。

## 功能点目的

### 1) 问题信号识别
- 目的：从用户消息中识别“应被防止的行为”。
- 覆盖信号：显式禁止语句、挫败反馈、纠正/回滚、重复问题（`plugins/hookify/agents/conversation-analyzer.md:20-45`）。

### 2) 工具行为归因
- 目的：把“问题描述”关联到工具类型与触发动作，便于生成可执行规则。
- 归因字段：`Which tool`、`What action`、`When`、`Why problematic`（`plugins/hookify/agents/conversation-analyzer.md:47-58`）。

### 3) 模式抽取与正则化
- 目的：将自然语言约束转为可匹配模式，降低手写规则成本。
- 要求给出 Bash/代码/路径模式示例，如 `rm\s+-rf`、`console\.log\(`、`\.env$`（`plugins/hookify/agents/conversation-analyzer.md:60-78`）。

### 4) 严重性分级
- 目的：帮助用户优先治理高风险问题，并影响后续是否建议 block。
- 分级定义：high/medium/low（`plugins/hookify/agents/conversation-analyzer.md:80-93`）。

### 5) 结构化结果输出
- 目的：产出可被 `/hookify` 二次交互消费的统一格式，避免自由文本漂移。
- 输出结构：Issue 分节 + Suggested Rule + Summary（`plugins/hookify/agents/conversation-analyzer.md:95-160`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### A. Agent 协议与配置

该文件是 Claude Code agent 描述文档，frontmatter 定义了运行配置：
- `name: conversation-analyzer`
- `model: inherit`
- `tools: ["Read", "Grep"]`
- `description` 含触发示例（`plugins/hookify/agents/conversation-analyzer.md:1-6`）

这意味着它是“提示协议 + 工具白名单”而非 Python/TS 可执行代码。

### B. 关键流程（从命令到规则）

1. 用户执行 `/hookify`。
2. 若无参数，命令文档要求进行会话分析（`plugins/hookify/commands/hookify.md:24-27`）。
3. 命令示例通过 `Task` 发起子任务并给出分析 prompt（`plugins/hookify/commands/hookify.md:30-58`）。
4. `conversation-analyzer` 规范要求返回结构化问题清单（`plugins/hookify/agents/conversation-analyzer.md:95-160`）。
5. `/hookify` 再通过 `AskUserQuestion` 让用户选择行为、动作（warn/block）与模式（`plugins/hookify/commands/hookify.md:62-80`）。
6. 最终写入 `.claude/hookify.*.local.md` 规则文件（`plugins/hookify/commands/hookify.md:84-156`）。
7. 后续工具事件触发 hooks，动态加载规则并执行（`plugins/hookify/hooks/pretooluse.py:35-57`，`plugins/hookify/core/config_loader.py:198-241`）。

### C. 数据结构映射关系

`conversation-analyzer` 的输出并非 JSON schema，但 Suggested Rule 实际对应 runtime 的 `Rule/Condition`：
- `event` -> `Rule.event`
- `pattern` -> simple pattern（会被转成 `Condition(field, operator='regex_match', pattern)`）
- `message` -> `Rule.message`
- `severity` 当前仅用于建议排序，不直接进入运行时数据结构

对应实现：
- `Rule`/`Condition` 定义：`plugins/hookify/core/config_loader.py:15-43`
- simple `pattern` 转条件：`plugins/hookify/core/config_loader.py:56-73`
- 求值入口：`plugins/hookify/core/rule_engine.py:35-94`

### D. 协议细节与命令耦合点

1. 输入侧协议：
- 来自会话消息历史，agent 需“逆序优先看最近消息”（`plugins/hookify/agents/conversation-analyzer.md:18-19`）。

2. 输出侧协议：
- 需要输出可读的固定段落（Issue / Suggested Rule / Summary），而不是任意文本（`plugins/hookify/agents/conversation-analyzer.md:95-160`）。

3. 与 `/hookify` 的耦合：
- 文本上要求“launch conversation-analyzer”，但 Task 示例 `subagent_type` 为 `general-purpose`（`plugins/hookify/commands/hookify.md:25` 与 `plugins/hookify/commands/hookify.md:33`），存在实现语义偏差风险。

### E. 与 Hook Runtime 的“间接协议”

agent 推荐的规则必须遵守 Hookify DSL，否则运行时不会匹配：
- 规则文件扫描路径固定为 `.claude/hookify.*.local.md`（`plugins/hookify/core/config_loader.py:210-211`）。
- `event` 需与 hook 事件映射一致：Bash->`bash`，Edit/Write/MultiEdit->`file`，Stop->`stop`，Prompt->`prompt`（`plugins/hookify/hooks/pretooluse.py:42-50`，`plugins/hookify/hooks/stop.py:37`，`plugins/hookify/hooks/userpromptsubmit.py:37`）。
- block/warn 的最终协议由 `RuleEngine` 输出格式决定（`plugins/hookify/core/rule_engine.py:60-91`）。

## 关键代码路径与文件引用

### 目标文件
- `plugins/hookify/agents/conversation-analyzer.md`

### 直接调用方（上游）
- `plugins/hookify/commands/hookify.md:24-31`（无参场景要求启动会话分析）
- `plugins/hookify/commands/hookify.md:30-58`（Task 子任务示例）

### 直接被调用方/下游消费者
- `plugins/hookify/commands/hookify.md:62-156`（基于分析结果继续提问并生成规则）

### 运行时关联实现（间接）
- `plugins/hookify/hooks/hooks.json:3-47`（Hook 注册）
- `plugins/hookify/hooks/pretooluse.py:35-70`
- `plugins/hookify/hooks/posttooluse.py:30-62`
- `plugins/hookify/hooks/stop.py:30-55`
- `plugins/hookify/hooks/userpromptsubmit.py:30-54`
- `plugins/hookify/core/config_loader.py:198-275`
- `plugins/hookify/core/rule_engine.py:35-274`

### 配置/文档/样例关联
- `plugins/hookify/README.md:39-52`（`/hookify` 有参/无参说明）
- `plugins/README.md:22`（插件目录索引中声明存在 `conversation-analyzer`）
- `plugins/hookify/skills/writing-rules/SKILL.md:11-99`（规则 DSL 契约）
- `plugins/hookify/examples/*.local.md`（规则样例）

## 依赖与外部交互

### 1) 直接依赖
- 模型：`inherit`（继承当前会话模型）
- 工具：`Read`、`Grep`
- 数据源：当前会话消息历史

### 2) 与插件其他组件交互
- 与命令层交互：由 `/hookify` 触发，并把结果交给命令继续执行交互式创建。
- 与规则层交互：通过 Suggested Rule 字段间接影响 `.claude` 规则文件内容，再由 hooks/core 执行。

### 3) 配置与协议依赖
- 依赖规则文件命名约定：`.claude/hookify.*.local.md`。
- 依赖 event 语义契约：`bash/file/stop/prompt/all`（文档侧见 `README` 与 `SKILL`，实现侧见 `hooks/*.py` + `core/*`）。

### 4) 测试与脚本上下文
- `plugins/hookify` 目录未发现专属 `tests/` 或 `test/` 自动化测试目录。
- agent 文件本身无可执行脚本，仅为提示规范。
- 可复用脚本主要在 `plugins/plugin-dev/skills/hook-development/scripts/`（如 `validate-hook-schema.sh`），用于通用 Hook 配置校验，不直接验证 `conversation-analyzer` 输出质量。

### 5) 外部系统交互
- 无网络调用。
- 无第三方依赖库调用（该 agent 文件为 Markdown 规范）。

## 风险、边界与改进建议

### 1) 调用语义漂移风险（高）
- 现状：命令写明“launch conversation-analyzer”，但 Task 示例给 `subagent_type: general-purpose`。
- 影响：可能无法稳定命中专用 agent 风格，输出格式一致性下降。
- 建议：显式绑定 agent 名称（若平台支持），或至少在 Task prompt 中加入严格 JSON schema 要求并校验字段。

### 2) 输出为半结构化文本，机器可消费性弱（中）
- 现状：输出模板是 Markdown 段落，不是严格结构化数据。
- 影响：下游若希望自动化生成规则，容易受格式波动影响。
- 建议：增加可选 JSON 输出模式（如 `issues[]` + `suggested_rule`），同时保留可读文本版本。

### 3) 严重级别与动作映射未闭环（中）
- 现状：agent 做了 severity 分类，但最终 action 仍靠后续人工选择。
- 影响：高危行为可能被配置为 warn，降低防护强度。
- 建议：在 `/hookify` 里默认推荐映射（high->block，medium->warn），用户可覆盖。

### 4) 正则建议与 runtime 操作符语义可能混淆（中）
- 现状：agent鼓励正则表达式；但运行时部分 operator（如 `contains/not_contains`）是字面字符串逻辑。
- 影响：用户误以为所有操作符都按 regex 解释，产生误配。
- 建议：在 agent 输出中标注“建议使用 `regex_match` 才按正则解析”，并区分 literal vs regex。

### 5) 误报控制依赖提示工程，缺少回归验证（中）
- 现状：文档强调“不要把假设讨论/教学内容当问题”，但没有样本集测试。
- 影响：提示词改动易引入误报且不易察觉。
- 建议：补充最小评测集（正例/反例/边界例）并固化回归检查。

### 6) 目录职责清晰但可观测性不足（低）
- 现状：分析结果仅在会话中出现，不持久化。
- 影响：难以审计“为何某条规则被建议创建”。
- 建议：可选保存分析摘要到 `.claude/hookify.analysis.local.md` 供后续审阅。
