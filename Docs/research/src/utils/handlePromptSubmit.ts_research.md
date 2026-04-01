# handlePromptSubmit.ts 深度研究

## 场景与职责

本模块是 Claude Code 用户输入处理的核心枢纽，负责处理所有用户输入的提交、命令队列管理、即时命令执行和查询触发。它是连接用户界面和后端处理的关键桥梁。

**核心场景：**
1. **用户输入提交**：处理用户在提示框中输入的内容
2. **命令队列处理**：管理待执行的命令队列
3. **即时命令执行**：/config、/doctor 等本地 JSX 命令
4. **查询触发**：将处理后的消息发送到 API
5. **中断处理**：当前查询执行中时的用户输入

## 功能点目的

### 1. 输入处理入口
- **目的**：统一处理所有用户输入路径
- **输入源**：
  - 直接用户输入（PromptInput）
  - 队列中的命令（queue processor）
  - 远程桥接消息（CCR/bridge）

### 2. 退出命令处理
- **目的**：捕获 exit/quit 等退出指令
- **处理**：转换为 `/exit` 命令或调用优雅关闭

### 3. 即时命令执行
- **目的**：在查询活跃时执行本地命令（如 /config）
- **条件**：
  - 查询守卫活跃或外部加载中
  - 命令标记为 `immediate`
  - 命令类型为 `local-jsx`

### 4. 命令队列管理
- **目的**：在查询执行中时排队用户输入
- **优先级**：支持 now/next/later 三级优先级
- **中断**：可中断工具（如 SleepTool）支持取消当前查询

### 5. 查询执行
- **目的**：将处理后的输入发送到 API
- **流程**：
  - 创建 AbortController
  - 处理用户输入（processUserInput）
  - 文件历史快照
  - 触发 onQuery 回调

## 具体技术实现

### 核心数据结构
```typescript
type BaseExecutionParams = {
  queuedCommands?: QueuedCommand[]  // 队列命令（优先处理）
  messages: Message[]               // 当前消息历史
  mainLoopModel: string             // 主循环模型
  ideSelection: IDESelection | undefined
  querySource: QuerySource          // 查询来源
  commands: Command[]               // 可用命令列表
  queryGuard: QueryGuard            // 查询守卫
  isExternalLoading?: boolean       // 外部加载状态
  setToolJSX: SetToolJSXFn
  getToolUseContext: (...) => ProcessUserInputContext
  setUserInputOnProcessing: (prompt?: string) => void
  setAbortController: (abortController: AbortController | null) => void
  onQuery: (...) => Promise<void>   // 查询回调
  setAppState: (updater: (prev: AppState) => AppState) => void
  canUseTool?: CanUseToolFn
}

export type HandlePromptSubmitParams = BaseExecutionParams & {
  input?: string                    // 用户输入
  mode?: PromptInputMode            // 输入模式
  pastedContents?: Record<number, PastedContent>
  helpers: PromptInputHelpers       // 输入框操作辅助
  onInputChange: (value: string) => void
  setPastedContents: React.Dispatch<...>
  abortController?: AbortController | null
  addNotification?: (...) => void
  setMessages?: (...) => void
  streamMode?: SpinnerMode
  hasInterruptibleToolInProgress?: boolean
  uuid?: UUID
  skipSlashCommands?: boolean       // 跳过斜杠命令（远程消息）
}
```

### 关键流程

#### handlePromptSubmit() - 主入口
```
1. 提取参数和辅助函数
2. 队列处理器路径（queuedCommands 存在）
   - 跳过输入验证和引用解析
   - 直接执行 executeUserInput
3. 直接用户输入路径
   a. 解析粘贴内容引用
   b. 检查退出命令
   c. 展开粘贴文本引用
   d. 处理即时命令（local-jsx）
   e. 检查查询守卫状态
      - 活跃：入队（支持中断）
      - 空闲：直接执行
```

#### executeUserInput() - 执行用户输入
```
1. 创建 AbortController
2. 设置查询守卫预留
3. 计算工作负载标签（workload）
4. 在 AsyncLocalStorage 上下文中执行
5. 遍历所有命令（支持批量）
   - 第一个命令：完整处理（附件、IDE 选择）
   - 后续命令：跳过附件（避免重复）
6. 处理用户输入（processUserInput）
7. 文件历史快照（如启用）
8. 触发 onQuery 或清理状态
9. 处理 nextInput（命令链）
```

### 即时命令执行流程
```typescript
if (immediateCommand && immediateCommand.type === 'local-jsx' && (queryGuard.isActive || isExternalLoading)) {
  // 1. 清空输入框
  // 2. 加载并执行命令
  const impl = await immediateCommand.load()
  const jsx = await impl.call(onDone, context, commandArgs)
  
  // 3. 设置工具 JSX（显示命令 UI）
  if (jsx && !doneWasCalled) {
    setToolJSX({ jsx, shouldHidePromptInput: false, isLocalJSXCommand: true, isImmediate: true })
  }
}
```

### 工作负载传播
```typescript
await runWithWorkload(turnWorkload, async () => {
  // 所有在此范围内的异步操作都继承 workload 上下文
  // 用于后台代理（bg agents）的上下文隔离
})
```

## 关键代码路径与文件引用

### 本文件导出
| 导出 | 类型 | 用途 |
|------|------|------|
| `handlePromptSubmit` | 函数 | 主入口函数 |
| `executeUserInput` | 函数 | 执行逻辑（内部） |
| `HandlePromptSubmitParams` | 类型 | 参数类型定义 |
| `PromptInputHelpers` | 类型 | 输入辅助函数类型 |

### 调用方
1. **PromptInput.tsx**: 用户提交输入时
2. **useQueueProcessor.ts**: 处理队列命令时
3. **useCommandKeybindings.tsx**: 快捷键触发命令时

### 依赖模块
```typescript
import type { UUID } from 'crypto'
import { logEvent } from 'src/services/analytics/index.js'
import { type Command, getCommandName, isCommandEnabled } from '../commands.js'
import { selectableUserMessagesFilter } from '../components/MessageSelector.js'
import type { SpinnerMode } from '../components/Spinner/types.js'
import type { QuerySource } from '../constants/querySource.js'
import { expandPastedTextRefs, parseReferences } from '../history.js'
import type { CanUseToolFn } from '../hooks/useCanUseTool.js'
import type { IDESelection } from '../hooks/useIdeSelection.js'
import type { AppState } from '../state/AppState.js'
import type { SetToolJSXFn } from '../Tool.js'
import type { LocalJSXCommandOnDone } from '../types/command.js'
import type { Message } from '../types/message.js'
import { isValidImagePaste, type PromptInputMode, type QueuedCommand } from '../types/textInputTypes.js'
import { createAbortController } from './abortController.js'
import type { PastedContent } from './config.js'
import { logForDebugging } from './debug.js'
import type { EffortValue } from './effort.js'
import type { FileHistoryState } from './fileHistory.js'
import { fileHistoryEnabled, fileHistoryMakeSnapshot } from './fileHistory.js'
import { gracefulShutdownSync } from './gracefulShutdown.js'
import { enqueue } from './messageQueueManager.js'
import { resolveSkillModelOverride } from './model/model.js'
import type { ProcessUserInputContext } from './processUserInput/processUserInput.js'
import { processUserInput } from './processUserInput/processUserInput.js'
import type { QueryGuard } from './QueryGuard.js'
import { queryCheckpoint, startQueryProfile } from './queryProfiler.js'
import { runWithWorkload } from './workloadContext.js'
```

## 依赖与外部交互

### 上游依赖

1. **processUserInput/processUserInput.ts**: 输入处理核心
   - `processUserInput()`: 处理输入并生成消息
   - 支持文本、斜杠命令、bash 命令

2. **messageQueueManager.ts**: 命令队列
   - `enqueue()`: 将命令加入队列
   - 支持优先级和元数据

3. **QueryGuard.ts**: 查询守卫
   - `reserve()`: 预留查询槽位
   - `cancelReservation()`: 取消预留
   - `isActive`: 查询是否活跃

4. **fileHistory.ts**: 文件历史
   - `fileHistoryMakeSnapshot()`: 创建文件历史快照

5. **workloadContext.ts**: 工作负载上下文
   - `runWithWorkload()`: 在特定工作负载上下文中执行

### 下游影响

1. **query.ts / QueryEngine.ts**: API 查询
   - `onQuery` 回调触发实际 API 调用

2. **AppState**: 状态更新
   - `setAppState` 更新全局状态
   - 消息、加载状态等

## 风险、边界与改进建议

### 已知风险

1. **竞态条件**
   - 风险：快速连续提交可能导致状态不一致
   - 缓解：QueryGuard 确保单查询执行

2. **内存泄漏**
   - 风险：AbortController 未正确清理
   - 缓解：try-finally 确保清理

3. **队列堆积**
   - 风险：大量命令入队导致内存压力
   - 缓解：队列大小限制（在 messageQueueManager 中）

4. **即时命令冲突**
   - 风险：多个即时命令同时执行
   - 缓解：isLocalJSXCommand 标志防止重叠

### 边界情况

1. **空输入**：直接返回，不处理
2. **仅图片输入**：处理图片，不发送文本查询
3. **远程消息**：`skipSlashCommands` 防止执行本地命令
4. **中断时提交**：取消当前查询，新输入入队
5. **批量命令**：第一个命令获取完整上下文，后续简化

### 改进建议

1. **输入预处理管道**
   - 建议：插件化的输入预处理链
   - 场景：自定义输入转换、敏感信息过滤

2. **智能队列管理**
   - 建议：相似命令合并、重复检测
   - 场景：用户多次按回车时的去重

3. **执行预览**
   - 建议：提交前显示将要执行的操作摘要
   - 场景：批量命令执行前确认

4. **取消恢复**
   - 建议：取消后支持恢复部分执行结果
   - 挑战：状态管理和一致性

5. **性能监控**
   - 建议：输入处理各阶段耗时追踪
   - 收益：识别性能瓶颈

### 测试要点

1. 直接输入和队列路径的正确性
2. 即时命令执行的隔离性
3. 查询守卫的并发控制
4. 批量命令的附件处理
5. 中断和取消的正确性
6. 工作负载上下文传播
7. 错误处理和清理
