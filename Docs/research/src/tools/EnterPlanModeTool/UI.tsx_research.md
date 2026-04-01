# UI.tsx 研究文档

## 场景与职责

UI.tsx 是 EnterPlanModeTool 的**用户界面渲染模块**，负责在终端 UI 中展示计划模式进入的相关视觉反馈。该模块使用 React 和 Ink（React for CLI）构建终端用户界面。

### 核心职责
1. **工具使用消息渲染**：展示用户请求进入计划模式时的视觉反馈
2. **工具结果消息渲染**：展示成功进入计划模式后的确认信息
3. **工具拒绝消息渲染**：展示用户拒绝进入计划模式时的反馈
4. **主题适配**：根据当前主题颜色渲染适当的视觉元素

### 使用场景
- 用户在权限对话框中批准进入计划模式后显示确认
- 用户拒绝进入计划模式时显示反馈
- 提供计划模式状态的视觉指示（通过颜色编码）

---

## 功能点目的

### 1. renderToolUseMessage
- **目的**：渲染工具使用时的消息（模型调用工具时）
- **当前实现**：返回 `null`，表示不显示特定的工具使用消息
- **原因**：EnterPlanMode 的权限请求由通用权限系统处理，不需要自定义工具使用 UI

### 2. renderToolResultMessage
- **目的**：渲染成功进入计划模式后的确认消息
- **视觉元素**：
  - 黑色圆圈图标（`BLACK_CIRCLE`）
  - 计划模式对应的颜色（`getModeColor('plan')`）
  - "Entered plan mode" 标题
  - "Claude is now exploring and designing..." 描述文本

### 3. renderToolUseRejectedMessage
- **目的**：渲染用户拒绝进入计划模式时的消息
- **视觉元素**：
  - 黑色圆圈图标
  - 默认模式颜色
  - "User declined to enter plan mode" 文本

---

## 具体技术实现

### 关键代码路径

**1. 导入依赖**
```typescript
import * as React from 'react'
import { BLACK_CIRCLE } from 'src/constants/figures.js'
import { getModeColor } from 'src/utils/permissions/PermissionMode.js'
import { Box, Text } from '../../ink.js'
import type { ToolProgressData } from '../../Tool.js'
import type { ProgressMessage } from '../../types/message.js'
import type { ThemeName } from '../../utils/theme.js'
import type { Output } from './EnterPlanModeTool.js'
```

**2. renderToolUseMessage 实现**
```typescript
export function renderToolUseMessage(): React.ReactNode {
  return null  // 不渲染自定义工具使用消息
}
```

**3. renderToolResultMessage 实现**
```typescript
export function renderToolResultMessage(
  _output: Output,
  _progressMessagesForMessage: ProgressMessage<ToolProgressData>[],
  _options: { theme: ThemeName },
): React.ReactNode {
  return (
    <Box flexDirection="column" marginTop={1}>
      <Box flexDirection="row">
        <Text color={getModeColor('plan')}>{BLACK_CIRCLE}</Text>
        <Text> Entered plan mode</Text>
      </Box>
      <Box paddingLeft={2}>
        <Text dimColor>
          Claude is now exploring and designing an implementation approach.
        </Text>
      </Box>
    </Box>
  )
}
```

**布局结构分析：**
```
┌─────────────────────────────────────┐
│ ● Entered plan mode                 │  ← 主标题行（带颜色编码的图标）
│   Claude is now exploring and       │  ← 描述文本（缩进2格，dimColor）
│   designing an implementation       │
│   approach.                         │
└─────────────────────────────────────┘
```

**4. renderToolUseRejectedMessage 实现**
```typescript
export function renderToolUseRejectedMessage(): React.ReactNode {
  return (
    <Box flexDirection="row" marginTop={1}>
      <Text color={getModeColor('default')}>{BLACK_CIRCLE}</Text>
      <Text> User declined to enter plan mode</Text>
    </Box>
  )
}
```

### 使用的视觉元素

| 元素 | 来源 | 用途 |
|------|------|------|
| `BLACK_CIRCLE` | `src/constants/figures.js` | 状态指示图标（⏺ 或 ●） |
| `getModeColor('plan')` | `src/utils/permissions/PermissionMode.js` | 计划模式主题色 |
| `getModeColor('default')` | `src/utils/permissions/PermissionMode.js` | 默认模式主题色 |
| `Box` | `../../ink.js` | 布局容器 |
| `Text` | `../../ink.js` | 文本渲染 |

### 颜色编码系统

计划模式使用特定的颜色编码（来自 `PermissionMode.ts`）：
```typescript
const PERMISSION_MODE_CONFIG = {
  plan: {
    color: 'planMode',  // 特定的计划模式颜色
    symbol: PAUSE_ICON, // ⏸
    // ...
  }
}
```

---

## 依赖与外部交互

### 依赖模块详解

**1. src/constants/figures.js**
```typescript
export const BLACK_CIRCLE = env.platform === 'darwin' ? '⏺' : '●'
```
- macOS 使用 ⏺（更好垂直对齐）
- Windows/Linux 使用 ●（通用支持）

**2. src/utils/permissions/PermissionMode.js**
```typescript
export function getModeColor(mode: PermissionMode): ModeColorKey {
  return getModeConfig(mode).color
}
```
- 提供权限模式到主题颜色的映射
- 确保视觉一致性和可访问性

**3. ../../ink.js**
- 项目内部的 Ink（React for CLI）封装
- 提供 `Box` 和 `Text` 组件用于终端 UI 构建

### 类型依赖

```typescript
// 来自 Tool.js - 工具进度数据类型
type ToolProgressData = /* ... */

// 来自 types/message.js - 进度消息类型
type ProgressMessage<T> = /* ... */

// 来自 utils/theme.js - 主题名称类型
type ThemeName = /* ... */

// 来自 EnterPlanModeTool.js - 输出类型
type Output = { message: string }
```

---

## 风险、边界与改进建议

### 已知限制

**1. 静态文本**
- 当前：所有文本都是硬编码的英文
- 影响：不支持国际化（i18n）
- 建议：如需多语言支持，应引入文本资源系统

**2. 有限的视觉反馈**
- 当前：仅显示简单的文本确认
- 对比：其他工具可能有更丰富的进度指示或动画
- 建议：考虑添加进入计划模式后的操作提示

**3. 主题依赖**
- 当前：依赖 `getModeColor('plan')` 返回有效颜色
- 风险：如果主题配置不完整，可能显示默认颜色

### 边界条件

| 场景 | 行为 |
|------|------|
| 主题切换 | 颜色自动适配新主题 |
| 终端宽度不足 | Ink 自动处理文本换行 |
| 非交互式会话 | 渲染被跳过，仅返回文本内容 |

### 改进建议

**1. 增强结果消息**
```typescript
// 建议：显示更多上下文信息
export function renderToolResultMessage(
  output: Output,
  _progressMessagesForMessage: ProgressMessage<ToolProgressData>[],
  _options: { theme: ThemeName },
): React.ReactNode {
  return (
    <Box flexDirection="column" marginTop={1}>
      <Box flexDirection="row">
        <Text color={getModeColor('plan')}>{BLACK_CIRCLE}</Text>
        <Text> Entered plan mode</Text>
      </Box>
      <Box paddingLeft={2}>
        <Text dimColor>
          Claude is now exploring and designing an implementation approach.
        </Text>
      </Box>
      {/* 建议添加：下一步提示 */}
      <Box paddingLeft={2} marginTop={1}>
        <Text dimColor>
          Use ExitPlanMode when ready to present your plan.
        </Text>
      </Box>
    </Box>
  )
}
```

**2. 添加加载状态**
- 当前：进入计划模式是瞬时的
- 建议：如果未来有异步初始化，应添加加载指示器

**3. 错误状态显示**
- 当前：没有专门的错误渲染函数
- 建议：如果工具调用可能失败，应添加 `renderToolUseErrorMessage`

### 测试建议

```typescript
// 需要测试的场景：
1. 渲染结果消息 - 验证结构和颜色
2. 渲染拒绝消息 - 验证文本内容
3. 主题切换 - 验证颜色更新
4. 窄终端 - 验证布局不损坏
```

### 相关文件引用

```
src/tools/EnterPlanModeTool/
├── UI.tsx                  # 本文件 - UI 渲染
├── EnterPlanModeTool.ts    # 工具主逻辑，使用本模块的渲染函数
├── constants.ts            # 常量定义
└── prompt.ts               # 提示内容

src/constants/figures.js              # BLACK_CIRCLE 定义
src/utils/permissions/PermissionMode.js   # getModeColor 实现
src/ink.js                            # Ink 组件封装
```

### 与工具主逻辑的集成

在 `EnterPlanModeTool.ts` 中，渲染函数通过工具定义注册：

```typescript
export const EnterPlanModeTool: Tool<InputSchema, Output> = buildTool({
  // ...
  renderToolUseMessage,        // 来自 UI.tsx
  renderToolResultMessage,     // 来自 UI.tsx
  renderToolUseRejectedMessage, // 来自 UI.tsx
  // ...
})
```

这种分离设计允许：
1. UI 逻辑与业务逻辑解耦
2. 独立测试渲染组件
3. 更容易进行 UI 修改而不影响核心逻辑
