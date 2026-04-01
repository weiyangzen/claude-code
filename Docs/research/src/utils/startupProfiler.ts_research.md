# startupProfiler.ts 深度研究

## 场景与职责

`startupProfiler.ts` 是一个**启动性能分析工具**，用于测量和报告初始化各阶段的时间消耗。支持两种模式：采样日志（Statsig）和详细分析（环境变量）。

**核心职责：**
1. 记录启动各阶段的检查点
2. 计算阶段持续时间
3. 采样记录到 Statsig（100% 内部用户，0.5% 外部用户）
4. 生成详细报告（当 `CLAUDE_CODE_PROFILE_STARTUP=1`）

**应用场景：**
- 启动性能监控
- 性能回归检测
- 开发调试

---

## 功能点目的

### 1. 模式配置
```typescript
const DETAILED_PROFILING = isEnvTruthy(process.env.CLAUDE_CODE_PROFILE_STARTUP)
const STATSIG_SAMPLE_RATE = 0.005
const STATSIG_LOGGING_SAMPLED =
  process.env.USER_TYPE === 'ant' || Math.random() < STATSIG_SAMPLE_RATE
const SHOULD_PROFILE = DETAILED_PROFILING || STATSIG_LOGGING_SAMPLED
```

### 2. 检查点记录
```typescript
export function profileCheckpoint(name: string): void
```

**功能：**
- 使用 `performance.mark()` 记录时间点
- 详细模式下捕获内存快照

### 3. 报告生成
```typescript
export function profileReport(): void
```

**功能：**
1. 记录 Statsig 事件（如果采样）
2. 写入详细报告文件（如果启用）

### 4. 阶段定义
```typescript
const PHASE_DEFINITIONS = {
  import_time: ['cli_entry', 'main_tsx_imports_loaded'],
  init_time: ['init_function_start', 'init_function_end'],
  settings_time: ['eagerLoadSettings_start', 'eagerLoadSettings_end'],
  total_time: ['cli_entry', 'main_after_run'],
}
```

---

## 具体技术实现

### 内存快照管理
```typescript
const memorySnapshots: NodeJS.MemoryUsage[] = []

export function profileCheckpoint(name: string): void {
  if (!SHOULD_PROFILE) return
  const perf = getPerformance()
  perf.mark(name)
  if (DETAILED_PROFILING) {
    memorySnapshots.push(process.memoryUsage())
  }
}
```

**设计要点：**
- 使用数组存储，索引与 mark 顺序对应
- 不使用 Map（检查点可能重复）

### 报告格式化
```typescript
function getReport(): string {
  const perf = getPerformance()
  const marks = perf.getEntriesByType('mark')
  // 格式化时间线...
  for (const [i, mark] of marks.entries()) {
    lines.push(
      formatTimelineLine(
        mark.startTime,
        mark.startTime - prevTime,
        mark.name,
        memorySnapshots[i],
        8,
        7,
      ),
    )
  }
}
```

### Statsig 日志
```typescript
export function logStartupPerf(): void {
  if (!STATSIG_LOGGING_SAMPLED) return
  
  const checkpointTimes = new Map<string, number>()
  for (const mark of marks) {
    checkpointTimes.set(mark.name, mark.startTime)
  }
  
  const metadata: Record<string, number | undefined> = {}
  for (const [phaseName, [start, end]] of Object.entries(PHASE_DEFINITIONS)) {
    const startTime = checkpointTimes.get(start)
    const endTime = checkpointTimes.get(end)
    if (startTime !== undefined && endTime !== undefined) {
      metadata[`${phaseName}_ms`] = Math.round(endTime - startTime)
    }
  }
  
  logEvent('tengu_startup_perf', metadata)
}
```

---

## 关键代码路径与文件引用

### 核心导出
| 导出 | 用途 |
|------|------|
| `profileCheckpoint` | 记录检查点 |
| `profileReport` | 生成报告 |
| `logStartupPerf` | 记录 Statsig 事件 |
| `isDetailedProfilingEnabled` | 检查详细模式 |
| `getStartupPerfLogPath` | 获取日志路径 |

### 依赖模块
| 模块 | 用途 |
|------|------|
| `path` | 路径操作 |
| `../bootstrap/state.js` | `getSessionId` |
| `../services/analytics/index.js` | `logEvent` |
| `./debug.js` | `logForDebugging` |
| `./envUtils.js` | `getClaudeConfigHomeDir`, `isEnvTruthy` |
| `./fsOperations.js` | `getFsImplementation` |
| `./profilerBase.js` | `formatMs`, `formatTimelineLine`, `getPerformance` |
| `./slowOperations.js` | `writeFileSync_DEPRECATED` |

### 调用方
| 文件 | 用途 |
|------|------|
| `src/setup.ts` | 设置检查点 |
| `src/services/analytics/firstPartyEventLogger.ts` | 事件日志 |
| `src/entrypoints/init.ts` | 初始化 |
| `src/entrypoints/cli.tsx` | CLI 入口 |
| `src/main.tsx` | 主入口 |
| `src/utils/gracefulShutdown.ts` | 关闭时报告 |
| `src/utils/headlessProfiler.ts` | 无头分析 |
| `src/utils/secureStorage/keychainPrefetch.ts` | 钥匙串预取 |
| `src/utils/secureStorage/macOsKeychainHelpers.ts` | 钥匙串帮助 |
| `src/utils/settings/mdm/settings.ts` | MDM 设置 |
| `src/utils/settings/settings.ts` | 设置 |
| `src/utils/telemetry/instrumentation.ts` | 遥测 |

---

## 依赖与外部交互

### 外部依赖
| 模块 | 用途 |
|------|------|
| `path` | 路径操作 |
| `perf_hooks`（延迟加载） | 性能测量 |

### 内部依赖
| 模块 | 用途 |
|------|------|
| `analytics/index.js` | 分析日志 |
| `profilerBase.js` | 共享分析基础设施 |
| `slowOperations.js` | 文件写入 |

### 环境变量
| 变量 | 用途 |
|------|------|
| `CLAUDE_CODE_PROFILE_STARTUP` | 启用详细分析 |
| `USER_TYPE` | 内部用户检测 |

---

## 风险、边界与改进建议

### 已知风险

1. **性能影响**
   - 即使采样模式也有 `performance.mark()` 开销
   - 内存快照增加 GC 压力

2. **检查点命名冲突**
   - 相同名称的检查点会覆盖
   - 需要命名规范

3. **采样偏差**
   - 随机采样可能错过特定场景
   - 会话级别采样无法 A/B 对比

### 边界情况

| 场景 | 处理 |
|------|------|
| 无检查点 | 报告 "No profiling checkpoints recorded" |
| 阶段检查点缺失 | 跳过该阶段，不记录 |
| 重复调用 profileReport | `reported` 标志防止重复 |
| 内存快照与检查点数量不匹配 | 依赖调用顺序一致性 |

### 改进建议

1. **动态启用**
   - 支持运行时启用/禁用
   - 信号触发详细报告

2. **更细粒度**
   - 支持嵌套检查点
   - 添加自定义元数据

3. **对比分析**
   - 保存基线报告
   - 自动检测回归

4. **可视化**
   - 生成火焰图
   - 集成 Chrome DevTools

5. **生产优化**
   - 减少采样率
   - 异步日志避免阻塞启动
