# overlayContext.tsx 研究文档

## 场景与职责

`overlayContext.tsx` 解决 **Escape 键在多层 UI 叠加时的协调问题**。当用户打开 Select 下拉框、多选框、历史搜索、帮助面板等“覆盖层（overlay）”时，按下 Escape 的意图通常是“关闭覆盖层”而非“取消当前 AI 请求”。该 Context 通过追踪当前激活的覆盖层集合，让 `CancelRequestHandler` 等全局按键处理器能够：

1. **感知是否有覆盖层激活** (`useIsOverlayActive`) — 有则让 Escape 优先关闭覆盖层。
2. **区分模态与非模态覆盖层** (`useIsModalOverlayActive`) — 模态覆盖层（如 Select）应禁用底层 `TextInput` 的焦点；非模态覆盖层（如 autocomplete）不应抢夺焦点。
3. **自动注册/注销覆盖层** (`useRegisterOverlay`) — 组件挂载即注册，卸载即清理，无需手动状态管理。

## 功能点目的

### 1. `useRegisterOverlay(id, enabled?)`
- 参数 `id` 为覆盖层唯一标识（如 `'select'`、`'multi-select'`、`'autocomplete'`）。
- 参数 `enabled` 默认为 `true`，支持条件注册（例如仅在 `onCancel` 存在时才注册 Select 为覆盖层）。
- 在 `useEffect` 中向 `AppState.activeOverlays` 添加 `id`；effect cleanup 时移除 `id`。
- 额外包含一个 `useLayoutEffect`，在注册后调用 `instances.get(process.stdout)?.invalidatePrevFrame()`，强制 Ink 立即重绘一帧，确保覆盖层视觉状态与按键处理同步。

### 2. `useIsOverlayActive()`
- 通过 `useAppState(s => s.activeOverlays.size > 0)` 订阅全局状态。
- 返回 `true` 当且仅当 `activeOverlays` 集合非空。

### 3. `useIsModalOverlayActive()`
- 遍历 `activeOverlays`，排除 `NON_MODAL_OVERLAYS`（当前仅包含 `'autocomplete'`）。
- 返回 `true` 当存在至少一个模态覆盖层。
- 被 `TextInput.tsx` 等组件用于决定是否保持焦点（`focus: !isModalOverlayActive`）。

## 具体技术实现

### 状态存储
覆盖层状态存储在 `AppState` 中：
```ts
// AppStateStore.ts
activeOverlays: ReadonlySet<string>   // 默认值：new Set<string>()
```

`useRegisterOverlay` 通过 `useContext(AppStoreContext)` 直接获取 store 的 `setState` 方法，绕过 `useSetAppState` Hook，以减少不必要的重渲染（注册方本身通常不需要订阅状态变化）。

### 关键流程
1. **Select 组件注册**：
   ```ts
   // src/components/CustomSelect/use-select-input.ts
   useRegisterOverlay('select', !!state.onCancel)
   ```
   只有当 Select 支持取消（即按 Escape 有意义）时才注册为覆盖层。

2. **多选组件注册**：
   ```ts
   // src/components/CustomSelect/use-multi-select-state.ts
   useRegisterOverlay('multi-select', !!state.onCancel)
   ```

3. **自动补全注册**：
   ```ts
   // src/hooks/useTypeahead.tsx
   useRegisterOverlay('autocomplete', isOpen)
   ```
   `'autocomplete'` 被加入 `NON_MODAL_OVERLAYS`，因此不会禁用 `TextInput` 焦点。

4. **CancelRequestHandler 消费**：
   ```ts
   // src/hooks/useCancelRequest.ts
   const isOverlayActive = useIsOverlayActive()
   const isContextActive =
     screen !== 'transcript' &&
     !isSearchingHistory &&
     !isMessageSelectorVisible &&
     !isLocalJSXCommand &&
     !isHelpOpen &&
     !isOverlayActive &&
     !(isVimModeEnabled() && vimMode === 'INSERT')
   ```
   当 `isOverlayActive` 为 `true` 时，`chat:cancel` 键绑定被禁用，Escape 将交给覆盖层自己的 `select:cancel` 处理。

### 非模态覆盖层白名单
```ts
const NON_MODAL_OVERLAYS = new Set(['autocomplete'])
```

## 关键代码路径与文件引用

| 文件 | 角色 |
|------|------|
| `src/context/overlayContext.tsx` | 本文件，覆盖层注册与查询 Hook |
| `src/state/AppStateStore.ts` | 定义 `activeOverlays` 状态切片 |
| `src/state/AppState.tsx` | 提供 `AppStoreContext`、`useAppState` |
| `src/hooks/useCancelRequest.ts` | 核心消费者，根据覆盖层状态决定是否响应 Escape |
| `src/components/CustomSelect/use-select-input.ts` | 注册 `'select'` 覆盖层 |
| `src/components/CustomSelect/use-multi-select-state.ts` | 注册 `'multi-select'` 覆盖层 |
| `src/hooks/useTypeahead.tsx` | 注册 `'autocomplete'` 非模态覆盖层 |
| `src/components/PromptInput/PromptInput.tsx` | 间接消费 `useIsModalOverlayActive` 控制焦点 |
| `src/components/ThinkingToggle.tsx` | 注册思考模式切换覆盖层 |
| `src/components/QuickOpenDialog.tsx` | 注册快速打开对话框覆盖层 |
| `src/components/HistorySearchDialog.tsx` | 注册历史搜索覆盖层 |

## 依赖与外部交互

- **React**：`useContext`、`useEffect`、`useLayoutEffect`。
- **`../state/AppState.js`**：强依赖 `AppStoreContext` 和 `useAppState`。
- **`../ink/instances.js`**：在 `useLayoutEffect` 中调用 `invalidatePrevFrame()`，强制 Ink 渲染器刷新。

## 风险、边界与改进建议

### 风险与边界
1. **`useRegisterOverlay` 直接消费 `AppStoreContext` 而非 `useSetAppState`**：
   - 虽然减少了注册组件的重渲染，但意味着 `useRegisterOverlay` 必须在 `AppStateProvider` 内部调用，否则会因 `store?.setState` 为 `undefined` 而静默失败（`if (!enabled || !setAppState) return`）。这种静默失败在调试时较难追踪。
2. **`useLayoutEffect` 的硬编码副作用**：
   - 每次 `enabled` 变为 `true` 都会触发 `instances.get(process.stdout)?.invalidatePrevFrame()`。在快速开关覆盖层的场景下（如 autocomplete 频繁显隐），可能导致不必要的额外渲染帧。
3. **`NON_MODAL_OVERLAYS` 是编译期常量**：
   - 当前仅包含 `'autocomplete'`。如果未来新增其他非模态覆盖层（如 inline tooltip），需要修改源码并重新编译，缺乏配置化扩展能力。
4. **覆盖层 ID 的字符串约定**：
   - 使用裸字符串（`'select'`、`'multi-select'`、`'autocomplete'`）作为标识，存在拼写错误风险，且无法被 TypeScript 枚举约束（因为文件被编译后类型信息丢失）。

### 改进建议
1. **导出受控的 Overlay ID 常量**：
   - 将 `'select'`、`'multi-select'`、`'autocomplete'` 等提取为 `export const OVERLAY_SELECT = 'select' as const`，在注册和消费端统一引用，减少拼写错误。
2. **增强 `useRegisterOverlay` 的调试能力**：
   - 在开发环境下，当 `setAppState` 不可用时抛出明确错误（而非静默返回），帮助定位 Provider 包裹问题。
3. **延迟或节流 `invalidatePrevFrame`**：
   - 对 `useLayoutEffect` 中的强制刷新增加 `requestAnimationFrame` 或微任务合并，避免连续注册/注销时的帧浪费。
4. **考虑引入覆盖层栈（Stack）语义**：
   - 当前是简单的 Set，无法表达“后开的覆盖层先关闭”的栈语义。如果未来支持嵌套模态（如 Select 内再弹出 Dialog），Set 无法支持按层级关闭。可评估将 `activeOverlays` 升级为有序数组或栈结构。
5. **与 `ModalContext` 的边界梳理**：
   - `overlayContext` 和 `modalContext` 都涉及“模态”概念，但前者关注 Escape 键协调，后者关注全屏布局的几何与滚动。建议在代码注释或架构文档中明确两者分工，防止新开发者混淆。
