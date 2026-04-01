# SessionMemory/sessionMemoryUtils.ts 研究文档

## 场景与职责

`sessionMemoryUtils.ts` 是 Session Memory 系统的状态管理工具模块，设计目标：
1. **避免循环依赖**：将不依赖 `runAgent` 的工具函数独立出来，供其他模块安全导入
2. **状态管理**：维护 Session Memory 相关的运行时状态（配置、提取状态、token 记录等）
3. **阈值判断**：提供初始化阈值和更新阈值的检查函数
4. **内容读取**：提供读取 Session Memory 文件内容的接口
5. **同步等待**：提供等待提取完成的机制（用于 compaction 协调）

该模块是 Session Memory 系统的"状态仓库"，被 `sessionMemory.ts`（核心协调）和 `sessionMemoryCompact.ts`（压缩集成）共同依赖。

## 功能点目的

### 1. 配置管理
维护 Session Memory 的三项阈值配置：
- `minimumMessageTokensToInit`: 初始化阈值（默认 10000 tokens）
- `minimumTokensBetweenUpdate`: 更新阈值（默认 5000 tokens）
- `toolCallsBetweenUpdates`: 工具调用阈值（默认 3 次）

支持通过 `setSessionMemoryConfig` 动态更新配置（来自 GrowthBook 远程配置）。

### 2. 提取状态管理
跟踪记忆提取的生命周期：
- `markExtractionStarted()`: 标记提取开始（记录时间戳）
- `markExtractionCompleted()`: 标记提取完成（清除时间戳）
- `waitForSessionMemoryExtraction()`: 等待提取完成（带 15s 超时和 1min 过期机制）

### 3. Token 记录与阈值判断
- `recordExtractionTokenCount()`: 记录上次提取时的总 token 数
- `hasMetInitializationThreshold()`: 检查是否达到初始化阈值
- `hasMetUpdateThreshold()`: 检查自上次提取以来的 token 增长是否达到更新阈值

### 4. 消息 ID 追踪
- `setLastSummarizedMessageId()`: 设置上次提取覆盖到的消息 UUID
- `getLastSummarizedMessageId()`: 获取该 UUID（用于 compaction 判断保留哪些消息）

### 5. 文件内容读取
- `getSessionMemoryContent()`: 读取 Session Memory 文件内容，处理文件不存在等错误

### 6. 初始化状态
- `markSessionMemoryInitialized()`: 标记 Session Memory 已初始化
- `isSessionMemoryInitialized()`: 检查是否已初始化

## 具体技术实现

### 状态变量
```typescript
// 当前配置（可被远程配置覆盖）
let sessionMemoryConfig: SessionMemoryConfig = {
  ...DEFAULT_SESSION_MEMORY_CONFIG,
}

// 上次提取覆盖到的消息 ID
let lastSummarizedMessageId: string | undefined

// 提取开始时间戳（undefined 表示未在提取）
let extractionStartedAt: number | undefined

// 上次提取时的总 token 数
let tokensAtLastExtraction = 0

// 是否已初始化
let sessionMemoryInitialized = false
```

### 超时与过期机制
```typescript
const EXTRACTION_WAIT_TIMEOUT_MS = 15000        // 15 秒等待超时
const EXTRACTION_STALE_THRESHOLD_MS = 60000     // 1 分钟过期阈值

export async function waitForSessionMemoryExtraction(): Promise<void> {
  const startTime = Date.now()
  while (extractionStartedAt) {
    const extractionAge = Date.now() - extractionStartedAt
    
    // 提取超过 1 分钟视为过期，不再等待
    if (extractionAge > EXTRACTION_STALE_THRESHOLD_MS) {
      return
    }
    
    // 等待超过 15 秒超时，不再等待
    if (Date.now() - startTime > EXTRACTION_WAIT_TIMEOUT_MS) {
      return
    }
    
    await sleep(1000)  // 每秒检查一次
  }
}
```

### 阈值计算
```typescript
// 初始化阈值检查
export function hasMetInitializationThreshold(currentTokenCount: number): boolean {
  return currentTokenCount >= sessionMemoryConfig.minimumMessageTokensToInit
}

// 更新阈值检查（基于自上次提取的增长量）
export function hasMetUpdateThreshold(currentTokenCount: number): boolean {
  const tokensSinceLastExtraction = currentTokenCount - tokensAtLastExtraction
  return tokensSinceLastExtraction >= sessionMemoryConfig.minimumTokensBetweenUpdate
}
```

### 文件读取与错误处理
```typescript
export async function getSessionMemoryContent(): Promise<string | null> {
  const fs = getFsImplementation()
  const memoryPath = getSessionMemoryPath()

  try {
    const content = await fs.readFile(memoryPath, { encoding: 'utf-8' })
    logEvent('tengu_session_memory_loaded', { content_length: content.length })
    return content
  } catch (e: unknown) {
    // 文件不可访问（不存在或无权限）返回 null
    if (isFsInaccessible(e)) return null
    throw e  // 其他错误抛出
  }
}
```

### 状态重置（测试用）
```typescript
export function resetSessionMemoryState(): void {
  sessionMemoryConfig = { ...DEFAULT_SESSION_MEMORY_CONFIG }
  tokensAtLastExtraction = 0
  sessionMemoryInitialized = false
  lastSummarizedMessageId = undefined
  extractionStartedAt = undefined
}
```

## 关键代码路径与文件引用

### 导出函数
| 函数名 | 用途 | 主要调用方 |
|--------|------|------------|
| `getSessionMemoryConfig()` | 获取当前配置 | `sessionMemory.ts` |
| `setSessionMemoryConfig()` | 设置配置 | `sessionMemory.ts:initSessionMemoryConfigIfNeeded` |
| `getLastSummarizedMessageId()` | 获取上次提取消息 ID | `sessionMemoryCompact.ts` |
| `setLastSummarizedMessageId()` | 设置上次提取消息 ID | `sessionMemory.ts`, `autoCompact.ts`, `compact.ts` |
| `markExtractionStarted()` | 标记提取开始 | `sessionMemory.ts` |
| `markExtractionCompleted()` | 标记提取完成 | `sessionMemory.ts` |
| `waitForSessionMemoryExtraction()` | 等待提取完成 | `sessionMemoryCompact.ts:trySessionMemoryCompaction` |
| `getSessionMemoryContent()` | 读取记忆文件 | `sessionMemoryCompact.ts`, `awaySummary.ts` |
| `recordExtractionTokenCount()` | 记录提取时 token 数 | `sessionMemory.ts` |
| `isSessionMemoryInitialized()` | 检查是否已初始化 | `sessionMemory.ts:shouldExtractMemory` |
| `markSessionMemoryInitialized()` | 标记已初始化 | `sessionMemory.ts:shouldExtractMemory` |
| `hasMetInitializationThreshold()` | 检查初始化阈值 | `sessionMemory.ts:shouldExtractMemory` |
| `hasMetUpdateThreshold()` | 检查更新阈值 | `sessionMemory.ts:shouldExtractMemory` |
| `getToolCallsBetweenUpdates()` | 获取工具调用阈值 | `sessionMemory.ts:shouldExtractMemory` |
| `resetSessionMemoryState()` | 重置所有状态 | 测试代码 |

### 导入依赖
```typescript
// 文件系统
import { getFsImplementation } from '../../utils/fsOperations.js'
import { getSessionMemoryPath } from '../../utils/permissions/filesystem.js'

// 错误处理
import { isFsInaccessible } from '../../utils/errors.js'

// 工具函数
import { sleep } from '../../utils/sleep.js'

// 分析日志
import { logEvent } from '../analytics/index.js'
```

## 依赖与外部交互

### 被调用方

1. **`sessionMemory.ts`**
   - 配置管理：`getSessionMemoryConfig`, `setSessionMemoryConfig`
   - 提取状态：`markExtractionStarted`, `markExtractionCompleted`
   - 阈值判断：`hasMetInitializationThreshold`, `hasMetUpdateThreshold`, `getToolCallsBetweenUpdates`
   - 初始化状态：`isSessionMemoryInitialized`, `markSessionMemoryInitialized`
   - Token 记录：`recordExtractionTokenCount`
   - 消息 ID：`setLastSummarizedMessageId`

2. **`sessionMemoryCompact.ts`**
   - `getLastSummarizedMessageId`: 确定 compaction 时保留消息的边界
   - `waitForSessionMemoryExtraction`: 确保提取完成后再进行 compaction
   - `getSessionMemoryContent`: 获取记忆内容用于生成摘要

3. **`autoCompact.ts`**
   - `setLastSummarizedMessageId(undefined)`: compaction 后重置消息 ID

4. **`compact.ts`**（传统压缩命令）
   - `setLastSummarizedMessageId(undefined)`: compaction 后重置消息 ID

5. **`awaySummary.ts`**（离开摘要）
   - `getSessionMemoryContent`: 获取记忆内容用于生成"你离开期间"摘要

### 依赖的底层服务

1. **`fsOperations.ts`**
   - `getFsImplementation`: 获取文件系统操作实现（支持虚拟文件系统）

2. **`filesystem.ts`**
   - `getSessionMemoryPath`: 获取记忆文件路径

3. **`errors.ts`**
   - `isFsInaccessible`: 判断错误是否为文件不可访问（ENOENT、EACCES、EPERM）

4. **`sleep.ts`**
   - `sleep`: 异步等待工具

5. **`analytics/index.ts`**
   - `logEvent`: 记录分析事件

## 风险、边界与改进建议

### 风险点

1. **模块级状态共享**
   - 风险：所有状态都是模块级变量，在测试或并发场景下可能相互干扰
   - 缓解：`resetSessionMemoryState` 函数用于测试清理
   - 但：生产环境无隔离机制

2. **提取状态过期机制**
   - 风险：提取超过 1 分钟后被视为过期，但可能只是提取耗时较长
   - 影响：`waitForSessionMemoryExtraction` 可能过早返回，导致 compaction 与提取竞争

3. **Token 计算一致性**
   - 风险：`tokensAtLastExtraction` 使用 `tokenCountWithEstimation` 的值，与 API 实际 token 可能有偏差
   - 影响：阈值判断可能不准确

4. **消息 ID 生命周期**
   - 风险：`lastSummarizedMessageId` 在 compaction 后被重置为 undefined
   - 影响：恢复会话时可能需要重新初始化

5. **文件读取错误处理**
   - `getSessionMemoryContent` 对所有 FS 不可访问错误返回 null
   - 可能掩盖权限问题

### 边界情况

1. **负 Token 增长**
   - `hasMetUpdateThreshold` 计算 `currentTokenCount - tokensAtLastExtraction`
   - 如果消息被删除（如 snip），差值可能为负，永远不会触发更新
   - 实际：compaction 会重置 `lastSummarizedMessageId`，间接重置 `tokensAtLastExtraction`

2. **并发提取**
   - `extractionStartedAt` 是简单的时间戳，无互斥机制
   - 依赖 `sessionMemory.ts` 的 `sequential` 包装器确保串行

3. **配置更新时机**
   - `setSessionMemoryConfig` 可被随时调用
   - 但配置变更不会立即影响正在进行的提取

4. **空内容处理**
   - `getSessionMemoryContent` 返回空字符串（文件存在但为空）与 null（文件不存在）
   - 调用方需要正确处理这两种情况

### 改进建议

1. **状态持久化**
   - 考虑将 `tokensAtLastExtraction` 和 `lastSummarizedMessageId` 持久化到磁盘
   - 支持会话恢复后保持连续性

2. **更精确的提取状态**
   - 使用 Promise 或 EventEmitter 替代轮询等待
   - 提供更精确的提取完成通知

3. **配置变更通知**
   - 添加配置变更回调机制
   - 支持动态调整阈值而不重启

4. **增强错误处理**
   - 区分不同类型的文件系统错误
   - 添加重试机制

5. **状态快照**
   - 提供获取完整状态快照的函数
   - 便于调试和状态恢复

6. **Token 计算优化**
   - 考虑使用更精确的 token 计数方式
   - 或与 API 实际使用的 token 对齐

7. **提取取消机制**
   - 当前无取消进行中的提取的机制
   - 可考虑添加 AbortController 支持
