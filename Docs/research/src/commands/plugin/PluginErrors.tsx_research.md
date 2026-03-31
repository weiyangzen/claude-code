# PluginErrors.tsx 研究文档

> 文件路径：`src/commands/plugin/PluginErrors.tsx`  
> 研究时间：2026-04-01  
> 执行器：kimi (k2p5)

---

## 场景与职责

`PluginErrors.tsx` 是插件子系统中专门负责**错误格式化与引导**的纯工具模块。它不渲染任何 UI，也不持有状态，仅对外暴露两个纯函数：

- `formatErrorMessage(error: PluginError): string` —— 将结构化错误对象转换为用户可读的单行消息。
- `getErrorGuidance(error: PluginError): string | null` —— 根据错误类型返回可操作的修复建议（guidance）。

该文件位于展示层（`src/commands/plugin/`），但本质上是**无 React 依赖的纯 TS 工具**。它的调用方主要是：
- `ManagePlugins.tsx` —— 在插件列表/详情中展示错误文本。
- `PluginSettings.tsx` —— 在 Errors Tab 中按错误类型渲染消息与建议。

---

## 功能点目的

1. **统一错误文案出口**  
   插件加载、安装、MCP/LSP 初始化等流程会产生大量 `PluginError` 结构化错误。该文件是所有面向用户文案的**唯一格式化层**，避免各调用方自行拼接字符串导致风格不一致。

2. **类型安全穷尽检查**  
   通过 `switch` + `const _exhaustive: never = error` 的 TS 模式，确保当 `PluginError` 联合类型新增分支时，编译器会强制提醒此处需要补充对应文案。

3. **可操作建议分层**  
   `formatErrorMessage` 负责“发生了什么”，`getErrorGuidance` 负责“怎么办”。后者在 `PluginSettings.tsx` 的 Errors Tab 中直接决定用户能否一键定位修复路径（如提示可用市场列表、建议启用依赖插件、或引导检查 SSH 密钥）。

---

## 具体技术实现（关键流程/数据结构/协议/命令）

### 数据结构：PluginError 联合类型

依赖 `src/types/plugin.ts` 中定义的 discriminated union，当前包含 20+ 种错误变体，核心字段如下：

| 类型 | 关键字段 | 说明 |
|------|---------|------|
| `generic-error` | `error: string` | 兜底类型，直接透传原始错误文本 |
| `plugin-not-found` | `pluginId`, `marketplace` | 在市场里找不到指定插件 |
| `marketplace-not-found` | `marketplace`, `availableMarketplaces: string[]` | 市场源不存在，附带已知市场列表 |
| `mcp-server-suppressed-duplicate` | `serverName`, `duplicateOf` | MCP 服务器因命令/URL 重复被去重跳过 |
| `dependency-unsatisfied` | `dependency`, `reason: 'not-enabled' \| 'not-found'` | 依赖插件未启用或未安装 |
| `lsp-server-crashed` | `serverName`, `exitCode`, `signal?` | LSP 进程崩溃 |
| `marketplace-blocked-by-policy` | `blockedByBlocklist?`, `allowedSources: string[]` | 企业策略拦截市场源 |

### 关键流程

#### 1. formatErrorMessage
```tsx
switch (error.type) {
  case 'path-not-found':
    return `${error.component} path not found: ${error.path}`
  case 'git-auth-failed':
    return `Git ${error.authType.toUpperCase()} authentication failed for ${error.gitUrl}`
  // ... 20+ 分支
}
const _exhaustive: never = error
return getPluginErrorMessage(_exhaustive) // 兜底 fallback
```

- 对 `mcp-server-suppressed-duplicate` 做了特殊解析：`duplicateOf` 若以 `plugin:` 开头，则提取插件名并生成“由插件 X 提供”的文案；否则显示为已配置的服务名。
- 对 `lsp-server-crashed` 区分 `signal` 与 `exitCode` 的优先级：有信号则报信号，否则报退出码。
- 穷尽分支后仍保留 `getPluginErrorMessage` 兜底，防止运行时遇到未编译覆盖的新类型导致 `undefined`。

#### 2. getErrorGuidance
```tsx
switch (error.type) {
  case 'git-auth-failed':
    return error.authType === 'ssh'
      ? 'Configure SSH keys or use HTTPS URL instead'
      : 'Configure credentials or use SSH URL instead'
  case 'marketplace-not-found':
    return error.availableMarketplaces.length > 0
      ? `Available marketplaces: ${error.availableMarketplaces.join(', ')}`
      : 'Add the marketplace first using /plugin marketplace add'
  // ...
}
```

- 部分类型（如 `marketplace-load-failed`、`generic-error`）返回 `null`，表示系统暂无通用修复建议，由上层决定是否静默或展示原始堆栈。
- `mcp-server-suppressed-duplicate` 的 guidance 会区分冲突来源：若是另一插件胜出，则建议禁用该插件；若是用户手动配置的 MCP 服务，则建议从配置中移除。

---

## 关键代码路径与文件引用

### 本文件
- `src/commands/plugin/PluginErrors.tsx`（124 行，含 source map）

### 直接依赖
- `src/types/plugin.ts` —— 引入 `PluginError` 类型与 `getPluginErrorMessage` 兜底函数。

### 调用方
- `src/commands/plugin/ManagePlugins.tsx:47`
  ```tsx
  import { formatErrorMessage, getErrorGuidance } from './PluginErrors.js'
  ```
  用于构建 `UnifiedInstalledItem` 的错误展示文本，以及在 `failed-plugin-details` 视图中渲染错误行。

- `src/commands/plugin/PluginSettings.tsx:26`
  ```tsx
  import { formatErrorMessage, getErrorGuidance } from './PluginErrors.js'
  ```
  在 `ErrorsTabContent` / `buildErrorRows` 中按错误分类生成 `ErrorRow` 的 `message` 与 `guidance` 字段。

### 相关类型定义路径
- `src/types/plugin.ts:101-283` —— `PluginError` 完整联合类型定义。
- `src/types/plugin.ts:295-363` —— `getPluginErrorMessage` 函数（与 `PluginErrors.tsx` 内容高度重合，但位于类型文件，作为更底层的兜底）。

---

## 依赖与外部交互

| 依赖 | 方向 | 说明 |
|------|------|------|
| `../../types/plugin.js` | 导入 | 类型定义 + fallback 格式化函数 |
| `ManagePlugins.tsx` | 被调用 | 渲染插件错误消息 |
| `PluginSettings.tsx` | 被调用 | 渲染 Errors Tab 中的消息与建议 |
| React / Ink | 无 | 本文件零 UI 运行时依赖 |

**无网络、无磁盘、无状态、无配置读取**。两个导出函数均为纯函数，输入 `PluginError` 输出 `string | null`，具备完全可测试性（但当前仓库中未找到对应单元测试文件）。

---

## 风险、边界与改进建议

### 风险与边界

1. **与 `types/plugin.ts` 的重复维护负担**  
   `getPluginErrorMessage`（在 `types/plugin.ts`）与 `formatErrorMessage`（在本文件）覆盖了几乎相同的错误类型，文案略有差异。若产品要求统一措辞，需要同时修改两处，容易遗漏。

2. **新增错误类型的编译时安全依赖人工补全**  
   虽然 `_exhaustive: never` 能在编译期告警，但如果开发者为了快速过编译而在 `switch` 中增加 `default` 分支，穷尽检查即失效。

3. **Guidance 的国际化缺失**  
   当前所有文案均为硬编码英文，未接入任何 i18n 框架。对于非英语用户，Errors Tab 的修复建议体验受限。

4. **无单元测试覆盖**  
   搜索全仓库未找到针对 `formatErrorMessage` 或 `getErrorGuidance` 的测试。该模块逻辑以 `switch` 为主，虽然简单，但 20+ 分支的回归风险仍建议用快照测试或参数化测试覆盖。

### 改进建议

1. **合并或委托单一真相源**  
   建议将 `getPluginErrorMessage` 从 `types/plugin.ts` 迁移至 `PluginErrors.tsx`，或让 `formatErrorMessage` 在 `default` 分支直接调用 `getPluginErrorMessage`，并删除本文件中的重复 `case`，避免双轨维护。

2. **补充参数化单元测试**  
   可基于 `PluginError` 的每个变体构造最小对象，断言 `formatErrorMessage` 与 `getErrorGuidance` 的输出包含预期子串。该测试零依赖、运行极快，能显著提升重构信心。

3. **为 guidance 引入可扩展的 action map**  
   当前 `getErrorGuidance` 返回纯文本。未来若 Errors Tab 需要“一键修复”按钮（如自动禁用冲突插件、自动添加市场），可将返回值升级为 `{ guidance: string, action?: ErrorAction }` 结构，减少上层 `PluginSettings.tsx` 的再分发逻辑。

4. **考虑 i18n 键提取**  
   若产品计划支持多语言，建议将错误文案抽离为键值对（如 `pluginErrors.gitAuthFailed.message` / `.guidance`），本文件仅保留映射表，便于后续接入翻译系统。
