# useArrowKeyHistory.tsx 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`useArrowKeyHistory.tsx` 是 Claude Code 的命令历史导航 Hook，提供类似于 Shell 的上下箭头历史记录浏览功能，支持草稿保存、模式过滤和搜索提示。

### 1.2 使用场景
| 场景 | 描述 |
|------|------|
| 上箭头导航 | 浏览历史记录中的上一条命令 |
| 下箭头导航 | 返回历史记录中的下一条命令 |
| 草稿保存 | 临时离开历史时保存当前输入 |
| Bash 模式过滤 | 仅显示以 `!` 开头的 Bash 命令 |
| 搜索提示 | 使用历史导航后提示搜索快捷键 |

### 1.3 调用方
- `src/components/PromptInput/PromptInput.tsx` - 主输入组件

---

## 2. 功能点目的

### 2.1 历史记录浏览
- **目的**：允许用户快速重用之前的输入
- **实现**：异步迭代历史记录，支持大文件的高效读取
- **特点**：分块加载（10条/块），减少磁盘 I/O

### 2.2 草稿保存与恢复
- **目的**：用户浏览历史后可返回原始输入
- **实现**：首次按上箭头时保存当前输入到 `lastShownHistoryEntry`
- **恢复**：下箭头回到索引 0 时恢复草稿

### 2.3 模式过滤
- **目的**：Bash 模式下仅显示 Bash 历史
- **实现**：检查每条历史记录的 `getModeFromInput(entry.display)`
- **触发**：仅在 Bash 模式（输入以 `!` 开头）时启用过滤

### 2.4 并发请求批处理
- **目的**：快速按键时避免重复磁盘读取
- **实现**：共享 `pendingLoad` Promise，批处理并发请求

---

## 3. 具体技术实现

### 3.1 数据结构

```typescript
export type HistoryMode = PromptInputMode  // 'prompt' | 'bash'

export function useArrowKeyHistory(
  onSetInput: (value: string, mode: HistoryMode, pastedContents: Record<number, PastedContent>) => void,
  currentInput: string,
  pastedContents: Record<number, PastedContent>,
  setCursorOffset?: (offset: number) => void,
  currentMode?: HistoryMode,
): {
  historyIndex: number           // 当前历史索引（React state）
  setHistoryIndex: (index: number) => void
  onHistoryUp: () => void        // 上箭头处理
  onHistoryDown: () => boolean   // 下箭头处理，返回是否到达底部
  resetHistory: () => void       // 重置历史状态
  dismissSearchHint: () => void  // 关闭搜索提示
}
```

### 3.2 共享加载状态

```typescript
// 模块级共享状态，批处理并发请求
let pendingLoad: Promise<HistoryEntry[]> | null = null
let pendingLoadTarget = 0
let pendingLoadModeFilter: HistoryMode | undefined = undefined

const HISTORY_CHUNK_SIZE = 10

async function loadHistoryEntries(
  minCount: number, 
  modeFilter?: HistoryMode
): Promise<HistoryEntry[]> {
  // 向上取整到块大小
  const target = Math.ceil(minCount / HISTORY_CHUNK_SIZE) * HISTORY_CHUNK_SIZE

  // 如果已有满足条件的加载在进行中，等待它
  if (pendingLoad && pendingLoadTarget >= target && pendingLoadModeFilter === modeFilter) {
    return pendingLoad
  }

  // 等待现有加载完成（无法中断）
  if (pendingLoad) {
    await pendingLoad
  }

  // 启动新加载
  pendingLoadTarget = target
  pendingLoadModeFilter = modeFilter
  pendingLoad = (async () => {
    const entries: HistoryEntry[] = []
    let loaded = 0
    for await (const entry of getHistory()) {
      // 应用模式过滤
      if (modeFilter) {
        const entryMode = getModeFromInput(entry.display)
        if (entryMode !== modeFilter) continue
      }
      entries.push(entry)
      if (++loaded >= pendingLoadTarget) break
    }
    return entries
  })()

  try {
    return await pendingLoad
  } finally {
    pendingLoad = null
    pendingLoadTarget = 0
    pendingLoadModeFilter = undefined
  }
}
```

### 3.3 上箭头处理（onHistoryUp）

```typescript
const onHistoryUp = useCallback((): void => {
  // 同步捕获和递增索引（处理快速按键）
  const targetIndex = historyIndexRef.current
  historyIndexRef.current++
  
  const inputAtPress = currentInputRef.current
  const pastedContentsAtPress = pastedContentsRef.current
  const modeAtPress = currentModeRef.current

  // 首次按上箭头：保存草稿
  if (targetIndex === 0) {
    initialModeFilterRef.current = modeAtPress === 'bash' ? modeAtPress : undefined
    
    const hasInput = inputAtPress.trim() !== ''
    setLastShownHistoryEntry(hasInput ? {
      display: inputAtPress,
      pastedContents: pastedContentsAtPress,
      mode: modeAtPress
    } : undefined)
  }

  const modeFilter = initialModeFilterRef.current

  void (async () => {
    const neededCount = targetIndex + 1

    // 模式过滤器变更时清空缓存
    if (historyCacheModeFilter.current !== modeFilter) {
      historyCache.current = []
      historyCacheModeFilter.current = modeFilter
      historyIndexRef.current = 0
    }

    // 按需加载更多条目
    if (historyCache.current.length < neededCount) {
      const entries = await loadHistoryEntries(neededCount, modeFilter)
      // 竞争条件处理：只更新如果加载了更多
      if (entries.length > historyCache.current.length) {
        historyCache.current = entries
      }
    }

    // 检查是否可以导航
    if (targetIndex >= historyCache.current.length) {
      historyIndexRef.current--  // 回滚
      return  // 保持草稿不变
    }

    const newIndex = targetIndex + 1
    setHistoryIndex(newIndex)
    updateInput(historyCache.current[targetIndex], true)

    // 导航 2 条后显示搜索提示
    if (newIndex >= 2 && !hasShownSearchHintRef.current) {
      hasShownSearchHintRef.current = true
      showSearchHint()
    }
  })()
}, [updateInput, showSearchHint])
```

### 3.4 下箭头处理（onHistoryDown）

```typescript
const onHistoryDown = useCallback((): boolean => {
  const currentIndex = historyIndexRef.current

  if (currentIndex > 1) {
    // 在历史中向下移动
    historyIndexRef.current--
    setHistoryIndex(currentIndex - 1)
    updateInput(historyCache.current[currentIndex - 2])
  } else if (currentIndex === 1) {
    // 回到草稿
    historyIndexRef.current = 0
    setHistoryIndex(0)
    if (lastShownHistoryEntry) {
      const savedMode = lastShownHistoryEntry.mode
      if (savedMode) {
        setInputWithCursor(
          lastShownHistoryEntry.display, 
          savedMode, 
          lastShownHistoryEntry.pastedContents ?? {}
        )
      } else {
        updateInput(lastShownHistoryEntry)
      }
    } else {
      // 过滤模式下保持模式
      setInputWithCursor('', initialModeFilterRef.current ?? 'prompt', {})
    }
  }

  return currentIndex <= 0  // 返回是否已在底部
}, [lastShownHistoryEntry, updateInput, setInputWithCursor])
```

### 3.5 Ref 同步模式

```typescript
// 保持 Refs 与 Props 同步（每次渲染同步更新）
currentInputRef.current = currentInput
pastedContentsRef.current = pastedContents
currentModeRef.current = currentMode

// 使用 Ref 进行同步读取
const targetIndex = historyIndexRef.current
historyIndexRef.current++
```

**原因**：React state 更新是异步的，快速按键可能导致闭包读取到过期值。

---

## 4. 关键代码路径与文件引用

### 4.1 依赖图

```
useArrowKeyHistory.tsx
├── react (useCallback, useRef, useState)
├── components/PromptInput/inputModes.js  (getModeFromInput)
├── context/notifications.js              (useNotifications)
├── components/ConfigurableShortcutHint.js
├── components/PromptInput/Notifications.js  (FOOTER_TEMPORARY_STATUS_TIMEOUT)
├── history.js                            (getHistory)
├── ink.js                                (Text)
└── types/textInputTypes.js               (PromptInputMode)
```

### 4.2 调用链

```
PromptInput.tsx
  └── useArrowKeyHistory()
      ├── loadHistoryEntries()      // 历史加载
      │   └── getHistory()          // 异步生成器
      ├── onHistoryUp()             // 上箭头
      │   ├── 保存草稿
      │   ├── loadHistoryEntries()
      │   └── updateInput()
      └── onHistoryDown()           // 下箭头
          ├── updateInput()
          └── 恢复草稿
```

### 4.3 使用示例

```typescript
// PromptInput.tsx
const {
  resetHistory,
  onHistoryUp,
  onHistoryDown,
  dismissSearchHint,
  historyIndex,
} = useArrowKeyHistory(
  (value: string, historyMode: HistoryMode, pastedContents: Record<number, PastedContent>) => {
    onChange(value)
    onModeChange(historyMode)
    setPastedContents(pastedContents)
  },
  input,
  pastedContents,
  setCursorOffset,
  mode,
)

// 按键处理
function handleHistoryUp() {
  if (!isCursorOnFirstLine) return  // 多行输入仅在首行触发
  onHistoryUp()
}

function handleHistoryDown() {
  if (!isCursorOnLastLine) return   // 多行输入仅在末行触发
  if (onHistoryDown() && footerItems.length > 0) {
    // 到达历史底部，进入 footer 导航
    selectFooterItem(footerItems[0])
  }
}
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 | 类型 |
|------|------|------|
| `react` | Hooks API | npm 包 |

### 5.2 历史存储

| 函数 | 来源 | 用途 |
|------|------|------|
| `getHistory()` | `history.js` | 异步生成历史记录 |
| `HistoryEntry` | `utils/config.js` | 历史条目类型 |

### 5.3 模式检测

| 函数 | 来源 | 用途 |
|------|------|------|
| `getModeFromInput()` | `inputModes.js` | 检测输入模式 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 缓解 |
|------|------|------|
| 快速按键竞态 | 异步加载可能导致顺序错乱 | Ref 同步模式 |
| 内存泄漏 | 大历史文件缓存 | 分块加载，仅缓存需要条目 |
| 草稿丢失 | 组件卸载时草稿未保存 | 外层组件负责持久化 |
| 模式切换混乱 | 模式过滤器变更时索引混乱 | 清空缓存重置索引 |

### 6.2 边界条件

1. **空历史**：`historyCache` 为空，导航无效果
2. **单条历史**：上箭头后无法继续上翻
3. **无输入草稿**：`lastShownHistoryEntry` 为 undefined，恢复时清空
4. **粘贴内容**：草稿包含 `pastedContents`，恢复时一并恢复
5. **搜索中**：`isSearchingHistory` 时关闭搜索提示

### 6.3 改进建议

1. **持久化草稿**：
   ```typescript
   // 使用 localStorage 保存跨会话草稿
   useEffect(() => {
     localStorage.setItem('draft', JSON.stringify(lastShownHistoryEntry))
   }, [lastShownHistoryEntry])
   ```

2. **模糊搜索集成**：
   ```typescript
   // 支持在历史中模糊搜索而非仅顺序浏览
   const [searchQuery, setSearchQuery] = useState('')
   const filteredHistory = useMemo(() => 
     historyCache.current.filter(h => h.display.includes(searchQuery)),
     [searchQuery]
   )
   ```

3. **无限滚动**：
   ```typescript
   // 虚拟滚动支持超大历史文件
   const { visibleItems, containerRef } = useVirtualScroll(historyCache.current)
   ```

4. **撤销支持**：
   ```typescript
   // 支持 Ctrl+Z 撤销历史导航
   const { undo, redo } = useHistoryUndoRedo()
   ```

5. **性能优化**：
   ```typescript
   // 使用 IndexedDB 替代内存缓存
   const historyDB = await openDB('history', 1)
   ```

### 6.4 代码质量

- **优点**：
  - Ref 同步模式处理快速按键
  - 批处理并发请求减少 I/O
  - 清晰的草稿保存/恢复逻辑
  
- **潜在改进**：
  - 添加更多 JSDoc 说明复杂逻辑
  - 提取加载逻辑为独立 Hook
  - 添加单元测试覆盖竞态条件

### 6.5 相关文档

- `history.js` - 历史记录存储实现
- `inputModes.js` - 输入模式检测
- `useHistorySearch.ts` - 历史搜索 Hook（相关功能）
