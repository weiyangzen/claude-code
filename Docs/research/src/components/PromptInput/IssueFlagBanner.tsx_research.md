# IssueFlagBanner.tsx 研究文档

## 场景与职责

`IssueFlagBanner.tsx` 是 Claude Code CLI 中用于**内部反馈收集**的组件，仅在 Anthropic 内部（`ant`）构建中启用。该组件在检测到用户对话中存在"摩擦信号"（friction）时，在转录区域显示一个提示横幅，鼓励用户通过 `/issue` 命令报告问题。

### 使用场景
- **内部测试**：Anthropic 员工使用内部构建版本时
- **摩擦检测**：当用户输入表现出对 AI 响应的不满或纠正意图时
- **问题收集**：引导用户主动报告遇到的问题

## 功能点目的

### 1. 摩擦检测提示
- 在对话中检测到用户可能遇到问题时显示提示
- 通过 `/issue` 命令引导用户报告问题

### 2. 构建类型隔离
- 仅在内部分支（`"external" !== 'ant'`）显示实际 UI
- 外部构建直接返回 `null`，完全禁用该功能

### 3. 视觉提示
- 使用旗帜图标（`FLAG_ICON`）吸引注意力
- 使用警告色（`warning`）突出显示重要性

## 具体技术实现

### 组件结构

```typescript
// 当前实现（已简化）
export function IssueFlagBanner() {
  return null;
}
```

### 原始设计（从 source map 恢复）

```tsx
export function IssueFlagBanner(): React.ReactNode {
  // 构建时条件判断，外部构建直接返回 null
  if ("external" !== 'ant') {
    return null
  }

  return (
    <Box flexDirection="row" marginTop={1} width="100%">
      <Box minWidth={2}>
        <Text color="warning">{FLAG_ICON}</Text>
      </Box>
      <Text>
        <Text dimColor>[ANT-ONLY] </Text>
        <Text color="warning" bold>
          Something off with Claude?
        </Text>
        <Text dimColor> /issue to report it</Text>
      </Text>
    </Box>
  )
}
```

### 关键流程

1. **构建时条件编译**：
   - 使用 `"external" !== 'ant'` 进行构建时判断
   - 该条件在构建阶段被评估，外部构建会完全消除相关代码

2. **摩擦检测逻辑**（在调用方 hook 中）：
   - 位于 `src/hooks/useIssueFlagBanner.ts`
   - 通过正则表达式检测用户输入中的摩擦信号
   - 考虑提交次数、冷却时间等因素决定是否显示横幅

## 依赖与外部交互

### 直接依赖

| 模块 | 路径 | 用途 |
|------|------|------|
| React | `'react'` | 组件基础 |
| FLAG_ICON | `../../constants/figures.js` | 旗帜图标 |
| Box, Text | `../../ink.js` | UI 布局组件 |

### 调用方

- **REPL.tsx** (`src/screens/REPL.tsx`)
  - 导入并渲染 IssueFlagBanner
  - 通过 `useIssueFlagBanner` hook 控制显示逻辑

### 关联 Hook

- **useIssueFlagBanner.ts** (`src/hooks/useIssueFlagBanner.ts`)
  - 实现摩擦检测逻辑
  - 管理显示状态（冷却时间、提交计数等）

### 摩擦检测模式

Hook 中定义的正则表达式模式：

```typescript
const FRICTION_PATTERNS = [
  // "No," or "No!" at start
  /^no[,!]\s/i,
  // Direct corrections
  /\bthat'?s (wrong|incorrect|not (what|right|correct))\b/i,
  /\bnot what I (asked|wanted|meant|said)\b/i,
  // Referencing prior instructions
  /\bI (said|asked|wanted|told you|already said)\b/i,
  // Questioning actions
  /\bwhy did you\b/i,
  /\byou should(n'?t| not)? have\b/i,
  /\byou were supposed to\b/i,
  // Explicit retry/revert
  /\btry again\b/i,
  /\b(undo|revert) (that|this|it|what you)\b/i,
];
```

## 风险、边界与改进建议

### 潜在风险

1. **功能被完全禁用**
   - 当前实现直接返回 `null`，功能实际上被禁用
   - 可能是临时措施或功能迁移
   - 建议：确认该功能是否仍需维护

2. **硬编码构建类型判断**
   - `"external" !== 'ant'` 是硬编码的字符串比较
   - 如果构建配置变更，可能导致功能意外启用或禁用

3. **source map 与源码不一致**
   - source map 显示完整实现，但实际源码只有 `return null`
   - 可能导致调试困难

### 边界情况

1. **外部构建**
   - 外部构建中该组件始终返回 `null`
   - 不会渲染任何内容，无性能开销

2. **多次触发**
   - Hook 中有冷却时间（`COOLDOWN_MS = 30 * 60 * 1000`，30分钟）
   - 最小提交次数要求（`MIN_SUBMIT_COUNT = 3`）

3. **会话兼容性检查**
   - Hook 中还检查会话是否兼容容器化
   - 某些外部命令（curl、ssh、kubectl 等）会禁用该功能

### 改进建议

1. **恢复或移除功能**
   - 如果不再需要，建议完全移除相关代码
   - 如果需要保留，恢复原始实现并添加功能开关

2. **配置化构建类型**
   - 使用环境变量或构建配置替代硬编码字符串
   - 示例：`process.env.BUILD_TARGET === 'ant'`

3. **增强摩擦检测**
   - 考虑使用更智能的 NLP 模型替代正则表达式
   - 添加用户反馈循环，持续改进检测准确率

4. **国际化支持**
   - 当前提示文本为英文
   - 考虑添加多语言支持

5. **更好的视觉设计**
   - 当前设计较为简单
   - 可考虑添加交互元素（如直接点击报告）

### 相关文件引用

- 实现文件：`src/components/PromptInput/IssueFlagBanner.tsx`
- 调用方：`src/screens/REPL.tsx`
- 检测逻辑：`src/hooks/useIssueFlagBanner.ts`
- 图标常量：`src/constants/figures.js`
