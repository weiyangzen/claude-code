# useAwaySummary.ts 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`useAwaySummary.ts` 是 Claude Code 的"离开期间摘要"功能 Hook，当用户离开终端超过 5 分钟后返回时，自动生成并显示会话摘要消息。

### 1.2 使用场景
| 场景 | 描述 |
|------|------|
| 长时间离开 | 用户离开 5+ 分钟后返回终端 |
| 多任务切换 | 用户在 Claude 工作时切换到其他应用 |
| 会话恢复 | 从长时间暂停中恢复工作上下文 |

### 1.3 调用方
- `src/screens/REPL.tsx` - 主应用界面

---

## 2. 功能点目的

### 2.1 离开检测
- **目的**：检测用户何时离开终端
- **实现**：监听终端焦点状态（DECSET 1004）
- **延迟**：5 分钟（`BLUR_DELAY_MS = 5 * 60_000`）

### 2.2 摘要生成
- **目的**：为用户生成离开期间发生事情的摘要
- **实现**：调用 `generateAwaySummary` 服务
- **条件**：
  - 终端已离开 5 分钟
  - 没有进行中的对话回合
  - 自上次用户消息后未生成过摘要

### 2.3 防重复
- **目的**：避免生成多个冗余摘要
- **实现**：`hasSummarySinceLastUserTurn` 检查

---

## 3. 具体技术实现

### 3.1 类型定义

```typescript
type SetMessages = (updater: (prev: Message[]) => Message[]) => void

// 检查自上次用户回合后是否已有摘要
function hasSummarySinceLastUserTurn(messages: readonly Message[]): boolean {
  for (let i = messages.length - 1; i >= 0; i--) {
    const m = messages[i]!
    // 找到用户消息且非元消息：停止搜索
    if (m.type === 'user' && !m.isMeta && !m.isCompactSummary) return false
    // 找到摘要消息：返回 true
    if (m.type === 'system' && m.subtype === 'away_summary') return true
  }
  return false
}
```

### 3.2 核心 Hook 实现

```typescript
export function useAwaySummary(
  messages: readonly Message[],
  setMessages: SetMessages,
  isLoading: boolean,
): void {
  const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null)
  const abortRef = useRef<AbortController | null>(null)
  const messagesRef = useRef(messages)
  const isLoadingRef = useRef(isLoading)
  const pendingRef = useRef(false)  // 等待加载完成标志
  const generateRef = useRef<(() => Promise<void>) | null>(null)

  // 同步 Refs
  messagesRef.current = messages
  isLoadingRef.current = isLoading

  // GrowthBook 功能开关（3P 默认关闭）
  const gbEnabled = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_sedge_lantern',
    false,
  )

  useEffect(() => {
    // 编译时功能门控
    if (!feature('AWAY_SUMMARY')) return
    // 运行时功能开关
    if (!gbEnabled) return

    // 清理函数
    function clearTimer(): void {
      if (timerRef.current !== null) {
        clearTimeout(timerRef.current)
        timerRef.current = null
      }
    }

    function abortInFlight(): void {
      abortRef.current?.abort()
      abortRef.current = null
    }

    // 摘要生成
    async function generate(): Promise<void> {
      pendingRef.current = false
      if (hasSummarySinceLastUserTurn(messagesRef.current)) return
      
      abortInFlight()
      const controller = new AbortController()
      abortRef.current = controller
      
      const text = await generateAwaySummary(
        messagesRef.current,
        controller.signal,
      )
      
      if (controller.signal.aborted || text === null) return
      setMessages(prev => [...prev, createAwaySummaryMessage(text)])
    }

    // 离开定时器触发
    function onBlurTimerFire(): void {
      timerRef.current = null
      if (isLoadingRef.current) {
        pendingRef.current = true  // 等待回合结束
        return
      }
      void generate()
    }

    // 焦点变化处理
    function onFocusChange(): void {
      const state = getTerminalFocusState()
      if (state === 'blurred') {
        clearTimer()
        timerRef.current = setTimeout(onBlurTimerFire, BLUR_DELAY_MS)
      } else if (state === 'focused') {
        clearTimer()
        abortInFlight()
        pendingRef.current = false
      }
      // 'unknown' → 无操作
    }

    const unsubscribe = subscribeTerminalFocus(onFocusChange)
    onFocusChange()  // 处理已离开的情况
    generateRef.current = generate

    return () => {
      unsubscribe()
      clearTimer()
      abortInFlight()
      generateRef.current = null
    }
  }, [gbEnabled, setMessages])

  // 回合结束时处理待处理的摘要
  useEffect(() => {
    if (isLoading) return
    if (!pendingRef.current) return
    if (getTerminalFocusState() !== 'blurred') return
    void generateRef.current?.()
  }, [isLoading])
}
```

### 3.3 防重复逻辑详解

```typescript
function hasSummarySinceLastUserTurn(messages: readonly Message[]): boolean {
  // 从最新消息开始逆向遍历
  for (let i = messages.length - 1; i >= 0; i--) {
    const m = messages[i]!
    
    // 找到用户消息（非元消息、非紧凑摘要）：摘要已过期
    if (m.type === 'user' && !m.isMeta && !m.isCompactSummary) {
      return false
    }
    
    // 找到摘要消息：无需再生成
    if (m.type === 'system' && m.subtype === 'away_summary') {
      return true
    }
  }
  
  // 遍历完成未找到用户消息或摘要
  return false
}
```

**场景示例**：
```
消息历史（从新到旧）：
1. [Assistant] 回复...          ← 当前
2. [System] away_summary         ← 找到摘要，返回 true
3. [User] 请继续...              ← 不会到达（已在 #2 返回）

消息历史（另一种情况）：
1. [Assistant] 回复...
2. [User] 新问题...              ← 找到用户消息，返回 false
3. [System] away_summary         ← 不会到达（旧的摘要已过期）
```

---

## 4. 关键代码路径与文件引用

### 4.1 依赖图

```
useAwaySummary.ts
├── react (useEffect, useRef)
├── bun:bundle (feature)
├── ink/terminal-focus-state.js
│   ├── getTerminalFocusState
│   └── subscribeTerminalFocus
├── services/analytics/growthbook.js  (getFeatureValue_CACHED_MAY_BE_STALE)
├── services/awaySummary.js           (generateAwaySummary)
├── types/message.js                  (Message)
└── utils/messages.js                 (createAwaySummaryMessage)
```

### 4.2 调用链

```
REPL.tsx
  └── useAwaySummary(messages, setMessages, isLoading)
      ├── subscribeTerminalFocus(onFocusChange)
      │   └── onFocusChange()
      │       ├── 'blurred' → setTimeout(5min)
      │       └── 'focused' → clearTimer()
      └── useEffect(isLoading)
          └── generate() [如果 pending]
```

### 4.3 摘要生成服务

```typescript
// services/awaySummary.ts
export async function generateAwaySummary(
  messages: readonly Message[],
  signal: AbortSignal,
): Promise<string | null> {
  // 使用 LLM 生成摘要
  // 返回 null 如果生成失败或被取消
}
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 | 类型 |
|------|------|------|
| `react` | Hooks API | npm 包 |
| `bun:bundle` | 编译时功能标志 | Bun 内置 |

### 5.2 终端焦点系统

| 函数/类型 | 来源 | 用途 |
|-----------|------|------|
| `getTerminalFocusState()` | `terminal-focus-state.js` | 获取当前焦点状态 |
| `subscribeTerminalFocus()` | `terminal-focus-state.js` | 订阅焦点变化 |
| `'blurred' \| 'focused' \| 'unknown'` | - | 焦点状态类型 |

### 5.3 功能开关

| 开关 | 类型 | 默认值 | 用途 |
|------|------|--------|------|
| `feature('AWAY_SUMMARY')` | 编译时 | - | 代码包含控制 |
| `tengu_sedge_lantern` | 运行时 | `false` | 3P 用户默认关闭 |

### 5.4 消息工具

| 函数 | 来源 | 用途 |
|------|------|------|
| `createAwaySummaryMessage(text)` | `utils/messages.js` | 创建摘要消息对象 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 缓解 |
|------|------|------|
| 定时器泄漏 | 组件卸载时未清理 | cleanup 函数清理所有资源 |
| 竞态条件 | 快速焦点切换 | `pendingRef` 和 `abortRef` 协调 |
| 重复生成 | 边界情况下多次触发 | `hasSummarySinceLastUserTurn` 检查 |
| 不支持终端 | 终端不支持 DECSET 1004 | 'unknown' 状态无操作 |

### 6.2 边界条件

1. **快速切换**：离开后立即返回，定时器被取消
2. **回合进行中**：`isLoading=true`，摘要延迟到回合结束
3. **回合结束已离开**：第二个 effect 捕获并触发
4. **已存在摘要**：`hasSummarySinceLastUserTurn` 阻止重复
5. **生成失败**：`text === null`，不添加消息
6. **用户取消**：`AbortSignal` 触发，优雅退出

### 6.3 改进建议

1. **可配置延迟**：
   ```typescript
   const BLUR_DELAY_MS = parseInt(
     process.env.CLAUDE_AWAY_SUMMARY_DELAY_MS || '300000',
     10
   )
   ```

2. **摘要长度限制**：
   ```typescript
   const text = await generateAwaySummary(messages, signal, { maxLength: 500 })
   ```

3. **用户偏好**：
   ```typescript
   const isEnabled = useAppState(s => s.preferences.awaySummaryEnabled)
   ```

4. **批量摘要**：
   ```typescript
   // 支持多次离开生成单个综合摘要
   const awayPeriods = detectAwayPeriods(messages)
   ```

5. **交互式摘要**：
   ```typescript
   // 允许用户点击摘要展开详情
   createAwaySummaryMessage(text, { expandable: true })
   ```

### 6.4 代码质量

- **优点**：
  - 完善的资源清理
  - 清晰的防重复逻辑
  - 支持取消和竞态处理
  - 编译时和运行时双重功能控制
  
- **潜在改进**：
  - 提取定时器逻辑为 `useTimeout` Hook
  - 添加 JSDoc 说明 `pendingRef` 的用途
  - 考虑使用状态机替代多个 Ref

### 6.5 相关文档

- `services/awaySummary.js` - 摘要生成服务
- `ink/terminal-focus-state.js` - 终端焦点检测
- `services/analytics/growthbook.js` - 功能开关系统
