# process.ts 研究文档

## 场景与职责

`process.ts` 是 Claude Code 中负责**进程 I/O 管理和错误处理**的模块。它提供标准输出/错误流的 EPIPE 错误处理、安全的流写入、进程退出辅助函数和 stdin 数据检测功能。

**核心职责：**
1. **EPIPE 错误处理**：防止管道断开时的内存泄漏
2. **安全流写入**：安全地写入 stdout/stderr
3. **错误退出**：统一的错误退出流程
4. **stdin 数据检测**：检测 stdin 是否有数据输入

**业务场景：**
- 管道命令（如 `claude -p | head -1`）时的优雅处理
- 防止输出错误导致的进程崩溃
- 非交互模式下的 stdin 检测

---

## 功能点目的

### 1. EPIPE 错误处理 (`registerProcessOutputErrorHandlers`)

注册 stdout 和 stderr 的 EPIPE 错误处理器：

```typescript
export function registerProcessOutputErrorHandlers(): void
```

**问题背景：**
- 当输出被管道到另一个命令（如 `head`），接收方提前关闭管道时
- Node.js 会抛出 EPIPE 错误
- 如果不处理，可能导致内存泄漏或进程异常

**解决方案：**
- 监听流的 `error` 事件
- 检测到 EPIPE 错误时销毁流
- 防止内存泄漏

### 2. 安全流写入 (`writeToStdout`, `writeToStderr`)

安全地向 stdout/stderr 写入数据：

```typescript
export function writeToStdout(data: string): void
export function writeToStderr(data: string): void
```

**安全特性：**
- 检查流是否已销毁
- 不处理背压（`write()` 返回 false 的情况）
- 注释中提到应考虑使用回调确保数据刷新

### 3. 错误退出 (`exitWithError`)

向 stderr 写入错误信息并退出进程：

```typescript
export function exitWithError(message: string): never
```

**特性：**
- 使用 `console.error` 输出错误
- 退出码 1
- 返回类型 `never`，表示不会返回

### 4. stdin 数据检测 (`peekForStdinData`)

检测 stdin 流是否有数据到达：

```typescript
export function peekForStdinData(
  stream: NodeJS.EventEmitter,
  ms: number
): Promise<boolean>
```

**用途：**
- `-p` 模式区分真正的管道输入和继承的空闲 stdin
- 第一个数据块到达时取消超时
- 返回 `true` 表示超时（无数据），`false` 表示流结束（有数据）

---

## 具体技术实现

### 数据结构

```typescript
// 使用 Node.js 内置类型
NodeJS.WriteStream
NodeJS.ErrnoException
NodeJS.EventEmitter
```

### 关键流程

**EPIPE 处理流程：**
1. 创建错误处理器函数
2. 检查错误码是否为 'EPIPE'
3. 如果是，销毁流
4. 将处理器注册到 stdout 和 stderr 的 `error` 事件

**安全写入流程：**
1. 检查流是否已销毁
2. 如果已销毁，直接返回
3. 调用 `stream.write(data)`
4. 不处理背压（注释中提到应改进）

**stdin 检测流程：**
1. 创建 Promise
2. 设置超时定时器
3. 监听 `end` 事件 → 返回 `false`
4. 监听 `data` 事件 → 清除超时
5. 超时触发 → 返回 `true`

### 关键代码路径

| 函数 | 路径 | 说明 |
|------|------|------|
| `handleEPIPE` | L1-9 | EPIPE 错误处理器工厂 |
| `registerProcessOutputErrorHandlers` | L12-15 | 注册错误处理器 |
| `writeOut` | L17-26 | 内部写入函数 |
| `writeToStdout` | L28-30 | 写入 stdout |
| `writeToStderr` | L32-34 | 写入 stderr |
| `exitWithError` | L37-43 | 错误退出 |
| `peekForStdinData` | L50-68 | stdin 数据检测 |

### 代码实现细节

**EPIPE 处理器：**
```typescript
function handleEPIPE(
  stream: NodeJS.WriteStream,
): (err: NodeJS.ErrnoException) => void {
  return (err: NodeJS.ErrnoException) => {
    if (err.code === 'EPIPE') {
      stream.destroy()
    }
  }
}

export function registerProcessOutputErrorHandlers(): void {
  process.stdout.on('error', handleEPIPE(process.stdout))
  process.stderr.on('error', handleEPIPE(process.stderr))
}
```

**安全写入：**
```typescript
function writeOut(stream: NodeJS.WriteStream, data: string): void {
  if (stream.destroyed) {
    return
  }
  // Note: we don't handle backpressure (write() returning false).
  // We should consider handling the callback to ensure we wait for data to flush.
  stream.write(data /* callback to handle here */)
}

export function writeToStdout(data: string): void {
  writeOut(process.stdout, data)
}

export function writeToStderr(data: string): void {
  writeOut(process.stderr, data)
}
```

**错误退出：**
```typescript
export function exitWithError(message: string): never {
  // biome-ignore lint/suspicious/noConsole:: intentional console output
  console.error(message)
  // eslint-disable-next-line custom-rules/no-process-exit
  process.exit(1)
}
```

**stdin 检测：**
```typescript
export function peekForStdinData(
  stream: NodeJS.EventEmitter,
  ms: number,
): Promise<boolean> {
  return new Promise<boolean>(resolve => {
    const done = (timedOut: boolean) => {
      clearTimeout(peek)
      stream.off('end', onEnd)
      stream.off('data', onFirstData)
      void resolve(timedOut)
    }
    const onEnd = () => done(false)
    const onFirstData = () => clearTimeout(peek)
    const peek = setTimeout(done, ms, true)
    stream.once('end', onEnd)
    stream.once('data', onFirstData)
  })
}
```

---

## 依赖与外部交互

### 导入依赖

```typescript
// 无外部导入，仅使用 Node.js 内置 API
```

### 外部调用方

由于 `process.ts` 是基础模块，可能在多个地方被引用，主要用于：
- 入口点错误处理
- 管道命令处理
- 非交互模式检测

### 依赖关系

- **无依赖**：纯 Node.js 内置 API
- **被依赖**：入口点、命令处理器等

---

## 风险、边界与改进建议

### 潜在风险

1. **背压未处理**
   - `write()` 返回 false 时不等待 drain 事件
   - 可能导致数据丢失
   - 注释中已提到需要改进

2. **EPIPE 处理器注册时机**
   - 如果在错误发生后才注册，无法捕获之前的错误
   - 应在进程启动早期注册

3. **stdin 检测竞态条件**
   - `peekForStdinData` 中 `once` 监听器可能在数据已到达后才注册
   - 如果数据在注册前已缓冲，可能检测不到

4. **流销毁副作用**
   - `stream.destroy()` 会触发 `close` 事件
   - 如果有其他监听器，可能产生意外行为

### 边界情况

1. **流已销毁**
   - `writeOut` 检查 `stream.destroyed`
   - 但检查和使用之间可能有竞态

2. **EPIPE 以外的错误**
   - 只处理 EPIPE，其他错误会抛出
   - 可能导致未捕获的异常

3. **stdin 已结束**
   - `peekForStdinData` 在流已结束时行为不确定

4. **超时为 0**
   - 立即超时，返回 `true`

5. **超大写入**
   - `write()` 可能分块写入
   - 不保证原子性

### 改进建议

1. **背压处理**
   ```typescript
   function writeOut(stream: NodeJS.WriteStream, data: string): Promise<void> {
     return new Promise((resolve, reject) => {
       if (stream.destroyed) {
         resolve()
         return
       }
       const canContinue = stream.write(data, err => {
         if (err) reject(err)
         else resolve()
       })
       if (!canContinue) {
         stream.once('drain', resolve)
       }
     })
   }
   ```

2. **错误分类处理**
   ```typescript
   function handleStreamError(stream: NodeJS.WriteStream) {
     return (err: NodeJS.ErrnoException) => {
       if (err.code === 'EPIPE') {
         stream.destroy()
       } else if (err.code === 'ECONNRESET') {
         // 连接重置，同样销毁
         stream.destroy()
       } else {
         // 其他错误，记录并重新抛出
         logError(err)
         throw err
       }
     }
   }
   ```

3. **stdin 检测改进**
   ```typescript
   export function peekForStdinData(
     stream: NodeJS.EventEmitter,
     ms: number
   ): Promise<boolean> {
     return new Promise(resolve => {
       // 检查是否已有数据
       if (stream.readableLength > 0) {
         resolve(false)
         return
       }
       // ... 现有逻辑 ...
     })
   }
   ```

4. **流状态封装**
   ```typescript
   class SafeWriteStream {
     private destroyed = false
     
     constructor(private stream: NodeJS.WriteStream) {
       stream.on('error', err => {
         if (err.code === 'EPIPE') {
           this.destroyed = true
           stream.destroy()
         }
       })
     }
     
     write(data: string): boolean {
       if (this.destroyed) return false
       return this.stream.write(data)
     }
   }
   ```

5. **测试覆盖**
   - 添加单元测试，模拟 EPIPE 错误
   - 测试背压场景
   - 测试 stdin 检测的各种情况
