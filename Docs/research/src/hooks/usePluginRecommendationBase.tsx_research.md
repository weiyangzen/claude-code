# Research: src/hooks/usePluginRecommendationBase.tsx

## 场景与职责

`usePluginRecommendationBase` 是一个**共享基础 Hook**，为 Claude Code 的插件推荐系统提供统一的状态机和安装辅助逻辑。当前被两类推荐场景复用：

1. **LSP 插件推荐**（`useLspPluginRecommendation.tsx`）：当用户编辑了某类文件且系统已安装对应 LSP 二进制时，推荐安装相关插件。
2. **Claude Code Hint 插件推荐**（`useClaudeCodeHintRecommendation.tsx`）：当 CLI/SDK 向 stderr 输出 `<claude-code-hint />` 标签时，推荐安装提示中指定的插件。

该基础 Hook 的核心职责是**去重、串行化和标准化推荐流程**：避免同一时间内弹出多个推荐、避免远程模式下误弹推荐、以及统一成功/失败的通知 UI。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| **远程模式拦截** | `getIsRemoteMode()` 为 true 时直接跳过推荐，因为远程会话（`--remote` / `--teleport`）的插件环境由远端管理。 |
| **已展示拦截** | 若当前已有 `recommendation` 状态，跳过新的解析，保证 UI 上只有一个推荐弹窗。 |
| **并发解析拦截** | `isCheckingRef` 防止 `tryResolve` 被重复调用（例如 effect 重跑或快速状态变化时）。 |
| **推荐状态管理** | 提供 `recommendation` / `clearRecommendation` / `tryResolve` 三元组，供上层 Hook 驱动。 |
| **安装辅助函数** | `installPluginAndNotify` 封装了从 marketplace 查询插件、执行安装、发送成功/失败通知的完整流程。 |

## 具体技术实现

### 状态机设计

```ts
export function usePluginRecommendationBase() {
  const [recommendation, setRecommendation] = React.useState(null)
  const isCheckingRef = React.useRef(false)

  const tryResolve = React.useCallback((resolve) => {
    if (getIsRemoteMode()) return
    if (recommendation) return
    if (isCheckingRef.current) return

    isCheckingRef.current = true
    resolve()
      .then(rec => { if (rec) setRecommendation(rec) })
      .catch(logError)
      .finally(() => { isCheckingRef.current = false })
  }, [recommendation])

  const clearRecommendation = () => setRecommendation(null)

  return { recommendation, clearRecommendation, tryResolve }
}
```

- `tryResolve` 接收一个返回 `Promise<推荐对象 | null>` 的 `resolve` 函数。
- 上层 Hook 在 `useEffect` 中调用 `tryResolve`，将具体的推荐检测逻辑注入。
- 由于 `tryResolve` 的 identity 随 `recommendation` 变化，清空推荐后会自动触发 effect 重跑，从而重新检测。

### 安装辅助：`installPluginAndNotify`

```ts
export async function installPluginAndNotify(
  pluginId: string,
  pluginName: string,
  keyPrefix: string,
  addNotification: AddNotification,
  install: (pluginData: PluginData) => Promise<void>,
): Promise<void>
```

流程：
1. `getPluginById(pluginId)` 从 marketplace 查询插件元数据。
2. 调用上层传入的 `install` 函数执行实际安装（LSP 推荐和 Hint 推荐的安装逻辑不同）。
3. 安装成功后发送绿色 `figures.tick` 通知；失败时发送红色错误通知。
4. 通知 key 使用 `${keyPrefix}-installed` / `${keyPrefix}-install-failed`，确保不会重复堆叠。

### React Compiler 缓存模式

源码中出现了 `_c`（React Compiler 运行时）和 `$[n]` 缓存槽模式：
```ts
const $ = _c(6)
let t0
if ($[0] !== recommendation) {
  t0 = resolve => { ... }
  $[0] = recommendation
  $[1] = t0
} else {
  t0 = $[1]
}
```
这表明该文件已经过 React Compiler 编译，依赖 identity 比较来避免不必要的闭包重建。

## 关键代码路径与文件引用

| 文件 | 作用 |
|------|------|
| `src/hooks/usePluginRecommendationBase.tsx` | 本文件：基础状态机 + 安装辅助。 |
| `src/hooks/useLspPluginRecommendation.tsx` | 调用方之一：基于文件编辑历史推荐 LSP 插件。 |
| `src/hooks/useClaudeCodeHintRecommendation.tsx` | 调用方之二：基于 stderr hint 标签推荐插件。 |
| `src/utils/plugins/marketplaceManager.ts` | `getPluginById` 来源：负责 marketplace 缓存、查询、git 拉取等。 |
| `src/context/notifications.js` | `useNotifications` / `addNotification` 来源：全局通知系统。 |
| `src/bootstrap/state.js` | `getIsRemoteMode()` 来源：判断当前是否为远程会话模式。 |
| `src/components/LspRecommendation/LspRecommendationMenu.tsx` | UI 层：渲染 LSP 推荐弹窗。 |
| `src/components/ClaudeCodeHint/PluginHintMenu.tsx` | UI 层：渲染 Hint 推荐弹窗。 |

## 依赖与外部交互

### 运行时依赖
- **React / React Compiler Runtime**：状态管理与缓存优化。
- **figures**：终端图标字符（`figures.tick`）。
- **Ink**：`Text` 组件用于通知 JSX。

### 与上层 Hook 的契约
- `tryResolve(resolve)`：注入异步检测逻辑。
- `clearRecommendation()`：用户响应后清空状态。
- `installPluginAndNotify(...)`：统一安装与通知。

### 与 marketplace 的交互
- 通过 `getPluginById` 查询本地缓存的 marketplace 数据；若未命中可能触发网络/git 拉取（具体逻辑在 `marketplaceManager.ts`）。

## 风险、边界与改进建议

### 风险与边界
1. **recommendation 状态是全局单例（per Hook 实例）**：每个使用 `usePluginRecommendationBase` 的 Hook 有自己的状态，但由于 LSP 和 Hint 推荐是分别调用的，理论上可能同时存在两个推荐。不过实际 UI 层（`LspRecommendationMenu` 和 `PluginHintMenu`）通常不会同时渲染，或优先级由调用方控制。
2. **`tryResolve` 的依赖导致 effect 重跑**：`tryResolve` 的 identity 随 `recommendation` 变化，这在 React Compiler 优化后仍保持稳定，但手动维护时容易出错。
3. **安装失败只发通知不恢复状态**：`installPluginAndNotify` 的 catch 块仅发送通知，不会自动重试或回滚，需要用户手动处理。
4. **远程模式判断在解析前**：`getIsRemoteMode()` 在 `tryResolve` 入口处判断，若远程模式在解析过程中切换，可能导致竞态（概率极低）。

### 改进建议
1. **提取为更通用的异步门控 Hook**：`tryResolve` 的模式（gate + async guard + setState）与插件推荐强耦合，但本质上是一个"只执行一次异步任务直到成功"的模式，可考虑抽象为 `useAsyncGate` 通用 Hook。
2. **统一推荐优先级**：若未来出现第三种推荐源，需要显式的优先级仲裁逻辑（如 Hint > LSP），避免用户被多个弹窗打扰。
3. **安装进度可视化**：当前安装是瞬时通知（成功/失败），对于大插件或慢网络，可增加"Installing…" 的进度通知。
4. **类型安全增强**：`recommendation` 当前是 `any`（`useState(null)` 无泛型），建议显式声明联合类型，减少上层隐式依赖。
