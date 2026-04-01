# 研究报告：src/utils/attachments.ts

> 研究范围：代码、脚本、配置、测试及必要实现上下文  
> 研究对象：`src/utils/attachments.ts`（~4000 行，模块核心）  
> 执行器：kimi / model=k2p5

---

## 一、场景与职责

`src/utils/attachments.ts` 是 Claude Code 中**附件（Attachment）生成与管理的中央工厂**。它在每次用户提交或工具轮次结束时被调用，负责把外部上下文（文件、IDE 选区、队列命令、诊断信息、内存文件、Agent 状态等）转换成模型可见的 `AttachmentMessage`。

### 1.1 核心使用场景

| 场景 | 触发时机 | 说明 |
|------|----------|------|
| **用户输入提交** | `processUserInput.ts` | 提取 `@提及文件`、MCP 资源、Agent `@提及`、技能发现等 |
| **工具轮次结束** | `query.ts` | 在模型完成一次 tool-use 循环后，注入队列命令、变更文件、嵌套内存、计划模式提醒等 |
| **Compaction（上下文压缩）后恢复** | `services/compact/compact.ts` | 重新生成被压缩掉的文件引用、Agent 列表、MCP 指令增量等 |
| **Hook / 子 Agent / 任务执行** | `toolHooks.ts`、`runAgent.ts` | 将异步 Hook 结果、子 Agent 输出包装为附件消息 |
| **REPL 通知** | `screens/REPL.tsx` | 把队列中的命令渲染为通知附件 |

### 1.2 模块职责边界

- **生成**：把原始数据（文件路径、IDE 选区、内存文件、诊断结果等）转换为类型安全的 `Attachment` 对象。
- **聚合**：按优先级和并发安全策略合并数十种附件来源。
- **去重/节流**：通过 `readFileState`（LRU 缓存）、`loadedNestedMemoryPaths`（Session 级 Set）、`sentSkillNames`（模块级 Map）等机制防止重复注入。
- **消息包装**：提供 `createAttachmentMessage()` 将 `Attachment` 包装成带 UUID 和时间戳的 `AttachmentMessage`。
- **渲染无关**：附件的 UI 渲染由 `AttachmentMessage.tsx` 负责；本模块只负责**数据生产**。

---

## 二、功能点目的

文件内定义了约 **50+ 种 Attachment 类型**，可归纳为 10 个功能域：

### 2.1 文件与目录附件
- `@file.txt` / `@"path with spaces"` → `FileAttachment` / `AlreadyReadFileAttachment`
- `@dir/` → `DirectoryAttachment`
- 支持行号语法 `@file.txt#L10-20`
- 大文件自动截断；PDF 超过阈值转为 `PDFReferenceAttachment`

### 2.2 IDE 集成附件
- `selected_lines_in_ide`：当前 IDE 中选中的代码块
- `opened_file_in_ide`：当前 IDE 打开的文件（同时触发嵌套内存加载）

### 2.3 变更文件追踪（Changed Files）
- 扫描 `readFileState` 中缓存的文件，检测自上次读取后的修改
- 文本文件生成 `edited_text_file`（基于 `getSnippetForTwoFileDiff` 的 diff 摘要）
- 图片文件生成 `edited_image_file`

### 2.4 嵌套内存（Nested Memory / CLAUDE.md）
- 根据 `@提及` 或 IDE 打开的文件路径，向上遍历目录树
- 加载匹配的 `CLAUDE.md`、条件规则（conditional rules）、Managed/User 规则
- 使用 `loadedNestedMemoryPaths` 做 Session 级去重

### 2.5 相关记忆预取（Relevant Memories）
- `startRelevantMemoryPrefetch()`：非阻塞预取，基于用户输入从 `memdir` 搜索相关记忆
- 受 `MAX_MEMORY_LINES`（200 行）和 `MAX_MEMORY_BYTES`（4KB）以及 `MAX_SESSION_BYTES`（60KB）三重限制

### 2.6 计划模式 / 自动模式附件
- `plan_mode`：每 5 个人类回合注入一次，交替 full/sparse 提醒
- `plan_mode_exit` / `plan_mode_reentry`：模式切换的一次性通知
- `auto_mode` / `auto_mode_exit`：基于 `TRANSCRIPT_CLASSIFIER` 特性的自动模式提醒

### 2.7 任务与队列附件
- `queued_command`：把消息队列中的 `prompt` / `task-notification` 转为附件
- `task_status`：统一任务框架（LocalShell/RemoteAgent/LocalAgent）的状态更新
- `todo_reminder` / `task_reminder`：当模型久未使用 TodoWrite/TaskUpdate 时的温和提醒

### 2.8 Agent / 技能 / MCP 增量附件
- `agent_mention`：用户 `@agent-xxx` 触发
- `agent_listing_delta`：Agent 池变化时的增量通知（从 AgentTool 描述中抽离，避免缓存失效）
- `skill_listing` / `dynamic_skill` / `skill_discovery`：技能发现与列表注入
- `deferred_tools_delta` / `mcp_instructions_delta`：ToolSearch 和 MCP 指令的增量更新

### 2.9 Hook 与诊断附件
- `async_hook_response`、`hook_success`、`hook_error_during_execution` 等 10 余种 Hook 附件
- `diagnostics` / `lsp_diagnostics`：IDE 诊断信息

### 2.10 系统与效率附件
- `date_change`：跨午夜时通知模型日期变更（尾部追加，避免前缀缓存失效）
- `token_usage`、`budget_usd`、`output_token_usage`：用量与预算提示
- `compaction_reminder`、`context_efficiency`：上下文压缩与 Snip 效率提醒
- `ultrathink_effort`：`ultrathink` 关键词触发的高强度思考标记
- `teammate_mailbox`、`team_context`：Agent Swarm 的队友通信

---

## 三、具体技术实现

### 3.1 关键数据结构

#### 3.1.1 Attachment 联合类型

```ts
export type Attachment =
  | FileAttachment
  | CompactFileReferenceAttachment
  | PDFReferenceAttachment
  | AlreadyReadFileAttachment
  | { type: 'edited_text_file'; filename: string; snippet: string }
  | { type: 'directory'; path: string; content: string; displayPath: string }
  | { type: 'queued_command'; prompt: string | ContentBlockParam[]; ... }
  | { type: 'plan_mode'; reminderType: 'full' | 'sparse'; ... }
  | ... // 约 50+ 种变体
```

每种附件都是**结构化的纯数据对象**，无方法，便于序列化到消息历史和跨进程传递。

#### 3.1.2 附件消息包装

```ts
export function createAttachmentMessage(attachment: Attachment): AttachmentMessage {
  return {
    attachment,
    type: 'attachment',
    uuid: randomUUID(),
    timestamp: new Date().toISOString(),
  }
}
```

所有附件消息均带 UUID，用于消息选择器、渲染键和持久化恢复。

### 3.2 核心入口：`getAttachments()`

```ts
export async function getAttachments(
  input: string | null,
  toolUseContext: ToolUseContext,
  ideSelection: IDESelection | null,
  queuedCommands: QueuedCommand[],
  messages?: Message[],
  querySource?: QuerySource,
  options?: { skipSkillDiscovery?: boolean },
): Promise<Attachment[]> {
```

#### 执行流程

1. **快速退出**：若环境变量 `CLAUDE_CODE_DISABLE_ATTACHMENTS` 或 `CLAUDE_CODE_SIMPLE` 为真，仅返回 `getQueuedCommandAttachments(queuedCommands)`（保证后台任务通知不丢失）。
2. **超时保护**：创建一个 1000ms 的 `AbortController`，防止附件计算阻塞主循环。
3. **分阶段并行收集**：
   - **Phase A - userInputAttachments**：依赖用户输入的附件（`@提及文件`、MCP 资源、Agent 提及、技能发现）。**必须先完成**，因为 `@提及文件` 会填充 `nestedMemoryAttachmentTriggers`。
   - **Phase B - allThreadAttachments**：线程安全附件（队列命令、日期变更、变更文件、嵌套内存、动态技能、计划模式、队友邮箱等）。主线程和子 Agent 均可获取。
   - **Phase C - mainThreadAttachments**：仅主线程可用的附件（IDE 选区、诊断、LSP、用量统计等）。
4. **错误隔离**：每个子附件收集器通过 `maybe()` 包装，单点失败只记录日志并返回 `[]`，不影响其他附件。

### 3.3 `@提及文件` 解析：`extractAtMentionedFiles()`

支持两种语法：

```ts
const quotedAtMentionRegex = /(^|\s)@"([^"]+)"/g   // @"path with spaces.txt"
const regularAtMentionRegex = /(^|\s)@([^\s]+)\b/g // @file.txt#L10-20
```

- 行号通过 `parseAtMentionedFileLines()` 解析：`file.txt#L10-20`
- 若路径是目录，则调用 `readdir` 生成 `DirectoryAttachment`（最多 1000 条目）
- 文件读取走 `generateFileAttachment()` → `FileReadTool.call()`

### 3.4 文件附件生成：`generateFileAttachment()`

```ts
export async function generateFileAttachment(
  filename: string,
  toolUseContext: ToolUseContext,
  successEventName: string,
  errorEventName: string,
  mode: 'compact' | 'at-mention',
  options?: { offset?: number; limit?: number }
)
```

关键逻辑：

1. **权限检查**：`isFileReadDenied()` 基于 `toolPermissionContext` 的 deny 规则。
2. **大小检查**：`at-mention` 模式下超过 `maxSizeBytes` 的非 PDF 文件直接拒绝。
3. **PDF 优化**：`tryGetPDFReference()` 对超过 `PDF_AT_MENTION_INLINE_THRESHOLD` 页的 PDF 返回轻量级引用而非全文。
4. **缓存命中**：若 `readFileState` 中存在且 `mtime` 未变，返回 `AlreadyReadFileAttachment`，避免重复上传大文件到 API。
5. **截断回退**：若 `FileReadTool.call()` 抛出 `MaxFileReadTokenExceededError` 或 `FileTooLargeError`，回退到只读前 `MAX_LINES_TO_READ` 行并标记 `truncated: true`。

### 3.5 嵌套内存加载：`getNestedMemoryAttachmentsForFile()`

加载顺序（优先级由低到高，后加载的覆盖先加载的）：

1. **Managed/User 条件规则**（`getManagedAndUserConditionalRules`）
2. **嵌套目录**（从 CWD 到目标文件目录）：`CLAUDE.md` + 无条件规则 + 条件规则
3. **CWD 级目录**（从根到 CWD）：仅条件规则

去重机制：
- `processedPaths`: 本次加载的 Set 去重
- `loadedNestedMemoryPaths`: Session 级 Set 去重（防止 LRU 驱逐后重复注入）
- `readFileState.has()`: 跨回合/跨工具调用去重

### 3.6 相关记忆预取：`startRelevantMemoryPrefetch()`

```ts
export type MemoryPrefetch = {
  promise: Promise<Attachment[]>
  settledAt: number | null
  consumedOnIteration: number
  [Symbol.dispose](): void
}
```

设计要点：
- **非阻塞**：在 `query.ts` 的主循环中，预取与模型流式响应/工具执行并行。
- **可丢弃**：通过 `using` 绑定，`[Symbol.dispose]` 在循环退出时自动 abort 并记录 telemetry。
- **消费条件**：仅在 `settledAt !== null && consumedOnIteration === -1` 时消费；若本轮未就绪，下轮再试。
- **去重**：`filterDuplicateMemoryAttachments()` 在消费时排除已被 `FileRead/Write/Edit` 加载到上下文的记忆。

### 3.7 变更文件检测：`getChangedFiles()`

```ts
export async function getChangedFiles(toolUseContext: ToolUseContext): Promise<Attachment[]>
```

- 遍历 `readFileState` 的所有 key
- 跳过带 `offset/limit` 的文件（TODO：尚未支持范围变更检测）
- 比较 `mtimeMs` 与缓存的 `timestamp`
- 文本文件用 `getSnippetForTwoFileDiff()` 生成 diff 摘要
- **仅 ENOENT 时驱逐缓存**；其他 IO 错误（EACCES、原子写竞争）保留缓存，避免误删导致后续 Edit 失败

### 3.8 技能列表管理：`getSkillListingAttachments()`

- 模块级 `sentSkillNames: Map<string, Set<string>>` 按 `agentId`（空字符串为主线程）隔离，确保子 Agent 有自己独立的 turn-0 列表。
- `resetSentSkillNames()`：在 `/reload-plugins`、技能文件变更、`clear caches` 时调用，强制重新广播。
- `suppressNextSkillListing()`：`--resume` 恢复会话时调用，避免重复注入已存在的技能列表。
- `filterToBundledAndMcp()`：当 `EXPERIMENTAL_SKILL_SEARCH` 开启时，仅保留内置 + MCP 技能，防止 200+ 用户技能撑爆上下文。

### 3.9 计划/自动模式附件的节流算法

以 `plan_mode` 为例：

```ts
function getPlanModeAttachmentTurnCount(messages: Message[]): {
  turnCount: number
  foundPlanModeAttachment: boolean
}
```

- **从后向前**扫描消息历史
- 只计 **人类回合**（`type === 'user' && !isMeta && !hasToolResultContent`），不计 assistant/tool 轮次
- 若自上次 `plan_mode` 附件以来已满 5 个人类回合，且当前仍在 `plan` 模式，则生成新附件
- `countPlanModeAttachmentsSinceLastExit()` 控制 full/sparse 交替周期（每第 1、6、11…次为 full）

`auto_mode` 逻辑类似，额外支持 `auto_mode_exit` 重置计数器。

### 3.10 队列命令附件：`getQueuedCommandAttachments()`

```ts
const INLINE_NOTIFICATION_MODES = new Set(['prompt', 'task-notification'])
```

- 过滤 `queuedCommands` 中 mode 为 `prompt` 或 `task-notification` 的命令
- 将粘贴的图片内容通过 `buildImageContentBlocks()` 转为 `ImageBlockParam[]`
- 生成 `queued_command` 附件，保留 `origin`、`isMeta`、`commandMode` 等元数据

---

## 四、关键代码路径与文件引用

### 4.1 直接调用方（按调用频率/重要性排序）

| 调用方文件 | 调用符号 | 作用 |
|------------|----------|------|
| `src/utils/processUserInput/processUserInput.ts` | `getAttachmentMessages()` | 用户提交时生成初始附件 |
| `src/query.ts` | `getAttachmentMessages()`, `createAttachmentMessage()`, `filterDuplicateMemoryAttachments()`, `startRelevantMemoryPrefetch()` | 工具轮次结束后的附件注入、记忆预取消费 |
| `src/services/compact/compact.ts` | `createAttachmentMessage()`, `generateFileAttachment()`, `getAgentListingDeltaAttachment()`, `getDeferredToolsDeltaAttachment()`, `getMcpInstructionsDeltaAttachment()` | Compaction 后恢复附件 |
| `src/utils/hooks.ts` | `createAttachmentMessage()` | 各种 Hook 结果包装 |
| `src/services/tools/toolHooks.ts` | `createAttachmentMessage()` | 工具 Hook 结果包装 |
| `src/services/tools/toolExecution.ts` | `createAttachmentMessage()` | 工具执行中的特殊消息（如 max turns） |
| `src/screens/REPL.tsx` | `createAttachmentMessage()`, `getQueuedCommandAttachments()` | REPL 通知渲染 |
| `src/tools/AgentTool/runAgent.ts` | `createAttachmentMessage()` | 子 Agent 上下文注入 |
| `src/utils/sessionStart.ts` | `createAttachmentMessage()` | 会话启动上下文 |
| `src/utils/processUserInput/processSlashCommand.tsx` | `getAttachmentMessages()`, `createAttachmentMessage()` | 斜杠命令中的附件提取 |

### 4.2 被调用方 / 强依赖文件

| 依赖文件 | 用途 |
|----------|------|
| `src/tools/FileReadTool/FileReadTool.ts` | 实际文件读取、验证、图片压缩 |
| `src/utils/claudemd.ts` | CLAUDE.md 发现、条件规则解析、内存文件加载 |
| `src/utils/fileStateCache.ts` | `FileStateCache`（LRU 缓存，跨回合去重） |
| `src/utils/task/framework.ts` | 统一任务附件生成（`generateTaskAttachments`） |
| `src/utils/teammateMailbox.ts` | Agent Swarm 的文件邮箱读写 |
| `src/services/skillSearch/prefetch.ts` | 技能发现预取（`getTurnZeroSkillDiscovery`） |
| `src/services/compact/autoCompact.ts` | 自动压缩状态、上下文窗口计算 |
| `src/utils/messages.ts` | `isThinkingMessage`, `getUserMessageText`, `isHumanTurn` |
| `src/utils/permissions/filesystem.ts` | `matchingRuleForInput`, `pathInAllowedWorkingPath` |
| `src/services/diagnosticTracking.ts` | IDE 诊断追踪器 |
| `src/services/lsp/LSPDiagnosticRegistry.ts` | LSP 诊断注册表 |
| `src/utils/imageResizer.ts` | 粘贴图片的 resize 与 downsample |
| `src/utils/pdf.ts` | PDF 页数统计 |

### 4.3 类型定义

- `Attachment` 及所有子类型：定义在 `src/utils/attachments.ts` 内
- `AttachmentMessage`：定义在 `src/types/message.ts`（或等效文件）
- `ToolUseContext`：定义在 `src/Tool.ts`
- `FileStateCache`：定义在 `src/utils/fileStateCache.ts`

### 4.4 渲染侧

- `src/components/messages/AttachmentMessage.tsx`：附件的 UI 渲染
- `src/components/messages/nullRenderingAttachments.ts`：定义了约 45 种**不渲染**的附件类型（如 `plan_mode`、`hook_success`、`budget_usd` 等），这些附件仅作为 LLM 上下文存在，不占用消息渲染预算

---

## 五、依赖与外部交互

### 5.1 运行时环境依赖

- **Node.js/Bun**：使用 `crypto.randomUUID()`、`fs/promises`（`stat`, `readdir`）
- **`bun:bundle` 的 `feature()`**：大量条件编译门控（`EXPERIMENTAL_SKILL_SEARCH`、`KAIROS`、`TRANSCRIPT_CLASSIFIER`、`BUDDY`、`HISTORY_SNIP` 等），未启用的特性在构建时通过 DCE（Dead Code Elimination）剔除
- **环境变量**：
  - `CLAUDE_CODE_DISABLE_ATTACHMENTS` / `CLAUDE_CODE_SIMPLE`：禁用附件
  - `CLAUDE_CODE_ENABLE_TOKEN_USAGE_ATTACHMENT`：启用 token 用量附件
  - `CLAUDE_CODE_VERIFY_PLAN`：启用计划验证提醒
  - `USER_TYPE === 'ant'`：控制 Ant 内部功能（如 teammate mailbox、verify plan reminder）

### 5.2 外部服务 / API 交互

- **Anthropic API（间接）**：附件最终通过 `normalizeMessagesForAPI()` 进入 API 请求；`FileReadTool.call()` 内部可能触发图片压缩和 token 估算
- **MCP 客户端**：`processMcpResourceAttachments()` 调用 `client.client.readResource()` 读取 MCP 资源
- **Growthbook（特性开关）**：`getFeatureValue_CACHED_MAY_BE_STALE()` 控制多个附件门控（如 `tengu_moth_copse` 控制记忆预取、`tengu_paper_halyard` 控制 Project 级内存跳过）
- **Telemetry**：大量 `logEvent()` 调用用于附件生成时长、成功率、大小的埋点

### 5.3 数据流总览

```
用户输入 / 工具结果
        ↓
processUserInput.ts / query.ts
        ↓
getAttachmentMessages() ──→ getAttachments()
        ↓
┌─────────────────┬─────────────────┬─────────────────┐
│  userInput      │  allThread      │  mainThread     │
│  (并行，先完成)  │  (并行)          │  (仅主线程)      │
└─────────────────┴─────────────────┴─────────────────┘
        ↓
  createAttachmentMessage()
        ↓
  AttachmentMessage (类型)
        ↓
  进入消息流 → normalizeMessagesForAPI() → LLM
```

---

## 六、风险、边界与改进建议

### 6.1 已知风险

#### R1：1000ms 硬超时可能导致附件静默丢失
`getAttachments()` 中设置了 `setTimeout(ac => ac.abort(), 1000, abortController)`。虽然 `maybe()` 会捕获 abort 错误并返回 `[]`，但某些附件（如 `@提及文件`、MCP 资源读取）若刚好卡在 1s 边界，会**无提示地丢失**，用户可能困惑为何 `@文件` 未生效。

#### R2：`readFileState` 的 LRU 驱逐导致变更检测失效
`FileStateCache` 默认 100 条目。在大型代码库中频繁读取不同文件时，LRU 可能驱逐旧条目。虽然 `getChangedFiles()` 只检测缓存中的文件，但驱逐意味着该文件不再被监控变更。对于长会话中的大项目，这可能导致**编辑后的文件变更提醒遗漏**。

#### R3：嵌套内存与 `getChangedFiles` 的 `isPartialView` 交互复杂
当 `CLAUDE.md` 被截断或剥离 frontmatter 后注入时，`readFileState` 中标记 `isPartialView: true` 并存储原始磁盘内容。`getChangedFiles` 用原始内容做 diff，但 `memoryFilesToAttachments()` 中设置 `offset: undefined, limit: undefined`，而 `getChangedFiles()` 明确跳过任何带 `offset/limit` 的条目。当前逻辑恰好不设置这两个字段，所以能工作，但**很容易因后续改动而破坏**。

#### R4：`sentSkillNames` 是模块级状态，无法跨进程恢复
`suppressNextSkillListing()` 仅能在当前进程内抑制一次。若进程崩溃重启后恢复会话，`sentSkillNames` 为空，可能导致技能列表重复注入（虽然 `conversationRecovery.ts` 会调用 `suppressNextSkillListing()`，但那是针对已存在附件的恢复会话）。

#### R5：Agent Swarm 的 teammate mailbox 存在竞态条件
`getTeammateMailboxAttachments()` 同时读取文件邮箱和 `AppState.inbox`，并通过 `from+timestamp+text 前缀` 做去重。若两条消息内容前 100 字符相同但后续不同，会发生**误去重**。此外，`markMessagesAsReadByPredicate` 与 `useInboxPoller` 的并发读写依赖文件锁，但锁的超时仅 100ms，高并发下可能失败。

#### R6：`getDirectoriesToProcess()` 的目录遍历在 Windows 上可能有边界问题
函数使用 `currentDir.startsWith(originalCwd)` 判断目录是否在 CWD 下，Windows 上路径大小写不敏感，可能导致**意外的目录包含或排除**。

### 6.2 边界条件

| 边界 | 行为 |
|------|------|
| `@提及` 目录条目 > 1000 | 截断并追加 `… and N more entries` |
| 记忆文件 > 200 行或 > 4096 字节 | `readMemoriesForSurfacing()` 截断并附加提示 |
| 会话累计记忆 > 60KB | `startRelevantMemoryPrefetch()` 直接返回 `undefined` |
| 技能列表（bundled+MCP）> 30 | 回退到仅 bundled |
| PDF 页数 > `PDF_AT_MENTION_INLINE_THRESHOLD` | 返回 `PDFReferenceAttachment` 而非全文 |
| 文件在 `readFileState` 中且 mtime 未变 | 返回 `AlreadyReadFileAttachment`，API 侧不重复发送 |
| 计划模式附件 | 仅当 `permissionContext.mode === 'plan'` 且距上次已满 5 人类回合时触发 |

### 6.3 改进建议

#### S1：将 1000ms 硬超时改为可配置或分级超时
- 对 I/O 密集型附件（文件读取、MCP、技能发现）使用更宽松的预算（如 3000ms）
- 或改为 Promise.race 后记录 `tengu_attachment_timeout` 事件，并在 UI 侧给出 "部分附件加载超时" 的提示

#### S2：为 `getChangedFiles()` 引入持久化变更追踪
- 当前依赖 `readFileState` 的 LRU，大项目下不可靠
- 可考虑维护一个独立的 `watchedFiles: Map<path, mtimeMs>`，不受 LRU 驱逐影响，专门用于变更检测

#### S3：统一 `@提及` 解析器
- 当前 `extractAtMentionedFiles()`、`extractAgentMentions()`、`extractMcpResourceMentions()` 使用三套正则，维护成本高
- 建议引入统一的 tokenizer/lexer，支持更复杂的语法（如转义、嵌套引号）

#### S4：将 `sentSkillNames` 等模块级状态持久化到会话存储
- 与 `conversationRecovery.ts` 协作，在 `--resume` 时恢复 `sentSkillNames` 和 `suppressNext` 状态
- 避免进程重启后的重复注入

#### S5：为 `teammate_mailbox` 引入更稳健的去重键
- 使用消息内容的完整哈希（如 SHA-256 前 16 字节）替代 `text.slice(0, 100)`
- 或引入服务器端/协调端的消息 ID

#### S6：增加 `getAttachments()` 的单元测试覆盖
- 当前仓库中未发现针对 `attachments.ts` 的 `.test.ts` 或 `.spec.ts` 文件
- 关键函数（`getDateChangeAttachments`、`collectRecentSuccessfulTools`、`filterDuplicateMemoryAttachments`、`getVerifyPlanReminderTurnCount`、`memoryFilesToAttachments`）已标记 `Exported for testing`，但缺少实际测试
- 建议补充：
  - 计划模式 / 自动模式的回合计数测试
  - `filterDuplicateMemoryAttachments` 的自引用过滤回归测试
  - `getChangedFiles` 的原子写竞争、ENOENT 驱逐测试
  - `extractAtMentionedFiles` 的边界语法测试

#### S7：文档化附件类型的生命周期
- 新附件类型需要在至少 4 个地方同步更新：`Attachment` 联合类型、`nullRenderingAttachments.ts`、`AttachmentMessage.tsx` 的 switch、以及可能的消息序列化边界
- 建议增加一个 `ADDING_NEW_ATTACHMENT.md` 的 checklist，降低新增附件的遗漏风险

---

## 七、结论

`src/utils/attachments.ts` 是 Claude Code 上下文注入系统的**心脏**，以 ~4000 行代码协调了 50 余种附件来源。其设计亮点包括：

1. **分阶段并行收集**：通过 `userInput` → `thread` → `mainThread` 的三阶段架构，在保证依赖顺序的同时最大化并发。
2. **多层去重机制**：Session 级 Set、LRU 缓存、消息历史扫描、累积字节预算共同防止上下文膨胀。
3. **非阻塞预取**：记忆预取和技能发现预取与主模型调用并行，不阻塞工具循环。
4. **错误隔离**：`maybe()` 包装确保单个附件失败不会拖垮整个回合。

主要风险集中在**硬超时导致的静默丢失**、**LRU 驱逐对变更检测的影响**、以及**模块级状态在跨进程恢复时的不一致**。建议通过增加持久化追踪、分级超时和补充单元测试来进一步提升可靠性。
