# src/query/stopHooks.ts 研究文档

## 场景与职责

`stopHooks.ts` 是 Claude Code 查询模块的 Stop Hook 处理器，负责在每个对话轮次结束时执行用户定义的 Stop Hook 脚本。这些钩子允许用户在 AI 响应后自动执行自定义逻辑，如代码检查、测试运行、工作流触发等。

### 核心设计原则
1. **异步生成器**: 使用 `AsyncGenerator` 模式，允许钩子执行过程中逐步产出消息和进度更新
2. **非阻塞与阻塞分离**: 区分非阻塞错误（记录但继续）和阻塞错误（阻止对话继续）
3. **多钩子类型支持**: 除标准 Stop Hook 外，还支持 TeammateIdle 和 TaskCompleted 钩子
4. **超时与取消**: 支持中止信号处理，确保用户取消操作能正确终止钩子执行

## 功能点目的

### handleStopHooks 函数
主入口函数，处理所有类型的 Stop Hook：
- 保存缓存安全参数快照（用于 `/btw` 命令和 side_question SDK 控制请求）
- 执行模板作业分类（当作为分派作业运行时）
- 执行后台任务（prompt 建议、记忆提取、auto-dream）
- 执行 Stop Hook 并处理结果
- 对 teammate 会话执行 TaskCompleted 和 TeammateIdle 钩子

### StopHookResult 类型
定义钩子执行结果：
- `blockingErrors`: 阻塞错误消息列表
- `preventContinuation`: 是否阻止对话继续

### 背景任务
在非 bare 模式下并行执行：
- `executePromptSuggestion`: 提示建议生成
- `executeExtractMemories`: 记忆提取（feature-gated）
- `executeAutoDream`: 自动梦境生成

## 具体技术实现

### 关键流程

```typescript
export async function* handleStopHooks(
  messagesForQuery: Message[],
  assistantMessages: AssistantMessage[],
  systemPrompt: SystemPrompt,
  userContext: { [k: string]: string },
  systemContext: { [k: string]: string },
  toolUseContext: ToolUseContext,
  querySource: QuerySource,
  stopHookActive?: boolean,
): AsyncGenerator<...> {
  // 1. 保存缓存安全参数
  if (querySource === 'repl_main_thread' || querySource === 'sdk') {
    saveCacheSafeParams(createCacheSafeParams(stopHookContext))
  }

  // 2. 模板作业分类（feature-gated）
  if (feature('TEMPLATES') && process.env.CLAUDE_JOB_DIR && ...) {
    await jobClassifierModule!.classifyAndWriteState(...)
  }

  // 3. 后台任务（非 bare 模式）
  if (!isBareMode()) {
    executePromptSuggestion(stopHookContext)
    executeExtractMemories(stopHookContext, ...)
    executeAutoDream(stopHookContext, ...)
  }

  // 4. Chicago MCP 清理（feature-gated）
  if (feature('CHICAGO_MCP') && !toolUseContext.agentId) {
    await cleanupComputerUseAfterTurn(toolUseContext)
  }

  // 5. 执行 Stop Hook
  const generator = executeStopHooks(...)
  for await (const result of generator) {
    // 处理进度消息、阻塞错误、阻止继续等
  }

  // 6. Teammate 钩子（如果是 teammate 会话）
  if (isTeammate()) {
    // 执行 TaskCompleted 和 TeammateIdle 钩子
  }
}
```

### 数据结构

```typescript
type StopHookResult = {
  blockingErrors: Message[]
  preventContinuation: boolean
}

// Hook 进度追踪
const hookInfos: StopHookInfo[] = []
const hookErrors: string[] = []
let preventedContinuation = false
let stopReason = ''
```

### 依赖模块

| 依赖 | 用途 |
|------|------|
| `bun:bundle` | `feature()` 编译时特性门控 |
| `../utils/hooks.js` | `executeStopHooks`, `executeTaskCompletedHooks`, `executeTeammateIdleHooks` |
| `../utils/messages.js` | 消息创建工具函数 |
| `../utils/tasks.js` | 任务列表管理 |
| `../utils/teammate.js` | Teammate 会话检测 |
| `../services/autoDream/autoDream.js` | Auto-dream 服务 |
| `../services/extractMemories/extractMemories.js` | 记忆提取服务（动态 require） |
| `../jobs/classifier.js` | 作业分类器（动态 require） |

## 关键代码路径与文件引用

### 调用方
- `src/query.ts:1267` - 主查询循环调用
  ```typescript
  const stopHookResult = yield* handleStopHooks(
    messagesForQuery,
    assistantMessages,
    systemPrompt,
    userContext,
    systemContext,
    toolUseContext,
    querySource,
    stopHookActive,
  )
  ```

### 被调用方（utils/hooks.ts）
- `executeStopHooks()`: 执行 Stop 类型钩子
- `executeTaskCompletedHooks()`: 执行 TaskCompleted 类型钩子
- `executeTeammateIdleHooks()`: 执行 TeammateIdle 类型钩子

### 消息创建
- `src/utils/messages.ts:4398` - `createStopHookSummaryMessage()`
  ```typescript
  export function createStopHookSummaryMessage(
    hookCount: number,
    hookInfos: StopHookInfo[],
    hookErrors: string[],
    preventedContinuation: boolean,
    stopReason: string | undefined,
    hasOutput: boolean,
    level: SystemMessageLevel,
    toolUseID?: string,
    hookLabel?: string,
    totalDurationMs?: number,
  ): SystemStopHookSummaryMessage
  ```

## 依赖与外部交互

### 上游依赖
1. **utils/hooks.ts**: 核心钩子执行引擎
2. **utils/messages.ts**: 消息创建工具
3. **utils/tasks.ts**: 任务管理
4. **utils/teammate.ts**: Teammate 检测
5. **services/autoDream/autoDream.ts**: Auto-dream 服务
6. **services/extractMemories/extractMemories.ts**: 记忆提取（动态加载）
7. **jobs/classifier.ts**: 作业分类（动态加载）

### 环境变量
| 变量名 | 作用 |
|--------|------|
| `CLAUDE_JOB_DIR` | 模板作业目录，触发作业分类 |
| `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION` | 启用提示建议 |

### 特性门控
| 特性 | 作用 |
|------|------|
| `EXTRACT_MEMORIES` | 启用记忆提取功能 |
| `TEMPLATES` | 启用模板作业分类 |
| `CHICAGO_MCP` | 启用 Computer Use 清理 |

## 风险、边界与改进建议

### 风险点
1. **钩子超时**: 作业分类有 60 秒超时，但其他后台任务无明确超时控制
2. **错误处理**: 钩子执行错误被捕获并记录，但可能导致用户难以诊断问题
3. **并发问题**: 多个钩子并行执行时，状态管理复杂

### 边界情况
1. **子代理**: `toolUseContext.agentId` 存在时跳过某些操作（如参数保存、MCP 清理）
2. **中止信号**: 钩子执行期间检查 `abortController.signal.aborted`，及时退出
3. **Bare 模式**: `--bare` 标志跳过所有后台任务

### 改进建议
1. **统一超时**: 为所有后台任务添加一致的 timeout 机制
2. **错误聚合**: 改进错误报告，提供更详细的钩子失败诊断信息
3. **性能监控**: 添加钩子执行时间的详细 metrics，识别慢钩子
4. **依赖解耦**: 动态 require 的模式可以考虑改为依赖注入，提高可测试性
5. **文档化**: 添加更多关于 Hook 协议（退出码、JSON 输出格式）的文档

### 代码质量观察
1. **行数**: 473 行，是该批次中最大的文件，职责较多
2. **复杂度**: 包含多个嵌套循环和条件分支，建议考虑拆分
3. **测试覆盖**: 需要确保各种钩子类型、错误场景、中止场景都有测试覆盖
