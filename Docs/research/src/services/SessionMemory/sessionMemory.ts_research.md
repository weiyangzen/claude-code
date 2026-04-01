# SessionMemory/sessionMemory.ts 研究文档

## 场景与职责

`sessionMemory.ts` 是 Session Memory 系统的核心协调模块，负责：
1. **初始化与注册**：在应用启动时注册 post-sampling hook
2. **触发判断**：基于 token 阈值和工具调用次数决定是否提取记忆
3. **Forked Agent 执行**：通过 `runForkedAgent` 在隔离上下文中执行记忆提取
4. **文件管理**：创建和管理会话记忆文件（`~/.claude/projects/{cwd}/{sessionId}/session-memory/summary.md`）
5. **手动提取**：支持 `/summary` 命令手动触发记忆提取

该模块是 Session Memory 系统的"大脑"，协调 prompts.ts（内容生成）、sessionMemoryUtils.ts（状态管理）和外部系统（forkedAgent、autoCompact 等）。

## 功能点目的

### 1. 自动记忆提取触发机制
基于双重阈值判断：
- **初始化阈值** (`minimumMessageTokensToInit`): 默认 10000 tokens，首次达到时初始化 session memory
- **更新阈值** (`minimumTokensBetweenUpdate`): 默认 5000 tokens，两次提取间的最小 token 增长
- **工具调用阈值** (`toolCallsBetweenUpdates`): 默认 3 次，确保有足够的新工具调用

触发条件（满足其一）：
- Token 阈值 AND 工具调用阈值同时满足
- Token 阈值满足 AND 上一轮 assistant 消息无工具调用（自然对话断点）

### 2. Forked Agent 执行记忆提取
使用 `runForkedAgent` 创建隔离的执行环境：
- **隔离上下文**：通过 `createSubagentContext` 防止污染父状态
- **缓存共享**：使用 `createCacheSafeParams` 确保与父查询共享 prompt cache
- **工具限制**：通过 `createMemoryFileCanUseTool` 仅允许编辑记忆文件

### 3. 远程配置集成
通过 GrowthBook 获取动态配置：
- `tengu_session_memory`: 功能开关
- `tengu_sm_config`: 阈值配置（初始化 token、更新 token、工具调用次数）

### 4. 手动记忆提取
`manuallyExtractSessionMemory` 函数供 `/summary` 命令调用：
- 跳过阈值检查
- 直接触发提取流程
- 返回提取结果（成功/失败、路径、错误信息）

## 具体技术实现

### 核心状态变量
```typescript
let lastMemoryMessageUuid: string | undefined  // 上次提取时的消息 UUID
let hasLoggedGateFailure = false               // 避免重复记录 gate 检查失败
```

### 触发判断算法
```typescript
export function shouldExtractMemory(messages: Message[]): boolean {
  // 1. 检查初始化阈值
  if (!isSessionMemoryInitialized()) {
    if (!hasMetInitializationThreshold(currentTokenCount)) return false
    markSessionMemoryInitialized()
  }

  // 2. 检查更新阈值
  const hasMetTokenThreshold = hasMetUpdateThreshold(currentTokenCount)
  
  // 3. 检查工具调用阈值
  const toolCallsSinceLastUpdate = countToolCallsSince(messages, lastMemoryMessageUuid)
  const hasMetToolCallThreshold = toolCallsSinceLastUpdate >= getToolCallsBetweenUpdates()

  // 4. 检查上一轮是否有工具调用
  const hasToolCallsInLastTurn = hasToolCallsInLastAssistantTurn(messages)

  // 触发条件：
  // (token 阈值 AND 工具调用阈值) OR (token 阈值 AND 无工具调用)
  const shouldExtract = 
    (hasMetTokenThreshold && hasMetToolCallThreshold) ||
    (hasMetTokenThreshold && !hasToolCallsInLastTurn)
}
```

### Forked Agent 执行流程
```typescript
const extractSessionMemory = sequential(async function (context: REPLHookContext) {
  // 1. 检查 querySource（仅主 REPL 线程执行）
  if (querySource !== 'repl_main_thread') return

  // 2. 检查功能开关（缓存值，非阻塞）
  if (!isSessionMemoryGateEnabled()) return

  // 3. 初始化配置（仅一次）
  initSessionMemoryConfigIfNeeded()

  // 4. 检查触发条件
  if (!shouldExtractMemory(messages)) return

  markExtractionStarted()

  // 5. 创建隔离上下文
  const setupContext = createSubagentContext(toolUseContext)

  // 6. 设置文件并读取当前内容
  const { memoryPath, currentMemory } = await setupSessionMemoryFile(setupContext)

  // 7. 构建提示词
  const userPrompt = await buildSessionMemoryUpdatePrompt(currentMemory, memoryPath)

  // 8. 执行 forked agent
  await runForkedAgent({
    promptMessages: [createUserMessage({ content: userPrompt })],
    cacheSafeParams: createCacheSafeParams(context),
    canUseTool: createMemoryFileCanUseTool(memoryPath),
    querySource: 'session_memory',
    forkLabel: 'session_memory',
    overrides: { readFileState: setupContext.readFileState },
  })

  // 9. 记录指标和更新状态
  logEvent('tengu_session_memory_extraction', {...})
  recordExtractionTokenCount(tokenCountWithEstimation(messages))
  updateLastSummarizedMessageIdIfSafe(messages)
  markExtractionCompleted()
})
```

### 文件设置流程
```typescript
async function setupSessionMemoryFile(toolUseContext: ToolUseContext) {
  // 1. 创建目录（0o700 权限）
  const sessionMemoryDir = getSessionMemoryDir()
  await fs.mkdir(sessionMemoryDir, { mode: 0o700 })

  // 2. 创建文件（如果不存在，使用 wx 标志）
  const memoryPath = getSessionMemoryPath()
  try {
    await writeFile(memoryPath, '', { encoding: 'utf-8', mode: 0o600, flag: 'wx' })
    // 写入默认模板
    const template = await loadSessionMemoryTemplate()
    await writeFile(memoryPath, template, { encoding: 'utf-8', mode: 0o600 })
  } catch (e) {
    if (code !== 'EEXIST') throw e  // 文件已存在是正常情况
  }

  // 3. 读取当前内容（使用 FileReadTool 确保缓存一致性）
  toolUseContext.readFileState.delete(memoryPath)  // 清除缓存
  const result = await FileReadTool.call({ file_path: memoryPath }, toolUseContext)
  
  return { memoryPath, currentMemory: output.file.content }
}
```

### 工具权限限制
```typescript
export function createMemoryFileCanUseTool(memoryPath: string): CanUseToolFn {
  return async (tool: Tool, input: unknown) => {
    // 仅允许 FileEditTool 且仅允许编辑记忆文件
    if (tool.name === FILE_EDIT_TOOL_NAME && 
        typeof input === 'object' && 
        input !== null && 
        'file_path' in input &&
        input.file_path === memoryPath) {
      return { behavior: 'allow' as const, updatedInput: input }
    }
    return { behavior: 'deny' as const, message: '...' }
  }
}
```

## 关键代码路径与文件引用

### 导出函数
| 函数名 | 用途 | 调用方 |
|--------|------|--------|
| `initSessionMemory()` | 初始化并注册 hook | `setup.ts:setup()` |
| `shouldExtractMemory()` | 判断是否触发提取 | 内部使用 |
| `manuallyExtractSessionMemory()` | 手动触发提取 | `/summary` 命令（已废弃） |
| `createMemoryFileCanUseTool()` | 创建工具权限函数 | 内部使用 |
| `resetLastMemoryMessageUuid()` | 重置状态（测试用） | 测试代码 |

### 导入依赖
```typescript
// 核心依赖
import { runForkedAgent, createCacheSafeParams, createSubagentContext } from '../../utils/forkedAgent.js'
import { registerPostSamplingHook } from '../../utils/hooks/postSamplingHooks.js'
import { FileReadTool } from '../../tools/FileReadTool/FileReadTool.js'

// Session Memory 内部
import { buildSessionMemoryUpdatePrompt, loadSessionMemoryTemplate } from './prompts.js'
import { 
  getSessionMemoryConfig, 
  setSessionMemoryConfig,
  markExtractionStarted,
  markExtractionCompleted,
  recordExtractionTokenCount,
  isSessionMemoryInitialized,
  markSessionMemoryInitialized,
  hasMetInitializationThreshold,
  hasMetUpdateThreshold,
  getToolCallsBetweenUpdates,
  setLastSummarizedMessageId,
} from './sessionMemoryUtils.js'

// 文件系统路径
import { getSessionMemoryDir, getSessionMemoryPath } from '../../utils/permissions/filesystem.js'

// Token 计算
import { tokenCountWithEstimation } from '../../utils/tokens.js'

// 远程配置
import { getFeatureValue_CACHED_MAY_BE_STALE, getDynamicConfig_CACHED_MAY_BE_STALE } from '../analytics/growthbook.js'

// 自动压缩集成
import { isAutoCompactEnabled } from '../compact/autoCompact.js'
```

## 依赖与外部交互

### 被调用方
1. **`setup.ts`**
   - `initSessionMemory()`: 应用启动时调用，注册 post-sampling hook

2. **`/summary` 命令（已废弃）**
   - `manuallyExtractSessionMemory()`: 手动触发记忆提取

### 调用方
1. **`forkedAgent.ts`**
   - `runForkedAgent`: 执行隔离的记忆提取 agent
   - `createSubagentContext`: 创建隔离上下文
   - `createCacheSafeParams`: 创建缓存共享参数

2. **`postSamplingHooks.ts`**
   - `registerPostSamplingHook`: 注册提取函数到 post-sampling 钩子

3. **`sessionMemoryUtils.ts`**
   - 各种状态管理函数（配置、提取状态、token 记录等）

4. **`prompts.ts`**
   - `buildSessionMemoryUpdatePrompt`: 构建更新提示词
   - `loadSessionMemoryTemplate`: 加载模板

5. **`filesystem.ts`**
   - `getSessionMemoryDir`, `getSessionMemoryPath`: 获取路径

6. **`autoCompact.ts`**
   - `isAutoCompactEnabled`: 检查自动压缩是否启用（session memory 依赖此设置）

7. **`FileReadTool.ts`**
   - 读取当前记忆文件内容

## 风险、边界与改进建议

### 风险点

1. **并发提取风险**
   - 缓解：使用 `sequential` 包装器确保提取串行执行
   - 但：如果提取耗时过长（>15s），`waitForSessionMemoryExtraction` 会超时继续

2. **Forked Agent 失败处理**
   - 风险：forked agent 执行失败不会阻止主流程
   - 但：失败时 `lastSummarizedMessageId` 不会更新，可能导致重复提取

3. **Token 计算一致性**
   - 风险：`tokenCountWithEstimation` 与 autocompact 使用相同计算方式，但可能与实际 API token 有偏差

4. **Gate 检查延迟**
   - 使用缓存的 GrowthBook 值，可能在功能开关切换后有延迟

5. **文件权限**
   - 目录权限 0o700，文件权限 0o600，但依赖于 umask 设置

### 边界情况

1. **消息 UUID 不存在**
   - `lastMemoryMessageUuid` 指向的消息可能被 compaction 删除
   - `countToolCallsSince` 会从头开始计数

2. **空消息列表**
   - `shouldExtractMemory` 在空列表时行为未定义（实际不会出现）

3. **远程配置为零值**
   - `initSessionMemoryConfigIfNeeded` 会过滤掉非正数配置，使用默认值

4. **Session Memory 文件被外部修改**
   - 每次提取前重新读取文件，但外部修改可能导致内容丢失

### 改进建议

1. **提取失败重试**
   - 添加失败重试机制，避免单次失败导致信息丢失

2. **提取进度指示**
   - 添加用户可见的提取状态（如状态栏图标）

3. **增量更新优化**
   - 当前是完整重写，可考虑增量更新以减少 token 消耗

4. **提取内容验证**
   - 添加提取结果验证，确保模板结构未被破坏

5. **与 Compaction 的协同优化**
   - 当前 compaction 会重置 `lastSummarizedMessageId`，可能导致不必要的全量提取
   - 可考虑保留部分历史上下文

6. **配置热更新**
   - 支持远程配置变更时自动更新阈值，无需重启

7. **提取历史记录**
   - 添加提取历史日志，便于调试和审计
