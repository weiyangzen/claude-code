# ModelSelector.tsx 研究文档

## 场景与职责

`ModelSelector.tsx` 是 Claude Code Agent 管理系统中的**模型选择交互组件**，用于让用户为子 Agent 指定运行时所使用的 LLM 模型。它出现在以下两个场景：

1. **创建 Agent 向导** — 在 `ModelStep`（`src/components/agents/new-agent-creation/wizard-steps/ModelStep.tsx`）中，让用户为新 Agent 选择模型。
2. **编辑 Agent** — 在 `AgentEditor`（`src/components/agents/AgentEditor.tsx`）中，让用户修改已有 Agent 的模型配置。

该组件基于 `CustomSelect/select.js` 构建，提供下拉列表式选择体验，运行在 Ink（React for Terminal）环境中。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| **模型选项展示** | 提供 `sonnet`、`opus`、`haiku`、`inherit` 四个标准选项，并附带描述说明 |
| **自定义模型兼容** | 若当前 Agent 已配置了一个不在标准别名列表中的完整模型 ID（如 `claude-opus-4-5`），将其作为首选项保留，避免用户确认时被意外覆盖 |
| **默认选中** | 支持通过 `initialModel` 指定默认选中项，未指定时回退到 `"sonnet"` |
| **取消/完成回调** | 用户选择后通过 `onComplete` 回传模型值；支持 `onCancel` 回退 |

## 具体技术实现

### 1. 模型选项构建逻辑

组件在渲染时动态构建 `modelOptions`：

```ts
const base = getAgentModelOptions()
if (initialModel && !base.some(o => o.value === initialModel)) {
  return [
    { value: initialModel, label: initialModel, description: 'Current model (custom ID)' },
    ...base,
  ]
}
return base
```

这一设计的核心目的是**保护非标准模型 ID**：当 Agent 配置文件中使用了一个不在预定义别名列表中的完整模型字符串时，该值会被注入为第一个选项，标签显示为原始 ID，描述为 `"Current model (custom ID)"`，确保用户即使不做任何操作直接确认，也不会丢失原有配置。

### 2. 标准模型选项定义

`getAgentModelOptions()` 定义在 `src/utils/model/agent.ts`，返回固定数组：

| value | label | description |
|-------|-------|-------------|
| `sonnet` | Sonnet | Balanced performance - best for most agents |
| `opus` | Opus | Most capable for complex reasoning tasks |
| `haiku` | Haiku | Fast and efficient for simple tasks |
| `inherit` | Inherit from parent | Use the same model as the main conversation |

### 3. 渲染结构

组件渲染结构非常简单：

```tsx
<Box flexDirection="column">
  <Box marginBottom={1}>
    <Text dimColor>Model determines the agent's reasoning capabilities and speed.</Text>
  </Box>
  <Select
    options={modelOptions}
    defaultValue={defaultModel}
    onChange={onComplete}
    onCancel={t3}
  />
</Box>
```

其中 `t3` 是取消回调的兜底逻辑：若调用方传了 `onCancel` 则执行它，否则执行 `onComplete(undefined)`。

### 4. React Compiler 缓存

该文件同样为 React Compiler 编译产物（`_c(11)`），使用 `$[n]` 数组缓存：

- `$[0]`/`$[1]` 缓存 `initialModel` 与计算后的 `modelOptions`
- `$[2]` 缓存顶部说明文字 JSX（常量）
- `$[3]`/`$[4]`/`$[5]` 缓存取消回调
- `$[6]`~`$[10]` 缓存最终根节点 JSX

### 5. Props 接口

```ts
interface ModelSelectorProps {
  initialModel?: string
  onComplete: (model?: string) => void
  onCancel?: () => void
}
```

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/agents/ModelSelector.tsx` | 本组件源码（React Compiler 编译后产物） |
| `src/utils/model/agent.ts` | `getAgentModelOptions()`、`getAgentModelDisplay()`、`getDefaultSubagentModel()` 等模型工具函数 |
| `src/components/agents/AgentEditor.tsx` | 调用方之一，在 `edit-model` 模式下渲染 `ModelSelector` |
| `src/components/agents/new-agent-creation/wizard-steps/ModelStep.tsx` | 调用方之二，在创建向导中渲染 `ModelSelector` |
| `src/components/CustomSelect/select.js` | 底层选择组件，提供 ↑/↓/Enter/Esc 交互 |
| `src/ink.js` | `Box`、`Text` 终端组件 |

## 依赖与外部交互

### 直接依赖

- **`React / Ink`**：`Box`, `Text`
- **`CustomSelect/select.js`**：`Select` 组件，提供通用下拉选择能力
- **`utils/model/agent.ts`**：`getAgentModelOptions` 获取标准选项列表

### 调用方交互

```
AgentEditor (edit-model)
└── ModelSelector(initialModel={agent.model}, onComplete, onCancel=undefined)
    └── onComplete(model) => handleSave({ model })

ModelStep (CreateAgentWizard)
└── ModelSelector(initialModel={wizardData.selectedModel}, onComplete, onCancel={goBack})
    └── onComplete(model) => updateWizardData({ selectedModel: model }) + goNext()
```

### 与模型解析系统的关联

`ModelSelector` 仅负责**交互层的选择与回传**，真正的模型解析与继承逻辑在 `src/utils/model/agent.ts` 的 `getAgentModel()` 中：

- 处理 `inherit` 选项的父模型继承
- 处理 Bedrock 跨区域推理前缀的传递
- 处理 `CLAUDE_CODE_SUBAGENT_MODEL` 环境变量覆盖
- 处理别名（`sonnet`/`opus`/`haiku`）到具体模型 ID 的解析

`ModelSelector` 本身不涉及上述运行时解析，仅将用户选择的字符串原样返回。

## 风险、边界与改进建议

### 风险与边界

1. **自定义模型 ID 的展示局限**
   - 当 `initialModel` 不在标准选项中时，组件仅将其原样显示为 `label`，不做任何有效性校验或截断。若模型 ID 极长，可能在终端窄屏下换行或截断，影响可读性。

2. **`defaultValue` 回退硬编码为 `"sonnet"`**
   - 若 `initialModel` 为 `undefined`，默认选中 `"sonnet"`。若产品策略调整默认模型（例如改为 `inherit`），需要修改此处硬编码。

3. **取消行为不一致**
   - `AgentEditor` 调用时未传 `onCancel`，此时按 Esc 会触发 `onComplete(undefined)`，即**将模型设为 undefined**（继承默认值）。这与 `ModelStep` 中按 Esc 回退上一步的行为语义不同，可能导致用户困惑。

4. **无搜索/过滤能力**
   - 当前仅 4 个标准选项 + 1 个自定义选项，列表很短，无需搜索。但随着未来模型家族扩展（如增加 `claude-4` 系列），纯上下键导航可能变得低效。

5. **编译产物可读性**
   - 与批次内其他文件一致，该文件为 React Compiler 编译输出，包含 `_c(11)` 等缓存机制，直接阅读源码成本较高。

### 改进建议

- **统一取消语义**：建议 `AgentEditor` 显式传入 `onCancel={() => setEditMode('menu')}`，使 Esc 行为在所有场景下保持一致（回退菜单而非清空模型）。
- **动态默认模型**：将 `"sonnet"` 回退逻辑改为调用 `getDefaultSubagentModel()` 或从配置读取，避免硬编码。
- **选项描述扩展**：在 `getAgentModelOptions()` 中增加模型能力标签（如 token 上限、是否支持 vision），帮助用户决策。
- **保留原始源码**：建议仓库中维护未编译的 `.tsx` 源码，降低后续研究与调试成本。
