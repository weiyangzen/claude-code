# useMinDisplayTime.ts 深度研究文档

## 场景与职责

`useMinDisplayTime` 是一个用于限制值显示最小时间的 React 钩子。它确保每个不同的值在屏幕上至少显示指定的时间，防止快速变化的值闪烁而过，用户来不及阅读。

### 核心场景

1. **进度文本防抖**：防止快速轮询的进度文本闪烁
2. **状态显示稳定**：确保状态消息有足够显示时间
3. **用户体验优化**：避免 UI 快速跳动造成视觉疲劳

### 与其他组件的关系

- 被各种显示组件使用，如进度指示器、状态消息等
- 与 `useNotifyAfterTimeout` 形成时间控制钩子家族
- 独立工具钩子，无复杂依赖

---

## 功能点目的

### 1. 最小显示时间保证

与 debounce（防抖）和 throttle（节流）不同：
- **Debounce**：等待安静后执行
- **Throttle**：限制执行频率
- **MinDisplayTime**：保证每个值的最小显示时间

### 2. 延迟切换

当新值到来时：
- 如果当前值已显示足够时间，立即切换
- 如果当前值显示时间不足，延迟到满足最小时间后再切换

### 3. 清理机制

组件卸载时清理定时器，防止内存泄漏。

---

## 具体技术实现

### 关键数据结构

```typescript
// 泛型支持任何类型的值
export function useMinDisplayTime<T>(value: T, minMs: number): T
```

### 核心实现

```typescript
export function useMinDisplayTime<T>(value: T, minMs: number): T {
  const [displayed, setDisplayed] = useState(value)
  const lastShownAtRef = useRef(0)
  
  useEffect(() => {
    const elapsed = Date.now() - lastShownAtRef.current
    
    // 如果已显示足够时间，立即更新
    if (elapsed >= minMs) {
      lastShownAtRef.current = Date.now()
      setDisplayed(value)
      return
    }
    
    // 否则延迟更新
    const timer = setTimeout(
      (shownAtRef, setFn, v) => {
        shownAtRef.current = Date.now()
        setFn(v)
      },
      minMs - elapsed,
      lastShownAtRef,
      setDisplayed,
      value,
    )
    
    return () => clearTimeout(timer)
  }, [value, minMs])
  
  return displayed
}
```

### 工作流程

```
时间线：
0ms:    值 A 显示，lastShownAt = 0
100ms:  值 B 到来，elapsed = 100 < minMs(500)
        设置定时器 400ms 后切换
500ms:  定时器触发，显示值 B，lastShownAt = 500
600ms:  值 C 到来，elapsed = 100 < minMs(500)
        设置定时器 400ms 后切换
900ms:  定时器触发，显示值 C
```

---

## 关键代码路径与文件引用

```
src/hooks/useMinDisplayTime.ts
├── useMinDisplayTime<T>()         # 行 10-35: 主钩子
│   ├── useState - displayed       # 行 11
│   ├── useRef - lastShownAtRef    # 行 12
│   └── useEffect - 切换逻辑       # 行 14-34
│       ├── 立即切换分支           # 行 16-20
│       └── 延迟切换分支           # 行 21-32
```

### 使用场景示例

```typescript
// 进度指示器
const progressText = useMinDisplayTime(rawProgressText, 500)

// 状态消息
const statusMessage = useMinDisplayTime(currentStatus, 1000)
```

---

## 依赖与外部交互

### React Hooks 使用

- `useState`: 管理当前显示的值
- `useRef`: 跟踪上次显示时间
- `useEffect`: 处理值变化和定时器

### 无外部依赖

纯 React 实现，无其他依赖。

---

## 风险、边界与改进建议

### 已知风险

1. **快速连续变化**
   - 如果值变化非常频繁，可能导致显示严重滞后
   - 缓解：合理设置 minMs

2. **定时器累积**
   - 如果值在延迟期间再次变化，会取消旧定时器创建新定时器
   - 当前实现正确处理了这种情况

3. **初始值处理**
   - 初始渲染时 lastShownAtRef 为 0
   - 如果 minMs 很大，初始值可能延迟显示

### 边界情况

| 场景 | 行为 |
|-----|------|
| minMs=0 | 立即切换，无延迟 |
| 值不变 | 不触发 effect，保持原显示 |
| 组件卸载 | 清理定时器 |
| 快速连续变化 | 取消旧定时器，创建新定时器 |

### 改进建议

1. **最大延迟限制**
   - 添加最大延迟时间，防止显示过于滞后
   - 例如：即使 minMs=5000，最多延迟 1000ms

2. **优先级支持**
   - 高优先级值可以跳过延迟
   - 例如错误消息立即显示

3. **平滑过渡**
   - 添加过渡动画
   - 视觉提示即将切换

4. **批量处理**
   - 支持批量值变化
   - 只显示最后一个值

### 测试建议

1. **单元测试**：
   - 立即切换场景
   - 延迟切换场景
   - 快速连续变化
   - 组件卸载清理

2. **集成测试**：
   - 与进度组件集成
   - 与状态消息集成

3. **性能测试**：
   - 高频变化场景
   - 内存泄漏检查
