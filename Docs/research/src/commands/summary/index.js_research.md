# 研究文档：src/commands/summary/index.js

## 1. 场景与职责

### 1.1 文件定位

`src/commands/summary/index.js` 是 Claude Code CLI 命令体系中的一个**停用占位符（stub）**。该文件导出一个最小化的命令对象，其 `isEnabled` 方法恒返回 `false`，使得该命令对用户完全不可见且不可调用。

### 1.2 设计意图

根据代码库中的注释和关联实现，`/summary` 命令的原始设计意图是：

> **允许用户手动触发一次 Session Memory 提取**，将当前对话的关键信息写入 `summary.md`，而不等待自动阈值触发。

这与 `/compact` 命令形成互补关系：
- `/compact`：压缩上下文，用摘要替换历史消息
- `/summary`：仅更新 session memory 文件，不修改消息历史

### 1.3 当前状态

该命令当前处于**完全禁用状态**：
- `isEnabled: () => false` — 命令被过滤，不会出现在任何命令列表中
- `isHidden: true` — 即使启用，也不会显示在帮助或自动补全中
- 无 `type`、`description`、`load` 等标准命令字段

## 2. 功能点目的

### 2.1 关联功能：Session Memory

Session Memory 是 Claude Code 的自动会话摘要系统：

1. **自动触发**：当对话达到阈值（token 数、工具调用次数）时，后台 forked agent 自动提取关键信息
2. **存储位置**：`~/.claude/projects/{sanitized-cwd}/{sessionId}/session-memory/summary.md`
3. **用途**：为 `/compact` 提供上下文摘要，避免重复读取完整历史

### 2.2 /summary 命令的预期功能

如果该命令被启用，应当：

1. **手动触发提取**：绕过自动阈值检查，立即执行 Session Memory 提取
2. **即时反馈**：告知用户摘要已更新，可在下次 `/compact` 时使用
3. **并发安全**：等待正在进行的自动提取完成，避免竞态条件

### 2.3 与相关功能的区别

| 功能 | 触发时机 | 修改消息历史 | 输出位置 |
|------|----------|--------------|----------|
| `/summary` (禁用) | 用户手动 | 否 | `summary.md` 文件 |
| `/compact` | 用户手动 | 是（替换为摘要） | 消息历史 |
| Auto Session Memory | 自动（阈值触发） | 否 | `summary.md` 文件 |
| Away Summary | 终端失焦 5 分钟后 | 否 | TUI 卡片 |

## 3. 具体技术实现

### 3.1 命令对象结构

```javascript
// src/commands/summary/index.js
export default { 
  isEnabled: () => false, 
  isHidden: true, 
  name: 'stub' 
};
```

**字段分析**：
- `isEnabled`: 恒返回 `false`，导致 `getCommands()` 过滤时排除此命令
- `isHidden`: 即使启用，也不显示在帮助中
- `name: 'stub'`: 非用户-facing 名称，实际命令名应为 `'summary'`

### 3.2 命令注册流程

```typescript
// src/commands.ts:142
import summary from './commands/summary/index.js'

// src/commands.ts:245 - 加入内部命令列表
export const INTERNAL_ONLY_COMMANDS = [
  // ...
  summary,
  // ...
].filter(Boolean)

// src/commands.ts:651 - 加入 Bridge 安全列表
export const BRIDGE_SAFE_COMMANDS: Set<Command> = new Set([
  compact,
  clear,
  cost,
  summary, // <-- 声明为 Bridge 安全
  releaseNotes,
  files,
].filter((c): c is Command => c !== null))
```

### 3.3 命令过滤机制

```typescript
// src/commands.ts:476-517
export async function getCommands(cwd: string): Promise<Command[]> {
  const allCommands = await loadAllCommands(cwd)
  // ...
  const baseCommands = allCommands.filter(
    _ => meetsAvailabilityRequirement(_) && isCommandEnabled(_),
  )
  // ...
}

// src/types/command.ts:214-216
export function isCommandEnabled(cmd: CommandBase): boolean {
  return cmd.isEnabled?.() ?? true
}
```

由于 `summary.isEnabled()` 返回 `false`，该命令永远不会出现在 `getCommands()` 返回的列表中。

### 3.4 预留的手动提取接口

```typescript
// src/services/SessionMemory/sessionMemory.ts:383-453
/**
 * Manually trigger session memory extraction, bypassing threshold checks.
 * Used by the /summary command.
 */
export async function manuallyExtractSessionMemory(
  messages: Message[],
  toolUseContext: ToolUseContext,
): Promise<ManualExtractionResult> {
  if (messages.length === 0) {
    return { success: false, error: 'No messages to summarize' }
  }
  markExtractionStarted()

  try {
    // 创建隔离上下文
    const setupContext = createSubagentContext(toolUseContext)
    
    // 设置文件系统并读取当前状态
    const { memoryPath, currentMemory } = await setupSessionMemoryFile(setupContext)
    
    // 创建提取消息
    const userPrompt = await buildSessionMemoryUpdatePrompt(currentMemory, memoryPath)
    
    // 获取系统提示
    const [rawSystemPrompt, userContext, systemContext] = await Promise.all([
      getSystemPrompt(tools, mainLoopModel),
      getUserContext(),
      getSystemContext(),
    ])
    
    // 使用 runForkedAgent 执行提取
    await runForkedAgent({
      promptMessages: [createUserMessage({ content: userPrompt })],
      cacheSafeParams: {
        systemPrompt,
        userContext,
        systemContext,
        toolUseContext: setupContext,
        forkContextMessages: messages,
      },
      canUseTool: createMemoryFileCanUseTool(memoryPath),
      querySource: 'session_memory',
      forkLabel: 'session_memory_manual',
      overrides: { readFileState: setupContext.readFileState },
    })

    // 记录手动提取事件
    logEvent('tengu_session_memory_manual_extraction', {})
    
    // 更新状态
    recordExtractionTokenCount(tokenCountWithEstimation(messages))
    updateLastSummarizedMessageIdIfSafe(messages)

    return { success: true, memoryPath }
  } catch (error) {
    return { success: false, error: errorMessage(error) }
  } finally {
    markExtractionCompleted()
  }
}
```

### 3.5 Session Memory 文件路径

```typescript
// src/utils/permissions/filesystem.ts:261-271
/**
 * Returns the session memory file path for the current session.
 * Path format: {projectDir}/{sessionId}/session-memory/summary.md
 */
export function getSessionMemoryPath(): string {
  return join(getSessionMemoryDir(), 'summary.md')
}

export function getSessionMemoryDir(): string {
  return join(getProjectDir(getCwd()), getSessionId(), 'session-memory') + sep
}
```

### 3.6 Session Memory 模板结构

```markdown
# Session Title
_A short and distinctive 5-10 word descriptive title for the session..._

# Current State
_What is actively being worked on right now?..._

# Task specification
_What did the user ask to build?..._

# Files and Functions
_What are the important files?..._

# Workflow
_What bash commands are usually run and in what order?..._

# Errors & Corrections
_Errors encountered and how they were fixed..._

# Codebase and System Documentation
_What are the important system components?..._

# Learnings
_What has worked well? What has not?..._

# Key results
_If the user asked a specific output..._

# Worklog
_Step by step, what was attempted, done?..._
```

## 4. 关键代码路径与文件引用

### 4.1 核心文件

| 文件路径 | 作用 |
|----------|------|
| `src/commands/summary/index.js` | **被研究对象**：stub 命令定义 |
| `src/commands.ts` | 命令总线：导入、注册、过滤命令 |
| `src/types/command.ts` | Command 类型定义和 `isCommandEnabled` 函数 |

### 4.2 Session Memory 相关文件

| 文件路径 | 作用 |
|----------|------|
| `src/services/SessionMemory/sessionMemory.ts` | 核心实现：自动提取、手动提取接口 |
| `src/services/SessionMemory/sessionMemoryUtils.ts` | 工具函数：状态管理、配置、等待机制 |
| `src/services/SessionMemory/prompts.ts` | 提示词模板和构建函数 |
| `src/services/compact/sessionMemoryCompact.ts` | Session Memory 压缩集成 |

### 4.3 调用链分析

```
用户输入 /summary
    ↓
REPL.tsx: processSlashCommand()
    ↓
hasCommand('summary', commands) 
    ↓
findCommand() in commands.ts
    ↓
返回 undefined（因为 summary 被 getCommands() 过滤）
    ↓
流程回退到普通用户提示（shouldQuery: true）
```

### 4.4 如果命令启用后的预期调用链

```
用户输入 /summary
    ↓
processSlashCommand() 找到命令
    ↓
命令类型决定执行路径：
    - 'local': 调用 load() 加载模块，执行 call()
    - 'local-jsx': 调用 load() 加载模块，执行 call() 返回 React 节点
    - 'prompt': 展开为提示消息发送给模型
    ↓
应调用 manuallyExtractSessionMemory()
    ↓
runForkedAgent() 执行提取
    ↓
更新 summary.md 文件
```

## 5. 依赖与外部交互

### 5.1 直接依赖

```javascript
// src/commands/summary/index.js
// 无直接依赖（纯 stub）
```

### 5.2 通过命令总线的间接依赖

| 依赖 | 来源 | 用途 |
|------|------|------|
| `src/commands.ts` | 导入并注册命令 | 命令总线 |
| `src/types/command.ts` | 类型定义 | 类型检查 |

### 5.3 预留实现依赖（manuallyExtractSessionMemory）

| 依赖 | 来源 | 用途 |
|------|------|------|
| `runForkedAgent` | `src/utils/forkedAgent.ts` | 执行 forked 提取 |
| `createSubagentContext` | `src/utils/forkedAgent.ts` | 创建隔离上下文 |
| `getSystemPrompt` | `src/constants/prompts.ts` | 获取系统提示 |
| `getUserContext` / `getSystemContext` | `src/context.ts` | 获取上下文 |
| `FileReadTool` | `src/tools/FileReadTool/FileReadTool.js` | 读取当前内存 |
| `buildSessionMemoryUpdatePrompt` | `src/services/SessionMemory/prompts.ts` | 构建提示 |
| `createMemoryFileCanUseTool` | `src/services/SessionMemory/sessionMemory.ts` | 工具权限控制 |

### 5.4 文件系统交互

```
写入路径: ~/.claude/projects/{sanitized-cwd}/{sessionId}/session-memory/summary.md
权限: 0o600（用户读写）
```

### 5.5 外部服务交互

- **Analytics**: `logEvent('tengu_session_memory_manual_extraction', {})`
- **Forked Agent**: 通过 `runForkedAgent` 调用 LLM API 生成摘要

## 6. 风险、边界与改进建议

### 6.1 当前风险

#### 6.1.1 类型安全漏洞

`summary` stub 对象缺少 `type`、`description`、`load` 等 `Command` 联合类型必需的字段，却被强制放入 `BRIDGE_SAFE_COMMANDS: Set<Command>` 和 `INTERNAL_ONLY_COMMANDS: Command[]` 中。虽然 `.filter((c): c is Command => c !== null)` 进行了非空断言，但并未验证字段完整性。

**风险**：如果未来某处代码假设 `BRIDGE_SAFE_COMMANDS` 中的命令都是完整可执行的 `local` 命令，可能在运行时访问 `cmd.load()` 时抛出 `TypeError`。

#### 6.1.2 注释与代码不一致

`manuallyExtractSessionMemory()` 函数明确标注 "Used by the /summary command"，但实际上没有任何调用方。这种注释与代码实际状态的不一致会导致维护者误解功能可用性。

#### 6.1.3 BRIDGE_SAFE_COMMANDS 无效声明

由于 `summary` 永远不会出现在 `getCommands()` 返回的列表中，`isBridgeSafeCommand()` 在检查 `BRIDGE_SAFE_COMMANDS.has(cmd)` 时，永远不会对 `summary` 返回 `true`。这使得 `BRIDGE_SAFE_COMMANDS` 中对 `summary` 的声明成为**无实际效果的文档注释**。

#### 6.1.4 用户输入 `/summary` 的意外行为

由于 `hasCommand('summary', commands)` 返回 `false`，用户输入 `/summary` 会被当作普通文本提示发送给模型。模型可能会尝试解释这个输入，而不是执行预期的本地命令，造成用户体验不一致。

### 6.2 实现风险（如果启用）

#### 6.2.1 Feature Gate 绕过

即使 `/summary` 被实现，如果 `tengu_session_memory` feature flag 关闭，`manuallyExtractSessionMemory()` 仍可能尝试执行（该函数内部没有 gate 检查），导致不必要的 forked agent 调用和 API 消耗。

**建议**：实现时应在命令入口层或函数内部增加 gate 检查。

#### 6.2.2 竞态条件

如果用户手动触发 `/summary` 时，后台自动提取正在进行中，`manuallyExtractSessionMemory()` 不会等待后台提取完成（与 `trySessionMemoryCompaction` 不同，后者会调用 `waitForSessionMemoryExtraction()`）。这可能导致两个 forked agent 同时编辑 `summary.md`，产生竞态条件。

**建议**：应在手动提取前调用 `waitForSessionMemoryExtraction()`。

### 6.3 改进建议

#### 6.3.1 方案 A：永久删除（如果功能不再计划）

如果产品决策是永久下线 `/summary` 命令：

1. 删除 `src/commands/summary/index.js` 及其目录
2. 从 `src/commands.ts` 中移除 `summary` 导入和 `INTERNAL_ONLY_COMMANDS`、`BRIDGE_SAFE_COMMANDS` 中的引用
3. 删除或更新 `manuallyExtractSessionMemory()` 的注释，移除 "Used by the /summary command" 的误导性说明

#### 6.3.2 方案 B：恢复功能（如果需要手动触发）

如果希望恢复该功能，可将 `src/commands/summary/index.js` 替换为标准的 local 命令结构：

```typescript
import type { Command, LocalCommandCall } from '../../commands.js'
import { manuallyExtractSessionMemory } from '../../services/SessionMemory/sessionMemory.js'
import type { ToolUseContext } from '../../Tool.js'

const call: LocalCommandCall = async (args, context) => {
  const { messages, toolUseContext } = context
  
  // 等待任何正在进行的提取
  const { waitForSessionMemoryExtraction } = await import('../../services/SessionMemory/sessionMemoryUtils.js')
  await waitForSessionMemoryExtraction()
  
  // 执行手动提取
  const result = await manuallyExtractSessionMemory(messages, toolUseContext)
  
  if (!result.success) {
    return { type: 'text', value: `Failed to generate summary: ${result.error}` }
  }
  
  return {
    type: 'text',
    value: `Session summary updated.`,
  }
}

const summary: Command = {
  type: 'local',
  name: 'summary',
  description: 'Manually update the session memory summary',
  supportsNonInteractive: true,
  load: () => Promise.resolve({ call }),
}

export default summary
```

#### 6.3.3 方案 C：与 /compact 集成（推荐）

考虑在 `/compact` 成功后自动触发一次 session memory 更新（如果自定义指令为空且 SM-compact 未命中），从而消除用户对 `/summary` 的显式需求。或者，在 `/compact` 的输出中明确告知用户"摘要已更新，可在下次会话中使用"。

### 6.4 测试建议

如果恢复该功能，应添加以下测试：

1. **单元测试**：验证 `manuallyExtractSessionMemory` 的调用逻辑
2. **集成测试**：验证与 `waitForSessionMemoryExtraction` 的并发控制
3. **E2E 测试**：验证用户输入 `/summary` 后的完整流程
4. **边界测试**：空消息、Feature gate 关闭、文件系统不可写等异常情况
