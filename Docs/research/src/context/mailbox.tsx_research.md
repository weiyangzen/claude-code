# mailbox.tsx 研究文档

## 场景与职责

`mailbox.tsx` 是 `Mailbox` 类（位于 `src/utils/mailbox.ts`）的 **React 上下文适配层**。它将一个命令式消息邮箱实例挂载到 React 组件树中，使得任何深度的组件都能通过 `useMailbox()` Hook 访问该邮箱，实现：

1. **跨组件异步消息投递**：外部系统（如 LSP、MCP、IDE 桥接）可向邮箱 `send(msg)`，React 组件通过 `useSyncExternalStore` 或轮询 `poll()` 消费消息。
2. **解耦非 React 代码与 UI**：邮箱作为边界对象，让命令式后端逻辑无需持有 React ref 即可与前端交互。

## 功能点目的

### 1. `MailboxProvider`
- 在挂载时通过 `useMemo(() => new Mailbox(), [])` 创建一个**单例邮箱实例**（Provider 生命周期内唯一）。
- 通过 `MailboxContext.Provider` 将实例注入子树。

### 2. `useMailbox`
- 从 Context 读取邮箱实例。
- **严格模式**：若不在 `MailboxProvider` 内调用，直接抛出 `Error("useMailbox must be used within a MailboxProvider")`。
- 返回类型为 `Mailbox`（非 undefined），保证调用方可以安全地调用 `mailbox.send()`、`mailbox.poll()`、`mailbox.receive()`、`mailbox.subscribe()`。

## 具体技术实现

### 依赖的 Mailbox 类（`src/utils/mailbox.ts`）
```ts
export class Mailbox {
  private queue: Message[] = []
  private waiters: Waiter[] = []
  private changed = createSignal()
  private _revision = 0

  send(msg: Message): void
  poll(fn?: (msg) => boolean): Message | undefined
  receive(fn?: (msg) => boolean): Promise<Message>
  subscribe = this.changed.subscribe
  get revision(): number
}
```

- **`send`**：将消息推入队列，或立即匹配并唤醒一个等待中的 `waiter`。
- **`poll`**：同步从队列中移除并返回第一条匹配消息。
- **`receive`**：异步等待匹配消息；若队列中已有匹配项则立即 `Promise.resolve`。
- **`subscribe` / `revision`**：基于自定义 `createSignal()` 实现的最小可订阅 store，兼容 `useSyncExternalStore`。

### 关键流程
1. **Provider 层级**：`AppState.tsx` 在渲染时将 `MailboxProvider` 包裹在 `VoiceProvider` 之外（更靠近根）：
   ```tsx
   <MailboxProvider><VoiceProvider>{children}</VoiceProvider></MailboxProvider>
   ```
   这意味着整个应用共享同一个 Mailbox。

2. **桥接消费**：`src/hooks/useMailboxBridge.ts` 是典型消费者：
   ```ts
   const mailbox = useMailbox()
   const subscribe = useMemo(() => mailbox.subscribe.bind(mailbox), [mailbox])
   const getSnapshot = useCallback(() => mailbox.revision, [mailbox])
   const revision = useSyncExternalStore(subscribe, getSnapshot)
   
   useEffect(() => {
     if (isLoading) return
     const msg = mailbox.poll()
     if (msg) onSubmitMessage(msg.content)
   }, [isLoading, revision, mailbox, onSubmitMessage])
   ```
   该 Hook 将 Mailbox 的 `revision` 变化桥接到 React 的 effect 系统，实现“外部消息 → React 状态/回调”的同步。

## 关键代码路径与文件引用

| 文件 | 角色 |
|------|------|
| `src/context/mailbox.tsx` | 本文件，React Context 封装 |
| `src/utils/mailbox.ts` | 底层 `Mailbox` 类实现 |
| `src/utils/signal.ts` | `createSignal` 实现，为 Mailbox 提供订阅能力 |
| `src/state/AppState.tsx` | 顶层挂载 `MailboxProvider` |
| `src/hooks/useMailboxBridge.ts` | 核心消费者，桥接 Mailbox → PromptInput 提交 |
| `src/screens/REPL.tsx` | 通过 `AppStateProvider` 间接包含 MailboxProvider |

## 依赖与外部交互

- **React**：`createContext`、`useContext`、`useMemo`。
- **`../utils/mailbox.js`**：运行时强依赖，Provider 实例化 `new Mailbox()`。
- **AppState**：`MailboxProvider` 被嵌套在 `AppStateProvider` 内部，但两者无直接数据交互。

## 风险、边界与改进建议

### 风险与边界
1. **单例生命周期与热重载**：`useMemo(() => new Mailbox(), [])` 在 React Fast Refresh 或 Strict Mode 双挂载时，理论上可能创建两个实例（虽然 `useMemo` 在严格模式下不保证只执行一次）。当前实现未处理邮箱状态在重挂载时的迁移，热重载后可能丢失队列中的未消费消息。
2. **`useMailbox` 的强约束**：抛出错误的严格检查在测试环境中是优点，但在某些动态渲染场景（如 Storybook 或单元测试）中，要求测试必须包裹 `MailboxProvider`，增加了测试样板代码。
3. **消息类型无版本控制**：`Mailbox` 的 `Message` 类型是简单的 `{ id, source, content, ... }`，若发送方与消费方对字段语义理解不一致，可能导致静默错误。

### 改进建议
1. **增加 Provider 的 `mailbox` prop**：允许外部传入预创建的 Mailbox 实例，便于测试注入和状态恢复。
   ```tsx
   type Props = { mailbox?: Mailbox; children: React.ReactNode }
   const mailbox = externalMailbox ?? useMemo(() => new Mailbox(), [])
   ```
2. **消息 Schema 校验**：在 `send` 或 `poll` 入口增加轻量级的 `zod` / 结构类型断言，防止跨模块消息格式漂移。
3. **暴露 `MailboxContext` 本身**：目前仅导出 `useMailbox`，某些高阶组件或测试工具可能需要直接消费 Context。建议同时导出 `MailboxContext`。
4. **考虑与 AppState 的 notifications 合并评估**：Mailbox 和 AppState 的 notification 队列在语义上有重叠（都是异步消息投递），可定期审视两者边界，避免功能重复。
