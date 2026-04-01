# TaskOutput.ts 深度研究文档

## 场景与职责

`TaskOutput.ts` 是 Claude Code 中 shell 命令输出的**单一数据源 (Single Source of Truth)**。它负责管理所有 shell 命令（bash、hooks、PowerShell 等）的 stdout/stderr 输出捕获、缓冲、持久化和进度追踪。

### 核心使用场景

1. **Bash 命令执行** (`Shell.ts`): stdout/stderr 直接写入文件 fd，绕过 JavaScript 层，通过轮询文件尾部获取进度
2. **Hooks 执行** (`hooks.ts`): 数据通过 pipe 流经 `writeStdout()`/`writeStderr()`，在内存中缓冲，超限后溢出到磁盘
3. **PowerShell 命令** (`PowerShellTool.tsx`): 类似 Bash，但使用 PowerShell 提供者
4. **Agent/Workflow 任务**: 子代理和工作流的输出管理

### 架构定位

```
┌─────────────────────────────────────────────────────────────────┐
│                        TaskOutput                               │
│                    (单一数据源)                                  │
├─────────────────────────────────────────────────────────────────┤
│  File Mode (Bash)          │  Pipe Mode (Hooks)                 │
│  - stdout→file fd          │  - writeStdout/writeStderr         │
│  - 轮询文件尾部获取进度     │  - 内存缓冲 + 磁盘溢出              │
│  - stderr 与 stdout 交错    │  - CircularBuffer 进度追踪          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │   DiskTaskOutput │  (diskOutput.ts)
                    │   - 异步磁盘写入 │
                    │   - 5GB 上限限制 │
                    └─────────────────┘
```

---

## 功能点目的

### 1. 双模式输出管理

| 模式 | 适用场景 | 数据流 | 进度获取 |
|------|---------|--------|----------|
| **File Mode** | Bash 命令 | stdout/stderr → 文件 fd | 轮询文件尾部 |
| **Pipe Mode** | Hooks、自定义回调 | 通过 writeStdout/writeStderr 方法 | 实时 CircularBuffer |

### 2. 内存管理与溢出控制

- **默认内存上限**: 8MB (`DEFAULT_MAX_MEMORY = 8 * 1024 * 1024`)
- **溢出策略**: 当内存缓冲超过上限时，自动将数据写入磁盘 (`DiskTaskOutput`)
- **GC 优化**: 使用 `Buffer.allocUnsafe` 和 `splice(0)` 模式确保内存及时回收

### 3. 进度追踪与回调

```typescript
type ProgressCallback = (
  lastLines: string,      // 最后 5 行
  allLines: string,       // 最后 100 行
  totalLines: number,     // 总行数估算
  totalBytes: number,     // 总字节数
  isIncomplete: boolean,  // 是否被截断
) => void
```

### 4. 共享轮询机制

- **轮询间隔**: 1000ms (`POLL_INTERVAL_MS`)
- **尾部读取大小**: 4096 bytes (`PROGRESS_TAIL_BYTES`)
- **动态注册**: React 组件通过 `startPolling`/`stopPolling` 控制轮询生命周期

---

## 具体技术实现

### 关键数据结构

```typescript
class TaskOutput {
  readonly taskId: string           // 任务唯一标识
  readonly path: string             // 输出文件路径
  readonly stdoutToFile: boolean    // true=文件模式, false=管道模式
  
  // 私有状态
  #stdoutBuffer = ''                // stdout 内存缓冲
  #stderrBuffer = ''                // stderr 内存缓冲
  #disk: DiskTaskOutput | null      // 磁盘写入器
  #recentLines: CircularBuffer<string>(1000)  // 最近行缓存
  #totalLines = 0                   // 总行数统计
  #totalBytes = 0                   // 总字节数统计
  
  // 共享轮询状态 (static)
  static #registry: Map<string, TaskOutput>      // 所有需要轮询的实例
  static #activePolling: Map<string, TaskOutput> // 当前正在轮询的实例
  static #pollInterval: Timer | null             // 共享轮询定时器
}
```

### 核心流程

#### 1. 文件模式进度轮询 (`#tick`)

```typescript
static #tick(): void {
  for (const [, entry] of TaskOutput.#activePolling) {
    void tailFile(entry.path, PROGRESS_TAIL_BYTES).then(
      ({ content, bytesRead, bytesTotal }) => {
        // 1. 从尾部内容中统计换行符数量
        // 2. 提取最后 5 行和最后 100 行的起始位置
        // 3. 如果文件大于 PROGRESS_TAIL_BYTES，使用比例估算总行数
        // 4. 调用 onProgress 回调更新 UI
      }
    )
  }
}
```

**行数估算算法**:
- 当 `bytesRead >= bytesTotal`: 精确计数（整个文件在 4KB 内）
- 当 `bytesRead < bytesTotal`: `Math.max(current, round((bytesTotal/bytesRead) * lineCount))`
- 使用 `Math.max` 确保行数不会倒退（处理行长度变化的边界情况）

#### 2. 管道模式写入 (`#writeBuffered`)

```typescript
#writeBuffered(data: string, isStderr: boolean): void {
  this.#totalBytes += data.length
  this.#updateProgress(data)
  
  if (this.#disk) {
    // 已溢出到磁盘，直接追加
    this.#disk.append(isStderr ? `[stderr] ${data}` : data)
    return
  }
  
  const totalMem = this.#stdoutBuffer.length + this.#stderrBuffer.length + data.length
  if (totalMem > this.#maxMemory) {
    // 触发溢出到磁盘
    this.#spillToDisk(isStderr ? data : null, isStderr ? null : data)
    return
  }
  
  // 保留在内存
  if (isStderr) this.#stderrBuffer += data
  else this.#stdoutBuffer += data
}
```

#### 3. 进度更新 (`#updateProgress`)

```typescript
#updateProgress(data: string): void {
  const MAX_PROGRESS_BYTES = 4096
  const MAX_PROGRESS_LINES = 100
  
  // 反向遍历字符串，统计换行符并提取最近行
  let pos = data.length
  while (pos > 0) {
    const prev = data.lastIndexOf('\n', pos - 1)
    if (prev === -1) break
    lineCount++
    // 提取非空行到 CircularBuffer
    pos = prev
  }
}
```

#### 4. 溢出到磁盘 (`#spillToDisk`)

```typescript
#spillToDisk(stderrChunk: string | null, stdoutChunk: string | null): void {
  this.#disk = new DiskTaskOutput(this.taskId)
  
  // 刷新现有缓冲
  if (this.#stdoutBuffer) {
    this.#disk.append(this.#stdoutBuffer)
    this.#stdoutBuffer = ''
  }
  if (this.#stderrBuffer) {
    this.#disk.append(`[stderr] ${this.#stderrBuffer}`)
    this.#stderrBuffer = ''
  }
  
  // 写入触发溢出的块
  if (stdoutChunk) this.#disk.append(stdoutChunk)
  if (stderrChunk) this.#disk.append(`[stderr] ${stderrChunk}`)
}
```

### 文件读取与清理

#### 读取 stdout (`getStdout`)

```typescript
async getStdout(): Promise<string> {
  if (this.stdoutToFile) {
    // 文件模式：从磁盘读取
    return this.#readStdoutFromFile()
  }
  
  // 管道模式
  if (this.#disk) {
    // 已溢出：返回尾部 + 截断提示
    const recent = this.#recentLines.getRecent(5)
    const tail = safeJoinLines(recent, '\n')
    return tail + `\nOutput truncated (${sizeKB}KB total). Full output saved to: ${this.path}`
  }
  return this.#stdoutBuffer
}
```

#### 文件读取 (`#readStdoutFromFile`)

```typescript
async #readStdoutFromFile(): Promise<string> {
  const maxBytes = getMaxOutputLength()  // 默认 30KB，上限 150KB
  const result = await readFileRange(this.path, 0, maxBytes)
  
  if (!result) return ''
  
  this.#outputFileSize = bytesTotal
  this.#outputFileRedundant = bytesTotal <= bytesRead
  
  return content
}
```

---

## 关键代码路径与文件引用

### 核心调用链

```
Shell.ts:exec()
  └── new TaskOutput(taskId, onProgress, !usePipeMode)
      ├── File Mode:
      │   └── open(outputPath) → child_process.spawn(stdio: [pipe, fd, fd])
      │       └── ShellCommand.ts:wrapSpawn()
      │           └── TaskOutput.startPolling(taskId)  [React 挂载时]
      │               └── setInterval(#tick, 1000ms)
      │                   └── tailFile() → onProgress()
      └── Pipe Mode:
          └── StreamWrapper (ShellCommand.ts)
              └── taskOutput.writeStdout/writeStderr()
                  └── #writeBuffered()
                      ├── 内存缓冲 (未超限)
                      └── #spillToDisk() → DiskTaskOutput.append() (超限)
```

### 外部调用方

| 文件 | 用途 |
|------|------|
| `src/utils/Shell.ts` | Bash 命令执行，创建 TaskOutput 实例 |
| `src/utils/ShellCommand.ts` | 包装子进程，管理 StreamWrapper |
| `src/utils/hooks.ts` | Hooks 执行，管道模式写入 |
| `src/tools/BashTool/BashTool.tsx` | Bash 工具，处理输出结果 |
| `src/tools/PowerShellTool/PowerShellTool.tsx` | PowerShell 命令执行 |
| `src/tools/AgentTool/AgentTool.tsx` | 子代理任务输出 |
| `src/tasks/LocalShellTask/LocalShellTask.tsx` | 本地 shell 任务 |
| `src/components/tasks/ShellDetailDialog.tsx` | UI 显示任务输出详情 |

### 依赖文件

| 文件 | 用途 |
|------|------|
| `src/utils/task/diskOutput.ts` | `DiskTaskOutput` 类，异步磁盘写入 |
| `src/utils/CircularBuffer.ts` | 循环缓冲区，存储最近行 |
| `src/utils/fsOperations.ts` | `readFileRange`, `tailFile` |
| `src/utils/shell/outputLimits.ts` | `getMaxOutputLength()` |
| `src/utils/stringUtils.ts` | `safeJoinLines` |
| `src/utils/debug.ts` | `logForDebugging` |

---

## 依赖与外部交互

### 类图关系

```
┌─────────────────┐     uses      ┌──────────────────┐
│   TaskOutput    │──────────────▶│  DiskTaskOutput  │
│   (task/)       │               │  (diskOutput.ts) │
└────────┬────────┘               └──────────────────┘
         │
         │ uses    ┌─────────────────┐
         └────────▶│  CircularBuffer │
                   │ (CircularBuffer)│
                   └─────────────────┘
         │
         │ uses    ┌─────────────────┐
         └────────▶│   fsOperations  │
                   │ (readFileRange) │
                   └─────────────────┘
```

### 环境变量依赖

| 变量 | 用途 | 来源 |
|------|------|------|
| `BASH_MAX_OUTPUT_LENGTH` | Bash 输出长度限制 | `shell/outputLimits.ts` |
| `TASK_MAX_OUTPUT_LENGTH` | 任务输出长度限制 | `outputFormatting.ts` |

---

## 风险、边界与改进建议

### 已知风险

#### 1. 并发会话文件冲突 (已修复)

**问题**: 多个 Claude Code 会话在同一项目目录运行时，启动清理会删除其他会话正在使用的输出文件。

**修复**: `diskOutput.ts` 中的 `getTaskOutputDir()` 现在包含 `sessionId`:
```typescript
_taskOutputDir = join(getProjectTempDir(), getSessionId(), 'tasks')
```

#### 2. 内存膨胀风险 (已缓解)

**问题**: 大量并发任务可能导致内存占用过高。

**缓解措施**:
- 8MB 内存上限，超限自动溢出到磁盘
- CircularBuffer 限制为 1000 行
- 使用 `Buffer.allocUnsafe` 避免零填充开销

#### 3. 磁盘填充攻击 (已缓解)

**问题**: 后台任务可能无限制写入磁盘。

**缓解措施**:
- 5GB 全局输出上限 (`MAX_TASK_OUTPUT_BYTES`)
- `ShellCommand.ts` 中的大小看门狗 (`SIZE_WATCHDOG_INTERVAL_MS = 5_000`)
- 超过上限时发送 SIGKILL

### 边界情况

| 场景 | 行为 |
|------|------|
| 文件不存在 (ENOENT) | `getStdout()` 返回诊断消息而非空字符串 |
| 空文件 | `#tick()` 仍然调用 `onProgress('', '', totalLines, bytesTotal, false)` |
| 超长单行输出 | 行数估算可能不准确，但使用单调最大值防止倒退 |
| 快速连续写入 | `DiskTaskOutput` 使用队列批量写入，减少 I/O |
| /clear 命令 | `regenerateSessionId()` 可能导致路径不匹配，已通过缓存解决 |

### 改进建议

#### 1. 可配置内存上限

当前 8MB 是硬编码的，建议通过环境变量或配置暴露:
```typescript
const DEFAULT_MAX_MEMORY = parseInt(
  process.env.CLAUDE_CODE_TASK_OUTPUT_MEMORY_LIMIT ?? '8388608',
  10
)
```

#### 2. 压缩大输出

对于超过一定阈值的输出，考虑使用 gzip 压缩存储:
```typescript
if (content.length > COMPRESSION_THRESHOLD) {
  return zlib.gzipSync(content)
}
```

#### 3. 更智能的行数估算

当前使用简单比例估算，对于行长度变化大的场景不准确:
```typescript
// 建议：使用加权移动平均或保留样本分布
const estimateLines = (samples: LineSample[]) => {
  const avgLength = samples.reduce((a, s) => a + s.length, 0) / samples.length
  return Math.round(bytesTotal / avgLength)
}
```

#### 4. 流式输出接口

当前 `getStdout()` 返回完整字符串，对于超大输出不友好:
```typescript
async *getStdoutStream(): AsyncGenerator<string> {
  // 返回生成器，支持流式消费
}
```

#### 5. 监控与指标

添加 OpenTelemetry 指标:
```typescript
// 溢出次数
// 平均输出大小
// 轮询延迟
// 磁盘写入队列深度
```

### 测试覆盖建议

当前未发现专门的 `TaskOutput.test.ts`，建议添加:

1. **单元测试**:
   - 内存溢出边界条件
   - 行数估算准确性
   - 并发轮询行为

2. **集成测试**:
   - 与 `Shell.ts` 的端到端流程
   - 大文件 (>5GB) 处理
   - 跨会话隔离

3. **性能测试**:
   - 高频率写入场景
   - 大量并发任务
   - 内存使用模式
