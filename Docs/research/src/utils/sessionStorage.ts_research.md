# sessionStorage.ts 深度研究文档

> 研究对象: `src/utils/sessionStorage.ts`  
> 文件大小: ~5100 行  
> 研究日期: 2026-04-01  
> 执行器: kimi (k2p5)

---

## 1. 场景与职责

### 1.1 核心定位

`sessionStorage.ts` 是 Claude Code CLI 的**会话持久化核心模块**，负责将用户与 AI 的对话历史（transcript）以 JSONL 格式持久化到本地文件系统，并提供会话恢复、检索、元数据管理等功能。

### 1.2 主要使用场景

| 场景 | 说明 |
|------|------|
| **实时对话记录** | 通过 `useLogMessages` Hook 监听消息变化，增量写入 transcript |
| **会话恢复 (/resume)** | 从 JSONL 文件重建对话链，支持 compact boundary 跳过历史 |
| **子代理会话** | 为 AgentTool 创建的子代理维护独立的 sidechain transcript |
| **远程会话同步** | 通过 Session Ingress API 将日志同步到云端 (CCR v1/v2) |
| **会话元数据管理** | 自定义标题、标签、PR 链接、worktree 状态等 |
| **工具结果持久化** | 大工具结果写入磁盘，transcript 中只保留引用 |

### 1.3 架构位置

```
UI Layer (REPL.tsx)
    ↓
Hooks (useLogMessages.ts) ────────┐
    ↓                              │
QueryEngine.ts ───────────────────┼──→ sessionStorage.ts → 文件系统
    ↓                              │         ↓
AgentTool/runAgent.ts ────────────┘    Session Ingress API
    ↓                                   (远程同步)
sessionRestore.ts
(恢复逻辑)
```

---

## 2. 功能点目的

### 2.1 核心功能模块

#### 2.1.1 消息持久化 (`recordTranscript` / `recordSidechainTranscript`)

- **目的**: 将内存中的消息数组持久化到 JSONL 文件
- **关键特性**:
  - 去重: 通过 `getSessionMessages` memoized 缓存避免重复写入
  - parentUuid 链: 维护消息父子关系用于重建对话链
  - 增量写入: `useLogMessages` 只传递新增消息切片
  - Sidechain 支持: 子代理消息写入独立文件

#### 2.1.2 会话文件加载 (`loadTranscriptFile`)

- **目的**: 从 JSONL 文件重建完整的对话状态
- **关键特性**:
  - 大文件优化: >5MB 时启用预压缩跳过 (pre-compact skip)
  - 死分支过滤: `walkChainBeforeParse` 在解析前过滤无效分支
  - 元数据恢复: 同时加载 summary、customTitle、tag、agent 设置等
  - 保留段处理: `applyPreservedSegmentRelinks` 处理 compact 保留的消息段

#### 2.1.3 元数据管理

| 功能 | 入口函数 | 存储方式 |
|------|----------|----------|
| 自定义标题 | `saveCustomTitle` | `custom-title` entry |
| AI 生成标题 | `saveAiGeneratedTitle` | `ai-title` entry |
| 标签 | `saveTag` | `tag` entry |
| 代理名称/颜色 | `saveAgentName` / `saveAgentColor` | `agent-name` / `agent-color` entry |
| PR 链接 | `linkSessionToPR` | `pr-link` entry |
| Worktree 状态 | `saveWorktreeState` | `worktree-state` entry |

#### 2.1.4 会话恢复支持

- **`loadConversationForResume`**: 加载会话用于 `--resume` / `--continue`
- **`hydrateRemoteSession`**: 从远程 Session Ingress 拉取日志
- **`hydrateFromCCRv2InternalEvents`**: CCR v2 内部事件恢复

### 2.2 性能优化功能

#### 2.2.1 Lite 加载模式

```typescript
// 只读取文件头尾 64KB，避免大文件全量读取
readLiteMetadata(filePath, fileSize, buf)
```

用于 `/resume` 列表展示，只提取 firstPrompt、customTitle 等元数据。

#### 2.2.2 渐进式加载

```typescript
enrichLogs(allLogs, startIndex, count)
```

先显示 lite 日志列表，用户滚动时再按需加载完整消息。

#### 2.2.3 写入队列与批量刷新

```typescript
// Project 类内部
writeQueues: Map<string, Array<{ entry: Entry; resolve: () => void }>>
flushTimer: 100ms 延迟批量写入
MAX_CHUNK_BYTES: 100MB 分块
```

---

## 3. 具体技术实现

### 3.1 数据结构与协议

#### 3.1.1 Entry 类型系统 (基于 `src/types/logs.ts`)

```typescript
type Entry =
  | TranscriptMessage    // user/assistant/attachment/system 消息
  | SummaryMessage       // compact summary
  | CustomTitleMessage   // 用户自定义标题
  | AiTitleMessage       // AI 生成标题
  | TagMessage           // 会话标签
  | AgentNameMessage     // 代理名称
  | AgentColorMessage    // 代理颜色
  | AgentSettingMessage  // 代理设置
  | PRLinkMessage        // PR 链接
  | FileHistorySnapshotMessage
  | AttributionSnapshotMessage
  | QueueOperationMessage
  | SpeculationAcceptMessage
  | ModeEntry            // coordinator/normal 模式
  | WorktreeStateEntry
  | ContentReplacementEntry  // 工具结果替换记录
  | ContextCollapseCommitEntry
  | ContextCollapseSnapshotEntry
```

#### 3.1.2 TranscriptMessage 结构

```typescript
type TranscriptMessage = SerializedMessage & {
  parentUuid: UUID | null           // 物理父节点（用于链式存储）
  logicalParentUuid?: UUID | null   // 逻辑父节点（compact boundary 时保留）
  isSidechain: boolean              // 是否子代理消息
  agentId?: string                  // 子代理 ID
  teamName?: string                 // 团队名称
  agentName?: string                // 代理自定义名称
  agentColor?: string               // 代理颜色
  promptId?: string                 // 用于 OTel 关联
}
```

#### 3.1.3 文件路径结构

```
~/.claude/projects/
  └── {sanitized_project_path}/
      ├── {sessionId}.jsonl              # 主会话文件
      ├── {sessionId}/
      │   ├── subagents/
      │   │   ├── agent-{agentId}.jsonl  # 子代理 sidechain
      │   │   └── agent-{agentId}.meta.json
      │   ├── remote-agents/
      │   │   └── remote-agent-{taskId}.meta.json
      │   └── tool-results/              # 大工具结果持久化
      │       └── {toolUseId}.{json|txt}
      └── ...
```

### 3.2 关键流程

#### 3.2.1 消息写入流程

```
1. useLogMessages 检测消息变化
   ↓
2. cleanMessagesForLogging 过滤 (移除 progress, 外部用户移除 REPL 等)
   ↓
3. recordTranscript / recordSidechainTranscript
   ↓
4. getProject().insertMessageChain()
   ↓
5. 检查 shouldSkipPersistence (test 环境、--no-session-persistence 等)
   ↓
6. materializeSessionFile() (首次写入时创建文件)
   ↓
7. appendEntry() → enqueueWrite() → scheduleDrain()
   ↓
8. 100ms 后 drainWriteQueue() 批量写入
   ↓
9. 同时 persistToRemote() 同步到 Session Ingress (如启用)
```

#### 3.2.2 会话加载流程

```
loadTranscriptFile(filePath)
   ↓
1. 检查文件大小 > 5MB ?
   ↓ 是
   readTranscriptForLoad() → 跳过 pre-boundary 内容
   ↓
2. 检查 hasPreservedSegment ?
   ↓ 否
   walkChainBeforeParse() → 字节级死分支过滤
   ↓
3. parseJSONL() 解析剩余内容
   ↓
4. 遍历 entries:
   - TranscriptMessage → messages Map
   - metadata entries → 各自 Map
   - compact_boundary → 清空 contextCollapseCommits
   ↓
5. applyPreservedSegmentRelinks() → 处理保留段重链
   ↓
6. applySnipRemovals() → 处理 snip 删除
   ↓
7. 计算 leafUuids (终端消息集合)
   ↓
8. 返回完整加载结果
```

#### 3.2.3 对话链重建 (`buildConversationChain`)

```typescript
// 从叶子节点回溯到根节点
while (currentMsg) {
  if (seen.has(currentMsg.uuid)) { /* 循环检测 */ break }
  seen.add(currentMsg.uuid)
  transcript.push(currentMsg)
  currentMsg = currentMsg.parentUuid 
    ? messages.get(currentMsg.parentUuid) 
    : undefined
}
transcript.reverse()
// 然后 recoverOrphanedParallelToolResults 恢复并行工具结果
```

### 3.3 关键技术细节

#### 3.3.1 Pre-compact Skip 优化

当文件 > 5MB 时，使用 `readTranscriptForLoad` 进行流式读取：

```typescript
// 状态机驱动的 chunked 读取
const s: LoadState = {
  out: { buf, len, cap },           // 输出缓冲区
  boundaryStartOffset: 0,          // 最后 compact boundary 位置
  hasPreservedSegment: false,
  lastSnapSrc: null,               // 最后一个 attribution snapshot
  // ... 跨 chunk 状态
}

// 扫描过程中:
// - 跳过 attribution-snapshot 行 (fd 级别过滤)
// - 遇到 compact_boundary 时重置输出缓冲区
// - 保留段边界不截断
```

#### 3.3.2 死分支过滤 (`walkChainBeforeParse`)

在 JSON.parse 之前用字节级扫描过滤无效分支：

```typescript
// 识别 transcript 消息的特征: {"parentUuid":
// 识别 uuid 特征: "uuid":"<36 chars>","timestamp":"
// 
// 1. 构建 msgIdx: [lineStart, lineEnd, parentStart] 三元组数组
// 2. 找到最后一个非 sidechain 的叶子节点
// 3. 沿 parentUuid 链回溯，收集存活消息
// 4. 如果死亡消息占比 > 50%，拼接存活消息返回新 Buffer
```

#### 3.3.3 Progress Bridge (兼容旧版本)

旧版本将 progress 消息写入 transcript 并参与 parentUuid 链：

```typescript
const progressBridge = new Map<UUID, UUID | null>()

// 加载时遇到 legacy progress entry:
progressBridge.set(entry.uuid, resolvedParent)

// 后续消息如果 parentUuid 指向 progress:
if (progressBridge.has(entry.parentUuid)) {
  entry.parentUuid = progressBridge.get(entry.parentUuid)
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 内部调用图

```
sessionStorage.ts
├── 写入路径
│   ├── recordTranscript() ─────→ useLogMessages.ts, QueryEngine.ts, queryHelpers.ts
│   ├── recordSidechainTranscript() → LocalMainSessionTask.ts, runAgent.ts, forkedAgent.ts
│   └── save* 系列 ─────────────→ 各命令实现 (rename.ts, tag/index.ts, etc.)
│
├── 读取路径
│   ├── loadTranscriptFile() ───→ sessionRestore.ts, conversationRecovery.ts
│   ├── loadMessageLogs() ──────→ ResumeConversation.tsx, listSessionsImpl.ts
│   └── getAgentTranscript() ───→ AgentTool/resumeAgent.ts
│
├── 恢复路径
│   ├── hydrateRemoteSession() ─→ remote session 恢复
│   └── hydrateFromCCRv2InternalEvents() → CCR v2 恢复
│
└── 工具函数
    ├── buildConversationChain() → 多处使用
    ├── cleanMessagesForLogging() → useLogMessages.ts
    └── 元数据读写 ─────────────→ 各功能模块
```

### 4.2 依赖文件

| 文件 | 用途 |
|------|------|
| `src/bootstrap/state.ts` | 获取 sessionId、originalCwd、切换会话 |
| `src/types/logs.ts` | Entry 类型定义、LogOption |
| `src/utils/sessionStoragePortable.ts` | 共享的 JSON 解析、路径处理、文件读取工具 |
| `src/services/api/sessionIngress.ts` | 远程会话同步 API |
| `src/utils/toolResultStorage.ts` | ContentReplacementRecord 类型、工具结果持久化 |
| `src/utils/fileHistory.ts` | FileHistorySnapshot 类型 |
| `src/utils/concurrentSessions.ts` | updateSessionName |

### 4.3 关键导出函数

```typescript
// 写入
export async function recordTranscript(messages: Message[], ...): Promise<UUID | null>
export async function recordSidechainTranscript(messages: Message[], agentId?: string): Promise<void>
export async function recordContentReplacement(replacements: ContentReplacementRecord[], agentId?: AgentId): Promise<void>

// 元数据
export function saveCustomTitle(sessionId: UUID, customTitle: string, ...): Promise<void>
export function saveAiGeneratedTitle(sessionId: UUID, aiTitle: string): void
export async function saveTag(sessionId: UUID, tag: string, ...): Promise<void>
export async function saveAgentName(sessionId: UUID, agentName: string, ...): Promise<void>
export function saveMode(mode: 'coordinator' | 'normal'): void
export function saveWorktreeState(worktreeSession: PersistedWorktreeSession | null): void

// 读取
export async function loadTranscriptFile(filePath: string, opts?: {...}): Promise<LoadResult>
export async function loadMessageLogs(limit?: number): Promise<LogOption[]>
export async function getAgentTranscript(agentId: AgentId): Promise<{messages: Message[], contentReplacements: ContentReplacementRecord[]} | null>

// 恢复
export async function hydrateRemoteSession(sessionId: string, ingressUrl: string): Promise<boolean>
export async function hydrateFromCCRv2InternalEvents(sessionId: string): Promise<boolean>

// 工具
export function buildConversationChain(messages: Map<UUID, TranscriptMessage>, leafMessage: TranscriptMessage): TranscriptMessage[]
export function cleanMessagesForLogging(messages: Message[], allMessages?: readonly Message[]): Transcript
export function clearSessionMessagesCache(): void
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 类别 | 依赖 | 用途 |
|------|------|------|
| 运行时 | `bun:bundle` (feature 函数) | 功能开关 |
| Node.js 内置 | `fs/promises`, `fs` (sync) | 文件操作 |
| Node.js 内置 | `crypto` (UUID 类型) | 类型定义 |
| Node.js 内置 | `path` | 路径处理 |
| 第三方 | `lodash-es/memoize` | 会话消息缓存 |

### 5.2 远程服务交互

#### 5.2.1 Session Ingress API (v1)

```typescript
// 写入
sessionIngress.appendSessionLog(sessionId, entry, remoteIngressUrl)
// - 使用 Last-Uuid header 实现乐观并发控制
// - 409 冲突时采用服务器 UUID 重试

// 读取
sessionIngress.getSessionLogs(sessionId, ingressUrl)
```

#### 5.2.2 CCR v2 Internal Events

```typescript
// 注册写入器
setInternalEventWriter(writer: InternalEventWriter): void

// 注册读取器
setInternalEventReader(reader: InternalEventReader, subagentReader: InternalEventReader): void
```

通过 `src/bridge/sessionRunner.ts` 等文件注册，用于云端会话恢复。

### 5.3 环境变量与配置

| 变量/配置 | 影响 |
|-----------|------|
| `NODE_ENV=test` | 默认跳过持久化 (除非 `TEST_ENABLE_SESSION_PERSISTENCE=1`) |
| `--no-session-persistence` | 禁用所有 transcript 写入 |
| `CLAUDE_CODE_SKIP_PROMPT_HISTORY` | 跳过提示历史 |
| `cleanupPeriodDays=0` (设置) | 禁用持久化 |
| `ENABLE_SESSION_PERSISTENCE` | 启用远程会话同步 |
| `CLAUDE_CODE_DISABLE_PRECOMPACT_SKIP` | 禁用预压缩跳过优化 |
| `CLAUDE_AFTER_LAST_COMPACT` | 只读取 compact boundary 后的内容 |

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 数据一致性风险

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| 并发写入 | 多个进程同时写入同一会话文件 | 单进程架构，sidechain 使用独立文件 |
| 写入中断 | 崩溃导致 JSONL 行不完整 | 下次启动时忽略损坏行 (parseJSONL 容错) |
| 远程同步失败 | Session Ingress 409/401 错误 | 指数退避重试，409 时采用服务器 UUID |
| 大文件 OOM | 超大 session 文件导致内存溢出 | 5MB+ 启用流式读取，50MB+ tombstone 跳过 |

#### 6.1.2 恢复一致性风险

```typescript
// checkResumeConsistency 监控恢复一致性
logEvent('tengu_resume_consistency_delta', {
  expected,  // 写入时记录的消息数
  actual,    // 恢复后重建的消息数
  delta      // 差异
})
```

已知问题场景:
- **Snip 操作**: 删除中间消息时，parentUuid 链需要重新链接
- **Compact 保留段**: 保留的消息段需要特殊处理 parentUuid
- **并行工具结果**: 流式工具执行产生的多条消息可能丢失

### 6.2 边界情况

#### 6.2.1 文件系统边界

```typescript
// 路径长度限制
MAX_SANITIZED_LENGTH = 200  // 字符

// 超长路径处理: 截断 + hash 后缀
function sanitizePath(name: string): string {
  if (sanitized.length <= MAX_SANITIZED_LENGTH) return sanitized
  const hash = Bun.hash(name).toString(36)
  return `${sanitized.slice(0, MAX_SANITIZED_LENGTH)}-${hash}`
}
```

#### 6.2.2 跨平台边界

- Windows 路径大小写不敏感处理 (`caseInsensitive = process.platform === 'win32'`)
- Git worktree 路径匹配时的 hash 冲突处理 (Bun.hash vs simpleHash)

#### 6.2.3 消息类型边界

```typescript
// isTranscriptMessage 是 transcript 消息的单一真相源
export function isTranscriptMessage(entry: Entry): entry is TranscriptMessage {
  return (
    entry.type === 'user' ||
    entry.type === 'assistant' ||
    entry.type === 'attachment' ||
    entry.type === 'system'
  )
}
// 注意: progress 消息明确排除，避免链断裂
```

### 6.3 改进建议

#### 6.3.1 短期改进

1. **增强监控**
   - 添加 `tengu_session_storage_write_latency` 直方图指标
   - 监控 `writeQueues` 积压情况

2. **错误处理细化**
   - 区分 `ENOSPC` (磁盘满) 和其他写入错误
   - 磁盘满时优雅降级 (只保留最近 N 条消息)

3. **配置统一**
   - 将分散的环境变量检查集中到配置对象
   - 添加 `sessionStorage.config.ts` 统一配置

#### 6.3.2 中期改进

1. **压缩优化**
   - 考虑对旧 session 文件进行 gzip 压缩
   - 实现透明压缩/解压层

2. **索引优化**
   - 为大型 session 文件构建 UUID → 文件偏移索引
   - 加速随机消息查找 (用于 tombstone 删除)

3. **并发控制**
   - 考虑使用文件锁防止极端情况下的并发写入
   - 或实现写前日志 (WAL) 模式

#### 6.3.3 长期改进

1. **存储引擎抽象**
   ```typescript
   interface SessionStorageBackend {
     append(sessionId: string, entries: Entry[]): Promise<void>
     load(sessionId: string, options?: LoadOptions): Promise<Entry[]>
     // ...
   }
   // 实现: FileSystemBackend, RemoteBackend, HybridBackend
   ```

2. **增量同步协议**
   - 实现基于 merkle tree 的增量同步
   - 减少远程同步带宽

3. **数据迁移工具**
   - 提供 session 文件压缩/归档工具
   - 支持导出为标准格式 (如 OpenAI 的 conversation format)

### 6.4 测试建议

当前测试覆盖情况:
- 未发现专门的 `sessionStorage.test.ts` 文件
- 功能主要通过集成测试覆盖

建议添加:
1. 单元测试: `walkChainBeforeParse` 的死分支检测逻辑
2. 单元测试: `applyPreservedSegmentRelinks` 的各种边界情况
3. 性能测试: 大文件 (>100MB) 加载性能基准
4. 模糊测试: JSONL 解析的容错能力

---

## 附录: 关键常量参考

```typescript
// 文件大小阈值
const MAX_TOMBSTONE_REWRITE_BYTES = 50 * 1024 * 1024  // 50MB
export const MAX_TRANSCRIPT_READ_BYTES = 50 * 1024 * 1024  // 50MB
export const SKIP_PRECOMPACT_THRESHOLD = 5 * 1024 * 1024   // 5MB

// 缓冲区大小
export const LITE_READ_BUF_SIZE = 65536  // 64KB，用于头尾读取
const TRANSCRIPT_READ_CHUNK_SIZE = 1024 * 1024  // 1MB，流式读取 chunk

// 写入控制
const FLUSH_INTERVAL_MS = 100  // 本地写入延迟
const REMOTE_FLUSH_INTERVAL_MS = 10  // 远程写入延迟
const MAX_CHUNK_BYTES = 100 * 1024 * 1024  // 批量写入分块

// 渐进加载
const INITIAL_ENRICH_COUNT = 50  // 初始加载会话数
```

---

*文档结束*
