# Speculation Service 研究文档

## 文件信息
- **文件路径**: `src/services/PromptSuggestion/speculation.ts`
- **文件大小**: 30,680 bytes
- **研究日期**: 2026-04-01

---

## 一、场景与职责

### 1.1 核心场景
Speculation（推测执行）是 Claude Code CLI 的**预测性执行系统**，旨在：
1. **零延迟响应**: 当用户接受提示建议时，已经预先执行了部分工作
2. **提升感知性能**: 用户看到"即时"完成的效果，实际工作在后台已进行
3. **流水线化建议**: 在当前推测执行完成时，预生成下一个建议

### 1.2 主要职责
- **推测执行管理**: 启动、监控、接受/中止推测执行会话
- **文件系统隔离**: 使用 overlay 目录隔离推测执行的文件修改
- **工具权限控制**: 严格限制推测执行期间允许的工具类型
- **消息准备与注入**: 将推测执行的消息注入主对话流程
- **时间节省追踪**: 计算并记录推测执行节省的时间
- **流水线建议**: 在推测执行期间预生成下一个提示建议

---

## 二、功能点目的

### 2.1 推测执行启动 (`startSpeculation`)
**目的**: 基于用户接受的提示建议，在后台预执行可能的用户意图

**关键设计决策**:
- 创建隔离的 overlay 文件系统，防止推测执行污染主工作区
- 使用 `runForkedAgent` 运行独立的 agent 循环
- 限制为只读工具和安全工具，遇到危险操作时停止
- 支持最大 20 轮对话和 100 条消息的限制

### 2.2 文件系统隔离 (`getOverlayPath`, `copyOverlayToMain`)
**目的**: 确保推测执行的文件修改不会直接影响主工作区

**机制**:
- Overlay 路径: `{tempDir}/speculation/{pid}/{id}/`
- Copy-on-write: 首次写入时复制原文件到 overlay
- 接受时合并: 将 overlay 中的修改复制回主工作区

### 2.3 工具权限控制 (`canUseTool` 回调)
**目的**: 限制推测执行期间允许的工具，防止危险操作

**工具分类**:
| 类别 | 工具 | 处理方式 |
|------|------|----------|
| **写工具** | Edit, Write, NotebookEdit | 检查权限模式，拒绝或停止 |
| **安全只读** | Read, Glob, Grep, ToolSearch, LSP, TaskGet, TaskList | 允许，路径重写到 overlay |
| **Bash** | Bash | 检查是否为只读命令 |
| **其他** | 所有其他工具 | 默认拒绝 |

**停止边界类型** (`CompletionBoundary`):
- `edit`: 文件编辑需要权限
- `bash`: 非只读 bash 命令
- `denied_tool`: 被拒绝的工具
- `complete`: 自然完成

### 2.4 消息准备 (`prepareMessagesForInjection`)
**目的**: 清理推测执行的消息，准备注入主对话

**清理操作**:
- 移除 `thinking` 和 `redacted_thinking` 块
- 移除未完成的 `tool_use` 和对应的 `tool_result`
- 移除中断消息
- 过滤掉仅包含空白的内容块

### 2.5 推测执行接受 (`acceptSpeculation`, `handleSpeculationAccept`)
**目的**: 将推测执行的结果合并到主对话流程

**流程**:
1. 停止推测执行
2. 将 overlay 文件复制到主工作区
3. 清理并注入消息到主对话
4. 更新 file state cache
5. 如果完成，提升流水线建议并启动下一轮推测

### 2.6 流水线建议 (`generatePipelinedSuggestion`)
**目的**: 在推测执行期间预生成下一个提示建议

**时机**: 当推测执行自然完成时 (`boundary.type === 'complete'`)

### 2.7 遥测追踪 (`logSpeculation`)
**事件类型**:
- `tengu_speculation`: 记录推测执行结果
  - `outcome`: `accepted` | `aborted` | `error`
  - `duration_ms`: 执行时长
  - `tools_executed`: 执行的工具数
  - `completed`: 是否完成
  - `boundary_type`: 停止边界类型
  - `boundary_tool`: 触发边界的工具

---

## 三、具体技术实现

### 3.1 关键流程

#### 3.1.1 推测执行启动流程 (`startSpeculation`)
```
1. 检查 isSpeculationEnabled() (ant-only + config 开关)
2. 中止任何现有的推测执行
3. 生成唯一 ID (randomUUID slice)
4. 创建 overlay 目录
5. 初始化状态 (messagesRef, writtenPathsRef, contextRef)
6. 设置 AppState.speculation 为 active
7. 调用 runForkedAgent 启动推测执行
   a. 使用 canUseTool 回调控制工具权限
   b. 使用 onMessage 回调追踪消息
   c. 设置 maxTurns = 20
8. 完成后启动流水线建议生成
```

#### 3.1.2 工具权限检查流程 (`canUseTool` 回调)
```
1. 检查是否为写工具 (Edit/Write/NotebookEdit)
   a. 检查权限模式 (acceptEdits/bypassPermissions/plan+bypass)
   b. 如果不允许，设置 edit 边界并中止
2. 检查是否为安全只读工具
   a. 路径重写: 已写入文件读取 overlay，否则读取原路径
   b. 写操作: Copy-on-write 到 overlay
3. 检查是否为 Bash 工具
   a. 检查是否为只读命令 (checkReadOnlyConstraints)
   b. 如果不是，设置 bash 边界并中止
4. 其他工具: 设置 denied_tool 边界并中止
```

#### 3.1.3 推测执行接受流程 (`handleSpeculationAccept`)
```
1. 清除提示建议状态
2. 立即注入用户消息 (视觉反馈)
3. 调用 acceptSpeculation
   a. 停止推测执行
   b. 复制 overlay 文件到主工作区 (如果有消息)
   c. 清理 overlay 目录
   d. 计算 timeSavedMs
4. 准备消息 (prepareMessagesForInjection)
5. 如果不完整，移除尾部 assistant 消息
6. 注入推测执行的消息
7. 更新 readFileState cache
8. 如果完成且有流水线建议，提升为当前建议并启动新推测
9. 返回 { queryRequired: !isComplete }
```

### 3.2 数据结构

#### 3.2.1 SpeculationState (AppState 中的状态)
```typescript
type SpeculationState =
  | { status: 'idle' }
  | {
      status: 'active'
      id: string
      abort: () => void
      startTime: number
      messagesRef: { current: Message[] }  // 可变引用，避免数组展开
      writtenPathsRef: { current: Set<string> }  // 相对路径集合
      boundary: CompletionBoundary | null
      suggestionLength: number
      toolUseCount: number
      isPipelined: boolean
      contextRef: { current: REPLHookContext }
      pipelinedSuggestion?: {
        text: string
        promptId: 'user_intent' | 'stated_intent'
        generationRequestId: string | null
      } | null
    }
```

#### 3.2.2 CompletionBoundary (完成边界)
```typescript
type CompletionBoundary =
  | { type: 'complete'; completedAt: number; outputTokens: number }
  | { type: 'bash'; command: string; completedAt: number }
  | { type: 'edit'; toolName: string; filePath: string; completedAt: number }
  | { type: 'denied_tool'; toolName: string; detail: string; completedAt: number }
```

#### 3.2.3 SpeculationResult (接受结果)
```typescript
type SpeculationResult = {
  messages: Message[]
  boundary: CompletionBoundary | null
  timeSavedMs: number
}
```

### 3.3 关键常量

```typescript
const MAX_SPECULATION_TURNS = 20        // 最大对话轮数
const MAX_SPECULATION_MESSAGES = 100    // 最大消息数

const WRITE_TOOLS = new Set(['Edit', 'Write', 'NotebookEdit'])
const SAFE_READ_ONLY_TOOLS = new Set([
  'Read', 'Glob', 'Grep', 'ToolSearch', 'LSP', 'TaskGet', 'TaskList'
])
```

### 3.4 路径重写机制

#### 3.4.1 写操作 (Copy-on-write)
```typescript
if (isWriteTool) {
  if (!writtenPathsRef.current.has(rel)) {
    // 首次写入: 复制原文件到 overlay
    const overlayFile = join(overlayPath, rel)
    await mkdir(dirname(overlayFile), { recursive: true })
    try {
      await copyFile(join(cwd, rel), overlayFile)
    } catch {
      // 原文件可能不存在（新文件创建）
    }
    writtenPathsRef.current.add(rel)
  }
  // 重写到 overlay 路径
  input = { ...input, [pathKey]: join(overlayPath, rel) }
}
```

#### 3.4.2 读操作 (Redirect)
```typescript
if (writtenPathsRef.current.has(rel)) {
  // 文件已被写入，从 overlay 读取
  input = { ...input, [pathKey]: join(overlayPath, rel) }
}
// 否则从原路径读取（无重写）
```

### 3.5 消息清理算法

```typescript
// 1. 找出有成功结果的 tool_use IDs
const toolIdsWithSuccessfulResults = new Set(
  messages
    .flatMap(m => m.message.content)
    .filter(b => b.type === 'tool_result' && !b.is_error)
    .map(b => b.tool_use_id)
)

// 2. 过滤条件
const keep = (b) =>
  b.type !== 'thinking' &&
  b.type !== 'redacted_thinking' &&
  !(b.type === 'tool_use' && !toolIdsWithSuccessfulResults.has(b.id)) &&
  !(b.type === 'tool_result' && !toolIdsWithSuccessfulResults.has(b.tool_use_id)) &&
  !(b.type === 'text' && b.text === INTERRUPT_MESSAGE)

// 3. 应用过滤并移除空消息
return messages
  .map(msg => filterContent(msg, keep))
  .filter(m => m !== null)
```

---

## 四、关键代码路径与文件引用

### 4.1 核心函数调用链

```
startSpeculation (推测执行入口)
├── abortSpeculation (中止现有)
├── mkdir (创建 overlay 目录)
├── runForkedAgent
│   ├── createSubagentContext
│   └── query
│       └── canUseTool (回调)
│           ├── 写工具检查 -> denySpeculation / 设置 edit 边界
│           ├── 安全只读工具 -> 路径重写
│           ├── Bash 检查 -> checkReadOnlyConstraints
│           └── 其他工具 -> denySpeculation
│       └── onMessage (回调)
│           └── updateActiveSpeculationState
└── generatePipelinedSuggestion (完成后)
    └── generateSuggestion (from promptSuggestion.ts)

handleSpeculationAccept (接受推测执行)
├── createUserMessage (立即注入用户消息)
├── acceptSpeculation
│   ├── abort()
│   ├── copyOverlayToMain (文件复制)
│   ├── safeRemoveOverlay (清理)
│   └── logSpeculation
├── prepareMessagesForInjection (消息清理)
├── setMessages (注入推测消息)
├── mergeFileStateCaches (更新 cache)
└── 如果完成: startSpeculation (新推测)

abortSpeculation (中止推测执行)
├── logSpeculation
├── abort()
└── safeRemoveOverlay
```

### 4.2 文件依赖关系

```
speculation.ts
├── 依赖
│   ├── ../../bootstrap/state.js (getCwdState)
│   ├── ../../state/AppStateStore.js (AppState, SpeculationState, etc.)
│   ├── ../../tools/BashTool/bashPermissions.js (commandHasAnyCd)
│   ├── ../../tools/BashTool/readOnlyValidation.js (checkReadOnlyConstraints)
│   ├── ../../types/logs.js (SpeculationAcceptMessage)
│   ├── ../../types/message.js (Message)
│   ├── ../../utils/abortController.js (createChildAbortController)
│   ├── ../../utils/array.js (count)
│   ├── ../../utils/config.js (getGlobalConfig)
│   ├── ../../utils/debug.js (logForDebugging)
│   ├── ../../utils/errors.js (errorMessage)
│   ├── ../../utils/fileStateCache.js (FileStateCache, mergeFileStateCaches)
│   ├── ../../utils/forkedAgent.js (runForkedAgent, createCacheSafeParams)
│   ├── ../../utils/format.js (formatDuration, formatNumber)
│   ├── ../../utils/hooks/postSamplingHooks.js (REPLHookContext)
│   ├── ../../utils/log.js (logError)
│   ├── ../../utils/messageQueueManager.js (SetAppState)
│   ├── ../../utils/messages.js (createSystemMessage, createUserMessage)
│   ├── ../../utils/permissions/filesystem.js (getClaudeTempDir)
│   ├── ../../utils/queryHelpers.js (extractReadFilesFromMessages)
│   ├── ../../utils/sessionStorage.js (getTranscriptPath)
│   ├── ../../utils/slowOperations.js (jsonStringify)
│   ├── ../analytics/index.js (logEvent)
│   └── ./promptSuggestion.js (generateSuggestion, getPromptVariant, etc.)
└── 被依赖
    ├── ./promptSuggestion.ts (executePromptSuggestion 中调用)
    └── REPL/UI 层 (接受/中止操作)
```

### 4.3 关键代码位置

| 功能 | 行号 | 说明 |
|------|------|------|
| `isSpeculationEnabled` | 337-343 | 功能开关检查 |
| `startSpeculation` | 402-715 | 推测执行启动主函数 |
| `acceptSpeculation` | 717-800 | 接受推测执行 |
| `abortSpeculation` | 802-833 | 中止推测执行 |
| `handleSpeculationAccept` | 835-991 | 处理接受操作（UI 层调用） |
| `prepareMessagesForInjection` | 203-271 | 消息清理准备 |
| `generatePipelinedSuggestion` | 345-400 | 流水线建议生成 |
| `copyOverlayToMain` | 99-117 | 文件复制到主工作区 |
| `getOverlayPath` | 80-82 | 获取 overlay 路径 |
| `updateActiveSpeculationState` | 310-328 | 状态更新辅助函数 |
| `logSpeculation` | 124-153 | 遥测记录 |
| `countToolsInMessages` | 155-164 | 工具使用计数 |

---

## 五、依赖与外部交互

### 5.1 外部服务依赖

| 服务 | 用途 | 关键交互 |
|------|------|----------|
| **Anthropic API** | 推测执行 agent 循环 | 通过 `runForkedAgent` -> `query` |
| **Analytics** | 遥测追踪 | `logEvent('tengu_speculation', ...)` |
| **文件系统** | Overlay 目录操作 | `mkdir`, `copyFile`, `rm` |

### 5.2 内部模块依赖

| 模块 | 依赖内容 | 用途 |
|------|----------|------|
| `forkedAgent.ts` | `runForkedAgent`, `createCacheSafeParams` | 创建隔离的推测执行 agent |
| `AppStateStore.ts` | `AppState`, `SpeculationState`, `CompletionBoundary` | 状态类型定义 |
| `readOnlyValidation.ts` | `checkReadOnlyConstraints` | Bash 命令只读检查 |
| `fileStateCache.ts` | `FileStateCache`, `mergeFileStateCaches` | 文件状态缓存管理 |
| `promptSuggestion.ts` | `generateSuggestion`, `shouldFilterSuggestion` | 建议生成和过滤 |
| `bashPermissions.ts` | `commandHasAnyCd` | 检测 cd 命令 |

### 5.3 环境变量依赖

| 变量 | 用途 |
|------|------|
| `USER_TYPE` | 区分 ant/external，影响调试日志和 feedback 消息 |
| `CLAUDE_CODE_SPECULATION_ENABLED` | 推测执行开关（通过 getGlobalConfig） |

---

## 六、风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 文件系统竞态条件
- **风险**: 用户在接受推测执行前修改了原文件，导致 overlay 的 copy-on-write 基于过时版本
- **缓解**: 推测执行通常很快完成，窗口期短
- **建议**: 添加文件修改时间检查，检测冲突

#### 6.1.2 无限推测循环
- **风险**: 流水线建议可能导致无限推测执行链
- **缓解**: `isPipelined` 标志追踪，防止递归过深
- **边界**: 依赖 `MAX_SPECULATION_TURNS` 和 `MAX_SPECULATION_MESSAGES` 限制

#### 6.1.3 存储空间泄漏
- **风险**: Overlay 目录可能因崩溃未清理
- **缓解**: `safeRemoveOverlay` 使用 `maxRetries` 和 `retryDelay`
- **建议**: 启动时清理孤儿 overlay 目录

#### 6.1.4 权限绕过风险
- **风险**: 推测执行期间的工具权限检查可能与主流程不一致
- **缓解**: `canUseTool` 回调显式检查权限模式
- **边界**: 复杂的权限规则可能未完全覆盖

#### 6.1.5 消息注入顺序错误
- **风险**: 推测消息注入顺序错误导致对话历史不一致
- **缓解**: `prepareMessagesForInjection` 严格清理和排序
- **边界**: 边界情况（如中断）可能处理不完善

### 6.2 边界条件

| 边界 | 行为 |
|------|------|
| 非 ant 用户 | 推测执行完全禁用 |
| `speculationEnabled: false` | 功能禁用 |
| 已有活跃推测 | 先中止现有，再启动新推测 |
| 达到 MAX_SPECULATION_TURNS (20) | 自动中止 |
| 达到 MAX_SPECULATION_MESSAGES (100) | 自动中止 |
| 用户输入时 | 中止当前推测执行 |
| 推测未完成 | 截断到最后的非 assistant 消息 |
| Overlay 目录创建失败 | 静默返回，不启动推测 |

### 6.3 改进建议

#### 6.3.1 文件冲突检测
```typescript
// 建议: 在接受前检查文件是否被修改
async function detectFileConflicts(
  writtenPaths: Set<string>,
  overlayPath: string,
  cwd: string
): Promise<string[]> {
  const conflicts: string[] = []
  for (const rel of writtenPaths) {
    const overlayStat = await stat(join(overlayPath, rel)).catch(() => null)
    const mainStat = await stat(join(cwd, rel)).catch(() => null)
    if (overlayStat && mainStat && overlayStat.mtime < mainStat.mtime) {
      conflicts.push(rel)
    }
  }
  return conflicts
}
```

#### 6.3.2 推测执行预览
- 当前: 用户看不到推测执行进度
- 建议: 可选的轻量级 UI 指示器显示推测执行状态

#### 6.3.3 智能边界预测
- 当前: 在编辑/bash 边界停止
- 建议: 基于用户历史行为预测哪些操作应该预执行

#### 6.3.4 多层级推测
- 当前: 单一层级推测执行
- 建议: 支持推测执行的推测执行（需要仔细管理资源）

#### 6.3.5 推测执行回滚
- 当前: 接受后无法回滚
- 建议: 添加 /undo-speculation 命令或确认对话框

#### 6.3.6 性能优化
```typescript
// 建议: 推测执行期间降低模型复杂度
const speculationModelConfig = {
  ...mainConfig,
  thinkingConfig: { budget_tokens: 1024 }, // 降低思考预算
  maxOutputTokens: 4096, // 限制输出长度
}
```

### 6.4 测试建议

| 测试类型 | 覆盖点 |
|----------|--------|
| 单元测试 | 路径重写逻辑、消息清理算法 |
| 集成测试 | 与 forkedAgent 的交互、文件系统操作 |
| 竞态测试 | 用户快速输入、多次接受/中止 |
| 边界测试 | MAX_SPECULATION_TURNS/MESSAGES 限制 |
| 权限测试 | 各种权限模式下的工具行为 |
| 恢复测试 | 崩溃后 overlay 目录清理 |

### 6.5 监控建议

| 指标 | 用途 |
|------|------|
| `speculation_accept_rate` | 接受率趋势 |
| `speculation_completion_rate` | 自然完成 vs 边界停止比例 |
| `speculation_time_saved_avg` | 平均节省时间 |
| `speculation_file_write_count` | 平均文件修改数 |
| `speculation_boundary_distribution` | 边界类型分布 |
