# Claude Code Diag Logger 研究文档

## 场景与职责

`logger.ts` 是 Claude Code 的 OpenTelemetry 诊断日志适配器，实现了 OpenTelemetry 的 `DiagLogger` 接口。该模块用于接收和记录 OTel 内部诊断信息，服务于以下场景：

1. **OTel 内部诊断**：捕获 OpenTelemetry SDK 内部的警告和错误
2. **调试支持**：在遥测系统故障时提供诊断信息
3. **错误追踪**：将 OTel 错误与 Claude Code 的错误日志系统集成

### 核心特点

- **极简实现**：仅实现必要的接口方法
- **错误分级**：区分 error/warn 和 info/debug/verbose
- **集成日志系统**：与 Claude Code 现有日志系统（`logError`, `logForDebugging`）集成
- **静默处理**：info/debug/verbose 级别完全静默，避免日志噪音

## 功能点目的

### 1. 错误捕获与记录
- **目的**：捕获 OTel SDK 内部错误，便于诊断遥测系统问题
- **实现**：`error()` 方法同时调用 `logError()` 和 `logForDebugging()`

### 2. 警告捕获与记录
- **目的**：捕获 OTel SDK 警告，了解潜在配置问题
- **实现**：`warn()` 方法同样调用 `logError()` 和 `logForDebugging()`

### 3. 日志级别过滤
- **目的**：避免 OTel 内部信息噪音影响正常日志
- **实现**：info/debug/verbose 方法为空实现（直接 return）

## 具体技术实现

### 类定义

```typescript
export class ClaudeCodeDiagLogger implements DiagLogger {
  error(message: string, ..._: unknown[]) {
    logError(new Error(message))
    logForDebugging(`[3P telemetry] OTEL diag error: ${message}`, {
      level: 'error',
    })
  }
  
  warn(message: string, ..._: unknown[]) {
    logError(new Error(message))
    logForDebugging(`[3P telemetry] OTEL diag warn: ${message}`, {
      level: 'warn',
    })
  }
  
  info(_message: string, ..._args: unknown[]) {
    return
  }
  
  debug(_message: string, ..._args: unknown[]) {
    return
  }
  
  verbose(_message: string, ..._args: unknown[]) {
    return
  }
}
```

### 方法详解

#### error(message, ...args)

```
功能：记录 OTel 错误
实现：
1. 创建 Error 对象并调用 logError() - 进入 Claude Code 错误日志系统
2. 调用 logForDebugging() 记录带前缀的调试信息
   - 前缀：[3P telemetry] OTEL diag error:
   - 级别：error
```

#### warn(message, ...args)

```
功能：记录 OTel 警告
实现：
1. 创建 Error 对象并调用 logError() - 同样进入错误日志系统
2. 调用 logForDebugging() 记录带前缀的调试信息
   - 前缀：[3P telemetry] OTEL diag warn:
   - 级别：warn
```

#### info(message, ...args)

```
功能：记录 OTel 信息（静默）
实现：直接 return，不记录任何内容
```

#### debug(message, ...args)

```
功能：记录 OTel 调试信息（静默）
实现：直接 return，不记录任何内容
```

#### verbose(message, ...args)

```
功能：记录 OTel 详细日志（静默）
实现：直接 return，不记录任何内容
```

## 关键代码路径与文件引用

### 导出类

- `ClaudeCodeDiagLogger` - 主导出类，实现 `DiagLogger` 接口

### 依赖文件

| 文件 | 用途 |
|-----|------|
| `src/utils/debug.js` | `logForDebugging()` 调试日志记录 |
| `src/utils/log.js` | `logError()` 错误日志记录 |

### 被调用方

- `src/utils/telemetry/instrumentation.ts` - 在 `initializeTelemetry()` 中设置为 OTel 诊断日志器：
  ```typescript
  diag.setLogger(new ClaudeCodeDiagLogger(), DiagLogLevel.ERROR)
  ```

### OpenTelemetry 依赖

- `@opentelemetry/api` - `DiagLogger` 接口定义

## 依赖与外部交互

### 初始化配置

在 `instrumentation.ts` 中初始化：

```typescript
import { DiagLogLevel, diag } from '@opentelemetry/api'
import { ClaudeCodeDiagLogger } from './logger.js'

// 设置诊断日志级别为 ERROR（只记录 error 和 warn）
diag.setLogger(new ClaudeCodeDiagLogger(), DiagLogLevel.ERROR)
```

### 日志级别映射

| OTel 级别 | 是否记录 | 处理方式 |
|----------|---------|---------|
| ERROR | ✅ | logError() + logForDebugging(level: 'error') |
| WARN | ✅ | logError() + logForDebugging(level: 'warn') |
| INFO | ❌ | 静默（空实现） |
| DEBUG | ❌ | 静默（空实现） |
| VERBOSE | ❌ | 静默（空实现） |

### 日志输出目标

1. **logError()**：
   - 进入 Claude Code 错误日志系统
   - 可能输出到 stderr、日志文件、错误追踪服务

2. **logForDebugging()**：
   - 进入调试日志系统
   - 受 `CLAUDE_CODE_DEBUG` 等环境变量控制
   - 格式：`[3P telemetry] OTEL diag {level}: {message}`

## 风险、边界与改进建议

### 风险

1. **日志重复**
   - error 和 warn 同时调用 logError() 和 logForDebugging()
   - 如果两个系统都输出到同一目标，可能产生重复日志

2. **错误对象创建开销**
   - 每次 error/warn 都创建新的 Error 对象
   - 高频调用时可能影响性能

3. **上下文丢失**
   - 未使用 `...args` 参数，可能丢失重要上下文信息
   - Error 对象没有堆栈跟踪（只有消息）

4. **日志级别不匹配**
   - warn 也调用 logError()，可能导致警告被当作错误处理
   - 某些监控系统可能因此产生误报

### 边界情况

1. **空消息**
   - 如果 OTel 传递空字符串，仍会创建 Error 对象

2. **超长消息**
   - 未对消息长度做限制，可能产生大日志

3. **特殊字符**
   - 消息中的特殊字符可能影响日志解析

4. **并发调用**
   - 多个 OTel 组件同时调用时，日志顺序可能交错

### 改进建议

1. **日志级别区分**
   - warn 方法应调用 logWarn() 而非 logError()
   - 添加 logWarn 到日志系统（如尚不存在）

2. **上下文保留**
   - 使用 `...args` 参数记录额外上下文
   - 示例：
     ```typescript
     warn(message: string, ...args: unknown[]) {
       logForDebugging(`[3P telemetry] OTEL diag warn: ${message}`, {
         level: 'warn',
         args: args.length > 0 ? args : undefined
       })
     }
     ```

3. **性能优化**
   - 添加日志采样（高频率时）
   - 延迟创建 Error 对象（仅在需要时）

4. **消息处理**
   - 添加消息截断（限制最大长度）
   - 清理/转义特殊字符
   - 添加时间戳（如底层日志系统未提供）

5. **可配置性**
   - 支持通过环境变量调整日志级别
   - 支持自定义日志前缀
   - 支持启用 info/debug 级别（开发调试时）

6. **错误追踪增强**
   - 为 Error 对象添加上下文堆栈
   - 关联到当前 span（如有）
   - 添加会话 ID 等标识符

7. **测试覆盖**
   - 添加单元测试验证各日志级别行为
   - 测试边界情况（空消息、超长消息、特殊字符）
   - 测试并发场景

### 代码示例改进

```typescript
import type { DiagLogger } from '@opentelemetry/api'
import { logForDebugging } from '../debug.js'
import { logError, logWarn } from '../log.js'

// 可配置的最大消息长度
const MAX_MESSAGE_LENGTH = 1000

// 截断消息
function truncateMessage(message: string): string {
  if (message.length <= MAX_MESSAGE_LENGTH) return message
  return message.slice(0, MAX_MESSAGE_LENGTH) + '... [truncated]'
}

export class ClaudeCodeDiagLogger implements DiagLogger {
  private formatMessage(level: string, message: string): string {
    return `[3P telemetry] OTEL diag ${level}: ${truncateMessage(message)}`
  }

  error(message: string, ...args: unknown[]) {
    const formatted = this.formatMessage('error', message)
    // 创建带上下文的错误
    const error = new Error(formatted)
    if (args.length > 0) {
      // 将额外参数附加到错误对象
      (error as Error & { diagArgs?: unknown[] }).diagArgs = args
    }
    logError(error)
    logForDebugging(formatted, { level: 'error', args: args.length > 0 ? args : undefined })
  }

  warn(message: string, ...args: unknown[]) {
    const formatted = this.formatMessage('warn', message)
    logWarn(formatted) // 使用 warn 级别而非 error
    logForDebugging(formatted, { level: 'warn', args: args.length > 0 ? args : undefined })
  }

  info(message: string, ...args: unknown[]) {
    // 开发环境可启用
    if (process.env.CLAUDE_CODE_OTEL_DEBUG) {
      logForDebugging(this.formatMessage('info', message), { 
        level: 'info', 
        args: args.length > 0 ? args : undefined 
      })
    }
  }

  debug(message: string, ...args: unknown[]) {
    if (process.env.CLAUDE_CODE_OTEL_DEBUG) {
      logForDebugging(this.formatMessage('debug', message), { 
        level: 'debug', 
        args: args.length > 0 ? args : undefined 
      })
    }
  }

  verbose(message: string, ...args: unknown[]) {
    // 完全静默
  }
}
```
