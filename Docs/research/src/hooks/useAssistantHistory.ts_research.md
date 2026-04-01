# useAssistantHistory.ts 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`useAssistantHistory.ts` 是 Claude Code 的远程会话历史懒加载 Hook，用于在 `claude assistant` 查看器模式下按需加载历史消息，支持滚动分页和视口填充。

### 1.2 使用场景
| 场景 | 描述 |
|------|------|
| 查看器模式启动 | 加载最新一页历史消息 |
| 向上滚动 | 接近顶部时自动加载更早的消息 |
| 视口填充 | 初始加载不足以填满屏幕时连锁加载 |
| 错误恢复 | 加载失败时显示重试提示 |

### 1.3 调用方
- `src/main.tsx` - 应用入口
- `src/screens/REPL.tsx` - 主界面（在 `feature('KAIROS')` 门控内）

---

## 2. 功能点目的

### 2.1 懒加载分页
- **目的**：避免一次性加载大量历史消息
- **实现**：基于游标的分页（`before_id`）
- **触发**：用户向上滚动接近顶部时

### 2.2 滚动锚定
- **目的**：加载新消息时保持视口位置不变
- **实现**：加载前记录高度，加载后补偿滚动位置

### 2.3 视口填充
- **目的**：确保初始加载有足够内容填满屏幕
- **实现**：连锁加载多页直到内容超过视口高度或达到上限

### 2.4 哨兵消息
- **目的**：提供加载状态和边界提示
- **类型**：
  - "loading older messages…"
  - "failed to load older messages — scroll up to retry"
  - "start of session"

---

## 3. 具体技术实现

### 3.1 类型定义

```typescript
type Props = {
  config: RemoteSessionConfig | undefined  // 查看器模式配置
  setMessages: React.Dispatch<React.SetStateAction<Message[]>>
  scrollRef: RefObject<ScrollBoxHandle | null>
  onPrepend?: (indexDelta: number, heightDelta: number) => void  // 预置回调
}

type Result = {
  maybeLoadOlder: (handle: ScrollBoxHandle) => void  // 滚动触发器
}

// 常量配置
const PREFETCH_THRESHOLD_ROWS = 40  // 距离顶部多少行触发加载
const MAX_FILL_PAGES = 10           // 视口填充最大连锁次数
```

### 3.2 核心状态（Refs）

```typescript
export function useAssistantHistory({ config, setMessages, scrollRef, onPrepend }: Props): Result {
  const enabled = config?.viewerOnly === true

  // 游标状态：null=无更多，undefined=初始未获取
  const cursorRef = useRef<string | null | undefined>(undefined)
  const ctxRef = useRef<HistoryAuthCtx | null>(null)
  const inflightRef = useRef(false)  // 防止重复请求

  // 滚动锚定：记录加载前高度
  const anchorRef = useRef<{ beforeHeight: number; count: number } | null>(null)

  // 视口填充预算
  const fillBudgetRef = useRef(0)

  // 稳定哨兵 UUID（虚拟滚动优化）
  const sentinelUuidRef = useRef(randomUUID())
}
```

### 3.3 页面转换

```typescript
function pageToMessages(page: HistoryPage): Message[] {
  const out: Message[] = []
  for (const ev of page.events) {
    const c = convertSDKMessage(ev, {
      convertUserTextMessages: true,
      convertToolResults: true,
    })
    if (c.type === 'message') out.push(c.message)
  }
  return out
}
```

### 3.4 预置消息（带锚定）

```typescript
const prepend = useCallback(
  (page: HistoryPage, isInitial: boolean) => {
    const msgs = pageToMessages(page)
    cursorRef.current = page.hasMore ? page.firstId : null

    // 非初始加载：记录锚定信息
    if (!isInitial) {
      const s = scrollRef.current
      anchorRef.current = s
        ? { beforeHeight: s.getFreshScrollHeight(), count: msgs.length }
        : null
    }

    // 更新或移除哨兵
    const sentinel = page.hasMore ? null : mkSentinel(SENTINEL_START)
    
    setMessages(prev => {
      // O(1) 移除现有哨兵（已知稳定 UUID）
      const base =
        prev[0]?.uuid === sentinelUuidRef.current ? prev.slice(1) : prev
      return sentinel ? [sentinel, ...msgs, ...base] : [...msgs, ...base]
    })
  },
  [setMessages],
)
```

### 3.5 初始加载

```typescript
useEffect(() => {
  if (!enabled || !config) return
  let cancelled = false
  
  void (async () => {
    // 1. 创建认证上下文
    const ctx = await createHistoryAuthCtx(config.sessionId).catch(() => null)
    if (!ctx || cancelled) return
    ctxRef.current = ctx

    // 2. 获取最新页面
    const page = await fetchLatestEvents(ctx)
    if (cancelled || !page) return

    // 3. 设置填充预算并预置
    fillBudgetRef.current = MAX_FILL_PAGES
    prepend(page, true)
  })()

  return () => { cancelled = true }
}, [enabled])  // config 身份稳定
```

### 3.6 加载更早消息

```typescript
const loadOlder = useCallback(async () => {
  if (!enabled || inflightRef.current) return
  const cursor = cursorRef.current
  const ctx = ctxRef.current
  if (!cursor || !ctx) return  // null=已耗尽，undefined=初始待处理

  inflightRef.current = true

  // 切换哨兵到 "loading..."
  setMessages(prev => {
    const base = prev[0]?.uuid === sentinelUuidRef.current ? prev.slice(1) : prev
    return [mkSentinel(SENTINEL_LOADING), ...base]
  })

  try {
    const page = await fetchOlderEvents(ctx, cursor)
    if (!page) {
      // 加载失败：显示重试提示
      setMessages(prev => {
        const base = prev[0]?.uuid === sentinelUuidRef.current ? prev.slice(1) : prev
        return [mkSentinel(SENTINEL_LOADING_FAILED), ...base]
      })
      return
    }
    prepend(page, false)
  } finally {
    inflightRef.current = false
  }
}, [enabled, prepend, setMessages])
```

### 3.7 滚动锚定补偿

```typescript
useLayoutEffect(() => {
  const anchor = anchorRef.current
  if (anchor === null) return
  anchorRef.current = null

  const s = scrollRef.current
  if (!s || s.isSticky()) return  // 粘性底部时无需补偿

  // 计算高度差并补偿
  const delta = s.getFreshScrollHeight() - anchor.beforeHeight
  if (delta > 0) s.scrollBy(delta)
  
  // 通知父组件调整 divider
  onPrepend?.(anchor.count, delta)
})
```

### 3.8 视口填充连锁

```typescript
useEffect(() => {
  if (
    fillBudgetRef.current <= 0 ||
    !cursorRef.current ||
    inflightRef.current
  ) {
    return
  }

  const s = scrollRef.current
  if (!s) return

  const contentH = s.getFreshScrollHeight()
  const viewH = s.getViewportHeight()

  // 内容未填满视口时继续加载
  if (contentH <= viewH) {
    fillBudgetRef.current--
    void loadOlder()
  } else {
    fillBudgetRef.current = 0
  }
})
```

### 3.9 滚动触发器

```typescript
const maybeLoadOlder = useCallback(
  (handle: ScrollBoxHandle) => {
    if (handle.getScrollTop() < PREFETCH_THRESHOLD_ROWS) {
      void loadOlder()
    }
  },
  [loadOlder],
)

return { maybeLoadOlder }
```

---

## 4. 关键代码路径与文件引用

### 4.1 依赖图

```
useAssistantHistory.ts
├── react (useCallback, useEffect, useLayoutEffect, useRef)
├── crypto (randomUUID)
├── assistant/sessionHistory.js
│   ├── createHistoryAuthCtx
│   ├── fetchLatestEvents
│   └── fetchOlderEvents
├── ink/components/ScrollBox.js  (ScrollBoxHandle)
├── remote/RemoteSessionManager.js  (RemoteSessionConfig)
├── remote/sdkMessageAdapter.js  (convertSDKMessage)
├── types/message.js             (Message, SystemInformationalMessage)
└── utils/debug.js               (logForDebugging)
```

### 4.2 调用链

```
REPL.tsx (viewerOnly mode)
  └── useAssistantHistory()
      ├── useEffect (初始加载)
      │   ├── createHistoryAuthCtx()
      │   ├── fetchLatestEvents()
      │   └── prepend(page, true)
      ├── loadOlder() (用户滚动触发)
      │   ├── fetchOlderEvents()
      │   └── prepend(page, false)
      ├── useLayoutEffect (锚定补偿)
      └── useEffect (视口填充连锁)
```

### 4.3 哨兵消息创建

```typescript
function mkSentinel(text: string): SystemInformationalMessage {
  return {
    type: 'system',
    subtype: 'informational',
    content: text,
    isMeta: false,
    timestamp: new Date().toISOString(),
    uuid: sentinelUuidRef.current,  // 稳定 UUID
    level: 'info',
  }
}
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 | 类型 |
|------|------|------|
| `react` | Hooks API | npm 包 |
| `crypto` | randomUUID | Node.js 内置 |

### 5.2 远程历史 API

| 函数 | 来源 | 用途 |
|------|------|------|
| `createHistoryAuthCtx()` | `assistant/sessionHistory.js` | 创建认证上下文 |
| `fetchLatestEvents()` | `assistant/sessionHistory.js` | 获取最新页面 |
| `fetchOlderEvents()` | `assistant/sessionHistory.js` | 获取更早页面 |
| `convertSDKMessage()` | `remote/sdkMessageAdapter.js` | SDK 消息转换 |

### 5.3 ScrollBox 接口

| 方法 | 用途 |
|------|------|
| `getFreshScrollHeight()` | 获取当前内容高度 |
| `getViewportHeight()` | 获取视口高度 |
| `getScrollTop()` | 获取滚动位置 |
| `scrollBy(delta)` | 补偿滚动 |
| `isSticky()` | 检查是否固定在底部 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 缓解 |
|------|------|------|
| 竞态条件 | 快速滚动可能触发重复加载 | `inflightRef` 锁 |
| 内存泄漏 | 大量历史消息累积 | 虚拟滚动（未实现） |
| 高度计算误差 | Yoga 布局引擎高度可能不准确 | 使用 `getFreshScrollHeight()` |
| 网络失败 | 加载失败时用户体验 | 哨兵消息提示重试 |

### 6.2 边界条件

1. **无历史**：`fetchLatestEvents` 返回空页面
2. **单页历史**：`hasMore=false`，显示 "start of session"
3. **全部过滤**：事件转换为 0 条消息，`MAX_FILL_PAGES` 限制连锁
4. **组件卸载**：`cancelled` 标志防止状态更新
5. **滚动到底部**：`isSticky()` 跳过锚定补偿

### 6.3 改进建议

1. **虚拟滚动**：
   ```typescript
   // 仅渲染可见消息
   const { virtualItems, totalHeight } = useVirtualizer(messages)
   ```

2. **预加载**：
   ```typescript
   // 接近阈值时预加载下一页
   const preloadThreshold = PREFETCH_THRESHOLD_ROWS * 2
   ```

3. **错误重试**：
   ```typescript
   // 指数退避重试
   const retryWithBackoff = useRetry({ maxAttempts: 3, baseDelay: 1000 })
   ```

4. **缓存**：
   ```typescript
   // 缓存已加载页面
   const pageCache = useRef(new Map<string, HistoryPage>())
   ```

5. **性能监控**：
   ```typescript
   // 记录加载性能指标
   performance.mark('history-load-start')
   // ... 加载 ...
   performance.mark('history-load-end')
   performance.measure('history-load', 'history-load-start', 'history-load-end')
   ```

### 6.4 代码质量

- **优点**：
  - 清晰的 Ref 状态管理
  - O(1) 哨兵移除优化
  - 完善的加载状态处理
  - 滚动锚定确保用户体验
  
- **潜在改进**：
  - 提取哨兵逻辑为独立 Hook
  - 添加更多 JSDoc 示例
  - 考虑使用 React Query 管理远程状态

### 6.5 相关文档

- `assistant/sessionHistory.js` - 远程历史 API
- `remote/sdkMessageAdapter.js` - 消息格式转换
- `ink/components/ScrollBox.js` - 滚动容器组件
