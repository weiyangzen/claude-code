# promptOverlayContext.tsx 研究文档

## 场景与职责

`promptOverlayContext.tsx` 是 Claude Code 全屏布局下的一个**渲染门户（Portal）系统**，专门用于将需要“悬浮在输入框上方”的内容逃逸出 `FullscreenLayout` 底部槽位的 `overflowY: hidden` 裁剪区域。

在 `FullscreenLayout` 中，底部区域（包含 prompt、spinner、权限请求等）设置了 `overflowY: hidden`，这是为了防止过高的粘贴内容挤压上方的 `ScrollBox`（参见注释中的 CC-668）。然而，slash-command 建议列表和临时对话框（如 `AutoModeOptInDialog`）需要向上浮动显示，若放在底部槽位内部会被裁剪到仅剩约 1 行。该 Context 通过将内容提升到 `FullscreenLayout` 的同级绝对定位容器中，解决了这一矛盾。

## 功能点目的

### 1. 双通道设计
Context 被拆分为 **data + setter** 两对，避免写入者因自己的写入而重渲染：

| 通道 | 用途 | 写入方 | 读取方 |
|------|------|--------|--------|
| `DataContext` / `SetContext` | 结构化建议数据 | `PromptInputFooter` | `FullscreenLayout` 内的 `SuggestionsOverlay` |
| `DialogContext` / `SetDialogContext` | 任意 React 节点（对话框） | `PromptInput` | `FullscreenLayout` 内的 `DialogOverlay` |

### 2. `useSetPromptOverlay(data)`
- 供 `PromptInputFooter.tsx` 在 slash-command 建议激活时调用。
- 通过 `useEffect` 将 `data` 写入 `SetContext`；组件卸载时自动清空（`set(null)`）。
- 若不在 Provider 内（非全屏模式），静默无操作（no-op）。

### 3. `useSetPromptOverlayDialog(node)`
- 供 `PromptInput.tsx` 在需要显示浮动对话框（如 `AutoModeOptInDialog`）时调用。
- 同样通过 `useEffect` 注册，卸载自动清理。

### 4. `usePromptOverlay()` / `usePromptOverlayDialog()`
- 供 `FullscreenLayout.tsx` 读取当前悬浮内容和对话框节点。

## 具体技术实现

### Context 层级结构
```tsx
<SetContext.Provider value={setData}>
  <SetDialogContext.Provider value={setDialog}>
    <DataContext.Provider value={data}>
      <DialogContext.Provider value={dialog}>
        {children}
      </DialogContext.Provider>
    </DataContext.Provider>
  </SetDialogContext.Provider>
</SetContext.Provider>
```

- **setter contexts 在外层**：写入组件只消费 `SetContext` / `SetDialogContext`，这些 value 是稳定的函数引用，因此写入操作不会触发写入组件自身的重渲染。
- **data contexts 在内层**：读取组件（`SuggestionsOverlay`、`DialogOverlay`）消费 `DataContext` / `DialogContext`，数据变化时只触发读取方和 `FullscreenLayout` 的重渲染。

### 关键流程
1. **FullscreenLayout 挂载 Provider**：
   ```tsx
   // FullscreenLayout.tsx
   return (
     <PromptOverlayProvider>
       {scrollableArea}
       {bottomSlot}
       {modalSlot}
     </PromptOverlayProvider>
   )
   ```

2. **PromptInputFooter 写入建议**：
   ```tsx
   // PromptInputFooter.tsx
   const suggestionData = useMemo(() => ({
     suggestions,
     selectedSuggestion,
     maxColumnWidth
   }), [suggestions, selectedSuggestion, maxColumnWidth])
   useSetPromptOverlay(suggestionData)
   ```

3. **FullscreenLayout 读取并渲染**：
   ```tsx
   // FullscreenLayout.tsx 内部
   function SuggestionsOverlay() {
     const data = usePromptOverlay()
     if (!data || data.suggestions.length === 0) return null
     return (
       <Box position="absolute" bottom="100%" left={0} right={0} paddingX={2} paddingTop={1} ...>
         <PromptInputFooterSuggestions ... overlay={true} />
       </Box>
     )
   }
   ```
   `bottom="100%"` 使建议列表紧贴底部槽位的上边缘，实现“向上浮动”。

4. **DialogOverlay 同理**：
   ```tsx
   function DialogOverlay() {
     const node = usePromptOverlayDialog()
     if (!node) return null
     return (
       <Box position="absolute" bottom="100%" left={0} right={0} ...>
         {node}
       </Box>
     )
   }
   ```

### 数据类型
```ts
export type PromptOverlayData = {
  suggestions: SuggestionItem[]
  selectedSuggestion: number
  maxColumnWidth?: number
}
```

## 关键代码路径与文件引用

| 文件 | 角色 |
|------|------|
| `src/context/promptOverlayContext.tsx` | 本文件，定义 Portal Context 和读写 Hook |
| `src/components/FullscreenLayout.tsx` | Provider 挂载方 + 读取渲染方（`SuggestionsOverlay`、`DialogOverlay`） |
| `src/components/PromptInput/PromptInputFooter.tsx` | 建议数据写入方（`useSetPromptOverlay`） |
| `src/components/PromptInput/PromptInput.tsx` | 对话框节点写入方（`useSetPromptOverlayDialog`） |
| `src/components/PromptInput/PromptInputFooterSuggestions.tsx` | 建议列表的纯展示组件，接收 `overlay` prop 调整布局 |

## 依赖与外部交互

- **React**：`createContext`、`useContext`、`useState`、`useEffect`。
- **`../components/PromptInput/PromptInputFooterSuggestions.js`**：类型依赖 `SuggestionItem`。
- **`../ink.js` (`Box`)**：`FullscreenLayout` 中使用 `Box` 做绝对定位容器。
- **无 AppState 依赖**：完全独立于全局状态管理。

## 风险、边界与改进建议

### 风险与边界
1. **Context 嵌套深度**：四个 Context 层层嵌套，虽然逻辑清晰，但增加了 React DevTools 中的上下文层级噪音，且每个 Provider 都会创建额外的虚拟 DOM 节点。
2. **`useSetPromptOverlay` 的静默 no-op**：当不在 `PromptOverlayProvider` 内时，Hook 直接返回不做任何事。这在非全屏模式下是预期行为，但如果开发者错误地在全屏组件树之外调用，可能导致悬浮内容不显示且无任何警告。
3. **`SuggestionsOverlay` 与 `DialogOverlay` 的 z-order**：两者都是 `position="absolute" bottom="100%"`，渲染顺序由 `FullscreenLayout` 中的 JSX 顺序决定（`SuggestionsOverlay` 先，`DialogOverlay` 后）。注释说明“Dialog 渲染在树顺序后面，因此如果两者同时出现，Dialog 会覆盖 Suggestions”，但两者同时出现的场景理论上不应发生，缺乏运行时保护。
4. **`PromptOverlayProvider` 包裹范围过大**：`FullscreenLayout` 将 Provider 包裹了整个布局（包括 `scrollable`、`bottom`、`modal`）。虽然功能正确，但意味着 `modal` 槽内的组件也能写入 prompt overlay，这可能引发意外的覆盖层竞争。

### 改进建议
1. **合并为单一 Context + 拆分消费**：
   - 可考虑使用 React 19 的 `use(Context)` 配合 `React.memo` 的自定义比较函数，或引入 Zustand/Jotai 等原子化状态库，将四重 Context 压缩为一个原子 store，减少嵌套层级。
2. **增加开发环境警告**：
   - 在 `useSetPromptOverlay` 和 `useSetPromptOverlayDialog` 中，当 `set` 为 `null` 且处于开发环境时，输出 `console.warn`，提示开发者检查 Provider 包裹。
3. **互斥锁或优先级机制**：
   - 若未来确实需要同时显示建议和对话框，应引入明确的层级管理（如 `zIndex` 或专门的堆叠上下文）。当前仅靠 JSX 顺序的隐式约定不够健壮。
4. **限制写入范围**：
   - 评估是否只在 `bottom` 子树中挂载 `PromptOverlayProvider`，避免 `modal` 槽意外写入。或者将 Provider 下移到 `PromptInput` 及其兄弟组件的共同父级。
5. **文档化 CC-668 背景**：
   - 在 `FullscreenLayout.tsx` 的 `overflowY="hidden"` 处添加交叉引用注释，指向 `promptOverlayContext.tsx`，帮助未来维护者理解为什么需要这个 Portal 机制。
