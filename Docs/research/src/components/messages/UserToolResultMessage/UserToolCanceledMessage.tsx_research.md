# UserToolCanceledMessage.tsx 深度研究文档

## 场景与职责

`UserToolCanceledMessage` 是 Claude Code 终端 UI 中用于渲染**工具执行被取消**状态的专用组件。当用户在工具执行过程中主动取消操作（如按 Ctrl+C 或点击取消按钮）时，该组件显示中断提示。

### 核心职责
1. **取消状态可视化**：清晰标识用户已中断工具执行
2. **引导用户下一步**：通过 `InterruptedByUser` 组件提示用户如何继续
3. **保持界面整洁**：单行高度，最小化视觉占用

## 功能点目的

### 1. 取消状态提示
- 使用 `InterruptedByUser` 组件显示中断提示
- 包含 "Interrupted" 文本和引导性问题 "What should Claude do instead?"

### 2. 与拒绝操作区分
- 取消是执行过程中的中断行为
- 拒绝是执行前的权限否决行为
- 两者通过不同组件路径区分

## 具体技术实现

### 组件接口
```typescript
export function UserToolCanceledMessage(): React.ReactNode
```

组件无 Props，为纯展示组件。

### 渲染结构
```tsx
<MessageResponse height={1}>
  <InterruptedByUser />
</MessageResponse>
```

### React Compiler 优化
- 使用 `_c(1)` 创建 1 个记忆化槽位
- 整个组件输出被完全缓存
- 使用 `Symbol.for("react.memo_cache_sentinel")` 标记常量缓存

## 关键代码路径与文件引用

### 直接依赖
| 文件路径 | 用途 |
|---------|------|
| `src/components/InterruptedByUser.js` | 中断提示组件 |
| `src/components/MessageResponse.js` | 消息响应容器 |

### 调用方
- `UserToolResultMessage.tsx`：当检测到 `CANCEL_MESSAGE` 前缀时渲染此组件

### 相关常量
```typescript
// src/utils/messages.ts
export const CANCEL_MESSAGE =
  "The user doesn't want to take this action right now. STOP what you are doing and wait for the user to tell you how to proceed."
```

### 依赖组件：InterruptedByUser
```tsx
// src/components/InterruptedByUser.tsx
export function InterruptedByUser(): React.ReactNode {
  return (
    <>
      <Text dimColor>Interrupted </Text>
      <Text dimColor>· What should Claude do instead?</Text>
    </>
  )
}
```

## 依赖与外部交互

### 上游数据流
1. **用户取消**：用户在工具执行过程中触发取消操作
2. **消息构造**：系统生成带 `CANCEL_MESSAGE` 前缀的取消消息
3. **条件渲染**：`UserToolResultMessage` 检测到前缀，渲染 `UserToolCanceledMessage`

### 触发场景
| 场景 | 取消机制 |
|-----|---------|
| 交互式取消 | 用户在权限提示界面点击 "Cancel" |
| 键盘中断 | 用户按 Ctrl+C 中断执行 |
| 超时取消 | 工具执行超时自动取消 |

## 风险、边界与改进建议

### 已知风险
1. **状态混淆**：用户可能混淆"取消"和"拒绝"的区别
2. **中断恢复**：取消后需要用户明确输入下一步指令，缺乏自动恢复机制

### 边界情况
| 场景 | 当前行为 | 评估 |
|-----|---------|------|
| 批量工具执行中取消 | 仅取消当前工具，后续工具继续 | 符合预期 |
| 嵌套工具取消 | 依赖外层工具的取消处理 | 需确保传播正确 |

### 改进建议
1. **添加上下文信息**：显示被取消的工具名称
   ```tsx
   <MessageResponse height={1}>
     <Text dimColor>{toolName} interrupted · </Text>
     <InterruptedByUser />
   </MessageResponse>
   ```

2. **快速操作提示**：添加常用操作的快捷方式提示（如 "/retry", "/skip"）

3. **取消原因收集**：在取消时提供简短的原因选项（如 "Too slow", "Wrong approach"）

4. **动画反馈**：添加短暂的视觉反馈，明确取消已生效

### 测试要点
- 验证取消消息正确显示
- 验证 `InterruptedByUser` 组件正确渲染
- 验证与拒绝消息的区分度
- 验证快速连续取消的处理
