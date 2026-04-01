# extractMemories.ts 深度研究文档

## 场景与职责

`extractMemories.ts` 是 Claude Code 的**自动记忆提取服务**核心模块，负责在每个查询循环结束时自动分析会话内容，将有价值的信息持久化到记忆目录中。

### 核心定位
- **触发时机**: 在每个完整的查询循环结束时（模型产生最终响应且无工具调用）通过 `handleStopHooks` 调用
- **执行模式**: 使用 Forked Agent 模式（`runForkedAgent`）—— 主会话的完美分叉，共享父会话的 prompt cache
- **设计哲学**: 状态采用闭包作用域（closure-scoped）而非模块级，便于测试隔离

### 与主代理的关系
- 主代理的 prompt 始终包含完整的保存指令
- 当主代理自行写入记忆时，`extractMemories` 会跳过该轮次（通过 `hasMemoryWritesSince` 检测）
- 主代理和后台代理互斥执行，避免重复写入

---

## 功能点目的

### 1. 自动记忆提取
在每次对话结束时自动分析最近的 ~N 条消息，识别并保存以下类型的记忆：
- **user**: 用户角色、目标、职责和知识
- **feedback**: 用户给出的工作指导（避免什么、保持什么）
- **project**: 项目正在进行的工作、目标、bug、事件
- **reference**: 外部系统信息指针

### 2. 智能去重与更新
- 检查现有记忆文件，优先更新而非创建重复项
- 预注入记忆目录清单（`formatMemoryManifest`），避免 forked agent 花费一轮执行 `ls`

### 3. 节流控制
- 支持每 N 个符合条件的轮次执行一次提取（通过 `tengu_bramble_lintel` 配置，默认 1）
- 尾随提取（trailing extraction）跳过节流检查

### 4. 团队记忆支持（TEAMMEM feature）
- 同时支持个人记忆目录和团队共享记忆目录
- 根据记忆类型自动路由到正确的目录（private vs team）

---

## 具体技术实现

### 关键流程

#### 初始化流程 (`initExtractMemories`)
```typescript
export function initExtractMemories(): void {
  // 闭包级可变状态
  const inFlightExtractions = new Set<Promise<void>>()  // 进行中的提取
  let lastMemoryMessageUuid: string | undefined         // 游标位置
  let hasLoggedGateFailure = false                      // 一次性日志标记
  let inProgress = false                                // 重叠执行防护
  let turnsSinceLastExtraction = 0                      // 轮次计数器
  let pendingContext: {...} | undefined                 // 暂存的上下文
}
```

#### 执行流程 (`executeExtractMemoriesImpl`)
1. **前置检查**:
   - 仅主代理执行（非子代理）
   - 功能开关检查 (`tengu_passport_quail`)
   - 自动记忆启用检查 (`isAutoMemoryEnabled`)
   - 远程模式跳过

2. **重叠处理**:
   - 如果正在执行中，暂存上下文供尾随执行
   - 使用 `pendingContext` 存储最新上下文（覆盖之前的）

3. **实际提取 (`runExtraction`)**:
   - 计算自上次提取以来的新消息数
   - 检测主代理是否已写入记忆（`hasMemoryWritesSince`）
   - 节流检查（非尾随执行）
   - 扫描现有记忆文件（`scanMemoryFiles`）
   - 构建提取 prompt（`buildExtractAutoOnlyPrompt` 或 `buildExtractCombinedPrompt`）
   - 运行 forked agent（`runForkedAgent`）
   - 更新游标位置
   - 提取写入的文件路径
   - 发送系统消息通知（`createMemorySavedMessage`）

### 关键数据结构

#### 消息可见性判断
```typescript
function isModelVisibleMessage(message: Message): boolean {
  return message.type === 'user' || message.type === 'assistant'
}
```

#### 记忆写入检测
```typescript
function hasMemoryWritesSince(
  messages: Message[],
  sinceUuid: string | undefined,
): boolean
```
遍历消息，检测是否有 `FileEditTool` 或 `FileWriteTool` 调用 targeting auto-memory 路径。

#### 工具权限控制 (`createAutoMemCanUseTool`)
创建受限的 `CanUseToolFn`，仅允许：
- `REPL_TOOL_NAME`: 允许（REPL 内部会重新检查）
- `FILE_READ_TOOL_NAME`, `GREP_TOOL_NAME`, `GLOB_TOOL_NAME`: 无限制读取
- `BASH_TOOL_NAME`: 仅只读命令（通过 `tool.isReadOnly()` 检查）
- `FILE_EDIT_TOOL_NAME`/`FILE_WRITE_TOOL_NAME`: 仅 auto-memory 目录内

### 协议与交互

#### Forked Agent 调用参数
```typescript
const result = await runForkedAgent({
  promptMessages: [createUserMessage({ content: userPrompt })],
  cacheSafeParams,           // 来自父会话的缓存安全参数
  canUseTool,                // 受限的工具权限函数
  querySource: 'extract_memories',
  forkLabel: 'extract_memories',
  skipTranscript: true,      // 不记录到 transcript，避免竞态
  maxTurns: 5,               // 硬限制防止验证循环
})
```

#### 缓存安全参数 (`CacheSafeParams`)
```typescript
type CacheSafeParams = {
  systemPrompt: SystemPrompt
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  toolUseContext: ToolUseContext
  forkContextMessages: Message[]
}
```
这些参数必须与父请求完全一致才能共享 prompt cache。

---

## 关键代码路径与文件引用

### 核心文件
| 文件 | 职责 |
|------|------|
| `src/services/extractMemories/extractMemories.ts` | 主实现，包含初始化、执行、工具权限控制 |
| `src/services/extractMemories/prompts.ts` | 提取 prompt 构建器（auto-only 和 combined 模式） |
| `src/utils/forkedAgent.ts` | Forked Agent 运行基础设施 |
| `src/query/stopHooks.ts` | 调用入口（`handleStopHooks`） |

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/memdir/memoryTypes.ts` | 记忆类型定义（user/feedback/project/reference） |
| `src/memdir/memoryScan.ts` | 记忆目录扫描（`scanMemoryFiles`, `formatMemoryManifest`） |
| `src/memdir/paths.ts` | 记忆路径解析（`getAutoMemPath`, `isAutoMemoryEnabled`） |
| `src/memdir/memdir.ts` | 记忆目录核心（`ENTRYPOINT_NAME`, `truncateEntrypointContent`） |
| `src/memdir/teamMemPaths.ts` | 团队记忆路径（TEAMMEM feature） |
| `src/memdir/teamMemPrompts.ts` | 团队记忆 prompt 构建 |
| `src/utils/hooks/postSamplingHooks.ts` | `REPLHookContext` 类型定义 |
| `src/utils/messages.ts` | `createMemorySavedMessage`, `createUserMessage` |

### 工具相关
| 文件 | 用途 |
|------|------|
| `src/tools/FileReadTool/prompt.ts` | `FILE_READ_TOOL_NAME` |
| `src/tools/FileEditTool/constants.ts` | `FILE_EDIT_TOOL_NAME` |
| `src/tools/FileWriteTool/prompt.ts` | `FILE_WRITE_TOOL_NAME` |
| `src/tools/BashTool/toolName.ts` | `BASH_TOOL_NAME` |
| `src/tools/GrepTool/prompt.ts` | `GREP_TOOL_NAME` |
| `src/tools/GlobTool/prompt.ts` | `GLOB_TOOL_NAME` |
| `src/tools/REPLTool/constants.ts` | `REPL_TOOL_NAME` |

### 调用链
```
queryLoop (query.ts)
  └── handleStopHooks (stopHooks.ts:65)
        └── executeExtractMemories (extractMemories.ts:598)
              └── executeExtractMemoriesImpl (extractMemories.ts:527)
                    └── runExtraction (extractMemories.ts:329)
                          └── runForkedAgent (forkedAgent.ts:489)
```

---

## 依赖与外部交互

### 外部服务
1. **Analytics** (`src/services/analytics/index.js`)
   - `logEvent`: 记录提取事件（`tengu_extract_memories_extraction`, `tengu_extract_memories_error` 等）

2. **GrowthBook** (`src/services/analytics/growthbook.js`)
   - `getFeatureValue_CACHED_MAY_BE_STALE`: 功能开关检查
   - `tengu_passport_quail`: 主开关
   - `tengu_bramble_lintel`: 节流间隔
   - `tengu_moth_copse`: 跳过索引模式

3. **文件系统**
   - 通过 `memoryScan.ts` 读取记忆目录
   - 通过 forked agent 写入记忆文件

### 环境变量
| 变量 | 用途 |
|------|------|
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 禁用自动记忆 |
| `CLAUDE_CODE_SIMPLE` | bare 模式，禁用后台功能 |
| `CLAUDE_CODE_REMOTE` | 远程模式检测 |
| `USER_TYPE` | 用户类型（ant 用户特殊处理） |

### Feature Flags (Bun bundle)
| Flag | 用途 |
|------|------|
| `TEAMMEM` | 团队记忆功能 |
| `EXTRACT_MEMORIES` | 记忆提取功能 |

---

## 风险、边界与改进建议

### 已知风险

#### 1. 竞态条件
- **风险**: 主代理和 forked agent 同时写入同一文件
- **缓解**: `hasMemoryWritesSince` 检测 + 游标推进机制确保互斥

#### 2. 无限循环/验证陷阱
- **风险**: Forked agent 可能陷入验证循环（读取→验证→再读取）
- **缓解**: `maxTurns: 5` 硬限制

#### 3. 上下文压缩导致游标丢失
- **风险**: `sinceUuid` 指向的消息可能被压缩移除
- **缓解**: 未找到 UUID 时回退到计算所有可见消息（而非返回 0）

#### 4. 远程模式下的敏感数据
- **风险**: 团队记忆可能包含敏感信息
- **缓解**: prompt 中明确要求 "You MUST avoid saving sensitive data within shared team memories"

### 边界情况

#### 1. 提取失败处理
```typescript
catch (error) {
  // Extraction is best-effort — log but don't notify on error
  logForDebugging(`[extractMemories] error: ${error}`)
  logEvent('tengu_extract_memories_error', {...})
}
```
失败时不通知用户，仅记录日志和事件。

#### 2. 空提取处理
- 如果没有写入任何文件，`writtenPaths.length === 0`
- 记录 `"[extractMemories] no memories saved this run"`
- 不发送系统消息

#### 3. 尾随提取（Trailing Extraction）
```typescript
const trailing = pendingContext
pendingContext = undefined
if (trailing) {
  await runExtraction({...trailing, isTrailingRun: true})
}
```
- 执行期间到达的新请求会触发尾随提取
- 尾随提取跳过节流检查
- 仅处理两次调用之间新增的消息

### 改进建议

#### 1. 可观测性增强
- 当前仅记录成功/失败事件，建议增加：
  - 提取延迟分布（P50/P95/P99）
  - 每轮提取的消息数分布
  - 缓存命中率趋势

#### 2. 智能节流
- 当前基于轮次的简单节流，建议考虑：
  - 基于消息内容重要性的动态节流
  - 用户活跃时段的提取频率调整

#### 3. 冲突解决
- 当前仅检测主代理写入并跳过，建议：
  - 添加文件级锁机制
  - 合并策略（当主代理和 forked agent 都修改时）

#### 4. 测试覆盖
- 闭包状态使测试需要调用 `initExtractMemories()` 重置
- 建议添加：
  - 并发提取场景测试
  - 大消息量性能测试
  - 团队记忆路由准确性测试

#### 5. 错误恢复
- 当前提取失败仅记录，建议：
  - 有限重试机制（指数退避）
  - 失败通知（可选，通过设置控制）
