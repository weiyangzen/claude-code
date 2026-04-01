# AgentProgressLine.tsx 深度研究文档

## 1. 场景与职责

### 1.1 功能定位
`AgentProgressLine` 是一个 React 组件，用于在终端 UI 中显示子代理（subagent）的执行进度状态。它是 Claude Code CLI 中多代理协作功能的核心 UI 组件，负责可视化展示每个子代理的执行状态、工具使用情况和 Token 消耗。

### 1.2 使用场景
- **多代理任务执行**：当主代理创建子代理执行并行或异步任务时，显示每个子代理的进度
- **树形进度展示**：在终端中以树形结构展示代理层级关系
- **实时状态更新**：动态显示代理的执行状态（初始化中、运行中、已完成、后台运行）

### 1.3 调用方
- 父组件通过 Props 传入代理状态和元数据
- 通常由 `AgentProgress` 或类似的父组件渲染
- 在 `showSpinnerTree` 配置启用时显示树形代理进度

---

## 2. 功能点目的

### 2.1 核心功能

| 功能点 | 目的 |
|--------|------|
| 树形结构渲染 | 使用 Unicode 树形字符 (`├─`, `└─`) 展示代理层级 |
| 状态可视化 | 通过颜色、动画和文本展示代理执行状态 |
| 工具使用统计 | 显示每个代理的工具调用次数 |
| Token 消耗 | 显示代理消耗的 Token 数量（格式化显示） |
| 异步/后台模式 | 区分前台执行和后台运行的代理 |

### 2.2 Props 接口

```typescript
type Props = {
  agentType: string;           // 代理类型标识
  description?: string;        // 代理描述
  name?: string;               // 代理名称
  descriptionColor?: keyof Theme;  // 描述文本颜色
  taskDescription?: string;    // 任务描述
  toolUseCount: number;        // 工具使用次数
  tokens: number | null;       // Token 消耗数量
  color?: keyof Theme;         // 代理类型标签颜色
  isLast: boolean;             // 是否为树中最后一个节点
  isResolved: boolean;         // 是否已完成
  isError: boolean;            // 是否出错
  isAsync?: boolean;           // 是否为异步代理
  shouldAnimate: boolean;      // 是否显示动画
  lastToolInfo?: string | null; // 最后使用的工具信息
  hideType?: boolean;          // 是否隐藏类型标签
}
```

---

## 3. 具体技术实现

### 3.1 状态文本逻辑

```typescript
const getStatusText = () => {
  if (!isResolved) {
    return lastToolInfo || "Initializing…"
  }
  if (isBackgrounded) {
    return taskDescription ?? "Running in the background"
  }
  return "Done"
}
```

状态优先级：
1. **未完成 (`!isResolved`)**：显示最后工具信息或 "Initializing…"
2. **后台运行 (`isBackgrounded`)**：显示任务描述或 "Running in the background"
3. **已完成**：显示 "Done"

### 3.2 树形字符选择

```typescript
const treeChar = isLast ? "\u2514\u2500" : "\u251C\u2500"  // └─ vs ├─
```

使用 Unicode Box Drawing 字符创建视觉树形结构。

### 3.3 样式渲染逻辑

**标准模式 (`!hideType`)**：
- 显示带背景色的代理类型标签
- 可选显示描述文本（带独立颜色）

**简洁模式 (`hideType`)**：
- 显示代理名称或描述
- 名称和描述同时存在时显示为 `name: description` 格式

### 3.4 统计信息展示

```typescript
// 非后台模式显示工具使用和 Token 统计
!isBackgrounded && (
  <> · {toolUseCount} tool {toolUseCount === 1 ? "use" : "uses"}
  {tokens !== null && <> · {formatNumber(tokens)} tokens</>}</>
)
```

### 3.5 依赖工具函数

**formatNumber** (`src/utils/format.ts`):
```typescript
export function formatNumber(number: number): string {
  const shouldUseConsistentDecimals = number >= 1000
  return getNumberFormatter(shouldUseConsistentDecimals)
    .format(number)  // eg. "1321" => "1.3K"
    .toLowerCase()   // eg. "1.3K" => "1.3k"
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 文件位置
```
src/components/AgentProgressLine.tsx
```

### 4.2 依赖关系

```
AgentProgressLine.tsx
├── react (React 运行时)
├── ../ink.js (Ink TUI 组件库 - Box, Text)
├── ../utils/format.js (formatNumber)
└── ../utils/theme.js (Theme 类型)
```

### 4.3 主题颜色键值

来自 `src/utils/theme.ts` 的 `Theme` 类型：
- `autoAccept`, `bashBorder`, `claude`, `permission`, `planMode`
- `ide`, `promptBorder`, `text`, `inverseText`, `inactive`
- `subtle`, `suggestion`, `remember`, `background`
- `success`, `error`, `warning`, `merged`
- 子代理专用颜色：`red_FOR_SUBAGENTS_ONLY`, `blue_FOR_SUBAGENTS_ONLY` 等

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 |
|------|------|
| `react` | React 组件运行时 |
| `ink` | 终端 UI 渲染库 |
| `formatNumber` | 数字格式化（K/M 缩写） |
| `Theme` | 主题类型定义 |

### 5.2 数据流

```
父组件
  │
  ▼
AgentProgressLine Props
  │
  ├── agentType → 类型标签渲染
  ├── isResolved/isAsync → 状态判断
  ├── toolUseCount/tokens → 统计信息
  └── lastToolInfo → 状态文本
  │
  ▼
Ink 组件渲染 (Box, Text)
  │
  ▼
终端输出
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| Token 为 null | tokens 可能为 null，需要条件渲染 | 已实现 `tokens !== null` 检查 |
| 长文本截断 | 描述文本过长可能影响布局 | 依赖 Ink 的 `wrap` 属性 |
| 颜色键无效 | descriptionColor/color 可能传入无效键 | TypeScript 类型检查 |

### 6.2 边界情况

1. **空状态**：当 `toolUseCount` 为 0 且 `tokens` 为 null 时，统计信息部分为空
2. **后台模式**：`isBackgrounded` 为 true 时隐藏工具统计，避免干扰
3. **最后一个节点**：`isLast` 影响树形字符和连接线样式

### 6.3 改进建议

1. **性能优化**：
   - 考虑使用 React.memo 避免不必要的重渲染
   - 频繁的状态更新可能导致终端闪烁

2. **可访问性**：
   - 添加对屏幕阅读器的支持
   - 提供纯文本模式（无颜色/无 Unicode）

3. **功能扩展**：
   - 支持显示执行时间
   - 添加进度百分比（如果可用）
   - 支持点击/交互展开详细信息

4. **代码质量**：
   - 将复杂的 JSX 逻辑提取为子组件
   - 添加单元测试覆盖各种状态组合

### 6.4 相关 Issue 参考

- 该组件与 `showSpinnerTree` 配置相关
- 子代理颜色系统 (`*_FOR_SUBAGENTS_ONLY`) 用于区分不同代理
