# src/utils/collapseHookSummaries.ts 深度研究文档

## 场景与职责

`collapseHookSummaries.ts` 负责合并连续的 Hook 摘要消息。当并行工具调用各自触发 Hook 时，会产生多个相同类型的 Hook 摘要，此模块将它们合并为单条摘要，减少消息噪音并聚合统计信息。

这是消息渲染优化的一部分，与 `collapseBackgroundBashNotifications.ts` 类似，但针对不同类型的消息。

## 功能点目的

### 1. Hook 摘要合并
- 识别带有 `hookLabel` 的系统 Hook 摘要消息
- 将连续的相同标签 Hook 摘要合并为单条
- 聚合统计信息（执行次数、错误、持续时间等）

### 2. 统计信息聚合
- 合并 `hookCount`（Hook 执行次数）
- 合并 `hookInfos` 和 `hookErrors` 数组
- 计算 `preventedContinuation` 和 `hasOutput` 的或值
- 取 `totalDurationMs` 的最大值（并行执行的最接近 wall-clock 时间）

## 具体技术实现

### 核心数据结构

```typescript
// 从 message.ts 导入
interface SystemStopHookSummaryMessage {
  type: 'system'
  subtype: 'stop_hook_summary'
  hookLabel: string
  hookCount: number
  hookInfos: HookInfo[]
  hookErrors: HookError[]
  preventedContinuation: boolean
  hasOutput: boolean
  totalDurationMs?: number
  // ... 其他字段
}

type RenderableMessage = /* 所有可渲染消息类型的联合 */
```

### 检测逻辑

```typescript
function isLabeledHookSummary(
  msg: RenderableMessage,
): msg is SystemStopHookSummaryMessage {
  return (
    msg.type === 'system' &&
    msg.subtype === 'stop_hook_summary' &&
    msg.hookLabel !== undefined
  )
}
```

### 合并算法

```typescript
export function collapseHookSummaries(
  messages: RenderableMessage[],
): RenderableMessage[] {
  const result: RenderableMessage[] = []
  let i = 0

  while (i < messages.length) {
    const msg = messages[i]!
    
    if (isLabeledHookSummary(msg)) {
      const label = msg.hookLabel
      const group: SystemStopHookSummaryMessage[] = []
      
      // 收集连续的相同标签 Hook 摘要
      while (i < messages.length) {
        const next = messages[i]!
        if (!isLabeledHookSummary(next) || next.hookLabel !== label) break
        group.push(next)
        i++
      }
      
      if (group.length === 1) {
        // 只有一个，直接保留
        result.push(msg)
      } else {
        // 合并为单条摘要
        result.push({
          ...msg,
          hookCount: group.reduce((sum, m) => sum + m.hookCount, 0),
          hookInfos: group.flatMap(m => m.hookInfos),
          hookErrors: group.flatMap(m => m.hookErrors),
          preventedContinuation: group.some(m => m.preventedContinuation),
          hasOutput: group.some(m => m.hasOutput),
          // 并行执行取最大值（最接近 wall-clock 时间）
          totalDurationMs: Math.max(...group.map(m => m.totalDurationMs ?? 0)),
        })
      }
    } else {
      // 非目标消息，直接保留
      result.push(msg)
      i++
    }
  }

  return result
}
```

## 依赖与外部交互

### 内部模块依赖

| 模块 | 用途 |
|------|------|
| `src/types/message.ts` | `RenderableMessage` 和 `SystemStopHookSummaryMessage` 类型 |

### 被依赖方

| 模块 | 用途 |
|------|------|
| `src/components/Messages.tsx` | 消息列表渲染时调用合并 |

## 风险、边界与改进建议

### 已知风险

1. **信息丢失**
   - 合并后丢失了每个单独 Hook 的详细信息
   - 无法区分哪些具体的 Hook 调用产生了输出或错误

2. **持续时间计算**
   - 使用 `Math.max` 假设并行执行，但如果是串行执行则不准确
   - 没有区分并行和串行场景

3. **标签冲突**
   - 不同场景但相同标签的 Hook 可能被错误合并
   - 例如不同工具调用的 `PostToolUse` Hook

### 边界情况

1. **空消息列表**
   - 空数组直接返回

2. **无标签 Hook 摘要**
   - `hookLabel === undefined` 的消息不被处理
   - 直接保留原样

3. **不同标签交错**
   - 只有连续的相同标签消息才会合并
   - 例如：PostToolUse、PostToolUse、Stop、Stop → 合并为 2+2

4. **零值处理**
   - `totalDurationMs` 使用 `?? 0` 处理 undefined
   - 如果所有值都是 undefined，结果为 0

5. **空数组聚合**
   - `hookInfos` 和 `hookErrors` 使用 `flatMap`，空数组会被展平为无元素

### 改进建议

1. **信息保留**
   - 添加展开功能，允许用户查看合并的详细信息
   - 保留前 N 个 Hook 的详细信息摘要

2. **持续时间计算改进**
   - 添加执行模式检测（并行/串行）
   - 串行模式下累加持续时间，并行模式下取最大值

3. **标签细分**
   - 添加工具名称到标签，避免不同工具的 Hook 被合并
   - 例如：`PostToolUse:FileReadTool`

4. **可配置性**
   - 添加合并阈值（如超过 N 个才合并）
   - 允许按标签配置合并行为

5. **可观测性**
   - 记录合并事件和合并前的消息数量
   - 添加 Hook 执行统计（每类型 Hook 的平均执行时间）

6. **代码示例**

```typescript
// 改进版本（带执行模式检测）
interface CollapsedHookSummary extends SystemStopHookSummaryMessage {
  executionMode: 'parallel' | 'sequential' | 'unknown'
  details?: Array<{
    hookLabel: string
    hookCount: number
    durationMs: number
  }>
}

function detectExecutionMode(
  group: SystemStopHookSummaryMessage[]
): 'parallel' | 'sequential' | 'unknown' {
  if (group.length < 2) return 'unknown'
  
  const durations = group.map(m => m.totalDurationMs ?? 0)
  const maxDuration = Math.max(...durations)
  const sumDuration = durations.reduce((a, b) => a + b, 0)
  
  // 如果最大值接近总和，可能是串行
  // 如果最大值远小于总和，可能是并行
  if (maxDuration > sumDuration * 0.8) {
    return 'sequential'
  } else if (maxDuration < sumDuration * 0.5) {
    return 'parallel'
  }
  return 'unknown'
}

function calculateTotalDuration(
  group: SystemStopHookSummaryMessage[],
  mode: 'parallel' | 'sequential' | 'unknown'
): number {
  const durations = group.map(m => m.totalDurationMs ?? 0)
  
  switch (mode) {
    case 'parallel':
      return Math.max(...durations)
    case 'sequential':
      return durations.reduce((a, b) => a + b, 0)
    default:
      // 未知时保守估计（取最大值）
      return Math.max(...durations)
  }
}

export function collapseHookSummaries(
  messages: RenderableMessage[],
  options: { keepDetails?: number } = {}
): RenderableMessage[] {
  const { keepDetails = 0 } = options
  // ... 合并逻辑 ...
  
  const executionMode = detectExecutionMode(group)
  const totalDurationMs = calculateTotalDuration(group, executionMode)
  
  const collapsed: CollapsedHookSummary = {
    ...msg,
    hookCount: group.reduce((sum, m) => sum + m.hookCount, 0),
    hookInfos: group.flatMap(m => m.hookInfos),
    hookErrors: group.flatMap(m => m.hookErrors),
    preventedContinuation: group.some(m => m.preventedContinuation),
    hasOutput: group.some(m => m.hasOutput),
    totalDurationMs,
    executionMode,
    // 保留前 N 个详情
    details: keepDetails > 0 
      ? group.slice(0, keepDetails).map(m => ({
          hookLabel: m.hookLabel,
          hookCount: m.hookCount,
          durationMs: m.totalDurationMs ?? 0
        }))
      : undefined
  }
  
  result.push(collapsed)
}
```

7. **与相关模块的对比**

| 特性 | `collapseHookSummaries` | `collapseBackgroundBashNotifications` |
|------|------------------------|--------------------------------------|
| 目标消息类型 | SystemStopHookSummaryMessage | NormalizedUserMessage |
| 合并条件 | 相同 `hookLabel` | 相同消息类型 + 完成状态 |
| 统计聚合 | 是（count, duration, errors） | 否（仅计数） |
| 环境检查 | 无 | fullscreen + !verbose |
| 使用场景 | 并行工具调用的 Hook | 后台 Bash 命令完成 |
