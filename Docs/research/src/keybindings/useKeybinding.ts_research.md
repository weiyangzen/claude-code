# useKeybinding.ts 研究文档

## 场景与职责

`src/keybindings/useKeybinding.ts` 是 Claude Code 快捷键系统的**React 集成层**。它提供 `useKeybinding` 和 `useKeybindings` 两个 Hook，将 Ink 的原始输入事件与声明式的快捷键配置连接起来。这是组件开发者使用快捷键系统的主要入口，负责处理动作注册、输入拦截、chord 状态管理等复杂逻辑。

## 功能点目的

1. **声明式快捷键绑定**：组件通过 `useKeybinding('action:name', handler)` 声明对特定动作的兴趣。
2. **多动作批量绑定**：`useKeybindings` 允许一次注册多个动作处理器，减少 `useInput` 调用次数。
3. **Chord 状态管理**：自动处理 chord 序列的开始、完成和取消，与 `KeybindingContext` 协同维护 pending 状态。
4. **事件传播控制**：通过 `stopImmediatePropagation()` 确保已处理的快捷键不会触发其他处理器。

## 具体技术实现

### 关键 Hook

#### `useKeybinding(action, handler, options?)`

```ts
function useKeybinding(
  action: string,
  handler: () => void | false | Promise<void>,
  options?: { context?: KeybindingContextName; isActive?: boolean }
): void
```

**执行流程：**
1. **处理器注册**（`useEffect`）：
   - 通过 `keybindingContext.registerHandler` 将 `{action, context, handler}` 注册到上下文。
   - 返回清理函数，组件卸载时自动注销。

2. **输入处理**（`useInput`）：
   - 构建上下文优先级列表：`[...activeContexts, context, 'Global']`，去重保留首次出现顺序。
   - 调用 `keybindingContext.resolve(input, key, uniqueContexts)` 解析按键。
   - 根据返回的 `ChordResolveResult` 类型处理：
     - `'match'`：清除 pending chord，若匹配当前 action 则执行 handler，返回非 `false` 时阻止传播。
     - `'chord_started'`：更新 pending 状态，阻止传播等待下一键。
     - `'chord_cancelled'`：清除 pending 状态。
     - `'unbound'`：清除 pending，阻止传播（显式解绑的键不应继续传递）。
     - `'none'`：不处理，允许其他处理器尝试。

#### `useKeybindings(handlers, options?)`

```ts
function useKeybindings(
  handlers: Record<string, () => void | false | Promise<void>>,
  options?: { context?: KeybindingContextName; isActive?: boolean }
): void
```

- 批量注册多个 action handler，内部循环调用 `registerHandler`。
- 输入处理逻辑与 `useKeybinding` 相同，但需检查 `result.action in handlers` 确定是否执行对应 handler。

### 关键设计决策

1. **Handler 返回 `false` 的含义**
   - 返回 `false` 表示"未消费"，事件继续传播。这允许 fall-through 行为，如 `ScrollKeybindingHandler` 在内容无需滚动时将事件传递给子组件。
   - `Promise<void>` 返回值仅用于 fire-and-forget 异步操作，`!== false` 检查仅针对同步 `false`。

2. **上下文优先级**
   - 构建的上下文列表顺序：`activeContexts`（动态注册的）→ 当前 Hook 的 `context` → `'Global'`。
   - 使用 `Set` 去重时保留首次出现顺序，确保更具体的上下文优先。

3. **与 ChordInterceptor 的协作**
   - 注释提到处理器注册到上下文是"for ChordInterceptor to invoke"，表明存在（或计划中的）一个 chord 拦截器组件，可在全局层面处理 chord 超时和状态显示。

## 关键代码路径与文件引用

| Hook | 被调用方 | 用途 |
|------|----------|------|
| `useKeybinding` | 遍布各 UI 组件（50+ 处） | 单动作快捷键绑定 |
| | `Messages.tsx`、`PromptInput.tsx` | 聊天相关动作 |
| | `PermissionRequest.tsx`、`TrustDialog.tsx` | 权限/信任对话框 |
| | `ThemePicker.tsx`、`Settings.tsx` | 设置界面 |
| `useKeybindings` | `useGlobalKeybindings.tsx` | 全局快捷键 |
| | `useHistorySearch.ts` | 历史搜索导航 |
| | `MessageSelector.tsx` | 消息选择器 |

## 依赖与外部交互

- `react`：`useCallback`, `useEffect`。
- `../ink.js`：`Key`, `useInput` —— Ink 的输入处理基础。
- `../ink/events/input-event.js`：`InputEvent` —— 用于 `stopImmediatePropagation()`。
- `./KeybindingContext.js`：`useOptionalKeybindingContext` —— 获取 keybinding 上下文。
- `./types.js`：`KeybindingContextName` —— 上下文类型。

## 风险、边界与改进建议

### 风险与边界

1. **useInput 的嵌套与顺序**
   - Ink 的 `useInput` 按照组件挂载顺序建立处理器队列，`stopImmediatePropagation()` 只能阻止后续处理器。若 `useKeybinding` 在组件树中挂载较晚，可能无法拦截到已处理的输入。

2. **Handler 的引用稳定性**
   - `useCallback` 依赖 `[action, context, handler, keybindingContext]`，若 `handler` 是内联函数，会导致每次渲染都重新创建回调，进而重新注册 input handler。

3. **Chord 超时的无感知**
   - Hook 本身不处理 chord 超时，依赖 `KeybindingProviderSetup.tsx` 中的 `ChordInterceptor` 或类似机制。若该组件未渲染，chord 可能永远停留在 pending 状态。

4. **批量绑定的原子性**
   - `useKeybindings` 在 `useEffect` 中逐个注册 handler，若组件在注册过程中卸载，部分 handler 可能已注册而未清理。

### 改进建议

1. **性能优化：避免重复上下文构建**
   - 当前每次输入都重新构建 `contextsToCheck` 数组和 `uniqueContexts` Set。可考虑在依赖变化时预计算。

2. **开发模式警告**
   - 若 `handler` 返回 `Promise` 且未正确处理错误，可能导致未捕获的 Promise 拒绝。可在开发模式下增加 `Promise.catch` 包装。

3. **Chord 超时暴露**
   - 考虑通过 Hook 返回 pending 状态或提供 `useChordState` Hook，让组件可自行实现 chord 提示 UI。

4. **Action 类型安全**
   - 当前 `action` 参数为 `string`，调用方可传入任意字符串，拼写错误只能在运行时通过 fallback 日志发现。可考虑结合 `schema.ts` 的 `KEYBINDING_ACTIONS` 生成类型安全的 action 联合类型。

5. **与 `types.ts` 的关联**
   - 当前依赖 `./types.js` 的 `KeybindingContextName`，若该类型定义变更（如新增上下文），Hook 的 `context` 参数类型应同步更新。
