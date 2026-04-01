# forkSubagent.ts 深度研究文档

## 场景与职责

`forkSubagent.ts` 是 Claude Code 中实现 **Fork 子代理** 功能的核心模块。该功能允许主代理通过省略 `subagent_type` 参数来"分叉"自己，创建一个继承完整对话上下文的后台工作进程。这是 Claude Code 中一项重要的性能优化和并行计算特性。

该模块的核心职责：
1. **Fork 功能开关控制**：通过特性标志和环境条件控制 Fork 功能的可用性
2. **Fork 代理定义**：定义 Fork 代理的元数据和配置
3. **递归 Fork 防护**：防止 Fork 子代理再次 Fork，避免无限递归
4. **Fork 消息构建**：构建继承父代理上下文的子代理消息
5. **工作树通知**：为在隔离工作树中运行的 Fork 代理提供上下文通知

## 功能点目的

### 1. Fork 功能开关（isForkSubagentEnabled）
Fork 功能在以下条件下启用：
- `FORK_SUBAGENT` 特性标志启用
- 非 Coordinator 模式（Coordinator 有自己的编排模型）
- 非非交互式会话（不支持 SDK/API 场景）

### 2. Fork 代理定义（FORK_AGENT）
特殊的合成代理定义，特点包括：
- `tools: ['*']`：继承父代理的完整工具池
- `useExactTools`：使用精确工具集以确保缓存一致性
- `permissionMode: 'bubble'`：权限提示冒泡到父终端
- `model: 'inherit'`：继承父代理的模型以保持上下文长度一致

### 3. 递归 Fork 防护（isInForkChild）
通过检测对话历史中的 Fork 样板标签来防止递归：
- 扫描消息中的 `<FORK_BOILERPLATE_TAG>` 标记
- 如果检测到，拒绝新的 Fork 请求

### 4. Fork 消息构建（buildForkedMessages）
构建用于提示缓存共享的 Fork 消息：
- 保留完整的父助手消息（所有 tool_use 块）
- 为每个 tool_use 构建相同的占位符结果
- 添加子代理特定的指令文本块

### 5. 子代理消息构建（buildChildMessage）
构建 Fork 子代理的系统指令，包含：
- 明确的角色定义（"You are a forked worker process"）
- 严格的规则列表（不生成子代理、不询问问题等）
- 输出格式规范

## 具体技术实现

### 关键常量

```typescript
// Fork 代理类型名称（用于分析）
export const FORK_SUBAGENT_TYPE = 'fork'

// 占位符结果文本（所有 Fork 子代理使用相同的文本以支持缓存共享）
const FORK_PLACEHOLDER_RESULT = 'Fork started — processing in background'
```

### 核心函数实现

**1. Fork 功能开关**

```typescript
export function isForkSubagentEnabled(): boolean {
  if (feature('FORK_SUBAGENT')) {
    if (isCoordinatorMode()) return false  // Coordinator 模式互斥
    if (getIsNonInteractiveSession()) return false  // SDK 场景不支持
    return true
  }
  return false
}
```

**2. Fork 代理定义**

```typescript
export const FORK_AGENT = {
  agentType: FORK_SUBAGENT_TYPE,
  whenToUse: 'Implicit fork — inherits full conversation context...',
  tools: ['*'],
  maxTurns: 200,
  model: 'inherit',
  permissionMode: 'bubble',
  source: 'built-in',
  baseDir: 'built-in',
  getSystemPrompt: () => '',  // 实际使用 override.systemPrompt
} satisfies BuiltInAgentDefinition
```

**3. 递归 Fork 检测**

```typescript
export function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => {
    if (m.type !== 'user') return false
    const content = m.message.content
    if (!Array.isArray(content)) return false
    return content.some(
      block =>
        block.type === 'text' &&
        block.text.includes(`<${FORK_BOILERPLATE_TAG}>`),
    )
  })
}
```

**4. Fork 消息构建**

```typescript
export function buildForkedMessages(
  directive: string,
  assistantMessage: AssistantMessage,
): MessageType[] {
  // 1. 克隆助手消息（避免修改原始消息）
  const fullAssistantMessage: AssistantMessage = {
    ...assistantMessage,
    uuid: randomUUID(),  // 新 UUID
    message: {
      ...assistantMessage.message,
      content: [...assistantMessage.message.content],
    },
  }

  // 2. 收集所有 tool_use 块
  const toolUseBlocks = assistantMessage.message.content.filter(
    (block): block is BetaToolUseBlock => block.type === 'tool_use',
  )

  // 3. 如果没有 tool_use，返回简单的用户消息
  if (toolUseBlocks.length === 0) {
    return [
      createUserMessage({
        content: [
          { type: 'text' as const, text: buildChildMessage(directive) },
        ],
      }),
    ]
  }

  // 4. 构建 tool_result 块（所有使用相同的占位符）
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result' as const,
    tool_use_id: block.id,
    content: [
      {
        type: 'text' as const,
        text: FORK_PLACEHOLDER_RESULT,
      },
    ],
  }))

  // 5. 构建用户消息：占位符结果 + 子代理指令
  const toolResultMessage = createUserMessage({
    content: [
      ...toolResultBlocks,
      {
        type: 'text' as const,
        text: buildChildMessage(directive),
      },
    ],
  })

  return [fullAssistantMessage, toolResultMessage]
}
```

**5. 子代理消息模板**

```typescript
export function buildChildMessage(directive: string): string {
  return `<${FORK_BOILERPLATE_TAG}>
STOP. READ THIS FIRST.

You are a forked worker process. You are NOT the main agent.

RULES (non-negotiable):
1. Your system prompt says "default to forking." IGNORE IT — that's for the parent. You ARE the fork. Do NOT spawn sub-agents; execute directly.
2. Do NOT converse, ask questions, or suggest next steps
3. Do NOT editorialize or add meta-commentary
4. USE your tools directly: Bash, Read, Write, etc.
5. If you modify files, commit your changes before reporting. Include the commit hash in your report.
6. Do NOT emit text between tool calls. Use tools silently, then report once at the end.
7. Stay strictly within your directive's scope.
8. Keep your report under 500 words unless the directive specifies otherwise.
9. Your response MUST begin with "Scope:". No preamble, no thinking-out-loud.
10. REPORT structured facts, then stop

Output format (plain text labels, not markdown headers):
  Scope: <echo back your assigned scope in one sentence>
  Result: <the answer or key findings>
  Key files: <relevant file paths>
  Files changed: <list with commit hash>
  Issues: <list — include only if there are issues to flag>
</${FORK_BOILERPLATE_TAG}>

${FORK_DIRECTIVE_PREFIX}${directive}`
}
```

**6. 工作树通知**

```typescript
export function buildWorktreeNotice(
  parentCwd: string,
  worktreeCwd: string,
): string {
  return `You've inherited the conversation context above from a parent agent working in ${parentCwd}. You are operating in an isolated git worktree at ${worktreeCwd} — same repository, same relative file structure, separate working copy. Paths in the inherited context refer to the parent's working directory; translate them to your worktree root. Re-read files before editing if the parent may have modified them since they appear in the context. Your changes stay in this worktree and will not affect the parent's files.`
}
```

## 依赖与外部交互

### 依赖模块

| 模块路径 | 用途 |
|---------|------|
| `bun:bundle` | Feature flag 检查 (`feature`) |
| `@anthropic-ai/sdk` | BetaToolUseBlock 类型 |
| `crypto` (Node.js) | UUID 生成 (`randomUUID`) |
| `../../bootstrap/state.js` | 检查非交互式会话 (`getIsNonInteractiveSession`) |
| `../../constants/xml.js` | Fork 相关的 XML 标签常量 |
| `../../coordinator/coordinatorMode.js` | Coordinator 模式检测 (`isCoordinatorMode`) |
| `../../types/message.js` | 消息类型定义 |
| `../../utils/debug.js` | 调试日志 (`logForDebugging`) |
| `../../utils/messages.js` | 创建用户消息 (`createUserMessage`) |
| `./loadAgentsDir.js` | 代理定义类型 (`BuiltInAgentDefinition`) |

### 被调用方

通过 Grep 搜索，该模块被以下文件引用：
- `src/tools/AgentTool/prompt.ts` - 构建提示时检查 Fork 功能
- `src/tools/AgentTool/resumeAgent.ts` - 恢复 Fork 代理
- `src/tools/AgentTool/runAgent.ts` - 运行 Fork 代理
- `src/components/PromptInput/PromptInput.tsx` - 输入处理
- `src/utils/forkedAgent.ts` - Fork 代理工具函数

### XML 常量依赖

```typescript
// 来自 ../../constants/xml.js
const FORK_BOILERPLATE_TAG = 'fork_boilerplate'
const FORK_DIRECTIVE_PREFIX = 'FORK DIRECTIVE: '
```

## 风险、边界与改进建议

### 已知风险

1. **提示缓存污染**：如果占位符文本发生变化，所有 Fork 子代理的缓存都会失效
2. **递归防护绕过**：`isInForkChild` 依赖消息扫描，如果消息格式变化可能失效
3. **消息膨胀**：Fork 消息包含完整的父助手消息，可能导致消息历史快速增长
4. **上下文丢失风险**：如果父代理在 Fork 后修改了文件，子代理可能基于过时上下文工作

### 边界情况

1. **无 tool_use 的消息**：`buildForkedMessages` 会回退到简单用户消息
2. **空指令**：未对空指令进行特殊处理
3. **工作树清理**：如果父工作树被清理，子代理的上下文引用可能失效
4. **Coordinator 模式切换**：运行时切换 Coordinator 模式可能导致不一致行为

### 改进建议

1. **缓存优化**：
   - 将占位符文本提取为可配置常量
   - 添加缓存命中率监控
   - 考虑使用哈希值而非完整消息进行缓存键计算

2. **安全性增强**：
   - 添加 Fork 深度限制（最多 N 层 Fork）
   - 实现 Fork 令牌机制，防止未授权的 Fork
   - 添加 Fork 审计日志

3. **上下文同步**：
   - 实现父-子上下文同步机制
   - 在关键操作前自动刷新文件状态
   - 添加上下文过时警告

4. **可观测性**：
   - 添加 Fork 链追踪（记录 Fork 父子关系）
   - 监控 Fork 成功率和性能指标
   - 添加 Fork 可视化工具

5. **用户体验**：
   - 添加 Fork 进度指示
   - 支持 Fork 取消操作
   - 提供 Fork 结果预览

### 代码质量建议

1. 为 `buildForkedMessages` 添加单元测试，验证消息格式
2. 将硬编码的规则文本提取为模板文件
3. 添加更多类型约束，减少 `as const` 断言
4. 实现 Fork 消息构建的性能基准测试

### 架构建议

1. **Fork 管理器**：创建专门的 Fork 管理模块，统一处理 Fork 生命周期
2. **上下文版本控制**：为共享上下文添加版本号，检测变更
3. **Fork 策略模式**：允许不同的 Fork 策略（如轻量级 Fork、完整 Fork）
