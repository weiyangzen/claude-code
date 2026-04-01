# TaskStopTool.ts 研究文档

> 文件路径：`src/tools/TaskStopTool/TaskStopTool.ts`  
> 研究时间：2026-04-01  
> 执行器：kimi / k2p5

---

## 一、场景与职责

`TaskStopTool.ts` 定义了 **TaskStop** 工具（内部名 `TaskStop`，曾用名/别名 `KillShell`），是 Claude Code 中唯一供 LLM 主动终止后台运行任务的入口。该工具面向两类核心场景：

1. **用户/LLM 主动停止后台任务**：例如终止一个运行过久的 Bash 命令、停止一个走错方向的 Agent worker、中断一个远程会话等。
2. **Coordinator 模式下的 worker 生命周期管理**：在 Coordinator 模式下，TaskStop 与 AgentTool、SendMessageTool 并列为 coordinator 仅有的几个可用工具之一，用于在发现任务方向错误时及时停止 worker。

该文件本身**只负责工具契约定义、输入校验与结果组装**，真正的“停止”动作委托给 `src/tasks/stopTask.ts` 中的 `stopTask()` 函数，以保证 LLM 调用路径与 SDK `stop_task` 控制请求路径共享同一套核心逻辑。

---

## 二、功能点目的

| 功能点 | 目的 |
|--------|------|
| **定义输入/输出 Schema** | 使用 Zod 描述工具参数：`task_id`（可选字符串）与兼容旧版的 `shell_id`（可选字符串）。输出包含 `message`、`task_id`、`task_type`、`command`。 |
| **别名兼容 KillShell** | 设置 `aliases: ['KillShell']`，保证旧 transcript 或旧 SDK 调用仍能命中该工具。 |
| **输入预校验 (`validateInput`)** | 在权限检查前快速失败：校验 `task_id`/`shell_id` 非空、任务存在、且状态为 `running`。失败时返回结构化错误码（1=缺失参数/未找到，3=未在运行）。 |
| **调用 `stopTask` 执行终止** | 通过 `getAppState` / `setAppState` 拿到任务状态，委托 `stopTask(id, {getAppState, setAppState})` 完成实际 kill。 |
| **结果序列化** | 使用 `jsonStringify` 将输出对象序列化为 API `tool_result` 的 `content`。 |
| **UI 渲染委托** | 将工具使用与结果渲染委托给同目录 `UI.tsx` 中的 `renderToolUseMessage` 与 `renderToolResultMessage`。 |
| **Deferred 加载** | `shouldDefer: true` 表示该工具默认不会随初始请求下发完整 schema，需通过 `ToolSearchTool` 按需加载（减少 prompt 体积）。 |
| **并发安全** | `isConcurrencySafe: () => true` 声明该工具可多实例并行调用。 |

---

## 三、具体技术实现

### 3.1 关键数据结构

```ts
// 输入（Zod strictObject）
{
  task_id?: string   // 要停止的任务 ID
  shell_id?: string  // 已废弃，兼容旧 KillShell
}

// 输出（Zod object）
{
  message: string    // 操作状态描述
  task_id: string    // 被停止的任务 ID
  task_type: string  // 被停止的任务类型
  command?: string   // 任务的命令/描述（可选，兼容旧 transcript）
}
```

### 3.2 核心执行流程

```
LLM 调用 TaskStop(task_id)
        │
        ▼
[toolExecution.ts] 解析 tool_use，发现 name 为 "TaskStop" 或别名 "KillShell"
        │
        ▼
[TaskStopTool.ts] validateInput()
    ├─ 取 id = task_id ?? shell_id
    ├─ id 为空 → 返回 {result:false, message:"Missing required parameter: task_id", errorCode:1}
    ├─ 在 appState.tasks[id] 中查找任务
    ├─ 任务不存在 → 返回 {result:false, message:"No task found with ID: ${id}", errorCode:1}
    └─ 任务状态不是 running → 返回 {result:false, message:"Task ${id} is not running...", errorCode:3}
        │
        ▼
[TaskStopTool.ts] call()
    ├─ 再次校验 id 非空（防御性编程）
    └─ await stopTask(id, {getAppState, setAppState})
        │
        ▼
[stopTask.ts] 查找任务 → 校验 running → 获取 taskImpl → await taskImpl.kill()
    ├─ 若是 local_bash 任务，设置 notified=true 以抑制后续 "exit code 137" 通知噪音
    └─ 同时通过 emitTaskTerminatedSdk() 向 SDK 消费者发送 task_notification(stopped)
        │
        ▼
返回 {data: {message, task_id, task_type, command}}
        │
        ▼
[TaskStopTool.ts] mapToolResultToToolResultBlockParam() 将结果 JSON 序列化后塞入 tool_result
```

### 3.3 关键代码路径

- **输入校验**：`validateInput`（第 60–91 行）
- **实际调用**：`call`（第 107–130 行）
- **Schema 定义**：`inputSchema` / `outputSchema`（第 10–35 行），使用 `lazySchema` 延迟构造以优化模块加载性能。
- **工具构建**：`buildTool({...})`（第 39–131 行），`satisfies ToolDef<InputSchema, Output>` 保证类型安全。

---

## 四、关键代码路径与文件引用

| 引用/被引用关系 | 文件路径 | 说明 |
|----------------|----------|------|
| **被调用方（核心）** | `src/tasks/stopTask.ts` | 实际执行停止逻辑，TaskStopTool 仅做工具层封装。 |
| **被调用方（UI）** | `src/tools/TaskStopTool/UI.tsx` | `renderToolUseMessage`、`renderToolResultMessage` 的宿主。 |
| **被调用方（序列化）** | `src/utils/slowOperations.ts` | `jsonStringify`：带慢操作监控的 JSON.stringify 包装。 |
| **被调用方（Schema 延迟构造）** | `src/utils/lazySchema.ts` | `lazySchema`：将 Zod schema 构造推迟到首次访问。 |
| **类型依赖** | `src/Tool.ts` | `buildTool`、`ToolDef`、`ValidationResult` 等类型。 |
| **类型依赖** | `src/Task.ts` | `TaskStateBase`：任务状态基类型。 |
| **常量依赖** | `src/tools/TaskStopTool/prompt.ts` | `DESCRIPTION`、`TASK_STOP_TOOL_NAME`。 |
| **调用方（注册）** | `src/tools.ts` | `getAllBaseTools()` 将 `TaskStopTool` 纳入全局工具池。 |
| **调用方（别名回退）** | `src/services/tools/toolExecution.ts` | 若模型传入 `"KillShell"`，通过 `findToolByName` + `aliases` 回退到 `TaskStopTool`。 |
| **调用方（Coordinator 模式）** | `src/coordinator/coordinatorMode.ts` | Coordinator system prompt 中显式引用 `TaskStop`，指导 coordinator 停止 worker。 |
| **调用方（常量聚合）** | `src/constants/tools.ts` | 导入 `TASK_STOP_TOOL_NAME`，并列入 `ALL_AGENT_DISALLOWED_TOOLS` 与 `COORDINATOR_MODE_ALLOWED_TOOLS`。 |

---

## 五、依赖与外部交互

### 5.1 运行时依赖

- **Zod v4**：输入/输出 schema 定义与校验（`z.strictObject`、`z.object`）。
- **AppState**：通过 `ToolUseContext` 提供的 `getAppState` / `setAppState` 访问任务状态树。
- **任务调度层**：`stopTask()` 会进一步通过 `getTaskByType()`（`src/tasks.ts`）拿到对应 `Task` 实现，再调用其 `kill()` 方法。

### 5.2 外部交互

- **SDK/Headless 模式**：`stopTask.ts` 在停止 `local_bash` 任务时会调用 `emitTaskTerminatedSdk()`（`src/utils/sdkEventQueue.ts`），向外部消费者（如 VS Code 扩展、Scuttle）推送 `task_notification` 事件，状态为 `stopped`。
- **Transcript 恢复**：`command` 字段标记为可选，因为旧 transcript 在 `--resume` 时可能缺少该字段；工具结果在恢复时不会重新校验，因此保持可选以避免崩溃。

---

## 六、风险、边界与改进建议

### 6.1 已知风险

1. **参数兼容债务**：`shell_id` 作为废弃字段仍保留在 schema 与校验逻辑中。虽然注释明确标注 deprecated，但长期维护会增加 schema 噪音。建议在未来 major 版本移除，并更新 alias 回退逻辑。
2. **重复校验**：`validateInput` 与 `call` 中均对 `id` 做了非空检查。`validateInput` 已经保证通过，但 `call` 中仍保留防御性 `throw`，逻辑上不会触发，属于冗余安全网。
3. **Deferred 工具的 schema 未发送问题**：作为 `shouldDefer: true` 的工具，若模型在未调用 `ToolSearchTool` 的情况下直接调用 `TaskStop`，`toolExecution.ts` 的 `buildSchemaNotSentHint` 会附加提示。这在实际使用中极少发生，但新用户或快速 transcript 中可能多一次 round-trip。

### 6.2 边界行为

- **停止已完成的任务**：`validateInput` 会拦截并返回 `errorCode: 3`，不会调用 `stopTask`。
- **停止不存在的任务**：返回 `errorCode: 1`，模型会收到明确的错误信息。
- **Agent 子代理禁用**：`src/constants/tools.ts` 将 `TaskStopTool` 列入 `ALL_AGENT_DISALLOWED_TOOLS`，因此 async agent / worker 无法调用该工具；只有主线程或 coordinator 能使用。
- **Coordinator Simple 模式**：当 `CLAUDE_CODE_SIMPLE` 启用且处于 coordinator 模式时，`src/tools.ts` 的 `getTools()` 会额外把 `TaskStopTool` 加入可用列表，保证 coordinator 在精简工具集下仍能管理 worker。

### 6.3 改进建议

1. **移除 `shell_id` 债务**：在确认所有旧 transcript / SDK 调用方已迁移后，从 schema 与代码中彻底移除 `shell_id`，仅保留 `aliases: ['KillShell']` 作为名称层面的兼容。
2. **统一错误类型**：`validateInput` 返回数字 `errorCode`，而 `stopTask.ts` 内部使用 `StopTaskError` 带字符串 `code`。建议将 `validateInput` 的错误码也收敛到 `StopTaskError` 的枚举，便于上层统一处理。
3. **增加测试覆盖**：当前仓库中未找到针对 `TaskStopTool` 或 `stopTask` 的单元测试。建议补充以下测试用例：
   - 正常停止 running 的 local_bash 任务
   - 停止 running 的 local_agent 任务
   - 尝试停止已完成的任务（校验失败路径）
   - 尝试停止不存在的任务（校验失败路径）
   - `shell_id` 别名兼容路径
   - `notified=true` 后 `emitTaskTerminatedSdk` 的调用断言
4. **UI.tsx 死代码清理**：`UI.tsx` 中存在 `"external" === 'ant'` 这种永远为 `false` 的分支，建议与 UI 文件一并清理（见 UI.tsx 研究文档）。
