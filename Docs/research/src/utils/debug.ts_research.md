# debug.ts 深度研究文档

## 场景与职责

`debug.ts` 是 Claude Code 的 **调试日志系统核心模块**，提供了全面的调试日志记录能力。它支持多种日志级别、多种输出目标（文件、stderr），并针对 Ant（内部）用户和普通用户有不同的默认行为。

### 核心职责

1. **调试日志记录**：根据配置的级别记录调试信息
2. **多目标输出**：支持文件输出和 stderr 输出
3. **级别控制**：支持 verbose、debug、info、warn、error 五级日志
4. **运行时启用**：支持在会话中动态启用调试模式
5. **缓冲写入**：非调试模式下使用缓冲写入优化性能
6. **最新日志链接**：维护指向最新日志的符号链接

### 使用场景

- **开发调试**：开发者使用 `--debug` 或 `-d` 标志启动详细日志
- **问题诊断**：通过 `/debug` 命令在会话中启用调试
- **Ant 内部调试**：Ant 用户自动记录调试日志用于 bug 报告
- **性能分析**：记录慢操作和性能指标
- **错误追踪**：记录错误堆栈和上下文

---

## 功能点目的

### 1. 调试模式检测 (`isDebugMode`)

**目的**：判断是否处于调试模式。

**检测条件**：
- `runtimeDebugEnabled` 为 true（通过 `/debug` 或 `enableDebugLogging()` 设置）
- `DEBUG` 或 `DEBUG_SDK` 环境变量为 truthy
- 命令行参数包含 `--debug` 或 `-d`
- 命令行参数包含 `--debug-to-stderr` 或 `-d2e`
- 命令行参数包含 `--debug-file` 或 `--debug-file=path`

### 2. 日志级别控制 (`getMinDebugLogLevel`)

**目的**：确定最小日志级别，过滤低级别日志。

**级别顺序**：verbose < debug < info < warn < error

**配置方式**：`CLAUDE_CODE_DEBUG_LOG_LEVEL` 环境变量

### 3. 调试日志记录 (`logForDebugging`)

**目的**：记录调试日志到文件或 stderr。

**处理流程**：
1. 检查日志级别是否满足最小要求
2. 检查是否应该记录（`shouldLogDebugMessage`）
3. 处理多行消息（转换为 JSON 避免破坏 jsonl 格式）
4. 格式化时间戳和级别
5. 输出到 stderr 或文件

### 4. 缓冲写入机制

**目的**：在非调试模式下优化性能，减少 I/O 操作。

**实现细节**：
- 使用 `BufferedWriter` 进行批量写入
- 默认刷新间隔：1 秒
- 最大缓冲区大小：100 条
- 调试模式下使用即时模式（同步写入）

### 5. 最新日志链接 (`updateLatestDebugLogSymlink`)

**目的**：在 `~/.claude/debug/latest` 创建指向当前日志的符号链接。

**用途**：
- 方便快速找到最新日志文件
- 支持 `/share` 命令自动附加最新日志

### 6. Ant 专用错误日志 (`logAntError`)

**目的**：仅对 Ant 用户记录错误堆栈，用于内部调试。

---

## 具体技术实现

### 核心数据结构

```typescript
type DebugLogLevel = 'verbose' | 'debug' | 'info' | 'warn' | 'error'

const LEVEL_ORDER: Record<DebugLogLevel, number> = {
  verbose: 0,
  debug: 1,
  info: 2,
  warn: 3,
  error: 4,
}
```

### 调试日志路径

```typescript
function getDebugLogPath(): string {
  return (
    getDebugFilePath() ??                          // 命令行指定
    process.env.CLAUDE_CODE_DEBUG_LOGS_DIR ??      // 环境变量指定目录
    join(getClaudeConfigHomeDir(), 'debug', `${getSessionId()}.txt`)  // 默认
  )
}
// 默认: ~/.claude/debug/<session-id>.txt
```

### 缓冲写入器配置

```typescript
debugWriter = createBufferedWriter({
  writeFn: content => {
    if (isDebugMode()) {
      // 即时模式：同步写入，确保不丢失
      getFsImplementation().appendFileSync(path, content)
    } else {
      // 缓冲模式：异步写入，约 1 秒刷新一次
      pendingWrite = pendingWrite.then(appendAsync.bind(...))
    }
  },
  flushIntervalMs: 1000,
  maxBufferSize: 100,
  immediateMode: isDebugMode(),
})
```

### 日志格式

```
2024-01-15T10:30:00.000Z [DEBUG] Message content
2024-01-15T10:30:01.000Z [INFO] Some information
2024-01-15T10:30:02.000Z [ERROR] Error message
```

### 过滤机制

```typescript
function shouldLogDebugMessage(message: string): boolean {
  // 测试环境不记录（除非输出到 stderr）
  if (process.env.NODE_ENV === 'test' && !isDebugToStdErr()) return false
  
  // 非 Ant 用户只在调试模式下记录
  if (process.env.USER_TYPE !== 'ant' && !isDebugMode()) return false
  
  // 检查 debug filter 模式
  const filter = getDebugFilter()
  return shouldShowDebugMessage(message, filter)
}
```

---

## 关键代码路径与文件引用

### 核心导出函数

| 函数 | 行号 | 用途 |
|------|------|------|
| `isDebugMode` | 44-57 | 检查是否处于调试模式 |
| `enableDebugLogging` | 64-69 | 动态启用调试模式 |
| `getMinDebugLogLevel` | 34-40 | 获取最小日志级别 |
| `logForDebugging` | 203-228 | 记录调试日志 |
| `flushDebugLogs` | 198-201 | 刷新缓冲的日志 |
| `getDebugLogPath` | 230-236 | 获取日志文件路径 |
| `logAntError` | 258-268 | 记录 Ant 专用错误 |
| `setHasFormattedOutput` / `getHasFormattedOutput` | 127-133 | 格式化输出标记 |

### 导出类型

| 类型 | 行号 | 用途 |
|------|------|------|
| `DebugLogLevel` | 18 | 日志级别类型 |

### 依赖文件

```
debug.ts
├── 被调用方（上游）
│   ├── src/ink/root.ts                      # Ink 渲染
│   ├── src/cli/print.ts                     # CLI 输出
│   ├── src/utils/cronScheduler.ts           # 调度器调试
│   ├── src/utils/cronTasksLock.ts           # 锁调试
│   ├── src/utils/cronTasks.ts               # 任务调试
│   └── ...（大量其他模块）
├── 被依赖模块（下游）
│   ├── src/utils/bufferedWriter.js          # BufferedWriter
│   ├── src/utils/cleanupRegistry.js         # registerCleanup
│   ├── src/utils/debugFilter.js             # 调试过滤
│   ├── src/utils/envUtils.js                # getClaudeConfigHomeDir, isEnvTruthy
│   ├── src/utils/fsOperations.js            # getFsImplementation
│   ├── src/utils/process.js                 # writeToStderr
│   └── src/utils/slowOperations.js          # jsonStringify
└── Node.js 内置
    ├── fs/promises                          # 文件操作
    └── path                                 # 路径处理
```

---

## 依赖与外部交互

### 运行时依赖

| 模块 | 用途 |
|------|------|
| `fs/promises` | 文件追加、目录创建、符号链接 |
| `path` | 路径拼接 |
| `lodash-es/memoize.js` | 缓存配置值 |

### 内部模块依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `src/utils/bufferedWriter.js` | `BufferedWriter`, `createBufferedWriter` | 缓冲写入 |
| `src/utils/cleanupRegistry.js` | `registerCleanup` | 退出时刷新日志 |
| `src/utils/debugFilter.js` | `DebugFilter`, `parseDebugFilter`, `shouldShowDebugMessage` | 日志过滤 |
| `src/utils/envUtils.js` | `getClaudeConfigHomeDir`, `isEnvTruthy` | 环境工具 |
| `src/utils/fsOperations.js` | `getFsImplementation` | FS 抽象 |
| `src/utils/process.js` | `writeToStderr` | stderr 输出 |
| `src/utils/slowOperations.js` | `jsonStringify` | JSON 序列化 |

### 动态导入

```typescript
// 在 appendAsync 中使用
await mkdir(dir, { recursive: true })
await appendFile(path, content)
```

---

## 风险、边界与改进建议

### 已知风险

1. **同步写入阻塞**
   - 调试模式下的同步写入可能阻塞事件循环
   - 大量日志时可能影响性能

2. **日志文件增长**
   - 长时间运行的会话可能产生巨大的日志文件
   - 没有自动轮转机制

3. **缓冲丢失**
   - 进程崩溃时，缓冲中的日志可能丢失
   - `immediateMode` 缓解但非完全解决

4. **隐私风险**
   - 调试日志可能包含敏感信息
   - Ant 用户自动记录，可能无意收集敏感数据

5. **符号链接失败**
   - Windows 上创建符号链接可能需要特殊权限
   - 失败时静默忽略，用户可能不知道

### 边界情况

| 场景 | 处理 |
|------|------|
| 调试目录创建失败 | 静默忽略，下次重试 |
| 日志文件写入失败 | 依赖底层错误处理 |
| 多行消息 | 转换为 JSON 字符串 |
| 格式化输出模式 | 多行消息 JSON 化 |
| 符号链接已存在 | 先删除再创建 |
| 进程退出 | 清理注册器刷新缓冲 |

### 改进建议

1. **日志轮转**
   - 实现按大小或时间的日志轮转
   - 自动清理旧日志文件

2. **隐私保护**
   - 添加敏感信息自动脱敏
   - 提供日志审查工具

3. **性能优化**
   - 考虑使用 Worker 线程进行日志写入
   - 实现更高效的缓冲策略

4. **可观测性**
   - 添加日志写入指标
   - 实现日志查看命令

5. **跨平台支持**
   - 改进 Windows 符号链接处理
   - 处理不同文件系统的限制

6. **配置增强**
   - 支持按模块/类别的日志级别控制
   - 支持动态调整日志级别

### 相关 Issue/PR 参考

- `#22257`：涉及 Perfetto 跟踪和异步写入问题
- 本模块是调试和诊断基础设施的核心
