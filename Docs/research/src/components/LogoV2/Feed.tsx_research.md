# Feed.tsx 深度研究文档

## 场景与职责

Feed.tsx 是 Claude Code 终端 UI 的 LogoV2 组件系统中的核心展示组件，负责渲染信息 feeds（信息流）。它主要用于 Claude Code 启动时的欢迎屏幕右侧信息面板，展示以下内容：

1. **Recent activity（最近活动）** - 显示用户最近的对话历史
2. **What's new（更新内容）** - 显示版本更新日志
3. **Project onboarding（项目引导）** - 新用户引导步骤
4. **Guest passes（访客通行证）** - 推广信息展示

该组件采用 React + Ink（终端渲染库）架构，支持响应式布局和智能宽度计算。

## 功能点目的

### 1. 类型定义

```typescript
export type FeedLine = {
  text: string;
  timestamp?: string;
};

export type FeedConfig = {
  title: string;
  lines: FeedLine[];
  footer?: string;
  emptyMessage?: string;
  customContent?: {
    content: React.ReactNode;
    width: number;
  };
};
```

- **FeedLine**: 定义单行内容的结构，支持可选的时间戳
- **FeedConfig**: 定义整个 feed 的配置，包括标题、内容行、页脚、空状态消息和自定义内容

### 2. calculateFeedWidth 函数

**目的**: 计算 feed 内容所需的最小宽度，用于父组件进行布局决策。

**算法逻辑**:
1. 从标题宽度开始
2. 如果有自定义内容，使用自定义内容宽度
3. 如果无内容且有 emptyMessage，使用 emptyMessage 宽度
4. 否则计算所有行的最大宽度（考虑时间戳列对齐）
5. 最后与页脚宽度比较取最大值

**关键代码路径**:
```typescript
// 时间戳宽度计算 - 用于列对齐
const maxTimestampWidth = Math.max(0, ...lines.map(line => 
  line.timestamp ? stringWidth(line.timestamp) : 0
));

// 每行宽度 = 文本宽度 + 时间戳宽度 + 间隔
const lineWidth = stringWidth(line.text) + 
  (timestampWidth > 0 ? timestampWidth + gap.length : 0);
```

### 3. Feed 组件

**目的**: 渲染一个格式化的信息面板。

**渲染逻辑**:
1. **标题**: 使用 `claude` 主题色加粗显示
2. **内容区域**:
   - 优先显示 `customContent`（如 Guest passes 的自定义 UI）
   - 无内容时显示 `emptyMessage`（灰色暗淡）
   - 正常内容行：时间戳右对齐 + 两空格间隔 + 文本
3. **页脚**: 灰色斜体显示

**React Compiler 优化**:
代码中使用了 `_c(15)` 进行 memoization，通过比较依赖项（lines, title, actualWidth 等）来避免不必要的重渲染。

## 具体技术实现

### 关键流程

1. **宽度计算流程**:
   ```
   FeedConfig → calculateFeedWidth → 各元素宽度比较 → 最大宽度
   ```

2. **渲染流程**:
   ```
   Feed props → 解构 config → 计算 maxTimestampWidth → 
   条件渲染(customContent/emptyMessage/lines) → 返回 Box 包裹的 JSX
   ```

3. **时间戳对齐机制**:
   - 计算所有行中最大时间戳宽度
   - 使用 `String.prototype.padEnd()` 进行右对齐填充
   - 时间戳与文本之间固定两个空格间隔

### 数据结构

```typescript
// 内部渲染使用的计算值
const maxTimestampWidth = Math.max(0, ...lines.map(line => 
  line.timestamp ? stringWidth(line.timestamp) : 0
));

// 文本可用宽度 = 实际宽度 - 时间戳占用宽度 - 间隔
const textWidth = Math.max(10, 
  actualWidth - (maxTimestampWidth > 0 ? maxTimestampWidth + 2 : 0)
);
```

### 协议/接口

- **输入**: `FeedProps` 包含 `config` 和 `actualWidth`
- **输出**: React 元素（Ink Box 包裹的格式化内容）
- **宽度约束**: 文本内容会被 `truncate` 函数截断以适应可用宽度

## 关键代码路径与文件引用

### 直接依赖

| 文件 | 用途 |
|------|------|
| `src/ink/stringWidth.ts` | 计算字符串显示宽度（处理 CJK、emoji 等宽字符） |
| `src/ink.ts` | Ink 组件库（Box, Text） |
| `src/utils/format.ts` | truncate 函数用于文本截断 |

### 调用方

| 文件 | 调用方式 |
|------|----------|
| `src/components/LogoV2/FeedColumn.tsx` | 导入并渲染 Feed 组件 |
| `src/components/LogoV2/feedConfigs.tsx` | 创建 FeedConfig 对象 |
| `src/components/LogoV2/LogoV2.tsx` | 通过 FeedColumn 间接使用 |

### 核心代码片段

**Feed.tsx 第 24-50 行 - 宽度计算**:
```typescript
export function calculateFeedWidth(config: FeedConfig): number {
  const { title, lines, footer, emptyMessage, customContent } = config;
  let maxWidth = stringWidth(title);
  
  if (customContent !== undefined) {
    maxWidth = Math.max(maxWidth, customContent.width);
  } else if (lines.length === 0 && emptyMessage) {
    maxWidth = Math.max(maxWidth, stringWidth(emptyMessage));
  } else {
    const gap = '  ';
    const maxTimestampWidth = Math.max(0, ...lines.map(line => 
      line.timestamp ? stringWidth(line.timestamp) : 0
));
    for (const line of lines) {
      const timestampWidth = maxTimestampWidth > 0 ? maxTimestampWidth : 0;
      const lineWidth = stringWidth(line.text) + 
        (timestampWidth > 0 ? timestampWidth + gap.length : 0);
      maxWidth = Math.max(maxWidth, lineWidth);
    }
  }
  if (footer) {
    maxWidth = Math.max(maxWidth, stringWidth(footer));
  }
  return maxWidth;
}
```

**Feed.tsx 第 82-86 行 - 条件渲染逻辑**:
```typescript
t3 = customContent 
  ? <>{customContent.content}{footer && <Text dimColor italic>{truncate(footer, actualWidth)}</Text>}</>
  : lines.length === 0 && emptyMessage 
    ? <Text dimColor>{truncate(emptyMessage, actualWidth)}</Text>
    : <>{lines.map((line, index) => {
        const textWidth = Math.max(10, actualWidth - (maxTimestampWidth > 0 ? maxTimestampWidth + 2 : 0));
        return <Text key={index}>
          {maxTimestampWidth > 0 && <><Text dimColor>{(line.timestamp || "").padEnd(maxTimestampWidth)}</Text>{"  "}</>}
          <Text>{truncate(line.text, textWidth)}</Text>
        </Text>;
      })}{footer && <Text dimColor italic>{truncate(footer, actualWidth)}</Text>}</>;
```

## 依赖与外部交互

### 运行时依赖

1. **react/compiler-runtime**: 用于 React Compiler 的缓存机制 (`_c` 函数)
2. **ink**: 终端 UI 渲染库
3. **stringWidth**: 精确的字符串宽度计算（处理 Unicode、emoji、ANSI）

### 编译时依赖

- TypeScript 类型系统
- React Compiler（用于自动 memoization）

### 相关配置类型

来自 `feedConfigs.tsx` 的实际使用示例：
```typescript
// Recent activity feed
export function createRecentActivityFeed(activities: LogOption[]): FeedConfig {
  const lines: FeedLine[] = activities.map(log => {
    const time = formatRelativeTimeAgo(log.modified);
    const description = log.summary && log.summary !== 'No prompt' 
      ? log.summary 
      : log.firstPrompt;
    return { text: description || '', timestamp: time };
  });
  return {
    title: 'Recent activity',
    lines,
    footer: lines.length > 0 ? '/resume for more' : undefined,
    emptyMessage: 'No recent activity'
  };
}
```

## 风险、边界与改进建议

### 已知风险

1. **宽度计算与实际渲染不一致**: 
   - `calculateFeedWidth` 和 `Feed` 组件分别计算宽度，存在逻辑重复
   - 如果两者实现不同步，可能导致布局问题

2. **React Compiler 依赖**:
   - 代码严重依赖 React Compiler 的 memoization
   - 如果编译器配置变更，可能影响性能

3. **硬编码值**:
   - `gap = '  '`（两个空格）是硬编码的
   - 最小文本宽度 `Math.max(10, ...)` 是魔法数字

### 边界情况

1. **空内容处理**:
   - `lines.length === 0` 时显示 `emptyMessage`
   - 但 `customContent` 优先级高于空消息

2. **宽度溢出**:
   - `actualWidth` 可能小于内容所需最小宽度
   - `truncate` 函数确保不会溢出，但可能导致内容被截断

3. **时间戳对齐**:
   - 只有部分行有时间戳时，无时间戳的行也会预留空间
   - 通过 `(line.timestamp || "").padEnd(maxTimestampWidth)` 实现

### 改进建议

1. **代码复用**:
   ```typescript
   // 建议：提取共享的宽度计算逻辑
   function calculateLineWidth(line: FeedLine, maxTimestampWidth: number): number {
     const gap = '  ';
     return stringWidth(line.text) + 
       (line.timestamp ? maxTimestampWidth + gap.length : 0);
   }
   ```

2. **配置化**:
   - 将 `gap` 长度、`minTextWidth` 等提取为配置参数
   - 支持自定义时间戳格式

3. **类型安全**:
   - `customContent.width` 是必需的，但使用时没有验证
   - 建议添加运行时宽度验证

4. **测试覆盖**:
   - 需要测试 CJK 字符、emoji、ANSI 颜色码的宽度计算
   - 测试极端宽度情况（`actualWidth < 10`）

5. **性能优化**:
   - `calculateFeedWidth` 在 `FeedColumn` 中对每个 feed 调用一次
   - 对于大量 feeds，可以考虑缓存计算结果

### 相关文件变更影响

- `src/ink/stringWidth.ts`: 如果宽度计算逻辑变更，会直接影响 Feed 布局
- `src/utils/truncate.ts`: 截断逻辑变更会影响内容显示
- `src/components/design-system/Divider.tsx`: FeedColumn 中用于分隔 feeds
