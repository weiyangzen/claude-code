# `src/utils/activityManager.ts` 深度研究文档

> 研究对象：`ActivityManager` 类及其导出实例 `activityManager`  
> 文件路径：`src/utils/activityManager.ts`  
> 关联文件：`src/bootstrap/state.ts`、`src/components/Spinner.tsx`、`src/screens/REPL.tsx`

---

## 1. 场景与职责

`activityManager.ts` 是 Claude Code CLI 内部用于**追踪并区分"用户活跃时间"与"CLI 活跃时间"**的核心工具模块。在终端交互场景中，系统需要回答两个问题：

1. **用户是否在积极操作**（例如打字、提交命令）？
2. **CLI 自身是否正在执行耗时操作**（例如等待模型响应、加载状态）？

这两个问题的答案直接影响到：
- **遥测统计（Telemetry）**：将活跃时间按 `type: 'user'` 或 `type: 'cli'` 归因上报；
- **UI 状态判断**：如 REPL 屏幕中是否展示"Claude 已完成响应且用户处于空闲"的通知；
- **会话生命周期管理**：判断当前会话是否仍有人为交互，避免将后台等待时间误判为用户活跃时间。

`ActivityManager` 采用**单例模式（Singleton）**管理全局状态，确保整个进程内对"活跃状态"的观测是统一且无歧义的。

---

## 2. 功能点目的

### 2.1 双轨时间追踪

模块将时间划分为两条互斥的轨道：

| 轨道 | 触发方式 | 统计标签 | 目的 |
|------|---------|---------|------|
| **User Activity** | `recordUserActivity()` | `type: 'user'` | 记录用户真实交互产生的活跃秒数 |
| **CLI Activity** | `startCLIActivity()` / `endCLIActivity()` | `type: 'cli'` | 记录 CLI 后台执行操作（如 Spinner 动画、模型请求）占用的活跃秒数 |

两条轨道的核心互斥逻辑体现在 `recordUserActivity()` 第 58 行：

```typescript
if (!this.isCLIActive && this.lastUserActivityTime !== 0) {
  // 仅在 CLI 不活跃时才累加用户时间
}
```

这意味着 **CLI 活动具有排他性优先级**——只要 CLI 正在执行操作（`isCLIActive === true`），即使用户在打字，这段时间也不会被计入 `user` 活跃时间，而是会被后续的 CLI 结束逻辑计入 `cli` 活跃时间。这种设计的合理性在于：当 Spinner 转动或模型在思考时，用户本质上处于"等待系统"状态，不应被视为主动工作。

### 2.2 去重与嵌套操作管理

`startCLIActivity(operationId: string)` 支持**相同 `operationId` 的重复调用**。第 83-85 行的去重逻辑如下：

```typescript
if (this.activeOperations.has(operationId)) {
  this.endCLIActivity(operationId)
}
```

当检测到重复的 `operationId` 时，系统会先结束旧操作（结算其已产生的 CLI 时间），再重新开启新操作。这避免了同一操作因 React 重渲染或 useEffect 重复触发而导致的时间重复计算，同时也保证了操作计时的"最近一次"语义。

此外，`activeOperations` 是一个 `Set<string>`，支持**多操作并发嵌套**。只要集合不为空，`isCLIActive` 就保持为 `true`；只有当最后一个操作被移除时，才会触发时间结算并将 `isCLIActive` 置为 `false`。

### 2.3 5 秒用户活动超时窗口

`USER_ACTIVITY_TIMEOUT_MS = 5000`（第 20 行）定义了判断用户是否"当前活跃"的阈值。该超时机制作用于两处：

1. **时间累加上限**：在 `recordUserActivity()` 第 69-71 行，两次用户活动间隔若超过 5 秒，则中间的空档**不计入**用户活跃时间：
   ```typescript
   if (timeSinceLastActivity < timeoutSeconds) {
     activeTimeCounter.add(timeSinceLastActivity, { type: 'user' })
   }
   ```
2. **状态查询**：在 `getActivityStates()` 第 156-157 行，通过比较当前时间与 `lastUserActivityTime` 的差值是否小于 5 秒，返回 `isUserActive` 布尔值：
   ```typescript
   const isUserActive = timeSinceUserActivity < this.USER_ACTIVITY_TIMEOUT_MS / 1000
   ```

该设计基于一个合理假设：如果用户超过 5 秒没有任何输入，则认为其已离开或暂停交互，中间的空闲时间不应被统计为有效工作时间。

### 2.4 测试辅助：单例的可控化

`ActivityManager` 虽然是单例，但提供了两个专门的测试入口：

- **`static resetInstance()`（第 48-50 行）**：将 `instance` 置为 `null`，用于测试间清理全局状态，防止测试用例相互污染。
- **`static createInstance(options?: ActivityManagerOptions)`（第 52-55 行）**：允许测试代码注入自定义的 `getNow` 和 `getActiveTimeCounter`，从而对时间流逝和计数器行为进行精确模拟与断言。

这两个方法的存在，使得单例在运行时保持全局一致性，同时在测试环境中具备完全的可控性。

---

## 3. 具体技术实现（关键流程/数据结构/协议/命令）

### 3.1 核心数据结构

```typescript
private activeOperations = new Set<string>()      // 当前正在运行的 CLI 操作 ID 集合
private lastUserActivityTime: number = 0          // 上次用户活动的绝对时间戳（ms）
private lastCLIRecordedTime: number               // 上次 CLI 计时开始/结算的绝对时间戳（ms）
private isCLIActive: boolean = false              // CLI 是否处于活跃状态（集合非空时为 true）
private static instance: ActivityManager | null = null  // 单例持有器
```

### 3.2 构造函数与依赖注入

```typescript
constructor(options?: ActivityManagerOptions) {
  this.getNow = options?.getNow ?? (() => Date.now())
  this.getActiveTimeCounter = options?.getActiveTimeCounter ?? getActiveTimeCounterImpl
  this.lastCLIRecordedTime = this.getNow()
}
```

- `getNow` 默认使用 `Date.now()`，测试可替换为固定时间戳或虚拟时钟；
- `getActiveTimeCounter` 默认从 `../bootstrap/state.js` 引入，返回一个 `AttributedCounter | null`；
- `lastCLIRecordedTime` 在构造时立即初始化，确保首次 CLI 活动有基准时间。

### 3.3 关键流程：用户活动记录

**方法**：`recordUserActivity()`（第 57-79 行）

1. 检查 `!this.isCLIActive && this.lastUserActivityTime !== 0`：
   - 若 CLI 正在活跃，**直接跳过累加**；
   - 若 `lastUserActivityTime` 为 0（首次记录），也跳过累加，仅设置基准时间。
2. 计算 `timeSinceLastActivity = (now - lastUserActivityTime) / 1000`，单位转换为秒；
3. 若 `timeSinceLastActivity > 0` 且小于 5 秒超时阈值，调用 `activeTimeCounter.add(timeSinceLastActivity, { type: 'user' })`；
4. 无论是否累加，都将 `lastUserActivityTime` 更新为当前时间。

### 3.4 关键流程：CLI 活动开始

**方法**：`startCLIActivity(operationId: string)`（第 81-93 行）

1. 若 `activeOperations.has(operationId)`，先调用 `this.endCLIActivity(operationId)` 结束旧实例（去重逻辑）；
2. 记录 `wasEmpty = this.activeOperations.size === 0`；
3. 将 `operationId` 加入 `activeOperations`；
4. 若 `wasEmpty` 为真（即此前无 CLI 活动），设置 `this.isCLIActive = true` 并将 `lastCLIRecordedTime` 更新为当前时间。

### 3.5 关键流程：CLI 活动结束

**方法**：`endCLIActivity(operationId: string)`（第 95-113 行）

1. 从 `activeOperations` 中删除 `operationId`；
2. 若集合变空，则：
   - 计算 `timeSinceLastRecord = (now - lastCLIRecordedTime) / 1000`；
   - 若大于 0，调用 `activeTimeCounter.add(timeSinceLastRecord, { type: 'cli' })`；
   - 更新 `lastCLIRecordedTime = now`；
   - 设置 `this.isCLIActive = false`。

### 3.6 辅助方法：自动包裹异步操作

**方法**：`trackOperation<T>(operationId: string, fn: () => Promise<T>): Promise<T>`（第 115-122 行）

```typescript
this.startCLIActivity(operationId)
try {
  return await fn()
} finally {
  this.endCLIActivity(operationId)
}
```

该方法主要用于测试与调试场景，确保异步函数在执行期间被完整计入 CLI 活跃时间，且无论成功或异常都会正确结束计时。

### 3.7 状态查询

**方法**：`getActivityStates()`（第 124-162 行）

返回一个包含三个字段的对象：
- `isUserActive`：基于 5 秒超时计算；
- `isCLIActive`：直接返回内部状态；
- `activeOperationCount`：返回 `activeOperations.size`。

---

## 4. 关键代码路径与文件引用

### 4.1 模块自身

| 行号 | 代码元素 | 说明 |
|------|---------|------|
| 1 | `import { getActiveTimeCounter as getActiveTimeCounterImpl } from '../bootstrap/state.js'` | 遥测计数器来源 |
| 7-11 | `ActivityManagerOptions` | 构造函数可选参数类型 |
| 13-23 | 私有字段声明 | 核心状态存储 |
| 25-34 | `constructor` | 依赖注入与初始化 |
| 36-46 | `getInstance()` | 标准单例获取 |
| 48-50 | `resetInstance()` | 测试清理入口 |
| 52-55 | `createInstance(options?)` | 测试构造入口 |
| 57-79 | `recordUserActivity()` | 用户活动时间累加 |
| 81-93 | `startCLIActivity(operationId)` | CLI 活动开始，含去重 |
| 95-113 | `endCLIActivity(operationId)` | CLI 活动结束，含时间结算 |
| 115-122 | `trackOperation(operationId, fn)` | 异步操作自动追踪 |
| 124-162 | `getActivityStates()` | 综合状态查询 |
| 164 | `export const activityManager = ActivityManager.getInstance()` | 全局单例导出 |

### 4.2 上游调用方

#### `src/components/Spinner.tsx`

- **第 15 行**：`import { activityManager } from '../utils/activityManager.js'`
- **第 174-180 行**：在 `useEffect` 中，以 `operationId = 'spinner-' + mode` 调用 `activityManager.startCLIActivity(operationId)`，并在组件卸载时调用 `activityManager.endCLIActivity(operationId)`。
- **第 329-336 行**（编译后代码路径）：存在另一处等价的 `useEffect` 调用，同样用于追踪 Spinner 的 CLI 活跃状态。

**作用**：当终端显示加载动画（Spinner）时，将这段时间标记为 `cli` 活跃时间，避免用户在此期间被误统计为活跃。

#### `src/screens/REPL.tsx`

- **第 225 行**：`import { activityManager } from '../utils/activityManager.js'`
- **第 3899-3902 行**：
  ```typescript
  useEffect(() => {
    activityManager.recordUserActivity()
    updateLastInteractionTime(true)
  }, [inputValue, submitCount])
  ```

**作用**：每当用户的 `inputValue` 或 `submitCount` 发生变化时，记录一次用户活动。这是用户活跃时间统计的主要触发源。

### 4.3 下游依赖

#### `src/bootstrap/state.ts`

- **第 41-43 行**：`AttributedCounter` 类型定义：
  ```typescript
  export type AttributedCounter = {
    add(value: number, additionalAttributes?: Attributes): void
  }
  ```
- **第 1021-1023 行**：`getActiveTimeCounter()` 实现：
  ```typescript
  export function getActiveTimeCounter(): AttributedCounter | null {
    return STATE.activeTimeCounter
  }
  ```

`STATE.activeTimeCounter` 在遥测系统初始化时由 `setMeter()` 创建并挂载，通常对应一个 OpenTelemetry 风格的计数器实例。

---

## 5. 依赖与外部交互

### 5.1 直接依赖

| 依赖 | 路径 | 用途 |
|------|------|------|
| `getActiveTimeCounter` | `../bootstrap/state.js` | 获取全局遥测计数器，用于实际时间数据上报 |
| `Date.now()` | 原生 API（可注入覆盖） | 获取当前时间戳 |

### 5.2 运行时交互模型

```
REPL.tsx (用户输入)
    │
    ▼
activityManager.recordUserActivity() ──► 计算间隔 ──► activeTimeCounter.add(..., { type: 'user' })
    │
Spinner.tsx (加载动画)
    │
    ▼
activityManager.startCLIActivity('spinner-' + mode)
    │
    ▼
activityManager.endCLIActivity('spinner-' + mode) ──► 计算间隔 ──► activeTimeCounter.add(..., { type: 'cli' })
```

### 5.3 单例生命周期

- **初始化**：模块加载时（第 164 行）立即执行 `ActivityManager.getInstance()`，创建默认实例；
- **运行时**：全局共享，所有调用方通过 `activityManager` 常量访问；
- **测试时**：通过 `ActivityManager.resetInstance()` 或 `ActivityManager.createInstance({ getNow: ..., getActiveTimeCounter: ... })` 替换实例，实现隔离与模拟。

---

## 6. 风险、边界与改进建议

### 6.1 当前风险

#### 风险 1：时间单位不一致的潜在混淆

代码中 `getNow()` 返回毫秒（`Date.now()`），而 `USER_ACTIVITY_TIMEOUT_MS` 也是毫秒；但在比较时，代码将其除以 1000 转换为秒：

```typescript
const timeSinceLastActivity = (now - this.lastUserActivityTime) / 1000
// ...
if (timeSinceLastActivity < timeoutSeconds)   // timeoutSeconds = 5
```

这里存在**单位混用**的隐患：虽然数学上 `(msDiff / 1000) < 5` 等价于 `msDiff < 5000`，但所有内部时间变量（如 `lastUserActivityTime`、`lastCLIRecordedTime`）存储的是毫秒，而传递给 `activeTimeCounter.add()` 的却是秒。这种不一致增加了维护者的心智负担，容易在后续修改中引入单位错误。

#### 风险 2：`recordUserActivity` 在 CLI 活跃期间的静默丢弃

当 `isCLIActive === true` 时，`recordUserActivity()` 不会累加任何时间，也不会给出任何提示。这在绝大多数情况下是正确的，但如果未来出现"用户边输入边等待"的场景（例如流式响应中用户继续打字），这些输入时间将完全丢失，既不计入 `user` 也不计入 `cli`。

#### 风险 3：单例在测试中的全局副作用

`createInstance()` 和 `resetInstance()` 虽然是测试友好设计，但如果生产代码中意外调用，会导致全局状态被替换或清空，影响后续所有依赖 `activityManager` 的模块。这两个方法目前没有任何运行时使用限制。

#### 风险 4：重复 `operationId` 的去重逻辑存在时间截断

`startCLIActivity` 在检测到重复 `operationId` 时，会先 `endCLIActivity` 再重新开始。这意味着：

```
t0: start("A")
t1: start("A")  // 触发 end("A")，结算 [t0, t1]，然后重新开始
```

如果调用方的意图是"刷新"或"延续"同一操作，那么 `t1` 到下一次 `end` 之间的时间会被正确记录，但 `[t0, t1]` 被强制切分。这在 React `useEffect` 的严格模式（Strict Mode）下可能导致时间被不必要地碎片化。

### 6.2 边界情况

| 场景 | 行为 | 代码位置 |
|------|------|---------|
| 首次调用 `recordUserActivity()` | 仅设置 `lastUserActivityTime`，不累加 | 第 58 行 `!== 0` 判断 |
| `endCLIActivity` 传入未存在的 `operationId` | `Set.delete` 返回 false，无其他副作用，若集合已为空则什么都不做 | 第 97 行 |
| `timeSinceLastActivity` 或 `timeSinceLastRecord` 为 0 或负数 | 跳过 `activeTimeCounter.add` | 第 64 行、第 106 行 `> 0` 判断 |
| `getActiveTimeCounter()` 返回 `null` | 跳过所有遥测上报，但状态更新正常进行 | 第 67 行、第 108 行 |
| 并发多个 CLI 操作 | 仅第一个操作触发 `isCLIActive = true`；仅最后一个操作触发结算 | 第 88-92 行、第 99-112 行 |

### 6.3 改进建议

#### 建议 1：统一时间单位

将所有内部时间存储统一为秒，或统一为毫秒，并在传递给 `activeTimeCounter.add()` 时进行一次性转换。例如：

```typescript
private readonly USER_ACTIVITY_TIMEOUT_S = 5
// 所有内部计算直接使用秒，避免到处 /1000
```

#### 建议 2：为 `resetInstance` / `createInstance` 添加环境 guard

```typescript
static resetInstance(): void {
  if (process.env.NODE_ENV !== 'test') {
    console.warn('resetInstance should only be used in tests')
  }
  ActivityManager.instance = null
}
```

这可以在不破坏现有测试的前提下，防止生产代码误用。

#### 建议 3：考虑引入"用户活动被 CLI 抑制"的显式记录

如果未来需要更精细的归因分析，可以在 `recordUserActivity()` 被 CLI 抑制时，将这段时间以 `type: 'cli'` 或 `type: 'waiting'` 的形式追加到 CLI 时间中，而不是直接丢弃。当前实现中这段时间是"黑洞"状态。

#### 建议 4：评估去重逻辑是否应改为 no-op

对于重复 `operationId` 的处理，可以考虑：

```typescript
if (this.activeOperations.has(operationId)) {
  return // 直接忽略重复 start
}
```

而不是先 `end` 再 `start`。这样可以避免 Strict Mode 下的时间碎片化。不过需要确认调用方（如 `Spinner.tsx` 的 `useEffect`）在重复触发时是否确实期望刷新计时。若期望刷新，则当前逻辑是合理的。

#### 建议 5：补充单元测试覆盖以下场景

- `recordUserActivity` 在 `isCLIActive` 期间的静默行为；
- 重复 `operationId` 触发去重后的时间分段准确性；
- `getActivityStates` 在 5 秒边界上的精确判断（刚好 4.999 秒 vs 5.001 秒）；
- `trackOperation` 在 `fn` 抛出异常时是否仍能正确 `endCLIActivity`。

---

## 总结

`activityManager.ts` 是一个职责单一但设计精巧的模块。它通过单例 `ActivityManager` 协调了用户输入（`REPL.tsx`）与 CLI 后台操作（`Spinner.tsx`）之间的时间归因冲突，利用 `Set<string>` 实现了多操作嵌套管理，并通过 5 秒超时窗口过滤掉无意义的空闲时间。其构造函数的可注入设计（`getNow`、`getActiveTimeCounter`）配合 `resetInstance` / `createInstance` 测试辅助方法，在保持运行时全局一致性的同时，为测试提供了良好的可控性。主要改进空间在于时间单位的统一、生产环境对测试辅助方法的防护，以及对"CLI 活跃期间用户活动被丢弃"这一行为的进一步显式化或文档化。
