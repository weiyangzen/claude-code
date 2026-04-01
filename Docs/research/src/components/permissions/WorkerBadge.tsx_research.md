# WorkerBadge.tsx 研究文档

## 场景与职责

`WorkerBadge.tsx` 是 Claude Code CLI 多智能体（Swarm）系统中的**工作者标识组件**。当团队成员（teammate/worker）发起权限请求时，该组件在权限对话框中显示一个彩色徽章，标识是哪个工作者正在请求权限。

该组件解决了多智能体协作场景中的**身份识别问题**：
- 当多个 AI 工作者同时运行时，用户需要知道是哪个具体工作者在请求权限
- 通过颜色编码和名称显示，提供直观的视觉区分
- 帮助用户建立对多智能体系统的信任感

## 功能点目的

### 1. 工作者身份可视化
- 显示工作者的名称（带 `@` 前缀）
- 使用颜色编码区分不同工作者
- 使用黑色圆点符号作为视觉标记

### 2. 主题感知颜色渲染
- 将工作者的颜色（通常是 AgentColorName）转换为 Ink 主题颜色
- 支持自定义颜色回退到 ANSI 颜色

### 3. 一致的视觉风格
- 所有权限请求中的工作者标识使用统一的视觉风格
- 与 `WorkerPendingPermission` 组件配合使用

## 具体技术实现

### 核心数据结构

```typescript
// 组件 Props
export type WorkerBadgeProps = {
  name: string;   // 工作者名称（如 "researcher", "coder"）
  color: string;  // 颜色标识（如 "blue", "green"）
};
```

### 渲染结构

```
Box (flexDirection="row", gap={1})
  └── Text (color={inkColor})
        ├── BLACK_CIRCLE  // ● 或 ⏺
        └── Text (bold)
              └── @{name}  // 如 @researcher
```

### 关键代码路径

```typescript
// 常量
import { BLACK_CIRCLE } from '../../constants/figures.js';  // ● 或 ⏺

// Ink 组件
import { Box, Text } from '../../ink.js';

// 颜色转换
import { toInkColor } from '../../utils/ink.js';

// 颜色转换逻辑（来自 utils/ink.ts）
const DEFAULT_AGENT_THEME_COLOR = 'cyan_FOR_SUBAGENTS_ONLY';

export function toInkColor(color: string | undefined): TextProps['color'] {
  if (!color) {
    return DEFAULT_AGENT_THEME_COLOR;
  }
  // 尝试映射到主题颜色
  const themeColor = AGENT_COLOR_TO_THEME_COLOR[color as AgentColorName];
  if (themeColor) {
    return themeColor;
  }
  // 回退到原始 ANSI 颜色
  return `ansi:${color}` as TextProps['color'];
}
```

### React Compiler 优化

组件使用 React Compiler 的缓存机制：
- 缓存 `inkColor` 计算结果
- 缓存名称文本元素
- 缓存最终渲染的 Box

## 依赖与外部交互

### 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `BLACK_CIRCLE` | `../../constants/figures.js` | 黑色圆点符号 |
| `Box`, `Text` | `../../ink.js` | Ink UI 组件 |
| `toInkColor` | `../../utils/ink.js` | 颜色转换函数 |

### 颜色映射

颜色映射定义在 `src/tools/AgentTool/agentColorManager.js`：

```typescript
export const AGENT_COLOR_TO_THEME_COLOR: Record<AgentColorName, keyof Theme> = {
  blue: 'blue',
  green: 'green', 
  yellow: 'yellow',
  cyan: 'cyan_FOR_SUBAGENTS_ONLY',
  magenta: 'magenta',
  red: 'red',
  // ...
};
```

### 被调用方

该组件被以下组件使用：

1. **PermissionRequestTitle.tsx** - 权限请求标题
   ```typescript
   // 在权限对话框标题中显示工作者徽章
   <PermissionRequestTitle 
     title={title}
     workerBadge={workerBadge}
     // ...
   />
   ```

2. **WorkerPendingPermission.tsx** - 工作者等待权限时显示的组件
   ```typescript
   // 在等待审批时显示工作者徽章
   <WorkerBadge name={agentName} color={agentColor} />
   ```

3. **各具体 PermissionRequest 组件** - 如 BashPermissionRequest、FileEditPermissionRequest 等

### 数据流

```
src/utils/teammate.ts (getAgentName, getTeammateColor)
  ↓
PermissionRequest.tsx (ToolUseConfirm.workerBadge)
  ↓
各具体 PermissionRequest 组件
  ↓
WorkerBadge (name, color)
  ↓
toInkColor() 转换
  ↓
渲染彩色徽章
```

## 风险、边界与改进建议

### 当前风险

1. **颜色回退机制**：
   - 未知颜色回退到 `ansi:${color}`，可能在某些终端上显示不正确
   - 没有验证 ANSI 颜色代码的有效性

2. **硬编码的默认颜色**：
   - `DEFAULT_AGENT_THEME_COLOR = 'cyan_FOR_SUBAGENTS_ONLY'`
   - 这个命名暗示了特定用途，但作为默认值可能不合适

3. **平台相关的符号**：
   - `BLACK_CIRCLE` 在 macOS 上使用 `⏺`，其他平台使用 `●`
   - 两种符号的视觉大小可能不一致

### 边界情况

1. **空名称**：如果 `name` 为空字符串，将显示 `@` 后面没有内容

2. **空颜色**：如果 `color` 为空，使用默认的 cyan 主题色

3. **特殊字符**：工作者名称中的特殊字符（如换行、控制字符）可能导致渲染问题

### 改进建议

1. **名称验证**：
   ```typescript
   // 添加名称清理
   const safeName = name.replace(/[\x00-\x1F\x7F]/g, '').trim();
   ```

2. **颜色验证**：
   ```typescript
   // 验证 ANSI 颜色代码
   function isValidAnsiColor(color: string): boolean {
     return /^[0-9]+$/.test(color) && parseInt(color) >= 0 && parseInt(color) <= 255;
   }
   ```

3. **国际化支持**：
   - 考虑 RTL（从右到左）语言的布局
   - `@` 符号在某些文化中可能有不同含义

4. **可访问性**：
   - 添加屏幕阅读器友好的标签
   - 确保颜色不是唯一的区分方式

5. **动画效果**：
   - 考虑添加微妙的脉冲动画表示"活动"状态
   - 帮助用户注意到新的权限请求

6. **工具提示**：
   - 添加悬停提示显示工作者的更多信息
   - 如工作者的角色、当前任务等

7. **尺寸变体**：
   - 支持不同尺寸的徽章（小、中、大）
   - 适应不同的 UI 上下文
