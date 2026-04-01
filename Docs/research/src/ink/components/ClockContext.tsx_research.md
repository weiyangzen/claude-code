# ClockContext.tsx 研究文档

## 场景与职责

`ClockContext` 是 Ink 框架中的时间/动画同步系统，提供：

1. **全局时间源**: 为动画和定时更新提供统一的时间基准
2. **订阅机制**: 支持组件订阅时间变化，实现动画帧更新
3. **焦点感知**: 根据终端焦点状态自动调整 tick 间隔（聚焦时 60fps，失焦时 30fps）
4. **性能优化**: 通过 keepAlive 机制，只在有订阅者时运行定时器

ClockContext 主要用于驱动动画（如加载动画、进度指示器）和需要定时更新的场景。

## 功能点目的

### 1. 时钟创建与管理
- **createClock**: 工厂函数创建时钟实例
- **订阅模型**: 支持多个订阅者，每个订阅者可设置 keepAlive
- **tick 同步**: 同一 tick 内所有订阅者看到相同的时间值，保证动画同步

### 2. 焦点感知调度
- **聚焦时**: FRAME_INTERVAL_MS (16ms ≈ 60fps)，流畅动画
- **失焦时**: FRAME_INTERVAL_MS * 2 (32ms ≈ 30fps)，节省资源
- **自动切换**: 通过 useTerminalFocus 监听焦点变化

### 3. 性能优化
- **按需启动**: 只有当有 keepAlive 订阅者时才启动定时器
- **时间快照**: tickTime 在 tick 开始时记录，确保同一 tick 内所有订阅者看到相同时间
- **暂停优化**: 没有 keepAlive 订阅者时，返回实时时间而非 stale tickTime

### 4. React Context 集成
- **ClockContext**: 提供时钟实例的 Context
- **ClockProvider**: 提供时钟实例的 Provider 组件
- **独立组件设计**: 单独组件避免 App.tsx 在时钟创建时重渲染

## 具体技术实现

### 关键数据结构

```typescript
// 时钟接口
export type Clock = {
  subscribe: (onChange: () => void, keepAlive: boolean) => () => void;
  now: () => number;
  setTickInterval: (ms: number) => void;
};

// 内部状态
const subscribers = new Map<() => void, boolean>();  // 订阅者 -> keepAlive
let interval: ReturnType<typeof setInterval> | null = null;
let currentTickIntervalMs = tickIntervalMs;
let startTime = 0;
let tickTime = 0;  // 当前 tick 的时间快照
```

### 关键流程

1. **时钟创建**
   ```typescript
   export function createClock(tickIntervalMs: number): Clock {
     const subscribers = new Map<() => void, boolean>();
     // ... 初始化状态
     
     function tick(): void {
       tickTime = Date.now() - startTime;
       for (const onChange of subscribers.keys()) {
         onChange();
       }
     }
     
     function updateInterval(): void {
       const anyKeepAlive = [...subscribers.values()].some(Boolean);
       if (anyKeepAlive) {
         // 启动或重启定时器
       } else if (interval) {
         // 停止定时器
       }
     }
     
     return { subscribe, now, setTickInterval };
   }
   ```

2. **订阅机制**
   ```typescript
   subscribe(onChange, keepAlive) {
     subscribers.set(onChange, keepAlive);
     updateInterval();
     return () => {
       subscribers.delete(onChange);
       updateInterval();
     };
   }
   ```

3. **时间获取**
   ```typescript
   now() {
     if (startTime === 0) startTime = Date.now();
     // 定时器运行时返回同步的 tickTime，否则返回实时时间
     if (interval && tickTime) return tickTime;
     return Date.now() - startTime;
   }
   ```

4. **Provider 实现**
   ```typescript
   export function ClockProvider({ children }) {
     const [clock] = useState(() => createClock(FRAME_INTERVAL_MS));
     const focused = useTerminalFocus();
     
     useEffect(() => {
       clock.setTickInterval(focused ? FRAME_INTERVAL_MS : BLURRED_TICK_INTERVAL_MS);
     }, [clock, focused]);
     
     return <ClockContext.Provider value={clock}>{children}</ClockContext.Provider>;
   }
   ```

### 代码路径

```
ClockContext.tsx
├── 导入依赖（React、constants、useTerminalFocus）
├── Clock 类型定义
├── createClock 工厂函数
│   ├── 初始化订阅者 Map
│   ├── tick 函数（触发所有订阅者）
│   ├── updateInterval 函数（管理定时器）
│   └── 返回 Clock 接口
├── ClockContext 创建
├── BLURRED_TICK_INTERVAL_MS 常量
├── ClockProvider 组件
│   ├── useState 创建时钟（一次性）
│   ├── useTerminalFocus 获取焦点状态
│   ├── useEffect 调整 tick 间隔
│   └── 渲染 Provider
└── 导出
```

## 关键代码路径与文件引用

### 核心依赖

| 文件 | 用途 |
|------|------|
| `src/ink/constants.ts` | FRAME_INTERVAL_MS 常量 (~60fps) |
| `src/ink/hooks/use-terminal-focus.ts` | 获取终端焦点状态 |
| `src/ink/components/TerminalFocusContext.tsx` | 终端焦点 Context |

### 被调用方

ClockContext 主要被以下 hooks 使用：
- `src/ink/hooks/use-animation-frame.ts` - 动画帧 hook
- 各种需要定时更新的动画组件

### 使用示例

```typescript
import { useContext } from 'react';
import { ClockContext } from './ClockContext';

function useAnimation(callback: () => void, keepAlive: boolean) {
  const clock = useContext(ClockContext);
  useEffect(() => {
    if (!clock) return;
    return clock.subscribe(callback, keepAlive);
  }, [clock, callback, keepAlive]);
}
```

## 依赖与外部交互

### 运行时依赖

1. **React**: useState、useEffect、createContext
2. **React Compiler**: 使用 `_c` 函数进行自动记忆化
3. **Terminal Focus State**: 通过 useTerminalFocus 监听焦点变化

### 交互流程

```
ClockContext
├── createClock
│   ├── subscribers Map 管理订阅者
│   ├── setInterval 管理 tick
│   └── tickTime 同步时间
├── ClockProvider
│   ├── useState 创建时钟（稳定引用）
│   ├── useTerminalFocus 监听焦点
│   └── useEffect 调整 setTickInterval
└── 消费者
    ├── useContext(ClockContext) 获取时钟
    ├── clock.subscribe 订阅更新
    └── clock.now() 获取时间
```

### 焦点感知流程

```
终端焦点变化
├── TerminalFocusContext 更新
├── useTerminalFocus 返回新值
├── ClockProvider effect 触发
├── clock.setTickInterval 调用
└── updateInterval 调整定时器间隔
    ├── 聚焦: 16ms (60fps)
    └── 失焦: 32ms (30fps)
```

## 风险、边界与改进建议

### 已知风险

1. **时间漂移**: 使用 setInterval 可能导致时间漂移，长时间运行后 tick 间隔可能不准确
2. **订阅者泄漏**: 如果订阅者忘记调用 unsubscribe，可能导致内存泄漏
3. **并发问题**: tick 函数同步调用所有订阅者，如果某个订阅者耗时较长，会阻塞其他订阅者

### 边界情况

1. **startTime 初始化**: now() 在第一次调用时才初始化 startTime，可能导致第一次返回值与预期不同
2. **keepAlive 全部为 false**: 此时定时器停止，now() 返回实时时间，可能与预期的时间同步行为不同
3. **快速订阅/取消**: 频繁的订阅和取消可能导致定时器频繁启停

### 改进建议

1. **使用 requestAnimationFrame**: 考虑使用更现代的调度机制，虽然 Node.js 环境没有原生 rAF
2. **时间精度**: 考虑使用 performance.now() 替代 Date.now() 获得更高精度
3. **防抖优化**: 对 updateInterval 进行防抖，避免频繁启停定时器
4. **订阅者优先级**: 考虑支持优先级机制，重要的动画优先更新
5. **时间校准**: 定期校准时间，防止 setInterval 漂移累积

6. **错误边界**: 在 tick 函数中包装 try-catch，防止单个订阅者错误影响其他订阅者
   ```typescript
   function tick(): void {
     tickTime = Date.now() - startTime;
     for (const onChange of subscribers.keys()) {
       try {
         onChange();
       } catch (e) {
         console.error('Clock tick error:', e);
       }
     }
   }
   ```

### 性能考虑

- 当前实现使用 Map 存储订阅者，查找和删除都是 O(1)
- tick 函数同步遍历所有订阅者，如果订阅者很多可能影响性能
- 考虑使用链表结构优化订阅者管理，避免遍历开销

### 测试建议

- 测试焦点切换时 tick 间隔是否正确调整
- 测试多个订阅者的同步行为
- 测试 keepAlive 机制是否正确启停定时器
- 测试长时间运行的时钟漂移情况
