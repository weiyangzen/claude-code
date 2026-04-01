# useTimeout.ts 深度研究文档

## 场景与职责

`useTimeout` 是一个极简的 React Hook，用于在指定延迟后触发状态变化。它封装了 `setTimeout` 的常用模式，提供了响应式的超时功能。

### 核心职责

1. **延迟触发**: 在指定延迟后自动设置状态为 true
2. **自动重置**: 依赖变化时自动重置计时器
3. **清理管理**: 组件卸载时自动清理计时器

### 使用场景

- **延迟显示**: 延迟显示提示、通知或 UI 元素
- **超时处理**: 实现操作超时逻辑
- **防抖/节流**: 配合其他 Hook 实现防抖或节流
- **动画延迟**: 控制动画的延迟触发

---

## 功能点目的

### 1. 延迟状态切换

在指定延迟后将 `isElapsed` 状态设置为 `true`。

### 2. 自动重置

当 `delay` 或 `resetTrigger` 变化时：
- 重置 `isElapsed` 为 `false`
- 重新启动计时器

### 3. 自动清理

组件卸载时自动清除计时器，防止内存泄漏。

---

## 具体技术实现

### 关键数据结构

```typescript
// Hook 签名
function useTimeout(delay: number, resetTrigger?: number): boolean

// 返回类型
interface UseTimeoutReturn {
  isElapsed: boolean  // 是否已超时
}
```

### 核心流程

```
调用 useTimeout(delay, resetTrigger)
  ↓
创建 isElapsed 状态（初始 false）
  ↓
useEffect 执行:
  1. 设置 isElapsed = false
  2. 创建 setTimeout，延迟后设置 isElapsed = true
  3. 返回清理函数清除计时器
  ↓
依赖 [delay, resetTrigger] 变化时:
  1. 执行清理函数
  2. 重新执行 effect
```

### 关键代码路径

#### Hook 实现（行 3-14）
```typescript
export function useTimeout(delay: number, resetTrigger?: number): boolean {
  const [isElapsed, setIsElapsed] = useState(false)

  useEffect(() => {
    setIsElapsed(false)
    const timer = setTimeout(setIsElapsed, delay, true)

    return () => clearTimeout(timer)
  }, [delay, resetTrigger])

  return isElapsed
}
```

### 实现要点

1. **立即重置**: Effect 首先设置 `isElapsed = false`，确保依赖变化时状态重置
2. **函数式 setTimeout**: 使用 `setTimeout(callback, delay, arg)` 形式直接传递参数
3. **清理函数**: 返回箭头函数清除计时器
4. **依赖数组**: `[delay, resetTrigger]` 确保任一变化都重启计时器

---

## 依赖与外部交互

### 核心依赖

| 模块 | 用途 |
|------|------|
| `react` | `useEffect`, `useState` |

### 外部交互

无外部依赖，纯 React 实现。

---

## 风险、边界与改进建议

### 已知风险

1. **延迟为负**: 传入负数 `delay` 可能导致意外行为
2. **延迟为零**: 零延迟可能导致立即执行，但状态更新可能异步
3. **频繁重置**: `resetTrigger` 频繁变化可能导致频繁重启计时器

### 边界情况

1. **组件卸载**: 清理函数确保计时器不会尝试更新已卸载组件
2. **延迟变化**: 延迟变化时立即重置，可能导致预期外的行为
3. **resetTrigger 相同值**: React 的依赖比较使用 `Object.is`，相同值不会触发重置

### 改进建议

1. **参数验证**: 添加 `delay` 参数验证，确保为非负数
   ```typescript
   if (delay < 0) {
     console.warn('useTimeout: delay should be non-negative')
     delay = 0
   }
   ```

2. **暂停/恢复**: 添加暂停和恢复功能
   ```typescript
   function useTimeout(delay: number, options?: { paused?: boolean }): boolean
   ```

3. **剩余时间**: 返回剩余时间用于进度显示
   ```typescript
   interface UseTimeoutReturn {
     isElapsed: boolean
     remainingTime: number
   }
   ```

4. **提前完成**: 提供手动完成计时器的方法
   ```typescript
   interface UseTimeoutReturn {
     isElapsed: boolean
     complete: () => void
   }
   ```

### 测试关注点

1. 延迟后状态正确变为 true
2. 依赖变化时计时器重置
3. 组件卸载时计时器清理
4. 零延迟行为
5. 频繁重置的处理
