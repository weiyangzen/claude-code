# MethodStep.tsx 研究文档

## 场景与职责

`MethodStep.tsx` 是 Claude Code"创建新 Agent"向导中的"创建方式"选择步骤。它让用户决定是通过 Claude AI 自动生成 Agent 配置（`generate`），还是手动逐步填写各项参数（`manual`）。这个选择会显著影响后续的向导流程：如果选择自动生成，向导会进入 `GenerateStep` 由 AI 根据用户输入生成配置；如果选择手动配置，则会跳过 `GenerateStep`，直接跳转到第 4 步（`PromptStep`）开始手动填写。

该组件在向导流程中位于 `LocationStep` 之后，是第 2 步（索引 1）。

## 功能点目的

1. **创建方式选择**：提供两个互斥选项——"Generate with Claude (recommended)" 和 "Manual configuration"。
2. **流程分支控制**：根据用户选择，决定后续向导步骤的走向：
   - `generate`：顺序进入 `GenerateStep`（索引 2）。
   - `manual`：直接跳转到 `PromptStep`（索引 3），跳过 `GenerateStep`。
3. **状态标记**：在向导数据中写入 `method` 和 `wasGenerated` 标志，供后续步骤（尤其是 `ConfirmStep`）判断 Agent 是否由 AI 生成。
4. **返回上一步**：支持 `Esc` 返回 `LocationStep`。

## 具体技术实现

### 关键流程

1. **读取上下文**：通过 `useWizard<AgentWizardData>()` 获取 `goNext`、`goBack`、`updateWizardData`、`goToStep`。
2. **定义静态选项**：`methodOptions` 是一个写死的数组，包含 `generate` 和 `manual` 两个选项。
3. **选择处理回调**：
   - 将值断言为 `'generate' | 'manual'`；
   - 调用 `updateWizardData({ method, wasGenerated: method === "generate" })`；
   - 如果 `method === "generate"`，调用 `goNext()`（进入索引 2 的 `GenerateStep`）；
   - 否则调用 `goToStep(3)`（直接跳到索引 3 的 `PromptStep`）。
4. **取消处理**：`onCancel` 映射到 `goBack()`。

### 数据结构

```typescript
const methodOptions = [
  { label: "Generate with Claude (recommended)", value: "generate" },
  { label: "Manual configuration", value: "manual" }
]
```

### 向导状态影响

写入 `wizardData` 的字段：
- `method`: `'generate' | 'manual'`
- `wasGenerated`: `boolean`

这两个字段在 `ConfirmStep` 或最终提交时可能被用来区分生成来源，影响展示文案或后续处理逻辑。

### React Compiler 缓存

与 `LocationStep` 类似，`MethodStep` 也使用了 React Compiler 编译后的 memo cache 模式，缓存了：
- `methodOptions` 静态数组
- 底部 `Byline` 快捷键提示
- `onChange` 和 `onCancel` 回调
- 最终的 `WizardDialogLayout` JSX

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/agents/new-agent-creation/wizard-steps/MethodStep.tsx` | 本文件 |
| `src/components/agents/new-agent-creation/CreateAgentWizard.tsx` | 调用方，编排整个向导步骤顺序 |
| `src/components/wizard/WizardProvider.tsx` | 提供 `goNext`、`goBack`、`goToStep`、`updateWizardData` 等 API |
| `src/components/wizard/useWizard.ts` | `useWizard` Hook |
| `src/components/CustomSelect/select.tsx` | 底层选择器组件 |
| `src/components/wizard/WizardDialogLayout.tsx` | 向导布局壳 |
| `src/components/design-system/Byline.tsx` | 底部提示行布局 |
| `src/components/design-system/KeyboardShortcutHint.tsx` | 快捷键提示 |
| `src/components/ConfigurableShortcutHint.tsx` | 可配置快捷键提示 |

## 依赖与外部交互

### 上游依赖

- **`CreateAgentWizard.tsx`**：将 `MethodStep` 固定放在 `steps[1]` 的位置。`steps` 数组的定义是：`[LocationStep, MethodStep, GenerateStep, TypeStep, PromptStep, DescriptionStep, ToolsStep, ModelStep, ColorStep, ...memorySteps, ConfirmStepWrapper]`。

### 下游依赖

- **`GenerateStep.tsx`**：当用户选择 `generate` 时，向导通过 `goNext()` 进入此步骤。`GenerateStep` 会调用 AI 生成 Agent 配置。
- **`PromptStep.tsx`**：当用户选择 `manual` 时，通过 `goToStep(3)` 直接跳转到这里。注意 `goToStep(3)` 中的 `3` 是硬编码的索引值，对应 `steps` 数组中 `PromptStep` 的位置（`LocationStep[0], MethodStep[1], GenerateStep[2], TypeStep[3]`... 等等，让我重新核对）。

**重要发现**：根据 `CreateAgentWizard.tsx` 中的 `steps` 数组定义：
```typescript
steps = [LocationStep, MethodStep, GenerateStep, t1(TypeStep), PromptStep, DescriptionStep, t2(ToolsStep), ModelStep, ColorStep, ...t3(memorySteps), t4(ConfirmStepWrapper)]
```

所以索引对应关系是：
- 0: LocationStep
- 1: MethodStep
- 2: GenerateStep
- 3: TypeStep
- 4: PromptStep
- 5: DescriptionStep
- 6: ToolsStep
- 7: ModelStep
- 8: ColorStep
- 9: MemoryStep（如果启用）
- 10: ConfirmStepWrapper

但 `MethodStep` 中 `goToStep(3)` 会跳到索引 3，即 `TypeStep`。这意味着选择 manual 后，用户会先填写 `TypeStep`（Agent type/identifier），然后顺序进入 `PromptStep`。这与 `goToStep(3)` 的代码逻辑一致。

### 横向依赖

- **`ConfirmStepWrapper.tsx`** / **`ConfirmStep.tsx`**：可能读取 `wizardData.wasGenerated` 来展示不同的确认信息或执行不同的验证逻辑。

## 风险、边界与改进建议

### 风险与边界

1. **硬编码的索引跳转**：`goToStep(3)` 是一个魔法数字，它与 `CreateAgentWizard.tsx` 中 `steps` 数组的顺序强耦合。如果有人在 `CreateAgentWizard.tsx` 中调整了步骤顺序（例如在 `MethodStep` 和 `GenerateStep` 之间插入新步骤），`MethodStep` 的跳转逻辑就会出错，导致用户被跳到错误的步骤。
2. **无生成能力检测**：如果当前环境无法连接 Claude API（如离线模式），"Generate with Claude" 选项仍然可用，用户选择后可能会在 `GenerateStep` 中遇到失败。
3. **推荐标签硬编码**：`(recommended)` 文本是写死在 `label` 中的，没有根据用户历史行为或环境条件动态调整。
4. **状态冗余**：`wasGenerated` 本质上是 `method === "generate"` 的派生值，却被持久化到状态中。如果未来允许在向导中途修改 `method`，需要同步更新 `wasGenerated`，否则会出现不一致。

### 改进建议

1. **消除魔法数字**：在 `CreateAgentWizard.tsx` 或一个共享常量文件中定义步骤枚举（如 `StepIndex = { TYPE: 3, PROMPT: 4, ... }`），让 `MethodStep` 引用常量而不是硬编码 `3`。
2. **动态禁用生成选项**：在渲染前检测当前是否具备 AI 生成条件（如 API 可用性、网络状态），如果不具备，将 "Generate with Claude" 选项设为 `disabled`，并附带提示说明。
3. **延迟计算 `wasGenerated`**：不在 `MethodStep` 中写入 `wasGenerated`，而是在 `ConfirmStep` 或最终提交时根据 `method` 字段动态计算，减少状态冗余。
4. **提取通用 SelectStep**：`MethodStep`、`LocationStep`、`MemoryStep` 等遵循几乎完全相同的模式，可以抽象为一个接受 `options`、`onSelect`、`subtitle` 等参数的通用组件，减少重复代码和维护成本。
