# 研究文档：src/moreright/useMoreRight.tsx

> 研究范围：`src/moreright/useMoreRight.tsx` 及其在代码库中的调用上下文、构建集成与运行时行为。
> 研究时间：2026-04-01
> 执行器：kimi (k2p5)

---

## 1. 场景与职责

`src/moreright/useMoreRight.tsx` 是一个 **React Hook 的公共接口占位符（stub）**，用于在 REPL（主交互界面）中集成一项名为 **"MoreRight"** 的内部功能。

### 关键场景特征

- **外部构建（external builds）专用**：当前仓库中的实现是一个无操作（no-op）stub。文件顶部注释明确说明 "the real hook is internal only"。
- **构建时条件编译**：该功能仅在内部（`ant`）构建中可能启用，通过 `"external" === 'ant'` 这一构建时常量进行死代码消除（dead code elimination）。
- **环境变量门控**：即使在内部分支中，也需要显式设置 `CLAUDE_MORERIGHT` 环境变量才能激活。
- **类型检查兼容性**：注释提到类型检查器会在 `scripts/external-stubs/src/moreright/` 路径下看到此文件（overlay 机制），因此文件必须完全自包含，不能使用任何相对路径导入（如 `../types/`）。

### 目录结构

```
src/moreright/
└── useMoreRight.tsx    # 唯一文件，stub 实现
```

---

## 2. 功能点目的

`useMoreRight` 的设计目的是在 REPL 的每一轮对话（turn）生命周期中插入三个扩展点：

1. **`onBeforeQuery`**：在用户消息已追加到对话历史、但尚未发起 API 调用之前执行。返回 `Promise<boolean>`，可用于拦截或修改本轮查询。
2. **`onTurnComplete`**：在模型响应结束（无论成功、失败还是被用户取消）后执行。返回 `Promise<void>`，可用于清理状态或触发后续动作。
3. **`render`**：返回一个 React 节点，用于在 REPL 的 UI 中渲染 MoreRight 相关的覆盖层或组件。

### 当前 stub 实现行为

在 **当前 stub 实现** 中，这三个扩展点均为空操作：
- `onBeforeQuery` 始终返回 `true`（不拦截）
- `onTurnComplete` 为空异步函数
- `render` 始终返回 `null`（不渲染任何内容）

---

## 3. 具体技术实现

### 3.1 接口契约

```typescript
// src/moreright/useMoreRight.tsx (L8-24)
// eslint-disable-next-line @typescript-eslint/no-explicit-any
type M = any;

export function useMoreRight(_args: {
  enabled: boolean;
  setMessages: (action: M[] | ((prev: M[]) => M[])) => void;
  inputValue: string;
  setInputValue: (s: string) => void;
  setToolJSX: (args: M) => void;
}): {
  onBeforeQuery: (input: string, all: M[], n: number) => Promise<boolean>;
  onTurnComplete: (all: M[], aborted: boolean) => Promise<void>;
  render: () => null;
}
```

**关键设计决策：**
- 类型参数 `M` 被别名为 `any`，这是刻意为之：因为文件不能依赖 `../types/message.js`，否则在 `scripts/external-stubs/` overlay 路径下类型检查会失败。
- `_args` 中的 `enabled` 由调用方（REPL）根据构建目标和环境变量计算得出。
- 使用 `_args` 命名约定表示该参数在 stub 中未被使用，但为保持接口兼容性而保留。

### 3.2 REPL 中的集成点

调用方位于 `src/screens/REPL.tsx`：

#### 导入（L68）
```tsx
import { useMoreRight } from '../moreright/useMoreRight.js';
```

#### 启用条件（L604）
```tsx
const moreRightEnabled = useMemo(
  () => "external" === 'ant' && isEnvTruthy(process.env.CLAUDE_MORERIGHT),
  []
);
```
- `"external" === 'ant'` 是一个 **构建时求值的常量**。在外部构建中，该表达式被编译为 `false`，因此 `moreRightEnabled` 恒为 `false`。
- 尽管如此，Hook 仍然被无条件调用（遵守 React Hooks 的调用顺序规则）。

#### Hook 调用与解构（L1661-1671）
```tsx
const {
  onBeforeQuery: mrOnBeforeQuery,
  onTurnComplete: mrOnTurnComplete,
  render: mrRender
} = useMoreRight({
  enabled: moreRightEnabled,
  setMessages,
  inputValue,
  setInputValue,
  setToolJSX
});
```

#### 生命周期回调注入

**1. 查询前拦截（L2907-2909）：**
```tsx
const latestMessages = messagesRef.current;
if (input) {
  await mrOnBeforeQuery(input, latestMessages, newMessages.length);
}
```
发生在 `setMessages` 已同步追加新消息之后、`onBeforeQueryCallback`（REPL 的 props 回调）之前。

**2. 查询完成清理（L2929-2930）：**
```tsx
resetLoadingState();
await mrOnTurnComplete(messagesRef.current, abortController.signal.aborted);
```
位于 `onQuery` 的 `finally` 块中，确保无论 `onQueryImpl` 是否抛出异常都会执行。

**3. 用户强制取消（L2161-2162）：**
```tsx
// forceEnd() skips the finally path — fire directly (aborted=true).
void mrOnTurnComplete(messagesRef.current, true);
```
当用户按 Escape 取消且 `queryGuard.forceEnd()` 跳过了 `finally` 路径时，直接触发 `mrOnTurnComplete`。

#### UI 渲染注入（L4892）
```tsx
{mrRender()}
```
渲染位置位于 `toolJSX` 覆盖层之后、`PromptInput` 及各类 survey/notification 组件之前。这意味着真实的 `render()` 若返回非 `null` 内容，将作为一个中间层插入到 REPL 的主内容区与输入区之间。

### 3.3 构建系统与条件编译

- 项目使用 `bun:bundle` 提供的 `feature()` 函数进行特性开关编译（如 `feature('VOICE_MODE')`、`feature('KAIROS')` 等）。
- `useMoreRight` 未使用 `feature()`，而是使用字符串常量比较 `"external" === 'ant'`。这是项目中常见的另一种构建时条件编译模式，Bun 打包器会在编译期将其求值为布尔常量，并消除不可达分支。
- 文件注释提到的 `scripts/external-stubs/src/moreright/` 路径在当前仓库中**不存在**。这表明该 overlay 逻辑存在于内部 CI/CD 或构建脚本中，用于在对外发布（external release）前将真实的内部实现替换为此 stub。

### 3.4 依赖工具函数

#### isEnvTruthy
位于 `src/utils/envUtils.ts`（L32-37）：
```typescript
export function isEnvTruthy(envVar: string | boolean | undefined): boolean {
  if (!envVar) return false
  if (typeof envVar === 'boolean') return envVar
  const normalizedValue = envVar.toLowerCase().trim()
  return ['1', 'true', 'yes', 'on'].includes(normalizedValue)
}
```
用于解析 `CLAUDE_MORERIGHT` 环境变量，支持 `1`、`true`、`yes`、`on` 等真值。

#### setToolJSX
位于 `src/screens/REPL.tsx`（L1060-1100）：
这是一个包装函数，用于管理工具级 JSX 覆盖层的显示。它处理本地 JSX 命令（如 `/btw`）与工具 JSX 之间的优先级关系。

---

## 4. 关键代码路径与文件引用

| 文件路径 | 角色 |
|---------|------|
| `src/moreright/useMoreRight.tsx` | **目标文件**。外部构建 stub，定义 `useMoreRight` 接口与 noop 实现。 |
| `src/screens/REPL.tsx` | **唯一调用方**。主 REPL 组件，负责实例化 hook、注入生命周期回调、渲染 `mrRender()`。 |
| `src/utils/envUtils.ts` | **间接依赖**。提供 `isEnvTruthy()`，用于解析 `CLAUDE_MORERIGHT` 环境变量。 |

### REPL.tsx 中的关键行号

| 行号 | 内容 |
|------|------|
| L68 | `import { useMoreRight } from '../moreright/useMoreRight.js'` |
| L604 | `moreRightEnabled` 状态计算 |
| L1665-1671 | `useMoreRight` Hook 调用 |
| L2908 | `mrOnBeforeQuery` 调用点（查询前） |
| L2162 | `mrOnTurnComplete` 调用点（强制取消路径） |
| L2930 | `mrOnTurnComplete` 调用点（正常 finally 路径） |
| L4892 | `mrRender()` 渲染点 |

---

## 5. 依赖与外部交互

### 5.1 传入依赖（由 REPL 注入）

| 依赖名 | 类型 | 说明 |
|--------|------|------|
| `enabled` | `boolean` | 功能总开关，由构建目标 + 环境变量共同决定。 |
| `setMessages` | `React.Dispatch<React.SetStateAction<MessageType[]>>` | 对话消息列表的 setter，允许 MoreRight 在查询前后修改消息历史。 |
| `inputValue` / `setInputValue` | `string` / `(s: string) => void` | 当前输入框内容与 setter，允许 MoreRight 读取或清空用户输入。 |
| `setToolJSX` | `(args: { jsx, shouldHidePromptInput, ... }) => void` | REPL 中用于显示工具级 JSX 覆盖层的 API。MoreRight 理论上可以借此显示自定义 UI。 |

### 5.2 外部交互

- **环境变量**：`process.env.CLAUDE_MORERIGHT`
- **构建系统**：`bun:bundle`（通过 `"external" === 'ant'` 进行死代码消除）
- **内部实现**：当前仓库中不存在。真实实现通过构建时的文件 overlay 替换此 stub。

---

## 6. 风险、边界与改进建议

### 6.1 风险

#### 1. 类型安全性薄弱
由于文件必须自包含（无相对导入），`useMoreRight` 使用 `type M = any` 替代了具体的 `MessageType`。这意味着 stub 与真实实现之间的类型契约缺乏编译期校验，若真实实现的签名发生变更，stub 不会报错，可能导致外部构建的类型检查通过但运行时行为不一致。

#### 2. React Hooks 规则依赖
REPL 中 `useMoreRight` 是无条件调用的（即使 `moreRightEnabled` 为 `false`）。这是正确的做法，符合 React Hooks 规则。但如果未来有开发者试图将其改为条件调用（如 `if (moreRightEnabled) { useMoreRight(...) }`），会导致 Hook 顺序错误和难以调试的崩溃。

#### 3. 生命周期回调中的异常处理缺失
`mrOnBeforeQuery` 和 `mrOnTurnComplete` 在 REPL 中通过 `await` 调用，但没有被包裹在独立的 `try/catch` 中（`onBeforeQuery` 在 `try` 块内，但 `mrOnBeforeQuery` 本身没有单独的错误隔离）。如果真实实现抛出异常，可能会导致：
- `onBeforeQuery` 阶段：直接中断查询流程，用户消息已追加但 API 未调用。
- `onTurnComplete` 阶段：阻塞 `finally` 块中的后续清理（如 `sendBridgeResultRef.current()`）。

#### 4. 构建时 overlay 的隐式耦合
`scripts/external-stubs/` 路径在当前仓库中不存在，说明构建逻辑分散在外部 CI 脚本中。这增加了维护成本：修改 `useMoreRight` 的接口时，需要同步更新内部实现、stub 文件以及外部构建脚本，但仓库内没有任何配置可以提醒开发者这一点。

### 6.2 边界情况

- **外部构建中 `enabled` 恒为 `false`**：由于 `"external" === 'ant'` 在发布构建中求值为 `false`，外部用户即使设置了 `CLAUDE_MORERIGHT=1`，该功能也不会被激活。
- **取消路径的双重 `onTurnComplete` 调用风险**：在强制取消时（L2162），`mrOnTurnComplete` 被显式调用一次；如果取消同时触发了 `finally` 块（某些竞态条件下），理论上存在重复调用的风险。不过 `queryGuard.forceEnd()` 的设计意图正是跳过 `finally`，实际中应不会重复触发。
- **`render()` 返回 `null` 的稳定性**：stub 的 `render` 返回 `null`，在 React 中这是一个稳定的叶子节点，不会引起额外的重渲染。

### 6.3 改进建议

#### 1. 增加类型契约的显式校验
在 stub 文件顶部添加 JSDoc 或类型断言，将 `M` 映射到 `MessageType` 的形状（即使通过 `satisfies` 或运行时断言），以便在接口变更时尽早发现不一致。

#### 2. 为生命周期回调添加错误隔离
建议在 REPL 中用 `try/catch` 包裹 `mrOnBeforeQuery` 和 `mrOnTurnComplete` 的调用，并记录错误日志，防止 MoreRight 的内部 bug 破坏整个 REPL 的查询生命周期。

```tsx
// 示例改进
try {
  await mrOnBeforeQuery(input, latestMessages, newMessages.length);
} catch (e) {
  logError('MoreRight onBeforeQuery failed', e);
}
```

#### 3. 将 overlay 构建逻辑文档化或脚本化到仓库内
即使 `scripts/external-stubs/` 目录为空，也建议在仓库根目录保留一个 `scripts/external-stubs/README.md` 或 `build-overlay.md`，说明哪些文件会被外部构建流程替换，以及替换的触发条件。

#### 4. 考虑使用 `feature()` 统一特性开关
项目中大量功能使用 `bun:bundle` 的 `feature('XXX')` 进行条件编译，而 `useMoreRight` 使用的是字符串常量比较。建议统一为 `feature('MORERIGHT')`，以便与现有的特性标志管理体系（如 GrowthBook 或构建配置）保持一致。

#### 5. 暴露 `enabled` 给调试工具
在 REPL 的 DevBar 或 `/status` 命令输出中增加 `moreRightEnabled` 的状态显示，便于内部开发者和支持团队快速判断该功能是否处于激活状态。

---

*文档结束*
