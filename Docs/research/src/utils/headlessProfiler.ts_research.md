# headlessProfiler.ts 深度研究

## 场景与职责

本模块专为无头模式（`-p` / `--print` 模式）设计，用于测量每轮对话的延迟指标。它在非交互式会话中追踪关键性能阶段，帮助识别性能瓶颈和优化机会。

**核心场景：**
1. **打印模式性能分析**：测量 `-p` 模式下各阶段耗时
2. **延迟指标收集**：系统消息输出、查询开始、首 token 响应时间
3. **采样遥测**：向 Statsig 发送性能数据（ant 用户 100%，外部 5%）
4. **启动性能关联**：复用 `CLAUDE_CODE_PROFILE_STARTUP` 环境变量控制详细日志

## 功能点目的

### 1. 分阶段性能测量
- **目的**：精确测量对话各阶段耗时
- **测量点**：
  - `turn_start`: 轮次开始
  - `system_message_yielded`: 系统消息输出（仅第 0 轮）
  - `query_started`: 查询开始
  - `api_request_sent`: API 请求发送
  - `first_chunk`: 首个响应块到达

### 2. 采样数据收集
- **目的**：平衡数据收集与性能开销
- **策略**：
  - ant 用户：100% 采样
  - 外部用户：5% 随机采样
  - 详细模式：`CLAUDE_CODE_PROFILE_STARTUP=1` 强制启用

### 3. 遥测上报
- **目的**：将性能数据发送到分析系统
- **事件**：`tengu_headless_latency`
- **数据**：各阶段耗时、检查点数量、入口点信息

## 具体技术实现

### 核心数据结构
```typescript
// 详细分析模式
const DETAILED_PROFILING = isEnvTruthy(process.env.CLAUDE_CODE_PROFILE_STARTUP)

// Statsig 采样配置
const STATSIG_SAMPLE_RATE = 0.05
const STATSIG_LOGGING_SAMPLED = process.env.USER_TYPE === 'ant' || Math.random() < STATSIG_SAMPLE_RATE

// 是否启用分析
const SHOULD_PROFILE = DETAILED_PROFILING || STATSIG_LOGGING_SAMPLED

// 性能标记前缀（避免与其他 profiler 冲突）
const MARK_PREFIX = 'headless_'

// 当前轮次追踪
let currentTurnNumber = -1
```

### 关键流程

#### headlessProfilerStartTurn() - 开始新轮次
```typescript
export function headlessProfilerStartTurn(): void {
  // 1. 检查是否非交互式会话
  if (!getIsNonInteractiveSession()) return
  
  // 2. 检查是否启用分析
  if (!SHOULD_PROFILE) return
  
  // 3. 递增轮次号
  currentTurnNumber++
  
  // 4. 清除之前的标记
  clearHeadlessMarks()
  
  // 5. 记录轮次开始
  perf.mark(`${MARK_PREFIX}turn_start`)
}
```

#### headlessProfilerCheckpoint(name: string) - 记录检查点
```typescript
export function headlessProfilerCheckpoint(name: string): void {
  if (!getIsNonInteractiveSession()) return
  if (!SHOULD_PROFILE) return
  
  perf.mark(`${MARK_PREFIX}${name}`)
  
  // 详细模式：输出调试日志
  if (DETAILED_PROFILING) {
    logForDebugging(`[headlessProfiler] Checkpoint: ${name} at ${perf.now().toFixed(1)}ms`)
  }
}
```

#### logHeadlessProfilerTurn() - 上报轮次数据
```typescript
export function logHeadlessProfilerTurn(): void {
  // 1. 检查条件和获取标记
  
  // 2. 构建检查点查找表
  const checkpointTimes = new Map<string, number>()
  for (const mark of marks) {
    const name = mark.name.slice(MARK_PREFIX.length)
    checkpointTimes.set(name, mark.startTime)
  }
  
  // 3. 计算各阶段耗时
  const metadata: Record<string, number | string | undefined> = {
    turn_number: currentTurnNumber,
  }
  
  // 系统消息时间（仅第 0 轮）
  if (systemMessageTime !== undefined && currentTurnNumber === 0) {
    metadata.time_to_system_message_ms = Math.round(systemMessageTime)
  }
  
  // 查询开始时间
  if (queryStartTime !== undefined) {
    metadata.time_to_query_start_ms = Math.round(queryStartTime - turnStart)
  }
  
  // 首响应时间（TTFT）
  if (firstChunkTime !== undefined) {
    metadata.time_to_first_response_ms = Math.round(firstChunkTime - turnStart)
  }
  
  // 查询开销（准备到发送）
  if (queryStartTime !== undefined && apiRequestTime !== undefined) {
    metadata.query_overhead_ms = Math.round(apiRequestTime - queryStartTime)
  }
  
  // 4. 发送遥测（如采样命中）
  if (STATSIG_LOGGING_SAMPLED) {
    logEvent('tengu_headless_latency', metadata)
  }
  
  // 5. 详细模式输出
  if (DETAILED_PROFILING) {
    logForDebugging(`[headlessProfiler] Turn ${currentTurnNumber} metrics: ...`)
  }
}
```

### 性能标记管理
```typescript
function clearHeadlessMarks(): void {
  const perf = getPerformance()
  const allMarks = perf.getEntriesByType('mark')
  for (const mark of allMarks) {
    if (mark.name.startsWith(MARK_PREFIX)) {
      perf.clearMarks(mark.name)
    }
  }
}
```

**设计决策：**
- 使用 `performance.mark()` API（Node.js 标准）
- 前缀隔离避免与其他 profiler 冲突
- 每轮清除旧标记防止内存增长

## 关键代码路径与文件引用

### 本文件导出
| 导出 | 类型 | 用途 |
|------|------|------|
| `headlessProfilerStartTurn` | 函数 | 开始新轮次分析 |
| `headlessProfilerCheckpoint` | 函数 | 记录检查点 |
| `logHeadlessProfilerTurn` | 函数 | 上报轮次数据 |

### 调用方
1. **cli/print.ts**: 打印模式的主循环
2. **QueryEngine.ts**: 查询引擎（无头模式）
3. **query.ts**: 查询处理
4. **services/api/claude.ts**: API 调用

### 依赖模块
```typescript
import { getIsNonInteractiveSession } from '../bootstrap/state.js'
import { logEvent } from '../services/analytics/index.js'
import { logForDebugging } from './debug.js'
import { isEnvTruthy } from './envUtils.js'
import { getPerformance } from './profilerBase.js'
import { jsonStringify } from './slowOperations.js'
```

## 依赖与外部交互

### 上游依赖

1. **profilerBase.ts**: 性能基础设施
   - `getPerformance()`: 获取 `perf_hooks.performance` 实例
   - 延迟加载避免启动开销

2. **bootstrap/state.ts**: 会话状态
   - `getIsNonInteractiveSession()`: 检测是否为无头模式

3. **services/analytics/index.ts**: 遥测
   - `logEvent()`: 发送分析事件

### 配置选项

**环境变量：**
| 变量 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `CLAUDE_CODE_PROFILE_STARTUP` | boolean | `false` | 启用详细日志 |
| `USER_TYPE` | string | - | `ant` 强制采样 |

**采样逻辑：**
```
启用分析 = 详细模式 OR ant用户 OR (随机数 < 0.05)
```

## 风险、边界与改进建议

### 已知风险

1. **性能开销**
   - 风险：频繁的性能 API 调用可能影响测量精度
   - 缓解：非采样用户完全跳过（`SHOULD_PROFILE` 检查）

2. **内存泄漏**
   - 风险：性能标记未清除导致内存增长
   - 缓解：每轮 `clearHeadlessMarks()`

3. **时间精度**
   - 风险：`performance.now()` 精度受限（防 Spectre）
   - 缓解：相对测量仍有效，绝对精度非关键

4. **采样偏差**
   - 风险：5% 采样可能错过偶发性能问题
   - 缓解：ant 用户 100% 采样捕获详细数据

### 边界情况

1. **非无头模式**：所有函数立即返回
2. **第 0 轮特殊处理**：系统消息时间仅记录一次
3. **缺失检查点**：灵活处理部分数据（不强制所有检查点）
4. **长时间运行**：轮次号可能溢出（实际不关键）

### 改进建议

1. **更多检查点**
   - 建议：添加工具执行阶段标记
   - 场景：识别工具调用耗时

2. **直方图数据**
   - 建议：收集多轮统计（P50/P95/P99）
   - 收益：更全面的性能视图

3. **内存使用追踪**
   - 建议：添加 RSS/堆内存标记
   - 场景：检测内存泄漏

4. **自定义标记**
   - 建议：允许调用方添加自定义检查点
   - 实现：扩展 `headlessProfilerCheckpoint` 参数

5. **离线分析**
   - 建议：支持写入本地文件而非仅遥测
   - 场景：无网络环境的性能分析

6. **火焰图集成**
   - 建议：与 perfettoTracing.ts 集成
   - 收益：可视化性能分析

### 测试要点

1. 采样逻辑正确性
2. 标记清除完整性
3. 非无头模式无操作
4. 数据格式正确性
5. 详细模式日志输出
6. 长时间运行稳定性
