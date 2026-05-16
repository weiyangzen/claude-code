# plugins/plugin-dev/agents/agent-creator.md 研究

## 场景与职责

`agent-creator` 是 `plugin-dev` 插件里的“agent 生成器”子代理，职责是把用户对新 agent 的自然语言需求收敛成可落地的 agent 配置文件（`agents/[identifier].md`）。

它在整体工作流中的位置是“实现阶段的 agent 产物生成器”：
- `/plugin-dev:create-plugin` 在 Phase 5 的 Agents 子流程明确要求“对每个 agent 使用 agent-creator 生成配置并落盘”（`plugins/plugin-dev/commands/create-plugin.md:192-200`）。
- `plugin-dev` README 将其列为三大专用 agents 之一，和 `plugin-validator`、`skill-reviewer` 组成“生成-校验-评审”闭环（`plugins/plugin-dev/README.md:30-40`、`plugins/plugin-dev/README.md:154-173`）。

能力边界：
- 该 agent 负责“设计并生成 agent 配置”，不是执行目标业务任务。
- 其工具权限仅 `Read + Write`（`plugins/plugin-dev/agents/agent-creator.md:34`），符合生成型角色，但不承担跨仓库深度检索。

## 功能点目的

1. 把模糊需求结构化
- 通过“意图提炼 -> 专家人设 -> 系统提示词架构 -> 标识符命名 -> 触发样例”链路，将“想要什么 agent”转成明确规格（`plugins/plugin-dev/agents/agent-creator.md:41-74`）。

2. 统一产物协议
- 强制输出包含 `name/description/model/color/tools` 的 frontmatter + 完整系统提示词，避免团队内 agent 文件质量漂移（`plugins/plugin-dev/agents/agent-creator.md:112-123`）。

3. 内建触发质量保障
- 要求 `description` 包含 2-4 个 `<example>`，覆盖显式与主动触发，减少自动触发误判（`plugins/plugin-dev/agents/agent-creator.md:68-73`、`82-93`）。

4. 对齐项目上下文
- 明确要求读取 `CLAUDE.md` 等项目规范，减少生成结果与仓库约定冲突（`plugins/plugin-dev/agents/agent-creator.md:39`、`43`、`53`）。

5. 输出后续验证建议
- 在产物说明中引导调用校验能力（`validate-agent.sh` 与 `plugin-validator`），将一次性生成接入质量门（`plugins/plugin-dev/agents/agent-creator.md:130`、`162`）。

## 具体技术实现（关键流程/数据结构/协议/命令）

### 1) 元数据与运行时协议
该文件采用 Claude Code agent 标准协议：`YAML frontmatter + markdown system prompt`。
- `name: agent-creator`（agent 标识）
- `description: Use this agent when...`（触发条件+示例）
- `model: sonnet`（固定模型）
- `color: magenta`
- `tools: ["Write", "Read"]`
见 `plugins/plugin-dev/agents/agent-creator.md:1-35`。

### 2) 关键流程（Process Contract）
核心实现是文本协议驱动的 5 步生成流程：
1. 理解需求（`Understand Request`）
2. 设计配置（identifier/description/examples/system prompt）
3. 选择配置（model/color/tools）
4. 生成文件（写入 `agents/[identifier].md`）
5. 给用户摘要与测试建议
见 `plugins/plugin-dev/agents/agent-creator.md:75-131`。

### 3) 隐式数据结构
虽然不是 JSON schema，但输出契约稳定：
- 输入：用户任务描述（自由文本）
- 中间结构：`identifier` + `description(with examples)` + `system prompt`
- 输出：agent markdown 文件 + 人类可读摘要（`## Agent Created` 模板）
见 `plugins/plugin-dev/agents/agent-creator.md:145-165`。

### 4) 关键命令与脚本耦合
文档中约定的校验命令为：
- `scripts/validate-agent.sh agents/[identifier].md`（`plugins/plugin-dev/agents/agent-creator.md:162`）

而 `plugin-dev` 内实际脚本位于：
- `plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh`（`plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:1-217`）

说明：该 agent 输出的是“目标插件内建议命令”，要求目标插件自行具备对应脚本或替换为可执行路径。

## 关键代码路径与文件引用

- 当前研究对象：`plugins/plugin-dev/agents/agent-creator.md:1-176`
- 主调用方：`plugins/plugin-dev/commands/create-plugin.md:192-200`
- 流程文档入口：`plugins/plugin-dev/README.md:30-40`、`154-173`
- 规则来源（同构提示词）：
  - `plugins/plugin-dev/skills/agent-development/SKILL.md:217-247`
  - `plugins/plugin-dev/skills/agent-development/references/agent-creation-system-prompt.md:7-71`
- 产物校验脚本：`plugins/plugin-dev/skills/agent-development/scripts/validate-agent.sh:1-217`

## 依赖与外部交互

1. 内部依赖
- 依赖 `/plugin-dev:create-plugin` 在 Phase 5 调用它进行 agent 生成（`plugins/plugin-dev/commands/create-plugin.md:192-199`）。
- 依赖 `agent-development` 技能中的 frontmatter/提示词约定来保持一致性（`plugins/plugin-dev/skills/agent-development/SKILL.md:60-197`）。

2. 工具与文件系统交互
- 通过 `Write` 写入 `agents/*.md`，通过 `Read`读取项目上下文。
- 无 `Bash/Grep/Glob` 权限，默认不做脚本级自动校验、批量扫描。

3. 外部交互
- 不直接调用外部服务。
- 主要与 Claude Code 运行时交互（agent 自动触发与任务编排）。

## 风险、边界与改进建议

1. 风险：模型策略不一致
- frontmatter 固定 `model: sonnet`（`plugins/plugin-dev/agents/agent-creator.md:32`），但正文又建议“默认用 inherit”（`103` 行）。这会导致“生成器自身配置”和“它建议生成的配置”存在认知分叉。

2. 风险：校验命令路径假设过强
- 输出模板写 `scripts/validate-agent.sh`（`162` 行），但很多目标插件默认并不存在该脚本；若不额外说明，用户会按模板执行失败。

3. 风险：格式残留
- 文件末尾存在孤立代码围栏收尾（`174` 行）与会话式总结句（`176` 行），不属于核心系统提示协议，可能干扰后续自动处理。

4. 风险：工具能力偏窄
- 缺少 `Grep/Glob`，对于“命名冲突检测、已有 agent 复用评估”需要额外上下文输入，否则只能靠用户描述与单文件读取。

5. 边界
- 该 agent 只负责“生成配置草案+文件”，不负责运行验证脚本、测试触发行为，也不负责插件全量合规判断（这属于 `plugin-validator`）。

6. 改进建议
- 统一模型策略：将“生成器自身 model”改为 `inherit`，或在正文明确为何固定 sonnet。
- 输出模板加入“脚本不存在时的替代命令”说明（例如指向 `plugin-dev` 的脚本路径或手工校验清单）。
- 清理尾部无关文本与多余围栏，保持文件是纯协议内容。
- 如要提升生成质量，可增补可选只读检索工具（`Grep/Glob`）用于冲突检测。
