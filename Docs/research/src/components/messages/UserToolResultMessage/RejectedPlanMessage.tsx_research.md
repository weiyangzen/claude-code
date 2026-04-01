# RejectedPlanMessage.tsx 深度研究文档

## 场景与职责

`RejectedPlanMessage` 是 Claude Code 终端 UI 中用于渲染**用户拒绝 Plan Mode 计划**的专用组件。当用户在计划模式（Plan Mode）下查看 Claude 提出的执行计划后选择拒绝时，该组件负责以结构化的方式展示被拒绝的计划内容。

### 核心职责
1. **计划拒绝可视化**：清晰标识用户已拒绝计划的状态
2. **计划内容展示**：使用 Markdown 渲染被拒绝的完整计划文本
3. **视觉区分**：通过特殊的边框颜色（`planMode`）和样式与常规消息区分

## 功能点目的

### 1. 计划拒绝状态提示
- 显示 "User rejected Claude's plan:" 提示文本
- 使用 `subtle` 颜色降低视觉优先级，避免过度强调负面操作

### 2. 计划内容渲染
- 使用 `Markdown` 组件渲染计划内容，支持格式化文本
- 将计划内容包裹在带圆角边框的 `Box` 容器中
- 边框使用 `planMode` 主题色，与计划模式视觉体系保持一致

### 3. Windows 终端兼容性
- 设置 `overflow="hidden"` 确保在 Windows Terminal 上正确渲染

## 具体技术实现

### 组件接口
```typescript
type Props = {
  plan: string;  // 被拒绝的计划内容（Markdown 格式）
};

export function RejectedPlanMessage({ plan }: Props): React.ReactNode
```

### 渲染结构
```tsx
<MessageResponse>
  <Box flexDirection="column">
    <Text color="subtle">User rejected Claude's plan:</Text>
    <Box 
      borderStyle="round" 
      borderColor="planMode" 
      paddingX={1} 
      overflow="hidden"
    >
      <Markdown>{plan}</Markdown>
    </Box>
  </Box>
</MessageResponse>
```

### React Compiler 优化
组件使用 React Compiler（`react/compiler-runtime`）进行自动记忆化：
- 使用 `_c(3)` 创建 3 个记忆化槽位
- 静态文本节点（`Text` 组件）使用 `Symbol.for("react.memo_cache_sentinel")` 进行常量缓存
- 动态内容（`plan`）变化时重新渲染 `MessageResponse` 包裹的完整结构

## 关键代码路径与文件引用

### 直接依赖
| 文件路径 | 用途 |
|---------|------|
| `src/components/Markdown.js` | Markdown 内容渲染 |
| `src/components/MessageResponse.js` | 消息响应容器（提供 ⎿ 前缀） |
| `src/ink.js` | Ink 终端 UI 组件（Box, Text） |

### 调用方
- `UserToolErrorMessage.tsx`：当检测到 `PLAN_REJECTION_PREFIX` 前缀时渲染此组件

### 相关常量
```typescript
// src/utils/messages.ts
export const PLAN_REJECTION_PREFIX =
  'The agent proposed a plan that was rejected by the user. The user chose to stay in plan mode rather than proceed with implementation.\n\nRejected plan:\n'
```

## 依赖与外部交互

### 上游数据流
1. **计划拒绝生成**：用户在计划模式 UI 中点击拒绝
2. **消息构造**：系统生成带 `PLAN_REJECTION_PREFIX` 前缀的拒绝消息
3. **条件渲染**：`UserToolErrorMessage` 检测到前缀，提取计划内容并渲染 `RejectedPlanMessage`

### 主题系统
- 依赖 Ink 的主题系统
- 使用 `planMode` 作为边框颜色标识
- 使用 `subtle` 作为提示文本颜色

## 风险、边界与改进建议

### 已知风险
1. **空计划内容**：如果 `plan` 参数为空字符串，组件仍会渲染边框容器，可能导致视觉上的空框
2. **Markdown 注入**：计划内容直接传递给 Markdown 组件，需确保上游已对内容进行安全处理

### 边界情况
| 场景 | 当前行为 | 建议 |
|-----|---------|------|
| plan = "" | 渲染空边框容器 | 添加空内容检查，提前返回 null |
| plan 包含特殊字符 | Markdown 组件负责转义 | 确保 Markdown 组件有 XSS 防护 |
| 超长计划内容 | 容器支持滚动（overflow=hidden） | 考虑添加最大高度限制 |

### 改进建议
1. **空状态处理**：添加对空计划内容的早期返回
   ```tsx
   if (!plan?.trim()) return null;
   ```

2. **内容截断**：对于极长的计划，考虑添加折叠/展开功能

3. **国际化**：当前文本硬编码为英文，建议支持 i18n

4. **可访问性**：考虑添加屏幕阅读器友好的 aria 标签

### 测试要点
- 验证计划内容正确渲染为 Markdown
- 验证 Windows Terminal 兼容性（overflow 设置）
- 验证空计划内容的处理
- 验证主题颜色正确应用
