# useHistorySearch.ts 研究文档

## 场景与职责

`useHistorySearch` 是 Claude Code PromptInput 中负责 **`Ctrl+R` 历史搜索** 的核心 Hook。它允许用户在输入框中按下 `Ctrl+R`（或配置的等效快捷键）后，进入历史搜索模式：用户输入查询字符串，Hook 实时从历史记录中反向查找匹配项，并将匹配到的历史命令填充到输入框中。

该 Hook 是 `PromptInput.tsx` 的重要组成部分，直接决定了用户如何通过键盘快速复用之前的提示词。

## 功能点目的

1. **启动/退出历史搜索**：通过键绑定 `history:search` 进入搜索模式；通过 `historySearch:cancel`、`historySearch:accept`、`historySearch:execute` 退出。

2. **实时反向搜索**：基于用户输入的 `historyQuery`，从全局历史文件（`~/.claude/history.jsonl`）中反向读取，找到第一个包含查询字符串的历史条目。

3. **支持连续翻找（Next Match）**：在搜索模式下，用户可以连续触发 `historySearch:next` 查找下一个匹配项（更旧的历史）。

4. **保留原始输入状态**：进入搜索时保存当前输入内容、光标位置、输入模式、粘贴内容；取消搜索时完整恢复这些状态。

5. **接受匹配并提交**：用户可以直接接受当前匹配并提交（`historySearch:execute`），或仅接受但不提交（`historySearch:accept`）。

6. **空查询回退**：当搜索查询为空时，自动恢复原始输入。

## 具体技术实现

### 关键流程

#### 1. 启动搜索（`handleStartSearch`）
```ts
const handleStartSearch = useCallback(() => {
  setIsSearching(true)
  setOriginalInput(currentInput)
  setOriginalCursorOffset(currentCursorOffset)
  setOriginalMode(currentMode)
  setOriginalPastedContents(currentPastedContents)
  historyReader.current = makeHistoryReader()
  seenPrompts.current.clear()
}, [...])
```
- 保存当前输入状态
- 创建新的 `historyReader`（`AsyncGenerator<HistoryEntry>`）
- 清空已见过的 prompts Set（用于去重）

#### 2. 搜索执行（`searchHistory`）
```ts
const searchHistory = useCallback(async (resume: boolean, signal?: AbortSignal): Promise<void> => {
  if (!isSearching) return
  if (historyQuery.length === 0) {
    // 恢复原始输入
    onInputChange(originalInput)
    onCursorChange(originalCursorOffset)
    onModeChange(originalMode)
    setPastedContents(originalPastedContents)
    return
  }
  if (!resume) {
    closeHistoryReader()
    historyReader.current = makeHistoryReader()
    seenPrompts.current.clear()
  }
  while (true) {
    if (signal?.aborted) return
    const item = await historyReader.current.next()
    if (item.done) {
      setHistoryFailedMatch(true)
      return
    }
    const display = item.value.display
    const matchPosition = display.lastIndexOf(historyQuery)
    if (matchPosition !== -1 && !seenPrompts.current.has(display)) {
      seenPrompts.current.add(display)
      setHistoryMatch(item.value)
      setHistoryFailedMatch(false)
      const mode = getModeFromInput(display)
      onModeChange(mode)
      onInputChange(display)
      setPastedContents(item.value.pastedContents)
      // 光标定位到匹配位置
      const value = getValueFromInput(display)
      const cleanMatchPosition = value.lastIndexOf(historyQuery)
      onCursorChange(cleanMatchPosition !== -1 ? cleanMatchPosition : matchPosition)
      return
    }
  }
}, [...])
```

#### 3. 历史读取器（`makeHistoryReader`）
来自 `src/history.ts`：
- 首先读取内存中的 `pendingEntries`（尚未 flush 到磁盘的历史）
- 然后使用 `readLinesReverse` 反向读取 `~/.claude/history.jsonl`
- 每行反序列化为 `LogEntry`，再解析粘贴内容引用，最终产出 `HistoryEntry`
- 过滤掉当前 session 中 `skippedTimestamps` 标记的条目（由 `removeLastFromHistory` 使用）

#### 4. 键绑定处理
```ts
useKeybinding('history:search', handleStartSearch, {
  context: 'Global',
  isActive: feature('HISTORY_PICKER') ? false : !isSearching,
})

const historySearchHandlers = useMemo(() => ({
  'historySearch:next': handleNextMatch,
  'historySearch:accept': handleAccept,
  'historySearch:cancel': handleCancel,
  'historySearch:execute': handleExecute,
}), [...])

useKeybindings(historySearchHandlers, {
  context: 'HistorySearch',
  isActive: isSearching,
})
```

#### 5. 特殊按键处理（Backspace）
```ts
const handleKeyDown = (e: KeyboardEvent): void => {
  if (!isSearching) return
  if (e.key === 'backspace' && historyQuery === '') {
    e.preventDefault()
    handleCancel()
  }
}
```
当搜索查询为空时按 Backspace，直接取消搜索并恢复原始输入。

#### 6. Backward-compat bridge
```ts
useInput(
  (_input, _key, event) => {
    handleKeyDown(new KeyboardEvent(event.keypress))
  },
  { isActive: isSearching },
)
```
由于 `PromptInput` 尚未将 `handleKeyDown` 直接绑定到 `<Box onKeyDown>`，这里通过 `useInput` 做兼容桥接。

### 数据结构

- **HistoryEntry**：
  ```ts
  export interface HistoryEntry {
    display: string
    pastedContents: Record<number, PastedContent>
  }
  ```

- **返回状态**：
  - `historyQuery: string`：当前搜索查询
  - `setHistoryQuery: (query: string) => void`：设置查询
  - `historyMatch: HistoryEntry | undefined`：当前匹配到的历史条目
  - `historyFailedMatch: boolean`：是否已遍历完所有历史但未找到匹配
  - `handleKeyDown: (e: KeyboardEvent) => void`：额外的键盘事件处理

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/hooks/useHistorySearch.ts` | 本 Hook 实现 |
| `src/components/PromptInput/PromptInput.tsx` | 唯一调用方 |
| `src/history.ts` | `makeHistoryReader`、`getHistory`、`addToHistory`、`readLinesReverse` |
| `src/components/PromptInput/inputModes.ts` | `getModeFromInput`、`getValueFromInput` |
| `src/keybindings/useKeybinding.ts` | `useKeybinding`、`useKeybindings` |
| `src/ink.js` | `useInput`、`KeyboardEvent` |
| `src/utils/config.ts` | `HistoryEntry`、`PastedContent` 类型 |
| `src/utils/fsOperations.ts` | `readLinesReverse`（反向逐行读取文件） |
| `src/utils/pasteStore.ts` | `retrievePastedText`（解析大粘贴内容的哈希引用） |

## 依赖与外部交互

### 内部依赖
- **React**：`useCallback`、`useEffect`、`useMemo`、`useRef`、`useState`
- **文件系统**：`readLinesReverse` 读取 `~/.claude/history.jsonl`
- **粘贴存储**：`pasteStore.ts` 用于解析历史记录中引用的外部粘贴内容

### 外部交互
- **历史文件**：`~/.claude/history.jsonl`，JSON Lines 格式，每行是一个 `LogEntry`
- **键绑定系统**：`history:search`（全局）、`historySearch:*`（HistorySearch 上下文）

## 风险、边界与改进建议

### 风险与边界

1. **`readLinesReverse` 的文件句柄泄漏风险**：`closeHistoryReader` 中显式调用 `historyReader.current.return(undefined)` 来触发 `readLinesReverse` 的 `finally` 块关闭文件句柄。注释明确说明了这一点，但如果 `historyReader` 的 `return()` 未被调用（如异常中断），仍可能导致文件描述符泄漏。

2. **搜索是串行遍历**：`searchHistory` 使用 `while (true)` + `await historyReader.current.next()` 串行遍历历史。对于历史记录很长的用户（如数千条），首次搜索或连续 `next` 可能会有可感知的延迟。虽然 `readLinesReverse` 是流式读取，但每行的 JSON 解析和粘贴内容解析仍是同步阻塞的。

3. **`display.lastIndexOf(historyQuery)` 的匹配策略**：搜索是在 `display`（完整输入字符串，包含模式前缀如 `!`）中做子串匹配。这意味着如果用户搜索 `!bash`，它也会匹配到包含 `!bash` 的普通 prompt，而不仅仅是 bash 模式输入。虽然通常不影响使用，但语义上不够精确。

4. **光标定位的复杂性**：匹配位置首先在 `display` 中计算，然后尝试在 `value`（去除模式字符后的纯内容）中重新定位。如果 `historyQuery` 包含模式字符（如 `!`），`value.lastIndexOf(historyQuery)` 可能返回 `-1`，此时 fallback 到 `display` 中的位置，可能导致光标位置看起来"不对齐"。

5. **`HISTORY_PICKER` feature flag 的互斥**：当 `feature('HISTORY_PICKER')` 为 true 时，`history:search` 被完全禁用（`isActive: false`），因为模态对话框接管了 `ctrl+r`。这意味着两种历史搜索模式是互斥的，用户无法同时享受两者的优点。

6. **`useInput` 兼容桥接的 TODO**：代码中明确标注了 `TODO(onKeyDown-migration): remove once PromptInput passes handleKeyDown`。这个兼容层增加了额外的按键处理路径，可能与其他 `useInput` 处理器产生微妙的优先级问题。

### 改进建议

1. **增加搜索索引或缓存**：对于频繁使用的查询，可以维护一个内存中的 LRU 缓存，记录 `(query, firstMatch)`，避免每次都从头遍历历史文件。

2. **支持正则/模糊搜索**：当前是纯子串匹配。可以考虑支持简单的模糊匹配（如每个字符都按顺序出现即可），提升搜索容错性。

3. **优化光标定位逻辑**：在计算光标位置时，应该始终基于用户实际看到的输入内容（即 `value`）来定位。如果 `historyQuery` 包含模式字符，可以在搜索前将其剥离，确保光标位置更自然。

4. **异步搜索的取消与节流**：当前 `useEffect` 在 `historyQuery` 每次变化时都会创建新的 `AbortController` 并重新搜索。如果用户快速输入（如连续敲 5 个字符），会触发 5 次搜索，前 4 次的 `AbortController.abort()` 可能无法及时阻止已经启动的 `await reader.next()`。可以考虑增加一个小的防抖（debounce，如 50-100ms），减少不必要的搜索。

5. **统一历史搜索入口**：`HISTORY_PICKER` 和 `useHistorySearch` 代表了两种不同的历史搜索 UX。建议评估是否可以将两者统一（如保留 `ctrl+r` 的实时搜索体验，同时用 `ctrl+shift+r` 打开模态选择器），而不是完全互斥。

6. **移除 `useInput` 兼容层**：跟进 `PromptInput` 的 `onKeyDown` 迁移工作，移除 `useInput` 桥接，简化事件处理路径。
