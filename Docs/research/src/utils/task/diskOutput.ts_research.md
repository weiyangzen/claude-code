# diskOutput.ts 深度研究文档

## 场景与职责

`diskOutput.ts` 是 Claude Code 中任务输出文件的**磁盘持久化层**。它负责管理所有任务输出文件的创建、追加写入、读取、清理和生命周期管理。该模块是 `TaskOutput.ts` 的底层依赖，同时也被其他模块直接用于任务输出操作。

### 核心职责

1. **文件路径管理**: 生成任务输出文件的标准化路径
2. **异步写入**: 通过 `DiskTaskOutput` 类实现高效的异步文件追加
3. **读取操作**: 支持增量读取 (`getTaskOutputDelta`) 和尾部读取 (`getTaskOutput`)
4. **生命周期管理**: 初始化、刷新、清理任务输出文件
5. **安全控制**: 5GB 磁盘上限，防止磁盘填充攻击

### 使用场景

| 场景 | 调用方 | 功能 |
|------|--------|------|
| Bash 命令输出 | `Shell.ts`, `TaskOutput.ts` | 创建输出文件，子进程直接写入 fd |
| Hooks 输出溢出 | `TaskOutput.ts` | 内存超限后溢出到磁盘 |
| 任务状态读取 | `framework.ts` | 增量读取新输出用于通知 |
| 任务清理 | `killShellTasks.ts`, `compact.ts` | 删除任务输出文件 |
| 代理转录 | `AgentTool.tsx` | 创建符号链接到代理转录文件 |

---

## 功能点目的

### 1. 安全的文件路径生成

```typescript
// 路径结构: {projectTempDir}/{sessionId}/tasks/{taskId}.output
// 示例: /tmp/claude-1000/home_user_project_abc123/tasks/b1a2b3c4.output
```

**安全特性**:
- 包含 `sessionId` 防止并发会话冲突
- 使用 `O_NOFOLLOW` 防止符号链接攻击
- 使用 `O_EXCL` 确保原子创建

### 2. 高效的异步写入 (`DiskTaskOutput`)

**设计目标**:
- 避免 `.then()` 链导致的内存保留问题
- 使用扁平队列 + 单次 drain 循环
- 每个 chunk 写入后立即 GC

**核心机制**:
```typescript
class DiskTaskOutput {
  #queue: string[] = []           // 写入队列
  #drain(): Promise<void>         // 批量写入循环
  #queueToBuffers(): Buffer       // 队列转 Buffer (使用 splice 释放内存)
}
```

### 3. 增量读取支持

```typescript
// 用于任务通知系统，只读取新增内容
getTaskOutputDelta(taskId, fromOffset, maxBytes)
  → { content: string, newOffset: number }
```

### 4. 5GB 磁盘上限

```typescript
export const MAX_TASK_OUTPUT_BYTES = 5 * 1024 * 1024 * 1024  // 5GB
export const MAX_TASK_OUTPUT_BYTES_DISPLAY = '5GB'
```

当超过上限时，追加截断标记并停止写入。

---

## 具体技术实现

### 关键数据结构

```typescript
class DiskTaskOutput {
  #path: string                    // 文件路径
  #fileHandle: FileHandle | null   // 文件句柄 (延迟打开)
  #queue: string[]                 // 待写入队列
  #bytesWritten = 0                // 已写入字节数 (UTF-16 code units)
  #capped = false                  // 是否已达上限
  #flushPromise: Promise<void> | null
  #flushResolve: (() => void) | null
}
```

### 核心流程

#### 1. 文件路径生成

```typescript
let _taskOutputDir: string | undefined

export function getTaskOutputDir(): string {
  if (_taskOutputDir === undefined) {
    // 首次调用时捕获 sessionId，避免 /clear 后 sessionId 变化导致路径不匹配
    _taskOutputDir = join(getProjectTempDir(), getSessionId(), 'tasks')
  }
  return _taskOutputDir
}
```

**关键设计**: `sessionId` 在首次调用时捕获，而不是每次重新读取。这确保背景任务在 `/clear` 后仍能访问其输出文件。

#### 2. 安全文件创建 (`initTaskOutput`)

```typescript
export function initTaskOutput(taskId: string): Promise<string> {
  return track((async () => {
    await ensureOutputDir()
    const outputPath = getTaskOutputPath(taskId)
    
    // SECURITY: O_NOFOLLOW 防止符号链接攻击
    // O_EXCL 确保创建新文件，如果已存在则失败
    const fh = await open(
      outputPath,
      process.platform === 'win32'
        ? 'wx'  // Windows: 使用字符串标志
        : fsConstants.O_WRONLY | 
          fsConstants.O_CREAT | 
          fsConstants.O_EXCL | 
          O_NOFOLLOW  // Unix: 数值标志
    )
    await fh.close()
    return outputPath
  })())
}
```

#### 3. 异步写入队列 (`DiskTaskOutput.append`)

```typescript
append(content: string): void {
  if (this.#capped) return
  
  // content.length (UTF-16) 可能低估 UTF-8 字节数最多 3 倍
  // 作为粗略的磁盘填充防护是可接受的
  this.#bytesWritten += content.length
  
  if (this.#bytesWritten > MAX_TASK_OUTPUT_BYTES) {
    this.#capped = true
    this.#queue.push(`\n[output truncated: exceeded ${MAX_TASK_OUTPUT_BYTES_DISPLAY} disk cap]\n`)
  } else {
    this.#queue.push(content)
  }
  
  if (!this.#flushPromise) {
    this.#flushPromise = new Promise(resolve => {
      this.#flushResolve = resolve
    })
    void track(this.#drain())  // 启动 drain 循环
  }
}
```

#### 4. Drain 循环 (`#drainAllChunks`)

```typescript
async #drainAllChunks(): Promise<void> {
  while (true) {
    try {
      if (!this.#fileHandle) {
        await ensureOutputDir()
        this.#fileHandle = await open(
          this.#path,
          process.platform === 'win32'
            ? 'a'
            : fsConstants.O_WRONLY | 
              fsConstants.O_APPEND | 
              fsConstants.O_CREAT | 
              O_NOFOLLOW
        )
      }
      
      while (true) {
        await this.#writeAllChunks()
        if (this.#queue.length === 0) break
      }
    } finally {
      if (this.#fileHandle) {
        const fileHandle = this.#fileHandle
        this.#fileHandle = null
        await fileHandle.close()
      }
    }
    
    // 检查关闭期间是否有新写入
    if (this.#queue.length) continue
    break
  }
}
```

#### 5. 批量写入 (`#writeAllChunks`)

```typescript
#writeAllChunks(): Promise<void> {
  // ⚠️ 极其精确：不要在这里添加 await！
  // 这会导致内存膨胀，因为队列增长时 Buffer[] 会被保留
  return this.#fileHandle!.appendFile(
    this.#queueToBuffers()  // 立即 GC
  )
}
```

#### 6. 队列转 Buffer (`#queueToBuffers`)

```typescript
#queueToBuffers(): Buffer {
  // 使用 splice 原地修改数组，通知 GC 可以释放
  const queue = this.#queue.splice(0, this.#queue.length)
  
  let totalLength = 0
  for (const str of queue) {
    totalLength += Buffer.byteLength(str, 'utf8')
  }
  
  const buffer = Buffer.allocUnsafe(totalLength)
  let offset = 0
  for (const str of queue) {
    offset += buffer.write(str, offset, 'utf8')
  }
  
  return buffer
}
```

**内存优化要点**:
- `splice(0, length)` 清空原数组，允许 GC 回收
- `Buffer.allocUnsafe` 避免零填充开销
- 单次 `buffer.write` 循环完成所有字符串写入

### 读取操作

#### 增量读取 (`getTaskOutputDelta`)

```typescript
export async function getTaskOutputDelta(
  taskId: string,
  fromOffset: number,
  maxBytes: number = DEFAULT_MAX_READ_BYTES,  // 8MB
): Promise<{ content: string; newOffset: number }> {
  try {
    const result = await readFileRange(
      getTaskOutputPath(taskId),
      fromOffset,
      maxBytes,
    )
    if (!result) {
      return { content: '', newOffset: fromOffset }
    }
    return {
      content: result.content,
      newOffset: fromOffset + result.bytesRead,
    }
  } catch (e) {
    const code = getErrnoCode(e)
    if (code === 'ENOENT') {
      return { content: '', newOffset: fromOffset }
    }
    logError(e)
    return { content: '', newOffset: fromOffset }
  }
}
```

#### 尾部读取 (`getTaskOutput`)

```typescript
export async function getTaskOutput(
  taskId: string,
  maxBytes: number = DEFAULT_MAX_READ_BYTES,
): Promise<string> {
  try {
    const { content, bytesTotal, bytesRead } = await tailFile(
      getTaskOutputPath(taskId),
      maxBytes,
    )
    if (bytesTotal > bytesRead) {
      return `[${Math.round((bytesTotal - bytesRead) / 1024)}KB of earlier output omitted]\n${content}`
    }
    return content
  } catch (e) {
    // ENOENT 返回空字符串，其他错误记录后返回空
  }
}
```

### 符号链接支持 (`initTaskOutputAsSymlink`)

用于代理转录文件，将任务输出链接到代理的转录文件:

```typescript
export function initTaskOutputAsSymlink(
  taskId: string,
  targetPath: string,
): Promise<string> {
  return track((async () => {
    try {
      await ensureOutputDir()
      const outputPath = getTaskOutputPath(taskId)
      
      try {
        await symlink(targetPath, outputPath)
      } catch {
        // 文件已存在，删除后重试
        await unlink(outputPath)
        await symlink(targetPath, outputPath)
      }
      
      return outputPath
    } catch (error) {
      logError(error)
      return initTaskOutput(taskId)  // 降级为普通文件
    }
  })())
}
```

---

## 关键代码路径与文件引用

### 核心调用链

```
Shell.ts:exec()
  ├── initTaskOutput(taskId)  [创建空文件]
  ├── open(outputPath)        [子进程写入]
  └── TaskOutput(taskId, onProgress, true)
      └── getTaskOutputPath(taskId)

TaskOutput.ts (pipe mode overflow)
  └── new DiskTaskOutput(taskId)
      ├── append(content)     [添加到队列]
      └── #drain()            [异步写入]

framework.ts:generateTaskAttachments()
  ├── getTaskOutputDelta(taskId, offset)  [增量读取]
  └── evictTaskOutput(taskId)             [清理]

AgentTool.tsx
  └── initTaskOutputAsSymlink(taskId, transcriptPath)
```

### 外部调用方

| 文件 | 调用函数 | 用途 |
|------|----------|------|
| `src/utils/task/TaskOutput.ts` | `DiskTaskOutput`, `getTaskOutputPath` | 磁盘写入和路径获取 |
| `src/utils/task/framework.ts` | `getTaskOutputDelta`, `evictTaskOutput`, `getTaskOutputPath` | 任务通知和清理 |
| `src/utils/Shell.ts` | `getTaskOutputDir`, `initTaskOutput` | Bash 命令输出文件 |
| `src/utils/ShellCommand.ts` | `MAX_TASK_OUTPUT_BYTES` | 大小看门狗限制 |
| `src/utils/hooks.ts` | `getTaskOutputPath` | Hooks 输出路径 |
| `src/tools/AgentTool/AgentTool.tsx` | `initTaskOutputAsSymlink` | 代理转录链接 |
| `src/tools/TaskOutputTool/TaskOutputTool.tsx` | `getTaskOutput` | 读取任务输出 |
| `src/tasks/LocalShellTask/killShellTasks.ts` | `cleanupTaskOutput` | 清理已杀死任务 |
| `src/services/compact/compact.ts` | `cleanupTaskOutput` | 压缩时清理 |

### 依赖文件

| 文件 | 用途 |
|------|------|
| `src/bootstrap/state.ts` | `getSessionId` |
| `src/utils/permissions/filesystem.ts` | `getProjectTempDir` |
| `src/utils/fsOperations.ts` | `readFileRange`, `tailFile` |
| `src/utils/errors.ts` | `getErrnoCode` |
| `src/utils/log.ts` | `logError` |

---

## 依赖与外部交互

### 模块依赖图

```
┌─────────────────────────────────────────────────────────────────┐
│                        diskOutput.ts                            │
├─────────────────────────────────────────────────────────────────┤
│  DiskTaskOutput (class)                                         │
│  ├── fs/promises (open, appendFile, close)                     │
│  ├── fsConstants (O_NOFOLLOW, O_WRONLY, etc.)                  │
│  └── track() (pending ops)                                      │
├─────────────────────────────────────────────────────────────────┤
│  Path Functions                                                 │
│  ├── getTaskOutputDir() → getProjectTempDir() + getSessionId() │
│  └── getTaskOutputPath() → join(getTaskOutputDir(), taskId)    │
├─────────────────────────────────────────────────────────────────┤
│  I/O Functions                                                  │
│  ├── initTaskOutput() → open() with O_EXCL | O_NOFOLLOW        │
│  ├── initTaskOutputAsSymlink() → symlink()                     │
│  ├── getTaskOutputDelta() → readFileRange()                    │
│  ├── getTaskOutput() → tailFile()                              │
│  └── cleanupTaskOutput() → unlink()                            │
└─────────────────────────────────────────────────────────────────┘
```

### 平台兼容性处理

```typescript
// Windows vs Unix 标志差异
const flags = process.platform === 'win32'
  ? 'wx'        // Windows: 字符串标志
  : fsConstants.O_WRONLY | fsConstants.O_CREAT | fsConstants.O_EXCL | O_NOFOLLOW

// O_NOFOLLOW 在 Windows 上不可用
const O_NOFOLLOW = fsConstants.O_NOFOLLOW ?? 0
```

---

## 风险、边界与改进建议

### 已知风险

#### 1. 测试异步清理竞态 (已缓解)

**问题**: 测试 teardown 时，异步写入可能在 `rmSync` 后完成，导致 ENOENT 未处理异常。

**修复**: `_pendingOps` Set 跟踪所有 fire-and-forget Promise:
```typescript
const _pendingOps = new Set<Promise<unknown>>()

function track<T>(p: Promise<T>): Promise<T> {
  _pendingOps.add(p)
  void p.finally(() => _pendingOps.delete(p)).catch(() => {})
  return p
}

export async function _clearOutputsForTest(): Promise<void> {
  while (_pendingOps.size > 0) {
    await Promise.allSettled([..._pendingOps])
  }
}
```

#### 2. UTF-16 vs UTF-8 字节计数偏差

**问题**: `content.length` (UTF-16 code units) 可能低估 UTF-8 字节数最多 3 倍。

**现状**: 作为粗略防护是可接受的，因为 5GB 上限是保守值。

**改进建议**:
```typescript
// 使用 Buffer.byteLength 进行精确计数
const byteLength = Buffer.byteLength(content, 'utf8')
this.#bytesWritten += byteLength
```

#### 3. 瞬态文件系统错误

**问题**: EMFILE (文件描述符耗尽)、EPERM (Windows 待删除) 等瞬态错误。

**修复**: `#drain()` 中的重试逻辑:
```typescript
async #drain(): Promise<void> {
  try {
    await this.#drainAllChunks()
  } catch (e) {
    logError(e)
    if (this.#queue.length > 0) {
      try {
        await this.#drainAllChunks()  // 重试一次
      } catch (e2) {
        logError(e2)
      }
    }
  }
}
```

### 边界情况

| 场景 | 行为 |
|------|------|
| 文件已存在 (O_EXCL) | 创建失败，调用方处理 |
| 符号链接已存在 | `initTaskOutputAsSymlink` 删除后重试 |
| 超过 5GB 上限 | 追加截断标记，停止写入 |
| 写入期间进程退出 | `finally` 块确保文件句柄关闭 |
| 关闭期间新写入 | 重新打开文件继续写入 |
| ENOENT 读取 | 返回空内容，不抛出 |

### 改进建议

#### 1. 写入压缩

对于大输出，考虑透明压缩:
```typescript
import { createGzip } from 'zlib'

class CompressedDiskTaskOutput {
  #compressor = createGzip()
  // 超过阈值后启用压缩
}
```

#### 2. 分片存储

5GB 单文件在部分文件系统上可能有问题:
```typescript
// 当文件超过 1GB 时创建新分片
getTaskOutputPath(taskId, shardIndex: number = 0)
```

#### 3. 写入速率限制

防止过快写入导致 I/O 饱和:
```typescript
import { setTimeout } from 'timers/promises'

async #drainWithRateLimit(): Promise<void> {
  const startTime = Date.now()
  await this.#writeAllChunks()
  const elapsed = Date.now() - startTime
  const minInterval = 100  // ms
  if (elapsed < minInterval) {
    await setTimeout(minInterval - elapsed)
  }
}
```

#### 4. 更精确的内存估计

```typescript
// 当前 (低估 UTF-8)
this.#bytesWritten += content.length

// 改进 (精确)
this.#bytesWritten += Buffer.byteLength(content, 'utf8')
```

#### 5. 异步预分配

大文件预分配减少碎片:
```typescript
if (this.#bytesWritten > PREALLOCATE_THRESHOLD) {
  await this.#fileHandle.truncate(PREALLOCATE_SIZE)
}
```

### 测试覆盖建议

当前未发现专门的 `diskOutput.test.ts`，建议添加:

1. **单元测试**:
   - `DiskTaskOutput` 队列行为
   - 5GB 上限触发
   - 重试逻辑
   - 内存释放验证

2. **集成测试**:
   - 并发写入同一文件
   - 跨平台路径处理
   - 符号链接降级

3. **性能测试**:
   - 高频率小写入
   - 大文件 (>1GB) 写入
   - 队列积压恢复
