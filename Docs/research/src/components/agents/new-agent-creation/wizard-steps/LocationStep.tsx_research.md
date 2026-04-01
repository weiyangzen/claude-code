# LocationStep.tsx 研究文档

## 场景与职责

`LocationStep.tsx` 是 Claude Code 中"创建新 Agent"向导（CreateAgentWizard）的第一个步骤组件。它负责让用户选择新创建的 Agent 配置文件将要保存的位置：是保存在当前项目目录下（`.claude/agents/`），还是保存在用户主目录下（`~/.claude/agents/`）。这个选择直接决定了 Agent 的可见范围——项目级 Agent 仅对当前工作目录可用，而用户级 Agent 在所有项目中都可以被调用。

该组件在向导流程中的位置是第 1 步（索引 0），由 `CreateAgentWizard.tsx` 的 `steps` 数组硬编码注入。用户在此步骤做出的选择会写入向导全局状态 `wizardData.location`，供后续步骤（如 `MemoryStep`）根据位置动态调整推荐选项。

## 功能点目的

1. **位置选择**：提供一个二选一的界面，让用户在 `projectSettings`（项目设置）和 `userSettings`（用户设置）之间选择。
2. **状态同步**：将用户的选择通过 `updateWizardData` 写入向导上下文，并自动触发 `goNext()` 进入下一步。
3. **取消操作**：支持通过 `Esc`（或用户自定义的 `confirm:no` 快捷键）取消整个向导流程。
4. **键盘导航**：支持 ↑/↓ 箭头键在选项间移动，Enter 键确认选择，符合终端 TUI 的交互习惯。

## 具体技术实现

### 关键流程

组件渲染流程如下：

1. 调用 `useWizard<AgentWizardData>()` 获取向导上下文中的 `goNext`、`updateWizardData`、`cancel`。
2. 使用 React Compiler 的 memo cache（`$ = _c(11)`）对静态选项数组和 JSX 片段进行缓存，避免不必要的重渲染。
3. 定义 `locationOptions` 数组，包含两个选项对象，每个对象有 `label`（显示文本）和 `value`（`SettingSource` 类型）。
4. 定义 `onChange` 回调：接收选中的 `value`，调用 `updateWizardData({ location: value as SettingSource })` 更新状态，然后调用 `goNext()` 前进。
5. 定义 `onCancel` 回调：调用 `cancel()` 退出向导。
6. 渲染 `WizardDialogLayout` 包裹的 `Select` 组件，将选项、回调和底部快捷键提示传入。

### 数据结构

```typescript
// 选项数据结构
const locationOptions = [
  {
    label: "Project (.claude/agents/)",
    value: "projectSettings" as SettingSource
  },
  {
    label: "Personal (~/.claude/agents/)",
    value: "userSettings" as SettingSource
  }
]
```

`SettingSource` 来自 `src/utils/settings/constants.ts`，是一个联合类型：`'userSettings' | 'projectSettings' | 'localSettings' | 'flagSettings' | 'policySettings'`。在此组件中仅使用前两个值。

### React Compiler 缓存模式

代码中大量使用了 React Compiler 生成的 memo cache 模式（`$[n] === Symbol.for("react.memo_cache_sentinel")`），这是编译后的特征。它缓存了：
- `locationOptions` 静态数组
- `Byline` + `KeyboardShortcutHint` + `ConfigurableShortcutHint` 组成的底部提示 JSX
- `onChange` 和 `onCancel` 回调函数
- 最终的 `WizardDialogLayout` JSX

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/agents/new-agent-creation/wizard-steps/LocationStep.tsx` | 本文件，LocationStep 组件实现 |
| `src/components/agents/new-agent-creation/CreateAgentWizard.tsx` | 调用方，将 `LocationStep` 作为向导第一步注入 `steps` 数组 |
| `src/components/wizard/WizardProvider.tsx` | 向导状态管理 Provider，提供 `updateWizardData`、`goNext`、`cancel` 等 API |
| `src/components/wizard/useWizard.ts` | `useWizard` Hook，LocationStep 通过它访问向导上下文 |
| `src/components/wizard/WizardDialogLayout.tsx` | 通用向导布局壳，负责标题、步骤计数器、Dialog 渲染 |
| `src/components/CustomSelect/select.tsx` | 底层选择器组件，处理键盘导航、选项渲染、Enter 确认 |
| `src/utils/settings/constants.ts` | `SettingSource` 类型定义和常量 |
| `src/components/design-system/Byline.tsx` | 底部提示行的布局组件，用 middot 分隔多个快捷键提示 |
| `src/components/design-system/KeyboardShortcutHint.tsx` | 键盘快捷键提示展示组件 |
| `src/components/ConfigurableShortcutHint.tsx` | 可配置的快捷键提示，支持从用户设置中读取自定义快捷键 |

## 依赖与外部交互

### 上游依赖（调用方）

- **`CreateAgentWizard.tsx`**：作为向导流程编排者，将 `LocationStep` 硬编码为 `steps[0]`。它通过 `WizardProvider` 提供全局状态容器。

### 下游依赖（被调用方）

- **`WizardDialogLayout`**：接收 `subtitle="Choose location"` 和 `footerText`，渲染带标题的对话框布局。
- **`Select`**（`src/components/CustomSelect/select.tsx`）：接收 `options`、`onChange`、`onCancel`，处理所有键盘事件和选项高亮逻辑。
- **`useWizard`**：提供类型安全的向导上下文访问。

### 横向依赖

- **`MemoryStep.tsx`**：读取 `wizardData.location` 的值。如果用户选择了 `userSettings`，`MemoryStep` 会将 "User scope" 设为默认推荐选项；如果选择了 `projectSettings`，则推荐 "Project scope"。

## 风险、边界与改进建议

### 风险与边界

1. **硬编码选项**：`locationOptions` 是写死的数组，如果未来需要支持 `localSettings` 或 `policySettings` 作为保存位置，必须修改源码。当前不支持 `flagSettings` 或 `policySettings` 的写入（虽然 `SettingSource` 类型包含它们）。
2. **无持久化默认值**：每次打开向导都会从空状态开始，没有记住用户上次的选择。
3. **Esc 取消无二次确认**：按 `Esc` 直接调用 `cancel()` 退出整个向导，没有"是否确认取消"的拦截，可能误操作。
4. **React Compiler 依赖**：代码已被 React Compiler 编译，手动修改时需要理解其 memo cache 语义，否则容易破坏性能优化或引入 bug。
5. **类型文件缺失**：`AgentWizardData` 和 `WizardContextValue` 的类型定义在源码仓库中没有显式的 `.ts` 文件，可能是构建时内联生成的，这给静态分析和 IDE 跳转带来困难。

### 改进建议

1. **动态选项生成**：根据 `getAllowedSettingSources()` 或权限配置动态生成可选的 `SettingSource`，而不是硬编码两个选项。
2. **记住上次选择**：在 `updateWizardData` 写入时，可将用户偏好缓存到本地状态或配置文件中，下次打开向导时预选中。
3. **取消确认**：在 `cancel` 前增加一个确认步骤，或至少在高成本输入场景（如已经填写了多步信息后）提供确认提示。
4. **提取通用 Step 模式**：`LocationStep`、`MethodStep`、`ModelStep` 等大量步骤遵循相同的"Select + goNext"模式，可以考虑抽象一个 `SelectStep` 高阶组件或工厂函数，减少重复代码。
