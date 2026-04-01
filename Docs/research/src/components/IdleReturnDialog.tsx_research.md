# IdleReturnDialog.tsx 深度研究文档

## 场景与职责

`IdleReturnDialog` 是 Claude Code 在用户长时间离开后返回时显示的对话框。该组件负责：

1. **空闲检测提示**：检测用户离开时间（idle time）和当前对话的 token 数量
2. **上下文管理建议**：建议用户是否继续当前对话或开启新对话
3. **用户偏好记忆**：支持 "Don't ask me again" 选项，记住用户选择
4. **Token 使用优化**：帮助用户管理上下文长度，减少不必要的 token 消耗

## 功能点目的

### 1. 空闲返回提示
- **目的**：在用户长时间离开后提供上下文管理建议
- **触发条件**：
  - 用户离开超过一定时间（由调用方控制）
  - 当前对话积累较多 tokens
- **显示信息**：
  - 离开时长（格式化显示："< 1m", "5m", "2h 30m" 等）
  - 当前对话 token 数量（格式化显示："1.2k", "500" 等）

### 2. 用户操作选项
- **目的**：让用户选择如何处理当前对话
- **选项**：
  1. **Continue this conversation** (`continue`) - 继续当前对话
  2. **Send message as a new conversation** (`clear`) - 清空上下文，作为新对话
  3. **Don't ask me again** (`never`) - 不再询问，记住此选择

### 3. 取消操作
- **目的**：允许用户不做选择直接关闭对话框
- **行为**：按 Esc 触发 `onCancel`，调用 `onDone("dismiss")`

## 具体技术实现

### 关键数据结构

```typescript
// 空闲返回操作类型
type IdleReturnAction = 'continue' | 'clear' | 'dismiss' | 'never';

// 组件 Props
type Props = {
  idleMinutes: number;        // 离开时间（分钟）
  totalInputTokens: number;   // 当前对话输入 token 数量
  onDone: (action: IdleReturnAction) => void;  // 完成回调
};
```

### 关键流程

1. **数据格式化流程**：
   ```
   1. 接收 idleMinutes 和 totalInputTokens
   2. 调用 formatIdleDuration(idleMinutes) 格式化时间
   3. 调用 formatTokens(totalInputTokens) 格式化 token 数
   4. 组合成标题字符串显示
   ```

2. **用户选择流程**：
   ```
   1. 渲染 Select 组件，显示三个选项
   2. 用户选择后调用 onChange → onDone(value)
   3. 用户按 Esc 调用 onCancel → onDone("dismiss")
   4. 父组件根据 action 执行相应操作
   ```

### 时间格式化逻辑

```typescript
function formatIdleDuration(minutes: number): string {
  if (minutes < 1) {
    return '< 1m';  // 小于1分钟
  }
  if (minutes < 60) {
    return `${Math.floor(minutes)}m`;  // 不足1小时，显示分钟
  }
  const hours = Math.floor(minutes / 60);
  const remainingMinutes = Math.floor(minutes % 60);
  if (remainingMinutes === 0) {
    return `${hours}h`;  // 整小时
  }
  return `${hours}h ${remainingMinutes}m`;  // 小时+分钟
}
```

### Token 格式化

使用 `src/utils/format.ts` 中的 `formatTokens` 函数：

```typescript
export function formatTokens(count: number): string {
  return formatNumber(count).replace('.0', '');
}
// 示例：1200 → "1.2k", 500 → "500"
```

## 关键代码路径与文件引用

### 本文件关键代码

```typescript
export function IdleReturnDialog({
  idleMinutes,
  totalInputTokens,
  onDone,
}: Props): React.ReactNode {
  const formattedIdle = formatIdleDuration(idleMinutes);
  const formattedTokens = formatTokens(totalInputTokens);

  return (
    <Dialog
      title={`You've been away ${formattedIdle} and this conversation is ${formattedTokens} tokens.`}
      onCancel={() => onDone('dismiss')}
    >
      <Box flexDirection="column">
        <Text>
          If this is a new task, clearing context will save usage and be faster.
        </Text>
      </Box>
      <Select
        options={[
          { value: 'continue', label: 'Continue this conversation' },
          { value: 'clear', label: 'Send message as a new conversation' },
          { value: 'never', label: "Don't ask me again" },
        ]}
        onChange={value => onDone(value)}
      />
    </Dialog>
  );
}
```

### 依赖文件

| 文件路径 | 用途 |
|---------|------|
| `src/utils/format.ts` | Token 格式化函数 `formatTokens` |
| `src/components/CustomSelect/index.ts` | Select 选择组件 |
| `src/components/design-system/Dialog.tsx` | 对话框 UI 组件 |
| `src/ink.tsx` | Ink 渲染组件 (`Box`, `Text`) |

### 调用方

- 主应用逻辑中检测空闲时间后调用
- 通常在 `src/components/App.tsx` 或相关状态管理文件中
- 触发条件：用户返回且满足空闲时间和 token 阈值

## 依赖与外部交互

### 外部依赖

1. **React Compiler Runtime**：使用 `_c` 函数进行编译时优化
2. **Ink**：终端 UI 渲染库

### 内部服务交互

1. **格式化服务**：
   - 依赖 `formatTokens` 将大数字转换为易读格式
   - 内部使用 `Intl.NumberFormat` 进行本地化格式化

2. **选择组件**：
   - 使用 `Select` 组件提供交互式选项
   - 支持键盘导航和选择

3. **对话框系统**：
   - 使用 `Dialog` 组件提供标准对话框外观
   - 支持取消操作和标题显示

### 配置持久化

"Don't ask me again" 选项的状态持久化由调用方处理：

```typescript
// 建议的调用方实现
function handleIdleReturn(action: IdleReturnAction) {
  if (action === 'never') {
    saveGlobalConfig(config => ({
      ...config,
      idleReturnDismissed: true
    }));
  } else if (action === 'clear') {
    clearConversationContext();
  }
  // continue 和 dismiss 不需要特殊处理
}
```

## 风险、边界与改进建议

### 潜在风险

1. **时间阈值敏感**：
   - 风险：空闲时间阈值设置不当可能频繁打扰用户或从不触发
   - 建议：将阈值配置化，允许用户自定义

2. **Token 计数不准确**：
   - 风险：`totalInputTokens` 可能不包含系统提示或其他隐藏 tokens
   - 建议：显示更准确的上下文长度估计

3. **"Never" 选项不可逆**：
   - 风险：用户选择 "Don't ask me again" 后无法轻松恢复
   - 建议：在设置中添加重新启用选项

### 边界情况

1. **极小时间值**：
   - `idleMinutes < 1` 显示 "< 1m"
   - 如果传入负数，行为未定义

2. **极大 Token 数**：
   - `formatTokens` 使用紧凑记数法，"1.2M" 可能不够精确
   - 建议：添加精确数字提示

3. **零 Token**：
   - 如果 `totalInputTokens === 0`，显示 "0 tokens"
   - 这在实际中不太可能，但处理正确

### 改进建议

1. **添加上下文预览**：
   ```typescript
   // 建议：显示对话摘要或最后几条消息
   <Box>
     <Text dimColor>Last message: {lastMessagePreview}</Text>
   </Box>
   ```

2. **智能建议**：
   ```typescript
   // 建议：根据任务类型给出建议
   const suggestion = detectTaskChange() 
     ? "This looks like a new task. Consider clearing context."
     : "You were in the middle of a task. Continue?";
   ```

3. **配置选项**：
   ```typescript
   // 建议：可配置的阈值
   interface IdleReturnConfig {
     minIdleMinutes: number;      // 默认 30
     minTokensThreshold: number;  // 默认 1000
     showDialog: boolean;         // 是否显示对话框
   }
   ```

4. **批量操作支持**：
   - 如果用户有多个待处理对话，提供批量管理选项

5. **历史记录**：
   - 记录用户的选择模式，用于优化建议
   - 例如：如果用户经常在 30 分钟后选择 "clear"，自动建议

### 相关配置项

```typescript
// GlobalConfig 中的相关字段（建议添加）
interface GlobalConfig {
  idleReturnDismissed?: boolean;  // 用户选择不再询问
  idleReturnThresholdMinutes?: number;  // 空闲时间阈值
  idleReturnTokenThreshold?: number;    // Token 数量阈值
}
```

### 测试建议

1. **单元测试**：
   - 测试 `formatIdleDuration` 的各种输入
   - 测试不同 action 的回调触发

2. **集成测试**：
   - 测试与 Select 组件的交互
   - 测试对话框的显示/隐藏逻辑

3. **边界测试**：
   - 测试极端值（0, 负数, 极大值）
   - 测试快速连续触发的情况
