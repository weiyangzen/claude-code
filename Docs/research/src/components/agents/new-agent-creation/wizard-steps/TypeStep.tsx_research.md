# TypeStep.tsx 研究文档

## 场景与职责

`TypeStep.tsx` 是 Claude Code"创建新 Agent"向导中的 Agent 标识符（`agentType`）输入步骤。它要求用户输入一个唯一的、符合命名规范的标识符，用于在后续对话中通过 `@agent-type` 的方式调用该 Agent。这个标识符也是 Agent 配置文件名和内部索引的关键字段。

该组件在手动配置流程中位于 `MethodStep` 之后（索引 3），在自动生成流程中同样位于 `GenerateStep` 之后（索引 3）。无论哪种创建方式，用户都需要经过此步骤来指定或确认 Agent 的标识符。

## 功能点目的

1. **标识符输入**：提供一个单行文本输入框，让用户输入 Agent 的类型名称（如 `test-runner`、`tech-lead`）。
2. **实时验证**：在用户提交时调用 `validateAgentType` 检查输入是否符合命名规范（非空、长度 3-50、仅包含字母数字和连字符、首尾必须是字母数字）。
3. **错误提示**：如果验证失败，在输入框下方显示具体的错误信息，并阻止向导前进。
4. **状态同步**：验证通过后，将 `trim()` 后的值写入 `wizardData.agentType`，然后进入下一步。
5. **返回上一步**：支持 `Esc` 返回上一步。

## 具体技术实现

### 关键流程

1. **初始化状态**：
   - `agentType`：从 `wizardData.agentType` 初始化，默认为 `""`；
   - `error`：初始为 `null`；
   - `cursorOffset`：初始化为 `agentType.length`，光标放在末尾。
2. **键盘绑定**：
   - `useKeybinding("confirm:no", goBack, { context: "Settings" })`：Esc 返回上一步。
3. **提交处理（`handleSubmit`）**：
   - 对输入值执行 `trim()`；
   - 调用 `validateAgentType(trimmedValue)`；
   - 如果返回错误字符串，设置 `error` 状态并阻止前进；
   - 否则清空 `error`，调用 `updateWizardData({ agentType: trimmedValue })`，然后 `goNext()`。
4. **渲染**：
   - `WizardDialogLayout` 包裹内容，`subtitle="Agent type (identifier)"`；
   - 内部垂直排列：提示文本、输入框、错误信息；
   - 底部 `Byline` 显示快捷键提示（Type 输入、Enter 继续、Esc 返回）。

### 数据结构

```typescript
type Props = {
  existingAgents: AgentDefinition[];
};

const [agentType, setAgentType] = useState(wizardData.agentType || "")
const [error, setError] = useState<string | null>(null)
const [cursorOffset, setCursorOffset] = useState(agentType.length)
```

注意：`existingAgents` prop 虽然在 `Props` 类型中定义了，但在当前编译后的代码中，组件函数签名是 `TypeStep(_props)`，而函数体中**没有使用** `_props` 或 `existingAgents`。这意味着 `existingAgents` 目前没有被用于实时查重，重复名称的检查被延迟到了最终的 `ConfirmStep`（通过 `validateAgent` 进行）。

### 验证规则

`validateAgentType`（`src/components/agents/validateAgent.ts`）的规则：
1. 不能为空；
2. 必须以字母数字开头和结尾；
3. 只能包含字母、数字和连字符（`[a-zA-Z0-9][a-zA-Z0-9-]*[a-zA-Z0-9]`）；
4. 长度至少 3 个字符；
5. 长度不能超过 50 个字符。

### `TextInput` 组件配置

```tsx
<TextInput
  value={agentType}
  onChange={setAgentType}
  onSubmit={handleSubmit}
  placeholder="e.g., test-runner, tech-lead, etc"
  columns={60}
  cursorOffset={cursorOffset}
  onChangeCursorOffset={setCursorOffset}
  focus={true}
  showCursor={true}
/>
```

`columns={60}` 设置了输入框宽度为 60 列；`placeholder` 提供了命名示例。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/agents/new-agent-creation/wizard-steps/TypeStep.tsx` | 本文件 |
| `src/components/agents/new-agent-creation/CreateAgentWizard.tsx` | 调用方，将 `TypeStep` 注入 `steps` 数组，并传入 `existingAgents` prop |
| `src/components/wizard/WizardProvider.tsx` | 提供向导状态与导航 API |
| `src/components/wizard/useWizard.ts` | `useWizard` Hook |
| `src/components/TextInput.tsx` | 文本输入组件 |
| `src/components/agents/validateAgent.ts` | 提供 `validateAgentType` 验证函数 |
| `src/tools/AgentTool/loadAgentsDir.ts` | `AgentDefinition` 类型定义 |
| `src/keybindings/useKeybinding.ts` | `useKeybinding` Hook 实现 |
| `src/components/wizard/WizardDialogLayout.tsx` | 向导布局壳 |
| `src/components/design-system/Byline.tsx` | 底部提示行布局 |
| `src/components/design-system/KeyboardShortcutHint.tsx` | 快捷键提示 |
| `src/components/ConfigurableShortcutHint.tsx` | 可配置快捷键提示 |

## 依赖与外部交互

### 上游依赖

- **`CreateAgentWizard.tsx`**：将 `TypeStep` 作为动态组件 `t1` 注入 `steps` 数组（`() => <TypeStep existingAgents={existingAgents} />`），位于索引 3。
- **`MethodStep.tsx`** / **`GenerateStep.tsx`**：`TypeStep` 的前一步。在手动流程中是 `MethodStep`，在自动生成流程中是 `GenerateStep`。

### 下游依赖

- **`PromptStep.tsx`**（手动流程）/ **`DescriptionStep.tsx`**（自动生成流程）：`TypeStep` 的下一步。用户输入标识符后进入此处。
- **`MemoryStep.tsx`**：读取 `wizardData.finalAgent?.agentType` 来加载记忆提示。`agentType` 是 `TypeStep` 写入的关键字段之一。
- **`ConfirmStepWrapper.tsx`** / **`ConfirmStep.tsx`**：最终确认时展示 `agentType`，并调用 `validateAgent` 进行完整验证（包括与 `existingAgents` 的重复检查）。

### 横向依赖

- **`validateAgent.ts`**：
  - `validateAgentType` 被 `TypeStep` 用于实时前端验证；
  - `validateAgent` 被 `ConfirmStep` 用于最终后端式验证，检查是否与现有 Agent 重复、工具是否有效、系统提示词长度等。

## 风险、边界与改进建议

### 风险与边界

1. **`existingAgents` prop 未被使用**：`TypeStep` 接收了 `existingAgents` 但在组件内部完全没有使用它。这意味着用户在输入标识符时，即使输入了一个已存在的名称，也不会得到实时反馈，只有到了最终确认步骤才会报错。这是一个明显的功能缺口。
2. **验证与提交耦合**：`handleSubmit` 同时负责验证和状态更新。虽然当前逻辑简单，但如果未来需要异步查重（如向服务端查询名称可用性），同步的 `handleSubmit` 架构需要重构。
3. **无命名空间支持**：当前正则表达式不支持冒号、斜杠等特殊字符，这意味着插件化的命名空间 Agent（如 `my-plugin:my-agent`）在标识符层面不被支持。虽然 `agentMemory.ts` 中有 `sanitizeAgentTypeForPath` 处理冒号，但输入验证阶段就已经拒绝了冒号。
4. **光标位置管理较原始**：`cursorOffset` 完全由 `TextInput` 的 `onChangeCursorOffset` 回调驱动，组件本身没有更智能的光标控制逻辑（如提交错误后自动高亮非法字符）。
5. **缺少输入建议/自动补全**：用户需要从零开始想一个标识符，系统没有基于项目上下文或已有 Agent 名称提供命名建议。

### 改进建议

1. **实时查重**：在 `TypeStep` 中利用 `existingAgents` prop 进行实时重复检查。可以在 `handleSubmit` 中增加：
   ```typescript
   const duplicate = existingAgents.find(a => a.agentType === trimmedValue)
   if (duplicate) {
     setError(`Agent type "${trimmedValue}" already exists in ${getAgentSourceDisplayName(duplicate.source)}`)
     return
   }
   ```
   这样可以将最终确认阶段的错误提前到输入阶段，显著改善用户体验。
2. **异步名称可用性检查**：如果未来 Agent 名称需要全局唯一（如跨团队共享），可以在用户输入停止后（debounce）发起异步可用性查询。
3. **放宽或扩展命名规则**：如果产品需要支持命名空间（如 `plugin:agent-name`），需要同步修改 `validateAgentType` 的正则表达式，并确保 `sanitizeAgentTypeForPath` 能正确处理文件系统路径映射。
4. **提供命名建议**：基于项目结构、已有 Agent 名称或 AI 生成，为用户提供几个候选标识符（如 `code-reviewer`、`bug-finder`、`api-designer`），用户可以通过 Tab 键快速切换或选择。
5. **输入即时反馈**：在用户输入过程中（而不仅是提交时）进行轻量级验证，例如当输入包含非法字符时立即变红或显示提示，而不是等到按 Enter 才报错。
6. **与 `GenerateStep` 的联动**：在自动生成流程中，AI 可能已经生成了一个建议的 `agentType`。`TypeStep` 应该预填充这个建议值，并允许用户修改，而不是总是从空字符串开始。
