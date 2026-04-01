# 研究文档：src/utils/hooks/sessionHooks.ts

> 生成时间：2026-04-01  
> 研究范围：代码、类型定义、调用方、被调用方、状态存储、执行链路  
> 文件大小：约 12.1 KB

---

## 1. 场景与职责

`sessionHooks.ts` 是 Claude Code 钩子（hook）子系统中负责**会话级（session-scoped）钩子生命周期管理**的核心模块。与持久化在 `settings.json` 中的配置钩子不同，session hooks 是**纯内存、临时、按会话隔离**的运行时钩子，常用于：

- **Skill / Agent 前置声明（frontmatter）钩子**：当用户加载某个 skill 或启动 subagent 时，将其声明的 hooks 注册到当前会话，会话结束自动清理。
- **结构化输出强制（structured output enforcement）**：在 agent hook 执行期间，通过 `FunctionHook` 动态注入一个 TypeScript 回调，强制子 agent 在停止前调用 `SyntheticOutputTool`。
- **运行时动态拦截**：如 `addFunctionHook` 提供的回调能力，可在特定 `HookEvent`（如 `Stop`）触发时，以同步/异步函数拦截对话流程。

核心职责概括为：
1. 按 `sessionId` 增删查改钩子；
2. 支持两种钩子形态：`HookCommand`（可序列化的命令/提示词/HTTP/agent 钩子）与 `FunctionHook`（不可序列化的内存回调）；
3. 提供批量清理能力，防止会话结束后内存泄漏；
4. 在高并发场景（如 `parallel()` 多 agent）下保证 `O(1)` 的写入性能。

---

## 2. 功能点目的

| 功能点 | 目的 |
|--------|------|
| `addSessionHook` | 向指定会话的某个事件注册一个 `HookCommand`，支持 matcher 与可选的 `skillRoot`，用于 frontmatter/skill 钩子注入。 |
| `addFunctionHook` | 向指定会话注册一个内存回调型钩子，返回唯一 `id` 供后续移除。用于结构化输出强制、运行时验证等需要代码逻辑的场景。 |
| `removeFunctionHook` | 按 `id` 精确移除某个 `FunctionHook`，清理空 matcher 与空事件条目。 |
| `removeSessionHook` | 按 `HookCommand` 内容比对移除（用于 `once: true` 的一次性钩子自清理）。 |
| `getSessionHooks` | 读取某会话下所有/某事件的 `HookCommand` 类型钩子（过滤掉 function 类型，因其不可持久化）。 |
| `getSessionFunctionHooks` | 单独读取某会话下所有/某事件的 `FunctionHook` 类型钩子，与上者隔离，避免混入持久化链路。 |
| `getSessionHookCallback` | 精确检索某个钩子的完整定义（包含 `onHookSuccess` 回调），用于钩子执行成功后触发后续逻辑（如一次性钩子自移除）。 |
| `clearSessionHooks` | 按 `sessionId` 整体删除，防止 agent/skill 会话结束后钩子泄漏到其他会话。 |

---

## 3. 具体技术实现

### 3.1 状态存储：Map 而非 Record

```ts
export type SessionHooksState = Map<string, SessionStore>
```

设计文档注释非常详细：使用 `Map` 而非 `Record` + spread 的核心原因是**性能与响应式短路**：

- 在高并发工作流（如 `parallel()` 启动 N 个 schema-mode agent）中，可能在一个同步 tick 内发生 N 次 `addFunctionHook`。
- 若用 `Record` + `{ ...prev, [sessionId]: ... }`，每次写入都是 `O(N)` 的浅拷贝，总复杂度 `O(N²)`，且会触发约 30 个 store listener 的重新计算。
- 使用 `Map` 后，`setAppState` 的 updater 直接对 `prev.sessionHooks.set(...)` 进行**可变写入**，然后 `return prev`。由于 `Object.is(next, prev)` 为 `true`，Zustand（或同类 store）会短路通知，实现**零监听器触发**。

> ⚠️ 这意味着 `sessionHooks` 虽然活在 React 状态树里，但**不被 reactive 读取**，只在查询循环中通过 `getAppState()` 快照访问。

### 3.2 数据结构

```ts
type SessionHookMatcher = {
  matcher: string
  skillRoot?: string
  hooks: Array<{ hook: HookCommand | FunctionHook; onHookSuccess?: OnHookSuccess }>
}

export type SessionStore = {
  hooks: { [event in HookEvent]?: SessionHookMatcher[] }
}
```

- 按 `HookEvent`（如 `Stop`、`PreToolUse` 等）分桶；
- 每个事件下是 `SessionHookMatcher[]`，支持同一 `matcher` + `skillRoot` 组合下的多钩子追加；
- `onHookSuccess` 回调用于 `once: true` 钩子的自清理（见 `registerSkillHooks.ts`）。

### 3.3 FunctionHook 的定义与限制

```ts
export type FunctionHook = {
  type: 'function'
  id?: string
  timeout?: number
  callback: FunctionHookCallback
  errorMessage: string
  statusMessage?: string
}

export type FunctionHookCallback = (
  messages: Message[],
  signal?: AbortSignal,
) => boolean | Promise<boolean>
```

- `callback` 返回 `true` 表示校验通过，`false` 则阻塞当前操作，并向模型/用户展示 `errorMessage`。
- `timeout` 默认 5000ms，可被调用方覆盖。
- **不可序列化**：`getSessionHooks` 会主动过滤 `type === 'function'` 的条目，防止其进入 settings.json 或 UI 持久化链路。
- **不可比较**：`hooksSettings.ts` 中的 `isHookEqual` 对 `function` 类型直接返回 `false`，因此 `removeSessionHook` 对 `FunctionHook` 无效，必须使用 `removeFunctionHook(id)`。

### 3.4 关键流程

#### 添加钩子（`addHookToSession`）
1. `setAppState(prev => { ... })` 获取或创建 `SessionStore`；
2. 按 `(matcher, skillRoot)` 查找已有 `SessionHookMatcher`，命中则追加 hook，否则新建 matcher；
3. 用 `{ ...store.hooks, [event]: updatedMatchers }` 生成新 hooks 对象（此处仅对 hooks 对象做不可变更新，外层 Map 直接 mutate）；
4. `prev.sessionHooks.set(sessionId, { hooks: newHooks })`，返回 `prev`。

#### 移除 FunctionHook（`removeFunctionHook`）
1. 遍历该事件下所有 matcher；
2. 对每个 matcher 的 `hooks` 数组过滤掉 `hook.id === targetId` 且 `type === 'function'` 的项；
3. 若 matcher 的 hooks 为空，则丢弃该 matcher；若事件下 matcher 为空，则从 `store.hooks` 中删除该事件键；
4. 同样通过 `prev.sessionHooks.set(...)` mutate 并返回 `prev`。

#### 获取并合并到主钩子链路（`getHooksConfig` in `src/utils/hooks.ts`）
```ts
const sessionHooks = getSessionHooks(appState, sessionId, hookEvent).get(hookEvent)
const sessionFunctionHooks = getSessionFunctionHooks(appState, sessionId, hookEvent).get(hookEvent)
```

- 在 `getHooksConfig` 中，session hooks 被合并到全局 hooks 列表的末尾（在 snapshot hooks 与 registered hooks 之后）。
- 若 `shouldAllowManagedHooksOnly()` 为 `true`，则**完全跳过** session hooks，防止 frontmatter 钩子绕过企业策略。

#### 执行 FunctionHook（`executeFunctionHook` in `src/utils/hooks.ts`）
```ts
Promise.resolve(hook.callback(messages, abortSignal))
```

- 仅在 REPL 主线程上下文中执行（需要 `messages` 数组）；
- 若 `callback` 返回 `false`，则产生 `blocking` 结果，向模型展示 `hook.errorMessage`；
- 若抛出异常，则产生 `non_blocking_error`。

---

## 4. 关键代码路径与文件引用

### 4.1 本文件导出的符号

| 符号 | 类型 | 说明 |
|------|------|------|
| `SessionHooksState` | type alias | `Map<string, SessionStore>` |
| `SessionStore` | type alias | 按事件分桶的 matcher 存储 |
| `FunctionHook` | type alias | 内存回调钩子定义 |
| `FunctionHookCallback` | type alias | 回调签名 `(messages, signal) => boolean \| Promise<boolean>` |
| `addSessionHook` | function | 注册 HookCommand 类型会话钩子 |
| `addFunctionHook` | function | 注册 FunctionHook，返回 id |
| `removeFunctionHook` | function | 按 id 移除 FunctionHook |
| `removeSessionHook` | function | 按 HookCommand 内容移除 |
| `getSessionHooks` | function | 获取可序列化的会话钩子 |
| `getSessionFunctionHooks` | function | 获取 FunctionHook 类型钩子 |
| `getSessionHookCallback` | function | 获取完整钩子条目（含 onHookSuccess）|
| `clearSessionHooks` | function | 清空某会话所有钩子 |

### 4.2 上游调用方

| 文件 | 调用符号 | 用途 |
|------|----------|------|
| `src/utils/hooks/registerSkillHooks.ts` | `addSessionHook`, `removeSessionHook` | 将 skill frontmatter 中的 hooks 注册为会话钩子；`once: true` 钩子通过 `onHookSuccess` 自清理。 |
| `src/utils/hooks/registerFrontmatterHooks.ts` | `addSessionHook` | 将 agent/skill frontmatter hooks 注册到会话；agent 的 `Stop` 事件会被映射为 `SubagentStop`。 |
| `src/utils/hooks/hookHelpers.ts` | `addFunctionHook` | `registerStructuredOutputEnforcement` 在 agent hook 子 agent 启动前注入结构化输出强制回调。 |
| `src/utils/swarm/teammateInit.ts` | `addFunctionHook` | （根据 grep） teammate 初始化时可能注入函数钩子。 |
| `src/tools/AgentTool/runAgent.ts` | `clearSessionHooks` | agent 结束时清理其会话钩子，防止泄漏。 |
| `src/utils/hooks/execAgentHook.ts` | `clearSessionHooks` | agent hook 执行完毕后清理为子 agent 注册的 structured output enforcement hook。 |

### 4.3 下游消费方

| 文件 | 消费符号 | 用途 |
|------|----------|------|
| `src/utils/hooks.ts` | `getSessionHooks`, `getSessionFunctionHooks`, `getSessionHookCallback`, `clearSessionHooks` | 在 `getHooksConfig`、`executeHooks`、`executeStopHooks` 等主链路中合并并执行会话钩子。 |
| `src/utils/hooks/hooksSettings.ts` | `getSessionHooks` | `getAllHooks` 将 session hooks 纳入 UI 展示（设置面板中的 Hooks 列表）。 |
| `src/state/AppStateStore.ts` | `SessionHooksState` | 定义 `AppState.sessionHooks` 的类型与初始值 `new Map()`。 |

---

## 5. 依赖与外部交互

### 5.1 直接依赖

| 模块 | 用途 |
|------|------|
| `src/entrypoints/agentSdkTypes.js` | `HOOK_EVENTS` 常量与 `HookEvent` 类型 |
| `src/state/AppState.js` | `AppState` 类型，用于 `setAppState` 回调签名 |
| `src/types/message.js` | `Message` 类型（`FunctionHookCallback` 参数） |
| `src/utils/debug.js` | `logForDebugging` 日志输出 |
| `src/utils/hooks.js` | `AggregatedHookResult` 类型（`OnHookSuccess` 回调参数） |
| `src/utils/settings/types.js` | `HookCommand` 类型 |
| `src/utils/hooks/hooksSettings.ts` | `isHookEqual` 用于 `removeSessionHook` 的内容比对 |

### 5.2 运行时依赖（调用方带入）

- `setAppState`：由 React/Zustand store 提供，所有 mutate 操作均通过它完成；
- `appState` / `sessionId`：由查询主循环或 agent 生命周期管理传入；
- `messages` 数组：仅在 `FunctionHook` 执行时由 REPL 上下文提供。

---

## 6. 风险、边界与改进建议

### 6.1 风险

1. **内存泄漏风险**
   - 若 `clearSessionHooks` 未被调用（如 agent 异常崩溃、流程中断），`Map` 中的钩子会一直保留。
   - 虽然 `Map` 本身在内存中，但 `FunctionHook` 可能闭包引用大量上下文，长期运行进程存在缓慢膨胀风险。

2. **FunctionHook 的身份比较缺陷**
   - `isHookEqual` 对 `function` 类型永远返回 `false`，因此 `removeSessionHook` 对 `FunctionHook` 无效。开发者必须牢记使用 `removeFunctionHook(id)`，否则可能产生“移除失败”的隐蔽 bug。

3. **并发写入的语义边界**
   - `setAppState` updater 内对 `prev.sessionHooks` 做 mutate 并 `return prev`，这在当前 store 实现下能正确短路，但**高度依赖 store 的引用相等性优化**。若未来迁移到不可变要求的 store，所有 `return prev` 的假设都会失效，导致更新被静默丢弃。

4. **FunctionHook 的阻塞能力缺乏超时强制中断**
   - `executeFunctionHook` 虽然构造了 `createCombinedAbortSignal(signal, { timeoutMs })`，但 `abortSignal` 只能给异步回调一个信号；若回调是同步死循环，Node 事件循环仍会被卡住，直到 `timeoutMs` 后 Promise 也并不会自动 reject（`AbortSignal` 本身不中断同步代码）。

### 6.2 边界

- **不可持久化**：`FunctionHook` 无法进入 settings.json，也无法在进程重启后恢复；session hooks 整体在进程重启后清空。
- **仅 REPL 上下文可用**：`FunctionHook` 需要 `messages` 数组，因此在非交互式/`-p` 模式下的 `executeHooksOutsideREPL` 中遇到 `function` 类型会直接报错并返回 `non_blocking_error`。
- **受 managed-only 策略限制**：当 `allowManagedHooksOnly` 开启时，session hooks 被完全屏蔽，这意味着 skill/agent frontmatter hooks 也会被禁用。

### 6.3 改进建议

1. **增加会话钩子自动过期/心跳清理机制**
   - 可考虑在 `SessionStore` 中记录 `lastAccessedAt`，由后台 housekeeping 定期扫描并清理长期无活动的 session hooks，降低泄漏风险。

2. **统一移除接口或增加类型级保护**
   - 当前 `removeSessionHook` 的签名接受 `HookCommand`，但传入 `FunctionHook` 会静默失败。建议将 `removeSessionHook` 的签名收紧为 `HookCommand`（排除 `FunctionHook`），或在运行时抛出明确错误。

3. **为 FunctionHook 增加 Worker/VM 隔离执行**
   - 目前 `FunctionHook` 在主线程直接执行用户/代码提供的回调，存在阻塞事件循环和代码注入风险。对于非内部使用的场景，可考虑在 `vm` 或 `worker_threads` 中运行，并通过 `Atomics`/`MessageChannel` 实现硬超时。

4. **补充单元测试覆盖**
   - 当前仓库中未找到针对 `sessionHooks.ts` 的专门测试文件。建议补充：
     - `addSessionHook` / `removeSessionHook` 的 matcher 合并与清理逻辑；
     - `addFunctionHook` / `removeFunctionHook` 的 id 生命周期；
     - `getSessionHooks` 正确过滤 `function` 类型；
     - `clearSessionHooks` 后 `getSessionHooks` 返回空 Map。
