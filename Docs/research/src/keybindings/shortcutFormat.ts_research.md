# shortcutFormat.ts 研究文档

## 场景与职责

`src/keybindings/shortcutFormat.ts` 提供**非 React 上下文下的快捷键展示文本获取能力**。与 `useShortcutDisplay.ts`（React Hook）形成互补，该模块可在命令处理器、工具类、hooks 等非 React 模块中使用，避免将 React 引入这些纯逻辑层的依赖图。

## 功能点目的

1. **无 React 依赖的快捷键查询**：通过同步加载绑定配置，返回指定动作的快捷键展示文本。
2. **Fallback 机制**：当动作未找到时返回默认文本，并记录分析事件以便监控迁移进度。
3. **单例日志去重**：避免重复记录相同的 fallback 事件，防止分析数据膨胀。

## 具体技术实现

### 关键函数

```ts
export function getShortcutDisplay(
  action: string,
  context: KeybindingContextName,
  fallback: string,
): string
```

#### 执行流程
1. 调用 `loadKeybindingsSync()` 获取当前绑定配置（含默认 + 用户自定义）。
2. 调用 `getBindingDisplayText(action, context, bindings)`（来自 `resolver.ts`）查找匹配。
3. 若找到，返回配置的快捷键字符串；否则：
   - 检查 `LOGGED_FALLBACKS` Set，确保同一 `action:context` 组合仅记录一次分析事件。
   - 记录 `tengu_keybinding_fallback_used` 事件，包含动作名、上下文、fallback 文本和原因。
   - 返回 `fallback` 参数。

### 关键数据结构

```ts
// 模块级状态，用于去重
const LOGGED_FALLBACKS = new Set<string>()
```

键格式：`` `${action}:${context}` ``

### TODO 注释说明

代码中包含重要 TODO：
> "Remove fallback parameter after migration is complete... The fallback exists as a safety net during migration..."

这表明 `fallback` 参数是**过渡期的防御性设计**。一旦确认所有动作都能在绑定配置中找到（无 `keybinding_fallback_used` 事件），即可移除 `fallback` 参数，简化 API。

## 关键代码路径与文件引用

| 函数 | 被调用方 | 用途 |
|------|----------|------|
| `getShortcutDisplay` | `src/hooks/useClipboardImageHint.ts` | 获取图片粘贴提示的快捷键 |
| | `src/hooks/useCancelRequest.ts` | 获取取消请求的快捷键展示 |
| | `src/query/stopHooks.ts` | 停止 hooks 的快捷键提示 |
| | `src/commands/compact/compact.ts` | compact 命令的快捷键展示 |
| | `src/commands/voice/voice.ts` | 语音命令的快捷键提示 |
| | `src/services/tips/tipRegistry.ts` | 提示注册表的快捷键展示 |
| | `src/screens/REPL.tsx` | REPL 界面的快捷键提示 |

## 依赖与外部交互

- `../services/analytics/index.js`：`logEvent` 用于记录 fallback 使用事件。
  - 使用 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 类型标记，确保不泄露代码或文件路径。
- `./loadUserBindings.js`：`loadKeybindingsSync` 同步加载绑定。
- `./resolver.js`：`getBindingDisplayText` 查询展示文本。
- `./types.js`：`KeybindingContextName` 类型。

## 风险、边界与改进建议

### 风险与边界

1. **同步加载的阻塞风险**
   - `loadKeybindingsSync()` 使用 `readFileSync`，在启动路径上可能阻塞事件循环。虽然 keybindings 文件通常很小（<10KB），但在极端情况下（如网络文件系统）可能成为性能瓶颈。

2. **Fallback 日志的持久化问题**
   - `LOGGED_FALLBACKS` 是内存中的 Set，进程重启后会清空。这意味着如果某动作在每次启动时都触发 fallback，会重复记录事件（跨会话无法去重）。这是有意设计（监控的是"每会话首次"而非"全局首次"），但需在数据分析时理解此语义。

3. **TODO 的迁移进度追踪**
   - 代码中的 TODO 表明这是一个临时设计，但缺少明确的追踪机制（如 GitHub issue 链接或截止日期）。存在遗忘风险，导致防御性代码长期保留。

4. **与 React Hook 的 API 不一致**
   - `getShortcutDisplay` 接受 `fallback` 参数，而 `useShortcutDisplay` 也接受 `fallback` 但额外记录 `reason: 'no_context'`。两处逻辑需要保持同步，但代码并未共享实现（除了底层的 `getBindingDisplayText`）。

### 改进建议

1. **移除 Fallback 参数（按 TODO 执行）**
   - 分析 `tengu_keybinding_fallback_used` 事件数据，确认无活跃 fallback 使用后：
     - 移除 `fallback` 参数
     - 将返回值改为 `string | undefined`
     - 调用方改为使用空值合并运算符：`getShortcutDisplay(...) ?? 'default'`
   - 这能简化 API 契约，明确责任边界。

2. **缓存优化**
   - `loadKeybindingsSync` 内部已有缓存（`cachedBindings`），但首次调用仍可能触发文件 IO。考虑在应用启动时预加载绑定，避免首次调用时的延迟。

3. **统一 Fallback 处理逻辑**
   - 将 fallback 检测和日志记录逻辑抽取到共享工具函数，供 `shortcutFormat.ts` 和 `useShortcutDisplay.ts` 共同使用，避免代码重复。

4. **异步替代方案**
   - 对于非关键路径的调用方，可提供 `getShortcutDisplayAsync` 版本，使用 `loadKeybindings()`（异步）避免阻塞。但当前调用方多在渲染/提示路径，同步 API 更符合使用模式。

5. **文档化迁移状态**
   - 在 TODO 注释中增加指向追踪 issue 或文档的链接，明确迁移的判定标准（如"连续 30 天无 fallback 事件"），防止技术债务累积。
