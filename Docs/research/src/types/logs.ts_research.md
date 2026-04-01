# logs.ts 研究文档

## 场景与职责

`src/types/logs.ts` 是 Claude Code CLI 的会话日志（Session Log）类型定义文件，定义了会话持久化的完整数据结构。核心职责：

1. **会话消息序列化**: 定义 `SerializedMessage` 和 `TranscriptMessage`，支持消息持久化到 JSONL
2. **会话元数据管理**: 定义 `LogOption` 类型，包含会话的完整元数据（标题、标签、分支、PR 链接等）
3. **会话恢复支持**: 定义恢复所需的各类 Entry 类型（WorktreeState, ContentReplacement, ContextCollapse 等）
4. **Agent 会话隔离**: 支持主会话和子代理（subagent）的独立日志存储

该文件是会话持久化系统的核心，被 20+ 文件依赖，直接影响 `--resume` 功能的可靠性。

## 功能点目的

### 1. 消息序列化体系
- **SerializedMessage**: 扩展 Message 类型，添加会话上下文（cwd, sessionId, timestamp, version 等）
- **TranscriptMessage**: 用于持久化的消息格式，包含 parentUuid 链式引用

### 2. 会话元数据（LogOption）
包含 30+ 字段的完整会话描述：
- **基础信息**: date, messages, firstPrompt, messageCount
- **会话标识**: sessionId, isSidechain, teamName, agentName, agentColor
- **Git 上下文**: gitBranch, projectPath
- **PR 集成**: prNumber, prUrl, prRepository
- **工作区**: worktreeSession（worktree 状态）
- **内容替换**: contentReplacements（大内容块替换记录）
- **会话摘要**: summary, customTitle, tag

### 3. 特殊 Entry 类型
- **SummaryMessage**: AI 生成的会话摘要
- **CustomTitleMessage**: 用户自定义会话标题
- **AiTitleMessage**: AI 建议的标题（与自定义标题区分）
- **TaskSummaryMessage**: 周期性 Agent 任务摘要（用于 `claude ps`）
- **PRLinkMessage**: 关联的 GitHub PR 信息
- **ModeEntry**: 会话模式（coordinator/normal）

### 4. 上下文压缩（Context Collapse）
- **ContextCollapseCommitEntry**: 持久化的压缩提交点（类型名混淆为 `marble-origami-commit`）
- **ContextCollapseSnapshotEntry**: 暂存队列和触发器状态快照（`marble-origami-snapshot`）

### 5. 归因追踪（Attribution）
- **AttributionSnapshotMessage**: 追踪 Claude 的字符级贡献
- **FileAttributionState**: 单文件的归因状态（内容哈希、贡献字符数、修改时间）

## 具体技术实现

### 关键数据结构

```typescript
// 序列化消息（持久化格式）
interface SerializedMessage extends Message {
  cwd: string
  userType: string
  entrypoint?: string           // CLAUDE_CODE_ENTRYPOINT
  sessionId: string
  timestamp: string
  version: string
  gitBranch?: string
  slug?: string                 // 用于恢复的计划标识
}

// 完整会话元数据
interface LogOption {
  date: string
  messages: SerializedMessage[]
  fullPath?: string
  value: number                 // 用于排序的数值
  created: Date
  modified: Date
  firstPrompt: string
  messageCount: number
  fileSize?: number
  isSidechain: boolean
  isLite?: boolean              // 轻量模式（消息未加载）
  sessionId?: string
  teamName?: string
  agentName?: string
  agentColor?: string
  agentSetting?: string
  isTeammate?: boolean
  leafUuid?: UUID
  summary?: string
  customTitle?: string
  tag?: string
  fileHistorySnapshots?: FileHistorySnapshot[]
  attributionSnapshots?: AttributionSnapshotMessage[]
  contextCollapseCommits?: ContextCollapseCommitEntry[]
  contextCollapseSnapshot?: ContextCollapseSnapshotEntry
  gitBranch?: string
  projectPath?: string
  prNumber?: number
  prUrl?: string
  prRepository?: string
  mode?: 'coordinator' | 'normal'
  worktreeSession?: PersistedWorktreeSession | null
  contentReplacements?: ContentReplacementRecord[]
}

// 上下文压缩提交点
interface ContextCollapseCommitEntry {
  type: 'marble-origami-commit'  // 混淆的类型名（避免外部构建泄漏）
  sessionId: UUID
  collapseId: string              // 16 位压缩 ID
  summaryUuid: string             // 摘要占位符的 UUID
  summaryContent: string          // 完整 <collapsed> XML
  summary: string                 // 纯文本摘要（用于 ctx_inspect）
  firstArchivedUuid: string       // 归档范围起始
  lastArchivedUuid: string        // 归档范围结束
}

// 上下文压缩快照
interface ContextCollapseSnapshotEntry {
  type: 'marble-origami-snapshot'
  sessionId: UUID
  staged: Array<{
    startUuid: string
    endUuid: string
    summary: string
    risk: number
    stagedAt: number
  }>
  armed: boolean                  // 触发器状态
  lastSpawnTokens: number         // 上次 spawn 时的 token 数
}

// Entry 联合类型（所有可持久化条目）
type Entry =
  | TranscriptMessage
  | SummaryMessage
  | CustomTitleMessage
  | AiTitleMessage
  | LastPromptMessage
  | TaskSummaryMessage
  | TagMessage
  | AgentNameMessage
  | AgentColorMessage
  | AgentSettingMessage
  | PRLinkMessage
  | FileHistorySnapshotMessage
  | AttributionSnapshotMessage
  | QueueOperationMessage
  | SpeculationAcceptMessage
  | ModeEntry
  | WorktreeStateEntry
  | ContentReplacementEntry
  | ContextCollapseCommitEntry
  | ContextCollapseSnapshotEntry
```

### 工具函数

```typescript
// 按修改时间排序日志（最新的在前）
export function sortLogs(logs: LogOption[]): LogOption[]
```

排序逻辑：
1. 首先按 modified 时间降序
2. 如果相同，按 created 时间降序

## 关键代码路径与文件引用

### 类型定义
- `src/types/logs.ts` - 本文件，日志类型定义

### 会话存储核心
- `src/utils/sessionStorage.ts` - 会话存储实现（Project 类）
- `src/utils/sessionRestore.ts` - 会话恢复逻辑
- `src/utils/sessionStoragePortable.ts` - 可移植存储工具

### 日志管理
- `src/utils/log.ts` - 日志工具（与类型同名但不同功能）
- `src/utils/crossProjectResume.ts` - 跨项目恢复
- `src/utils/conversationRecovery.ts` - 对话恢复

### 元数据提取
- `src/utils/agenticSessionSearch.ts` - Agent 会话搜索
- `src/utils/attribution.ts` - 归因计算
- `src/utils/commitAttribution.ts` - 提交归因
- `src/utils/fileHistory.ts` - 文件历史

### UI 组件
- `src/components/LogSelector.tsx` - 日志选择器
- `src/components/SessionPreview.tsx` - 会话预览
- `src/screens/ResumeConversation.tsx` - 恢复会话界面

### 统计与分析
- `src/utils/stats.ts` - 会话统计
- `src/commands/insights.ts` - 会话洞察
- `src/commands/branch/branch.ts` - 分支管理

### 其他使用者
- `src/main.tsx` - 主入口
- `src/screens/REPL.tsx` - REPL 主界面
- `src/services/PromptSuggestion/speculation.ts` - 推测执行
- `src/services/api/sessionIngress.ts` - 会话入口
- `src/utils/logoV2Utils.ts` - Logo 工具
- `src/components/LogoV2/feedConfigs.tsx` - Logo 配置

## 依赖与外部交互

### 导入依赖
```typescript
import type { UUID } from 'crypto'                                    // Node.js crypto
import type { FileHistorySnapshot } from 'src/utils/fileHistory.js'   // 文件历史
import type { ContentReplacementRecord } from 'src/utils/toolResultStorage.js'  // 内容替换
import type { AgentId } from './ids.js'                               // Agent ID
import type { Message } from './message.js'                           // 消息类型
import type { QueueOperationMessage } from './messageQueueTypes.js'   // 队列操作
```

### 被依赖方（20+ 文件）
主要分布：
- 会话存储与恢复（`src/utils/sessionStorage.ts`, `src/utils/sessionRestore.ts`）
- UI 组件（`src/components/LogSelector.tsx`, `src/screens/ResumeConversation.tsx`）
- 工具和分析（`src/utils/attribution.ts`, `src/utils/stats.ts`）

## 风险、边界与改进建议

### 潜在风险

1. **类型名混淆的安全性问题**
   - `marble-origami-commit` 和 `marble-origami-snapshot` 是混淆名
   - 目的是防止外部构建通过类型名推断功能
   - 但混淆名增加了代码理解难度

2. **LogOption 的字段膨胀**
   - 30+ 字段的接口难以维护
   - 新增字段需要考虑向后兼容性
   - 部分字段（如 `value`）的用途不明确

3. **时间类型不一致**
   - `created` 和 `modified` 使用 `Date` 对象
   - 但 `timestamp` 字段使用 ISO 字符串
   - 序列化/反序列化时容易出错

4. **可选字段过多**
   - 大多数字段都是可选的（`?`）
   - 运行时难以确定哪些字段一定存在
   - 增加了空值检查负担

### 边界情况

1. **轻量日志（isLite）**
   - `isLite: true` 时 `messages` 数组为空
   - 需要额外加载才能获取完整消息
   - 用于 `--resume` 列表的快速显示

2. **Sidechain 会话**
   - `isSidechain: true` 表示子代理会话
   - 存储路径与主会话不同（`subagents/` 子目录）
   - 恢复逻辑需要特殊处理

3. **Worktree 会话**
   - `worktreeSession` 为 `null` 表示已退出 worktree
   - `undefined` 表示从未进入 worktree
   - 恢复时需要检查 worktree 路径是否存在

4. **内容替换重建**
   - `contentReplacements` 用于恢复时重建提示缓存
   - Agent 子链和主线程使用不同的替换记录
   - 通过 `agentId` 字段区分

### 改进建议

1. **类型分层**
   ```typescript
   // 建议：将 LogOption 分层
   interface LogOptionBase { /* 核心字段 */ }
   interface LogOptionMetadata { /* 元数据字段 */ }
   interface LogOptionAgent { /* Agent 相关字段 */ }
   interface LogOption extends LogOptionBase, LogOptionMetadata, LogOptionAgent {}
   ```

2. **时间类型统一**
   ```typescript
   // 建议：统一使用 ISO 字符串或统一使用 Date
   interface LogOption {
     createdAt: string  // ISO 8601
     modifiedAt: string // ISO 8601
   }
   ```

3. **字段文档化**
   - 为每个字段添加 JSDoc 说明
   - 标明字段的用途、来源、是否持久化

4. **Entry 类型验证**
   - 当前 Entry 联合类型无运行时验证
   - 建议增加 Zod Schema 用于加载时验证

5. **混淆名可配置**
   ```typescript
   // 建议：通过构建时替换而非硬编码
   const COLLAPSE_COMMIT_TYPE = process.env.FEATURE_GATE 
     ? 'context-collapse-commit' 
     : 'marble-origami-commit'
   ```

6. **排序函数优化**
   ```typescript
   // 当前实现每次创建新数组
   // 建议：支持原地排序选项
   export function sortLogs(logs: LogOption[], mutate = false): LogOption[]
   ```
