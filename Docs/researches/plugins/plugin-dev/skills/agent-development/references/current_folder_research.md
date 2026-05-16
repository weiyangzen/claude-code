# plugins/plugin-dev/skills/agent-development/references 研究

## 场景与职责

`plugins/plugin-dev/skills/agent-development/references` 是 `agent-development` 技能的“深度规则库”，承接 `SKILL.md` 的渐进式披露设计：

- `SKILL.md` 保留核心流程，`references/` 承载高密度细则（`plugins/plugin-dev/skills/agent-development/SKILL.md:377-385`）。
- 目录内 3 个文件分别负责：
  - agent 生成系统提示词基线：`agent-creation-system-prompt.md`（`.../references/agent-creation-system-prompt.md:7-71`）
  - system prompt 架构模式库：`system-prompt-design.md`（`.../references/system-prompt-design.md:5-230`）
  - 触发样例编排规范：`triggering-examples.md`（`.../references/triggering-examples.md:5-491`）

该目录的职责不是执行逻辑，而是定义“文档协议”：

1. 约束 AI 辅助 agent 生成输出结构（JSON 三元组）。
2. 约束 agent 的触发描述写法（`<example>`/`<commentary>`）。
3. 约束 system prompt 的结构化质量标准。
4. 为 `agent-creator`、`create-plugin`、`plugin-validator` 等上层流程提供一致知识源。

## 功能点目的

### 1. 统一 AI 生成契约

`agent-creation-system-prompt.md` 明确输出必须是且仅是 JSON，字段固定为 `identifier/whenToUse/systemPrompt`（`.../agent-creation-system-prompt.md:55-60`），目的：

- 把自然语言需求收敛到稳定的数据结构。
- 降低“生成格式漂移”导致的落盘失败。
- 让后续 frontmatter 映射可脚本化（`.../agent-creation-system-prompt.md:91-120`）。

### 2. 提供可复用的 Prompt 模板体系

`system-prompt-design.md` 给出通用骨架与四类专用模式（Analysis/Generation/Validation/Orchestration）（`.../references/system-prompt-design.md:40-230`），目的：

- 让 agent 从“角色描述”升级为“可执行流程”。
- 统一质量标准、输出格式、边界处理。

### 3. 提升触发命中率与可解释性

`triggering-examples.md` 规定 `<example>` 的结构、示例类型（显式/主动/隐式/工具模式）和数量建议（`.../references/triggering-examples.md:125-189,325-344`），目的：

- 提升触发条件的可学习性与覆盖面。
- 让“为什么触发”通过 `<commentary>` 可审计（`.../references/triggering-examples.md:80-105`）。

### 4. 作为 plugin-dev 工具链的知识上游

该目录被 `agent-development/SKILL.md`、`README` 作为官方参考资源列出（`plugins/plugin-dev/skills/agent-development/SKILL.md:383-385`，`plugins/plugin-dev/README.md:167-171`），支撑 `/plugin-dev:create-plugin` Phase 5/6 的 agent 生成与校验步骤（`plugins/plugin-dev/commands/create-plugin.md:193-200,252-256`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 关键流程 A：AI 辅助创建（references 驱动）

1. 用户给出 agent 需求。
2. 使用 `agent-creation-system-prompt` 约束模型输出 JSON 三元组（`.../agent-creation-system-prompt.md:55-60`）。
3. 将 JSON 映射为 `agents/[identifier].md`（frontmatter + system prompt）（`.../agent-creation-system-prompt.md:91-120`）。
4. 用 `validate-agent.sh` 做结构与质量检查（`plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:7-17,51-203`）。
5. 在真实场景下验证触发命中（依据 `triggering-examples` 的显式/主动样例策略）。

该流程在 `agent-creator` 中被几乎原样内嵌（`plugins/plugin-dev/agents/agent-creator.md:37-123`）。

### 关键流程 B：手工编写 agent（references 校准）

1. 按 `SKILL.md` 写 frontmatter 与 prompt 主体（`plugins/plugin-dev/skills/agent-development/SKILL.md:20-58,60-160`）。
2. 用 `system-prompt-design` 对齐职责、流程、输出、边界四大块（`.../references/system-prompt-design.md:5-38`）。
3. 用 `triggering-examples` 补齐 2-4 个触发场景并覆盖不同措辞（`.../references/triggering-examples.md:191-230,325-344`）。
4. 执行脚本校验并场景化回归。

### 数据结构与“软协议”

1. JSON 输出契约：
   - `identifier`: 低耦合 agent 标识（kebab-case）。
   - `whenToUse`: 必须含 `Use this agent when...` 与 `<example>`。
   - `systemPrompt`: 第二人称、结构化、可执行。
   依据：`.../agent-creation-system-prompt.md:55-60`。

2. description 触发协议：
   - `<example>` 包含 `Context/user/assistant/commentary/assistant`。
   - `commentary` 必须解释触发决策逻辑。
   依据：`.../triggering-examples.md:9-19,80-121`。

3. system prompt 结构协议：
   - Role → Responsibilities → Process → Quality Standards → Output Format → Edge Cases。
   依据：`.../system-prompt-design.md:9-38`。

### 命令与脚本

references 目录本身无可执行脚本，实际执行依赖外层工具：

```bash
bash plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh <agent-file>
```

被 `create-plugin` 与 `plugin-validator` 流程引用（`plugins/plugin-dev/commands/create-plugin.md:199,255`，`plugins/plugin-dev/agents/plugin-validator.md:86-96`）。

## 关键代码路径与文件引用

### 目标目录核心文件

- `plugins/plugin-dev/skills/agent-development/references/agent-creation-system-prompt.md`
  - 生成提示词主体：`7-71`
  - JSON 契约：`55-60`
  - JSON→agent 文件映射：`91-120`
  - plugin-dev 集成步骤：`195-205`

- `plugins/plugin-dev/skills/agent-development/references/system-prompt-design.md`
  - 核心结构模板：`5-38`
  - 四类模式模板：`40-230`
  - 可执行指令与常见陷阱：`261-337`
  - 长度/测试建议：`338-401`

- `plugins/plugin-dev/skills/agent-development/references/triggering-examples.md`
  - `<example>` 标准格式：`9-19`
  - 类型策略（显式/主动/隐式/工具模式）：`125-189`
  - 示例数量策略：`325-344`
  - 排错与最佳实践：`439-488`

### 上下游依赖路径（调用方/被调用方）

- 调用方：
  - `plugins/plugin-dev/skills/agent-development/SKILL.md:383-385`（将 references 作为权威参考入口）
  - `plugins/plugin-dev/README.md:167-171`（对外宣告该技能资源组成）
  - `plugins/plugin-dev/commands/create-plugin.md:160,193-200`（要求加载 agent-development 并生成 agent）

- 被调用方/实现承接：
  - `plugins/plugin-dev/agents/agent-creator.md:37-123`（复用 reference 中的生成契约和流程）
  - `plugins/plugin-dev/skills/agent-development/examples/agent-creation-prompt.md:17-53`（把 reference 方法变成可执行模板）
  - `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:51-203`（承接结构校验）
  - `plugins/plugin-dev/agents/plugin-validator.md:86-96`（在插件验收阶段调用校验规则）

## 依赖与外部交互

### 内部依赖

1. 文档依赖：
- 依赖 `agent-development/SKILL.md` 作为入口与导航。
- 依赖 `examples/` 将参考规范转为落地样板。

2. 脚本依赖：
- references 不执行脚本，但其规则最终由 `validate-agent.sh` 承接并验证。

3. 配置依赖：
- references 规范直接映射到 agent frontmatter（`name/description/model/color/tools`）与 system prompt 文本。

### 外部交互与运行时边界

1. 依赖 Claude Code 的 agent 自动发现与触发机制（`agents/*.md` + description 语义匹配），目录本身不实现触发算法。
2. 依赖项目上下文（如 `CLAUDE.md`）进入生成过程（`.../agent-creation-system-prompt.md:10,24,156-159`）。
3. 无网络调用、无二进制协议；主要是 Markdown 文档协议 + Shell 校验命令。

### 测试现状

1. references 目录无自动化测试文件。
2. 当前验证方式为：
- 静态检查：`validate-agent.sh`
- 行为检查：按 `triggering-examples` 进行场景手测
3. `SKILL.md` 提及 `test-agent-trigger.sh`，但 `scripts/` 当前仅有 `validate-agent.sh`（`plugins/plugin-dev/skills/agent-development/SKILL.md:398-400`；`plugins/plugin-dev/skills/agent-development/scripts/` 实际目录）。

## 风险、边界与改进建议

### 风险 1（高）：规范多源复制，易分叉

- `agent-creation-system-prompt.md` 与 `agents/agent-creator.md` 存在高重复提示词片段（`.../references/agent-creation-system-prompt.md:7-70` vs `plugins/plugin-dev/agents/agent-creator.md:37-74`）。
- 任一处更新后若另一处未同步，会导致“文档建议”和“实际代理行为”不一致。

建议：提取单一权威模板源，其他文件引用或自动生成。

### 风险 2（高）：reference 协议与校验器能力不完全匹配

- references 要求多行 description + 多个 `<example>`；
- 现有 `validate-agent.sh` 对 description 的读取方式偏首行匹配（见脚本实现），容易误报“缺少 `<example>`”。

建议：增强 frontmatter 多行解析，或引入 YAML 解析器，避免 reference 与工具链之间的“假失败”。

### 风险 3（中）：触发测试能力文档与脚本清单不一致

- 文档写有 `test-agent-trigger.sh`，实际不存在。

建议：二选一：
1. 补齐脚本并纳入 SKILL/README。
2. 删除无效引用，改成明确的手测步骤模板。

### 风险 4（中）：命令路径语义不统一

- examples 中出现 `./scripts/validate-agent.sh`（相对 skill 目录）；
- 从仓库根目录执行时应使用 `bash plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh ...`。

建议：在 reference 或 examples 中同时给出两种执行上下文的命令写法。

### 边界说明

1. 本目录是“规范层”，不直接执行 agent 调度或质量门禁。
2. 它提供的是经验化模式，不是强类型 schema；最终正确性仍依赖脚本与实测。
3. 不覆盖插件运行时协议（hooks/MCP/命令执行）的实现细节，仅与 agent 设计相关。

### 改进优先级建议

1. 先统一“权威提示词源”并减少重复维护。
2. 修复校验器对多行 description 的解析，保证 references 可被正确验证。
3. 明确触发测试脚本去留并更新文档。
4. 增加一份 machine-readable 契约（如 JSON Schema）描述 `identifier/whenToUse/systemPrompt`，降低解释歧义。
