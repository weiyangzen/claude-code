# Research: src/hooks/usePromptSuggestion.ts

## 场景与职责

`usePromptSuggestion` 是一个用于管理**输入框提示建议（Prompt Suggestion）**的 React Hook。它从全局 `AppState` 中读取由后台 forked agent（speculation 流水线）生成的建议文本，并在用户输入框为空且助手未响应时将其展示为幽灵文本（ghost text）。

该 Hook 的核心职责包括：
1. **建议展示控制**：决定何时显示/隐藏建议（输入为空、助手未在响应时显示）。
2. **用户交互追踪**：记录建议展示时间（`shownAt`）、接受时间（`acceptedAt`）、首次按键时间（`firstKeystrokeAt`）。
3. **分析埋点**：在用户提交输入时，统一上报 `tengu_prompt_suggestion` 事件，包含接受/忽略结果、耗时、相似度等维度。
4. **建议生命周期管理**：提供 `markShown`、`markAccepted`、`logOutcomeAtSubmission`、`resetSuggestion` 等标准动作。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| **建议可见性控制** | 仅在 `inputValue.length === 0 && !isAssistantResponding` 时展示建议，避免干扰用户已有输入或助手正在输出时弹出建议。 |
| **展示状态标记** | `markShown` 在建议首次渲染时记录 `shownAt` 时间戳，用于后续计算接受/忽略耗时。 |
| **接受状态标记** | `markAccepted` 在用户按下 Tab 接受建议时记录 `acceptedAt`。 |
| **提交时埋点** | `logOutcomeAtSubmission` 在最终提交时判断建议是被接受（Tab 或输入完全匹配建议）还是被忽略，并上报分析事件。 |
| **焦点状态追踪** | `wasFocusedWhenShown` 记录建议展示时终端是否处于焦点状态，用于分析用户注意力。 |
| **相似度计算** | 通过 `finalInput.length / suggestionText.length` 计算用户输入与建议的相似度，作为模型质量指标。 |

## 具体技术实现

### 状态读取

```ts
const promptSuggestion = useAppState(s => s.promptSuggestion)
const setAppState = useSetAppState()
const isTerminalFocused = useTerminalFocus()
```

从 `AppState` 解构出：
- `text`：建议文本。
- `promptId`：建议变体 ID（用于分析）。
- `shownAt` / `acceptedAt`：时间戳。
- `generationRequestId`：生成请求 ID（用于分析）。

### 建议可见性

```ts
const suggestion =
  isAssistantResponding || inputValue.length > 0 ? null : suggestionText
```

返回的 `suggestion` 是上层组件实际渲染的幽灵文本内容。

### 展示时焦点捕获

```ts
if (shownAt > 0 && shownAt !== prevShownAt.current) {
  prevShownAt.current = shownAt
  wasFocusedWhenShown.current = isTerminalFocused
  firstKeystrokeAt.current = 0
}
```

利用条件赋值在渲染阶段同步捕获焦点状态，避免引入额外的 `useEffect`。

### 首次按键追踪

```ts
if (inputValue.length > 0 && firstKeystrokeAt.current === 0 && isValidSuggestion) {
  firstKeystrokeAt.current = Date.now()
}
```

当用户在建议可见期间开始输入时，记录首次按键时间，用于计算 `timeToFirstKeystrokeMs`。

### 提交时结果判定

```ts
const tabWasPressed = acceptedAt > shownAt
const wasAccepted = tabWasPressed || finalInput === suggestionText
```

- **Tab 接受**：用户显式按 Tab 填充建议。
- **Enter 接受**：用户未按 Tab，但直接提交了与建议完全一致的输入（空输入 + Enter 的情况）。

### 分析事件字段

```ts
logEvent('tengu_prompt_suggestion', {
  source: 'cli',
  outcome: wasAccepted ? 'accepted' : 'ignored',
  prompt_id: promptId,
  generationRequestId,
  acceptMethod: tabWasPressed ? 'tab' : 'enter',
  timeToAcceptMs: timeMs - shownAt,
  timeToIgnoreMs: timeMs - shownAt,
  timeToFirstKeystrokeMs: firstKeystrokeAt.current - shownAt,
  wasFocusedWhenShown: wasFocusedWhenShown.current,
  similarity: Math.round((finalInput.length / (suggestionText?.length || 1)) * 100) / 100,
  ...(process.env.USER_TYPE === 'ant' && { suggestion, userInput: finalInput }),
})
```

## 关键代码路径与文件引用

| 文件 | 作用 |
|------|------|
| `src/hooks/usePromptSuggestion.ts` | 本 Hook：建议状态管理、交互追踪、埋点。 |
| `src/components/PromptInput/PromptInput.tsx` | 调用方：读取 `suggestion`、`markShown`、`markAccepted`、`logOutcomeAtSubmission`，绑定到输入框和提交流。 |
| `src/state/AppState.js` / `src/state/AppStateStore.js` | `promptSuggestion` 状态的存储与更新接口。 |
| `src/ink/hooks/use-terminal-focus.js` | 提供 `useTerminalFocus`，判断终端当前是否获得焦点。 |
| `src/services/analytics/index.js` | `logEvent` 来源：分析埋点 SDK。 |
| `src/services/PromptSuggestion/speculation.ts` | `abortSpeculation` 来源：当建议需要被清除时，中止当前的 speculation 流程。 |

## 依赖与外部交互

### 运行时依赖
- **React**：`useCallback`、`useRef`。
- **AppState**：通过 `useAppState` / `useSetAppState` 订阅和修改全局状态。
- **Ink**：`useTerminalFocus` 来自 Ink 的终端焦点钩子。
- **Analytics**：`logEvent` 上报到内部分析系统。

### 与调用方的契约
- `usePromptSuggestion({ inputValue, isAssistantResponding })`
- 返回：
  - `suggestion: string | null`：当前应展示的建议文本。
  - `markAccepted: () => void`：Tab 键触发。
  - `markShown: () => void`：建议首次渲染时触发。
  - `logOutcomeAtSubmission(finalInput, opts?)`：提交时触发，可选 `skipReset` 避免重置建议（用于 speculation 接受路径）。

### 与 speculation 系统的交互
- 建议文本由 `speculation.ts` 中的 `generatePipelinedSuggestion` 在 speculation 完成后生成，并写入 `AppState.promptSuggestion`。
- `resetSuggestion` 会调用 `abortSpeculation(setAppState)`，确保清除建议时同时中止后台的 speculation 流水线。

## 风险、边界与改进建议

### 风险与边界
1. **`shownAt` 依赖导致的无限循环风险**：注释中明确说明 `markShown` 使用 `setAppState` 回调形式读取 `prev.promptSuggestion.shownAt`，而不是将 `shownAt` 作为 `useCallback` 依赖，否则会导致"回调被调用时触发无限循环"。
2. **相似度计算在空建议时的除零保护**：`suggestionText?.length || 1` 确保了分母不会为零，但若建议为空字符串，相似度会被错误地放大。不过实际上空建议不会进入 `isValidSuggestion` 分支。
3. **ant-only 的 PII 数据**：`suggestion` 和 `userInput` 仅在 `USER_TYPE === 'ant'` 时上报，这是内部 dogfooding 的数据采集策略，外部构建不会包含这些字段。
4. **焦点状态的非响应式读取**：`wasFocusedWhenShown` 在 `shownAt` 变化时一次性捕获，若建议展示期间焦点发生变化，不会更新该值。这是设计上的取舍（只关心"展示瞬间"的焦点状态）。
5. **`firstKeystrokeAt` 的渲染阶段赋值**：在函数组件的渲染阶段直接修改 ref（`firstKeystrokeAt.current = Date.now()`）在严格模式下可能被调用两次，但由于是时间戳，差值通常在毫秒级，对分析影响可忽略。

### 改进建议
1. **建议接受率的 A/B 实验支持**：当前 `promptId` 和 `generationRequestId` 已具备实验归因能力，但可以在 Hook 层增加 `experimentId` 字段的透传，方便更细粒度的实验分析。
2. **相似度算法优化**：当前使用简单长度比，未来可引入编辑距离（Levenshtein）或 token 级别的 Jaccard 相似度，更准确地衡量用户输入与建议的偏离程度。
3. **建议展示时长上限**：当前没有自动隐藏建议的逻辑，若用户长时间不输入，建议会一直显示。可考虑在展示 N 秒后自动淡出或降低透明度。
4. **多语言/长文本建议的截断显示**：`PromptInput.tsx` 中直接渲染完整 `suggestion`，若建议文本极长可能撑爆输入框。可在 Hook 或 UI 层增加截断逻辑。
