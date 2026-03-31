# 研究文档：src/commands/passes/index.ts

## 场景与职责

本文件是 Claude Code CLI 中 `/passes` 内置命令的**注册入口（Command Registry Entry）**。它遵循项目统一的命令模块化组织约定——`src/commands/<command-name>/index.ts` 仅负责导出命令元数据，而将实际执行逻辑延迟加载到同目录下的实现文件中。

在系统启动时，`src/commands.ts` 会导入本模块（`import passes from './commands/passes/index.js'`），并将其注入全局命令列表 `COMMANDS`。因此，本文件是 `/passes` 命令被用户、自动补全、帮助系统以及模型工具发现的第一道关口。

## 功能点目的

1. **命令声明**：向框架注册一个名为 `passes` 的斜杠命令。
2. **懒加载（Lazy Loading）**：通过 `load` 函数将 `passes.tsx` 的实际实现延迟到命令首次被调用时才导入，降低启动内存开销。
3. **动态可见性控制**：基于用户的 Guest Passes 资格缓存状态，决定该命令是否出现在帮助、自动补全和命令列表中。
4. **动态描述文案**：根据用户是否处于带奖励的推荐活动（referrer reward），切换不同的命令描述，提升转化率。

## 具体技术实现

### 命令类型与结构

文件导出一个满足 `Command` 类型（来自 `src/types/command.ts`）的对象：

- `type: 'local-jsx'` — 表明该命令会渲染一个本地 React/Ink 组件，而非直接返回文本或 prompt 扩展。
- `name: 'passes'` — 命令标识符，用户通过输入 `/passes` 触发。
- `description` — 使用 **getter**，在每次读取时动态计算：
  - 若 `getCachedReferrerReward()` 返回非空奖励信息，描述为 `"Share a free week of Claude Code with friends and earn extra usage"`。
  - 否则为 `"Share a free week of Claude Code with friends"`。
- `isHidden` — 使用 **getter**，在每次命令列表刷新时评估：
  - 调用 `checkCachedPassesEligibility()`，返回 `{ eligible, hasCache }`。
  - 当 `!eligible || !hasCache` 时，命令被隐藏。
- `load: () => import('./passes.js')` — 返回一个 Promise，解析为 `LocalJSXCommandModule`，其 `call` 函数定义在 `passes.tsx` 中。

### 资格检查逻辑（依赖侧）

`checkCachedPassesEligibility()` 和 `getCachedReferrerReward()` 均来自 `src/services/api/referral.ts`：

- 它们只读取 `GlobalConfig` 中的 `passesEligibilityCache`（按 `orgId` 索引），**不发起同步网络请求**。
- 缓存命中与否决定了命令的可见性，从而实现了“无网络阻塞”的命令列表渲染。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/commands.ts:129` | `import passes from './commands/passes/index.js'` |
| `src/commands.ts:338` | `passes` 被加入 `COMMANDS` 数组，成为内置命令 |
| `src/commands.ts:483-484` | `getCommands()` 中通过 `meetsAvailabilityRequirement()` 和 `isCommandEnabled()` 过滤 |
| `src/types/command.ts:144-152` | `LocalJSXCommand` 类型定义，说明 `load` 必须返回 `Promise<LocalJSXCommandModule>` |
| `src/services/api/referral.ts:83-126` | `checkCachedPassesEligibility()` 实现 |
| `src/services/api/referral.ts:150-156` | `getCachedReferrerReward()` 实现 |
| `src/utils/config.ts:301-305` | `GlobalConfig.passesEligibilityCache` 类型声明 |

## 依赖与外部交互

### 直接依赖

- `../../commands.js`（类型 `Command`）
- `../../services/api/referral.js`（`checkCachedPassesEligibility`, `getCachedReferrerReward`）

### 运行时依赖

- `./passes.js`：由 `passes.tsx` 经构建/编译生成的产物，包含 `call` 函数和 React 组件。

### 数据流

```
用户启动 CLI
  → src/commands.ts 加载本模块
  → isHidden getter 读取 referral.ts 中的缓存资格
  → 若 eligible && hasCache，则 passes 出现在命令列表
  → 用户输入 /passes
  → load() 动态导入 passes.js
  → 执行 passes.tsx 中的 call() 函数
```

## 风险、边界与改进建议

### 风险

1. **getter 执行频率**：`isHidden` 和 `description` 在每次 `getCommands()` 调用时都会重新求值（例如每次 REPL 渲染、自动补全刷新）。虽然当前依赖的缓存读取是 O(1) 内存查找，但如果未来 `referral.ts` 中的逻辑变重，可能成为性能瓶颈。
2. **冷启动隐藏**：若用户首次启动且本地无 `passesEligibilityCache`，`hasCache` 为 `false`，命令会被隐藏。`referral.ts` 会在后台发起 `fetchAndStorePassesEligibility()`，但用户必须等到**下一次会话**才能看到命令（见 `referral.ts:246-252` 注释）。这可能导致用户困惑。

### 边界

- 本文件**不处理任何网络 I/O**或 UI 渲染，仅做元数据声明和可见性控制。
- `isHidden` 的判定完全基于本地缓存，若缓存过期（>24h），仍可能显示命令，直到用户实际调用时才在 `Passes` 组件中触发后台刷新并发现资格已失效。

### 改进建议

1. **添加 `availability` 限制**：当前 `passes` 命令对所有用户可见（只要缓存说 eligible）。可以考虑显式声明 `availability: ['claude-ai']`，因为 Guest Passes 功能目前仅面向 Claude.ai 订阅者（`shouldCheckForPasses()` 内部已做此限制）。这样可以在 `meetsAvailabilityRequirement()` 层提前过滤，减少不必要的 `isHidden` 计算。
2. **缓存预热提示**：在冷启动隐藏命令的场景下，可以考虑在后台获取完成后，通过某种方式（如 tip 或通知）告知用户 `/passes` 已可用，而无需等待下次重启。
3. **类型安全**：`load: () => import('./passes.js')` 返回的是隐式 `Promise<any>`，可以显式标注为 `Promise<LocalJSXCommandModule>` 以增强编译期检查。
