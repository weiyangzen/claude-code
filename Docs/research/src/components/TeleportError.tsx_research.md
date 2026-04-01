# TeleportError.tsx 研究文档

## 场景与职责

`TeleportError.tsx` 是 Claude Code **Teleport（远程会话恢复/创建）** 功能的前置校验与错误引导组件。Teleport 允许用户从另一台机器或云端恢复 Claude Code 会话，或创建远程会话。由于该功能要求：

1. 用户已登录 Claude.ai 账号（`needsLogin`）
2. 当前 Git 工作目录干净，无未提交更改（`needsGitStash`）

`TeleportError` 负责在 UI 层检测这两个前置条件，并在条件不满足时拦截正常流程，向用户展示可交互的修复对话框：

- **需要登录** → 引导用户通过 `ConsoleOAuthFlow` 完成 Claude.ai OAuth 登录。
- **Git 不干净** → 引导用户通过 `TeleportStash` 自动 stash 变更后继续。

## 功能点目的

1. **前置条件统一校验**：将 `checkNeedsClaudeAiLogin()` 和 `checkIsGitClean()` 的异步检查封装为 `getTeleportErrors()`，返回需要处理的错误集合。
2. **可忽略错误配置**：支持 `errorsToIgnore` prop，允许调用方跳过特定错误类型（例如某些流程可忽略 `needsGitStash`）。
3. **状态机驱动的错误展示**：组件内部维护 `currentError` 状态，按优先级依次处理 `needsLogin` → `needsGitStash`，每次只展示一种错误界面。
4. **登录流程嵌入**：直接复用 `ConsoleOAuthFlow` 组件，以 `mode="login" forceLoginMethod="claudeai"` 强制走 Claude.ai 订阅账号登录路径。
5. **Stash 流程嵌入**：直接复用 `TeleportStash` 组件，自动执行 `git stash` 并在成功后回调继续。
6. **稳定默认参数**：使用模块级常量 `EMPTY_ERRORS_TO_IGNORE` 避免每次渲染创建新的 `Set` 对象，防止 `useEffect` 依赖变化导致的重复触发。

## 具体技术实现

### 关键流程

#### 1. 错误检测（`getTeleportErrors`）
```ts
export async function getTeleportErrors(): Promise<Set<TeleportLocalErrorType>> {
  const errors = new Set<TeleportLocalErrorType>();
  const [needsLogin, isGitClean] = await Promise.all([
    checkNeedsClaudeAiLogin(),
    checkIsGitClean(),
  ]);
  if (needsLogin) errors.add('needsLogin');
  if (!isGitClean) errors.add('needsGitStash');
  return errors;
}
```
- 两个检查并行执行。
- `checkNeedsClaudeAiLogin`：若用户是 Claude.ai 订阅者，则尝试刷新/校验 OAuth token。
- `checkIsGitClean`：调用 `getIsClean({ ignoreUntracked: true })`，忽略未跟踪文件（因为它们不会因分支切换而丢失）。

#### 2. 组件状态机
- `currentError: TeleportLocalErrorType | null` — 当前展示的错误类型。
- `isLoggingIn: boolean` — 是否已进入登录流程（`ConsoleOAuthFlow`）。
- `checkErrors`：异步回调，先过滤 `errorsToIgnore`，然后按优先级设置 `currentError`；若全部过滤完则调用 `onComplete()`。
- `useEffect`：在 `checkErrors` 变化时触发检测（即组件挂载和依赖变化时）。

#### 3. 交互处理
- **Stash 完成** → 重新调用 `checkErrors`。
- **登录完成** → 退出登录 UI，重新调用 `checkErrors`。
- **登录选择** → 用户选择 "Login with Claude account" 进入 `ConsoleOAuthFlow`；选择 "Exit" 则调用 `onCancel`。
- **onCancel** → 调用 `gracefulShutdownSync(0)`，**直接退出整个应用**。

#### 4. 渲染分支
- `currentError === null` → 返回 `null`（不渲染）。
- `currentError === 'needsGitStash'` → 渲染 `<TeleportStash onStashAndContinue={handleStashComplete} onCancel={onCancel} />`。
- `currentError === 'needsLogin'`：
  - `isLoggingIn === true` → 渲染 `<ConsoleOAuthFlow onDone={handleLoginComplete} mode="login" forceLoginMethod="claudeai" />`。
  - `isLoggingIn === false` → 渲染 `Dialog` + `Select`，提供 "Login with Claude account" / "Exit" 两个选项。

### 数据结构

- `TeleportLocalErrorType = 'needsLogin' | 'needsGitStash'`
- `TeleportErrorProps`：
  - `onComplete: () => void` — 所有错误解决后的回调。
  - `errorsToIgnore?: ReadonlySet<TeleportLocalErrorType>` — 可忽略的错误集合。

### 协议/命令

- **OAuth 校验**：`src/utils/background/remote/preconditions.ts` → `checkNeedsClaudeAiLogin()` 内部调用 `checkAndRefreshOAuthTokenIfNeeded()`，可能触发 OAuth token 刷新网络请求。
- **Git 状态检查**：`src/utils/background/remote/preconditions.ts` → `checkIsGitClean()` 内部调用 `getIsClean({ ignoreUntracked: true })`，执行 `git status` 等子进程命令（定义在 `src/utils/git.ts`）。
- **Stash 操作**：`TeleportStash` 组件内部调用 `stashToCleanState('Teleport auto-stash')`，执行 `git stash` 子进程命令。

## 关键代码路径与文件引用

- **本文件**：`src/components/TeleportError.tsx`
- **调用方**：
  - `src/components/ResumeTask.tsx` — 在加载远程可恢复会话列表前，先渲染 `TeleportError` 完成前置校验。
  - `src/components/tasks/RemoteSessionDetailDialog.tsx` — 远程会话详情弹窗中可能使用。
  - `src/utils/teleport.tsx` — `handleTeleportPrerequisites()` 函数通过 `root.render(<TeleportError ... />)` 以程序化方式渲染该组件，处理 teleport 前置条件。
- **依赖类型与工具**：
  - `src/utils/background/remote/preconditions.ts` — `checkIsGitClean`、`checkNeedsClaudeAiLogin`。
  - `src/utils/gracefulShutdown.ts` — `gracefulShutdownSync`。
  - `src/components/ConsoleOAuthFlow.tsx` — OAuth 登录流程 UI。
  - `src/components/CustomSelect/index.ts` → `Select` 组件。
  - `src/components/design-system/Dialog.tsx` — 对话框容器。
  - `src/components/TeleportStash.tsx` — Git stash 引导 UI。
  - `src/ink.js` — `Box`、`Text`。

## 依赖与外部交互

| 依赖 | 路径 | 用途 |
|------|------|------|
| `checkNeedsClaudeAiLogin` | `src/utils/background/remote/preconditions.ts` | 校验是否需要 Claude.ai 登录 |
| `checkIsGitClean` | `src/utils/background/remote/preconditions.ts` | 校验 Git 工作目录是否干净 |
| `gracefulShutdownSync` | `src/utils/gracefulShutdown.ts` | 用户选择 Exit 时同步触发优雅退出 |
| `ConsoleOAuthFlow` | `src/components/ConsoleOAuthFlow.tsx` | 登录流程 UI |
| `TeleportStash` | `src/components/TeleportStash.tsx` | Git stash 流程 UI |
| `Dialog` | `src/components/design-system/Dialog.tsx` | 登录选择对话框容器 |
| `Select` | `src/components/CustomSelect/index.ts` | 登录/退出选项选择器 |
| `Box`, `Text` | `src/ink.js` | Ink 布局与文本渲染 |

## 风险、边界与改进建议

### 风险与边界

1. **`onCancel` 直接退出应用**：`_temp()` 函数（绑定到 `onCancel`）调用 `gracefulShutdownSync(0)`。这意味着用户在 Teleport 错误对话框中按 Esc 或选择 Exit 时，**整个 Claude Code 进程会退出**，而不是返回上一级菜单。对于误触的用户来说体验较激进。
2. **`checkErrors` 的 `useEffect` 依赖设计**：`useEffect` 在 `checkErrors` 变化时触发，而 `checkErrors` 是一个在渲染过程中创建的异步函数（被 React Compiler 缓存）。虽然缓存机制保证了稳定性，但若 `onComplete` 或 `errorsToIgnore` 引用不稳定，仍可能导致重复检测。
3. **错误优先级硬编码**：`needsLogin` 始终优先于 `needsGitStash`。若用户同时需要登录和 stash，必须先完成登录才能看到 stash 提示。这种串行处理增加了用户操作步骤。
4. **`EMPTY_ERRORS_TO_IGNORE` 的 mutable 风险**：虽然声明为 `ReadonlySet`，但底层仍是普通 `Set` 实例。若某处代码错误地进行了类型断言并修改了它，会影响所有使用默认参数的调用方。
5. **无重试/刷新机制**：除了 stash 和登录各自的子流程外，组件本身没有提供 "重新检测" 按钮。若网络瞬断导致 `checkNeedsClaudeAiLogin` 误判，用户只能退出重进。

### 改进建议

1. **缓和取消行为**：将 `onCancel` 的默认行为从直接退出改为调用一个可选的 `onCancel` prop（由调用方决定是退出还是返回）。`ResumeTask.tsx` 中可以选择返回普通 teleport 流程，`utils/teleport.tsx` 中可以选择退出。
2. **并行提示优化**：评估是否可以在 UI 上同时展示多个前置条件（例如一个对话框内同时提示 "需要登录" 和 "需要 stash"），减少用户操作轮次。
3. **增加显式重试按钮**：在错误检测失败（尤其是网络相关）时，提供一个 "Check again" 选项，避免用户被迫退出。
4. **冻结 `EMPTY_ERRORS_TO_IGNORE`**：使用 `Object.freeze(new Set())` 进一步防止意外修改。
5. **添加单元测试**：
   - 测试 `getTeleportErrors` 在四种组合（都满足/仅登录/仅 stash/都不满足）下的返回。
   - 测试 `TeleportError` 组件的状态转换：mount → 检测 → 渲染 Dialog → 选择登录 → 渲染 ConsoleOAuthFlow → onDone → 重新检测 → onComplete。
   - 测试 `errorsToIgnore` 的过滤逻辑。
