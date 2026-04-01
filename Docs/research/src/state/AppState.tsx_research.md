# src/state/AppState.tsx 研究文档

## 场景与职责

`AppState.tsx` 是 Claude Code 交互式会话的 **React 状态根节点**。它将底层的命令式 store（`createStore`）桥接到 React 组件树，提供：

1. **`AppStateProvider`** — 全局唯一的 Context Provider，负责创建 store、挂载副作用、嵌套检测，并向子树注入 `MailboxProvider` 与 `VoiceProvider`。
2. **`useAppState`** — 基于 `useSyncExternalStore` 的细粒度订阅 Hook，组件可按 selector 订阅切片，避免不必要的重渲染。
3. **`useSetAppState`** / **`useAppStateStore`** — 分别暴露 `setState` 与整个 store 引用，供只写或需要命令式访问的场景使用。
4. **`useAppStateMaybeOutsideOfProvider`** — 安全降级版 Hook，在 Provider 外返回 `undefined`，用于可能在不同渲染上下文复用的组件。

该文件是 **.tsx** 的原因仅在于它使用 JSX 与 React Hook；类型定义已全部迁移到 `AppStateStore.ts`，本文件仅做向后兼容的 re-export。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `AppStateProvider` | 在交互式入口（`App.tsx`）中包裹整个组件树，确保所有 UI 共享同一套 AppState；禁止嵌套 Provider 以防止状态隔离混乱。 |
| 挂载时禁用 bypass permissions | 若远程设置已在挂载前加载且检测到 killswitch，自动将 `toolPermissionContext` 置为 disabled 状态。 |
| `useSettingsChange` 集成 | 通过 `useEffectEvent` 将 `applySettingsChange` 绑定到设置变更监听器，实现磁盘设置热重载。 |
| `useAppState` | 让大量组件以最小粒度订阅状态（如 `useAppState(s => s.verbose)`），利用 `Object.is` 比较实现精确重渲染控制。 |
| `VoiceProvider` 条件注入 | 仅当 `feature('VOICE_MODE')` 为 true 时注入语音 Context；外部构建通过 passthrough 消除死代码。 |

## 具体技术实现

### 1. Store 创建与挂载副作用

```tsx
const [store] = useState(() => createStore(initialState ?? getDefaultAppState(), onChangeAppState))
```

- `useState` 的 lazy init 保证 **单例 store** 只在首次渲染时创建。
- `onChangeAppState`（来自 `onChangeAppState.ts`）作为第二个参数传入 `createStore`，所有 `setState` 触发后都会先执行副作用，再通知订阅者。

挂载 `useEffect` 检查：

```tsx
if (toolPermissionContext.isBypassPermissionsModeAvailable && isBypassPermissionsModeDisabled()) {
  store.setState(prev => ({ ...prev, toolPermissionContext: createDisabledBypassPermissionsContext(prev.toolPermissionContext) }))
}
```

### 2. React Compiler Memo Cache

文件已被 React Compiler（19+）编译，出现 `_c(N)` 形式的 memo cache。所有 JSX 节点与 callback 都通过数组槽位做引用比较，减少不必要的对象重建。

### 3. useSyncExternalStore 订阅模型

```tsx
export function useAppState(selector) {
  const store = useAppStore()
  const get = () => {
    const state = store.getState()
    const selected = selector(state)
    return selected
  }
  return useSyncExternalStore(store.subscribe, get, get)
}
```

- `store.subscribe` 来自 `createStore`，内部维护 `Set<Listener>`。
- 每次 `setState` 后遍历 listener，触发 `useSyncExternalStore` 的重新快照。
- selector 必须返回 **稳定引用**（如已有子对象或原始值），否则会因 `Object.is` 始终不等而导致订阅组件每次都会重渲染。

### 4. 嵌套 Provider 防护

```tsx
const hasAppStateContext = useContext(HasAppStateContext)
if (hasAppStateContext) {
  throw new Error("AppStateProvider can not be nested within another AppStateProvider")
}
```

通过独立的 `HasAppStateContext`（boolean）检测嵌套，比检查 store 是否为 null 更可靠。

## 关键代码路径与文件引用

| 代码路径 | 说明 |
|----------|------|
| `AppStateProvider` (L37) | 根 Provider，被 `src/components/App.tsx` 直接引用。 |
| `useAppStore` (L117) | 内部 Hook，从 `AppStoreContext` 取 store，null 时抛 `ReferenceError`。 |
| `useAppState` (L142) | 被 **100+ 组件/Hook** 使用，如 `src/screens/REPL.tsx`、`src/hooks/useTasksV2.ts`、`src/components/PromptInput/PromptInput.tsx`。 |
| `useSetAppState` (L170) | 被大量 mutation 调用方使用，如 `src/state/teammateViewHelpers.ts`、`src/components/tasks/BackgroundTasksDialog.tsx`。 |
| `useAppStateMaybeOutsideOfProvider` (L186) | 用于可能在 Provider 外渲染的组件，如 `src/components/InvalidConfigDialog.tsx`。 |
| `createStore` (L50) | 来自 `./store.js`，极简订阅式 store。 |
| `applySettingsChange` (L84) | 来自 `../utils/settings/applySettingsChange.js`，热重载入口。 |

## 依赖与外部交互

### 直接依赖

- `./store.js` → `createStore`（命令式 store 工厂）
- `./AppStateStore.js` → `AppState` / `AppStateStore` / `getDefaultAppState` 类型与默认值
- `./onChangeAppState.js` → 全局状态变更副作用回调
- `../context/mailbox.js` → `MailboxProvider`（跨组件消息邮箱）
- `../context/voice.js` → `VoiceProvider`（语音模式，条件 require）
- `../hooks/useSettingsChange.js` → 监听磁盘设置变更
- `../utils/settings/applySettingsChange.js` → 将设置变更应用到 AppState
- `../utils/permissions/permissionSetup.js` → bypass permissions killswitch 检测

### 调用方（上游）

- `src/components/App.tsx`：顶层包装，传入 `initialState` 与 `onChangeAppState`。
- `src/main.tsx`：通过 `getDefaultAppState()` 构造初始状态并注入 `<App>`。
- 几乎所有组件与自定义 Hook 都通过 `useAppState` / `useSetAppState` 消费本文件导出。

## 风险、边界与改进建议

### 风险

1. **Selector 不稳定导致全树重渲染**  
   `useAppState` 的文档已明确警告不要返回新对象，但这是一个常见的 footgun。若某组件写 `useAppState(s => ({ v: s.verbose }))`，会导致该组件在每次 `setState` 时都重渲染，削弱 `useSyncExternalStore` 的优势。

2. **React Compiler 编译后调试困难**  
   文件已被编译器转换，源码中的变量名被压缩为 `t0`, `t1`…，生产环境中堆栈可读性下降；同时若未来需要手动优化 memo，需与编译器输出对抗。

3. **嵌套 Provider 的硬性限制**  
   某些测试或嵌套场景（如画中画、多会话标签页）若需要隔离状态，当前会直接抛错。虽然设计上不支持，但可能限制未来架构演进。

4. **VoiceProvider 的条件 require**  
   使用 `feature('VOICE_MODE') ? require(...) : passthrough` 模式，Bun bundler 的 DCE 必须足够可靠，否则可能将 ant-only 的语音代码带入外部构建。

### 边界

- `useAppStateMaybeOutsideOfProvider` 在 Provider 外使用 `NOOP_SUBSCRIBE`，因此即使 selector 变化也不会触发重渲染；它只会在组件其他原因重渲染时重新计算。
- `useAppState` 内部有一个被 `false &&` 屏蔽的 selector 全状态返回检测，说明团队曾尝试运行时防护，但已关闭（可能因性能开销）。

### 改进建议

1. **引入 selector 开发时校验**  
   可在非生产环境下恢复或实现一个轻量 wrapper，检测 selector 返回值是否 === state，给出可读错误提示。

2. **拆分 Provider 与 Hooks**  
   当前文件混合了 JSX Provider 与 Hook 工厂，且因 .tsx 后缀导致 .ts 调用方误导入。可进一步将 Hook 全部迁移到 `AppStateHooks.ts`，本文件仅保留 Provider。

3. **考虑 useShallow 或 proxy-based 方案**  
   若 selector 不稳定问题持续，可引入 `useShallow`（如 zustand 的做法）或评估 valtio 等 proxy 方案，降低开发者心智负担。

4. **编译产物与源码分离**  
   若 CI 支持，可将 React Compiler 输出放到 `src/__compiled__`，保留原始 TSX 作为 source of truth，便于阅读与审查。
