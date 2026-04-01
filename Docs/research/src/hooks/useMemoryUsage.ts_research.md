# useMemoryUsage.ts 深度研究文档

## 场景与职责

`useMemoryUsage` 是一个用于监控 Node.js 进程内存使用情况的 React 钩子。它定期轮询内存状态，并在内存使用超过阈值时返回警告信息，用于 UI 显示内存压力提示。

### 核心场景

1. **内存监控**：每 10 秒检查一次堆内存使用情况
2. **阈值告警**：根据内存使用水平返回不同状态
3. **UI 集成**：与通知系统集成显示内存警告

### 与其他组件的关系

- 被 `Notifications` 组件使用，决定是否显示内存警告
- 返回 null 时正常状态，不触发重渲染优化
- 独立运行，不依赖其他业务逻辑

---

## 功能点目的

### 1. 内存阈值分级

| 状态 | 阈值 | 堆内存使用 |
|-----|------|-----------|
| `normal` | < 1.5 GB | 正常 |
| `high` | 1.5 GB - 2.5 GB | 高 |
| `critical` | ≥ 2.5 GB | 严重 |

### 2. 性能优化

- **正常状态返回 null**：99% 以上的用户不会触发警告，返回 null 避免不必要的重渲染
- **状态不变时跳过更新**：使用函数式 setState，如果状态相同则保持原引用
- **10 秒轮询间隔**：平衡及时性和性能开销

### 3. 渐进式警告

- 只有状态变为 `high` 或 `critical` 时才返回具体信息
- UI 可以根据状态显示不同级别的警告

---

## 具体技术实现

### 关键数据结构

```typescript
export type MemoryUsageStatus = 'normal' | 'high' | 'critical'

export type MemoryUsageInfo = {
  heapUsed: number      // 堆内存使用量（字节）
  status: MemoryUsageStatus
}

// 阈值常量
const HIGH_MEMORY_THRESHOLD = 1.5 * 1024 * 1024 * 1024      // 1.5 GB
const CRITICAL_MEMORY_THRESHOLD = 2.5 * 1024 * 1024 * 1024  // 2.5 GB
```

### 核心实现

```typescript
export function useMemoryUsage(): MemoryUsageInfo | null {
  const [memoryUsage, setMemoryUsage] = useState<MemoryUsageInfo | null>(null)
  
  useInterval(() => {
    const heapUsed = process.memoryUsage().heapUsed
    
    // 确定状态
    const status: MemoryUsageStatus =
      heapUsed >= CRITICAL_MEMORY_THRESHOLD
        ? 'critical'
        : heapUsed >= HIGH_MEMORY_THRESHOLD
          ? 'high'
          : 'normal'
    
    // 优化：正常状态返回 null，避免重渲染
    setMemoryUsage(prev => {
      if (status === 'normal') return prev === null ? prev : null
      return { heapUsed, status }
    })
  }, 10_000)  // 10 秒间隔
  
  return memoryUsage
}
```

### process.memoryUsage()

Node.js 内置方法，返回：
```typescript
{
  rss: number,        // Resident Set Size - 进程占用的总内存
  heapTotal: number,  // V8 分配的堆内存总量
  heapUsed: number,   // V8 已使用的堆内存
  external: number,   // 外部内存使用（如 Buffer）
  arrayBuffers: number // ArrayBuffer 内存
}
```

---

## 关键代码路径与文件引用

```
src/hooks/useMemoryUsage.ts
├── MemoryUsageStatus 类型         # 行 4
├── MemoryUsageInfo 类型           # 行 6-9
├── HIGH_MEMORY_THRESHOLD          # 行 11: 1.5 GB
├── CRITICAL_MEMORY_THRESHOLD      # 行 12: 2.5 GB
└── useMemoryUsage()               # 行 18-39: 主钩子
    ├── useState                   # 行 19
    └── useInterval                # 行 21-36: 轮询逻辑
```

### 使用位置

```
src/components/notifications/Notifications.tsx
└── 使用 memoryUsage 决定是否显示内存警告
```

---

## 依赖与外部交互

### React Hooks 使用

- `useState`: 管理内存使用状态
- `useInterval`: 来自 `usehooks-ts`，定时轮询

### Node.js API

```typescript
const heapUsed = process.memoryUsage().heapUsed
```

### 外部依赖

```typescript
import { useInterval } from 'usehooks-ts'
```

---

## 风险、边界与改进建议

### 已知风险

1. **内存泄漏检测有限**
   - 只监控堆内存，不监控 RSS 或外部内存
   - 某些内存泄漏可能不在堆中

2. **阈值固定**
   - 1.5GB/2.5GB 阈值对所有用户相同
   - 不同系统可能有不同的内存限制

3. **无自动恢复**
   - 只报告状态，不自动采取措施
   - 用户可能忽略警告

### 边界情况

| 场景 | 行为 |
|-----|------|
| 堆内存 < 1.5GB | 返回 null |
| 堆内存 1.5-2.5GB | 返回 { heapUsed, status: 'high' } |
| 堆内存 ≥ 2.5GB | 返回 { heapUsed, status: 'critical' } |
| 从 high 降到 normal | 返回 null |
| 组件卸载 | 自动清理 interval |

### 改进建议

1. **动态阈值**
   - 根据系统总内存动态调整阈值
   - 考虑 32 位/64 位系统差异

2. **更多指标**
   - 监控 RSS 和外部内存
   - 计算内存增长趋势

3. **自动优化**
   - 达到 critical 时自动触发垃圾回收
   - 提示用户保存并重启

4. **历史趋势**
   - 记录内存使用历史
   - 显示内存增长图表

5. **内存分析**
   - 集成 heapdump 功能
   - 帮助诊断内存泄漏

### 测试建议

1. **单元测试**：
   - 阈值边界测试
   - 状态转换测试

2. **集成测试**：
   - 与 Notifications 组件集成
   - 长时间运行稳定性

3. **压力测试**：
   - 高内存使用场景
   - 验证及时性
