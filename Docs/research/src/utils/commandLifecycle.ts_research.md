# commandLifecycle.ts 研究文档

> 文件路径：`src/utils/commandLifecycle.ts`  
> 行数：21 行  
> 研究日期：2026-04-01

---

## 场景与职责

在 Claude Code 的交互式 CLI 中，某些长生命周期命令（如 REPL 中的代码执行、远程 IO 操作）需要向外部监听器通知自己的状态变化：何时开始、何时完成。`commandLifecycle.ts` 提供了一个极简的**发布-订阅（pub/sub）通道**，用于在命令执行器与外部 UI/状态管理之间传递生命周期事件。

由于该模块位于工具层（`src/utils`），它的设计目标是**零依赖、全局单例、类型安全**，让任何模块都能在不引入复杂事件总线的情况下注册生命周期回调。

---

## 功能点目的

| 导出项 | 签名 | 目的 |
|--------|------|------|
| `setCommandLifecycleListener` | `(cb: CommandLifecycleListener \| null) => void` | 注册或注销唯一的生命周期监听器 |
| `notifyCommandLifecycle` | `(uuid: string, state: CommandLifecycleState) => void` | 通知指定 UUID 的命令状态变更 |

**状态类型：**
```ts
type CommandLifecycleState = 'started' | 'completed'
```

---

## 具体技术实现

### 3.1 实现全貌

```ts
type CommandLifecycleState = 'started' | 'completed'

type CommandLifecycleListener = (
  uuid: string,
  state: CommandLifecycleState,
) => void

let listener: CommandLifecycleListener | null = null

export function setCommandLifecycleListener(
  cb: CommandLifecycleListener | null,
): void {
  listener = cb
}

export function notifyCommandLifecycle(
  uuid: string,
  state: CommandLifecycleState,
): void {
  listener?.(uuid, state)
}
```

### 3.2 设计特点

1. **单监听器模型**
   只维护一个 `listener` 变量，而非监听器数组。这意味着：
   - 调用方（通常是应用启动时的某个初始化模块）拥有对监听器的排他控制权。
   - 避免了多监听器带来的去重、泄漏、调用顺序等复杂度。
   - 若多次调用 `setCommandLifecycleListener`，后一次会覆盖前一次。

2. **可选链调用**
   `notifyCommandLifecycle` 使用 `listener?.(...)`，在未注册监听器时静默无操作，调用方无需前置判空。

3. **以 UUID 为标识**
   命令通过 `uuid` 区分，允许并发执行多个命令时，监听器可以精确追踪每一条命令的起止。

---

## 关键代码路径与文件引用

### 调用方（通知端）
| 文件 | 调用场景 |
|------|----------|
| `src/cli/structuredIO.ts` | 结构化 IO 命令开始/完成时通知 |
| `src/cli/print.ts` | 打印输出命令的生命周期 |
| `src/cli/remoteIO.ts` | 远程 IO 操作的生命周期 |
| `src/query.ts` | 查询执行的生命周期 |

### 消费方（监听端）
由于该模块是单例 pub/sub，监听器的注册点通常在应用初始化阶段。根据调用方分布，可能的注册点包括：
- `src/screens/REPL.tsx` 或 `src/main.tsx`：REPL 屏幕需要知道当前命令是否还在执行，以更新加载状态或允许中断。
- `src/state/AppStateStore.ts` 或相关状态管理：追踪全局命令执行计数。

---

## 依赖与外部交互

- **零外部依赖**：不导入任何其他模块。
- **纯内存状态**：模块级单例 `listener`。
- **无副作用**：不触及 DOM、不操作文件、不发起网络请求。

---

## 风险、边界与改进建议

### 风险与边界

1. **单监听器覆盖**
   如果两个不相关的模块都尝试 `setCommandLifecycleListener`，后注册的会把先注册的覆盖掉，导致先注册的模块收不到事件。目前代码中没有断言或警告。

2. **无类型导出**
   `CommandLifecycleState` 和 `CommandLifecycleListener` 只在模块内部定义，未导出。如果外部模块想存储监听器引用或封装包装函数，需要自行重复定义类型。

3. **无卸载保护**
   若注册监听器的模块被热重载或动态卸载，但忘记在卸载前调用 `setCommandLifecycleListener(null)`，旧的闭包可能继续持有已卸载模块的引用，造成内存泄漏。

### 改进建议

1. **导出类型**
   将 `CommandLifecycleState` 和 `CommandLifecycleListener` 改为 `export type`，方便下游复用。

2. **增加调试覆盖警告**
   在 `setCommandLifecycleListener` 中增加开发环境断言：
   ```ts
   if (process.env.NODE_ENV !== 'production' && listener !== null && cb !== null) {
     console.warn('[commandLifecycle] Overwriting existing lifecycle listener')
   }
   ```

3. **支持多监听器（可选）**
   如果未来有多个独立模块需要监听命令生命周期，可将内部实现改为 `Set<CommandLifecycleListener>`，并增加 `removeCommandLifecycleListener` API。但需评估是否值得引入复杂度。

4. **增加 `error` 状态**
   目前只有 `started` 和 `completed`。实际命令可能失败或被取消，建议扩展为：
   ```ts
   type CommandLifecycleState = 'started' | 'completed' | 'failed' | 'cancelled'
   ```
   这样监听器可以更准确地反映命令结果。
