# QueryGuard.ts 深度研究文档

## 1. 场景与职责

`QueryGuard` 是 REPL（`src/screens/REPL.tsx`）中负责**查询生命周期同步状态管理**的核心类。它的唯一职责是：以**同步、原子、可外部订阅**的方式，追踪“当前是否有一条用户输入（query）正在被处理或即将被处理”。

在引入 `QueryGuard` 之前，REPL 使用了一个极易出错的“双状态”模式：
- `isLoading`：由 React `useState` 管理的异步状态，控制 spinner 等 UI 的显示；
- `isQueryRunning`：一个 `useRef` 布尔值，用于在回调中同步判断是否有查询在跑。

这两个状态分别由不同的代码路径更新，且 React 的批量更新（batching）会导致 `isLoading` 的变更滞后于 `isQueryRunning`。结果是：用户在查询尚未真正结束时再次提交输入，或者队列处理器（queue processor）在异步间隙中错误地 dequeue 了新任务，造成竞态、重复提交或状态不一致。

`QueryGuard` 的设计目标就是**消除这种双状态分裂**，用一个类内的同步状态机作为“单一真相源”（single source of truth），并通过 `useSyncExternalStore` 将其同步暴露给 React 组件树。

---

## 2. 功能点目的

### 2.1 三态状态机：idle → dispatching → running

`QueryGuard` 内部维护三种状态（`_status` 字段，见 `src/utils/QueryGuard.ts:17`）：

| 状态 | 含义 | `isActive` / `getSnapshot` |
|------|------|---------------------------|
| `idle` | 无查询在进行，可以安全 dequeue 并处理新输入 | `false` |
| `dispatching` | 已从队列中取出（或直接从 onSubmit 进入），但异步链尚未到达 `onQuery` 的 `tryStart()` | `true` |
| `running` | `onQuery` 已调用 `tryStart()`，真正的 API 查询正在执行 | `true` |

**为什么需要 `dispatching` 状态？**

在队列处理路径中，流程是：
1. `useQueueProcessor` 的 effect 发现队列非空且 `!isQueryActive`，于是调用 `processQueueIfReady`；
2. `processQueueIfReady` dequeue 一个命令并调用 `executeQueuedInput`（即 `handlePromptSubmit`）；
3. `handlePromptSubmit` 同步进入 `executeUserInput`，后者在第一个 `await` **之前**调用 `queryGuard.reserve()`；
4. 从 `reserve()` 到 `onQuery` 内部的 `tryStart()` 之间，存在一段异步间隙（`processUserInput` 可能涉及 await `BashTool.call()`、await `getMessagesForSlashCommand` 等）。

如果没有 `dispatching`，在这段间隙里状态仍然是 `idle`，那么：
- 用户快速连击回车，`handlePromptSubmit` 的 `queryGuard.isActive` 检查（`src/utils/handlePromptSubmit.ts:313`）会返回 `false`，导致重复启动新查询；
- `useQueueProcessor` 的 effect 可能在同一 tick 再次触发，再次 dequeue 并启动下一个命令。

`dispatching` 状态就是用来**覆盖这段异步间隙**，确保从“决定开始处理输入”到“真正进入 API 调用”之间的整个窗口内，`isActive` 都为 `true`，从而阻止重入。

### 2.2 世代计数器（generation counter）

`QueryGuard` 维护一个私有字段 `_generation`（`src/utils/QueryGuard.ts:18`），并在以下时机变化：
- `tryStart()` 成功时：`++this._generation`（`src/utils/QueryGuard.ts:33`）；
- `forceEnd()` 时：`++this._generation`（`src/utils/QueryGuard.ts:46`）。

`tryStart()` 返回当前的 generation 编号，`end(generation)` 则要求传入的 generation 与当前 `_generation` 严格匹配，否则拒绝状态转换（`src/utils/QueryGuard.ts:37-42`）。

**目的：防止过时的 `finally` 块错误地清理当前正在运行的查询。**

考虑以下场景：
1. 查询 A 启动，`tryStart()` 返回 `generation = 1`；
2. 用户在查询 A 执行过程中按下 `Ctrl+C`（或远程发送中断），REPL 调用 `queryGuard.forceEnd()`，状态切回 `idle`，`generation` 提升到 `2`；
3. 查询 A 的 `onQuery` 的 `finally` 块稍后执行，试图调用 `queryGuard.end(1)`；
4. 由于 `1 !== 2`，`end()` 返回 `false`，不会把当前已经是 `idle`（或可能已被新查询 B 设为 `running` / `generation = 3`）的状态错误地覆盖。

这本质上是一种**乐观并发控制**：用单调递增的 generation 作为 token，确保只有“真正拥有当前运行权”的那一轮查询才能合法地标记自己结束。

### 2.3 与 `useSyncExternalStore` 的集成

`QueryGuard` 提供了两个用于 React 的 API（`src/utils/QueryGuard.ts:54-57`）：

```typescript
subscribe = this._changed.subscribe
getSnapshot = (): boolean => this._status !== 'idle'
```

在 `REPL.tsx` 中（`src/screens/REPL.tsx:904`）：

```typescript
const isQueryActive = React.useSyncExternalStore(queryGuard.subscribe, queryGuard.getSnapshot)
```

以及在 `useQueueProcessor.ts` 中（`src/hooks/useQueueProcessor.ts:35-38`）也做了同样的订阅。

**为什么选择 `useSyncExternalStore` 而不是普通的 `useState`？**

1. **同步读取**：`useSyncExternalStore` 保证在渲染过程中读取的是**最新快照**，不会因为 React 的并发特性或批量更新而读到过期值。对于防止竞态（“我能不能现在 dequeue？”）来说，这一点至关重要。
2. **避免 setter 扩散**：如果使用 `useState`，则需要在 `QueryGuard` 外部持有 `setState`，导致状态变更逻辑分散到多个文件，重新引入双状态问题。`useSyncExternalStore` 让 `QueryGuard` 自己管理变更通知，REPL 只负责消费。
3. **与 Ink 的兼容性**：在终端 UI 框架 Ink 中，Context 的传播有时会出现延迟或通知丢失（`src/hooks/useQueueProcessor.ts:41-42` 注释明确提到这一点）。`useSyncExternalStore` 绕过 Context，直接订阅外部 store，消除了这类问题。

---

## 3. 具体技术实现

### 3.1 关键数据结构

```typescript
// src/utils/QueryGuard.ts:17-18
private _status: 'idle' | 'dispatching' | 'running' = 'idle'
private _generation = 0
private _changed = createSignal()
```

- `_status`：三态状态机的核心；
- `_generation`：64 位整数（JavaScript number），单调递增；
- `_changed`：基于 `src/utils/signal.ts` 的轻量级事件信号，用于通知所有订阅者状态已变。

### 3.2 状态转换协议

| 方法 | 触发条件 | 转换 | 副作用 |
|------|---------|------|--------|
| `reserve()` | `executeUserInput` 的 try 块开头（`src/utils/handlePromptSubmit.ts:437`） | `idle → dispatching` | 若原状态非 `idle` 则拒绝（返回 `false`） |
| `cancelReservation()` | `executeUserInput` 的 finally 块（`src/utils/handlePromptSubmit.ts:603`），或本地 slash 命令跳过查询时（`src/utils/handlePromptSubmit.ts:578`） | `dispatching → idle` | 若原状态非 `dispatching` 则 no-op |
| `tryStart()` | `onQuery` 内部，真正开始 API 调用前 | `idle → running` 或 `dispatching → running` | `++_generation`，返回新 generation |
| `end(generation)` | `onQuery` 的 finally 块，查询正常结束 | `running → idle` | generation 必须匹配 |
| `forceEnd()` | 用户中断（Ctrl+C）或外部强制终止 | `dispatching/running → idle` | `++_generation`，无条件覆盖 |

### 3.3 关键流程：直接用户提交 vs 队列处理器

#### 直接用户提交路径
1. 用户在 `PromptInput` 中按回车 → `REPL.tsx` 的 `onSubmit`；
2. `onSubmit` 调用 `handlePromptSubmit`；
3. `handlePromptSubmit` 检查 `queryGuard.isActive`（`src/utils/handlePromptSubmit.ts:313`）。如果为 `true`，说明有查询在跑，将输入 enqueue；
4. 如果为 `false`，直接调用 `executeUserInput`；
5. `executeUserInput` 的 try 块中调用 `queryGuard.reserve()`，进入 `dispatching`；
6. 经过 `processUserInput` 的异步处理后，进入 `onQuery`；
7. `onQuery` 开头调用 `queryGuard.tryStart()`，进入 `running` 并拿到 generation；
8. 查询结束（成功、失败、中断），`onQuery` 的 finally 调用 `queryGuard.end(gen)`，回到 `idle`。

#### 队列处理器路径
1. `useQueueProcessor` 的 effect 订阅了 `queryGuard` 和命令队列；
2. 当 `isQueryActive` 为 `false`、队列非空、且无本地 JSX UI 阻塞时，调用 `processQueueIfReady`（`src/hooks/useQueueProcessor.ts:60`）；
3. `processQueueIfReady` dequeue 命令并调用 `executeQueuedInput`（即 `handlePromptSubmit` 的包装）；
4. 从 `executeQueuedInput` → `handlePromptSubmit` → `executeUserInput` → `queryGuard.reserve()` 的调用链**在第一个真正的 await 之前全部是同步的**；
5. 因此，当 React 由于队列变化重新调度 `useQueueProcessor` 的 effect 时，`isQueryActive` 已经因为 `reserve()` 而变为 `true`，effect 提前返回，不会重复触发（`src/hooks/useQueueProcessor.ts:49`）。

### 3.3.1 关于 `cancelReservation` 的安全网作用

`executeUserInput` 的 `finally` 块（`src/utils/handlePromptSubmit.ts:597-609`）包含：

```typescript
queryGuard.cancelReservation()
setUserInputOnProcessing(undefined)
```

这段代码是一个**安全网**：
- 如果 `onQuery` 正常执行完毕，它自己的 `finally` 会调用 `queryGuard.end(gen)`，状态已回到 `idle`，此时 `cancelReservation()` 发现状态不是 `dispatching`，直接 no-op；
- 如果 `processUserInput` 抛出异常，或者某些本地 slash 命令导致 `newMessages.length === 0` 从而跳过了 `onQuery`，那么状态可能停留在 `dispatching`，`finally` 中的 `cancelReservation()` 就会把它正确释放回 `idle`。

这意味着**释放逻辑是单点且幂等的**，不再需要 `useQueueProcessor` 自己维护 `.finally()`。

---

## 4. 关键代码路径与文件引用

### 4.1 QueryGuard 本体

**文件**：`src/utils/QueryGuard.ts`

| 行号 | 内容 |
|------|------|
| 17-18 | 私有状态字段 `_status`、`_generation`、`_changed` 声明 |
| 20-25 | `reserve()`：idle → dispatching |
| 27-31 | `cancelReservation()`：dispatching → idle |
| 33-40 | `tryStart()`：dispatching/idle → running，generation 自增 |
| 42-48 | `end(generation)`：running → idle，带 generation 校验 |
| 50-54 | `forceEnd()`：强制回到 idle，generation 自增 |
| 56-63 | `isActive` getter、`generation` getter、`subscribe`、`getSnapshot` |
| 65-67 | `_notify()`：内部调用 `this._changed.emit()` |

### 4.2 REPL 中的使用

**文件**：`src/screens/REPL.tsx`

| 行号 | 内容 |
|------|------|
| 35 | `import { QueryGuard } from '../utils/QueryGuard.js'` |
| 897-900 | 注释说明：QueryGuard 取代了容易出错的双状态模式 |
| 900 | `const queryGuard = React.useRef(new QueryGuard()).current` |
| 904 | `const isQueryActive = React.useSyncExternalStore(queryGuard.subscribe, queryGuard.getSnapshot)` |
| 911-916 | `isLoading = isQueryActive \|\| isExternalLoading`：本地查询状态与外部加载状态合并 |
| 950-953 | 利用 `isQueryActive` 的 false→true 跳变，在渲染期间同步重置计时 ref，修复 INC-4549 |

### 4.3 队列处理器中的使用

**文件**：`src/hooks/useQueueProcessor.ts`

| 行号 | 内容 |
|------|------|
| 7 | `import type { QueryGuard } from '../utils/QueryGuard.js'` |
| 35-38 | `useSyncExternalStore(queryGuard.subscribe, queryGuard.getSnapshot)` |
| 48-60 | effect 逻辑：若 `isQueryActive` 为 true 则提前返回，否则调用 `processQueueIfReady` |
| 53-59 | 注释详细解释了 reservation 现在由 `handlePromptSubmit` 拥有，同步链保证在 React 重新运行 effect 前 `isQueryActive` 已为 true |

### 4.4 提交处理中的使用

**文件**：`src/utils/handlePromptSubmit.ts`

| 行号 | 内容 |
|------|------|
| 31 | `import type { QueryGuard } from './QueryGuard.js'` |
| 251 | 本地 JSX 立即命令执行前检查 `queryGuard.isActive` |
| 313 | 判断当前是否活跃，决定是否 enqueue 或直接执行 |
| 437 | `executeUserInput` try 块中调用 `queryGuard.reserve()` |
| 578 | 本地 slash 命令无消息产出时，主动 `queryGuard.cancelReservation()` 避免 spinner 闪烁 |
| 603 | `finally` 块中安全网式 `queryGuard.cancelReservation()` |

### 4.5 底层信号依赖

**文件**：`src/utils/signal.ts`

| 行号 | 内容 |
|------|------|
| 18-25 | `Signal<Args>` 类型定义：subscribe / emit / clear |
| 27-42 | `createSignal()` 工厂函数，基于 `Set<listener>` 实现 |

---

## 5. 依赖与外部交互

### 5.1 内部依赖

- **`src/utils/signal.ts`**：`QueryGuard` 唯一的内部运行时依赖。`createSignal` 提供了一个零状态、纯通知机制的事件原语，与 `QueryGuard` 的“自身持有状态、外部只读快照”模型完美契合。

### 5.2 外部交互方

| 交互方 | 角色 |
|--------|------|
| `REPL.tsx` | 创建并持有 `QueryGuard` 实例；通过 `useSyncExternalStore` 订阅其快照；将 `isQueryActive` 与 `isExternalLoading` 合并为 `isLoading`；在渲染期间利用 `isQueryActive` 跳变同步重置计时器 ref |
| `useQueueProcessor.ts` | 订阅 `QueryGuard` 以决定是否可以安全 dequeue；依赖 `isQueryActive` 的同步性防止竞态 dequeue |
| `handlePromptSubmit.ts` | 直接操作 `QueryGuard` 的方法（`reserve`、`cancelReservation`、`isActive`）；是状态转换的主要触发源 |
| `queueProcessor.ts` | 不直接引用 `QueryGuard`，但其 `processQueueIfReady` 的调用时机完全由 `useQueueProcessor` 根据 `QueryGuard` 的状态控制 |

### 5.3 与 React 并发特性的关系

`useSyncExternalStore` 是 React 18 为“外部同步 store”设计的官方 API。它在以下方面优于手动 `useState` + `useEffect`：
- ** tearing 防护**：在并发渲染中，如果组件被中断并恢复，`useSyncExternalStore` 会确保读取到的快照始终一致；
- **服务端渲染支持**：可以提供 `getServerSnapshot`（虽然 REPL 是客户端 CLI 应用，暂不涉及 SSR）；
- **同步刷新**：store 变更会同步触发订阅组件的重新渲染，不会像 `setState` 那样被批量延迟。

对于 Ink（基于 React 的终端渲染框架）来说，同步刷新尤为重要，因为终端 UI 对输入响应的延迟非常敏感，任何批量更新延迟都会导致用户感知到的卡顿或状态跳跃。

---

## 6. 风险、边界与改进建议

### 6.1 已识别的风险与边界

#### 6.1.1 `forceEnd` 与 `end` 的竞态窗口

虽然 generation 机制已经极大地降低了过时 `finally` 块错误清理的风险，但以下极端场景仍值得注意：
- 查询 A 的 `onQuery` 正在执行，`generation = 1`；
- 用户发送中断，`forceEnd()` 将状态切为 `idle`，`generation = 2`；
- 在查询 A 的 `finally` 块执行 `end(1)` 之前，队列处理器或用户立即提交了查询 B；
- 查询 B 调用 `tryStart()`，状态变为 `running`，`generation = 3`；
- 查询 A 的 `finally` 块终于执行到 `end(1)`，由于 `1 !== 3`，拒绝转换——**这是正确行为**；
- 但如果查询 A 的 `finally` 块不仅调用 `end(1)`，还做了其他副作用（如 `setAbortController(null)`），这些副作用仍可能干扰查询 B。

**缓解**：`handlePromptSubmit.ts` 的 `finally` 块只调用 `cancelReservation()`（对 `running` 状态无影响）和 `setUserInputOnProcessing(undefined)`，后者是 UI 占位符的清理，对新的查询 B 基本无害。真正的 `setAbortController(null)` 位于 `onQuery` 的 finally 中，而 `onQuery` 的 `end(1)` 会失败，但 `setAbortController(null)` 仍会被执行——**这意味着查询 B 的 abort controller 可能被 A 的 finally 覆盖为 null**。

不过，在现有代码中，`onQuery` 的 `finally` 通常如下结构：
```typescript
try { ... } finally {
  queryGuard.end(gen);
  setAbortController(null);
}
```
如果 `end(1)` 因 generation 不匹配而失败，但 `setAbortController(null)` 仍然执行，那么查询 B 的 abort controller 会被意外清空。这是一个**潜在的隐蔽 bug**，虽然当前代码可能通过其他机制（如 `abortControllerRef` 的同步更新）部分缓解，但仍值得警惕。

#### 6.1.2 `dispatching` 状态的无限期滞留

如果 `executeUserInput` 中的 `processUserInput` 抛出了异常，且异常被 `try/finally` 捕获，`cancelReservation()` 会释放状态。但如果异常发生在 `reserve()` 之前（理论上不可能，因为 `reserve()` 是同步调用的第一句话），或者 `onQuery` 的调用者忘记包裹 `try/finally`，`dispatching` 可能永远滞留。

**当前状态**：`executeUserInput` 已经用 `try/finally` 完全包裹，风险极低。

#### 6.1.3 外部加载状态的割裂

`isLoading` 是 `isQueryActive || isExternalLoading` 的或运算（`src/screens/REPL.tsx:916`）。`isExternalLoading` 用于远程会话（`useRemoteSession`、`useDirectConnect`）和前台后台任务（`useSessionBackgrounding`）。这些路径不经过 `QueryGuard`，因此：
- 如果未来某个外部加载路径也想要利用队列阻塞逻辑，它需要单独与 `QueryGuard` 集成；
- `handlePromptSubmit.ts:313` 的检查是 `queryGuard.isActive || isExternalLoading`，这保证了外部加载期间本地输入也会被正确 enqueue 或阻止。

### 6.2 改进建议

#### 6.2.1 将 `setAbortController(null)` 纳入 generation 保护

建议把 `setAbortController` 的清理也绑定到 generation，例如：

```typescript
// 在 REPL.tsx 的 onQuery 中
const startGen = queryGuard.tryStart();
if (startGen == null) return;
try {
  // ... 查询逻辑 ...
} finally {
  if (queryGuard.end(startGen)) {
    setAbortController(null);
  }
}
```

这样可以彻底消除“旧查询 finally 清空新查询 abort controller”的竞态风险。

#### 6.2.2 暴露更细粒度的快照

当前 `getSnapshot` 只返回布尔值（`isActive`）。如果未来 UI 需要区分“正在预处理（dispatching）”和“正在与模型对话（running）”（例如显示不同的 spinner 文案），可以将 `getSnapshot` 改为返回状态字符串或一个枚举值，同时保持向后兼容：

```typescript
getSnapshot = (): 'idle' | 'dispatching' | 'running' => this._status
```

由于 `useSyncExternalStore` 要求快照值支持 `===` 比较，字符串字面量完全满足要求。

#### 6.2.3 增加 dev-only 的不变式断言

在 `reserve`、`tryStart`、`end`、`cancelReservation`、`forceEnd` 的关键状态转换点增加 `console.assert` 或自定义 invariant，可以在开发阶段快速捕获非法转换。例如：

```typescript
reserve(): boolean {
  if (this._status !== 'idle') {
    console.assert(false, `QueryGuard.reserve() called in ${this._status}`);
    return false;
  }
  // ...
}
```

#### 6.2.4 文档化状态机图

虽然代码注释已经相当详尽（`src/utils/QueryGuard.ts` 顶部 20 行注释完整描述了状态与转换），但建议在项目 Wiki 或 README 中补充一张 Mermaid 状态图，方便新成员快速理解 `idle → dispatching → running → idle` 的完整闭环。

---

## 总结

`QueryGuard` 是一个小而精的状态机类，通过**三态同步状态机** + **世代计数器** + **`useSyncExternalStore` 订阅**，彻底解决了 REPL 中 `isLoading` 与 `isQueryRunning` 双状态不同步的历史顽疾。它将查询生命周期的所有权收敛到单一对象，消除了队列处理器、用户提交、中断取消等多条代码路径之间的竞态窗口，是 REPL 状态管理中的关键基础设施。
