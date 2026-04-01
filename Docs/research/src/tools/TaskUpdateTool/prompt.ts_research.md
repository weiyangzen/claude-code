# 研究报告：src/tools/TaskUpdateTool/prompt.ts

## 场景与职责

`prompt.ts` 负责为 `TaskUpdateTool` 提供面向大语言模型（LLM）的提示文本。它导出两个字符串常量：`DESCRIPTION`（工具的一句话能力描述）和 `PROMPT`（详细的使用说明、字段解释、状态工作流与 JSON 示例）。这两个常量被 `TaskUpdateTool.ts` 注入到 `buildTool` 的 `description()` 和 `prompt()` 方法中，最终出现在发给模型的 system prompt 工具定义部分，直接影响模型何时、如何以及以什么参数调用该工具。

## 功能点目的

1. **DESCRIPTION**：作为工具的“一句话摘要”，在工具列表中向模型快速传达 `TaskUpdateTool` 的核心能力——更新任务列表中的已有任务。
2. **PROMPT**：承担详细的教学职责，具体包括：
   - **使用时机教学**：告诉模型在什么场景下应该调用该工具（完成任务、删除废弃任务、更新需求或依赖关系）。
   - **状态流转约束**：明确任务状态只能按 `pending` → `in_progress` → `completed` 前进，并用 `deleted` 永久移除任务。
   - **完成标准强化**：通过多条 `IMPORTANT` / `Never` / `ONLY` 指令，反复告诫模型不要过早标记任务完成（如测试失败、实现不完整、遇到未解决错误时均不得标记为 `completed`）。
   - **字段说明**：逐条解释 `status`、`subject`、`description`、`activeForm`、`owner`、`metadata`、`addBlocks`、`addBlockedBy` 的语义与用途。
   - **staleness 提醒**：要求模型在更新前先用 `TaskGet` 读取任务的最新状态，防止基于陈旧信息做更新。
   - **JSON 示例**：提供 5 个典型用例的 JSON 输入示例（标记进行中、标记完成、删除、认领、设置依赖），降低模型构造错误参数的概率。

## 具体技术实现

文件实现为纯静态字符串导出，无运行时计算、无外部依赖、无国际化（i18n）逻辑。内容结构如下：

```ts
export const DESCRIPTION = 'Update a task in the task list'

export const PROMPT = `Use this tool to update a task in the task list.

## When to Use This Tool
...

## Fields You Can Update
...

## Status Workflow
...

## Staleness
...

## Examples
...`
```

### 被消费方式

在 `TaskUpdateTool.ts` 中：

```ts
async description() {
  return DESCRIPTION
},
async prompt() {
  return PROMPT
},
```

`buildTool` 将这两个方法包装进 `Tool` 对象。在 system prompt 组装阶段（通常由 `getSystemPrompt()` 或类似逻辑负责），每个延迟加载（`shouldDefer: true`）的工具会在被 `ToolSearch` 解锁后，将其 `description` 和 `prompt` 拼接进模型可见的上下文。

## 关键代码路径与文件引用

| 文件 | 关系 | 说明 |
|------|------|------|
| `src/tools/TaskUpdateTool/TaskUpdateTool.ts` | 直接导入并消费 | 将 `DESCRIPTION` 和 `PROMPT` 分别作为 `description()` 和 `prompt()` 的返回值 |
| `src/Tool.ts` | 间接依赖 | `buildTool` 与 `ToolDef` 类型定义了 `description` 和 `prompt` 的接口契约 |
| System Prompt 组装层 | 最终消费方 | 在运行时调用 `tool.description()` 与 `tool.prompt()` 生成模型可见文本（具体文件不在本批次研究范围内，通常为 `src/utils/systemPrompt*.ts` 系列文件） |

## 依赖与外部交互

- **零运行时依赖**：该文件不 import 任何模块，也不依赖环境变量或全局状态。
- **单向数据流**：仅被 `TaskUpdateTool.ts` 读取，不向任何其他系统回写数据或触发副作用。
- **与模型行为的强关联**：prompt 中的措辞直接影响模型的工具调用准确率。例如：
  - `"ONLY mark a task as completed when you have FULLY accomplished it"` 旨在减少模型过早关闭任务的幻觉；
  - `"After resolving, call TaskList to find your next task"` 旨在促进 swarm 场景下的任务自调度；
  - `addBlockedBy` 的示例 `"{"taskId": "2", "addBlockedBy": ["1"]}"` 帮助模型理解依赖方向（2 被 1 阻塞）。

## 风险、边界与改进建议

### 风险与边界

1. **静态文本的维护负担**：`PROMPT` 是一个长达 70+ 行的模板字符串，任何措辞微调都需要修改源码并重新构建/发布。当前没有 A/B 测试框架或动态 prompt 配置机制来支持快速迭代。
2. **无国际化支持**：提示文本完全为英文。虽然 Claude Code 目前主要面向英语用户，但随着产品扩展，硬编码英文 prompt 可能成为非英语用户的使用障碍。
3. **示例覆盖有限**：仅提供了 5 个示例，未覆盖 `metadata` 更新、`activeForm` 修改、多字段同时更新等场景。模型在遇到这些场景时可能需要更多推理步骤才能构造正确参数。
4. **与 `TaskCreateTool` / `TaskGetTool` prompt 的协同一致性风险**：任务工具族的 prompt 分散在各自文件中，若某次更新只改了 `TaskUpdateTool` 的措辞而忘了同步 `TaskCreateTool` 或 `TaskGetTool`，可能导致模型对任务系统的整体理解出现偏差（例如状态名称不一致、示例风格不统一）。
5. **Staleness 提醒的效力有限**：虽然 prompt 中明确要求 "Make sure to read a task's latest state using `TaskGet` before updating it"，但模型并不总是严格遵守。该约束目前仅靠 prompt engineering 实现，没有运行时强制校验（例如不会检查 `messages` 历史中是否真的出现过对同一 `taskId` 的 `TaskGet` 调用）。

### 改进建议

1. **增加更多边缘场景示例**：在 `## Examples` 中补充 `metadata` 更新（包括设值为 `null` 删除键）和 `activeForm` 修改的示例，降低模型在这些字段上的犯错率。
2. **建立任务工具族 Prompt 风格指南**：在 `AGENTS.md` 或团队文档中定义任务相关工具 prompt 的编写规范（如状态名称必须大写还是小写、示例 JSON 的缩进风格、`IMPORTANT` 标注的使用频率），确保 `TaskCreate`、`TaskGet`、`TaskUpdate`、`TaskList` 的 prompt 在迭代中保持一致。
3. **考虑运行时 staleness 辅助**：虽然不应过度约束模型，但可以在 `TaskUpdateTool.ts` 的 `call()` 方法中增加轻量级的“乐观并发”提示。例如，若检测到任务文件的最后修改时间晚于当前会话中最近一次 `TaskGet` 的时间戳，可在返回结果中追加警告（`warning: task was modified since last read`），以软性方式强化 staleness 意识。
4. **Prompt 版本化或动态化（长期）**：若未来需要频繁调整 prompt 策略，可考虑将 `PROMPT` 从源码中抽离到配置层（如 GrowthBook 字符串配置或本地 `~/.claude/prompts/`），支持热更新和灰度实验，而无需重新发版。
5. **精简重复强调句**：当前 prompt 中使用了大量否定句和强调句（`Never mark...`、`ONLY mark...`、`IMPORTANT: Always...`）。虽然这有助于对齐模型行为，但过长的禁令列表可能增加 prompt 长度并稀释重点。建议定期进行 prompt 蒸馏实验，找出对模型行为影响最大的 2-3 条核心指令，将次要信息移至 `TaskListTool` 或全局 system prompt 中统一说明。
