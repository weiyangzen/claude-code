# microCompact.ts 深度研究文档

## 场景与职责

`microCompact.ts` 实现了 Claude Code 的微压缩（Microcompact）功能，这是一种轻量级的上下文管理机制，通过清理旧工具结果的内容来减少 token 消耗，而无需进行完整的对话摘要。

该模块支持三种压缩路径：
1. **时间触发微压缩** (`maybeTimeBasedMicrocompact`): 基于空闲时间触发，清理过期工具结果
2. **缓存微压缩** (`cachedMicrocompactPath`): 使用 API 缓存编辑功能，在服务端删除工具结果
3. **遗留微压缩**: 已移除，由缓存微压缩完全替代

## 功能点目的

### 1. 时间触发微压缩
- 当距离上次助手响应超过阈值（默认 60 分钟）时触发
- 假设服务端缓存已过期，直接内容清理旧工具结果
- 在 API 调用前执行，减少实际发送的 token 数

### 2. 缓存微压缩（Cached Microcompact）
- 使用 API 的 `cache_edits` 功能在服务端删除工具结果
- 不修改本地消息内容，通过 `cache_reference` 和 `cache_edits` 在 API 层处理
- 保留缓存前缀的有效性，仅删除指定内容

### 3. Token 估算
- 提供消息 token 数的粗略估算 (`estimateMessageTokens`)
- 用于缓存微压缩的触发判断
- 4/3 保守系数补偿估算误差

### 4. 状态管理
- 管理缓存微压缩的模块级状态（`cachedMCState`, `pendingCacheEdits`）
- 提供状态重置和清理 API

## 具体技术实现

### 关键常量

```typescript
const IMAGE_MAX_TOKEN_SIZE = 2000
const TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result content cleared]'

// 可压缩工具集合
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,
  ...SHELL_TOOL_NAMES,
  GREP_TOOL_NAME,
  GLOB_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
])
```

### 核心数据结构

```typescript
export type PendingCacheEdits = {
  trigger: 'auto'
  deletedToolIds: string[]
  baselineCacheDeletedTokens: number  // 用于计算增量
}

export type MicrocompactResult = {
  messages: Message[]
  compactionInfo?: {
    pendingCacheEdits?: PendingCacheEdits
  }
}
```

### 核心流程

#### 1. 主入口流程 (`microcompactMessages`)

```
microcompactMessages(messages, toolUseContext, querySource)
  │
  ├─> 清除警告抑制状态
  │   └─> clearCompactWarningSuppression()
  │
  ├─> 时间触发微压缩尝试
  │   └─> maybeTimeBasedMicrocompact()
  │       ├─ 触发 → 返回清理后的消息
  │       └─ 未触发 → 继续
  │
  ├─> 缓存微压缩检查
  │   ├─ feature('CACHED_MICROCOMPACT') 启用？
  │   ├─ isMainThreadSource(querySource)？（仅主线程）
  │   ├─ isCachedMicrocompactEnabled()？
  │   └─ isModelSupportedForCacheEditing(model)？
  │
  ├─> 缓存微压缩执行
  │   └─> cachedMicrocompactPath()
  │       ├─ 有工具可删除 → 返回 pendingCacheEdits
  │       └─ 无 → 返回原消息
  │
  └─> 默认返回原消息
```

#### 2. 时间触发微压缩 (`maybeTimeBasedMicrocompact`)

```
输入: messages, querySource
  │
  ├─> 触发条件评估 (evaluateTimeBasedTrigger)
  │   ├─ 配置 enabled？
  │   ├─ querySource 是主线程？
  │   ├─ 存在上次助手消息？
  │   └─ 时间差 >= gapThresholdMinutes？
  │
  ├─> 收集可压缩工具 ID
  │   └─> collectCompactableToolIds(messages)
  │
  ├─> 计算保留/清理集合
  │   ├─ keepRecent = max(1, config.keepRecent)
  │   ├─ keepSet = 最近 keepRecent 个工具 ID
  │   └─ clearSet = 其余工具 ID
  │
  ├─> 执行内容清理
  │   └─ 遍历消息，将 clearSet 中的 tool_result 内容替换为
  │       TIME_BASED_MC_CLEARED_MESSAGE
  │
  ├─> 状态清理
  │   ├─> resetMicrocompactState()  // 清除缓存微压缩状态
  │   └─> notifyCacheDeletion()     // 通知缓存断点检测
  │
  └─> 返回清理后的消息
```

#### 3. 缓存微压缩 (`cachedMicrocompactPath`)

```
输入: messages, querySource
  │
  ├─> 获取/初始化状态
  │   ├─> getCachedMCModule()  // 惰性加载
  │   └─> ensureCachedMCState()
  │
  ├─> 收集可压缩工具 ID
  │   └─> collectCompactableToolIds(messages)
  │
  ├─> 注册工具结果
  │   └─ 遍历消息，按用户消息分组注册 tool_result
  │       ├─> registerToolResult(state, toolUseId)
  │       └─> registerToolMessage(state, toolIds)
  │
  ├─> 获取待删除工具
  │   └─> getToolResultsToDelete(state)
  │       - 基于 count-based 触发/保留阈值
  │
  ├─> 创建 cache_edits 块
  │   └─> createCacheEditsBlock(state, toolsToDelete)
  │       - 存储到 pendingCacheEdits
  │
  ├─> 记录埋点
  │   └─> logEvent('tengu_cached_microcompact', {...})
  │
  ├─> 获取 baseline token 数
  │   └─ 从最后助手消息的 usage 中提取
  │       cache_deleted_input_tokens
  │
  └─> 返回 MicrocompactResult
      ├─ messages: 原消息（未修改）
      └─ compactionInfo.pendingCacheEdits
```

### Token 估算算法

```typescript
export function estimateMessageTokens(messages: Message[]): number {
  let totalTokens = 0
  
  for (const message of messages) {
    if (message.type !== 'user' && message.type !== 'assistant') continue
    if (!Array.isArray(message.message.content)) continue
    
    for (const block of message.message.content) {
      switch (block.type) {
        case 'text':
          totalTokens += roughTokenCountEstimation(block.text)
          break
        case 'tool_result':
          totalTokens += calculateToolResultTokens(block)
          break
        case 'image':
        case 'document':
          totalTokens += IMAGE_MAX_TOKEN_SIZE  // ~2000 tokens
          break
        case 'thinking':
          totalTokens += roughTokenCountEstimation(block.thinking)
          break
        case 'redacted_thinking':
          totalTokens += roughTokenCountEstimation(block.data)
          break
        case 'tool_use':
          totalTokens += roughTokenCountEstimation(
            block.name + jsonStringify(block.input ?? {})
          )
          break
        default:
          totalTokens += roughTokenCountEstimation(jsonStringify(block))
      }
    }
  }
  
  // 保守系数: 4/3
  return Math.ceil(totalTokens * (4 / 3))
}
```

## 关键代码路径与文件引用

### 内部依赖

| 导入路径 | 用途 |
|---------|------|
| `src/tools/*/prompt.js` / `constants.js` | 工具名称常量 |
| `src/utils/shell/shellToolUtils.js` | `SHELL_TOOL_NAMES` |
| `src/utils/tokens.js` | `roughTokenCountEstimation` |
| `./compactWarningState.js` | 警告抑制控制 |
| `./timeBasedMCConfig.js` | 时间触发配置 |

### 外部调用方

| 调用方 | 路径 | 用途 |
|-------|------|------|
| `query/deps.ts` | `src/query/deps.ts:4,26,36` | 主查询循环调用 |
| `claude.ts` | `src/services/api/claude.ts` | API 层消费 pendingCacheEdits |
| `REPL.tsx` | `src/screens/REPL.tsx` | 重置微压缩状态 |
| `postCompactCleanup.ts` | `src/services/compact/postCompactCleanup.ts:10` | 压缩后状态清理 |
| `context.tsx` | `src/commands/context/context.tsx` | 上下文分析 |
| `sessionMemoryCompact.ts` | `src/services/compact/sessionMemoryCompact.ts:41` | Token 估算 |

### 关键调用链

```
query.ts:mainLoop
  └─> microcompactMessages (via deps)
      ├─> maybeTimeBasedMicrocompact
      │   └─> 直接修改消息内容
      │
      └─> cachedMicrocompactPath
          ├─> getCachedMCModule (惰性加载)
          ├─> registerToolResult / registerToolMessage
          ├─> getToolResultsToDelete
          └─> createCacheEditsBlock
              └─> pendingCacheEdits (模块级状态)

API 请求构建时 (claude.ts)
  └─> consumePendingCacheEdits()
      └─> 将 cache_edits 注入 API 请求
```

## 依赖与外部交互

### Feature Flags

| Flag | 用途 |
|------|------|
| `CACHED_MICROCOMPACT` | 控制缓存微压缩功能开关 |
| `PROMPT_CACHE_BREAK_DETECTION` | 控制缓存断点检测通知 |

### 环境变量

| 变量 | 说明 |
|------|------|
| `USER_TYPE` | 影响埋点详细程度 |

### GrowthBook Flags

| Flag | 配置项 | 说明 |
|------|--------|------|
| `tengu_slate_heron` | `TimeBasedMCConfig` | 时间触发微压缩配置 |

### 配置类型 (TimeBasedMCConfig)

```typescript
export type TimeBasedMCConfig = {
  enabled: boolean              // 主开关
  gapThresholdMinutes: number   // 触发阈值（默认 60）
  keepRecent: number            // 保留最近工具数（默认 5）
}
```

## 风险、边界与改进建议

### 已知风险

1. **循环依赖风险**:
   - `cachedMicrocompact.ts` 通过动态导入避免循环依赖
   - 状态管理分散在模块级别，可能难以追踪

2. **时间触发与缓存微压缩的互斥**:
   - 时间触发执行后会 `resetMicrocompactState()`
   - 可能导致缓存微压缩状态丢失

3. **Token 估算精度**:
   - 4/3 系数是经验值，可能与实际 token 数有偏差
   - 影响缓存微压缩的触发准确性

4. **主线程限制**:
   - 缓存微压缩仅限主线程，子 Agent 无法使用
   - 可能导致多 Agent 场景下上下文管理不一致

### 边界条件

| 场景 | 行为 |
|------|------|
| 无可压缩工具 | 返回原消息，无压缩 |
| 工具数 <= keepRecent | 无工具被清理 |
| 时间差 < gapThreshold | 不触发时间压缩 |
| 非主线程 querySource | 跳过缓存微压缩 |
| 不支持缓存编辑的模型 | 跳过缓存微压缩 |
| pendingCacheEdits 未消费 | 下次调用返回新值，旧值丢失 |

### 改进建议

1. **状态管理集中化**:
   - 考虑将 `cachedMCState` 和 `pendingCacheEdits` 封装到类中
   - 提供更清晰的生命周期管理

2. **Token 估算优化**:
   - 针对特定模型校准估算系数
   - 考虑引入模型特定的 token 计算

3. **配置动态化**:
   - 将 `COMPACTABLE_TOOLS` 清单迁移到远程配置
   - 支持运行时调整

4. **监控增强**:
   - 添加微压缩效果评估埋点
   - 跟踪估算值与实际值的偏差

5. **子 Agent 支持**:
   - 评估缓存微压缩在子 Agent 中的可行性
   - 或提供更清晰的降级策略

6. **时间触发与缓存协调**:
   - 明确两种路径的优先级和互斥规则
   - 避免状态竞争
