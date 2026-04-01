# src/commands/usage/usage.tsx 深度研究文档

## 场景与职责

### 1.1 功能定位
`src/commands/usage/usage.tsx` 是 `/usage` 斜杠命令的**实际执行体**。作为 `local-jsx` 类型命令的实现模块，它导出一个符合 `LocalJSXCommandCall` 签名的 `call` 函数。当用户在 REPL 中触发 `/usage` 时，命令系统通过 `index.ts` 懒加载本模块，并调用 `call()` 函数，返回一个 React 元素树，最终由 Ink 渲染在终端界面中。

### 1.2 使用场景
- **用户主动查询限额**：用户在 CLI 中输入 `/usage`，终端弹出 Settings 面板并自动定位到 "Usage" Tab。
- **与 Settings 面板集成**：`/usage` 不是独立的页面，而是复用现有的 `Settings` 组件基础设施，通过 `defaultTab="Usage"` 参数直接跳转到 Usage 页签。
- **非交互式会话不支持**：由于返回的是 JSX UI，该命令无法在纯管道/非 TUI 模式下执行。

### 1.3 执行生命周期
```
用户输入 /usage
    ↓
src/commands/usage/index.ts 的 load() 动态导入本文件
    ↓
调用 call(onDone, context, args)
    ↓
返回 <Settings onClose={onDone} context={context} defaultTab="Usage" />
    ↓
Ink 渲染 Settings 组件 → 内部渲染 Usage Tab
    ↓
用户按 Esc 或选择关闭 → onDone 回调被触发 → 命令结束
```

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `call` 函数 | 作为 `LocalJSXCommandCall` 的实现，是命令系统与本模块的唯一交互接口 |
| 渲染 `Settings` 组件 | 复用设置面板容器，统一处理 Esc 关闭、Tab 切换、快捷键等交互 |
| `defaultTab="Usage"` | 确保用户输入 `/usage` 后直接看到 Usage 页签，而非默认的 Status 页签 |
| 透传 `onDone` 和 `context` | 将命令系统的完成回调和工具上下文传递给 Settings 组件，保证面板关闭后 CLI 恢复正常 REPL 流程 |

---

## 具体技术实现

### 3.1 源码全文
```typescript
import * as React from 'react';
import { Settings } from '../../components/Settings/Settings.js';
import type { LocalJSXCommandCall } from '../../types/command.js';

export const call: LocalJSXCommandCall = async (onDone, context) => {
  return <Settings onClose={onDone} context={context} defaultTab="Usage" />;
};
```

### 3.2 类型系统解析

#### `LocalJSXCommandCall`（定义于 `src/types/command.ts:131-135`）
```typescript
export type LocalJSXCommandCall = (
  onDone: LocalJSXCommandOnDone,
  context: ToolUseContext & LocalJSXCommandContext,
  args: string,
) => Promise<React.ReactNode>
```

本文件的 `call` 函数签名与之完全匹配：
- **`onDone: LocalJSXCommandOnDone`**：当用户关闭 Settings 面板时调用，可选地传入结果文本和显示选项。例如 `onDone("Status dialog dismissed", { display: "system" })`。
- **`context: ToolUseContext & LocalJSXCommandContext`**：包含当前会话的完整上下文，如消息历史、MCP 配置、IDE 安装状态、主题、恢复会话函数等。Settings 组件需要 `context` 来支持 Config Tab 中的部分配置项。
- **`args: string`**：用户输入 `/usage` 时可能附带的参数（如 `/usage extra`）。本实现忽略该参数，因为 Usage 功能不需要命令行参数。
- **返回值 `Promise<React.ReactNode>`**：返回一个 React 元素，Ink 会将其渲染到终端。

### 3.3 `LocalJSXCommandOnDone` 详解
```typescript
export type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: CommandResultDisplay  // 'skip' | 'system' | 'user'
    shouldQuery?: boolean
    metaMessages?: string[]
    nextInput?: string
    submitNextInput?: boolean
  },
) => void
```

在 `Settings.tsx` 中，`onClose` 就是 `onDone`。当用户按 Esc 关闭面板时：
```typescript
onClose("Status dialog dismissed", { display: "system" })
```
这会在对话历史中插入一条系统消息（对用户可见但标记为系统消息），告知面板已被关闭。

### 3.4 与 Settings 组件的集成

`Settings.tsx`（`src/components/Settings/Settings.tsx`）是一个受控的 Tab 容器，接收以下与本命令相关的 props：

| Prop | 本文件传入值 | 在 Settings 中的作用 |
|------|-------------|---------------------|
| `onClose` | `onDone` | 关闭面板的回调，触发命令结束 |
| `context` | `context` | 透传给 Status/Config/Usage 等子 Tab |
| `defaultTab` | `"Usage"` | `useState(defaultTab)` 初始化当前选中 Tab |

**Settings 内部的 Tab 结构：**
```typescript
const tabs = [
  <Tab key="status" title="Status"><Status ... /></Tab>,
  <Tab key="config" title="Config"><Config ... /></Tab>,
  <Tab key="usage" title="Usage"><Usage /></Tab>,
  // Gates Tab（当前被 feature flag 关闭）
]
```

本命令通过 `defaultTab="Usage"` 确保 `selectedTab` 初始值为 `"Usage"`，用户打开后直接看到 Usage 内容。

### 3.5 Usage Tab 的内部实现（`src/components/Settings/Usage.tsx`）

虽然不在本文件中，但它是 `call()` 返回的 React 树的核心渲染部分，值得在研究上下文中说明：

#### 数据获取
```typescript
const loadUtilization = React.useCallback(async () => {
  setIsLoading(true);
  setError(null);
  try {
    const data = await fetchUtilization();
    setUtilization(data);
  } catch (err) {
    // 错误处理：展示 API 错误响应体
  } finally {
    setIsLoading(false);
  }
}, []);

useEffect(() => {
  void loadUtilization();
}, [loadUtilization]);
```

#### 限额展示逻辑
```typescript
const subscriptionType = getSubscriptionType();
const showSonnetBar = subscriptionType === 'max' || subscriptionType === 'team' || subscriptionType === null;

const limits = [
  { title: 'Current session', limit: utilization.five_hour },
  { title: 'Current week (all models)', limit: utilization.seven_day },
  ...(showSonnetBar ? [{ title: 'Current week (Sonnet only)', limit: utilization.seven_day_sonnet }] : [])
];
```

- **Max/Team 计划**：Sonnet 有独立的周限额，显示 3 个进度条。
- **Pro/Enterprise 计划**：Sonnet 周限额与总周限额一致，不显示重复条。
- **未知计划（`null`）**：保守地显示 Sonnet 条，与后端错误提示文案 `rateLimitMessages.ts` 保持一致。

#### 额外使用额度（Extra Usage）
仅对 Pro/Max 用户展示：
- 未启用且 `/extra-usage` 命令可用时：提示 `/extra-usage to enable`
- 无限额（`monthly_limit === null`）：显示 "Unlimited"
- 有限额：显示进度条和 `$used / $limit spent`

#### 超支信用推广（OverageCreditUpsell）
```typescript
{isEligibleForOverageCreditGrant() && <OverageCreditUpsell maxWidth={maxWidth} />}
```
调用 `isEligibleForOverageCreditGrant()` 检查用户是否符合免费超支信用额度资格，若符合则展示推广信息。

---

## 关键代码路径与文件引用

### 4.1 完整调用链
```
用户输入 /usage
    ↓
src/commands.ts:findCommand('usage', commands)
    ↓
src/commands/usage/index.ts:Command.load()
    ↓
【本文件】src/commands/usage/usage.tsx:call(onDone, context)
    ↓
src/components/Settings/Settings.tsx (props: onClose=onDone, context, defaultTab="Usage")
    ↓
src/components/Settings/Usage.tsx:Usage()
    ↓
src/services/api/usage.ts:fetchUtilization()
    ↓
GET /api/oauth/usage (Claude.ai API)
```

### 4.2 核心文件引用

| 文件路径 | 与本文件关系 | 用途 |
|----------|-------------|------|
| `src/commands/usage/index.ts` | **调用方/入口** | 懒加载本文件，提供命令元数据 |
| `src/components/Settings/Settings.tsx` | **直接依赖** | 设置面板容器，处理 Tab 状态和关闭逻辑 |
| `src/components/Settings/Usage.tsx` | **间接依赖（子组件）** | Usage Tab 的具体 UI 和数据获取逻辑 |
| `src/services/api/usage.ts` | **间接依赖（API 层）** | `fetchUtilization()` 获取后端限额数据 |
| `src/types/command.ts` | **类型依赖** | `LocalJSXCommandCall`、`LocalJSXCommandOnDone` 等类型定义 |
| `src/utils/auth.ts` | **间接依赖** | `getSubscriptionType()` 决定 Sonnet 条和 Extra Usage 的显示 |

### 4.3 快捷键绑定
在 `Usage.tsx` 中绑定了以下快捷键：
- **`r`**（`settings:retry`）：在加载出错时重试获取数据
- **`Esc`**（`confirm:no`）：关闭 Settings 面板（实际处理在 `Settings.tsx` 中）

---

## 依赖与外部交互

### 5.1 直接依赖
| 模块 | 导入内容 | 用途 |
|------|---------|------|
| `react` | `React` | JSX 运行时 |
| `../../components/Settings/Settings.js` | `Settings` (组件) | 渲染设置面板 |
| `../../types/command.js` | `LocalJSXCommandCall` (type) | 类型约束 |

### 5.2 间接/运行时依赖（通过 Settings → Usage 组件链）
| 模块 | 作用 |
|------|------|
| `src/services/api/usage.ts` | 发起 `GET /api/oauth/usage` 请求 |
| `src/utils/auth.ts` | 订阅类型检测、OAuth token 检查 |
| `src/utils/format.ts` | `formatResetText()` 格式化重置时间 |
| `src/cost-tracker.ts` | `formatCost()` 格式化额外使用额度金额 |
| `src/keybindings/useKeybinding.ts` | 绑定 `r` 重试快捷键 |
| `src/hooks/useTerminalSize.ts` | 获取终端尺寸以适配布局 |
| `src/ink.ts` | `Box`、`Text` 等 Ink 组件 |
| `src/components/design-system/ProgressBar.tsx` | 限额进度条 UI |
| `src/components/LogoV2/OverageCreditUpsell.tsx` | 超支信用推广组件 |

### 5.3 外部 API 交互
本文件本身不直接发起网络请求，但通过 `Usage.tsx` → `fetchUtilization()` 间接调用：

**端点**：`GET ${BASE_API_URL}/api/oauth/usage`

**认证要求**：
- OAuth Access Token（Claude.ai）
- `user:profile` scope

**请求头**：
```typescript
{
  'Content-Type': 'application/json',
  'User-Agent': getClaudeCodeUserAgent(),
  'Authorization': 'Bearer <oauth_token>'
}
```

**超时**：5 秒

---

## 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 `call` 函数忽略 `args` 参数
- **风险**：用户可能尝试输入 `/usage sonnet` 或 `/usage extra` 等参数，但本实现完全忽略 `args`，不会给出任何反馈，用户体验不佳。
- **代码位置**：本文件第 4 行 `async (onDone, context) => { ... }` 未使用第三个参数 `args`。

#### 6.1.2 依赖 Settings 组件的隐式耦合
- **风险**：`Settings` 组件的 `defaultTab` 是字符串字面量 `"Usage"`，若 `Settings.tsx` 中 Tab 的 `key` 或标题被重命名（如改为 `"Plan Usage"`），`defaultTab` 可能失效，导致打开后未选中预期 Tab。
- **现状**：`Settings.tsx` 中 Usage Tab 的 `key="usage"` 和 `title="Usage"` 与本文件的 `"Usage"` 匹配，但缺乏类型安全约束。

#### 6.1.3 API 失败时的错误信息可能过于技术化
- **风险**：`Usage.tsx` 在 `catch` 块中直接将 API 响应体 `JSON.stringify` 后展示给用户（`Failed to load usage data: ${responseBody}`），可能暴露内部错误细节或造成用户困惑。
- **代码位置**：`src/components/Settings/Usage.tsx:196-197`

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 非 TUI 模式（如管道输入） | `local-jsx` 命令无法执行，取决于上层是否拦截 |
| OAuth token 过期 | `fetchUtilization()` 返回 `null`，UI 显示 "Loading usage data…"  indefinitely |
| API 超时（5s） | 捕获错误，展示重试选项（按 `r`） |
| 无 profile scope | `fetchUtilization()` 返回 `{}`，Usage 组件展示 "/usage is only available for subscription plans." |
| 窄终端（<62 字符） | `LimitBar` 切换为垂直堆叠布局 |
| 用户快速连续输入 `/usage` | 每次都会重新挂载 `Usage` 组件，重新发起 `fetchUtilization()`，无缓存 |

### 6.3 改进建议

#### 6.3.1 支持命令参数
```typescript
export const call: LocalJSXCommandCall = async (onDone, context, args) => {
  // 例如：/usage extra 可直接高亮 Extra Usage 区域
  const highlightExtra = args.trim().toLowerCase() === 'extra';
  return <Settings onClose={onDone} context={context} defaultTab="Usage" usageHighlight={highlightExtra} />;
};
```

#### 6.3.2 将 `defaultTab` 字面量类型化
在 `src/types/command.ts` 或 `src/components/Settings/Settings.tsx` 中导出 Tab 名称联合类型：
```typescript
export type SettingsTab = 'Status' | 'Config' | 'Usage' | 'Gates';
```
并将 `Settings` 的 `defaultTab` prop 和本文件的调用点都约束为该类型，防止拼写错误。

#### 6.3.3 增加数据缓存
当前每次打开 `/usage` 都重新请求 API。建议：
- 在 `Usage.tsx` 中使用 React Context 或 SWR 风格的短期缓存（如 30 秒），减少重复请求。
- 或在 `fetchUtilization()` 层添加简单的内存缓存，并在 token 变更时自动失效。

#### 6.3.4 优化 token 过期体验
当 `fetchUtilization()` 检测到 token 过期返回 `null` 时，当前 UI 只显示 "Loading usage data…"。建议改为：
```typescript
if (data === null) {
  return <Text color="error">Session expired. Please run /login to refresh.</Text>;
}
```

#### 6.3.5 错误信息脱敏
在 `Usage.tsx` 中避免直接展示原始 API 响应体：
```typescript
setError('Failed to load usage data. Please try again later.');
// 同时将原始错误记录到调试日志
logError(err as Error);
```

### 6.4 测试建议

| 测试场景 | 验证点 |
|----------|--------|
| `call` 返回值 | 返回的 React 元素类型为 `Settings`，props 正确 |
| Settings 默认 Tab | `defaultTab="Usage"` 被正确传递 |
| `onDone` 回调 | 关闭 Settings 面板后 `onDone` 被调用，参数符合预期 |
| Max 用户 | 显示 3 个限额条（会话、周、Sonnet） |
| Pro 用户 | 显示 2 个限额条，Extra Usage 区域可见 |
| API 错误 | 展示友好错误信息和重试按钮 |
| 窄终端 | `LimitBar` 布局正确切换为垂直模式 |
| Token 过期 | 优雅处理，引导重新登录 |
