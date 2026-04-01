# useShortcutDisplay.ts 研究文档

## 场景与职责

`src/keybindings/useShortcutDisplay.ts` 提供**React Hook 形式的快捷键展示文本获取能力**。与 `shortcutFormat.ts`（纯函数）形成互补，该 Hook 适用于函数组件内部，用于动态显示当前配置的快捷键（如按钮提示、帮助文本）。它封装了 fallback 处理和分析日志记录，确保 UI 与配置的一致性。

## 功能点目的

1. **React 友好的快捷键查询**：通过 Hook 接口获取动作对应的快捷键展示文本。
2. **Fallback 安全网**：当绑定未找到或上下文不可用时返回默认文本，并记录分析事件。
3. **单次日志记录**：使用 `useRef` 确保每个组件实例仅记录一次 fallback 事件，避免渲染循环中的重复日志。

## 具体技术实现

### 关键 Hook

```ts
export function useShortcutDisplay(
  action: string,
  context: KeybindingContextName,
  fallback: string,
): string
```

#### 执行流程
1. 通过 `useOptionalKeybindingContext()` 获取 keybinding 上下文。
2. 调用 `keybindingContext?.getDisplayText(action, context)` 查询展示文本。
3. 判断是否为 fallback：
   - `resolved === undefined` 表示未找到绑定。
   - `reason` 分为两种情况：
     - 上下文存在但动作未找到：`'action_not_found'`
     - 上下文不存在（`!keybindingContext`）：`'no_context'`
4. **日志记录**（`useEffect`）：
   - 使用 `hasLoggedRef` 确保仅首次 fallback 时记录。
   - 发送 `tengu_keybinding_fallback_used` 分析事件，包含动作、上下文、fallback 文本和原因。
5. 返回 `resolved` 或 `fallback`。

### 关键设计决策

1. **单次日志的 useRef 模式**
   ```ts
   const hasLoggedRef = useRef(false)
   useEffect(() => {
     if (isFallback && !hasLoggedRef.current) {
       hasLoggedRef.current = true
       logEvent(...)
     }
   }, [isFallback, ...])
   ```
   - 避免每次渲染（如状态更新导致的重渲染）都发送分析事件。
   - 组件卸载后重新挂载会重置 ref，允许再次记录（符合"每实例首次"语义）。

2. **与 shortcutFormat.ts 的差异**
   | 特性 | `useShortcutDisplay` | `getShortcutDisplay` |
   |------|----------------------|----------------------|
   | 上下文 | React Hook | 纯函数 |
   | 日志去重 | useRef 每实例 | Set 每进程 |
   | 无上下文原因 | `'no_context'` | 不适用（直接 fallback） |
   | 适用场景 | 组件内部 | Hooks、命令、工具类 |

3. **TODO 注释**
   - 与 `shortcutFormat.ts` 相同的迁移 TODO：计划在确认无 fallback 使用后移除 `fallback` 参数。

## 关键代码路径与文件引用

| Hook | 被调用方 | 用途 |
|------|----------|------|
| `useShortcutDisplay` | `src/components/Messages.tsx` | 消息展开/折叠快捷键提示 |
| | `src/components/ResumeTask.tsx` | 恢复任务快捷键 |
| | `src/components/ThemePicker.tsx` | 主题选择器快捷键 |
| | `src/components/ConfigurableShortcutHint.tsx` | 可配置快捷键提示组件 |
| | `src/components/PromptInput/PromptInputHelpMenu.tsx` | 输入框帮助菜单 |
| | `src/hooks/useTypeahead.tsx` | 类型提示快捷键 |
| | `src/screens/REPL.tsx` | REPL 界面快捷键展示 |

## 依赖与外部交互

- `react`：`useEffect`, `useRef`。
- `../services/analytics/index.js`：`logEvent` 用于记录 fallback 使用。
  - 使用 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 类型标记。
- `./KeybindingContext.js`：`useOptionalKeybindingContext` —— 获取上下文。
- `./types.js`：`KeybindingContextName` —— 上下文类型。

## 风险、边界与改进建议

### 风险与边界

1. **上下文缺失的静默处理**
   - 若组件在 `KeybindingProvider` 外渲染（如应用启动早期），`useOptionalKeybindingContext()` 返回 `undefined`，Hook 直接返回 `fallback`。
   - 这可能导致 UI 显示默认快捷键，而实际绑定可能不同（若用户自定义）。但这种情况通常只在应用初始化瞬间出现，影响有限。

2. **StrictMode 下的双重日志**
   - React StrictMode 会故意双重调用某些函数以检测副作用。虽然 `useRef` 模式通常能抵御（ref 在两次调用间保持），但在某些 React 版本或并发模式下可能存在边缘情况。

3. **Fallback 参数的过渡性**
   - 与 `shortcutFormat.ts` 相同，`fallback` 参数是临时防御措施。若迁移完成后未移除，会导致 API 冗余和调用方责任不清。

4. **与 shortcutFormat.ts 的逻辑重复**
   - fallback 检测、日志记录、返回值选择等逻辑在两处重复实现（虽然底层都使用 `getBindingDisplayText`）。若需修改日志字段或 fallback 语义，需要同步更新两处。

### 改进建议

1. **统一 Fallback 逻辑**
   - 将 fallback 处理和日志记录抽取到共享工具函数，供 Hook 和纯函数版本共同使用。例如：
     ```ts
     function handleFallback(action, context, fallback, reason, hasLoggedFlag)
     ```

2. **移除 Fallback 参数（按 TODO 执行）**
   - 分析 `tengu_keybinding_fallback_used` 事件数据，确认无活跃 fallback 后：
     - 移除 `fallback` 参数
     - 返回类型改为 `string | undefined`
     - 调用方使用 `??` 运算符提供默认值

3. **上下文缺失警告（开发模式）**
   - 在开发模式下，若检测到 `!keybindingContext`，可输出 `console.warn` 提示开发者组件可能渲染在 Provider 外，帮助定位布局问题。

4. **支持动态 Action**
   - 当前 `action` 参数变化时会重新查询和记录日志。若 action 频繁变化（如基于状态的动态动作），可能导致多次日志。可考虑增加 `action` 变化时的去重逻辑，或明确文档化此行为。

5. **与 `types.ts` 的关联**
   - 当前依赖 `./types.js` 的 `KeybindingContextName`，若该类型定义变更（如新增上下文），Hook 的参数类型应同步更新。建议建立类型级别的关联。

## 与相关模块的对比

```
useShortcutDisplay.ts          shortcutFormat.ts
        │                            │
        ├── React Hook API           ├── 纯函数 API
        ├── useRef 单次日志          ├── Set 进程级去重
        ├── 感知上下文缺失           ├── 不感知上下文（直接加载）
        └── 组件级使用               └── 通用工具使用
        
共同依赖：
    ├── KeybindingContext.tsx (getDisplayText)
    ├── resolver.ts (getBindingDisplayText)
    └── analytics/index.ts (logEvent)
```

该模块在整个 keybindings 系统中处于"展示层"，负责将内部绑定配置转换为人类可读的 UI 文本，是提升用户体验的关键环节。
