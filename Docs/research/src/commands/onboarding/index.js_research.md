# 研究文档：src/commands/onboarding/index.js

> 研究范围：代码、脚本、配置、测试及必要实现上下文  
> 目标文件：`src/commands/onboarding/index.js`  
> 研究时间：2026-04-01

---

## 1. 场景与职责

`src/commands/onboarding/index.js` 在仓库中是一个**被禁用的占位符（stub）命令**。它本身不承载任何可执行的 onboarding 业务逻辑，而是作为命令注册表中的一个"保留位"存在。真正的用户首次引导（onboarding）流程由 `src/components/Onboarding.tsx` 组件在应用启动阶段通过 `showSetupScreens()` 驱动，属于**启动时自动弹出的 UI 流程**，而非用户主动输入 `/onboarding` 触发的 slash command。

该文件的核心职责可归纳为：
1. **命令占位**：在 `src/commands.ts` 的命令注册表中保留 `onboarding` 这个名字，防止被其他动态技能或插件意外占用。
2. **访问控制**：通过 `isEnabled: () => false` 和 `isHidden: true` 确保该命令在任何场景下都不会被用户（包括内部员工）调用或出现在帮助/补全列表中。
3. **内部构建隔离**：通过被归入 `INTERNAL_ONLY_COMMANDS`，仅在 `USER_TYPE === 'ant'` 的内部构建中才会被加载到命令列表，但即便如此也保持禁用状态。

---

## 2. 功能点目的

### 2.1 为什么需要一个"空"的 onboarding 命令文件？

在 Claude Code 的架构中，所有内置命令都必须在 `src/commands.ts` 中被静态导入并注册到 `COMMANDS()` 或 `INTERNAL_ONLY_COMMANDS` 中。保留 `src/commands/onboarding/index.js` 这个 stub 的目的包括：

- **名称空间保护**：避免第三方插件或本地技能目录中出现名为 `onboarding` 的命令时产生命名冲突。
- **未来扩展性**：如果产品决策改变，允许用户通过 `/onboarding` 重新进入引导流程，只需替换该 stub 为真正的 `local-jsx` 或 `prompt` 类型命令实现，无需改动命令注册基础设施。
- **构建一致性**：内部构建的 `INTERNAL_ONLY_COMMANDS` 数组需要显式引用所有内部命令；`onboarding` 作为历史遗留或规划中的内部命令，以 stub 形式满足数组类型检查。

### 2.2 真正的 onboarding 功能在哪里？

用户首次启动 Claude Code 时经历的引导流程（主题选择、OAuth 登录、安全提示、终端设置等）由以下模块协同完成：

| 功能 | 负责模块 |
|------|----------|
| 引导 UI 渲染 | `src/components/Onboarding.tsx` |
| 启动时判断是否显示引导 | `src/interactiveHelpers.tsx` 中的 `showSetupScreens()` |
| 标记引导完成 | `src/interactiveHelpers.tsx` 中的 `completeOnboarding()` |
| 引导状态持久化 | `src/utils/config.ts` 中的 `GlobalConfig.hasCompletedOnboarding` |
| 登录后刷新授权服务 | `src/main.tsx`（启动路径）和 `src/commands/login/login.tsx`（/login 路径） |
| 项目级引导（workspace/claudemd） | `src/projectOnboardingState.ts` |

---

## 3. 具体技术实现（关键流程/数据结构/协议/命令）

### 3.1 目标文件内容

```js
// src/commands/onboarding/index.js
export default { isEnabled: () => false, isHidden: true, name: 'stub' };
```

该导出对象符合 `CommandBase` 的最小子集：
- `name: 'stub'` — 命令标识名（注意不是 `'onboarding'`，这是有意为之的占位名）。
- `isEnabled: () => false` — 永远返回 `false`，在 `getCommands()` 的过滤阶段被剔除。
- `isHidden: true` — 即使意外通过过滤，也不会出现在帮助、typeahead 等 UI 中。

> 注：该对象缺少 `type`（如 `'prompt'` / `'local'` / `'local-jsx'`）以及对应的具体执行字段（如 `getPromptForCommand` 或 `load`），因此它**不是一个合法的完整 `Command` 对象**。但由于 `isEnabled` 永远为 `false`，它永远不会被 `findCommand()` 命中并执行，类型不完整在实际运行中不会引发错误。

### 3.2 命令注册与过滤流程

在 `src/commands.ts` 中：

```ts
// 第 35 行
import onboarding from './commands/onboarding/index.js'

// 第 225-254 行
export const INTERNAL_ONLY_COMMANDS = [
  backfillSessions,
  breakCache,
  // ...
  onboarding,   // <-- 此处引入
  share,
  // ...
].filter(Boolean)

// 第 258-346 行
const COMMANDS = memoize((): Command[] => [
  addDir,
  advisor,
  // ... 大量内置命令 ...
  ...(process.env.USER_TYPE === 'ant' && !process.env.IS_DEMO
    ? INTERNAL_ONLY_COMMANDS
    : []),
])
```

`getCommands(cwd)` 的逻辑（第 476-517 行）会调用 `isCommandEnabled(_)` 过滤：

```ts
export function isCommandEnabled(cmd: CommandBase): boolean {
  return cmd.isEnabled?.() ?? true
}
```

由于 `onboarding.isEnabled()` 返回 `false`，它永远不会出现在最终返回的命令列表中。

### 3.3 真正的 Onboarding 启动流程

#### 3.3.1 触发条件

`src/interactiveHelpers.tsx` 第 104-123 行：

```ts
export async function showSetupScreens(root, permissionMode, ...): Promise<boolean> {
  if ("production" === 'test' || isEnvTruthy(false) || process.env.IS_DEMO) {
    return false; // 测试/演示模式跳过
  }
  const config = getGlobalConfig();
  let onboardingShown = false;
  if (!config.theme || !config.hasCompletedOnboarding) {
    onboardingShown = true;
    const { Onboarding } = await import('./components/Onboarding.js');
    await showSetupDialog(root, done => 
      <Onboarding onDone={() => { completeOnboarding(); void done(); }} />
    );
  }
  // ... 后续还有 TrustDialog、GroveDialog、API Key 审批等
  return onboardingShown;
}
```

触发 onboarding UI 的条件是：
1. 用户尚未选择主题（`!config.theme`），或
2. 用户尚未完成 onboarding（`!config.hasCompletedOnboarding`）。

#### 3.3.2 Onboarding UI 步骤

`src/components/Onboarding.tsx` 定义了以下步骤（`StepId`）：

| 步骤 ID | 条件 | 内容 |
|---------|------|------|
| `preflight` | `oauthEnabled` | 预检步骤（网络/环境检查） |
| `theme` | 始终 | 主题选择器 (`ThemePicker`) |
| `api-key` | 存在 `ANTHROPIC_API_KEY` 且为新 key | API Key 审批 (`ApproveApiKey`) |
| `oauth` | `oauthEnabled` | OAuth 登录流 (`ConsoleOAuthFlow`) |
| `security` | 始终 | 安全提示 |
| `terminal-setup` | `shouldOfferTerminalSetup()` | 终端快捷键/偏好设置推荐 |

#### 3.3.3 完成标记

`src/interactiveHelpers.tsx` 第 32-38 行：

```ts
export function completeOnboarding(): void {
  saveGlobalConfig(current => ({
    ...current,
    hasCompletedOnboarding: true,
    lastOnboardingVersion: MACRO.VERSION
  }));
}
```

完成 onboarding 后，会写入全局配置：
- `hasCompletedOnboarding: true`
- `lastOnboardingVersion: MACRO.VERSION`（用于未来版本升级时重置 onboarding）

### 3.4 登录/登出与 Onboarding 状态的交互

#### 3.4.1 登录后自动标记完成

`src/cli/handlers/auth.ts`（非交互式 `claude auth login`）第 166-171 行：

```ts
saveGlobalConfig(current => {
  if (current.hasCompletedOnboarding) return current
  return { ...current, hasCompletedOnboarding: true }
})
```

`src/commands/login/login.tsx`（交互式 `/login`）没有直接调用 `completeOnboarding()`，但 `src/main.tsx` 在检测到 `onboardingShown` 为 `true` 后，会执行与 `/login` 成功相同的后处理逻辑（刷新 remote managed settings、policy limits、GrowthBook、trusted device 等）。

#### 3.4.2 登出后重置 Onboarding

`src/commands/logout/logout.tsx` 第 16-47 行：

```ts
export async function performLogout({ clearOnboarding = false }): Promise<void> {
  // ...
  saveGlobalConfig(current => {
    const updated = { ...current };
    if (clearOnboarding) {
      updated.hasCompletedOnboarding = false;
      updated.subscriptionNoticeCount = 0;
      updated.hasAvailableSubscription = false;
      // 清空已批准的 custom API key 列表
    }
    updated.oauthAccount = undefined;
    return updated;
  });
}
```

注意 `clearOnboarding` 默认为 `false`，因此普通登出**不会**重置 onboarding 状态。

### 3.5 项目级 Onboarding（Project Onboarding）

除了全局的 `hasCompletedOnboarding`，还有项目级别的 onboarding 状态：

`src/projectOnboardingState.ts`：
- `getSteps()`：检查当前目录是否为空、是否存在 `CLAUDE.md`。
- `isProjectOnboardingComplete()`：判断项目级步骤是否全部完成。
- `maybeMarkProjectOnboardingComplete()`：在 REPL 每次用户提交消息时被调用（`src/screens/REPL.tsx` 第 2675 行），如果项目级 onboarding 已完成，则写入项目配置 `hasCompletedProjectOnboarding: true`。

这与 `src/commands/onboarding/index.js` 无直接关系，但属于 onboarding 概念体系的一部分。

### 3.6 配置数据结构

`src/utils/config.ts` 中相关的 `GlobalConfig` 字段：

```ts
export type GlobalConfig = {
  theme: ThemeSetting
  hasCompletedOnboarding?: boolean
  lastOnboardingVersion?: string
  hasCompletedClaudeInChromeOnboarding?: boolean
  hasIdeOnboardingBeenShown?: Record<string, boolean>
  // ...
}
```

`ProjectConfig` 中相关字段（通过 `getCurrentProjectConfig()` 访问）：

```ts
hasCompletedProjectOnboarding?: boolean
projectOnboardingSeenCount?: number
```

---

## 4. 关键代码路径与文件引用

### 4.1 直接引用目标文件的代码

| 引用文件 | 引用方式 | 用途 |
|----------|----------|------|
| `src/commands.ts:35` | `import onboarding from './commands/onboarding/index.js'` | 导入 stub |
| `src/commands.ts:243` | `onboarding` 放入 `INTERNAL_ONLY_COMMANDS` | 内部构建注册 |

### 4.2 真正的 Onboarding 实现路径

| 文件 | 相关函数/组件 | 职责 |
|------|---------------|------|
| `src/components/Onboarding.tsx` | `Onboarding` 组件 | 引导 UI（主题、OAuth、安全提示、终端设置） |
| `src/interactiveHelpers.tsx` | `showSetupScreens()`, `completeOnboarding()` | 启动时触发引导、标记完成 |
| `src/main.tsx` | `main()` / `run()` | 调用 `showSetupScreens()`，处理引导后的服务刷新 |
| `src/screens/REPL.tsx` | `onQueryImpl()` | 调用 `maybeMarkProjectOnboardingComplete()` |
| `src/projectOnboardingState.ts` | `maybeMarkProjectOnboardingComplete()` | 项目级 onboarding 完成检测与标记 |
| `src/commands/login/login.tsx` | `call()` / `Login` | `/login` 命令，引导中的 OAuth 复用此组件 |
| `src/cli/handlers/auth.ts` | `authLogin()`, `installOAuthTokens()` | CLI 登录成功后标记 onboarding 完成 |
| `src/commands/logout/logout.tsx` | `performLogout()` | 可选重置 onboarding 状态 |

### 4.3 配置与状态保护路径

| 文件 | 相关函数 | 职责 |
|------|----------|------|
| `src/utils/config.ts` | `saveGlobalConfig()`, `wouldLoseAuthState()` | 持久化 onboarding 状态，防止并发写覆盖导致状态丢失（GH #3117） |
| `src/utils/config.ts` | `GlobalConfig` 类型定义 | 声明 `hasCompletedOnboarding` 等字段 |

---

## 5. 依赖与外部交互

### 5.1 目标文件的依赖

`src/commands/onboarding/index.js` **零依赖**。它是一个纯 JavaScript 对象字面量导出，不导入任何内部模块或第三方库。

### 5.2 上游依赖（调用/注册方）

- **`src/commands.ts`**：命令注册中心。依赖 `src/types/command.ts` 中定义的类型系统。
- **构建时环境变量**：`process.env.USER_TYPE === 'ant'` 决定是否将 `INTERNAL_ONLY_COMMANDS`（含 onboarding）加入可用命令列表。

### 5.3 下游关联（真正的 onboarding 流程依赖）

| 依赖类别 | 具体依赖 | 说明 |
|----------|----------|------|
| UI 框架 | `react`, `ink` | `Onboarding.tsx` 使用 React + Ink 渲染 TUI |
| 配置存储 | `src/utils/config.ts` | 读写 `~/.claude/config.json`（或等价路径） |
| 安全存储 | `src/utils/secureStorage/index.ts` | OAuth token 存储，与 onboarding 中的登录步骤相关 |
| 授权服务 | `src/services/oauth/client.ts` | `createAndStoreApiKey`, `fetchAndStoreUserRoles` |
| 分析/遥测 | `src/services/analytics/index.ts` | `logEvent('tengu_began_setup')` 等 |
| 功能开关 | `bun:bundle` 的 `feature()` | 控制某些 onboarding 步骤是否显示（如 Grove 政策弹窗） |
| 终端检测 | `src/utils/env.ts` | `env.terminal` 用于判断终端类型以推荐设置 |

### 5.4 外部系统交互

真正的 onboarding 流程涉及以下外部交互：
1. **OAuth 授权服务器**：`ConsoleOAuthFlow` 会打开浏览器或提供 URL，引导用户完成 claude.ai 或 Console 的 OAuth 授权。
2. **Anthropic API**：登录成功后可能调用 `fetchAndStoreClaudeCodeFirstTokenDate()` 或 `createAndStoreApiKey()`。
3. **GrowthBook/Statsig**：用于功能开关（如 Grove 政策弹窗、Claude in Chrome onboarding）。

---

## 6. 风险、边界与改进建议

### 6.1 当前风险

#### R1：Stub 命令的类型不完整
`src/commands/onboarding/index.js` 导出的对象缺少 `type` 字段以及对应的具体执行属性（如 `load` 或 `getPromptForCommand`）。虽然 `isEnabled: () => false` 保证了它永远不会被执行，但如果未来某次重构中 `getCommands()` 的过滤逻辑被提前或绕过（例如直接通过 `INTERNAL_ONLY_COMMANDS` 索引访问），运行时可能会因为缺少 `type` 而抛出异常。

#### R2：名称不一致
Stub 的 `name` 是 `'stub'` 而不是 `'onboarding'`。这在命令查找、日志记录或调试时会造成困惑——开发者可能期望 `findCommand('onboarding', ...)` 能找到它，但实际上它注册的名字是 `'stub'`。

#### R3：Onboarding 与 `/login` 的后处理逻辑分散且需要手动同步
`src/main.tsx` 第 2279-2297 行和 `src/commands/login/login.tsx` 第 26-56 行都包含几乎相同的 post-login 刷新逻辑（`refreshRemoteManagedSettings`, `refreshPolicyLimits`, `resetUserCache`, `refreshGrowthBookAfterAuthChange`, `enrollTrustedDevice` 等）。代码注释也明确提醒 "Keep in sync with the post-login logic in src/commands/login.tsx"。这种重复增加了维护成本，未来新增一个 post-login 步骤时容易遗漏其中一处。

#### R4：配置状态丢失防护的边界情况
`src/utils/config.ts` 中的 `wouldLoseAuthState()` 只检查 `hasCompletedOnboarding` 和 `oauthAccount`，但不检查 `lastOnboardingVersion`。在极端并发写场景下，如果 onboarding 版本重置逻辑与配置写操作竞态，可能导致 `lastOnboardingVersion` 回退，从而在下次版本升级时错误地重新触发 onboarding。

### 6.2 边界情况

#### B1：非交互式模式（`-p` / `--print`）
`showSetupScreens()` 不会被非交互式路径调用。因此通过 `claude auth login`（`src/cli/handlers/auth.ts`）登录是唯一能标记 onboarding 完成的非交互式入口。

#### B2：演示模式（`IS_DEMO`）
演示环境完全跳过 onboarding，但 `hasCompletedOnboarding` 配置字段可能仍为 `undefined` 或 `false`。如果演示用户后续切换到正常模式，可能会再次触发 onboarding。

#### B3：Homespace 环境
在 `isRunningOnHomespace()` 为 `true` 时，`ANTHROPIC_API_KEY` 环境变量被忽略（见 `src/components/Onboarding.tsx` 第 102 行和 `src/interactiveHelpers.tsx` 第 206 行）。这意味着即使设置了 API Key，onboarding 中的 "API Key 审批" 步骤也不会出现，用户必须通过 OAuth 流程。

#### B4：登出不清除 onboarding 默认行为
`performLogout({ clearOnboarding: false })` 是默认行为。用户登出后重新登录不会重新经历 onboarding UI，但会重新走 OAuth 流程。这在多账号切换场景下是符合预期的，但如果用户希望"彻底重置"，没有显式的 `/onboarding` 命令可供调用。

### 6.3 改进建议

#### S1：将 stub 完善为类型安全的完整 Command 对象
建议将 `src/commands/onboarding/index.js` 改写为：

```ts
import type { Command } from '../../types/command.js'

const onboarding: Command = {
  type: 'local-jsx',
  name: 'onboarding',
  description: 'Restart the onboarding flow',
  isEnabled: () => false,
  isHidden: true,
  load: async () => {
    const { Onboarding } = await import('../../components/Onboarding.js')
    return {
      call: async (onDone, context) => {
        return <Onboarding onDone={() => onDone()} />
      }
    }
  }
}

export default onboarding
```

这样做的好处：
- 消除类型不完整的风险；
- `name` 与文件路径语义一致；
- 为未来启用 `/onboarding` 命令做好准备（只需将 `isEnabled` 改为动态判断即可）。

#### S2：提取统一的 post-login 刷新函数
将 `src/main.tsx` 和 `src/commands/login/login.tsx` 中重复的 post-login 逻辑提取到一个单独的模块（如 `src/utils/auth/postLoginRefresh.ts`）中，统一维护。

#### S3：考虑在 `/logout` 时提供 `--clear-onboarding` 的显式 CLI 选项
目前 `performLogout` 的 `clearOnboarding` 参数只在内部调用中使用。如果产品希望支持"彻底重置用户体验"，可以在 CLI 的 `claude auth logout` 中暴露 `--clear-onboarding` 标志。

#### S4：增强 `wouldLoseAuthState` 的检查范围
将 `lastOnboardingVersion` 纳入 `wouldLoseAuthState` 的保护范围，防止版本标记在并发写中丢失。

---

## 7. 结论

`src/commands/onboarding/index.js` 本身是一个**零业务逻辑的占位符文件**，其存在意义在于命令注册表中的名称保留和访问控制。Claude Code 中所有与用户首次引导相关的实际功能——UI 渲染、状态判断、配置持久化、登录后刷新——都分布在 `src/components/Onboarding.tsx`、`src/interactiveHelpers.tsx`、`src/main.tsx`、`src/utils/config.ts` 等模块中。理解该文件必须结合上述上下文，否则容易误判为" onboarding 功能缺失"。
