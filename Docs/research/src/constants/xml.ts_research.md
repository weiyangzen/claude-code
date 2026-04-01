# xml.ts 深度研究文档

## 场景与职责

`src/constants/xml.ts` 是 Claude Code CLI 的 XML 标签名常量模块，定义了在消息传递、终端输出、任务通知等场景中使用的 XML 标签名称。该模块提供了一种结构化的方式来标记和解析不同类型的消息内容，支持命令元数据、终端活动、任务通知、远程协作等多种功能。

**主要使用场景：**
1. **命令元数据标记**：标记斜杠命令的名称、消息和参数
2. **终端活动包装**：区分终端输入/输出与真实用户提示
3. **任务通知传递**：后台任务完成状态的通知格式
4. **远程协作通信**：Ultraplan 模式、远程审查、队友消息等
5. **跨会话通信**：UDS（Unix Domain Socket）消息传递

**设计目标：**
- 提供统一的标签命名规范
- 支持消息的结构化解析和渲染
- 区分不同类型的内容以便特殊处理

## 功能点目的

### 1. 命令元数据标签

用于标记斜杠命令的相关信息：

| 标签 | 用途 |
|------|------|
| `command-name` | 命令名称（如 "clear", "compact"） |
| `command-message` | 命令相关的消息内容 |
| `command-args` | 命令参数 |

**使用场景**：解析用户输入的斜杠命令，提取命令结构和参数。

### 2. 终端活动标签

用于包装终端/bash 命令的输入输出，表示这是终端活动而非实际用户提示：

| 标签 | 用途 |
|------|------|
| `bash-input` | Bash 命令输入 |
| `bash-stdout` | Bash 标准输出 |
| `bash-stderr` | Bash 标准错误 |
| `local-command-stdout` | 本地命令标准输出 |
| `local-command-stderr` | 本地命令标准错误 |
| `local-command-caveat` | 本地命令警告/注意事项 |

**终端输出标签集合**：
```typescript
export const TERMINAL_OUTPUT_TAGS = [
  BASH_INPUT_TAG,
  BASH_STDOUT_TAG,
  BASH_STDERR_TAG,
  LOCAL_COMMAND_STDOUT_TAG,
  LOCAL_COMMAND_STDERR_TAG,
  LOCAL_COMMAND_CAVEAT_TAG,
] as const
```

### 3. 任务通知标签

用于后台任务完成通知：

| 标签 | 用途 |
|------|------|
| `task-notification` | 任务通知根标签 |
| `task-id` | 任务标识符 |
| `tool-use-id` | 工具使用 ID |
| `task-type` | 任务类型 |
| `output-file` | 输出文件路径 |
| `status` | 任务状态 |
| `summary` | 任务摘要 |
| `reason` | 原因/说明 |

### 4. 工作树标签

用于 Git 工作树操作：

| 标签 | 用途 |
|------|------|
| `worktree` | 工作树根标签 |
| `worktreePath` | 工作树路径 |
| `worktreeBranch` | 工作树分支 |

### 5. 远程协作标签

| 标签 | 用途 |
|------|------|
| `ultraplan` | Ultraplan 模式（远程并行规划会话） |
| `remote-review` | 远程审查结果 |
| `remote-review-progress` | 远程审查进度（run_hunt.sh 心跳） |
| `teammate-message` | 队友消息（Swarm 间通信） |
| `channel-message` | 外部频道消息 |
| `channel` | 频道标识 |
| `cross-session-message` | 跨会话 UDS 消息 |

### 6. Fork 子 Agent 标签

| 标签 | 用途 |
|------|------|
| `fork-boilerplate` | Fork 子 Agent 首条消息的规则/格式样板 |
| `fork-directive-prefix` | 指令文本前缀 |

### 7. 通用参数模式

用于识别常见的帮助和信息请求参数：

```typescript
// 帮助参数：help, -h, --help
export const COMMON_HELP_ARGS = ['help', '-h', '--help']

// 信息参数：list, show, display, current, view, get, check, describe, print, version, about, status, ?
export const COMMON_INFO_ARGS = [
  'list', 'show', 'display', 'current', 'view', 'get',
  'check', 'describe', 'print', 'version', 'about', 'status', '?',
]
```

### 8. 其他标签

| 标签 | 用途 |
|------|------|
| `tick` | 标记/勾选 |

## 具体技术实现

### 常量定义模式

所有标签名都定义为字符串常量：

```typescript
// XML tag names used to mark skill/command metadata in messages
export const COMMAND_NAME_TAG = 'command-name'
export const COMMAND_MESSAGE_TAG = 'command-message'
export const COMMAND_ARGS_TAG = 'command-args'

// XML tag names for terminal/bash command input and output in user messages
export const BASH_INPUT_TAG = 'bash-input'
export const BASH_STDOUT_TAG = 'bash-stdout'
export const BASH_STDERR_TAG = 'bash-stderr'
// ...
```

### 标签集合

终端输出标签使用 `as const` 断言确保类型安全：

```typescript
export const TERMINAL_OUTPUT_TAGS = [
  BASH_INPUT_TAG,
  BASH_STDOUT_TAG,
  BASH_STDERR_TAG,
  LOCAL_COMMAND_STDOUT_TAG,
  LOCAL_COMMAND_STDERR_TAG,
  LOCAL_COMMAND_CAVEAT_TAG,
] as const
```

这允许 TypeScript 推断出具体的字面量类型：
```typescript
type TerminalOutputTag = typeof TERMINAL_OUTPUT_TAGS[number]
// = 'bash-input' | 'bash-stdout' | 'bash-stderr' | ...
```

## 关键代码路径与文件引用

### 调用方分析

该模块被广泛使用，以下是主要调用方：

| 调用文件 | 调用内容 | 用途 |
|----------|----------|------|
| `src/utils/messages.ts` | `COMMAND_ARGS_TAG`, `COMMAND_MESSAGE_TAG`, `COMMAND_NAME_TAG`, `LOCAL_COMMAND_CAVEAT_TAG`, `LOCAL_COMMAND_STDOUT_TAG` | 消息解析和构建 |
| `src/cli/print.ts` | `BASH_STDOUT_TAG`, `BASH_STDERR_TAG` | CLI 输出打印 |
| `src/tasks/LocalMainSessionTask.ts` | `TASK_NOTIFICATION_TAG` 等 | 任务通知处理 |
| `src/screens/REPL.tsx` | `BASH_INPUT_TAG` 等 | REPL 界面渲染 |
| `src/tasks/LocalShellTask/LocalShellTask.tsx` | 终端相关标签 | Shell 任务处理 |
| `src/QueryEngine.ts` | `TEAMMATE_MESSAGE_TAG` 等 | 查询引擎消息处理 |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | `TASK_NOTIFICATION_TAG` 等 | Agent 任务处理 |
| `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | `TASK_NOTIFICATION_TAG` 等 | 远程 Agent 任务 |
| `src/commands/tag/tag.tsx` | 终端相关标签 | Tag 命令处理 |
| `src/commands/model/model.tsx` | 终端相关标签 | Model 命令处理 |
| `src/hooks/useInboxPoller.ts` | `TEAMMATE_MESSAGE_TAG` 等 | 收件箱轮询 |
| `src/utils/collapseBackgroundBashNotifications.ts` | 终端相关标签 | 后台 Bash 通知折叠 |
| `src/components/messages/UserCommandMessage.tsx` | `COMMAND_NAME_TAG` 等 | 用户命令消息渲染 |
| `src/components/messages/UserTextMessage.tsx` | 终端相关标签 | 用户文本消息渲染 |
| `src/services/mcp/channelNotification.ts` | `CHANNEL_MESSAGE_TAG` 等 | MCP 频道通知 |
| `src/utils/log.ts` | 终端相关标签 | 日志记录 |
| `src/components/messages/UserChannelMessage.tsx` | `CHANNEL_MESSAGE_TAG` 等 | 频道消息渲染 |
| `src/components/messages/UserTeammateMessage.tsx` | `TEAMMATE_MESSAGE_TAG` 等 | 队友消息渲染 |
| `src/tools/AgentTool/forkSubagent.ts` | `FORK_BOILERPLATE_TAG`, `FORK_DIRECTIVE_PREFIX` | Fork 子 Agent |
| `src/tools/SkillTool/SkillTool.ts` | `COMMAND_NAME_TAG` 等 | 技能工具 |
| `src/tools/SkillTool/prompt.ts` | `COMMAND_NAME_TAG` 等 | 技能提示词 |
| `src/tools/SleepTool/prompt.ts` | `COMMAND_ARGS_TAG` | Sleep 工具提示词 |
| `src/utils/teammateMailbox.ts` | `TEAMMATE_MESSAGE_TAG` | 队友邮箱 |
| `src/components/PromptInput/PromptInputQueuedCommands.tsx` | `COMMAND_NAME_TAG` 等 | 排队命令输入 |
| `src/utils/task/framework.ts` | `TASK_NOTIFICATION_TAG` 等 | 任务框架 |
| `src/utils/messages/mappers.ts` | `FORK_BOILERPLATE_TAG` 等 | 消息映射 |
| `src/utils/attribution.ts` | `FORK_BOILERPLATE_TAG` | 归因处理 |
| `src/utils/sessionStorage.ts` | `FORK_BOILERPLATE_TAG` | 会话存储 |
| `src/utils/processUserInput/processSlashCommand.tsx` | `COMMAND_NAME_TAG` 等 | 斜杠命令处理 |
| `src/utils/swarm/inProcessRunner.ts` | `TEAMMATE_MESSAGE_TAG` 等 | 进程内运行器 |

### 核心消费代码示例

```typescript
// src/utils/messages.ts
import {
  COMMAND_ARGS_TAG,
  COMMAND_MESSAGE_TAG,
  COMMAND_NAME_TAG,
  LOCAL_COMMAND_CAVEAT_TAG,
  LOCAL_COMMAND_STDOUT_TAG,
} from '../constants/xml.js'

// 构建命令消息
function createCommandMessage(name: string, args: string[], message?: string) {
  return `<${COMMAND_NAME_TAG}>${name}</${COMMAND_NAME_TAG}>
<${COMMAND_ARGS_TAG}>${args.join(' ')}</${COMMAND_ARGS_TAG}>
${message ? `<${COMMAND_MESSAGE_TAG}>${message}</${COMMAND_MESSAGE_TAG}>` : ''}`
}
```

```typescript
// src/utils/processUserInput/processSlashCommand.tsx
import {
  COMMAND_NAME_TAG,
  COMMAND_ARGS_TAG,
  COMMON_HELP_ARGS,
  COMMON_INFO_ARGS,
} from '../constants/xml.js'

// 解析斜杠命令
function parseSlashCommand(content: string) {
  // 提取命令名称和参数
  const nameMatch = content.match(new RegExp(`<${COMMAND_NAME_TAG}>(.+?)</${COMMAND_NAME_TAG}>`))
  const argsMatch = content.match(new RegExp(`<${COMMAND_ARGS_TAG}>(.+?)</${COMMAND_ARGS_TAG}>`))
  
  return {
    name: nameMatch?.[1],
    args: argsMatch?.[1]?.split(/\s+/) ?? [],
    isHelpRequest: argsMatch?.[1]?.split(/\s+/).some(arg => COMMON_HELP_ARGS.includes(arg)),
  }
}
```

```typescript
// src/components/messages/UserTextMessage.tsx
import { TERMINAL_OUTPUT_TAGS } from '../../constants/xml.js'

// 判断消息是否为终端输出
function isTerminalOutput(content: string): boolean {
  return TERMINAL_OUTPUT_TAGS.some(tag => 
    content.includes(`<${tag}>`)
  )
}
```

## 依赖与外部交互

### 外部依赖

该模块是纯常量模块，无运行时依赖。

### 标签使用约定

1. **自闭合 vs 成对标签**
   - 当前所有标签都是成对使用的（`<tag>content</tag>`）
   - 未来可能需要自闭合标签用于元数据标记

2. **命名规范**
   - 使用 kebab-case（短横线连接）
   - 标签名应具有描述性且简洁

3. **嵌套规则**
   - 某些标签可以嵌套（如 `task-notification` 包含 `task-id`）
   - 需要文档化有效的嵌套结构

### 与消息系统的集成

XML 标签被集成到消息系统的多个层面：

```
用户消息
├── 斜杠命令 (command-name, command-args, command-message)
├── 终端输出 (bash-input, bash-stdout, bash-stderr)
└── 本地命令输出 (local-command-stdout, local-command-stderr, local-command-caveat)

系统消息
├── 任务通知 (task-notification 及其子标签)
├── 队友消息 (teammate-message)
└── 频道消息 (channel-message)

Fork 子 Agent
└── 样板包装 (fork-boilerplate, fork-directive-prefix)
```

## 风险、边界与改进建议

### 潜在风险

1. **标签冲突**
   - 如果消息内容本身包含 XML 标签，可能导致解析错误
   - 需要对特殊字符进行转义

2. **标签膨胀**
   - 过多的标签类型增加复杂性
   - 需要定期审查和清理未使用的标签

3. **解析性能**
   - 正则表达式解析 XML 在大消息上可能性能不佳
   - 考虑使用专门的 XML 解析器

4. **版本兼容性**
   - 标签结构变更可能影响旧消息的解析
   - 需要版本控制策略

### 边界情况

1. **空标签**：`<tag></tag>` 和 `<tag/>` 的处理
2. **嵌套深度**：过深的嵌套可能导致栈溢出
3. **非法字符**：标签内容中的 `<`, `>`, `&` 需要转义
4. **大小写敏感**：XML 标签名是大小写敏感的

### 改进建议

1. **类型安全增强**
   ```typescript
   // 为每个标签定义类型
   type CommandNameTag = { type: 'command-name'; value: string }
   type CommandArgsTag = { type: 'command-args'; values: string[] }
   
   // 联合类型
   type XmlTag = CommandNameTag | CommandArgsTag | /* ... */
   
   // 解析函数返回类型化结果
   export function parseXmlTags(content: string): XmlTag[]
   ```

2. **验证和清理**
   ```typescript
   // 转义特殊字符
   export function escapeXmlContent(content: string): string {
     return content
       .replace(/&/g, '&amp;')
       .replace(/</g, '&lt;')
       .replace(/>/g, '&gt;')
   }
   
   // 验证标签结构
   export function validateXmlStructure(content: string): boolean
   ```

3. **结构化构建器**
   ```typescript
   // 使用构建器模式创建标签
   export class XmlMessageBuilder {
     private parts: string[] = []
     
     addCommand(name: string, args: string[]): this {
       this.parts.push(`<${COMMAND_NAME_TAG}>${escapeXmlContent(name)}</${COMMAND_NAME_TAG}>`)
       this.parts.push(`<${COMMAND_ARGS_TAG}>${args.map(escapeXmlContent).join(' ')}</${COMMAND_ARGS_TAG}>`)
       return this
     }
     
     build(): string {
       return this.parts.join('\n')
     }
   }
   ```

4. **文档生成**
   ```typescript
   // 从代码生成标签文档
   export function generateTagDocumentation(): string {
     return `
# XML Tags Reference

## Command Tags
- \`${COMMAND_NAME_TAG}\`: Command name
- \`${COMMAND_ARGS_TAG}\`: Command arguments
...
`
   }
   ```

5. **迁移到标准格式**
   - 考虑使用 JSON 替代 XML 进行结构化数据
   - 或使用专门的协议缓冲区（Protocol Buffers）

6. **标签命名空间**
   ```typescript
   // 添加命名空间避免冲突
   export const NAMESPACE = 'cc' // Claude Code
   export const COMMAND_NAME_TAG = `${NAMESPACE}:command-name`
   ```

7. **模式定义**
   ```typescript
   // 定义有效的标签结构
   export const TAG_SCHEMA = {
     'task-notification': {
       required: ['task-id', 'status'],
       optional: ['summary', 'reason', 'output-file'],
       children: ['task-id', 'tool-use-id', 'task-type', 'output-file', 'status', 'summary', 'reason'],
     },
     // ...
   }
   ```
