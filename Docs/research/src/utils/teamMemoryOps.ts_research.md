# teamMemoryOps.ts 研究文档

## 场景与职责

`teamMemoryOps.ts` 提供了团队内存（team memory）文件操作的检测和摘要功能。该模块用于识别工具调用是否针对团队内存文件，并在搜索/读取摘要中生成相应的描述文本。

## 功能点目的

### 团队内存操作检测
- **搜索检测** (`isTeamMemorySearch`): 检查搜索工具是否针对团队内存路径
- **写入检测** (`isTeamMemoryWriteOrEdit`): 检查 Write/Edit 工具是否针对团队内存文件

### 摘要文本生成
- **摘要追加** (`appendTeamMemorySummaryParts`): 将团队内存操作描述追加到摘要数组
- **动词时态**: 根据活动状态选择进行时或过去时

## 具体技术实现

### 团队内存路径检测

```typescript
export function isTeamMemorySearch(toolInput: unknown): boolean {
  const input = toolInput as
    | { path?: string; pattern?: string; glob?: string }
    | undefined
  if (!input) {
    return false
  }
  if (input.path && isTeamMemFile(input.path)) {
    return true
  }
  return false
}

export function isTeamMemoryWriteOrEdit(
  toolName: string,
  toolInput: unknown,
): boolean {
  if (toolName !== FILE_WRITE_TOOL_NAME && toolName !== FILE_EDIT_TOOL_NAME) {
    return false
  }
  const input = toolInput as { file_path?: string; path?: string } | undefined
  const filePath = input?.file_path ?? input?.path
  return filePath !== undefined && isTeamMemFile(filePath)
}
```

### 摘要生成

```typescript
export function appendTeamMemorySummaryParts(
  memoryCounts: {
    teamMemoryReadCount?: number
    teamMemorySearchCount?: number
    teamMemoryWriteCount?: number
  },
  isActive: boolean,
  parts: string[],
): void {
  const teamReadCount = memoryCounts.teamMemoryReadCount ?? 0
  const teamSearchCount = memoryCounts.teamMemorySearchCount ?? 0
  const teamWriteCount = memoryCounts.teamMemoryWriteCount ?? 0

  // 读取操作
  if (teamReadCount > 0) {
    const verb = isActive
      ? parts.length === 0 ? 'Recalling' : 'recalling'
      : parts.length === 0 ? 'Recalled' : 'recalled'
    parts.push(`${verb} ${teamReadCount} team ${teamReadCount === 1 ? 'memory' : 'memories'}`)
  }

  // 搜索操作
  if (teamSearchCount > 0) {
    const verb = isActive
      ? parts.length === 0 ? 'Searching' : 'searching'
      : parts.length === 0 ? 'Searched' : 'searched'
    parts.push(`${verb} team memories`)
  }

  // 写入操作
  if (teamWriteCount > 0) {
    const verb = isActive
      ? parts.length === 0 ? 'Writing' : 'writing'
      : parts.length === 0 ? 'Wrote' : 'wrote'
    parts.push(`${verb} ${teamWriteCount} team ${teamWriteCount === 1 ? 'memory' : 'memories'}`)
  }
}
```

## 关键代码路径与文件引用

### 本文件导出
- `isTeamMemorySearch(toolInput)`: 检测团队内存搜索
- `isTeamMemoryWriteOrEdit(toolName, toolInput)`: 检测团队内存写入/编辑
- `appendTeamMemorySummaryParts(memoryCounts, isActive, parts)`: 追加摘要
- `isTeamMemFile`: 从 `../memdir/teamMemPaths.js` 重新导出

### 依赖模块

| 模块 | 用途 |
|------|------|
| `../memdir/teamMemPaths.js` | `isTeamMemFile` |
| `../tools/FileEditTool/constants.js` | `FILE_EDIT_TOOL_NAME` |
| `../tools/FileWriteTool/prompt.js` | `FILE_WRITE_TOOL_NAME` |

### 调用方

| 文件 | 用途 |
|------|------|
| `src/utils/collapseReadSearch.ts` | 检测团队内存操作并生成摘要 |

### 调用代码片段

```typescript
// src/utils/collapseReadSearch.ts
const teamMemOps = feature('TEAMMEM')
  ? (require('./teamMemoryOps.js') as typeof import('./teamMemoryOps.js'))
  : null

// 检测团队内存搜索
if (teamMemOps?.isTeamMemorySearch(toolInput)) {
  return { isCollapsible: true, isSearch: true, /* ... */ }
}

// 生成摘要
teamMemOps?.appendTeamMemorySummaryParts(memoryCounts, isActive, parts)
```

## 依赖与外部交互

### 与团队内存路径的集成
- 使用 `isTeamMemFile` 检测路径是否在团队内存目录内
- 团队内存路径: `<memoryBase>/projects/<project>/memory/team/`
- 通过 `feature('TEAMMEM')` 进行条件加载

### 与工具系统的集成
- 检测 `FileWrite` 和 `FileEdit` 工具
- 检测搜索工具的 `path` 参数
- 硬编码工具名称常量

### 与摘要系统的集成
- 修改传入的 `parts` 数组（副作用 API）
- 根据 `isActive` 选择动词时态
- 处理单复数形式

## 风险、边界与改进建议

### 潜在风险

1. **硬编码工具名**: 工具名称变更需要同步更新
2. **副作用 API**: `appendTeamMemorySummaryParts` 修改输入数组，可能导致意外
3. **类型安全**: 使用 `unknown` 和类型断言，运行时可能出错

### 边界情况

1. **空路径**: 路径为空字符串时返回 `false`
2. **相对路径**: 依赖 `isTeamMemFile` 正确处理相对路径
3. **计数为零**: 所有计数为零时不添加任何部分

### 改进建议

1. **纯函数 API**: 改为返回新数组而非修改输入
```typescript
export function getTeamMemorySummaryParts(
  memoryCounts: MemoryCounts,
  isActive: boolean,
  existingParts: string[]
): string[] {
  const parts: string[] = []
  // ... 生成 parts
  return [...existingParts, ...parts]
}
```

2. **更多操作类型**: 支持删除、重命名等操作
```typescript
export function isTeamMemoryDelete(toolName: string, toolInput: unknown): boolean
export function isTeamMemoryMove(toolName: string, toolInput: unknown): boolean
```

3. **详细摘要**: 添加文件级别的详细信息
```typescript
export interface TeamMemoryOperation {
  type: 'read' | 'search' | 'write'
  path: string
  timestamp: number
}

export function getDetailedSummary(operations: TeamMemoryOperation[]): string
```

4. **国际化**: 支持多语言摘要
```typescript
export function appendTeamMemorySummaryParts(
  memoryCounts: MemoryCounts,
  isActive: boolean,
  parts: string[],
  locale?: string
): void
```

5. **配置化动词**: 允许自定义动词
```typescript
export interface VerbConfig {
  active: { first: string; subsequent: string }
  completed: { first: string; subsequent: string }
}
```

6. **类型安全**: 使用更严格的类型定义
```typescript
interface ToolInputWithPath {
  path?: string
  file_path?: string
}

function hasPath(input: unknown): input is ToolInputWithPath {
  return typeof input === 'object' && input !== null && ('path' in input || 'file_path' in input)
}
```
