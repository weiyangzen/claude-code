# UserPromptMessage.tsx 研究文档

## 场景与职责

`UserPromptMessage` 是一个 React 组件，用于渲染用户的主要提示消息。这是 Claude Code 中最核心的用户消息组件，处理普通用户输入、Brief 模式布局、文本截断和性能优化。

**核心职责：**
- 渲染用户的主要提示文本
- 支持 Brief 模式（聊天样式）布局
- 处理大文本的性能优化（截断显示）
- 集成 KAIROS 功能（实验性功能）
- 支持消息操作选中状态

## 功能点目的

1. **主消息渲染**：显示用户输入的核心提示内容
2. **Brief 模式**：支持简洁的聊天样式布局（You 标签 + 时间戳）
3. **性能优化**：对大文本进行头尾截断，避免渲染性能问题
4. **功能开关**：集成 GrowthBook 功能开关控制 Brief 模式
5. **选中状态**：支持消息操作系统的选中高亮

## 具体技术实现

### 关键流程

```
输入: addMargin, param (TextBlockParam), isTranscriptMode?, timestamp?
  ↓
检查 KAIROS 功能开关
  ↓
如果是 KAIROS 构建：
  获取 isBriefOnly 状态
  获取 viewingAgentTaskId
  检查 briefEnvEnabled
  计算 useBriefLayout
  ↓
截断文本（如果超过 10000 字符）
  ↓
获取选中状态 isSelected
  ↓
如果文本为空：记录错误并返回 null
  ↓
渲染 Box 容器（根据状态设置背景色）
  ↓
渲染 HighlightedThinkingText（处理 Brief/普通布局）
```

### 数据结构

**Props 接口：**
```typescript
{
  addMargin: boolean           // 是否在顶部添加边距
  param: TextBlockParam        // 包含 text 的文本块
  isTranscriptMode?: boolean   // 是否为转录模式
  timestamp?: string           // 时间戳（Brief 模式使用）
}

// TextBlockParam 结构
{
  text: string  // 用户输入的提示文本
}
```

**文本截断常量：**
```typescript
const MAX_DISPLAY_CHARS = 10_000      // 最大显示字符数
const TRUNCATE_HEAD_CHARS = 2_500     // 头部保留字符数
const TRUNCATE_TAIL_CHARS = 2_500     // 尾部保留字符数
```

### 关键代码路径

**文件位置：** `src/components/messages/UserPromptMessage.tsx`

**Brief 模式计算（KAIROS 功能）：**
```typescript
const isBriefOnly = feature('KAIROS') || feature('KAIROS_BRIEF') ?
  useAppState(s => s.isBriefOnly) : false;

const viewingAgentTaskId = feature('KAIROS') || feature('KAIROS_BRIEF') ?
  useAppState(s => s.viewingAgentTaskId) : null;

const briefEnvEnabled = feature('KAIROS') || feature('KAIROS_BRIEF') ?
  useMemo(() => isEnvTruthy(process.env.CLAUDE_CODE_BRIEF), []) : false;

const useBriefLayout = feature('KAIROS') || feature('KAIROS_BRIEF') ?
  (getKairosActive() || getUserMsgOptIn() && 
   (briefEnvEnabled || getFeatureValue_CACHED_MAY_BE_STALE('tengu_kairos_brief', false))) &&
  isBriefOnly && !isTranscriptMode && !viewingAgentTaskId : false;
```

**文本截断逻辑：**
```typescript
const displayText = useMemo(() => {
  if (text.length <= MAX_DISPLAY_CHARS) return text;
  const head = text.slice(0, TRUNCATE_HEAD_CHARS);
  const tail = text.slice(-TRUNCATE_TAIL_CHARS);
  const hiddenLines = countCharInString(text, '\n', TRUNCATE_HEAD_CHARS) - 
                      countCharInString(tail, '\n');
  return `${head}\n… +${hiddenLines} lines …\n${tail}`;
}, [text]);
```

**背景色计算：**
```typescript
backgroundColor={
  isSelected ? 'messageActionsBackground' : 
  useBriefLayout ? undefined : 
  'userMessageBackground'
}
```

**渲染结构：**
```tsx
<Box 
  flexDirection="column" 
  marginTop={addMargin ? 1 : 0}
  backgroundColor={...}
  paddingRight={useBriefLayout ? 0 : 1}
>
  <HighlightedThinkingText 
    text={displayText} 
    useBriefLayout={useBriefLayout} 
    timestamp={useBriefLayout ? timestamp : undefined}
  />
</Box>
```

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| feature | 'bun:bundle' | 编译时功能开关 |
| TextBlockParam | '@anthropic-ai/sdk/resources/index.mjs' | SDK 类型 |
| React | 'react' | UI 框架 |
| getKairosActive, getUserMsgOptIn | '../../bootstrap/state.js' | KAIROS 状态 |
| Box | '../../ink.js' | 终端 UI 组件 |
| getFeatureValue_CACHED_MAY_BE_STALE | '../../services/analytics/growthbook.js' | 功能开关值 |
| useAppState | '../../state/AppState.js' | 应用状态管理 |
| isEnvTruthy | '../../utils/envUtils.js' | 环境变量检查 |
| logError | '../../utils/log.js' | 错误日志 |
| countCharInString | '../../utils/stringUtils.js' | 字符计数 |
| MessageActionsSelectedContext | '../messageActions.js' | 消息操作上下文 |
| HighlightedThinkingText | './HighlightedThinkingText.js' | 文本渲染组件 |

### 相关常量与工具

**envUtils.js：**
```typescript
export function isEnvTruthy(envVar: string | boolean | undefined): boolean {
  if (!envVar) return false
  if (typeof envVar === 'boolean') return envVar
  const normalizedValue = envVar.toLowerCase().trim()
  return ['1', 'true', 'yes', 'on'].includes(normalizedValue)
}
```

**stringUtils.js：**
```typescript
export function countCharInString(
  str: { indexOf(search: string, start?: number): number },
  char: string,
  start = 0,
): number {
  let count = 0
  let i = str.indexOf(char, start)
  while (i !== -1) {
    count++
    i = str.indexOf(char, i + 1)
  }
  return count
}
```

### HighlightedThinkingText 组件

处理两种布局模式：
1. **Brief 布局**：显示 "You" 标签 + 时间戳，文本颜色根据队列状态变化
2. **普通布局**：显示指针图标，支持彩虹色高亮思考触发词

## 风险、边界与改进建议

### 潜在风险

1. **Hook 条件调用**：在 feature() 条件内使用 hook，虽然注释说明 feature() 是编译时常量，但仍存在风险
2. **性能问题**：即使使用 React.memo，Ink 输出遍历仍可能导致大文本的按键延迟
3. **截断信息丢失**：截断时只显示隐藏行数，不显示隐藏字符数

### 边界情况

1. **空文本**：记录错误并返回 null
2. **超大文本（>10000 字符）**：头尾截断，中间显示省略
3. **Brief 模式条件复杂**：多个条件组合决定是否使用 Brief 布局
4. **转录模式**：禁用 Brief 布局
5. **查看 Agent 任务时**：禁用 Brief 布局

### 改进建议

1. **虚拟滚动**：对于超大文本，考虑使用虚拟滚动只渲染可见部分
2. **渐进加载**：大文本可以渐进加载，先显示前 N 行
3. **搜索功能**：在大文本中搜索关键词
4. **复制优化**：提供复制完整文本（非截断版本）的功能
5. **字数统计**：显示总字符数和截断信息
6. **可配置截断**：允许用户配置截断阈值
7. **Hook 重构**：将条件 hook 调用重构为更安全的形式

### 测试建议

1. 各种文本长度测试（空、短、长、超长）
2. Brief 模式条件组合测试
3. 截断逻辑准确性测试
4. 性能测试（大文本渲染延迟）
5. 选中状态样式测试
6. 特殊字符和编码测试
