# signal.ts 深度研究

## 场景与职责

`signal.ts` 是一个**极简的事件信号原语**，用于实现发布-订阅模式。它将原本在代码库中重复约 15 次的监听器集合样板代码压缩为一行创建。

**核心职责：**
1. 提供无状态的事件通知机制
2. 替代重复的 `new Set()` + `subscribe` + `notify` 样板
3. 区别于状态存储（AppState、createStore），仅通知事件发生

**应用场景：**
- 设置变更通知
- 技能变更检测
- 分析事件流
- 任何需要"某事发生"通知但不需要"当前值"的场景

---

## 功能点目的

### 1. Signal 类型定义
```typescript
export type Signal<Args extends unknown[] = []> = {
  subscribe: (listener: (...args: Args) => void) => () => void
  emit: (...args: Args) => void
  clear: () => void
}
```

**设计特点：**
- 泛型参数 `Args` 支持自定义事件参数
- `subscribe` 返回取消订阅函数
- 无 `getState` 方法（区别于 Store）

### 2. Signal 创建
```typescript
export function createSignal<Args extends unknown[] = []>(): Signal<Args>
```

**实现：**
```typescript
export function createSignal<Args extends unknown[] = []>(): Signal<Args> {
  const listeners = new Set<(...args: Args) => void>()
  return {
    subscribe(listener) {
      listeners.add(listener)
      return () => {
        listeners.delete(listener)
      }
    },
    emit(...args) {
      for (const listener of listeners) listener(...args)
    },
    clear() {
      listeners.clear()
    },
  }
}
```

---

## 具体技术实现

### 内存管理
- 使用 `Set` 存储监听器，自动去重
- 取消订阅函数通过闭包捕获监听器引用
- `clear()` 方法用于 dispose/reset 场景

### 类型安全
- 泛型参数确保事件参数类型安全
- 默认 `Args = []` 支持无参数事件

### 使用模式
```typescript
// 创建信号
const changed = createSignal<[SettingSource]>()

// 导出订阅函数
export const subscribe = changed.subscribe

// 触发事件
changed.emit('userSettings')
```

---

## 关键代码路径与文件引用

### 核心导出
| 导出 | 用途 |
|------|------|
| `Signal` | 信号类型定义 |
| `createSignal` | 信号创建函数 |

### 调用方
| 文件 | 用途 |
|------|------|
| `src/keybindings/loadUserBindings.ts` | 键绑定变更 |
| `src/skills/loadSkillsDir.ts` | 技能目录变更 |
| `src/services/analytics/growthbook.ts` | GrowthBook 特性变更 |
| `src/bootstrap/state.ts` | 全局状态变更 |
| `src/utils/fastMode.ts` | 快速模式变更 |
| `src/utils/settings/changeDetector.ts` | 设置变更检测 |
| `src/utils/messageQueueManager.ts` | 消息队列变更 |
| `src/hooks/fileSuggestions.ts` | 文件建议变更 |
| `src/hooks/useTasksV2.ts` | 任务变更 |
| `src/utils/awsAuthStatusManager.ts` | AWS 认证状态 |
| `src/utils/suggestions/slackChannelSuggestions.ts` | Slack 频道建议 |
| `src/utils/skills/skillChangeDetector.ts` | 技能变更检测 |
| `src/utils/tasks.ts` | 任务状态 |
| `src/utils/classifierApprovals.ts` | 分类器审批 |
| `src/utils/claudeCodeHints.ts` | 提示变更 |
| `src/utils/QueryGuard.ts` | 查询守卫 |
| `src/utils/mailbox.ts` | 邮箱变更 |

---

## 依赖与外部交互

### 外部依赖
- 无外部依赖

### 内部依赖
- 无内部依赖

---

## 风险、边界与改进建议

### 已知风险

1. **内存泄漏**
   - 未取消订阅的监听器持续占用内存
   - 缓解：始终使用返回的取消订阅函数

2. **同步执行**
   - 监听器同步执行，可能阻塞 emit 调用
   - 长时间运行的监听器影响性能

3. **错误传播**
   - 监听器抛出的错误会中断其他监听器
   - 无内置错误隔离机制

### 边界情况

| 场景 | 处理 |
|------|------|
| 重复订阅同一函数 | Set 自动去重 |
| emit 时订阅/取消订阅 | 不影响当前 emit（Set 遍历安全） |
| 无监听器时 emit | 空操作 |
| 空参数 emit | 支持（默认 Args = []） |

### 改进建议

1. **异步支持**
   - 添加 `emitAsync` 变体
   - 支持微任务或宏任务调度

2. **错误处理**
   - 添加监听器错误隔离
   - 提供错误回调或事件

3. **一次性订阅**
   - 添加 `once` 方法
   - 自动取消订阅执行一次的监听器

4. **性能优化**
   - 添加监听器数量限制
   - 实现优先级/顺序控制

5. **调试支持**
   - 添加监听器数量统计
   - 记录 emit 历史（开发模式）

6. **与 RxJS 对比**
   - 考虑是否需要 Observable 接口
   - 评估迁移到标准事件库的成本
