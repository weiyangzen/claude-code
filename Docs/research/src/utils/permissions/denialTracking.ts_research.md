# denialTracking.ts 研究文档

## 场景与职责

`denialTracking.ts` 是 Claude Code 权限分类器的拒绝跟踪基础设施，用于跟踪连续拒绝和总拒绝次数，以决定何时回退到提示模式（prompting）。

该模块的核心职责：

1. **拒绝状态跟踪**：记录连续拒绝次数和总拒绝次数
2. **回退决策**：当拒绝次数超过阈值时，决定回退到用户提示
3. **状态管理**：提供不可变的状态更新函数

## 功能点目的

### 1. 拒绝跟踪状态

```typescript
export type DenialTrackingState = {
  consecutiveDenials: number  // 连续拒绝次数
  totalDenials: number        // 总拒绝次数
}
```

### 2. 拒绝限制常量

```typescript
export const DENIAL_LIMITS = {
  maxConsecutive: 3,  // 最大连续拒绝次数
  maxTotal: 20,       // 最大总拒绝次数
} as const
```

### 3. 状态操作函数

- `createDenialTrackingState()`：创建初始状态
- `recordDenial(state)`：记录一次拒绝，增加计数器
- `recordSuccess(state)`：记录一次成功，重置连续拒绝计数
- `shouldFallbackToPrompting(state)`：检查是否应该回退到提示

## 具体技术实现

### 状态创建

```typescript
export function createDenialTrackingState(): DenialTrackingState {
  return {
    consecutiveDenials: 0,
    totalDenials: 0,
  }
}
```

### 记录拒绝

```typescript
export function recordDenial(state: DenialTrackingState): DenialTrackingState {
  return {
    ...state,
    consecutiveDenials: state.consecutiveDenials + 1,
    totalDenials: state.totalDenials + 1,
  }
}
```

实现特点：
- 使用展开运算符创建新对象，保持不可变性
- 同时增加连续拒绝和总拒绝计数

### 记录成功

```typescript
export function recordSuccess(state: DenialTrackingState): DenialTrackingState {
  if (state.consecutiveDenials === 0) return state // 无变化需要
  return {
    ...state,
    consecutiveDenials: 0,
  }
}
```

实现特点：
- 如果连续拒绝已为 0，直接返回原状态（优化）
- 只重置连续拒绝计数，保留总拒绝计数

### 回退决策

```typescript
export function shouldFallbackToPrompting(state: DenialTrackingState): boolean {
  return (
    state.consecutiveDenials >= DENIAL_LIMITS.maxConsecutive ||
    state.totalDenials >= DENIAL_LIMITS.maxTotal
  )
}
```

回退条件：
- 连续拒绝达到或超过 3 次，或
- 总拒绝达到或超过 20 次

## 关键代码路径与文件引用

### 导出内容
| 导出 | 类型 | 说明 |
|------|------|------|
| `DenialTrackingState` | Type | 拒绝跟踪状态类型 |
| `DENIAL_LIMITS` | Constant | 拒绝限制常量 |
| `createDenialTrackingState` | Function | 创建初始状态 |
| `recordDenial` | Function | 记录拒绝 |
| `recordSuccess` | Function | 记录成功 |
| `shouldFallbackToPrompting` | Function | 检查是否回退 |

### 被引用位置
- `src/Tool.ts`：`DenialTrackingState` 类型导入
- `src/utils/permissions/permissions.ts`：拒绝跟踪逻辑
- 自动模式分类器决策流程

### 状态流转

```
初始状态: { consecutiveDenials: 0, totalDenials: 0 }
    │
    ├── 拒绝 ──→ recordDenial ──→ { consecutiveDenials: 1, totalDenials: 1 }
    │                              │
    │                              ├── 拒绝 ──→ { consecutiveDenials: 2, totalDenials: 2 }
    │                              │              │
    │                              │              ├── 拒绝 ──→ { consecutiveDenials: 3, totalDenials: 3 }
    │                              │              │              │
    │                              │              │              ├── 连续拒绝 >= 3 ──→ 回退到提示
    │                              │              │              │
    │                              │              │              └── 成功 ──→ { consecutiveDenials: 0, totalDenials: 3 }
    │                              │              │
    │                              │              └── 成功 ──→ { consecutiveDenials: 0, totalDenials: 2 }
    │                              │
    │                              └── 成功 ──→ { consecutiveDenials: 0, totalDenials: 1 }
    │
    └── 成功 ──→ recordSuccess ──→ { consecutiveDenials: 0, totalDenials: 0 } (无变化)
```

## 依赖与外部交互

### 上游依赖
该模块**无外部导入**，完全自包含。

### 下游消费者
| 模块 | 用途 |
|------|------|
| `Tool.ts` | `DenialTrackingState` 类型定义 |
| `permissions.ts` | 拒绝跟踪和回退逻辑 |

## 风险、边界与改进建议

### 风险点
1. **阈值硬编码**：`maxConsecutive: 3` 和 `maxTotal: 20` 是硬编码的，可能需要针对不同场景调整
2. **状态持久化**：当前状态仅在内存中，会话重启后重置
3. **误报累积**：如果分类器有系统性误报，可能快速触发回退

### 边界条件
1. **整数溢出**：虽然 JavaScript 数字可以很大，但 `totalDenials` 理论上可能溢出（实际不太可能达到）
2. **并发访问**：状态对象在多个异步操作中共享，需要确保正确的更新顺序
3. **成功重置**：成功只重置连续拒绝，总拒绝永不重置

### 改进建议
1. **可配置阈值**：从配置或远程设置加载阈值

```typescript
// 建议：可配置阈值
export function getDenialLimits(): { maxConsecutive: number; maxTotal: number } {
  return {
    maxConsecutive: getConfig('denial.maxConsecutive') ?? 3,
    maxTotal: getConfig('denial.maxTotal') ?? 20,
  }
}
```

2. **指数退避**：考虑使用指数退避策略而非固定阈值

```typescript
// 建议：指数退避
export function shouldFallbackToPrompting(state: DenialTrackingState): boolean {
  const baseDelay = 1000 // 1 second
  const maxDelay = 30000 // 30 seconds
  const delay = Math.min(baseDelay * Math.pow(2, state.consecutiveDenials), maxDelay)
  // ... 使用 delay 进行退避
}
```

3. **持久化状态**：考虑在会话间持久化拒绝状态
4. **分类统计**：按工具类型或拒绝原因分类统计

```typescript
// 建议：分类统计
export type CategorizedDenialState = {
  byTool: Map<string, number>
  byReason: Map<string, number>
  consecutiveDenials: number
  totalDenials: number
}
```

5. **自动恢复**：在回退到提示后，考虑在条件改善时自动恢复自动模式
6. **监控告警**：当拒绝率异常高时，触发监控告警
