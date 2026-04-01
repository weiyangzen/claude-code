# src/utils/collapseBackgroundBashNotifications.ts 深度研究文档

## 场景与职责

`collapseBackgroundBashNotifications.ts` 负责在 UI 中合并连续的后台 Bash 命令完成通知。当多个后台命令在短时间内完成时，将它们合并为一条 "N background commands completed" 通知，减少消息噪音。

这是消息渲染优化的一部分，仅在 fullscreen 模式下生效，且可以通过 verbose 模式禁用。

## 功能点目的

### 1. 后台命令完成通知合并
- 识别后台 Bash 命令完成的通知消息
- 将连续的完成通知合并为单条汇总通知
- 保留失败/被杀死的任务的单独通知

### 2. 消息类型过滤
- 只处理特定格式的任务通知（包含 `<task-notification>` 标签）
- 只合并状态为 "completed" 的通知
- 只合并 Bash 类型的后台任务（通过 `BACKGROUND_BASH_SUMMARY_PREFIX` 识别）

## 具体技术实现

### 核心数据结构

```typescript
// 从 message.ts 导入的消息类型
type RenderableMessage = /* ... */
type NormalizedUserMessage = /* ... */
```

### 关键常量

```typescript
// 来自 LocalShellTask.tsx
const BACKGROUND_BASH_SUMMARY_PREFIX = 'Background command '

// 来自 xml.ts
const TASK_NOTIFICATION_TAG = 'task-notification'
const STATUS_TAG = 'status'
const SUMMARY_TAG = 'summary'
```

### 检测逻辑

```typescript
function isCompletedBackgroundBash(
  msg: RenderableMessage,
): msg is NormalizedUserMessage {
  // 必须是用户消息
  if (msg.type !== 'user') return false
  
  const content = msg.message.content[0]
  // 必须是文本内容
  if (content?.type !== 'text') return false
  // 必须包含任务通知标签
  if (!content.text.includes(`<${TASK_NOTIFICATION_TAG}`)) return false
  // 只合并成功的完成
  if (extractTag(content.text, STATUS_TAG) !== 'completed') return false
  // 必须是 Bash 类型的后台任务
  return (
    extractTag(content.text, SUMMARY_TAG)?.startsWith(
      BACKGROUND_BASH_SUMMARY_PREFIX,
    ) ?? false
  )
}
```

### 合并算法

```typescript
export function collapseBackgroundBashNotifications(
  messages: RenderableMessage[],
  verbose: boolean,
): RenderableMessage[] {
  // Fullscreen 未启用或 verbose 模式，直接返回
  if (!isFullscreenEnvEnabled()) return messages
  if (verbose) return messages

  const result: RenderableMessage[] = []
  let i = 0

  while (i < messages.length) {
    const msg = messages[i]!
    
    if (isCompletedBackgroundBash(msg)) {
      // 收集连续的完成通知
      let count = 0
      while (i < messages.length && isCompletedBackgroundBash(messages[i]!)) {
        count++
        i++
      }
      
      if (count === 1) {
        // 只有一个，直接保留
        result.push(msg)
      } else {
        // 多个，合成一条汇总通知
        result.push({
          ...msg,
          message: {
            role: 'user',
            content: [
              {
                type: 'text',
                text: `<${TASK_NOTIFICATION_TAG}>
<${STATUS_TAG}>completed</${STATUS_TAG}>
<${SUMMARY_TAG}>${count} background commands completed</${SUMMARY_TAG}>
</${TASK_NOTIFICATION_TAG}>`,
              },
            ],
          },
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
| `src/constants/xml.ts` | XML 标签常量 |
| `src/tasks/LocalShellTask/LocalShellTask.tsx` | `BACKGROUND_BASH_SUMMARY_PREFIX` 常量 |
| `src/types/message.ts` | 消息类型定义 |
| `src/utils/fullscreen.ts` | `isFullscreenEnvEnabled()` 检查 |
| `src/utils/messages.ts` | `extractTag()` 函数 |

### 被依赖方

| 模块 | 用途 |
|------|------|
| `src/components/Messages.tsx` | 消息列表渲染时调用合并 |

## 风险、边界与改进建议

### 已知风险

1. **消息格式依赖**
   - 依赖特定的 XML 标签格式，如果格式变更需要同步更新
   - 硬编码的字符串匹配可能 fragile

2. **过度合并**
   - 所有连续的完成通知都被合并，可能丢失有用信息
   - 用户无法知道具体哪些命令完成了

3. **与 Monitor 任务的混淆**
   - 注释说明 Monitor 类型的完成有意不在这里合并
   - 需要确保类型检测准确

### 边界情况

1. **空消息列表**
   - 空数组直接返回，不报错

2. **非文本内容**
   - 只检查 `content[0]`，忽略其他内容块
   - 如果第一个内容块不是文本，不处理

3. **状态标签缺失**
   - `extractTag` 返回 undefined，不会匹配 'completed'

4. **混合消息**
   - 完成通知和其他消息交错时，只合并连续的完成通知
   - 例如：完成、完成、其他、完成 → 合并为 2+1

5. **Verbose 模式**
   - 启用 verbose 时完全禁用合并
   - 用于调试或需要查看所有通知的场景

### 改进建议

1. **可配置性**
   - 添加设置控制合并阈值（如超过 N 个才合并）
   - 允许禁用特定类型的合并

2. **信息保留**
   - 合并时保留命令列表（可展开查看详情）
   - 添加时间范围信息（"5 commands completed in 30s"）

3. **智能合并**
   - 按命令类型分组合并（如所有 git 命令合并）
   - 按时间窗口合并（如 10 秒内的完成）

4. **可观测性**
   - 记录合并事件（用于分析通知频率）
   - 添加合并统计（每会话合并了多少通知）

5. **代码健壮性**
   - 使用类型守卫而非运行时字符串检查
   - 添加消息格式验证
   - 单元测试覆盖各种边界情况

6. **代码示例**

```typescript
// 改进版本（带时间窗口和详情保留）
interface CollapseOptions {
  verbose: boolean
  timeWindowMs?: number  // 时间窗口内合并
  maxDetails?: number    // 保留的最大详情数
}

export function collapseBackgroundBashNotifications(
  messages: RenderableMessage[],
  options: CollapseOptions,
): RenderableMessage[] {
  const { verbose, timeWindowMs = 5000, maxDetails = 3 } = options
  
  if (!isFullscreenEnvEnabled() || verbose) return messages
  
  const result: RenderableMessage[] = []
  let i = 0
  
  while (i < messages.length) {
    const msg = messages[i]!
    
    if (isCompletedBackgroundBash(msg)) {
      const group: RenderableMessage[] = [msg]
      const startTime = extractTimestamp(msg)
      
      // 收集时间窗口内的完成通知
      while (i + 1 < messages.length) {
        const next = messages[i + 1]!
        if (!isCompletedBackgroundBash(next)) break
        
        const nextTime = extractTimestamp(next)
        if (nextTime - startTime > timeWindowMs) break
        
        group.push(next)
        i++
      }
      
      if (group.length === 1) {
        result.push(msg)
      } else {
        // 提取前 N 个命令详情
        const details = group
          .slice(0, maxDetails)
          .map(m => extractCommandName(m))
          .join(', ')
        const more = group.length > maxDetails 
          ? ` and ${group.length - maxDetails} more` 
          : ''
          
        result.push(createCollapsedMessage(
          group.length,
          details + more
        ))
      }
    } else {
      result.push(msg)
    }
    i++
  }
  
  return result
}
```
