# PromptStep.tsx 研究文档

## 场景与职责

`PromptStep.tsx` 是 Claude Code"创建新 Agent"向导中的系统提示词（System Prompt）输入步骤。它提供一个多行文本输入界面，让用户输入定义 Agent 行为的核心系统提示词。这是 Agent 配置中最关键的内容之一，直接决定了 Agent 的角色、能力和响应风格。

该组件在手动配置流程中位于 `TypeStep`（索引 3）之后、`DescriptionStep`（索引 5）之前；在自动生成流程中，用户不会经过此步骤（因为提示词由 AI 生成）。

## 功能点目的

1. **系统提示词输入**：提供一个支持键盘输入、光标移动、多行文本的输入框。
2. **外部编辑器支持**：允许用户通过快捷键（默认 `ctrl+g`，可配置为 `chat:externalEditor`）打开系统编辑器（如 VS Code、vim）来编辑提示词。
3. **必填验证**：用户提交时检查提示词是否为空（`trim()` 后长度是否为 0），如果为空则显示错误信息并阻止前进。
4. **状态同步**：将验证通过的提示词写入 `wizardData.systemPrompt`，然后进入下一步。
5. **返回上一步**：支持 `Esc` 返回 `TypeStep`。

## 具体技术实现

### 关键流程

1. **初始化状态**：
   - `systemPrompt`：从 `wizardData.systemPrompt` 初始化，如果不存在则为空字符串 `""`；
   - `cursorOffset`：初始化为 `systemPrompt.length`，将光标放在文本末尾；
   - `error`：初始为 `null`。
2. **键盘绑定**：
   - `useKeybinding("confirm:no", goBack, { context: "Settings" })`：Esc 返回上一步；
   - `useKeybinding("chat:externalEditor", handleExternalEditor, { context: "Chat" })`：触发外部编辑器。
3. **外部编辑器逻辑（`handleExternalEditor`）**：
   - 异步调用 `editPromptInEditor(systemPrompt)`；
   - 如果返回的 `result.content !== null`，则更新 `systemPrompt` 并将 `cursorOffset` 设为新内容长度。
4. **提交逻辑（`handleSubmit`）**：
   - 对 `systemPrompt` 执行 `trim()`；
   - 如果为空，设置 `error = "System prompt is required"` 并直接 `return`；
   - 否则清空 `error`，调用 `updateWizardData({ systemPrompt: trimmedPrompt })`，然后 `goNext()`。
5. **渲染**：
   - 使用 `WizardDialogLayout` 包裹内容；
   - 内部使用 `Box flexDirection="column"` 垂直排列：标题文本、提示文本、`TextInput` 输入框、错误信息（如果有）。

### 数据结构

```typescript
const [systemPrompt, setSystemPrompt] = useState(wizardData.systemPrompt || "")
const [cursorOffset, setCursorOffset] = useState(systemPrompt.length)
const [error, setError] = useState<string | null>(null)
```

### `TextInput` 组件配置

```tsx
<TextInput
  value={systemPrompt}
  onChange={setSystemPrompt}
  onSubmit={handleSubmit}
  placeholder="You are a helpful code reviewer who..."
  columns={80}
  cursorOffset={cursorOffset}
  onChangeCursorOffset={setCursorOffset}
  focus={true}
  showCursor={true}
/>
```

`columns={80}` 设置了输入框的显示宽度为 80 列；`focus={true}` 和 `showCursor={true}` 确保组件挂载后立即获得焦点并显示光标。

### 外部编辑器调用链

`editPromptInEditor`（`src/utils/promptEditor.ts`）的工作流程：
1. 生成临时文件路径；
2. 将当前提示词写入临时文件；
3. 调用 `editFileInEditor`：
   - 检测外部编辑器类型（GUI 编辑器如 VS Code，或终端编辑器如 vim）；
   - 对于终端编辑器，调用 Ink 实例的 `enterAlternateScreen()` 进入全屏编辑模式；
   - 对于 GUI 编辑器，调用 `pause()` 和 `suspendStdin()` 暂停 TUI；
   - 使用 `execSync_DEPRECATED` 同步执行编辑器命令；
   - 读取编辑后的文件内容；
   - 恢复 Ink 的渲染和输入监听；
4. 清理临时文件；
5. 返回编辑后的内容。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/agents/new-agent-creation/wizard-steps/PromptStep.tsx` | 本文件 |
| `src/components/agents/new-agent-creation/CreateAgentWizard.tsx` | 调用方，编排向导步骤 |
| `src/components/wizard/WizardProvider.tsx` | 提供向导状态与导航 API |
| `src/components/wizard/useWizard.ts` | `useWizard` Hook |
| `src/components/TextInput.tsx` | 封装了 `useTextInput` 和 `BaseTextInput` 的文本输入组件 |
| `src/utils/promptEditor.ts` | 提供 `editPromptInEditor`，负责临时文件和外部编辑器调用 |
| `src/utils/editor.ts` | `editFileInEditor` 内部调用，获取用户配置的外部编辑器 |
| `src/keybindings/useKeybinding.ts` | `useKeybinding` Hook 实现 |
| `src/components/wizard/WizardDialogLayout.tsx` | 向导布局壳 |

## 依赖与外部交互

### 上游依赖

- **`TypeStep.tsx`**：在手动流程中，`PromptStep` 的前一步。用户在此输入了 `agentType`。
- **`CreateAgentWizard.tsx`**：将 `PromptStep` 放在 `steps[4]` 的位置。

### 下游依赖

- **`DescriptionStep.tsx`**：`PromptStep` 的下一步，用户输入系统提示词后进入此处填写 Agent 描述。
- **`MemoryStep.tsx`**：读取 `wizardData.systemPrompt` 来构造 `getSystemPrompt` 闭包。如果用户在 `PromptStep` 中修改了系统提示词，`MemoryStep` 会使用最新值。
- **`ConfirmStep.tsx`**：最终确认时展示 `systemPrompt` 的内容。

### 外部系统交互

- **文件系统**：`editPromptInEditor` 会创建临时文件、写入内容、调用外部编辑器、读取内容、删除临时文件。
- **子进程**：`execSync_DEPRECATED` 会同步启动用户配置的外部编辑器进程（如 `code -w`、`vim` 等）。
- **Ink TUI 运行时**：编辑期间会暂停或进入 alternate screen，涉及 `src/ink/instances.js` 中的全局 Ink 实例操作。

## 风险、边界与改进建议

### 风险与边界

1. **同步阻塞外部编辑器**：`editFileInEditor` 使用 `execSync_DEPRECATED` 同步执行编辑器命令。如果用户选择的编辑器配置错误或无法启动（如 `code` 不在 PATH 中），整个 TUI 会卡住直到超时，且没有友好的错误提示。
2. **临时文件清理的可靠性**：虽然代码在 `finally` 块中尝试删除临时文件，但如果进程被强制终止（如 SIGKILL），临时文件可能残留。
3. **验证过于简单**：仅检查 `trim()` 后是否为空，没有检查长度上限、敏感词、或提示词质量。过短的提示词可能导致 Agent 行为不稳定。
4. **无撤销/重做支持**：`TextInput` 组件虽然支持基础的历史记录（通过 `useTextInput`），但在 `PromptStep` 中没有暴露 `ctrl+z` 的撤销功能，用户误删大量文本后难以恢复。
5. **光标位置重置问题**：从外部编辑器返回后，`cursorOffset` 被强制设为文本末尾（`result.content.length`），而不是保留用户在外部编辑器中的光标位置。虽然外部编辑器无法直接传回光标位置，但这是一个体验上的损失。
6. **React Compiler 编译后的复杂性**：与项目中其他组件一样，代码已被 React Compiler 编译，包含大量 memo cache 逻辑，手动修改时需要特别小心。

### 改进建议

1. **增强外部编辑器错误处理**：在 `editFileInEditor` 或 `PromptStep` 中捕获编辑器启动失败的情况，给出明确的错误提示（如"无法启动 VS Code，请检查 `code` 是否在 PATH 中"），并提供重试或继续在当前界面编辑的选项。
2. **增加提示词质量检查**：除了非空检查外，可以增加最小长度警告（如少于 50 字符时提示"提示词可能过短"）、或基于简单启发式的质量评分。
3. **支持模板/示例**：在输入框下方提供几个常用 Agent 角色的快速插入模板（如 "Code Reviewer"、"Test Writer"、"Documentation Assistant"），降低用户的创作门槛。
4. **保留历史草稿**：如果用户在此步骤花费较长时间输入，可以定期将草稿自动保存到本地状态或临时文件中，防止意外退出向导导致内容丢失。
5. **改进光标恢复**：虽然外部编辑器不返回光标位置，但可以考虑在打开编辑器前记录当前光标位置，返回后恢复该位置，而不是总是放到末尾。
6. **增加字数/字符数统计**：在界面底部显示当前提示词的字符数，帮助用户把握长度。
