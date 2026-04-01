# useLogMessages.ts 深度研究文档

## 场景与职责

`useLogMessages` 是一个用于消息日志记录的 React 钩子，负责将对话消息增量写入到会话的 transcript 文件（JSONL 格式）。它是 Claude Code 会话持久化系统的核心组件之一。

### 核心场景

1. **增量消息记录**：只记录新增消息，避免重复写入
2. **会话压缩感知**：检测会话压缩（compaction）事件，正确处理消息链
3. **多智能体支持**：为 Agent Swarm 记录团队名称和代理名称
4. **性能优化**：避免 O(n) 扫描，使用指针跟踪记录位置

### 与其他组件的关系

- 被 `REPL.tsx` 使用，监听消息变化
- 与 `sessionStorage.ts` 紧密集成，实际写入操作由后者处理
- 支持 Agent Swarm 的 transcript 关联

---

## 功能点目的

### 1. 增量记录

- 使用 `lastRecordedLengthRef` 跟踪上次记录的消息数量
- 只将新增的消息（tail）传递给 `recordTranscript`
- 避免每次渲染都扫描全部消息

### 2. 压缩检测与处理

识别三种状态变化：
- **首次渲染**：`firstMessageUuidRef` 为 undefined
- **增量更新**：第一条消息 UUID 不变，长度增加
- **压缩事件**：第一条消息 UUID 改变（会话被重建）
- **同头收缩**：第一条消息 UUID 不变，长度减少（tombstone 过滤）

### 3. 父消息指针管理

- 使用 `lastParentUuidRef` 跟踪链中最后一条消息
- 为增量写入提供 `parentHint`，确保消息链正确
- 通过 `cleanMessagesForLogging` 和 `isChainParticipant` 确定链参与者

### 4. 并发安全

- 使用 `callSeqRef` 序列号防止竞态条件
- 异步写入完成后检查序列号，避免旧写入覆盖新状态

---

## 具体技术实现

### 关键数据结构

```typescript
// Refs 用于跟踪记录状态
const lastRecordedLengthRef = useRef(0)           // 上次记录的消息数
const lastParentUuidRef = useRef<UUID | undefined>(undefined)  // 最后父消息 UUID
const firstMessageUuidRef = useRef<UUID | undefined>(undefined) // 首消息 UUID
const callSeqRef = useRef(0)                      // 调用序列号
```

### 核心算法

#### 状态检测逻辑

```typescript
const currentFirstUuid = messages[0]?.uuid as UUID | undefined
const prevLength = lastRecordedLengthRef.current

// 检测各种状态
const wasFirstRender = firstMessageUuidRef.current === undefined
const isIncremental = 
  currentFirstUuid !== undefined &&
  !wasFirstRender &&
  currentFirstUuid === firstMessageUuidRef.current &&
  prevLength <= messages.length

const isSameHeadShrink = 
  currentFirstUuid !== undefined &&
  !wasFirstRender &&
  currentFirstUuid === firstMessageUuidRef.current &&
  prevLength > messages.length

// 计算起始索引
const startIndex = isIncremental ? prevLength : 0
```

#### 消息切片与记录

```typescript
// 获取需要记录的消息切片
const slice = startIndex === 0 ? messages : messages.slice(startIndex)
const parentHint = isIncremental ? lastParentUuidRef.current : undefined

// 异步记录
const seq = ++callSeqRef.current
void recordTranscript(
  slice,
  isAgentSwarmsEnabled() 
    ? { teamName: teamContext?.teamName, agentName: teamContext?.selfAgentName }
    : {},
  parentHint,
  messages
).then(lastRecordedUuid => {
  // 防止竞态：检查序列号
  if (seq !== callSeqRef.current) return
  if (lastRecordedUuid && !isIncremental) {
    lastParentUuidRef.current = lastRecordedUuid
  }
})
```

#### 同步父指针更新

```typescript
if (isIncremental || wasFirstRender || isSameHeadShrink) {
  // 使用 cleanMessagesForLogging 过滤，确保与 recordTranscript 一致
  const last = cleanMessagesForLogging(slice, messages).findLast(isChainParticipant)
  if (last) lastParentUuidRef.current = last.uuid as UUID
}
```

### 依赖函数

#### `recordTranscript`

位于 `src/utils/sessionStorage.ts`，负责：
- 将消息写入 JSONL 文件
- 处理 `messagesToKeep`（压缩时保留的消息）
- 返回最后记录的 UUID

#### `cleanMessagesForLogging`

过滤消息，移除：
- 不可记录的消息类型
- REPL 工具调用（外部用户）
- 将虚拟消息提升为正式消息

#### `isChainParticipant`

判断消息是否参与 parentUuid 链：
```typescript
export function isChainParticipant(m: Pick<Message, 'type'>): boolean {
  return m.type !== 'progress'
}
```

---

## 关键代码路径与文件引用

```
src/hooks/useLogMessages.ts
├── useLogMessages()               # 行 19-119: 主钩子
│   ├── Ref 初始化                 # 行 25-32
│   ├── useEffect                  # 行 34-118
│   │   ├── 状态检测               # 行 37-57
│   │   ├── 切片计算               # 行 59-64
│   │   ├── recordTranscript 调用  # 行 68-89
│   │   └── 同步父指针更新         # 行 91-114
│   └── 依赖数组                   # 行 116-118
```

### 依赖文件

```
src/utils/sessionStorage.ts
├── recordTranscript()             # 行 1000+: 核心记录函数
├── cleanMessagesForLogging()      # 消息清理
├── isChainParticipant()           # 链参与者判断
└── Project 类                     # 文件写入管理

src/utils/agentSwarmsEnabled.ts
└── isAgentSwarmsEnabled()         # Agent Swarm 功能开关

src/state/AppState.ts
└── teamContext                    # 团队上下文
```

---

## 依赖与外部交互

### React Hooks 使用

- `useRef`: 跟踪记录状态（避免重渲染）
- `useEffect`: 监听消息变化，触发记录
- `useAppState`: 获取 teamContext

### 与 AppState 的交互

```typescript
const teamContext = useAppState(s => s.teamContext)
```

### 与 sessionStorage 的交互

```typescript
import { recordTranscript, cleanMessagesForLogging, isChainParticipant } from '../utils/sessionStorage.js'

// Fire and forget - 不阻塞 UI
void recordTranscript(slice, teamInfo, parentHint, messages)
```

---

## 风险、边界与改进建议

### 已知风险

1. **竞态条件**
   - 压缩事件和增量渲染可能同时发生
   - 缓解：`callSeqRef` 序列号检查，但仍有极小概率问题

2. **消息丢失**
   - 如果 `recordTranscript` 失败，没有重试机制
   - 缓解：写入队列机制在 sessionStorage.ts 中处理

3. **性能问题**
   - `cleanMessagesForLogging` 每次都要扫描切片
   - 大切片时可能影响性能

### 边界情况

| 场景 | 行为 |
|-----|------|
| ignore=true | 跳过记录 |
| messages 为空 | 直接返回 |
| startIndex === messages.length | 无新消息，直接返回 |
| 压缩后首次写入 | 全量写入，使用 async 返回的 parentUuid |
| 同头收缩 | 同步更新 parentUuid，基于 survivors |

### 改进建议

1. **重试机制**
   - 添加写入失败重试
   - 记录失败时告警用户

2. **批量优化**
   - 高频消息更新时批量写入
   - 减少文件系统操作

3. **更智能的压缩检测**
   - 使用更可靠的压缩事件通知
   - 替代 UUID 比较

4. **遥测集成**
   - 记录写入延迟指标
   - 监控失败率

5. **配置化**
   - 允许用户配置日志级别
   - 支持选择性记录某些消息类型

### 测试建议

1. **单元测试**：
   - 各种状态检测逻辑
   - 竞态条件模拟

2. **集成测试**：
   - 与 sessionStorage 的集成
   - 压缩事件处理

3. **性能测试**：
   - 大量消息时的写入性能
   - 内存使用监控
