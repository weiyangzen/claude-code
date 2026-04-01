# prompt.ts 研究文档

## 场景与职责

`prompt.ts` 是 `TaskGetTool` 模块的**提示词与描述定义文件**，负责向模型提供该工具的使用说明。它导出了两个核心常量：
- `DESCRIPTION`：工具的简短功能描述，用于 ToolSearch、工具列表渲染和系统提示中的工具描述。
- `PROMPT`：详细的工具使用指南，在模型需要调用该工具时注入到对话上下文中，指导模型何时使用、期望什么输出、有哪些最佳实践。

## 功能点目的

1. **描述工具能力**：`DESCRIPTION` 用一句话概括工具用途（`Get a task by ID from the task list`），帮助模型在工具选择和搜索阶段快速识别该工具。
2. **指导使用时机**：`PROMPT` 明确告诉模型在哪些场景下应该调用 TaskGetTool，例如：
   - 开始工作前需要获取任务的完整描述和上下文。
   - 需要理解任务依赖关系（blocks / blockedBy）。
   - 被分配任务后获取完整需求。
3. **解释输出语义**：`PROMPT` 详细说明返回字段的含义（subject、description、status、blocks、blockedBy），降低模型对输出结构的误解。
4. **嵌入最佳实践**：提示模型在获取任务后先检查 `blockedBy` 是否为空再开始工作，并建议配合 `TaskList` 工具使用。

## 具体技术实现

### 导出结构

```typescript
export const DESCRIPTION = 'Get a task by ID from the task list'

export const PROMPT = `Use this tool to retrieve a task by its ID from the task list.

## When to Use This Tool

- When you need the full description and context before starting work on a task
- To understand task dependencies (what it blocks, what blocks it)
- After being assigned a task, to get complete requirements

## Output

Returns full task details:
- **subject**: Task title
- **description**: Detailed requirements and context
- **status**: 'pending', 'in_progress', or 'completed'
- **blocks**: Tasks waiting on this one to complete
- **blockedBy**: Tasks that must complete before this one can start

## Tips

- After fetching a task, verify its blockedBy list is empty before beginning work.
- Use TaskList to see all tasks in summary form.
`
```

### 在工具定义中的使用

在 `src/tools/TaskGetTool/TaskGetTool.ts` 中：

```typescript
import { DESCRIPTION, PROMPT } from './prompt.js'

export const TaskGetTool = buildTool({
  name: TASK_GET_TOOL_NAME,
  async description() {
    return DESCRIPTION
  },
  async prompt() {
    return PROMPT
  },
  // ...
})
```

- `description()` 返回 `DESCRIPTION`，用于系统提示中的工具摘要和 ToolSearch 关键词匹配。
- `prompt()` 返回 `PROMPT`，当工具被延迟加载（`shouldDefer: true`）或模型显式请求时，该提示会被附加到工具 schema 之后，作为调用指导。

### 提示词设计特点

- **结构化 Markdown**：使用二级标题（`## When to Use This Tool`、`## Output`、`## Tips`）组织内容，便于模型解析层次。
- **场景化列表**：`When to Use This Tool` 采用 bullet list 列出三个典型场景，帮助模型建立调用直觉。
- **字段语义明确**：`Output` 部分对每个字段给出简洁定义，特别区分了 `blocks`（ outgoing 依赖）和 `blockedBy`（ incoming 依赖）的方向性。
- **行动建议**：`Tips` 部分给出可操作的建议，直接关联到 swarm 协作中的任务阻塞检查逻辑。

## 关键代码路径与文件引用

### 本文件
- `src/tools/TaskGetTool/prompt.ts` — 提示词定义源文件。

### 直接引用方
- `src/tools/TaskGetTool/TaskGetTool.ts` — 导入 `DESCRIPTION` 和 `PROMPT`，分别用于 `buildTool` 的 `description()` 和 `prompt()` 方法。

### 间接引用方
- `src/Tool.ts` — `Tool` 类型定义了 `description()` 和 `prompt()` 的接口契约。
- `src/tools/ToolSearchTool/ToolSearchTool.ts` — 在工具搜索匹配时可能读取 `description()` 返回值作为匹配文本。
- 系统提示组装逻辑（如 `src/constants/prompts.ts` 或相关入口）— 在组装延迟加载工具的提示时调用 `tool.prompt()`。

## 依赖与外部交互

该文件**无任何外部依赖**，也不产生任何外部交互。它是一个纯字符串常量模块，所有内容在编译时即确定，运行时仅涉及字符串返回。

## 风险、边界与改进建议

### 风险

1. **提示词与实际 schema 不同步**：若未来 `TaskGetTool.ts` 的 `outputSchema` 增加了新字段（如 `owner`、`metadata`），而 `PROMPT` 中的 `## Output` 部分未及时更新，模型可能无法充分利用新字段，甚至产生幻觉。
2. **状态值硬编码**：`PROMPT` 中直接写死了 `status: 'pending', 'in_progress', or 'completed'`，而真实的状态定义来自 `TaskStatusSchema()`。若 `TaskStatusSchema` 未来扩展新状态（如 `cancelled`），提示词将过时。
3. **语言单一**：当前提示词为纯英文。若项目未来需要支持多语言模型或本地化，硬编码的英文提示词可能成为障碍。

### 边界

- **无动态内容**：`PROMPT` 是静态字符串，无法根据当前任务列表状态、环境变量或用户偏好进行动态调整。例如，无法在提示中告知模型当前任务列表中有多少任务。
- **无版本控制**：提示词内容的修改不会在代码层面留下版本标记，难以追踪某次行为变更是否由提示词调整引起。

### 改进建议

1. **与 schema 联动生成输出说明**：可考虑在 `prompt.ts` 中导入 `TaskStatusSchema` 的描述，动态生成状态值列表，避免硬编码。例如：
   ```typescript
   import { TASK_STATUSES } from '../../utils/tasks.js'
   
   const statusList = TASK_STATUSES.map(s => `'${s}'`).join(', ')
   ```
   这样当 `TASK_STATUSES` 扩展时，提示词自动同步。
2. **增加字段变更同步检查**：在项目测试或 lint 规则中增加一项检查，验证 `PROMPT` 中 `## Output` 部分列出的字段是否与 `outputSchema` 定义一致。虽然实现成本较高，但能有效防止文档漂移。
3. **提取可复用的提示词片段**：`"Use TaskList to see all tasks in summary form"` 这一建议在多个任务工具（TaskCreate、TaskUpdate、TaskGet）的提示词中反复出现。可考虑将其提取为共享的提示词片段常量，减少重复并便于统一维护。
4. **考虑增加 `owner` 字段说明**：如果 `TaskGetTool` 的输出 schema 未来扩展了 `owner` 字段，应在 `## Output` 中补充：
   ```
   - **owner**: The agent or user currently assigned to this task
   ```
   并更新 `## Tips` 以提示模型在 owner 非空时注意任务已被认领。
