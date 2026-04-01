# ToolsStep.tsx 研究文档

## 场景与职责

`ToolsStep.tsx` 是 Claude Code"创建新 Agent"向导中的工具选择步骤。它允许用户为新 Agent 配置可用的工具（Tools）集合。工具决定了 Agent 能执行哪些操作，例如读取文件、编辑代码、执行 Bash 命令、搜索网络等。用户可以选择"所有工具"（默认），也可以精确指定允许使用的工具子集。

该组件在向导流程中位于 `DescriptionStep` 之后、`ModelStep` 之前（索引 6）。与 `ModelStep` 类似，`ToolsStep` 本身不直接渲染复杂的工具选择 UI，而是将交互逻辑委托给 `ToolSelector` 组件，自己仅负责向导状态衔接。

## 功能点目的

1. **工具选择入口**：作为向导步骤节点，展示工具选择界面。
2. **状态同步**：将用户选择的工具列表通过 `updateWizardData` 写入 `wizardData.selectedTools`，然后触发 `goNext()`。
3. **默认值传递**：将当前已选中的工具列表（`wizardData.selectedTools`）作为 `initialTools` 传递给 `ToolSelector`，支持用户回退后保留之前的选择。
4. **流程导航**：支持通过 `Esc` 返回上一步（`DescriptionStep`）。

## 具体技术实现

### 关键流程

1. **读取上下文**：通过 `useWizard<AgentWizardData>()` 获取 `goNext`、`goBack`、`updateWizardData`、`wizardData`。
2. **接收外部 props**：`ToolsStep` 接收 `tools: Tools` prop（来自 `CreateAgentWizard` 的注入），并将其原样传递给 `ToolSelector`。
3. **定义完成回调 `handleComplete`**：
   - 接收 `selectedTools: string[] | undefined`；
   - 调用 `updateWizardData({ selectedTools })`；
   - 调用 `goNext()` 前进到 `ModelStep`。
4. **渲染 `WizardDialogLayout`**：
   - `subtitle="Select tools"`
   - `footerText` 包含键盘快捷键提示（Enter 切换选择、↑/↓ 导航、Esc 返回）
   - 内部渲染 `ToolSelector`，传入 `tools`、`initialTools={wizardData.selectedTools}`、`onComplete={handleComplete}`、`onCancel={goBack}`。

### 数据结构

```typescript
type Props = {
  tools: Tools;
};
```

`Tools` 类型来自 `src/Tool.js`，通常是一个 `Tool[]` 数组或类似的集合类型。

`wizardData.selectedTools` 的类型是 `string[] | undefined`：
- `undefined` 表示"使用所有工具"（默认语义）；
- `string[]` 表示显式选中的工具名称列表。

### React Compiler 缓存

编译后的代码缓存了：
- `handleComplete` 回调（依赖 `goNext` 和 `updateWizardData`）
- 底部 `Byline` 快捷键提示
- `WizardDialogLayout` + `ToolSelector` 的完整 JSX

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/agents/new-agent-creation/wizard-steps/ToolsStep.tsx` | 本文件，ToolsStep 组件 |
| `src/components/agents/new-agent-creation/CreateAgentWizard.tsx` | 调用方，将 `ToolsStep` 放在 `steps` 数组的第 6 位，并传入 `tools` prop |
| `src/components/agents/ToolSelector.tsx` | 被调用方，负责实际的工具分类、选择、键盘导航和渲染逻辑 |
| `src/components/wizard/WizardProvider.tsx` | 提供向导状态与导航 API |
| `src/components/wizard/useWizard.ts` | `useWizard` Hook |
| `src/components/wizard/WizardDialogLayout.tsx` | 向导布局壳 |
| `src/Tool.js` | `Tools` 类型定义 |
| `src/tools/AgentTool/agentToolUtils.ts` | `ToolSelector` 内部调用 `filterToolsForAgent` 和 `resolveAgentTools` |
| `src/components/design-system/Byline.tsx` | 底部提示行布局 |
| `src/components/design-system/KeyboardShortcutHint.tsx` | 快捷键提示 |
| `src/components/ConfigurableShortcutHint.tsx` | 可配置快捷键提示 |

## 依赖与外部交互

### 上游依赖

- **`CreateAgentWizard.tsx`**：将 `ToolsStep` 作为 `steps[6]` 注入向导流程，并从上层接收 `tools` prop 后传递给 `ToolsStep`。
- **`DescriptionStep.tsx`**：在向导顺序中位于 `ToolsStep` 之前，用户按 `Esc` 会返回到 `DescriptionStep`。

### 下游依赖

- **`ToolSelector.tsx`**（`src/components/agents/ToolSelector.tsx`）：
  - 调用 `filterToolsForAgent({ tools, isBuiltIn: false, isAsync: false })` 过滤出可供自定义 Agent 使用的工具；
  - 将工具按功能分类为 Read-only、Edit、Execution、MCP、Other 五个桶（bucket）；
  - 支持按桶全选/全不选、按 MCP Server 全选/全不选、按单个工具切换；
  - 如果用户选择了所有可用工具，则向 `onComplete` 传递 `undefined`（保留"所有工具"语义）；
  - 否则传递选中的工具名称数组。
- **`ModelStep.tsx`**：`ToolsStep` 的下一步。
- **`ConfirmStep.tsx`**：最终确认时展示 `selectedTools`，并通过 `validateAgent` 验证工具列表的有效性。

### 横向依赖

- **`validateAgent.ts`**（`src/components/agents/validateAgent.ts`）：
  - 在最终确认阶段检查 `agent.tools` 是否为数组；
  - 如果 `tools === undefined`，给出警告 "Agent has access to all tools"；
  - 如果 `tools.length === 0`，给出警告 "No tools selected - agent will have very limited capabilities"；
  - 调用 `resolveAgentTools` 检查是否有无效工具名。

## 风险、边界与改进建议

### 风险与边界

1. **`undefined` 语义的特殊性**：`selectedTools` 为 `undefined` 时表示"所有工具"，这个语义在 `ToolSelector` 内部被转换，但在 `ToolsStep` 中只是透明传递。如果未来有其他组件错误地将空数组 `[]` 和 `undefined` 混用，会导致 Agent 实际可用工具范围出现歧义。
2. **`ToolSelector` 的复杂性**：`ToolSelector` 是一个超过 500 行的大型组件，包含大量状态管理（`selectedTools`、`focusIndex`、`showIndividualTools`）和键盘事件处理。`ToolsStep` 作为其父组件，对其内部行为几乎没有控制力，调试问题需要深入 `ToolSelector`。
3. **工具列表的动态性**：`tools` prop 来自上层，可能包含 MCP 动态加载的工具。如果 MCP 工具在向导打开后发生变化（如 MCP server 断开），`ToolSelector` 中显示的工具列表不会自动更新，因为 `ToolsStep` 和 `CreateAgentWizard` 在挂载时就已经固定了 `tools`。
4. **无工具能力说明**：`ToolSelector` 主要显示工具名称，没有展示每个工具的具体功能说明，新用户可能不清楚某些工具的作用。
5. **MCP 工具名称可读性**：MCP 工具在内部以 `mcp__<serverName>__<toolName>` 的格式命名，`ToolSelector` 会将其解析为 `<toolName> (<serverName>)` 显示，但这个转换逻辑在 `ToolSelector` 内部，如果格式约定改变，需要同步修改。

### 改进建议

1. **明确化"所有工具"语义**：考虑引入一个显式的常量或枚举（如 `ALL_TOOLS = '*'`）来代替 `undefined`，使代码意图更清晰，减少空值相关的 bug。
2. **拆分 `ToolSelector`**：`ToolSelector` 目前承担了太多职责（分类、导航、渲染、状态管理）。可以将其拆分为 `ToolBuckets`、`ToolList`、`ToolSelectionState` 等更小、更专注的组件或 Hook。
3. **实时同步工具列表**：如果技术上可行，可以让 `ToolsStep` 订阅工具集合的变化（如 MCP server 的在线状态），在变化时重新渲染 `ToolSelector` 并给出提示。
4. **增加工具提示/帮助**：在 `ToolSelector` 中为每个工具增加一个帮助入口（如按 `?` 键显示工具描述），帮助用户理解每个工具的作用。
5. **智能默认选择**：根据 Agent 的类型或描述，使用 AI 推荐一组默认工具，减少用户手动选择的工作量。例如，一个"代码审查"Agent 可能默认只需要 `FileReadTool`、`GrepTool` 和 `BashTool`。
6. **与 `validateAgent` 的联动**：在 `ToolSelector` 的确认按钮上实时显示验证结果（如"未选择任何工具，Agent 能力将非常有限"），而不是等到最终确认步骤才给出警告。
