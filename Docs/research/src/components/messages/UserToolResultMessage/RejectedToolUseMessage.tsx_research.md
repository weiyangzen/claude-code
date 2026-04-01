# RejectedToolUseMessage.tsx 深度研究文档

## 场景与职责

`RejectedToolUseMessage` 是 Claude Code 终端 UI 中用于渲染**通用工具使用被拒绝**状态的最简组件。当用户拒绝某个工具调用（非计划模式场景）时，该组件显示简洁的拒绝提示。

### 核心职责
1. **工具拒绝状态提示**：显示 "Tool use rejected" 文本
2. **最小化视觉干扰**：使用单行高度、暗淡颜色，避免过度吸引注意力
3. **一致性体验**：与 `UserToolCanceledMessage` 等组件保持视觉风格一致

## 功能点目的

### 1. 简洁拒绝提示
- 单行文本显示 "Tool use rejected"
- 使用 `dimColor` 样式降低视觉优先级
- 固定高度为 1，保持消息列表紧凑

### 2. 与取消操作区分
- 虽然视觉上与取消操作类似，但语义上明确标识为"拒绝"而非"取消"
- 通过不同的组件路径支持未来可能的差异化处理

## 具体技术实现

### 组件接口
```typescript
export function RejectedToolUseMessage(): React.ReactNode
```

组件无 Props，为纯展示组件。

### 渲染结构
```tsx
<MessageResponse height={1}>
  <Text dimColor>Tool use rejected</Text>
</MessageResponse>
```

### React Compiler 优化
- 使用 `_c(1)` 创建 1 个记忆化槽位
- 整个组件输出被完全缓存（无外部依赖）
- 使用 `Symbol.for("react.memo_cache_sentinel")` 标记常量缓存

## 关键代码路径与文件引用

### 直接依赖
| 文件路径 | 用途 |
|---------|------|
| `src/ink.js` | Ink 终端 UI 组件（Text） |
| `src/components/MessageResponse.js` | 消息响应容器 |

### 调用方
- `UserToolErrorMessage.tsx`：当检测到 `REJECT_MESSAGE_WITH_REASON_PREFIX` 前缀时渲染此组件

### 相关常量
```typescript
// src/utils/messages.ts
export const REJECT_MESSAGE_WITH_REASON_PREFIX =
  "The user doesn't want to proceed with this tool use. The tool use was rejected (eg. if it was a file edit, the new_string was NOT written to the file). To tell you how to proceed, the user said:\n"
```

### 对比组件
| 组件 | 用途 | 视觉差异 |
|-----|------|---------|
| `RejectedToolUseMessage` | 通用工具拒绝 | 单行，"Tool use rejected" |
| `UserToolCanceledMessage` | 工具取消 | 显示 `InterruptedByUser` 组件 |
| `FallbackToolUseRejectedMessage` | 回退拒绝 UI | 显示 "Interrupted" 提示 |

## 依赖与外部交互

### 上游数据流
1. **用户拒绝**：用户在权限提示界面选择拒绝工具调用
2. **消息构造**：系统生成带 `REJECT_MESSAGE_WITH_REASON_PREFIX` 前缀的拒绝消息
3. **条件渲染**：`UserToolErrorMessage` 检测到前缀，渲染 `RejectedToolUseMessage`

### 主题系统
- 使用 Ink 的 `dimColor` 属性
- 继承终端主题中的暗淡颜色配置

## 风险、边界与改进建议

### 已知风险
1. **信息过于简略**：仅显示 "Tool use rejected"，用户无法从 UI 中获知具体是哪个工具被拒绝
2. **与取消操作混淆**：视觉上与取消操作非常相似，可能导致用户混淆

### 边界情况
| 场景 | 当前行为 | 评估 |
|-----|---------|------|
| 批量工具拒绝 | 每个被拒绝的工具都渲染此组件 | 可能导致重复消息，但符合预期 |
| 快速连续拒绝 | 每个拒绝独立渲染 | 消息列表会增长，符合预期 |

### 改进建议
1. **添加工具名称**：考虑在消息中显示被拒绝的工具名称
   ```tsx
   <Text dimColor>{toolName} tool use rejected</Text>
   ```

2. **差异化视觉**：与取消操作使用不同的颜色或图标，增强区分度

3. **国际化**：当前文本硬编码，建议支持 i18n

4. **可点击反馈**：考虑添加反馈机制（如 `/feedback` 快捷方式）

### 测试要点
- 验证文本正确显示
- 验证 `dimColor` 样式应用
- 验证高度固定为 1
- 验证在消息列表中的视觉一致性
