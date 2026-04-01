# useTurnDiffs.ts 深度研究文档

## 场景与职责

`useTurnDiffs` 是一个 React Hook，用于从消息历史中提取基于回合（turn）的差异信息。它分析用户提示和助手响应，收集文件编辑操作，生成每回合的文件变更统计。

### 核心职责

1. **回合检测**: 识别用户提示开始的每个回合
2. **文件编辑收集**: 收集每个回合中的文件编辑操作
3. **差异统计**: 计算每个回合的文件变更统计（新增/删除行数）
4. **增量处理**: 只处理新增消息，避免重复计算
5. **回滚支持**: 支持消息历史回滚时的缓存重置

### 使用场景

- **差异显示**: 在 UI 中显示每回合的文件变更摘要
- **代码审查**: 帮助用户回顾每次对话的文件修改
- **统计信息**: 显示代码变更的统计信息
- **变更追踪**: 追踪哪些回合修改了哪些文件

---

## 功能点目的

### 1. 回合定义

一个回合定义为：
- 以用户提示开始（非工具结果、非 meta 消息）
- 包含随后的助手响应和工具结果
- 直到下一个用户提示开始新回合

### 2. 文件编辑识别

识别两种文件编辑工具：
- **FileEditTool**: 文件编辑，包含结构化补丁
- **FileWriteTool**: 文件写入（创建或更新），可能包含内容

### 3. 增量累积

使用缓存实现增量处理：
- 只处理上次处理后的新消息
- 缓存已完成的回合和当前回合
- 消息历史回滚时重置缓存

### 4. 统计计算

每个回合的统计信息：
- 变更文件数
- 新增行数
- 删除行数

---

## 具体技术实现

### 关键数据结构

```typescript
// 回合文件差异
interface TurnFileDiff {
  filePath: string
  hunks: StructuredPatchHunk[]
  isNewFile: boolean
  linesAdded: number
  linesRemoved: number
}

// 回合差异
interface TurnDiff {
  turnIndex: number
  userPromptPreview: string
  timestamp: string
  files: Map<string, TurnFileDiff>
  stats: {
    filesChanged: number
    linesAdded: number
    linesRemoved: number
  }
}

// 缓存结构
interface TurnDiffCache {
  completedTurns: TurnDiff[]       // 已完成的回合
  currentTurn: TurnDiff | null     // 当前回合
  lastProcessedIndex: number       // 上次处理的索引
  lastTurnIndex: number            // 回合计数器
}

// 文件编辑结果联合类型
type FileEditResult = FileEditOutput | FileWriteOutput
```

### 核心流程

#### 1. 增量处理流程
```
useMemo 执行
  ↓
检查消息长度 < lastProcessedIndex
  ↓
是 → 重置缓存（用户回滚了对话）
  ↓
遍历新消息（lastProcessedIndex 到 messages.length）
  ↓
跳过非用户消息
  ↓
检查是否为工具结果消息
  ↓
不是工具结果且不是 meta → 开始新回合
  ↓
是工具结果且 currentTurn 存在 → 收集文件编辑
  ↓
更新 lastProcessedIndex
  ↓
构建结果数组（completedTurns + currentTurn）
  ↓
反转数组（最新回合在前）
  ↓
返回结果
```

#### 2. 新回合开始逻辑（行 130-144）
```typescript
if (!isToolResult && !message.isMeta) {
  // Start a new turn on user prompt
  if (c.currentTurn && c.currentTurn.files.size > 0) {
    computeTurnStats(c.currentTurn)
    c.completedTurns.push(c.currentTurn)
  }

  c.lastTurnIndex++
  c.currentTurn = {
    turnIndex: c.lastTurnIndex,
    userPromptPreview: getUserPromptPreview(message),
    timestamp: message.timestamp,
    files: new Map(),
    stats: { filesChanged: 0, linesAdded: 0, linesRemoved: 0 },
  }
}
```

#### 3. 文件编辑收集逻辑（行 145-196）
```typescript
} else if (c.currentTurn && message.toolUseResult) {
  const result = message.toolUseResult
  if (isFileEditResult(result)) {
    const { filePath, structuredPatch } = result
    const isNewFile = 'type' in result && result.type === 'create'

    // Get or create file entry
    let fileEntry = c.currentTurn.files.get(filePath)
    if (!fileEntry) {
      fileEntry = {
        filePath,
        hunks: [],
        isNewFile,
        linesAdded: 0,
        linesRemoved: 0,
      }
      c.currentTurn.files.set(filePath, fileEntry)
    }

    // Handle new file with synthetic hunk
    if (isNewFile && structuredPatch.length === 0 && isFileWriteOutput(result)) {
      const content = result.content
      const lines = content.split('\n')
      const syntheticHunk: StructuredPatchHunk = {
        oldStart: 0,
        oldLines: 0,
        newStart: 1,
        newLines: lines.length,
        lines: lines.map(l => '+' + l),
      }
      fileEntry.hunks.push(syntheticHunk)
      fileEntry.linesAdded += lines.length
    } else {
      // Append hunks
      fileEntry.hunks.push(...structuredPatch)
      const { added, removed } = countHunkLines(structuredPatch)
      fileEntry.linesAdded += added
      fileEntry.linesRemoved += removed
    }

    // Mark as new file if created
    if (isNewFile) {
      fileEntry.isNewFile = true
    }
  }
}
```

### 关键代码路径

#### 文件编辑结果类型守卫（行 36-47）
```typescript
function isFileEditResult(result: unknown): result is FileEditResult {
  if (!result || typeof result !== 'object') return false
  const r = result as Record<string, unknown>
  const hasFilePath = typeof r.filePath === 'string'
  const hasStructuredPatch =
    Array.isArray(r.structuredPatch) && r.structuredPatch.length > 0
  const isNewFile = r.type === 'create' && typeof r.content === 'string'
  return hasFilePath && (hasStructuredPatch || isNewFile)
}
```

#### 新文件合成 Hunk（行 166-180）
```typescript
if (
  isNewFile &&
  structuredPatch.length === 0 &&
  isFileWriteOutput(result)
) {
  const content = result.content
  const lines = content.split('\n')
  const syntheticHunk: StructuredPatchHunk = {
    oldStart: 0,
    oldLines: 0,
    newStart: 1,
    newLines: lines.length,
    lines: lines.map(l => '+' + l),
  }
  fileEntry.hunks.push(syntheticHunk)
  fileEntry.linesAdded += lines.length
}
```

新文件的特殊处理：由于 `structuredPatch` 为空，需要从 `content` 生成合成的 hunk。

#### 行数统计（行 55-68）
```typescript
function countHunkLines(hunks: StructuredPatchHunk[]): {
  added: number
  removed: number
} {
  let added = 0
  let removed = 0
  for (const hunk of hunks) {
    for (const line of hunk.lines) {
      if (line.startsWith('+')) added++
      else if (line.startsWith('-')) removed++
    }
  }
  return { added, removed }
}
```

#### 用户提示预览（行 70-77）
```typescript
function getUserPromptPreview(message: Message): string {
  if (message.type !== 'user') return ''
  const content = message.message.content
  const text = typeof content === 'string' ? content : ''
  if (text.length <= 30) return text
  return text.slice(0, 29) + '…'
}
```

---

## 依赖与外部交互

### 核心依赖

| 模块 | 用途 |
|------|------|
| `diff` | `StructuredPatchHunk` 类型 |
| `react` | `useMemo`, `useRef` |
| `../tools/FileEditTool/types.js` | `FileEditOutput` |
| `../tools/FileWriteTool/FileWriteTool.js` | `FileWriteOutput` |
| `../types/message.js` | `Message` 类型 |

### 外部交互

1. **消息系统**: 
   - 分析 `messages` 数组提取回合信息
   - 识别用户消息、助手消息、工具结果

2. **工具系统**: 
   - 识别 `FileEditTool` 和 `FileWriteTool` 的输出
   - 提取 `structuredPatch` 和 `content`

---

## 风险、边界与改进建议

### 已知风险

1. **缓存内存占用**: 大型对话的缓存可能占用较多内存
2. **引用稳定性**: `useMemo` 依赖 `messages` 数组引用，如果父组件每次创建新数组会导致重复计算
3. **回滚检测**: 通过长度比较检测回滚，可能误判（如消息替换）

### 边界情况

1. **空消息历史**: 返回空数组
2. **无文件编辑**: 回合不包含文件编辑时不会出现在结果中
3. **同一文件多次编辑**: 同一回合中同一文件被多次编辑，hunks 会累积
4. **工具结果无编辑**: 工具结果不是文件编辑时忽略

### 改进建议

1. **深度比较**: 使用深度比较或消息 ID 检测变化，避免不必要的重计算
2. **缓存持久化**: 考虑将缓存持久化到 localStorage，支持页面刷新恢复
3. **增量序列化**: 支持将差异信息序列化，用于导出或分享
4. **文件过滤**: 支持按文件路径过滤显示的差异
5. **差异对比**: 支持对比不同回合之间的差异
6. **性能优化**: 对于超大型对话，考虑虚拟化或分页

### 测试关注点

1. 回合检测的准确性
2. 文件编辑的正确收集
3. 同一文件多次编辑的累积
4. 新文件合成 hunk 的正确性
5. 消息回滚时的缓存重置
6. 行数统计的准确性
7. 空消息历史和边界情况
