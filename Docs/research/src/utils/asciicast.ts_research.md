# asciicast.ts 研究文档

## 场景与职责

`asciicast.ts` 实现终端会话录制功能，生成符合 [asciicast v2 格式](https://github.com/asciinema/asciinema/blob/develop/doc/asciicast-v2.md) 的录制文件（`.cast`）。该功能主要用于内部测试和调试（`USER_TYPE=ant`）。

## 功能点目的

### 1. 录制文件路径管理 (`getRecordFilePath`)
- 根据环境变量决定是否启用录制
- 生成基于会话 ID 和时间戳的文件路径
- 支持 `--resume` 后的文件重命名

### 2. 录制器安装 (`installAsciicastRecorder`)
- 包装 `process.stdout.write` 捕获所有输出
- 使用 `BufferedWriter` 批量写入磁盘
- 处理终端尺寸变化事件

### 3. 录制控制
- `flushAsciicastRecorder` - 强制刷新缓冲区
- `renameRecordingForSession` - 会话恢复后重命名文件
- `getSessionRecordingPaths` - 获取当前会话的所有录制文件

## 具体技术实现

### 关键数据结构

```typescript
// 录制状态
const recordingState: { 
  filePath: string | null; 
  timestamp: number 
} = { filePath: null, timestamp: 0 }

// 录制器接口
type AsciicastRecorder = {
  flush(): Promise<void>
  dispose(): Promise<void>
}
```

### asciicast v2 格式

```typescript
// 头部（JSON 对象）
{
  version: 2,
  width: 80,      // 终端列数
  height: 24,     // 终端行数
  timestamp: 1234567890,  // Unix 时间戳
  env: {
    SHELL: '/bin/bash',
    TERM: 'xterm-256color'
  }
}

// 事件行（JSON 数组）
[time, type, data]
// time: 相对开始时间的秒数（浮点数）
// type: 'o' 输出, 'i' 输入, 'r' 尺寸变化
// data: 字符串内容
```

### 录制流程

```typescript
export function installAsciicastRecorder(): void {
  const filePath = getRecordFilePath()
  if (!filePath) return

  const { cols, rows } = getTerminalSize()
  const startTime = performance.now()

  // 写入头部
  const header = jsonStringify({
    version: 2,
    width: cols,
    height: rows,
    timestamp: Math.floor(Date.now() / 1000),
    env: { SHELL: process.env.SHELL || '', TERM: process.env.TERM || '' }
  })
  // ... 写入文件

  // 创建缓冲写入器
  const writer = createBufferedWriter({
    writeFn(content: string) {
      const currentPath = recordingState.filePath
      if (!currentPath) return
      pendingWrite = pendingWrite
        .then(() => appendFile(currentPath, content))
        .catch(() => {})
    },
    flushIntervalMs: 500,
    maxBufferSize: 50,
    maxBufferBytes: 10 * 1024 * 1024, // 10MB
  })

  // 包装 stdout.write
  const originalWrite = process.stdout.write.bind(process.stdout)
  process.stdout.write = function (chunk, encodingOrCb?, cb?) {
    const elapsed = (performance.now() - startTime) / 1000
    const text = typeof chunk === 'string' ? chunk : Buffer.from(chunk).toString('utf-8')
    writer.write(jsonStringify([elapsed, 'o', text]) + '\n')
    
    // 透传给原始 write
    if (typeof encodingOrCb === 'function') {
      return originalWrite(chunk, encodingOrCb)
    }
    return originalWrite(chunk, encodingOrCb, cb)
  }

  // 监听终端尺寸变化
  function onResize(): void {
    const elapsed = (performance.now() - startTime) / 1000
    const { cols: newCols, rows: newRows } = getTerminalSize()
    writer.write(jsonStringify([elapsed, 'r', `${newCols}x${newRows}`]) + '\n')
  }
  process.stdout.on('resize', onResize)

  // 注册清理
  registerCleanup(async () => {
    await recorder?.dispose()
    recorder = null
  })
}
```

### 会话恢复处理

```typescript
export async function renameRecordingForSession(): Promise<void> {
  const oldPath = recordingState.filePath
  if (!oldPath || recordingState.timestamp === 0) return

  const newPath = join(
    projectDir,
    `${getSessionId()}-${recordingState.timestamp}.cast`,
  )
  if (oldPath === newPath) return

  // 刷新缓冲区
  await recorder?.flush()
  
  // 重命名文件
  try {
    await rename(oldPath, newPath)
    recordingState.filePath = newPath
  } catch {
    // 静默失败
  }
}
```

## 关键代码路径与文件引用

### 本文件导出
- `getRecordFilePath(): string | null` - 获取录制文件路径
- `getSessionRecordingPaths(): string[]` - 获取会话录制文件列表
- `renameRecordingForSession(): Promise<void>` - 重命名录制文件
- `flushAsciicastRecorder(): Promise<void>` - 刷新录制缓冲区
- `installAsciicastRecorder(): void` - 安装录制器
- `_resetRecordingStateForTesting(): void` - 测试重置

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/bootstrap/state.ts` | 会话 ID、原始工作目录 |
| `src/utils/bufferedWriter.ts` | 缓冲写入器 |
| `src/utils/cleanupRegistry.ts` | 清理注册 |
| `src/utils/debug.ts` | 调试日志 |
| `src/utils/envUtils.ts` | 环境变量检查 |
| `src/utils/fsOperations.ts` | 文件系统操作 |
| `src/utils/path.ts` | 路径处理（`sanitizePath`） |
| `src/utils/slowOperations.ts` | JSON 序列化 |

### 调用方
- `src/main.tsx` - 主入口安装录制器
- `src/utils/sessionRestore.ts` - 会话恢复后重命名

## 依赖与外部交互

### 环境变量
| 变量 | 作用 |
|------|------|
| `USER_TYPE` | 用户类型，必须为 `'ant'` 才启用 |
| `CLAUDE_CODE_TERMINAL_RECORDING` | 设置为 `1` 启用录制 |

### 文件路径
```
~/.claude/projects/{sanitized_cwd}/{sessionId}-{timestamp}.cast
```

## 风险、边界与改进建议

### 已知限制
1. **仅内部使用**：通过 `USER_TYPE` 限制，外部用户无法使用
2. **stdout 独占**：只捕获 stdout，不捕获 stderr
3. **无输入录制**：当前只记录输出（`'o'` 类型），不记录输入
4. **单会话**：每个启动生成独立文件，`--continue` 产生多个文件

### 边界条件
1. **磁盘空间**：10MB 缓冲区限制，但总文件大小无限制
2. **并发写入**：`pendingWrite` 链确保顺序写入
3. **进程退出**：依赖 `registerCleanup` 确保数据刷新

### 安全风险
1. **敏感信息**：录制文件包含所有终端输出，可能包含敏感数据
2. **文件权限**：使用 `0o600` 权限创建文件
3. **路径遍历**：使用 `sanitizePath` 处理工作目录名

### 性能考虑
1. **同步开销**：每次 `stdout.write` 都有额外处理
2. **内存使用**：缓冲写入器限制内存使用
3. **磁盘 I/O**：批量写入减少 I/O 次数

### 改进建议
1. **输入录制**：添加输入事件（`'i'` 类型）记录
2. **压缩支持**：对大录制文件进行 gzip 压缩
3. **自动清理**：添加保留策略，自动删除旧录制
4. **选择性录制**：允许用户指定要录制的命令范围
5. **实时流式**：支持实时上传/流式传输录制内容

### 测试建议
1. 测试长会话的录制稳定性
2. 测试终端尺寸变化的记录
3. 测试会话恢复后的文件重命名
4. 测试磁盘满时的行为
