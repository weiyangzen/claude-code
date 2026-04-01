# renderToolActivity.tsx 研究文档

## 场景与职责

`renderToolActivity.tsx` 是 Claude Code 终端 UI 中负责将**单个工具调用活动（ToolActivity）渲染为人类可读字符串/React 节点**的纯函数组件文件。它主要服务于异步代理（Async Agent）和进程内队友（In-Process Teammate）的进度详情弹窗，用于在 "Progress" 区域展示最近执行的工具列表。

消费方：
- `src/components/tasks/AsyncAgentDetailDialog.tsx`（行 17 引用）
- `src/components/tasks/InProcessTeammateDetailDialog.tsx`（行 16 引用）

该组件的核心价值在于**解耦工具定义与 UI 渲染**：它不需要知道具体有哪些工具，只需通过统一的 `Tool` 接口（`findToolByName`、`inputSchema.safeParse`、`userFacingName`、`renderToolUseMessage`）即可生成友好的活动描述。

---

## 功能点目的

### `renderToolActivity`
签名：
```ts
export function renderToolActivity(
  activity: ToolActivity,
  tools: Tools,
  theme: ThemeName,
): React.ReactNode
```

执行流程：
1. **工具查找**：`findToolByName(tools, activity.toolName)`。
   - 若找不到工具，直接回退到 `activity.toolName`（原始工具名）。
2. **输入解析**：`tool.inputSchema.safeParse(activity.input)`。
   - 解析成功则使用 `parsed.data`。
   - 解析失败则回退到空对象 `{}`（防御性编程，避免 malformed input 导致 UI 崩溃）。
3. **获取面向用户的名称**：`tool.userFacingName(parsedInput)`。
   - 若返回空/假值，回退到 `activity.toolName`。
4. **渲染工具参数摘要**：`tool.renderToolUseMessage(parsedInput, { theme, verbose: false })`。
   - 若返回非空值，将其包装为 `<Text>{userFacingName}({toolArgs})</Text>`。
   - 若返回空值，仅返回 `userFacingName` 字符串。
5. **异常兜底**：整个 `try/catch` 块包裹上述逻辑，任何异常（如工具方法抛出、schema 解析异常）都回退到 `activity.toolName`。

---

## 具体技术实现（关键流程/数据结构/协议/命令）

### 核心类型
```ts
// 来自 src/tasks/LocalAgentTask/LocalAgentTask.js
export type ToolActivity = {
  toolName: string;
  input: Record<string, unknown>;
  activityDescription?: string;  // 预计算描述（本组件未使用）
  isSearch?: boolean;
  isRead?: boolean;
};
```

### 工具接口调用链
```ts
const tool = findToolByName(tools, activity.toolName);
const parsed = tool.inputSchema.safeParse(activity.input);
const parsedInput = parsed.success ? parsed.data : {};
const userFacingName = tool.userFacingName(parsedInput);
const toolArgs = tool.renderToolUseMessage(parsedInput, { theme, verbose: false });
```

### 为什么 `verbose: false`？
`renderToolUseMessage` 的第二个参数包含 `verbose` 标志。在进度详情弹窗中，空间紧凑，因此强制使用非详细模式，只展示最精简的参数摘要（如文件路径、搜索模式等）。

### 为什么 `safeParse` 失败回退到 `{}`？
`activity.input` 来自运行时记录的原始 tool_use 输入。在极少数情况下（如工具 schema 升级后读取旧会话的历史记录），输入可能不符合当前 schema。回退到 `{}` 可以确保 `userFacingName({})` 和 `renderToolUseMessage({}, ...)` 仍能返回一个合理的默认字符串（通常是工具名本身），而不是让整个弹窗崩溃。

### 为什么不用 `activityDescription`？
`ToolActivity` 类型中确实包含 `activityDescription`（由 `createActivityDescriptionResolver` 在 `LocalAgentTask.tsx` 中预计算）。但 `renderToolActivity` 选择直接调用工具的 `userFacingName` + `renderToolUseMessage`，原因是：
- `activityDescription` 通常是一个句子（如 `"Reading src/foo.ts"`），适合 spinner 提示。
- `renderToolActivity` 需要生成的是**工具调用表达式风格**的文本（如 `Read(src/foo.ts)`），与详情弹窗中 "Progress" 列表的紧凑格式更匹配。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/tasks/renderToolActivity.tsx` | 本文件 |
| `src/components/tasks/AsyncAgentDetailDialog.tsx` | 调用 `renderToolActivity` 渲染 async agent 的 recentActivities |
| `src/components/tasks/InProcessTeammateDetailDialog.tsx` | 调用 `renderToolActivity` 渲染 teammate 的 recentActivities |
| `src/Tool.ts` | `findToolByName`、`Tools` 类型、`Tool` 接口定义 |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | `ToolActivity` 类型定义、进度追踪逻辑 |
| `src/utils/theme.ts` | `ThemeName` 类型 |
| `src/ink.js` | `Text` 组件 |

---

## 依赖与外部交互

### 运行时依赖
- **`findToolByName`**：来自 `src/Tool.ts`，按主名或别名在工具数组中查找。
- **Zod schema**：`tool.inputSchema.safeParse` 是 Zod v4 的 API（项目中统一使用 `zod/v4`）。
- **Ink `Text`**：仅在需要同时展示 `userFacingName` 和 `toolArgs` 时用于包裹文本节点。

### 数据流
1. `LocalAgentTask.tsx` 中的 `updateProgressFromMessage` 在每次 assistant message 到达时，提取 `tool_use` 块并构建 `ToolActivity` 对象，追加到 `ProgressTracker.recentActivities`。
2. `AsyncAgentDetailDialog.tsx` / `InProcessTeammateDetailDialog.tsx` 从 `agent.progress.recentActivities` 读取活动列表。
3. 对每个 `activity`，调用 `renderToolActivity(activity, tools, theme)` 生成 React 节点。
4. 详情弹窗将生成的节点按时间顺序渲染为带 `›` 前缀的列表项。

---

## 风险、边界与改进建议

### 风险
1. **`tools` 数组与 `activity` 不同步**：如果 `activity` 记录的是某个 MCP 工具或动态加载工具的调用，而当前 `tools` 数组中该工具已被卸载或重命名，`findToolByName` 会返回 `undefined`，导致回退到原始工具名，用户看到的可能是内部 ID（如 `mcp__slack__send_message`）而非友好名称。
2. **性能隐患**：`renderToolActivity` 在每次渲染时都会执行 `safeParse` 和两次工具方法调用（`userFacingName`、`renderToolUseMessage`）。虽然单个调用开销很小，但如果 `recentActivities` 有 5 项且详情弹窗每秒重渲染一次，累积调用次数可能较高。不过目前该组件未被观察到性能瓶颈。
3. **异常静默吞掉**：`try/catch` 捕获所有异常并回退到 `activity.toolName`，这虽然保证了 UI 不崩溃，但也掩盖了工具实现中的潜在 bug（如 `renderToolUseMessage` 对空对象处理不当）。

### 边界
- 若 `tool.renderToolUseMessage` 返回 `null` / `undefined` / 空字符串，组件不会渲染括号参数，仅返回 `userFacingName`。
- 若 `userFacingName` 返回空字符串，组件直接回退到 `activity.toolName`。
- 该组件**不处理** `activityDescription` 字段，也不区分 `isSearch` / `isRead` 标志——这些分类信息由调用方（详情弹窗）或 `collapseReadSearch.ts` 处理。

### 改进建议
1. **增加缓存层**：`renderToolActivity` 的输入（`activity`, `tools`, `theme`）在相邻渲染帧之间通常不变。可以在调用方（如 `AsyncAgentDetailDialog`）使用 `useMemo` 对 `recentActivities.map(...)` 做缓存，避免不必要的重复解析。目前 `AsyncAgentDetailDialog.tsx` 中该列表确实是在 render 中直接 `map`，没有额外 memoization。
2. **暴露更详细的 fallback 信息**：在开发模式下，可以在 `catch` 块中通过 `console.error` 或 `logForDebugging` 记录异常，帮助定位工具渲染 bug，而不是完全静默。
3. **复用 `activityDescription` 作为 fallback**：当 `findToolByName` 找不到工具时，除了回退到 `toolName`，还可以优先使用 `activity.activityDescription`（如果存在），这样即使工具被卸载，用户仍能看到预计算的人类可读描述。
4. **统一工具活动渲染**：`renderToolActivity` 与 `Spinner` 组件中可能存在的类似逻辑（如 `getActivityDescription`）可以考虑合并为一个统一的 `formatToolActivity` 工具函数，减少概念重复。
