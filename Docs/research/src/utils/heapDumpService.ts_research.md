# heapDumpService.ts 深度研究

## 场景与职责

本模块提供堆内存转储和诊断服务，用于排查内存泄漏问题。通过 `/heapdump` 命令触发，可捕获 V8 堆快照和详细的内存诊断信息，帮助区分 V8 堆内存泄漏和原生内存泄漏。

**核心场景：**
1. **手动堆转储**：用户通过 `/heapdump` 命令主动触发
2. **自动内存监控**：内存超过 1.5GB 时自动触发
3. **内存泄漏诊断**：分析 detached contexts、active handles 等指标
4. **跨平台支持**：适配 Bun 和 Node.js 的不同 API

## 功能点目的

### 1. 堆快照捕获
- **目的**：捕获 V8 堆内存的完整状态
- **输出**：`.heapsnapshot` 文件（Chrome DevTools 可打开）
- **位置**：用户桌面（`~/Desktop`）

### 2. 内存诊断
- **目的**：提供堆外的内存使用上下文
- **指标**：
  - 进程内存使用（RSS、堆、外部、ArrayBuffers）
  - V8 堆统计（限制、malloc 内存、detached contexts）
  - 资源使用（峰值 RSS、CPU 时间）
  - 活动句柄和请求数
  - 打开的文件描述符（Linux）
  - smaps 汇总（Linux）

### 3. 泄漏检测启发式
- **目的**：自动识别潜在泄漏指标
- **规则**：
  - detached contexts > 0：可能的 iframe/context 泄漏
  - active handles > 100：可能的 timer/socket 泄漏
  - 原生内存 > 堆内存：可能的原生 addon 泄漏
  - 内存增长率 > 100MB/小时：高增长率警告
  - 文件描述符 > 500：可能的文件泄漏

### 4. 自动触发
- **目的**：在内存异常时自动捕获
- **阈值**：1.5GB 堆使用
- **序列**：支持多次转储（dump1, dump2...）

## 具体技术实现

### 核心数据结构
```typescript
export type HeapDumpResult = {
  success: boolean
  heapPath?: string      // 堆快照路径
  diagPath?: string      // 诊断文件路径
  error?: string         // 错误信息
}

export type MemoryDiagnostics = {
  timestamp: string
  sessionId: string
  trigger: 'manual' | 'auto-1.5GB'
  dumpNumber: number     // 本次会话的第 N 次转储
  uptimeSeconds: number
  memoryUsage: {
    heapUsed: number
    heapTotal: number
    external: number
    arrayBuffers: number
    rss: number
  }
  memoryGrowthRate: {
    bytesPerSecond: number
    mbPerHour: number
  }
  v8HeapStats: {
    heapSizeLimit: number
    mallocedMemory: number
    peakMallocedMemory: number
    detachedContexts: number  // 关键泄漏指标！
    nativeContexts: number
  }
  v8HeapSpaces?: Array<{
    name: string
    size: number
    used: number
    available: number
  }>
  resourceUsage: {
    maxRSS: number
    userCPUTime: number
    systemCPUTime: number
  }
  activeHandles: number
  activeRequests: number
  openFileDescriptors?: number  // Linux only
  analysis: {
    potentialLeaks: string[]
    recommendation: string
  }
  smapsRollup?: string  // Linux only
  platform: string
  nodeVersion: string
  ccVersion: string
}
```

### 关键流程

#### performHeapDump() - 主入口
```
1. 捕获内存诊断（先执行，避免堆快照分配影响）
2. 确保桌面目录存在
3. 生成文件名（sessionId + suffix）
4. 写入诊断 JSON（权限 0o600）
5. 写入堆快照
6. 记录遥测事件
7. 返回结果
```

**关键设计**：诊断在堆快照之前捕获，因为堆快照本身会分配大量内存。

#### captureMemoryDiagnostics() - 诊断收集
```typescript
export async function captureMemoryDiagnostics(
  trigger: 'manual' | 'auto-1.5GB',
  dumpNumber = 0,
): Promise<MemoryDiagnostics> {
  // 1. 基础指标
  const usage = process.memoryUsage()
  const heapStats = getHeapStatistics()
  const resourceUsage = process.resourceUsage()
  const uptimeSeconds = process.uptime()
  
  // 2. V8 堆空间统计（Bun 不支持）
  let heapSpaceStats: HeapSpaceInfo[] | undefined
  try {
    heapSpaceStats = getHeapSpaceStatistics()
  } catch {
    // Bun 不支持
  }
  
  // 3. 活动句柄/请求（Node.js 内部 API）
  const activeHandles = (process as any)._getActiveHandles().length
  const activeRequests = (process as any)._getActiveRequests().length
  
  // 4. Linux 特定指标
  let openFileDescriptors: number | undefined
  try {
    openFileDescriptors = (await readdir('/proc/self/fd')).length
  } catch {
    // 非 Linux
  }
  
  // 5. smaps 汇总（Linux）
  let smapsRollup: string | undefined
  try {
    smapsRollup = await readFile('/proc/self/smaps_rollup', 'utf8')
  } catch {
    // 非 Linux 或无权限
  }
  
  // 6. 计算增长率
  const bytesPerSecond = uptimeSeconds > 0 ? usage.rss / uptimeSeconds : 0
  const mbPerHour = (bytesPerSecond * 3600) / (1024 * 1024)
  
  // 7. 泄漏检测启发式
  const potentialLeaks: string[] = []
  if (heapStats.number_of_detached_contexts > 0) {
    potentialLeaks.push(`${heapStats.number_of_detached_contexts} detached context(s)...`)
  }
  if (activeHandles > 100) {
    potentialLeaks.push(`${activeHandles} active handles...`)
  }
  // ... 更多规则
  
  return { /* ... */ }
}
```

#### writeHeapSnapshot() - 堆快照写入
```typescript
async function writeHeapSnapshot(filepath: string): Promise<void> {
  if (typeof Bun !== 'undefined') {
    // Bun：同步写入避免大字符串跨线程克隆
    /* eslint-disable custom-rules/no-sync-fs */
    writeFileSync(filepath, Bun.generateHeapSnapshot('v8', 'arraybuffer'), { mode: 0o600 })
    /* eslint-enable custom-rules/no-sync-fs */
    Bun.gc(true)  // 强制 GC 尝试释放快照内存
    return
  }
  
  // Node.js：流式写入
  const writeStream = createWriteStream(filepath, { mode: 0o600 })
  const heapSnapshotStream = getHeapSnapshot()
  await pipeline(heapSnapshotStream, writeStream)
}
```

**平台差异处理：**
| 平台 | API | 写入方式 | 特殊处理 |
|------|-----|----------|----------|
| Bun | `Bun.generateHeapSnapshot` | 同步 | 强制 GC |
| Node.js | `v8.getHeapSnapshot` | 流式 | pipeline |

## 关键代码路径与文件引用

### 本文件导出
| 导出 | 类型 | 用途 |
|------|------|------|
| `performHeapDump` | 函数 | 执行堆转储 |
| `captureMemoryDiagnostics` | 函数 | 捕获诊断信息 |
| `HeapDumpResult` | 类型 | 转储结果类型 |
| `MemoryDiagnostics` | 类型 | 诊断数据类型 |

### 调用方
1. **commands/heapdump/heapdump.ts**: `/heapdump` 命令实现
2. **auto memory monitoring**: 内存超过 1.5GB 自动触发

### 依赖模块
```typescript
import { createWriteStream, writeFileSync } from 'fs'
import { readdir, readFile, writeFile } from 'fs/promises'
import { join } from 'path'
import { pipeline } from 'stream/promises'
import { getHeapSnapshot, getHeapSpaceStatistics, getHeapStatistics, type HeapSpaceInfo } from 'v8'
import { getSessionId } from '../bootstrap/state.js'
import { logEvent } from '../services/analytics/index.js'
import { logForDebugging } from './debug.js'
import { toError } from './errors.js'
import { getDesktopPath } from './file.js'
import { getFsImplementation } from './fsOperations.js'
import { logError } from './log.js'
import { jsonStringify } from './slowOperations.js'
```

## 依赖与外部交互

### 上游依赖

1. **v8 模块**: Node.js 内置
   - `getHeapSnapshot()`: 获取堆快照流
   - `getHeapStatistics()`: V8 堆统计
   - `getHeapSpaceStatistics()`: 堆空间细分（Bun 不支持）

2. **process 对象**: Node.js 全局
   - `memoryUsage()`: 进程内存使用
   - `resourceUsage()`: CPU 和资源使用
   - `_getActiveHandles()`: 活动句柄（内部 API）
   - `_getActiveRequests()`: 活动请求（内部 API）

3. **/proc 文件系统**: Linux 特定
   - `/proc/self/fd`: 文件描述符列表
   - `/proc/self/smaps_rollup`: 内存映射汇总

### 文件输出

**输出位置**：`~/Desktop/`

**文件名格式**：
- 堆快照：`{sessionId}.heapsnapshot` 或 `{sessionId}-dump{n}.heapsnapshot`
- 诊断：`{sessionId}-diagnostics.json` 或 `{sessionId}-dump{n}-diagnostics.json`

**文件权限**：`0o600`（仅所有者可读写）

## 风险、边界与改进建议

### 已知风险

1. **堆快照崩溃**
   - 风险：超大堆（>4GB）时 `getHeapSnapshot` 可能崩溃
   - 缓解：诊断数据先写入，即使快照失败也有信息

2. **同步写入阻塞**
   - 风险：Bun 的同步写入可能阻塞事件循环
   - 权衡：避免大字符串跨线程克隆的内存开销

3. **内部 API 依赖**
   - 风险：`_getActiveHandles()` 和 `_getActiveRequests()` 是内部 API
   - 缓解：try-catch 包装，失败时返回 undefined

4. **隐私泄露**
   - 风险：堆快照包含内存中的敏感数据
   - 缓解：文件权限 0o600，存储在用户桌面

### 边界情况

1. **Bun 不支持 `getHeapSpaceStatistics`**：优雅跳过
2. **非 Linux 平台**：/proc 指标返回 undefined
3. **权限不足**：/proc 文件读取失败静默处理
4. **桌面目录不存在**：自动创建
5. **磁盘空间不足**：写入失败返回错误结果

### 改进建议

1. **增量堆快照**
   - 建议：支持仅捕获变化的对象
   - 收益：减少快照大小和分析时间

2. **自动分析**
   - 建议：集成基本泄漏模式检测
   - 实现：解析堆快照中的可疑保留路径

3. **压缩输出**
   - 建议：支持 gzip 压缩堆快照
   - 收益：减少磁盘使用

4. **远程上传**
   - 建议：支持安全上传到分析服务
   - 场景：用户无法本地分析大快照

5. **历史趋势**
   - 建议：保存多次转储的摘要趋势
   - 收益：可视化内存增长曲线

6. **阈值配置**
   - 建议：允许用户配置自动触发阈值
   - 实现：环境变量或配置选项

### 测试要点

1. 跨平台兼容性（Bun/Node.js/Linux/macOS/Windows）
2. 大堆快照稳定性
3. 权限错误处理
4. 磁盘空间不足处理
5. 诊断数据完整性
6. 文件权限正确性
