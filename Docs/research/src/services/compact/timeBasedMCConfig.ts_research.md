# timeBasedMCConfig.ts 深度研究文档

## 场景与职责

`timeBasedMCConfig.ts` 是 Claude Code 微压缩(microcompact)功能的配置模块，专门管理**基于时间的微压缩**特性。该特性用于在长时间空闲后自动清理旧的工具结果，利用服务器端提示缓存的过期机制来优化上下文大小。

**核心场景：**
1. **长时间空闲后的请求** - 当用户在上次助手响应后等待较长时间再发送新请求时
2. **服务器缓存过期优化** - 利用服务器端 1 小时缓存 TTL 的过期机制
3. **API 调用前优化** - 在发送请求前清理内容，减少需要重写的数据量

**关键设计决策：**
- 仅在主线程执行（子代理生命周期短，不适用基于间隔的清理）
- 在 API 调用前执行（`microcompactMessages` 中，位于 `callModel` 上游）
- 60 分钟阈值与服务器 1 小时缓存 TTL 对齐，确保不会强制导致本不会发生的缓存未命中

## 功能点目的

### 1. 基于时间的微压缩配置类型定义

**配置结构**：
```typescript
export type TimeBasedMCConfig = {
  enabled: boolean              // 主开关，false 时功能无操作
  gapThresholdMinutes: number   // 触发阈值（分钟），默认 60
  keepRecent: number            // 保留的最近工具结果数，默认 5
}
```

**配置字段说明**：

| 字段 | 默认值 | 说明 |
|-----|-------|------|
| `enabled` | `false` | 功能主开关，需要显式启用 |
| `gapThresholdMinutes` | `60` | 安全选择：服务器 1 小时缓存 TTL 保证已过期，不会强制未命中 |
| `keepRecent` | `5` | 保留最近 N 个可压缩工具结果，避免清理所有上下文 |

### 2. 默认配置

```typescript
const TIME_BASED_MC_CONFIG_DEFAULTS: TimeBasedMCConfig = {
  enabled: false,
  gapThresholdMinutes: 60,
  keepRecent: 5,
}
```

**设计考虑**：
- 默认禁用，需要 GrowthBook 显式启用
- 60 分钟阈值确保服务器缓存已过期
- 保留 5 个最近结果确保模型仍有工作上下文

### 3. 配置获取函数

```typescript
export function getTimeBasedMCConfig(): TimeBasedMCConfig
```

**实现细节**：
- 使用 `getFeatureValue_CACHED_MAY_BE_STALE` 从 GrowthBook 获取配置
- 配置键：`tengu_slate_heron`
- 提升 GB 读取位置，确保每次评估路径都触发曝光（exposure）

**曝光提升模式**：
```typescript
export function getTimeBasedMCConfig(): TimeBasedMCConfig {
  // Hoist the GB read so exposure fires on every eval path
  return getFeatureValue_CACHED_MAY_BE_STALE<TimeBasedMCConfig>(
    'tengu_slate_heron',
    TIME_BASED_MC_CONFIG_DEFAULTS,
  )
}
```

## 具体技术实现

### 核心算法

基于时间的微压缩在 `microCompact.ts` 中的实现逻辑：

```typescript
// 1. 评估触发条件
const trigger = evaluateTimeBasedTrigger(messages, querySource)
if (!trigger) return null

// 2. 收集可压缩工具 ID
const compactableIds = collectCompactableToolIds(messages)

// 3. 确定保留和清理集合
const keepRecent = Math.max(1, config.keepRecent)  // 至少保留 1 个
const keepSet = new Set(compactableIds.slice(-keepRecent))
const clearSet = new Set(compactableIds.filter(id => !keepSet.has(id)))

// 4. 内容清理
for (const message of messages) {
  if (message.type === 'user' && Array.isArray(message.message.content)) {
    for (const block of message.message.content) {
      if (block.type === 'tool_result' && clearSet.has(block.tool_use_id)) {
        block.content = TIME_BASED_MC_CLEARED_MESSAGE
      }
    }
  }
}
```

### 触发评估函数

```typescript
export function evaluateTimeBasedTrigger(
  messages: Message[],
  querySource: QuerySource | undefined,
): { gapMinutes: number; config: TimeBasedMCConfig } | null {
  const config = getTimeBasedMCConfig()
  
  // 需要显式的主线程 querySource
  if (!config.enabled || !querySource || !isMainThreadSource(querySource)) {
    return null
  }
  
  const lastAssistant = messages.findLast(m => m.type === 'assistant')
  if (!lastAssistant) return null
  
  const gapMinutes = (Date.now() - new Date(lastAssistant.timestamp).getTime()) / 60_000
  if (!Number.isFinite(gapMinutes) || gapMinutes < config.gapThresholdMinutes) {
    return null
  }
  
  return { gapMinutes, config }
}
```

### 可压缩工具定义

```typescript
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

## 关键代码路径与文件引用

### 调用方 (Callers)

| 文件路径 | 调用函数 | 场景 |
|---------|---------|------|
| `src/services/compact/microCompact.ts:267` | `maybeTimeBasedMicrocompact` | 微压缩时优先检查时间触发 |
| `src/services/compact/microCompact.ts:422` | `evaluateTimeBasedTrigger` | 触发评估（独立导出） |

### 被调用方 (Callees)

| 函数 | 来源文件 | 用途 |
|-----|---------|------|
| `getFeatureValue_CACHED_MAY_BE_STALE` | `growthbook.ts` | 获取远程配置 |

### 相关常量

| 常量 | 定义位置 | 用途 |
|-----|---------|------|
| `TIME_BASED_MC_CLEARED_MESSAGE` | `microCompact.ts` | 清理后的占位消息 |

### 导出内容

| 导出 | 类型 | 用途 |
|-----|------|------|
| `TimeBasedMCConfig` | type | 配置类型定义 |
| `getTimeBasedMCConfig` | function | 获取配置的主入口 |

## 依赖与外部交互

### 导入依赖

```typescript
import { getFeatureValue_CACHED_MAY_BE_STALE } from '../analytics/growthbook.js'
```

### GrowthBook 配置键

| 配置键 | 类型 | 用途 |
|-------|------|------|
| `tengu_slate_heron` | `TimeBasedMCConfig` | 基于时间的微压缩配置 |

### 相关模块

| 模块 | 关系 |
|-----|------|
| `microCompact.ts` | 主要消费者，实现时间触发逻辑 |
| `growthbook.ts` | 配置来源 |

## 风险、边界与改进建议

### 已知风险

1. **时间计算不准确**
   - 使用客户端 `Date.now()` 与消息时间戳比较
   - 如果系统时间被修改，可能导致错误触发或错过触发
   - 缓解：使用 `Number.isFinite` 检查计算结果

2. **缓存状态不一致**
   - 假设服务器缓存已过期，但实际可能因各种原因仍有缓存
   - 可能导致不必要的上下文丢失
   - 缓解：60 分钟阈值保守设置，确保高概率缓存已过期

3. **GrowthBook 缓存延迟**
   - `getFeatureValue_CACHED_MAY_BE_STALE` 可能返回过期配置
   - 新配置可能需要时间传播

4. **工具结果清理过度**
   - `keepRecent` 设置过低可能导致模型丢失重要上下文
   - 缓解：默认 5 个，且强制至少保留 1 个

### 边界情况

1. **无助手消息**
   ```typescript
   const lastAssistant = messages.findLast(m => m.type === 'assistant')
   if (!lastAssistant) return null
   ```
   没有助手消息时不触发

2. **时间戳解析失败**
   ```typescript
   if (!Number.isFinite(gapMinutes)) return null
   ```
   无效时间戳不触发

3. **无可压缩工具**
   ```typescript
   if (clearSet.size === 0) return null
   ```
   没有可清理的工具时不触发

4. **所有工具都需保留**
   ```typescript
   const keepRecent = Math.max(1, config.keepRecent)
   ```
   当工具数 ≤ keepRecent 时，不清理任何工具

5. **已清理过的工具**
   ```typescript
   block.content !== TIME_BASED_MC_CLEARED_MESSAGE
   ```
   避免重复清理已清理的工具

### 改进建议

1. **添加服务器端缓存状态验证**
   - 通过 API 响应头或其他机制确认缓存状态
   - 仅在确认缓存未命中时执行清理

2. **自适应阈值**
   - 根据历史缓存命中率动态调整阈值
   - 对于缓存稳定的用户降低阈值

3. **工具重要性排序**
   - 不仅基于时间顺序保留工具
   - 考虑工具类型和内容重要性

4. **配置热更新**
   - 当前配置在函数调用时获取
   - 考虑添加配置缓存和监听机制

5. **更精确的时间同步**
   - 使用服务器时间而非客户端时间
   - 或定期校准客户端时间偏移

6. **添加遥测**
   - 记录触发频率和节省的 token 数
   - 分析清理对后续请求质量的影响

### 代码示例：改进的触发评估

```typescript
export function evaluateTimeBasedTrigger(
  messages: Message[],
  querySource: QuerySource | undefined,
  serverTimestamp?: number,  // 新增：服务器时间
): { gapMinutes: number; config: TimeBasedMCConfig } | null {
  const config = getTimeBasedMCConfig()
  
  if (!config.enabled || !querySource || !isMainThreadSource(querySource)) {
    return null
  }
  
  const lastAssistant = messages.findLast(m => m.type === 'assistant')
  if (!lastAssistant) return null
  
  // 使用服务器时间（如果可用）
  const now = serverTimestamp ?? Date.now()
  const lastTime = new Date(lastAssistant.timestamp).getTime()
  
  // 检查时间戳有效性
  if (!Number.isFinite(lastTime) || lastTime > now) {
    logForDebugging('Invalid timestamp in time-based MC evaluation', {
      lastTime,
      now,
      messageId: lastAssistant.uuid,
    })
    return null
  }
  
  const gapMinutes = (now - lastTime) / 60_000
  
  // 添加最小间隔检查（避免过于频繁的触发）
  const MIN_INTERVAL_MINUTES = 5
  if (gapMinutes < Math.max(config.gapThresholdMinutes, MIN_INTERVAL_MINUTES)) {
    return null
  }
  
  return { gapMinutes, config }
}
```

### 配置扩展示例

```typescript
export type TimeBasedMCConfig = {
  enabled: boolean
  gapThresholdMinutes: number
  keepRecent: number
  // 建议添加：
  minToolsToTrigger?: number      // 最少需要多少个工具才触发
  preserveToolTypes?: string[]    // 始终保留的工具类型
  adaptiveThreshold?: boolean     // 是否启用自适应阈值
}
```
