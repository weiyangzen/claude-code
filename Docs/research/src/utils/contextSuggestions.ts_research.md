# src/utils/contextSuggestions.ts 深度研究文档

## 1. 场景与职责

`contextSuggestions.ts` 是 Claude Code CLI 的上下文优化建议生成模块。它基于上下文使用数据（来自 `analyzeContext.ts` 的 `ContextData`），智能分析潜在的上下文浪费并提供可操作的优化建议。

### 主要职责
- **容量警告**: 当上下文接近满载时发出警告
- **工具结果优化**: 识别大型工具结果并提供缩减建议
- **文件读取优化**: 检测重复或过度文件读取
- **内存文件管理**: 提醒用户清理过大的内存文件
- **自动压缩建议**: 当自动压缩被禁用时提醒用户

## 2. 功能点目的

### 2.1 建议类型

```typescript
export type SuggestionSeverity = 'info' | 'warning'

export type ContextSuggestion = {
  severity: SuggestionSeverity
  title: string
  detail: string
  savingsTokens?: number  // 预计可节省的令牌数
}
```

### 2.2 触发阈值

```typescript
const LARGE_TOOL_RESULT_PERCENT = 15       // 工具结果占上下文 > 15%
const LARGE_TOOL_RESULT_TOKENS = 10_000    // 工具结果 > 10k 令牌
const READ_BLOAT_PERCENT = 5               // 读取结果占上下文 > 5%
const NEAR_CAPACITY_PERCENT = 80           // 上下文使用率 > 80%
const MEMORY_HIGH_PERCENT = 5              // 内存文件占上下文 > 5%
const MEMORY_HIGH_TOKENS = 5_000           // 内存文件 > 5k 令牌
```

### 2.3 具体建议场景

#### 上下文接近满载
```typescript
if (data.percentage >= NEAR_CAPACITY_PERCENT) {
  suggestions.push({
    severity: 'warning',
    title: `Context is ${data.percentage}% full`,
    detail: data.isAutoCompactEnabled
      ? 'Autocompact will trigger soon, which discards older messages. Use /compact now to control what gets kept.'
      : 'Autocompact is disabled. Use /compact to free space, or enable autocompact in /config.',
  })
}
```

#### Bash 结果过大
```typescript
case BASH_TOOL_NAME:
  return {
    severity: 'warning',
    title: `Bash results using ${tokenStr} tokens (${percent.toFixed(0)}%)`,
    detail: 'Pipe output through head, tail, or grep to reduce result size. Avoid cat on large files — use Read with offset/limit instead.',
    savingsTokens: Math.floor(tokens * 0.5),
  }
```

#### Read 工具优化
```typescript
case FILE_READ_TOOL_NAME:
  return {
    severity: 'info',
    title: `Read results using ${tokenStr} tokens (${percent.toFixed(0)}%)`,
    detail: 'Use offset and limit parameters to read only the sections you need. Avoid re-reading entire files when you only need a few lines.',
    savingsTokens: Math.floor(tokens * 0.3),
  }
```

#### Grep 工具优化
```typescript
case GREP_TOOL_NAME:
  return {
    severity: 'info',
    title: `Grep results using ${tokenStr} tokens (${percent.toFixed(0)}%)`,
    detail: 'Add more specific patterns or use the glob or type parameter to narrow file types. Consider Glob for file discovery instead of Grep.',
    savingsTokens: Math.floor(tokens * 0.3),
  }
```

#### WebFetch 优化
```typescript
case WEB_FETCH_TOOL_NAME:
  return {
    severity: 'info',
    title: `WebFetch results using ${tokenStr} tokens (${percent.toFixed(0)}%)`,
    detail: 'Web page content can be very large. Consider extracting only the specific information needed.',
    savingsTokens: Math.floor(tokens * 0.4),
  }
```

#### 内存文件过大
```typescript
if (memoryPercent >= MEMORY_HIGH_PERCENT && totalMemoryTokens >= MEMORY_HIGH_TOKENS) {
  suggestions.push({
    severity: 'info',
    title: `Memory files using ${formatTokens(totalMemoryTokens)} tokens (${memoryPercent.toFixed(0)}%)`,
    detail: `Largest: ${largestFiles}. Use /memory to review and prune stale entries.`,
    savingsTokens: Math.floor(totalMemoryTokens * 0.3),
  })
}
```

#### 自动压缩禁用提醒
```typescript
if (!data.isAutoCompactEnabled && data.percentage >= 50 && data.percentage < NEAR_CAPACITY_PERCENT) {
  suggestions.push({
    severity: 'info',
    title: 'Autocompact is disabled',
    detail: 'Without autocompact, you will hit context limits and lose the conversation. Enable it in /config or use /compact manually.',
  })
}
```

### 2.4 建议排序

```typescript
suggestions.sort((a, b) => {
  // 1. 警告优先于信息
  if (a.severity !== b.severity) {
    return a.severity === 'warning' ? -1 : 1
  }
  // 2. 按可节省令牌数降序
  return (b.savingsTokens ?? 0) - (a.savingsTokens ?? 0)
})
```

## 3. 具体技术实现

### 3.1 主入口函数

```typescript
export function generateContextSuggestions(data: ContextData): ContextSuggestion[] {
  const suggestions: ContextSuggestion[] = []
  
  checkNearCapacity(data, suggestions)
  checkLargeToolResults(data, suggestions)
  checkReadResultBloat(data, suggestions)
  checkMemoryBloat(data, suggestions)
  checkAutoCompactDisabled(data, suggestions)
  
  // 排序：警告优先，然后按节省令牌数降序
  suggestions.sort(...)
  
  return suggestions
}
```

### 3.2 工具建议生成器

```typescript
function getLargeToolSuggestion(
  toolName: string,
  tokens: number,
  percent: number,
): ContextSuggestion | null {
  const tokenStr = formatTokens(tokens)
  
  switch (toolName) {
    case BASH_TOOL_NAME:      // 'Bash'
    case FILE_READ_TOOL_NAME: // 'Read'
    case GREP_TOOL_NAME:      // 'Grep'
    case WEB_FETCH_TOOL_NAME: // 'WebFetch'
    default:                  // 其他工具（>= 20% 才提示）
  }
}
```

### 3.3 工具名称常量

```typescript
import { BASH_TOOL_NAME } from '../tools/BashTool/toolName.js'      // 'Bash'
import { FILE_READ_TOOL_NAME } from '../tools/FileReadTool/prompt.js' // 'Read'
import { GREP_TOOL_NAME } from '../tools/GrepTool/prompt.js'          // 'Grep'
import { WEB_FETCH_TOOL_NAME } from '../tools/WebFetchTool/prompt.js' // 'WebFetch'
```

## 4. 关键代码路径与文件引用

### 4.1 核心导出

| 导出 | 用途 | 调用方 |
|------|------|--------|
| `generateContextSuggestions(data)` | 生成上下文建议 | `ContextSuggestions.tsx`, `ContextVisualization.tsx` |
| `ContextSuggestion` 类型 | 建议数据结构 | UI 组件 |
| `SuggestionSeverity` 类型 | 建议严重级别 | UI 组件 |

### 4.2 调用链

**上下文可视化**:
```
src/components/ContextVisualization.tsx
  └── generateContextSuggestions(contextData)
        └── 渲染建议列表
```

**上下文建议组件**:
```
src/components/ContextSuggestions.tsx
  └── generateContextSuggestions(contextData)
        └── 显示建议卡片
```

### 4.3 数据来源

```typescript
// 来自 analyzeContext.ts 的 ContextData
import type { ContextData } from './analyzeContext.js'

// ContextData 包含：
// - categories: 各类别的令牌数
// - percentage: 上下文使用率
// - rawMaxTokens: 最大令牌数
// - messageBreakdown: 消息详细分解
// - memoryFiles: 内存文件列表
// - isAutoCompactEnabled: 自动压缩状态
```

## 5. 依赖与外部交互

### 5.1 直接依赖

```typescript
import { BASH_TOOL_NAME } from '../tools/BashTool/toolName.js'
import { FILE_READ_TOOL_NAME } from '../tools/FileReadTool/prompt.js'
import { GREP_TOOL_NAME } from '../tools/GrepTool/prompt.js'
import { WEB_FETCH_TOOL_NAME } from '../tools/WebFetchTool/prompt.js'
import type { ContextData } from './analyzeContext.js'
import { getDisplayPath } from './file.js'
import { formatTokens } from './format.js'
```

### 5.2 被依赖情况

```
contextSuggestions.ts
    ↑
    ├── src/components/ContextVisualization.tsx
    └── src/components/ContextSuggestions.tsx
```

### 5.3 依赖关系图

```
contextSuggestions.ts
├── analyzeContext.ts (ContextData 类型)
├── format.ts (formatTokens)
├── file.ts (getDisplayPath)
└── 各工具的 toolName/prompt.ts (工具名称常量)
    ↑
    └── ContextVisualization.tsx / ContextSuggestions.tsx
```

## 6. 风险、边界与改进建议

### 6.1 风险分析

| 风险 | 可能性 | 影响 | 说明 |
|------|--------|------|------|
| 阈值不合理 | 中 | 中 | 固定阈值可能不适合所有使用场景 |
| 建议不准确 | 中 | 低 | 节省令牌数是估算值，实际节省可能不同 |
| 建议疲劳 | 中 | 中 | 频繁显示建议可能让用户忽视重要警告 |
| 工具名称变更 | 低 | 高 | 工具名称常量变更会导致匹配失败 |

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 无 messageBreakdown | 跳过工具相关检查 |
| 无 memoryFiles | 跳过内存文件检查 |
| 工具结果刚好在阈值边缘 | 使用严格比较（>=） |
| 多个工具都超过阈值 | 每个工具生成独立建议 |
| 已触发 LargeToolResults | ReadBloat 检查跳过（避免重复） |

### 6.3 改进建议

#### 短期
1. **可配置阈值**: 允许用户调整触发阈值：
   ```typescript
   const thresholds = {
     largeToolResultPercent: getConfig().contextSuggestionThresholds?.largeToolResultPercent ?? 15,
     // ...
   }
   ```

2. **建议去重**: 合并相似建议，避免重复提示：
   ```typescript
   // 如果 Bash 和 Read 都建议减少输出，合并为一条通用建议
   ```

3. **动态节省估算**: 基于历史数据改进节省估算：
   ```typescript
   // 分析用户采纳建议后的实际节省
   ```

#### 中期
4. **个性化建议**: 基于用户历史行为提供个性化建议：
   ```typescript
   // 如果用户经常使用某个工具，提供更具体的使用技巧
   ```

5. **交互式建议**: 提供一键应用建议的功能：
   ```typescript
   export type ContextSuggestion = {
     // ... 现有字段
     action?: { label: string; handler: () => Promise<void> }
   }
   ```

6. **建议历史**: 跟踪已显示的建议，避免重复提示：
   ```typescript
   // 记录上次显示时间，实现冷却期
   ```

#### 长期
7. **AI 驱动建议**: 使用轻量级模型分析上下文并生成建议：
   ```typescript
   export async function generateAIContextSuggestions(
     data: ContextData,
     messageHistory: Message[],
   ): Promise<ContextSuggestion[]>
   ```

8. **预测性建议**: 在用户遇到问题前主动提供建议：
   ```typescript
   // 基于趋势预测何时会达到上下文限制
   ```

### 6.4 测试建议

当前无直接测试，建议添加：
- 各阈值边界测试（刚好低于/高于阈值）
- 多工具建议排序测试
- 空数据测试
- 建议文本验证测试
- 节省令牌数计算测试
