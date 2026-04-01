# useInputBuffer.ts 深度研究文档

## 场景与职责

`useInputBuffer` 是一个通用的输入缓冲区管理钩子，用于实现输入内容的撤销（undo）功能。它主要服务于 REPL 的命令行输入组件，提供类似编辑器的撤销体验。

### 核心场景

1. **命令行输入撤销**：用户在输入复杂命令时，可以撤销到之前的状态
2. **粘贴内容管理**：跟踪粘贴内容的位置，支持智能撤销
3. **防抖处理**：避免快速输入时频繁创建缓冲区条目

### 与其他组件的关系

- 被 `PromptInput` 组件使用，管理输入框的状态历史
- 与粘贴处理系统集成，记录粘贴内容的位置信息

---

## 功能点目的

### 1. 缓冲区条目管理

每个缓冲区条目包含：
- `text`: 输入文本内容
- `cursorOffset`: 光标位置
- `pastedContents`: 粘贴内容的记录（位置映射）
- `timestamp`: 时间戳

### 2. 防抖机制

防止快速输入时创建过多缓冲区条目：
- 默认防抖间隔：300ms
- 如果变化间隔小于防抖时间，使用 setTimeout 延迟提交
- 新的变化会取消之前的待处理提交

### 3. 撤销功能

- 支持逐步撤销到之前的输入状态
- 维护当前索引，支持撤销后重新编辑
- 撤销时返回完整的条目信息（文本、光标位置、粘贴内容）

### 4. 缓冲区大小限制

- 默认最大条目数：50
- 超出限制时丢弃最旧的条目
- 防止内存无限增长

---

## 具体技术实现

### 关键数据结构

```typescript
export type BufferEntry = {
  text: string
  cursorOffset: number
  pastedContents: Record<number, PastedContent>
  timestamp: number
}

export type UseInputBufferProps = {
  maxBufferSize: number   // 默认 50
  debounceMs: number      // 默认 300ms
}

export type UseInputBufferResult = {
  pushToBuffer: (text: string, cursorOffset: number, pastedContents?: Record<number, PastedContent>) => void
  undo: () => BufferEntry | undefined
  canUndo: boolean
  clearBuffer: () => void
}
```

### 核心算法

#### 防抖提交逻辑

```typescript
const pushToBuffer = useCallback((text, cursorOffset, pastedContents) => {
  const now = Date.now()
  
  // 清除待处理的提交
  if (pendingPush.current) {
    clearTimeout(pendingPush.current)
    pendingPush.current = null
  }
  
  // 如果变化太快，延迟提交
  if (now - lastPushTime.current < debounceMs) {
    pendingPush.current = setTimeout(
      pushToBuffer,
      debounceMs,
      text,
      cursorOffset,
      pastedContents
    )
    return
  }
  
  lastPushTime.current = now
  // ... 实际提交逻辑
}, [debounceMs, maxBufferSize, currentIndex, buffer.length])
```

#### 撤销逻辑

```typescript
const undo = useCallback((): BufferEntry | undefined => {
  if (currentIndex < 0 || buffer.length === 0) {
    return undefined
  }
  
  const targetIndex = Math.max(0, currentIndex - 1)
  const entry = buffer[targetIndex]
  
  if (entry) {
    setCurrentIndex(targetIndex)
    return entry
  }
  return undefined
}, [buffer, currentIndex])
```

#### 缓冲区截断逻辑

```typescript
setBuffer(prevBuffer => {
  // 如果在撤销后编辑，截断当前位置之后的历史
  const newBuffer = currentIndex >= 0 
    ? prevBuffer.slice(0, currentIndex + 1) 
    : prevBuffer
  
  // 如果与最后一条相同，不添加
  const lastEntry = newBuffer[newBuffer.length - 1]
  if (lastEntry && lastEntry.text === text) {
    return newBuffer
  }
  
  // 添加新条目
  const updatedBuffer = [...newBuffer, { text, cursorOffset, pastedContents, timestamp: now }]
  
  // 限制大小
  if (updatedBuffer.length > maxBufferSize) {
    return updatedBuffer.slice(-maxBufferSize)
  }
  return updatedBuffer
})
```

---

## 关键代码路径与文件引用

```
src/hooks/useInputBuffer.ts
├── BufferEntry 类型定义           # 行 4-9
├── UseInputBufferProps 类型       # 行 11-14
├── UseInputBufferResult 类型      # 行 16-25
├── useInputBuffer()               # 行 27-132
│   ├── pushToBuffer()             # 行 36-96: 防抖提交
│   ├── undo()                     # 行 98-112: 撤销操作
│   ├── clearBuffer()              # 行 114-122: 清空缓冲区
│   └── canUndo 计算               # 行 124: 撤销可用性
```

### 使用位置

```
src/components/PromptInput/PromptInput.tsx
├── 初始化: useInputBuffer({ maxBufferSize: 50, debounceMs: 300 })
├── 输入变化: pushToBuffer(inputValue, cursorOffset, pastedContents)
└── 撤销快捷键: undo() 恢复之前状态
```

---

## 依赖与外部交互

### 依赖类型

```typescript
import type { PastedContent } from '../utils/config.js'
```

`PastedContent` 定义：
```typescript
type PastedContent = {
  content: string
  timestamp: number
  // 其他元数据...
}
```

### React Hooks 使用

- `useState`: 管理 buffer 和 currentIndex 状态
- `useRef`: 管理 lastPushTime 和 pendingPush
- `useCallback`: 缓存 pushToBuffer、undo、clearBuffer 函数

---

## 风险、边界与改进建议

### 已知风险

1. **防抖延迟**
   - 如果用户在防抖期间快速输入然后立即撤销，可能撤销到不一致的状态
   - 缓解：防抖时间较短（300ms），影响有限

2. **内存使用**
   - 虽然限制了条目数，但每个条目包含完整文本
   - 大量长文本输入时内存占用可能较高

3. **并发修改**
   - 如果外部直接修改输入值而不通过 pushToBuffer，缓冲区与实际输入可能不同步

### 边界情况

| 场景 | 行为 |
|-----|------|
| 缓冲区为空时撤销 | 返回 undefined |
| 重复文本提交 | 不添加新条目 |
| 撤销到开头后继续撤销 | 保持在索引 0 |
| 防抖期间组件卸载 | 待处理的 setTimeout 可能泄漏（已修复） |

### 改进建议

1. **重做（Redo）支持**
   - 当前只支持撤销，可以扩展支持重做
   - 需要维护一个"未来"缓冲区

2. **选择性撤销**
   - 支持按类型撤销（如只撤销粘贴操作）
   - 需要扩展 BufferEntry 元数据

3. **持久化**
   - 会话间保留输入历史
   - 需要与 sessionStorage 集成

4. **压缩优化**
   - 对相似条目使用 diff 存储，减少内存占用
   - 适用于长文本编辑场景

5. **可视化**
   - 显示撤销历史列表，支持跳转到特定版本
   - 类似编辑器的本地历史功能

### 测试建议

1. **单元测试**：
   - 防抖行为测试
   - 缓冲区大小限制测试
   - 撤销/重做流程测试

2. **集成测试**：
   - 与 PromptInput 组件集成
   - 粘贴操作与撤销的交互
