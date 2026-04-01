# prompt.ts 研究文档

> 文件路径：`src/tools/TaskStopTool/prompt.ts`  
> 研究时间：2026-04-01  
> 执行器：kimi / k2p5

---

## 一、场景与职责

`prompt.ts` 是 `TaskStopTool` 的**提示词与常量定义文件**，职责极其单一且明确：

1. 导出工具的**内部名称常量** `TASK_STOP_TOOL_NAME`，供工具定义、常量聚合、权限系统、Coordinator 模式等多个模块引用。
2. 导出工具的**自然语言描述** `DESCRIPTION`，在模型请求时作为该工具的 system prompt 片段注入，告知 LLM 该工具的用途与调用时机。

该文件是 Claude Code 工具目录下的标准模式文件（与 `src/tools/BashTool/prompt.ts`、`src/tools/GlobTool/prompt.ts` 等保持一致），属于“纯数据、无逻辑”的叶子模块。

---

## 二、功能点目的

| 功能点 | 目的 |
|--------|------|
| `TASK_STOP_TOOL_NAME` | 提供单一可信来源（Single Source of Truth）的工具名称字符串 `'TaskStop'`，避免在代码库中硬编码散落。 |
| `DESCRIPTION` | 为 LLM 提供简洁的工具说明：用于终止后台任务、接收 `task_id`、返回成功/失败状态。 |

---

## 三、具体技术实现

### 3.1 源码全文

```ts
export const TASK_STOP_TOOL_NAME = 'TaskStop'

export const DESCRIPTION = `
- Stops a running background task by its ID
- Takes a task_id parameter identifying the task to stop
- Returns a success or failure status
- Use this tool when you need to terminate a long-running task
`
```

### 3.2 设计细节

- **无 Zod/无运行时依赖**：该文件不导入任何外部模块，是构建图中的叶子节点，加载开销为零。
- **DESCRIPTION 格式**：采用 Markdown 无序列表（`- `）形式，与 Claude Code 其他工具的 prompt 风格一致，便于在 system prompt 中拼接成统一的工具说明区块。
- **名称与别名分离**：`TASK_STOP_TOOL_NAME` 只声明主名称 `'TaskStop'`；废弃别名 `'KillShell'` 及其兼容逻辑完全由 `TaskStopTool.ts` 管理，不在本文件中体现，避免常量污染。

---

## 四、关键代码路径与文件引用

| 引用关系 | 文件路径 | 说明 |
|---------|----------|------|
| **被导入（工具定义）** | `src/tools/TaskStopTool/TaskStopTool.ts` | 导入 `DESCRIPTION` 作为 `prompt()` 返回值；导入 `TASK_STOP_TOOL_NAME` 作为 `buildTool({name})`。 |
| **被导入（常量聚合）** | `src/constants/tools.ts` | 导入 `TASK_STOP_TOOL_NAME` 并用于构造 `ALL_AGENT_DISALLOWED_TOOLS`、`COORDINATOR_MODE_ALLOWED_TOOLS` 等集合。 |
| **被导入（Coordinator 模式）** | `src/coordinator/coordinatorMode.ts` | 导入 `TASK_STOP_TOOL_NAME` 并嵌入 coordinator system prompt 的模板字符串中。 |
| **被导入（权限/分类器）** | `src/utils/permissions/classifierDecision.ts` | 引用 `TASK_STOP_TOOL_NAME` 进行工具名称判断（通过 Grep 确认存在引用）。 |
| **被导入（权限规则解析）** | `src/utils/permissions/permissionRuleParser.ts` | 在权限规则解析中引用该工具名。 |
| **被导入（Streamlined 转换）** | `src/utils/streamlinedTransform.ts` | 在工具名映射/转换逻辑中引用。 |

---

## 五、依赖与外部交互

- **零运行时依赖**：该文件不 `import` 任何模块，也不依赖任何全局变量。
- **纯字符串导出**：`DESCRIPTION` 在 `TaskStopTool.ts` 的 `prompt()` 方法中被直接返回；`prompt()` 方法由 `toolExecution.ts` 在组装 system prompt 时异步调用。
- **与权限系统的交互**：`TASK_STOP_TOOL_NAME` 被 `src/constants/tools.ts` 导入后，用于定义哪些工具对 async agent 禁用、哪些工具在 coordinator 模式下可用。权限系统（如 `src/utils/permissions/permissions.ts`）通过工具名称字符串匹配来决定放行策略，因此该常量的准确性直接影响安全边界。

---

## 六、风险、边界与改进建议

### 6.1 已知风险

1. **名称变更的级联影响**：`TASK_STOP_TOOL_NAME` 被近 10 个文件引用，若未来需要重命名工具（例如彻底去掉 `KillShell` 遗留痕迹而改为 `StopTask`），需要同步修改所有引用点。虽然 TypeScript 会在编译期捕获未更新的导入，但字符串集合（如 `src/constants/tools.ts` 中的 `Set`）不会自动感知重命名，属于潜在的维护风险。

2. **DESCRIPTION 过于简略**：当前描述仅 4 行，未提及：
   - `shell_id` 废弃参数的存在（虽然不建议模型使用，但旧 transcript 恢复时可能遇到）
   - 只能停止 `status === 'running'` 的任务
   - 停止后任务可继续通过 `SendMessageTool` 恢复（Coordinator 模式场景）
   这可能导致模型在错误场景下调用（例如尝试停止已完成的任务）。

### 6.2 边界行为

- **无边界逻辑**：该文件本身不包含任何条件分支或运行时计算，不存在输入边界。
- **DESCRIPTION 空白字符**：模板字符串首尾各有一个换行符，在 `TaskStopTool.ts` 的 `prompt()` 中直接返回。system prompt 组装层通常会做 trim 或格式化，因此额外的首尾换行不会产生影响。

### 6.3 改进建议

1. **扩充 DESCRIPTION 以提升模型调用准确率**：
   ```ts
   export const DESCRIPTION = `
   - Stops a running background task by its ID
   - Takes a task_id parameter identifying the task to stop
   - The task must currently be in the "running" status; stopping a completed or failed task will return an error
   - Returns a success or failure status
   - Use this tool when you need to terminate a long-running task or worker
   - In coordinator mode, stopped workers can later be continued with SendMessage
   `
   ```
   增加状态限制说明可减少模型误调用；提及 `SendMessage` 恢复有助于 coordinator 模式下的正确操作。

2. **考虑统一工具常量文件**：Claude Code 中每个工具目录都有一个 `prompt.ts`（或 `constants.ts`）导出 `TOOL_NAME`。可以考虑在 `src/constants/tools.ts` 中统一反向导出所有工具名（`export { TASK_STOP_TOOL_NAME } from '../tools/TaskStopTool/prompt.js'`），减少跨目录导入的分散性。当前 `src/constants/tools.ts` 已经做了这种聚合，但方向是“从各工具导入到常量文件”，这是合理的模式，应继续保持。

3. **添加 JSDoc 注释**：为 `TASK_STOP_TOOL_NAME` 和 `DESCRIPTION` 添加简短 JSDoc，说明其用途与消费位置，提升代码可读性：
   ```ts
   /** Internal tool name for the task stop tool. */
   export const TASK_STOP_TOOL_NAME = 'TaskStop'

   /** System prompt description shown to the model for TaskStop. */
   export const DESCRIPTION = `...`
   ```
