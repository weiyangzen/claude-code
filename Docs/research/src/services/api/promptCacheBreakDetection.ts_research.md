# Prompt Cache Break Detection 服务研究文档

## 文件信息
- **路径**: `src/services/api/promptCacheBreakDetection.ts`
- **大小**: 26,288 bytes
- **最后更新**: 2026-04-01

---

## 场景与职责

Prompt Cache Break Detection 服务是 Claude Code 的 **提示缓存诊断系统**，用于检测和解释 Anthropic API 提示缓存命中率下降的原因。该服务处理以下核心场景：

1. **缓存命中率监控**: 跟踪每次 API 调用的 `cache_read_input_tokens` 变化
2. **缓存中断检测**: 当缓存读取 token 数显著下降（>5% 且绝对值 >2000）时触发检测
3. **根因分析**: 比较系统提示词、工具定义、模型、缓存策略等状态变化
4. **调试支持**: 生成 diff 文件供开发调试，发送详细 analytics 事件
5. **特殊场景处理**: 处理 compaction、缓存删除、TTL 过期等预期内的缓存下降

---

## 功能点目的

### 1. 状态记录 (`recordPromptState`)
在每次 API 调用前记录完整的提示状态快照：
- 系统提示词哈希（含和不含 `cache_control`）
- 工具定义哈希（整体 + 单个工具）
- 模型、Fast 模式、全局缓存策略
- Beta 头、自动模式、超额状态、缓存微压缩状态
- Effort 值、额外 body 参数

### 2. 中断检测 (`checkResponseForCacheBreak`)
在 API 响应后检查缓存命中率变化：
- 计算缓存读取 token 数变化
- 排除首次调用、Haiku 模型、缓存删除后的预期下降
- 检测阈值: 下降 >5% 且绝对值 >2000 tokens

### 3. 根因分析
基于 `pendingChanges` 分析中断原因：
- 系统提示词变化（字符数变化）
- 工具变化（添加/删除/修改）
- 模型切换、Fast 模式切换
- 缓存策略变化、Beta 头变化
- 自动模式、超额状态、缓存微压缩切换
- Effort 值变化、额外 body 参数变化

### 4. TTL 过期检测
根据上次助手消息时间判断：
- < 5 分钟: 可能是服务端问题
- 5 分钟 - 1 小时: 可能是 5 分钟 TTL 过期
- > 1 小时: 可能是 1 小时 TTL 过期

### 5. 特殊场景通知
- `notifyCacheDeletion`: 缓存微压缩删除后的通知
- `notifyCompaction`: Compaction 后的基线重置
- `cleanupAgentTracking`: Agent 清理时的状态删除

---

## 具体技术实现

### 关键数据结构

```typescript
// 状态快照（API 调用前记录）
type PromptStateSnapshot = {
  system: TextBlockParam[]
  toolSchemas: BetaToolUnion[]
  querySource: QuerySource
  model: string
  agentId?: AgentId
  fastMode?: boolean
  globalCacheStrategy?: string
  betas?: readonly string[]
  autoModeActive?: boolean
  isUsingOverage?: boolean
  cachedMCEnabled?: boolean
  effortValue?: string | number
  extraBodyParams?: unknown
}

// 内部状态存储
type PreviousState = {
  systemHash: number
  toolsHash: number
  cacheControlHash: number  // 捕获 scope/TTL 变化
  toolNames: string[]
  perToolHashes: Record<string, number>
  systemCharCount: number
  model: string
  fastMode: boolean
  globalCacheStrategy: string
  betas: string[]
  autoModeActive: boolean
  isUsingOverage: boolean
  cachedMCEnabled: boolean
  effortValue: string
  extraBodyHash: number
  callCount: number
  pendingChanges: PendingChanges | null
  prevCacheReadTokens: number | null
  cacheDeletionsPending: boolean
  buildDiffableContent: () => string
}

// 待处理变化（用于生成根因）
type PendingChanges = {
  systemPromptChanged: boolean
  toolSchemasChanged: boolean
  // ... 其他变化标志
  addedTools: string[]
  removedTools: string[]
  changedToolSchemas: string[]
  // ... 其他变化详情
}
```

### 关键流程

#### 状态记录流程
```
recordPromptState(snapshot)
├── 获取 tracking key（基于 querySource + agentId）
├── 计算各类哈希
│   ├── strippedSystem（移除 cache_control）
│   ├── strippedTools（移除 cache_control）
│   ├── cacheControlHash（仅 cache_control）
│   └── perToolHashes（延迟计算）
├── 检查是否有前一个状态
│   ├── 无 → 初始化状态，返回
│   └── 有 → 比较所有字段
├── 如果有变化
│   ├── 计算工具增删列表
│   ├── 计算变化的工具 schema
│   └── 存储 pendingChanges
└── 更新所有哈希值
```

#### 中断检测流程
```
checkResponseForCacheBreak(querySource, cacheReadTokens, ...)
├── 获取 tracking key 和状态
├── 跳过检查条件
│   ├── 无状态
│   ├── Haiku 模型
│   └── 首次调用（prevCacheReadTokens === null）
├── 检查 cacheDeletionsPending
│   └── 是 → 重置标志，返回（预期下降）
├── 计算 token 下降
│   ├── 下降 < 5% 或 < 2000 tokens → 返回
├── 构建根因说明
│   ├── 从 pendingChanges 生成描述列表
│   ├── 无变化 → 检查 TTL 过期
│   └── 无任何解释 → "unknown cause"
├── 发送 analytics 事件
├── 生成 diff 文件（如可能）
└── 记录调试日志
```

### 哈希计算

```typescript
function computeHash(data: unknown): number {
  const str = jsonStringify(data)
  if (typeof Bun !== 'undefined') {
    const hash = Bun.hash(str)
    // Bun.hash 可能返回 bigint，安全转换为 number
    return typeof hash === 'bigint' ? Number(hash & 0xffffffffn) : hash
  }
  // Node.js 降级使用 djb2
  return djb2Hash(str)
}
```

### 工具名脱敏

```typescript
function sanitizeToolName(name: string): string {
  // MCP 工具名是用户配置的，可能包含路径信息
  return name.startsWith('mcp__') ? 'mcp' : name
}
```

---

## 关键代码路径与文件引用

### 核心实现
- `src/services/api/promptCacheBreakDetection.ts` - 本文件

### 调用方
| 文件 | 调用函数 | 用途 |
|------|---------|------|
| `src/services/api/claude.ts` | `recordPromptState`, `checkResponseForCacheBreak` | 每次 API 调用前后 |
| `src/commands/compact/compact.ts` | `notifyCompaction` | Compaction 后重置基线 |
| `src/commands/clear/caches.ts` | `resetPromptCacheBreakDetection` | /clear 命令重置 |
| `src/tools/AgentTool/runAgent.ts` | `cleanupAgentTracking` | Agent 结束时清理 |

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/services/analytics/index.ts` | `logEvent` - 发送 `tengu_prompt_cache_break` |
| `src/services/analytics/metadata.ts` | `sanitizeToolNameForAnalytics` |
| `src/types/message.ts` | `Message`, `AssistantMessage` 类型 |
| `src/types/connectorText.ts` | `isConnectorTextBlock` |
| `src/utils/hash.ts` | `djb2Hash`（Node.js 降级） |
| `src/utils/log.ts` | `logError` |
| `src/utils/debug.ts` | `logForDebugging` |
| `src/utils/permissions/filesystem.ts` | `getClaudeTempDir` |
| `src/utils/slowOperations.ts` | `jsonStringify` |
| `src/constants/querySource.ts` | `QuerySource` 类型 |

---

## 依赖与外部交互

### Analytics 事件

| 事件名 | 触发时机 | 关键字段 |
|--------|---------|---------|
| `tengu_prompt_cache_break` | 检测到缓存中断 | 所有变化标志、工具列表、token 数变化、时间间隔 |

### 事件字段详解
```typescript
logEvent('tengu_prompt_cache_break', {
  // 布尔标志（哪些因素发生了变化）
  systemPromptChanged, toolSchemasChanged, modelChanged,
  fastModeChanged, cacheControlChanged, globalCacheStrategyChanged,
  betasChanged, autoModeChanged, overageChanged, cachedMCChanged,
  effortChanged, extraBodyChanged,
  
  // 数量统计
  addedToolCount, removedToolCount, systemCharDelta,
  
  // 脱敏后的列表（逗号分隔字符串）
  addedTools, removedTools, changedToolSchemas,
  addedBetas, removedBetas,
  
  // 缓存策略变化
  prevGlobalCacheStrategy, newGlobalCacheStrategy,
  
  // 调用上下文
  callNumber, prevCacheReadTokens, cacheReadTokens, cacheCreationTokens,
  
  // 时间分析
  timeSinceLastAssistantMsg, lastAssistantMsgOver5minAgo, lastAssistantMsgOver1hAgo,
  
  // 调试信息
  requestId
})
```

### Diff 文件生成

```typescript
async function writeCacheBreakDiff(prevContent: string, newContent: string): Promise<string | undefined> {
  const diffPath = join(getClaudeTempDir(), `cache-break-${randomSuffix}.diff`)
  const patch = createPatch('prompt-state', prevContent, newContent, 'before', 'after')
  await writeFile(diffPath, patch)
  return diffPath
}
```

Diff 文件保存到临时目录，路径包含在调试日志中供开发使用。

---

## 风险、边界与改进建议

### 已知风险

1. **内存泄漏风险**
   ```typescript
   const MAX_TRACKED_SOURCES = 10
   ```
   - 每个 source 存储约 300KB+ 的 diffableContent
   - 使用 LRU 淘汰策略，但大量 subagent 仍可能导致内存压力

2. **哈希冲突**
   - 使用 32 位哈希（djb2 或 Bun.hash 截断）
   - 虽然概率低，但理论上有碰撞风险

3. **误报风险**
   - 服务端路由变化、计费/推理不一致等不可见因素
   - 约 90% 的 "无变化" 中断是服务端原因（per BQ 分析）

4. **Tracking Key 限制**
   ```typescript
   const TRACKED_SOURCE_PREFIXES = [
     'repl_main_thread', 'sdk', 'agent:custom', 'agent:default', 'agent:builtin'
   ]
   ```
   - 仅跟踪特定 source 类型
   - `speculation`, `session_memory` 等短生命周期 source 不跟踪

### 边界情况

| 场景 | 行为 |
|------|------|
| `compact` source | 映射到 `repl_main_thread`（共享服务端缓存）|
| 缓存删除后 | `cacheDeletionsPending` 标志阻止误报 |
| Compaction 后 | `prevCacheReadTokens` 重置为 null |
| Agent 清理 | `cleanupAgentTracking` 删除对应状态 |
| `/clear` 命令 | `resetPromptCacheBreakDetection` 清空所有状态 |
| Haiku 模型 | 完全跳过检测（不同缓存行为）|
| 首次调用 | 无 prevCacheReadTokens，跳过检测 |

### 改进建议

1. **服务端原因细分**
   ```typescript
   // 当前
   reason = 'likely server-side (prompt unchanged, <5min gap)'
   
   // 建议：结合更多信号
   if (cacheCreationTokens > prevCacheCreationTokens * 0.5) {
     reason = 'likely server-side cache eviction'
   } else if (timeSinceLastAssistantMsg < 60000) {
     reason = 'likely server-side routing change'
   }
   ```

2. **内存优化**
   - 考虑压缩 `buildDiffableContent` 存储
   - 或使用 WeakRef 允许垃圾回收

3. **持久化分析**
   - 将中断历史持久化到磁盘
   - 支持跨会话的趋势分析

4. **实时监控**
   ```typescript
   // 建议添加指标
   logEvent('tengu_prompt_cache_health', {
     source,
     cacheHitRate: cacheReadTokens / totalInputTokens,
     callCount,
     avgTokensPerCall
   })
   ```

5. **工具变化细化**
   - 当前仅记录变化的工具名
   - 建议记录具体哪个 schema 字段变化（参数、描述等）

6. **测试覆盖**
   - 添加针对各种变化组合的单元测试
   - 模拟服务端导致的缓存下降
   - 测试内存限制下的 LRU 行为

### 相关模式
- 与 `logging.ts` 共享 `GlobalCacheStrategy` 类型
- 与 `AgentTool` 紧密集成（每个 Agent 有独立 tracking）
- 与 `compact` 命令配合（重置基线）
- 与 `cached microcompact` 功能配合（预期缓存下降）

### 性能考虑
- 哈希计算使用 `jsonStringify`，大提示词可能有性能影响
- `perToolHashes` 延迟计算（仅在工具变化时）
- Diff 文件异步写入，不阻塞主流程
