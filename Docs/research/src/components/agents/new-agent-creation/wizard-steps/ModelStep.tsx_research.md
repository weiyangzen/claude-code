# ModelStep.tsx 研究文档

## 场景与职责

`ModelStep.tsx` 是 Claude Code"创建新 Agent"向导中的模型选择步骤。它允许用户为新创建的 Agent 指定要使用的 AI 模型，例如 Sonnet、Opus、Haiku，或者选择 "Inherit from parent" 以继承主对话的模型配置。模型选择直接影响 Agent 的推理能力、响应速度和成本。

该组件在向导流程中位于 `ToolsStep` 之后、`ColorStep` 之前（索引 7）。它本身不直接渲染选项列表，而是将具体的模型选择 UI 委托给 `ModelSelector` 组件，自己仅负责向导状态管理和流程推进。

## 功能点目的

1. **模型选择入口**：作为向导的一个步骤节点，提供模型选择的交互界面。
2. **状态同步**：将用户选择的模型值通过 `updateWizardData` 写入 `wizardData.selectedModel`，然后触发 `goNext()` 进入下一步。
3. **流程导航**：支持通过 `Esc`（`confirm:no`）返回上一步（`ToolsStep`）。
4. **默认值传递**：将当前已选中的模型（如果用户之前已经选择过并回退）作为 `initialModel` 传递给 `ModelSelector`，保证状态不丢失。

## 具体技术实现

### 关键流程

1. **读取上下文**：通过 `useWizard<AgentWizardData>()` 获取 `goNext`、`goBack`、`updateWizardData`、`wizardData`。
2. **定义完成回调 `handleComplete`**：
   - 接收 `model?: string` 参数；
   - 调用 `updateWizardData({ selectedModel: model })`；
   - 调用 `goNext()` 前进到 `ColorStep`。
3. **渲染 `WizardDialogLayout`**：
   - `subtitle="Select model"`
   - `footerText` 包含键盘快捷键提示（↑/↓ 导航、Enter 选择、Esc 返回）
   - 内部渲染 `ModelSelector`，传入 `initialModel={wizardData.selectedModel}`、`onComplete={handleComplete}`、`onCancel={goBack}`。

### 数据结构

`ModelStep` 本身没有复杂的数据结构，主要依赖 `AgentWizardData` 中的 `selectedModel?: string` 字段。

### React Compiler 缓存

编译后的代码使用 memo cache 缓存了：
- `handleComplete` 回调（依赖 `goNext` 和 `updateWizardData`）
- 底部 `Byline` 快捷键提示
- `WizardDialogLayout` + `ModelSelector` 的完整 JSX

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/agents/new-agent-creation/wizard-steps/ModelStep.tsx` | 本文件，ModelStep 组件 |
| `src/components/agents/new-agent-creation/CreateAgentWizard.tsx` | 调用方，将 `ModelStep` 放在 `steps` 数组的第 7 位 |
| `src/components/agents/ModelSelector.tsx` | 被调用方，实际负责模型选项的渲染、键盘导航和选择逻辑 |
| `src/components/wizard/WizardProvider.tsx` | 提供向导状态与导航 API |
| `src/components/wizard/useWizard.ts` | `useWizard` Hook |
| `src/components/wizard/WizardDialogLayout.tsx` | 向导布局壳 |
| `src/utils/model/agent.ts` | `ModelSelector` 内部调用 `getAgentModelOptions()` 的来源 |
| `src/components/design-system/Byline.tsx` | 底部提示行布局 |
| `src/components/design-system/KeyboardShortcutHint.tsx` | 快捷键提示 |
| `src/components/ConfigurableShortcutHint.tsx` | 可配置快捷键提示 |

## 依赖与外部交互

### 上游依赖

- **`CreateAgentWizard.tsx`**：将 `ModelStep` 作为 `steps[7]` 注入向导流程。
- **`ToolsStep.tsx`**：在向导顺序中位于 `ModelStep` 之前，用户按 `Esc` 会从 `ModelStep` 返回到 `ToolsStep`。

### 下游依赖

- **`ModelSelector.tsx`**（`src/components/agents/ModelSelector.tsx`）：
  - 调用 `getAgentModelOptions()` 获取基础选项列表（`sonnet`、`opus`、`haiku`、`inherit`）；
  - 如果 `initialModel` 是一个不在基础列表中的自定义模型 ID（如 `claude-opus-4-5`），会将其作为第一个选项注入，标签显示为 "Current model (custom ID)"；
  - 默认选中 `initialModel ?? "sonnet"`；
  - 使用 `src/components/CustomSelect/select.tsx` 作为底层选择器。
- **`ColorStep.tsx`**：`ModelStep` 的下一步，用户选择模型后进入此处。

### 横向依赖

- **`getAgentModelOptions()`**（`src/utils/model/agent.ts`）：定义了 Agent 可用的模型别名选项及其描述文本。该函数返回：
  - `sonnet`：Balanced performance - best for most agents
  - `opus`：Most capable for complex reasoning tasks
  - `haiku`：Fast and efficient for simple tasks
  - `inherit`：Use the same model as the main conversation

## 风险、边界与改进建议

### 风险与边界

1. **薄封装层**：`ModelStep` 几乎只是一个状态传递层，90% 的交互逻辑都在 `ModelSelector` 中。如果 `ModelSelector` 的接口发生变化（如新增必填 prop），`ModelStep` 需要同步修改。
2. **模型选项与后端不一致**：`getAgentModelOptions()` 返回的是前端硬编码的别名列表。如果后端新增了新的模型别名或废弃了旧别名，前端需要手动更新此函数，否则用户可能无法选择新模型或看到已废弃的选项。
3. **自定义模型 ID 的显示逻辑在 `ModelSelector`**：虽然这是合理的设计，但意味着 `ModelStep` 对"为什么界面上多了一个选项"没有直接控制权，调试时需要深入 `ModelSelector`。
4. **无模型能力说明的扩展空间**：当前每个选项只有一行 description。如果未来需要展示更详细的模型能力对比（如 token 上限、成本估算），需要修改 `ModelSelector` 和 `getAgentModelOptions` 的数据结构。
5. **缺少模型验证**：`ModelStep` 允许 `model` 为 `undefined`（用户直接按 Esc 取消选择时，`ModelSelector` 的 `onCancel` 会被触发，不会调用 `onComplete`）。但如果 `ModelSelector` 内部允许提交 `undefined`，`ModelStep` 也会接受并写入状态。

### 改进建议

1. **合并 ModelStep 与 ModelSelector**：考虑到 `ModelStep` 的逻辑极其简单，可以考虑将 `ModelSelector` 直接作为向导步骤使用，或者将 `ModelStep` 的职责进一步下放到 `ModelSelector`（如让 `ModelSelector` 自己感知向导上下文）。不过当前的分层也有好处：解耦 UI 组件与向导流程。
2. **动态拉取模型列表**：从服务端或配置中心动态获取可用的模型别名列表，而不是在前端硬编码，减少发布依赖。
3. **增加模型详情预览**：在选择界面中增加一个可展开的详情区域，展示当前选中模型的关键参数（如上下文长度、相对成本），帮助用户做出更明智的选择。
4. **统一步骤索引常量**：如 `MethodStep` 文档中所述，所有硬编码的 `goToStep(n)` 和步骤顺序都应该使用共享常量，避免维护风险。
