# modalContext.tsx 研究文档

## 场景与职责

`modalContext.tsx` 为 Claude Code 的**全屏布局（FullscreenLayout）模态槽（modal slot）**提供上下文感知能力。当 slash-command 对话框（如 `/config`、`/help`）被渲染在 `FullscreenLayout` 的 `modal` 属性中时，该 Context 让内部组件能够：

1. **感知自身是否处于模态槽中** (`useIsInsideModal`)，从而抑制顶层边框（`Pane` 跳过 `Divider`）。
2. **获取模态槽的可用行列尺寸** (`useModalOrTerminalSize`)，避免 `Select` 分页按终端全高计算导致溢出。
3. **获取模态滚动容器的 ref** (`useModalScrollRef`)，供 `Tabs` 等组件在切换标签时重置滚动位置。

## 功能点目的

### 1. `ModalContext` / `useIsInsideModal`
- `null` 表示不在模态槽中；非 null 对象表示在模态槽中。
- `Pane.tsx` 使用它来决定是否跳过顶部分隔线渲染，防止与 `FullscreenLayout` 已绘制的 `▔` 分隔线重复。

### 2. `useModalOrTerminalSize(fallback)`
- 若在模态槽中，返回模态槽的 `rows` 和 `columns`（已扣除顶部窥视行和分隔线）。
- 若不在模态槽中，返回传入的 `fallback`（通常是 `useTerminalSize()` 结果）。
- 用于 `Select` 等组件计算最大可见选项数，避免内容超出模态可用高度。

### 3. `useModalScrollRef`
- 返回模态槽内部 `ScrollBox` 的 ref（由 `FullscreenLayout` 通过 `modalScrollRef` prop 传入）。
- `Tabs.tsx` 在检测到 `modalScrollRef` 存在时，将 `ScrollBox` 的 `key` 设为 `selectedTabIndex`，并在每次切换标签时复用该 ref，实现“切换标签自动回滚到顶部”的效果。

## 具体技术实现

### 数据结构
```ts
type ModalCtx = {
  rows: number
  columns: number
  scrollRef: RefObject<ScrollBoxHandle | null> | null
}
```

### 关键流程
1. **FullscreenLayout 设置 Context**：
   ```tsx
   // FullscreenLayout.tsx
   modal != null && (
     <ModalContext value={{
       rows: terminalRows - MODAL_TRANSCRIPT_PEEK - 1,  // 默认 peek=2，再减1行分隔线
       columns: columns - 4,                            // 左右各 paddingX=2
       scrollRef: modalScrollRef ?? null
     }}>
       <Box position="absolute" bottom={0} ...>
         <Box flexShrink={0}>
           <Text color="permission">{"▔".repeat(columns)}</Text>
         </Box>
         <Box paddingX={2} ...>{modal}</Box>
       </Box>
     </ModalContext>
   )
   ```

2. **Pane 消费 Context**：
   ```tsx
   // Pane.tsx
   if (useIsInsideModal()) {
     return <Box flexDirection="column" paddingX={1} flexShrink={0}>{children}</Box>
   }
   return (
     <Box flexDirection="column" paddingTop={1}>
       <Divider color={color} />
       <Box flexDirection="column" paddingX={2}>{children}</Box>
     </Box>
   )
   ```

3. **Tabs 消费 scrollRef**：
   ```tsx
   // Tabs.tsx
   const modalScrollRef = useModalScrollRef()
   // ...
   {modalScrollRef ? (
     <ScrollBox key={selectedTabIndex} ref={modalScrollRef} ...>
       {children}
     </ScrollBox>
   ) : (
     <Box ...>{children}</Box>
   )}
   ```
   通过 `key={selectedTabIndex}` 强制 React 在切换标签时卸载并重新挂载 `ScrollBox`，从而将 `scrollTop` 重置为 `0`。

## 关键代码路径与文件引用

| 文件 | 角色 |
|------|------|
| `src/context/modalContext.tsx` | 本文件，定义 Context 和三个消费 Hook |
| `src/components/FullscreenLayout.tsx` | **唯一设置方**，计算模态尺寸并注入 Context |
| `src/components/design-system/Pane.tsx` | 消费者，抑制模态内重复边框 |
| `src/components/design-system/Tabs.tsx` | 消费者，利用 `modalScrollRef` 重置滚动 |
| `src/ink/components/ScrollBox.tsx` | `ScrollBoxHandle` 类型定义 |
| `src/commands/*`（如 `config.tsx`、`help.tsx`、`theme.tsx` 等） | 间接消费方，这些命令组件通常渲染在 `Pane` 或 `Tabs` 内 |

## 依赖与外部交互

- **React**：`createContext`、`useContext`、`RefObject`。
- **`../ink/components/ScrollBox.js`**：类型依赖 `ScrollBoxHandle`。
- **`FullscreenLayout.tsx`**：强耦合，Context 的生命周期和值完全由它控制。

## 风险、边界与改进建议

### 风险与边界
1. **`columns - 4` 的硬编码假设**：`FullscreenLayout` 假设模态内容左右总内边距为 `4`（即 `paddingX={2}`）。如果 `Pane` 或其他容器调整了模态内的水平内边距，`columns` 计算将不准确，可能导致内容折行或截断异常。
2. **`rows` 计算与 `MODAL_TRANSCRIPT_PEEK` 耦合**：`rows = terminalRows - MODAL_TRANSCRIPT_PEEK - 1` 中的 `-1` 是为了扣除分隔线行。若未来分隔线被移除或改为多行，该公式会失效。
3. **`useModalOrTerminalSize` 的 fallback 必须稳定**：该 Hook 将 `fallback` 对象加入 memo cache 依赖数组。如果调用方每次渲染都创建新的 fallback 对象（如 `useModalOrTerminalSize({ rows, columns })` 中对象字面量未缓存），会导致 Hook 每次返回新引用，触发下游重渲染。

### 改进建议
1. **提取模态尺寸计算函数**：将 `terminalRows → modalRows` 和 `columns → modalColumns` 的转换逻辑提取为纯函数并添加单元测试，避免魔法数字散落在 JSX 中。
2. **统一 padding 常量**：将 `paddingX={2}` 和 `columns - 4` 中的 `4` 关联到同一常量（如 `MODAL_PADDING_X = 2`，`MODAL_COLUMNS_DEDUCTION = MODAL_PADDING_X * 2`）。
3. **为 `useModalOrTerminalSize` 增加 fallback 稳定化**：在 Hook 内部对 fallback 进行浅比较或结构稳定化，减少调用方的 memo 负担。
4. **考虑 Context 拆分**：当前 `ModalContext` 同时承载“是否在模态中”、“尺寸”、“scrollRef”三个语义。若未来模态槽支持嵌套或多种模态类型，可拆分为 `ModalPresenceContext` 和 `ModalGeometryContext`，降低无关消费者的重渲染。
