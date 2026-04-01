# heatmap.ts 深度研究

## 场景与职责

本模块生成 GitHub 风格的终端活动热力图，用于 `/stats` 命令展示用户的 Claude Code 使用活动。通过可视化每日消息数量，帮助用户了解自己的使用模式和活跃程度。

**核心场景：**
1. **活动可视化**：在终端显示过去 N 周的活动热力图
2. **使用统计展示**：配合 `/stats` 命令提供图形化反馈
3. **用户参与度追踪**：通过连续活跃天数等指标激励用户

## 功能点目的

### 1. GitHub 风格热力图
- **目的**：直观展示活动时间分布
- **风格**：7 行（周日到周六）× N 列（周）的网格
- **配色**：Claude 橙色（#da7756）的 4 级强度

### 2. 自适应强度计算
- **目的**：根据数据分布自动调整颜色强度
- **算法**：基于百分位数（P25, P50, P75）划分 4 级
- **优势**：适应不同使用强度的用户

### 3. 响应式布局
- **目的**：适应不同终端宽度
- **计算**：根据终端宽度动态计算可显示的周数
- **上限**：最多 52 周（1 年）

## 具体技术实现

### 核心数据结构
```typescript
export type HeatmapOptions = {
  terminalWidth?: number  // 终端宽度（字符数）
  showMonthLabels?: boolean  // 是否显示月份标签
}

type Percentiles = {
  p25: number
  p50: number
  p75: number
}

type DailyActivity = {
  date: string  // YYYY-MM-DD 格式
  messageCount: number
  sessionCount: number
  toolCallCount: number
}
```

### 关键流程

#### generateHeatmap() - 主入口
```typescript
export function generateHeatmap(
  dailyActivity: DailyActivity[],
  options: HeatmapOptions = {},
): string {
  const { terminalWidth = 80, showMonthLabels = true } = options
  
  // 1. 计算布局
  const dayLabelWidth = 4
  const availableWidth = terminalWidth - dayLabelWidth
  const width = Math.min(52, Math.max(10, availableWidth))
  
  // 2. 构建活动映射
  const activityMap = new Map<string, DailyActivity>()
  for (const activity of dailyActivity) {
    activityMap.set(activity.date, activity)
  }
  
  // 3. 计算百分位数
  const percentiles = calculatePercentiles(dailyActivity)
  
  // 4. 计算日期范围（从今天回溯 N 周）
  const today = new Date()
  today.setHours(0, 0, 0, 0)
  const currentWeekStart = new Date(today)
  currentWeekStart.setDate(today.getDate() - today.getDay())
  const startDate = new Date(currentWeekStart)
  startDate.setDate(startDate.getDate() - (width - 1) * 7)
  
  // 5. 生成网格
  const grid: string[][] = Array.from({ length: 7 }, () => Array(width).fill(''))
  const monthStarts: { month: number; week: number }[] = []
  
  const currentDate = new Date(startDate)
  for (let week = 0; week < width; week++) {
    for (let day = 0; day < 7; day++) {
      if (currentDate > today) {
        grid[day]![week] = ' '
      } else {
        const dateStr = toDateString(currentDate)
        const activity = activityMap.get(dateStr)
        const intensity = getIntensity(activity?.messageCount || 0, percentiles)
        grid[day]![week] = getHeatmapChar(intensity)
      }
      currentDate.setDate(currentDate.getDate() + 1)
    }
  }
  
  // 6. 构建输出（月份标签 + 网格 + 图例）
  // ...
  return lines.join('\n')
}
```

#### calculatePercentiles() - 百分位数计算
```typescript
function calculatePercentiles(dailyActivity: DailyActivity[]): Percentiles | null {
  const counts = dailyActivity
    .map(a => a.messageCount)
    .filter(c => c > 0)
    .sort((a, b) => a - b)
  
  if (counts.length === 0) return null
  
  return {
    p25: counts[Math.floor(counts.length * 0.25)]!,
    p50: counts[Math.floor(counts.length * 0.5)]!,
    p75: counts[Math.floor(counts.length * 0.75)]!,
  }
}
```

#### 强度映射
```typescript
function getIntensity(count: number, percentiles: Percentiles | null): number {
  if (count === 0 || !percentiles) return 0
  if (count >= percentiles.p75) return 4
  if (count >= percentiles.p50) return 3
  if (count >= percentiles.p25) return 2
  return 1
}

function getHeatmapChar(intensity: number): string {
  switch (intensity) {
    case 0: return chalk.gray('·')      // 无活动
    case 1: return claudeOrange('░')    // 低强度
    case 2: return claudeOrange('▒')    // 中低强度
    case 3: return claudeOrange('▓')    // 中高强度
    case 4: return claudeOrange('█')    // 高强度
    default: return chalk.gray('·')
  }
}
```

### 输出格式示例
```
    Jan      Feb      Mar      Apr      May
Sun ·······································
Mon ·░·····································
Tue ·▒·····································
Wed ·█·····································
Thu ·▓·····································
Fri ·░·····································
Sat ·······································

    Less ░ ▒ ▓ █ More
```

## 关键代码路径与文件引用

### 本文件导出
| 导出 | 类型 | 用途 |
|------|------|------|
| `generateHeatmap` | 函数 | 生成热力图字符串 |
| `HeatmapOptions` | 类型 | 配置选项类型 |

### 调用方
1. **commands/stats/stats.tsx**: `/stats` 命令
2. **components/Stats.tsx**: 统计组件

### 依赖模块
```typescript
import chalk from 'chalk'
import type { DailyActivity } from './stats.js'
import { toDateString } from './statsCache.js'
```

## 依赖与外部交互

### 上游依赖

1. **stats.ts**: 统计数据
   - `DailyActivity` 类型定义
   - 活动数据聚合逻辑

2. **statsCache.ts**: 日期工具
   - `toDateString()`: 日期格式化为 YYYY-MM-DD

3. **chalk**: 终端颜色
   - `chalk.hex('#da7756')`: Claude 橙色
   - `chalk.gray()`: 灰色（无活动）

### 数据流
```
会话数据 → stats.ts (aggregateClaudeCodeStats) → DailyActivity[] → heatmap.ts → 终端字符串
```

## 风险、边界与改进建议

### 已知风险

1. **时区问题**
   - 风险：日期计算可能受时区影响
   - 缓解：使用本地时间，与 GitHub 行为一致

2. **终端宽度变化**
   - 风险：运行时终端宽度可能改变
   - 缓解：调用时传入当前宽度

3. **颜色支持**
   - 风险：某些终端可能不支持 256 色
   - 现状：使用基本 ANSI 颜色，兼容性较好

### 边界情况

1. **无活动数据**：全部显示为 `·`
2. **单日活动**：只有一个单元格有颜色
3. **超窄终端**：最小 10 周宽度，可能截断
4. **未来日期**：显示为空格
5. **闰年/ DST**：Date 对象自动处理

### 改进建议

1. **交互式热力图**
   - 建议：支持悬停显示具体数值
   - 实现：使用 Ink 的鼠标事件

2. **多维度热力图**
   - 建议：切换显示消息数/会话数/工具调用数
   - 收益：更全面的活动视图

3. **对比模式**
   - 建议：显示去年同期对比
   - 场景：年度使用趋势分析

4. **自定义时间范围**
   - 建议：支持指定起始/结束日期
   - 实现：扩展 HeatmapOptions

5. **导出功能**
   - 建议：支持导出为图片或 SVG
   - 场景：分享使用统计

6. **动画效果**
   - 建议：数据更新时的过渡动画
   - 实现：Ink 的动画支持

### 测试要点

1. 不同终端宽度下的布局
2. 百分位数计算正确性
3. 日期范围计算（边界周）
4. 无数据/少数据场景
5. 跨月/跨年的月份标签
6. 颜色输出正确性
