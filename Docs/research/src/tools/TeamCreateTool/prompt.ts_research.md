# prompt.ts 研究文档

## 场景与职责

`prompt.ts` 是 `TeamCreateTool` 的模型提示词（system prompt 片段）工厂模块。它通过 `getPrompt()` 函数返回一段 Markdown 格式的文本，向 Claude 解释：

1. 何时应该主动调用 `TeamCreate`；
2. 创建团队后会生成什么文件和目录；
3. 团队工作流的完整生命周期（创建 → 任务 → 招募 → 分配 → 完成 → 解散）；
4. 与队友协作时的通信规范、任务所有权规则、自动消息投递机制等。

这段 prompt 不是面向用户的文档，而是直接注入到模型的 system prompt 中，影响模型的工具选择和行为策略。

## 功能点目的

- **触发时机教育**：明确告诉模型，只要用户提到"team"、"swarm"、"agents work together"，或者任务复杂到适合并行处理，就应该主动调用 `TeamCreate`。
- **降低使用门槛**：通过给出具体的 JSON 示例和文件路径约定，让模型知道调用参数怎么写、团队配置存在哪里。
- **规范协作行为**：强调多项关键行为约束，例如：
  - 消息自动投递，不需要手动查收件箱；
  - 队友 idle 是正常状态，不要过度反应；
  - 用队友的 `name` 而不是 `agentId` 进行通信和任务分配；
  - 不要用终端工具偷看队友活动，要用 `SendMessage`；
  - 不要用 JSON 状态消息，要用自然语言沟通。

## 具体技术实现

### 代码结构

```ts
export function getPrompt(): string {
  return `
# TeamCreate

## When to Use
...
`.trim()
}
```

- 使用模板字符串包裹一段长 Markdown。
- 末尾调用 `.trim()` 去除首尾空白，保证注入到 system prompt 时格式整洁。
- 无动态插值，返回的是静态字符串，但每次调用会重新构造字符串对象。

### Prompt 内容结构拆解

| 章节 | 核心内容 |
|------|----------|
| **When to Use** | 触发条件：用户明确要求团队/群体协作，或任务复杂到适合并行。强调"When in doubt, prefer spawning a team"。 |
| **Choosing Agent Types for Teammates** | 教育模型如何为队友选择 `subagent_type`：只读代理（Explore/Plan）不能做实现工作；全功能代理才能编辑文件；自定义代理要看描述。 |
| **Team = TaskList** | 说明团队与任务列表 1:1 对应，给出 JSON 示例和生成的文件路径（`~/.claude/teams/{team-name}/config.json`、`~/.claude/tasks/{team-name}/`）。 |
| **Team Workflow** | 7 步工作流：Create → TaskCreate → Spawn teammates → Assign → Work → Idle → Shutdown。 |
| **Task Ownership** | 任何代理都能用 `TaskUpdate` 的 `owner` 参数分配任务所有权。 |
| **Automatic Message Delivery** | **IMPORTANT** 级别强调：队友消息自动投递，不需要手动检查收件箱；忙碌时消息会排队。 |
| **Teammate Idle State** | 解释 idle 是正常的等待状态，idle 队友可以接收消息；peer DM 的摘要可见性规则。 |
| **Discovering Team Members** | 教模型通过 `Read` 工具读取 `~/.claude/teams/{team-name}/config.json` 来发现成员；强调始终使用 `name` 进行通信和任务分配。 |
| **Task List Coordination** | 6 条任务协作规范：定期检查 TaskList、按 ID 顺序优先认领、创建新任务、标记完成、协调阻塞、通知团队领导。 |
| **Communication Rules** | 4 条通信铁律：不用终端偷看、必须用 `SendMessage`、不要发结构化 JSON 状态、用 `TaskUpdate` 标记完成。 |

### 关键行为约束（模型指令）

- **优先创建团队**："When in doubt about whether a task warrants a team, prefer spawning a team."
- **自动消息投递**："Messages from teammates are automatically delivered to you. You do NOT need to manually check your inbox."
- **Idle 正常化**："Teammates go idle after every turn—this is completely normal and expected."
- **名称引用规范**："Always refer to teammates by their NAME... Names are used for: `to` when sending messages; Identifying task owners"
- **禁止 JSON 状态消息**："Do NOT send structured JSON status messages like `{"type":"idle",...}`"

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/tools/TeamCreateTool/prompt.ts` | 本文件，提供 `getPrompt()` |
| `src/tools/TeamCreateTool/TeamCreateTool.ts` | 在 `buildTool` 的 `prompt()` 方法中调用 `getPrompt()` |
| `src/Tool.ts` | `Tool.prompt` 接口定义 |

## 依赖与外部交互

### 调用方

- **`src/tools/TeamCreateTool/TeamCreateTool.ts`**：
  ```ts
  async prompt() {
    return getPrompt()
  }
  ```
  当系统组装 system prompt 时，框架会调用工具的 `prompt()` 方法，将返回的 Markdown 追加到模型可见的指令中。

### 被调用方/依赖模块

- 无外部运行时依赖。该模块是纯字符串常量工厂。

## 风险、边界与改进建议

### 风险与边界

1. **Prompt 膨胀**
   - 当前 prompt 约 6.9KB（113 行），在 system prompt 中占用了相当可观的 token 预算。对于不使用 Swarm 功能的会话，这段文本仍然会被加载（只要 `TeamCreateTool` 被注册到 tools 列表且 `isEnabled()` 为 true）。虽然 `shouldDefer: true` 意味着 schema 会被延迟发送，但 `prompt()` 返回的文本通常还是会在 system prompt 中完整呈现。

2. **静态文本的维护成本**
   - 所有路径、行为规则、示例都硬编码在字符串中。如果 `~/.claude/teams/` 的底层路径约定发生变化，或工作流步骤有调整，需要手动同步修改此文件，容易遗漏。

3. **重复强调可能导致的模型困惑**
   - 多个章节使用了 "IMPORTANT" 或 "**IMPORTANT**" 强调。虽然目的是提高模型遵守率，但过度使用强调标签可能降低其边际效果。

4. **无国际化（i18n）支持**
   - Prompt 完全使用英文撰写。对于非英语用户，模型需要自行翻译理解，虽然 Claude 具备多语言能力，但直接提供本地化 prompt 可能减少歧义。

### 改进建议

1. **条件加载或摘要化**
   - 如果 Swarm 功能启用率不高，可考虑在 `prompt()` 中根据上下文动态裁剪：例如，若当前会话已经在一个团队中，可以省略 "When to Use" 和基础介绍，只保留与当前角色相关的协作规范。

2. **提取可配置的路径常量**
   - 将 `~/.claude/teams/{team-name}/config.json` 和 `~/.claude/tasks/{team-name}/` 中的路径前缀提取到 `src/utils/envUtils.ts` 或共享常量中，并在 prompt 中通过轻量级函数插值生成，避免硬编码与实际文件系统实现脱节。

3. **增加结构化示例**
   - 当前只有一个极简的 JSON 示例。可考虑增加一个完整的端到端示例（创建团队 → 招募 researcher 和 tester → 分配任务 → 发送消息），帮助模型更好地理解多步骤协作模式。

4. **A/B 测试强调语效果**
   - 通过分析模型在实际对话中违反规则（如发送 JSON 状态消息、用 agentId 代替 name）的频率，评估当前 prompt 中强调语的有效性，并迭代优化措辞。

5. **Prompt 版本化（可选）**
   - 若团队工作流未来会频繁迭代，可考虑将 prompt 内容托管为版本化配置（如 `.claude/prompts/team-create.md`），使非工程师也能调整措辞，而无需修改 TypeScript 源码。不过这会引入运行时文件读取依赖，需要权衡。
