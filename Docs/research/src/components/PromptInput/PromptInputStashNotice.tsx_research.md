# PromptInputStashNotice.tsx 深度研究文档

> **研究对象**: `src/components/PromptInput/PromptInputStashNotice.tsx`  
> **研究范围**: 源码、调用方、stash 功能机制及相关依赖  
> **执行器**: kimi (k2p5)  
> **研究日期**: 2026-04-01

---

## 1. 场景与职责

### 1.1 核心定位

`PromptInputStashNotice.tsx` 是 Claude Code CLI 中** stash 功能的视觉反馈组件**。当用户通过 `ctrl+s` 将当前输入内容暂存（stash）后，该组件在输入区域显示一条提示，告知用户存在已暂存的内容，且该内容会在提交后自动恢复。

### 1.2 应用场景

| 场景 | 说明 |
|------|------|
| **用户暂存输入后** | 用户按下 `ctrl+s`，当前输入被保存到 stash，组件显示提示 |
| **提交后自动恢复** | 用户发送一条新消息后，之前 stash 的内容自动回填到输入框 |
| **清空输入时隐藏** | 当 stash 被消费或清除后，`hasStash` 变为 false，提示消失 |

### 1.3 职责边界

- **纯展示组件**：只根据 `hasStash` boolean 决定是否渲染提示文本
- **无状态管理**：不管理 stash 内容，只接收 props 进行条件渲染
- **最小侵入**：组件体积极小，对整体布局影响微乎其微

---

## 2. 功能点目的

### 2.1 Stash 状态可视化

让用户明确知道：
1. 当前有输入内容被暂存
2. 暂存的内容会在下次提交后自动恢复

这避免了用户误以为输入丢失，或反复尝试重新输入相同内容。

### 2.2 与快捷键系统呼应

提示文本中的 "Stashed" 概念与快捷键 `ctrl+s`（`chat:stash` 动作）形成对应，强化用户心智模型。

---

## 3. 具体技术实现

### 3.1 组件 Props

```typescript
type Props = {
  hasStash: boolean;
};
```

### 3.2 渲染逻辑

```typescript
export function PromptInputStashNotice({ hasStash }: Props): React.ReactNode {
  if (!hasStash) {
    return null;
  }

  return (
    <Box paddingLeft={2}>
      <Text dimColor>
        {figures.pointerSmall} Stashed (auto-restores after submit)
      </Text>
    </Box>
  );
}
```

### 3.3 视觉设计

- **位置**：左对齐，`paddingLeft={2}`
- **颜色**：`dimColor` — 使用暗淡颜色，避免抢夺输入框的视觉焦点
- **图标**：`figures.pointerSmall` — 一个小箭头符号（`›` 或类似字符，取决于终端和 `figures` 包）

### 3.4 React Compiler 编译特征

源码经过 React Compiler 编译：
- 使用 `_c(1)` 进行 memoization
- `Symbol.for("react.memo_cache_sentinel")` 用于检测首次渲染
- 编译后的代码将 JSX 结果缓存到 `$[0]` 中，后续渲染直接复用

---

## 4. 关键代码路径与文件引用

### 4.1 组件入口

| 路径 | 说明 |
|------|------|
| `src/components/PromptInput/PromptInputStashNotice.tsx` | 主组件文件（编译后输出，仅 25 行） |
| `src/components/PromptInput/PromptInput.tsx` | 直接调用方，传入 `hasStash={!!stashedPrompt}` |

### 4.2 核心依赖

| 路径 | 说明 |
|------|------|
| `figures` | npm 包，提供跨平台终端符号（`pointerSmall`） |
| `src/ink.js` | `Box`、`Text` 组件 |

### 4.3 Stash 功能链路

```
用户按下 ctrl+s
  ↓
PromptInput.tsx 中的 keybinding 处理
  ↓
setStashedPrompt({ text, cursorOffset, pastedContents })
  ↓
stashedPrompt 状态更新
  ↓
PromptInputStashNotice hasStash={!!stashedPrompt}
  ↓
渲染 "› Stashed (auto-restores after submit)"
```

自动恢复链路：
```
用户提交消息
  ↓
onSubmit 处理完成
  ↓
PromptInput.tsx 中的 useEffect 或提交后逻辑
  ↓
将 stashedPrompt.text 恢复到输入框
  ↓
setStashedPrompt(undefined)
  ↓
PromptInputStashNotice 隐藏
```

---

## 5. 依赖与外部交互

### 5.1 调用方数据准备

在 `PromptInput.tsx` 中：

```typescript
<PromptInputStashNotice hasStash={!!stashedPrompt} />
```

`stashedPrompt` 的类型为：
```typescript
stashedPrompt: {
  text: string;
  cursorOffset: number;
  pastedContents: Record<number, PastedContent>;
} | undefined
```

### 5.2 与输入缓冲系统的交互

Stash 功能与 `useInputBuffer` 提供的撤销（undo）功能是正交的：
- **Undo**（`ctrl+_`）：恢复输入框的历史编辑状态
- **Stash**（`ctrl+s`）：将当前输入保存到一旁，清空输入框，提交后自动恢复

### 5.3 Stash Hint 通知

在 `PromptInput.tsx` 中还有一套独立的 "stash hint" 通知系统：

```typescript
// 当用户逐渐清空大量输入时，提示可以使用 stash
if (clearedSubstantialInput && !wasRapidClear) {
  const config = getGlobalConfig();
  if (!config.hasUsedStash) {
    addNotification({
      key: 'stash-hint',
      jsx: <Text dimColor>
          Tip:{' '}
          <ConfigurableShortcutHint action="chat:stash" context="Chat" fallback="ctrl+s" description="stash" />
        </Text>,
      priority: 'immediate',
      timeoutMs: FOOTER_TEMPORARY_STATUS_TIMEOUT
    });
  }
}
```

`PromptInputStashNotice` 与 stash hint 通知是两个独立的功能：
- **Stash Notice**：用户已经使用了 stash，显示状态提示
- **Stash Hint**：用户还没用过 stash，在用户行为符合模式时进行教育引导

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 说明 |
|------|------|
| **功能过于简单，几乎无风险** | 组件逻辑极简，条件渲染 + 静态 JSX，出错概率极低 |
| **编译后代码包含 source map** | inline base64 source map 占用了大部分文件体积（25 行中约 20 行是 source map） |
| **无测试覆盖** | 虽然逻辑简单，但项目中未找到针对该组件的测试 |

### 6.2 边界情况

- **`hasStash` 从 `true` 变为 `false`**：组件返回 `null`，React 卸载该节点
- **`hasStash` 保持 `true`**：由于 React Compiler 缓存，不会创建新的 JSX 节点
- **终端宽度极窄**：文本较长，但 `Text` 组件默认会换行或截断，不会破坏布局
- **多行 stash 内容**：提示文本只显示 "Stashed"，不显示 stash 内容的具体长度或摘要

### 6.3 改进建议

1. **显示 stash 内容摘要**：在提示中增加 stash 内容的前几个字符（如 `Stashed: "fix bug in..." (auto-restores after submit)`），帮助用户回忆暂存了什么
2. **增加快捷键提示**：在提示中显示恢复/取消 stash 的快捷键（如果有的话），提升可发现性
3. **考虑动画过渡**：为提示的显示/隐藏添加简单的淡入淡出动画，提升视觉体验（Ink 支持有限，需谨慎）
4. **剥离生产 source map**：编译产物中的 inline source map 可以配置为生产环境剥离，减小包体积
5. **增加简单快照测试**：即使逻辑简单，一个快照测试也能防止未来意外修改提示文本
