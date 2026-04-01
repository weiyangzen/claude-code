# slowOperations.ts 深度研究

## 场景与职责

`slowOperations.ts` 是一个**性能监控包装器模块**，为常见的昂贵操作（JSON 序列化、克隆、文件写入）添加延迟检测和日志记录。

**核心职责：**
1. 包装昂贵操作并测量执行时间
2. 对超过阈值的操作记录调试日志
3. 区分内部（ant）和外部构建的不同行为
4. 提供统一的替代函数替代原生操作

**应用场景：**
- 检测性能回归
- 识别瓶颈操作
- 开发调试

---

## 功能点目的

### 1. 慢操作阈值配置
```typescript
const SLOW_OPERATION_THRESHOLD_MS = (() => {
  const envValue = process.env.CLAUDE_CODE_SLOW_OPERATION_THRESHOLD_MS
  if (envValue !== undefined) {
    const parsed = Number(envValue)
    if (!Number.isNaN(parsed) && parsed >= 0) {
      return parsed
    }
  }
  if (process.env.NODE_ENV === 'development') {
    return 20  // 开发环境 20ms
  }
  if (process.env.USER_TYPE === 'ant') {
    return 300  // 内部用户 300ms
  }
  return Infinity  // 外部用户默认关闭
})()
```

### 2. 慢操作日志记录器
```typescript
class AntSlowLogger {
  constructor(args: IArguments) {
    this.startTime = performance.now()
    this.args = args
    this.err = new Error()  // 捕获堆栈
  }

  [Symbol.dispose](): void {
    const duration = performance.now() - this.startTime
    if (duration > SLOW_OPERATION_THRESHOLD_MS && !isLogging) {
      // 记录慢操作...
    }
  }
}
```

**使用模式（using 声明）：**
```typescript
using _ = slowLogging`JSON.stringify(${value})`
return JSON.stringify(value)
```

### 3. 构建时特性分支
```typescript
export const slowLogging = feature('SLOW_OPERATION_LOGGING')
  ? slowLoggingAnt
  : slowLoggingExternal
```

- **Ant 构建**：创建 `AntSlowLogger`，实际计时和记录
- **外部构建**：返回 `NOOP_LOGGER`，零开销

### 4. 包装函数

#### JSON 操作
```typescript
export function jsonStringify(value: unknown, replacer?, space?): string
export const jsonParse: typeof JSON.parse
```

#### 克隆操作
```typescript
export function clone<T>(value: T, options?: StructuredSerializeOptions): T
export function cloneDeep<T>(value: T): T  // lodash
```

#### 文件写入（已弃用）
```typescript
export function writeFileSync_DEPRECATED(
  filePath: string,
  data: string | NodeJS.ArrayBufferView,
  options?: WriteFileOptionsWithFlush
): void
```

**特点：**
- 支持 `flush` 选项确保数据落盘
- 标记为 DEPRECATED，推荐使用异步 API

---

## 具体技术实现

### 描述构建
```typescript
function buildDescription(args: IArguments): string {
  const strings = args[0] as TemplateStringsArray
  let result = ''
  for (let i = 0; i < strings.length; i++) {
    result += strings[i]
    if (i + 1 < args.length) {
      const v = args[i + 1]
      if (Array.isArray(v)) {
        result += `Array[${(v as unknown[]).length}]`
      } else if (v !== null && typeof v === 'object') {
        result += `Object{${Object.keys(v as Record<string, unknown>).length} keys}`
      } else if (typeof v === 'string') {
        result += v.length > 80 ? `${v.slice(0, 80)}…` : v
      } else {
        result += String(v)
      }
    }
  }
  return result
}
```

### 调用帧提取
```typescript
export function callerFrame(stack: string | undefined): string {
  if (!stack) return ''
  for (const line of stack.split('\n')) {
    if (line.includes('slowOperations')) continue
    const m = line.match(/([^/\\]+?):(\d+):\d+\)?$/)
    if (m) return ` @ ${m[1]}:${m[2]}`
  }
  return ''
}
```

### JSON.parse 优化
```typescript
export const jsonParse: typeof JSON.parse = (text, reviver) => {
  using _ = slowLogging`JSON.parse(${text})`
  // V8 优化：避免传递 undefined 的 reviver
  return typeof reviver === 'undefined'
    ? JSON.parse(text)
    : JSON.parse(text, reviver)
}
```

---

## 关键代码路径与文件引用

### 核心导出
| 导出 | 用途 |
|------|------|
| `SLOW_OPERATION_THRESHOLD_MS` | 慢操作阈值 |
| `callerFrame` | 调用帧提取 |
| `slowLogging` | 日志记录标签函数 |
| `jsonStringify` | 包装后的 JSON.stringify |
| `jsonParse` | 包装后的 JSON.parse |
| `clone` | 包装后的 structuredClone |
| `cloneDeep` | 包装后的 lodash cloneDeep |
| `writeFileSync_DEPRECATED` | 包装后的 fs.writeFileSync |

### 依赖模块
| 模块 | 用途 |
|------|------|
| `bun:bundle` | 特性检测 |
| `fs` | 同步文件操作 |
| `lodash-es/cloneDeep.js` | 深度克隆 |
| `../bootstrap/state.js` | `addSlowOperation` |
| `./debug.js` | `logForDebugging` |

### 调用方（广泛使用的模块）
| 文件 | 用途 |
|------|------|
| `src/server/createDirectConnectSession.ts` | 服务器 |
| `src/cli/structuredIO.ts` | 结构化 IO |
| `src/main.tsx` | 主入口 |
| `src/bootstrap/state.js` | 状态管理 |
| `src/services/api/claude.ts` | API 调用 |
| `src/utils/settings/settings.ts` | 设置 |
| `src/utils/sessionStorage.ts` | 会话存储 |
| `src/utils/messages.ts` | 消息处理 |
| `src/utils/permissions/permissions.ts` | 权限 |

---

## 依赖与外部交互

### 外部依赖
| 模块 | 用途 |
|------|------|
| `bun:bundle` | 构建时特性 |
| `fs` | 文件操作 |
| `lodash-es/cloneDeep.js` | 深度克隆 |

### 内部依赖
| 模块 | 用途 |
|------|------|
| `../bootstrap/state.js` | 慢操作统计 |
| `./debug.js` | 调试日志 |

### 环境变量
| 变量 | 用途 |
|------|------|
| `CLAUDE_CODE_SLOW_OPERATION_THRESHOLD_MS` | 自定义阈值 |
| `NODE_ENV` | 开发环境检测 |
| `USER_TYPE` | 内部用户检测 |

---

## 风险、边界与改进建议

### 已知风险

1. **递归日志**
   - `logForDebugging` 可能触发文件写入，导致递归
   - 缓解：`isLogging` 守卫标志

2. **性能开销**
   - 即使外部构建也有函数调用开销
   - 缓解：使用 `using` 声明和 Disposable 模式

3. **堆栈捕获开销**
   - `new Error()` 在构造时捕获堆栈
   - 缓解：仅在慢操作确认后读取 `.stack`

### 边界情况

| 场景 | 处理 |
|------|------|
| 阈值 = 0 | 所有操作都被记录 |
| 阈值 = Infinity | 无操作被记录 |
| 嵌套慢操作 | `isLogging` 防止递归 |
| 无效 env 值 | 使用默认阈值 |

### 改进建议

1. **采样策略**
   - 添加概率采样避免高频操作淹没日志
   - 支持按操作类型设置不同阈值

2. **聚合报告**
   - 定期汇总慢操作统计
   - 提供热点分析

3. **可视化集成**
   - 与 DevBar 集成显示慢操作警告
   - 添加火焰图支持

4. **异步操作**
   - 扩展支持异步操作的延迟测量
   - 添加 Promise 包装器

5. **生产监控**
   - 将慢操作指标发送到遥测系统
   - 设置告警阈值
