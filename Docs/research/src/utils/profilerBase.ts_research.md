# profilerBase.ts 研究文档

## 场景与职责

`profilerBase.ts` 是 Claude Code 中负责**性能分析基础设施**的共享模块。它为多个性能分析器（startupProfiler、queryProfiler、headlessProfiler）提供统一的时间线格式和性能 API 访问。

**核心职责：**
1. **性能 API 访问**：懒加载 `perf_hooks` 模块
2. **时间格式化**：统一的时间格式化（毫秒，3位小数）
3. **时间线行格式化**：统一的时间线报告格式
4. **内存信息格式化**：RSS 和堆内存使用格式化

**业务场景：**
- 启动性能分析（startupProfiler）
- 查询性能分析（queryProfiler）
- 无头模式性能分析（headlessProfiler）
- 性能报告生成和显示

---

## 功能点目的

### 1. 性能 API 访问 (`getPerformance`)

懒加载 `perf_hooks` 模块的 `performance` API：

```typescript
export function getPerformance(): typeof PerformanceType
```

**设计考虑：**
- 仅在需要时才加载 `perf_hooks` 模块
- `perf_hooks.performance` 是进程范围的单例
- 所有性能分析器共享同一个时间线

### 2. 时间格式化 (`formatMs`)

将毫秒数格式化为固定 3 位小数的字符串：

```typescript
export function formatMs(ms: number): string
```

**示例：**
- `formatMs(1.5)` → `"1.500"`
- `formatMs(123.4567)` → `"123.457"`
- `formatMs(0)` → `"0.000"`

### 3. 时间线行格式化 (`formatTimelineLine`)

格式化单条时间线记录：

```typescript
export function formatTimelineLine(
  totalMs: number,
  deltaMs: number,
  name: string,
  memory: NodeJS.MemoryUsage | undefined,
  totalPad: number,
  deltaPad: number,
  extra?: string
): string
```

**输出格式：**
```
[+  123.456ms] (+   45.678ms) operation_name [extra_info] | RSS: 12.3MB, Heap: 4.5MB
```

**参数说明：**
| 参数 | 说明 |
|------|------|
| `totalMs` | 从启动开始的总时间 |
| `deltaMs` | 距离上一个操作的时间差 |
| `name` | 操作名称 |
| `memory` | 内存使用情况（可选） |
| `totalPad` | 总时间列的填充宽度 |
| `deltaPad` | 增量时间列的填充宽度 |
| `extra` | 额外信息（可选） |

---

## 具体技术实现

### 数据结构

```typescript
// 使用 Node.js 内置类型
import type { performance as PerformanceType } from 'perf_hooks'
import type { MemoryUsage } from 'process'
```

### 关键流程

**性能 API 获取流程：**
1. 检查 `performance` 变量是否已初始化
2. 如果未初始化，使用 `require('perf_hooks').performance`
3. 缓存并返回

**时间线行格式化流程：**
1. 格式化总时间（`totalMs`），使用 `totalPad` 填充
2. 格式化增量时间（`deltaMs`），使用 `deltaPad` 填充
3. 格式化内存信息（如果提供）
4. 拼接成完整字符串

### 关键代码路径

| 函数 | 路径 | 说明 |
|------|------|------|
| `getPerformance` | L14-20 | 性能 API 获取 |
| `formatMs` | L22-24 | 时间格式化 |
| `formatTimelineLine` | L33-46 | 时间线行格式化 |

### 代码实现细节

**性能 API 懒加载：**
```typescript
let performance: typeof PerformanceType | null = null

export function getPerformance(): typeof PerformanceType {
  if (!performance) {
    // eslint-disable-next-line @typescript-eslint/no-require-imports
    performance = require('perf_hooks').performance
  }
  return performance!
}
```

**时间格式化：**
```typescript
export function formatMs(ms: number): string {
  return ms.toFixed(3)
}
```

**时间线行格式化：**
```typescript
export function formatTimelineLine(
  totalMs: number,
  deltaMs: number,
  name: string,
  memory: NodeJS.MemoryUsage | undefined,
  totalPad: number,
  deltaPad: number,
  extra = '',
): string {
  const memInfo = memory
    ? ` | RSS: ${formatFileSize(memory.rss)}, Heap: ${formatFileSize(memory.heapUsed)}`
    : ''
  return `[+${formatMs(totalMs).padStart(totalPad)}ms] (+${formatMs(deltaMs).padStart(deltaPad)}ms) ${name}${extra}${memInfo}`
}
```

**输出示例：**
```
[+ 1234.567ms] (+  45.678ms) initialize_modules | RSS: 45.2MB, Heap: 12.3MB
[+ 1280.245ms] (+  45.678ms) load_config [cached] | RSS: 45.5MB, Heap: 12.5MB
```

---

## 依赖与外部交互

### 导入依赖

```typescript
import type { performance as PerformanceType } from 'perf_hooks'
import { formatFileSize } from './format.js'
```

### 外部调用方

| 调用方文件 | 用途 |
|-----------|------|
| `src/utils/headlessProfiler.ts` | 无头模式性能分析 |
| `src/utils/queryProfiler.ts` | 查询性能分析 |
| `src/utils/startupProfiler.ts` | 启动性能分析 |

### 依赖模块

| 模块 | 用途 |
|------|------|
| `format.js` | `formatFileSize` 用于内存大小格式化 |

### 使用模式

```typescript
// 在性能分析器中使用
import { getPerformance, formatTimelineLine, formatMs } from './profilerBase.js'

const performance = getPerformance()
const start = performance.now()

// ... 执行操作 ...

const end = performance.now()
const totalMs = end - startTime
const deltaMs = end - lastTime

console.log(formatTimelineLine(totalMs, deltaMs, 'operation_name', process.memoryUsage(), 8, 7))
```

---

## 风险、边界与改进建议

### 潜在风险

1. **精度丢失**
   - `toFixed(3)` 四舍五入到 3 位小数
   - 对于微秒级测量可能不够精确
   - 某些场景可能需要更高精度

2. **内存使用**
   - `process.memoryUsage()` 调用有一定开销
   - 频繁调用可能影响性能测量本身

3. **时序问题**
   - `performance.now()` 返回的是相对时间
   - 不同分析器之间的时间基准需要统一

4. **模块加载顺序**
   - 懒加载可能导致第一次调用有轻微延迟
   - 对于启动分析，应在最开始就初始化

### 边界情况

1. **负时间差**
   - 如果 `deltaMs` 为负，格式化后显示负值
   - 可能表示计时错误

2. **极大时间值**
   - `totalPad` 和 `deltaPad` 可能不足以容纳大数值
   - 需要调用方预估最大值

3. **内存信息缺失**
   - `memory` 为 `undefined` 时不显示内存信息
   - 这是正常行为

4. **空操作名称**
   - 空字符串作为 `name` 会导致格式异常

### 改进建议

1. **精度可配置**
   ```typescript
   let precision = 3
   
   export function setPrecision(p: number): void {
     precision = p
   }
   
   export function formatMs(ms: number): string {
     return ms.toFixed(precision)
   }
   ```

2. **自动填充宽度**
   ```typescript
   export function formatTimelineLineAuto(
     totalMs: number,
     deltaMs: number,
     name: string,
     memory?: NodeJS.MemoryUsage
   ): string {
     // 根据数值大小自动计算填充宽度
     const totalPad = Math.max(8, String(Math.floor(totalMs)).length + 4)
     const deltaPad = Math.max(7, String(Math.floor(deltaMs)).length + 4)
     return formatTimelineLine(totalMs, deltaMs, name, memory, totalPad, deltaPad)
   }
   ```

3. **类型安全增强**
   ```typescript
   export interface TimelineEntry {
     totalMs: number
     deltaMs: number
     name: string
     memory?: NodeJS.MemoryUsage
     extra?: string
   }
   
   export function formatTimelineEntry(entry: TimelineEntry, totalPad: number, deltaPad: number): string {
     return formatTimelineLine(
       entry.totalMs,
       entry.deltaMs,
       entry.name,
       entry.memory,
       totalPad,
       deltaPad,
       entry.extra
     )
   }
   ```

4. **性能测量包装器**
   ```typescript
   export function measure<T>(
     name: string,
     fn: () => T,
     onComplete: (entry: TimelineEntry) => void
   ): T {
     const perf = getPerformance()
     const start = perf.now()
     const memBefore = process.memoryUsage()
     
     const result = fn()
     
     const end = perf.now()
     const memAfter = process.memoryUsage()
     
     onComplete({
       totalMs: end - globalStartTime,
       deltaMs: end - start,
       name,
       memory: memAfter,
       extra: `heap_delta=${formatFileSize(memAfter.heapUsed - memBefore.heapUsed)}`
     })
     
     return result
   }
   ```

5. **测试覆盖**
   - 添加单元测试，验证格式化输出
   - 测试边界值（0、负数、极大值）
   - 测试内存信息格式化
