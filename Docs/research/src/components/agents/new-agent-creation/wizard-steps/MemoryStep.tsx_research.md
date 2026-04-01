# MemoryStep.tsx 研究文档

## 场景与职责

`MemoryStep.tsx` 是 Claude Code"创建新 Agent"向导中的记忆（Memory）配置步骤。它允许用户为新 Agent 选择持久化记忆的作用域（scope），包括：用户级（`user`）、项目级（`project`）、本地级（`local`）或不启用（`none`）。持久化记忆意味着 Agent 可以在多次调用之间保留学习到的上下文，这些上下文存储在文件系统的 `MEMORY.md` 中。

该步骤在向导流程中的位置是动态的：如果 `isAutoMemoryEnabled()` 返回 true，则 `MemoryStep` 会被 `CreateAgentWizard.tsx` 插入到 `ColorStep` 之后、`ConfirmStepWrapper` 之前；否则整个步骤被跳过。

此外，`MemoryStep` 不仅记录用户的选择，还会根据选择**实时构造** `finalAgent` 对象中的 `getSystemPrompt` 闭包，将记忆提示（memory prompt）动态注入到系统提示词中。

## 功能点目的

1. **记忆作用域选择**：提供 4 个选项（user/project/local/none），并根据用户之前选择的 `location`（在 `LocationStep` 中）调整默认推荐项的顺序。
2. **动态系统提示注入**：当用户选择启用记忆时，构造一个 `getSystemPrompt` 闭包，在原有 `systemPrompt` 基础上追加记忆相关的 prompt 内容（通过 `loadAgentMemoryPrompt` 加载）。
3. **状态更新**：将 `selectedMemory` 和更新后的 `finalAgent` 写入向导全局状态，然后前进到下一步。
4. **返回上一步**：支持 `Esc`（`confirm:no`）返回上一步。

## 具体技术实现

### 关键流程

1. **读取上下文**：通过 `useWizard<AgentWizardData>()` 获取 `goNext`、`goBack`、`updateWizardData`、`wizardData`。
2. **绑定返回快捷键**：使用 `useKeybinding("confirm:no", goBack, { context: "Confirmation" })` 将 Esc 映射为返回。
3. **动态选项排序**：
   - 如果 `wizardData.location === "userSettings"`，则默认推荐 "User scope" 排在第一位；
   - 否则默认推荐 "Project scope" 排在第一位。
4. **选择处理（`handleSelect`）**：
   - 如果值为 `"none"`，则 `memory = undefined`；
   - 否则将值断言为 `AgentMemoryScope`；
   - 读取 `wizardData.finalAgent?.agentType` 和 `wizardData.systemPrompt`；
   - 构造新的 `finalAgent`：展开旧对象，注入 `memory`，并替换 `getSystemPrompt` 为一个返回 `systemPrompt + "\n\n" + loadAgentMemoryPrompt(agentType, memory)` 的闭包；
   - 调用 `updateWizardData({ selectedMemory: memory, finalAgent: ... })`；
   - 调用 `goNext()`。

### 数据结构

```typescript
type MemoryOption = {
  label: string;
  value: AgentMemoryScope | 'none';
};

// 动态生成的选项示例（location === "userSettings" 时）
const memoryOptions = [
  { label: "User scope (~/.claude/agent-memory/) (Recommended)", value: "user" },
  { label: "None (no persistent memory)", value: "none" },
  { label: "Project scope (.claude/agent-memory/)", value: "project" },
  { label: "Local scope (.claude/agent-memory-local/)", value: "local" },
]
```

### `getSystemPrompt` 闭包构造

这是 `MemoryStep` 最核心的技术细节。它在用户做出选择的那一刻，就把记忆提示的加载逻辑**预绑定**到 `finalAgent` 中：

```typescript
getSystemPrompt: isAutoMemoryEnabled() && memory && agentType
  ? () => wizardData.systemPrompt + "\n\n" + loadAgentMemoryPrompt(agentType, memory)
  : () => wizardData.systemPrompt
```

注意这里有两个条件：
- `isAutoMemoryEnabled()`：功能开关；
- `memory && agentType`：必须同时有记忆作用域和已确定的 agentType。

如果条件不满足，闭包直接返回原始的 `systemPrompt`，避免不必要的计算。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/agents/new-agent-creation/wizard-steps/MemoryStep.tsx` | 本文件 |
| `src/components/agents/new-agent-creation/CreateAgentWizard.tsx` | 调用方，条件性将 `MemoryStep` 注入 `steps` 数组 |
| `src/components/wizard/WizardProvider.tsx` | 提供向导状态与导航 API |
| `src/tools/AgentTool/agentMemory.ts` | 提供 `AgentMemoryScope` 类型和 `loadAgentMemoryPrompt` |
| `src/memdir/paths.ts` | 提供 `isAutoMemoryEnabled` 功能开关 |
| `src/keybindings/useKeybinding.ts` | `useKeybinding` Hook 实现 |
| `src/components/CustomSelect/select.tsx` | 底层选择器组件 |
| `src/components/wizard/WizardDialogLayout.tsx` | 向导布局壳 |

## 依赖与外部交互

### 上游依赖

- **`LocationStep.tsx`**：`MemoryStep` 读取 `wizardData.location` 来决定默认推荐的记忆作用域。如果 `LocationStep` 未正确写入 `location`，`MemoryStep` 的推荐逻辑会回退到 `project` 优先。
- **`PromptStep.tsx`**：`MemoryStep` 读取 `wizardData.systemPrompt` 作为构造 `getSystemPrompt` 闭包的基础文本。
- **`TypeStep.tsx`**：`MemoryStep` 读取 `wizardData.finalAgent?.agentType` 来加载对应 agent 的记忆目录。

### 下游依赖

- **`ConfirmStepWrapper.tsx`** / **`ConfirmStep.tsx`**：最终展示和确认 `finalAgent` 的完整定义，包括已注入的 `memory` 和 `getSystemPrompt`。
- **`agentMemory.ts`**：`loadAgentMemoryPrompt` 会同步调用 `ensureMemoryDirExists(memoryDir)`（fire-and-forget 的 `void` 调用），确保记忆目录存在，并返回包含 `scopeNote` 和 `extraGuidelines` 的 prompt 文本。

### 外部系统交互

- **文件系统**：`loadAgentMemoryPrompt` -> `getAgentMemoryDir` -> `join(getCwd(), '.claude', 'agent-memory', ...)` 或 `join(getMemoryBaseDir(), 'agent-memory', ...)`。这意味着选择一旦确认，就会在文件系统上预留对应的目录路径。

## 风险、边界与改进建议

### 风险与边界

1. **同步调用异步 IO**：`loadAgentMemoryPrompt` 内部调用 `ensureMemoryDirExists` 是异步的，但被以 `void` 方式 fire-and-forget。注释明确说明这是因为在 React render 同步上下文中无法使用 `await`。虽然设计上认为"Agent 在第一次 API 往返后才会尝试写入"，但在极端快的交互或测试环境中仍可能存在竞态条件。
2. **闭包捕获旧状态**：`getSystemPrompt` 闭包在 `MemoryStep` 选择时捕获了当时的 `wizardData.systemPrompt`。如果用户在后续步骤（如 `DescriptionStep`）修改了系统提示，但不再经过 `MemoryStep`，则闭包中的 `systemPrompt` 可能是过时的。不过向导流程是线性的，通常不会再修改。
3. **`agentType` 依赖前置步骤**：如果 `finalAgent` 或 `agentType` 尚未生成（例如用户跳过了 `TypeStep`），`loadAgentMemoryPrompt` 不会被调用，记忆功能实际上不会生效。但向导流程保证了 `TypeStep` 在 `MemoryStep` 之前。
4. **硬编码的选项标签文本**：选项中的路径描述（如 `~/.claude/agent-memory/`）是写死的字符串，如果 `getMemoryBaseDir()` 的实际返回值与描述不符，会造成用户困惑。
5. **无记忆内容预览**：用户在选择记忆作用域时，无法看到当前是否已有该 Agent 的历史记忆文件，也无法预览记忆内容。

### 改进建议

1. **延迟闭包构造**：将 `getSystemPrompt` 的构造延迟到向导最终提交时（如 `ConfirmStep` 或 `CreateAgentWizard` 的 `onComplete`），这样可以确保使用最终版本的 `systemPrompt` 和 `agentType`，避免中间状态被固化。
2. **动态路径显示**：使用 `getAgentMemoryDir` 或 `getMemoryScopeDisplay` 动态生成选项标签中的路径文本，确保与实际文件系统路径一致。
3. **记忆状态提示**：在选择界面中增加一行提示，告知用户该作用域下是否已存在 `MEMORY.md` 文件，帮助用户做出更明智的选择。
4. **竞态条件防护**：如果未来 React 支持在 render 中发起真正的异步操作，或者向导架构改为异步步骤，应该将 `ensureMemoryDirExists` 改为显式的 `await`，并在目录创建失败时给出友好提示。
5. **抽象通用 SelectStep**：`MemoryStep` 与 `LocationStep`、`MethodStep` 等高度相似，都是"展示选项 -> 选择 -> 更新状态 -> goNext"，可以抽象为通用组件。
