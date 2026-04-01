# useMailboxBridge.ts 深度研究文档

## 场景与职责

`useMailboxBridge` 是一个轻量级的消息桥接钩子，用于连接 React 组件与底层的 `Mailbox` 系统。它提供了一种响应式的方式来监听邮箱消息变化并自动提交消息。

### 核心场景

1. **消息订阅**：使用 `useSyncExternalStore` 订阅邮箱状态变化
2. **自动提交**：当有新消息且不在加载状态时自动提交
3. **轻量级集成**：为组件提供简单的邮箱接入方式

### 与其他组件的关系

- 被 `REPL.tsx` 使用，连接主输入循环与邮箱系统
- 依赖 `MailboxContext` 提供的邮箱实例
- 与 `useInboxPoller` 形成互补：后者处理多智能体消息，前者处理通用消息

---

## 功能点目的

### 1. 外部存储同步

使用 React 的 `useSyncExternalStore` API：
- `subscribe`: 绑定邮箱的订阅方法
- `getSnapshot`: 获取邮箱的 revision（版本号）
- 当 revision 变化时触发重渲染

### 2. 消息轮询与提交

- 监听 `isLoading` 状态，避免在加载时提交
- 调用 `mailbox.poll()` 获取新消息
- 通过 `onSubmitMessage` 回调提交消息内容

### 3. 轻量级设计

- 仅 21 行代码，职责单一
- 不处理消息分类或权限逻辑
- 简单的订阅-轮询-提交模式

---

## 具体技术实现

### 关键数据结构

```typescript
type Props = {
  isLoading: boolean                    // 是否正在处理请求
  onSubmitMessage: (content: string) => boolean  // 消息提交回调
}
```

### 核心实现

```typescript
export function useMailboxBridge({ isLoading, onSubmitMessage }: Props): void {
  const mailbox = useMailbox()
  
  // 创建稳定的订阅函数
  const subscribe = useMemo(() => mailbox.subscribe.bind(mailbox), [mailbox])
  
  // 创建快照获取函数
  const getSnapshot = useCallback(() => mailbox.revision, [mailbox])
  
  // 使用 React 的外部存储同步 API
  const revision = useSyncExternalStore(subscribe, getSnapshot)
  
  // 当 revision 变化且不在加载时，轮询并提交消息
  useEffect(() => {
    if (isLoading) return
    const msg = mailbox.poll()
    if (msg) onSubmitMessage(msg.content)
  }, [isLoading, revision, mailbox, onSubmitMessage])
}
```

### Mailbox 接口

```typescript
class Mailbox {
  revision: number = 0                    // 版本号，每次更新递增
  subscribe(callback: () => void): () => void  // 订阅变化
  poll(): { content: string } | null      // 获取并移除最新消息
  post(content: string): void             // 发布消息（内部使用）
}
```

---

## 关键代码路径与文件引用

```
src/hooks/useMailboxBridge.ts
├── Props 类型定义                 # 行 4-7
├── useMailboxBridge()             # 行 9-21: 主钩子
│   ├── useMailbox()               # 行 10: 获取邮箱实例
│   ├── useMemo - subscribe        # 行 12: 绑定订阅方法
│   ├── useCallback - getSnapshot  # 行 13: 快照获取
│   ├── useSyncExternalStore       # 行 14: 外部存储同步
│   └── useEffect - 轮询提交       # 行 16-20: 消息处理
```

### 依赖文件

```
src/context/mailbox.tsx
├── MailboxContext                 # React Context
├── MailboxProvider                # Provider 组件
└── useMailbox()                   # 邮箱实例 hook

src/utils/mailbox.ts
└── Mailbox 类                     # 邮箱实现
```

---

## 依赖与外部交互

### React Hooks 使用

- `useMailbox`: 从 Context 获取邮箱实例
- `useMemo`: 缓存订阅函数
- `useCallback`: 缓存快照获取函数
- `useSyncExternalStore`: 同步外部存储状态
- `useEffect`: 处理消息轮询和提交

### 与 Mailbox 的交互

```typescript
const mailbox = useMailbox()

// 订阅变化
const subscribe = useMemo(() => mailbox.subscribe.bind(mailbox), [mailbox])

// 获取版本快照
const getSnapshot = useCallback(() => mailbox.revision, [mailbox])

// 同步状态
const revision = useSyncExternalStore(subscribe, getSnapshot)

// 轮询消息
const msg = mailbox.poll()
```

### 与父组件的交互

```typescript
// 通过 props 接收提交回调
onSubmitMessage: (content: string) => boolean

// 提交成功返回 true，失败返回 false
if (msg) onSubmitMessage(msg.content)
```

---

## 风险、边界与改进建议

### 已知风险

1. **消息丢失**
   - `poll()` 会移除消息，如果提交失败消息丢失
   - 缓解：`onSubmitMessage` 返回 boolean，但失败时无重试

2. **竞态条件**
   - 快速连续的消息可能导致部分丢失
   - 缓解：Mailbox 内部应该有队列，但此钩子只处理单条

3. **加载状态依赖**
   - 如果在加载期间有新消息，会被跳过
   - 缓解：Mailbox 应该保留未处理的消息

### 边界情况

| 场景 | 行为 |
|-----|------|
| isLoading=true | 跳过提交，消息保留在邮箱 |
| mailbox.poll()=null | 无操作 |
| onSubmitMessage 返回 false | 消息已移除但提交失败 |
| 组件卸载 | 订阅自动清理 |

### 改进建议

1. **消息队列**
   - 支持批量处理多条消息
   - 提交失败时重新入队

2. **错误处理**
   - 添加提交失败的回调
   - 支持重试机制

3. **优先级**
   - 支持消息优先级
   - 高优先级消息可中断当前加载

4. **遥测**
   - 记录消息处理延迟
   - 监控丢失率

### 测试建议

1. **单元测试**：
   - 订阅/取消订阅
   - 消息轮询流程
   - 加载状态处理

2. **集成测试**：
   - 与 Mailbox 类集成
   - 与 REPL 组件集成

3. **边界测试**：
   - 快速连续消息
   - 组件卸载时
