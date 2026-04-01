# 研究文档：src/utils/unaryLogging.ts

## 场景与职责

本模块是 Claude Code **一元操作（unary action）的 analytics 日志封装层**。所谓“一元操作”，指用户面对单个建议或请求做出接受/拒绝/响应的交互，例如：

- 接受或拒绝一次 `str_replace`（单处或多处）；
- 接受或拒绝一次 `write_file`；
- 对单个 `tool_use` 的权限请求做出决定。

模块将这些离散的用户决策统一格式化为 `tengu_unary_event` 事件，发送到后台 analytics 系统（Statsig/GrowthBook），用于衡量工具接受率、用户反馈分布等产品指标。

## 功能点目的

| 导出符号 | 目的 |
|---------|------|
| `CompletionType` | 类型联合：`'str_replace_single' \| 'str_replace_multi' \| 'write_file_single' \| 'tool_use_single'`。 |
| `logUnaryEvent(event)` | 异步发送 `tengu_unary_event`，包含 completion_type、event 类型、以及 metadata。 |

## 具体技术实现

### 1. 事件类型定义

```ts
type LogEvent = {
  completion_type: CompletionType
  event: 'accept' | 'reject' | 'response'
  metadata: {
    language_name: string | Promise<string>
    message_id: string
    platform: string
    hasFeedback?: boolean
  }
}
```

- `event` 取值为 `accept`（用户同意）、`reject`（用户拒绝）、`response`（用户给出某种响应，不一定是二元决策）。
- `language_name` 可能是同步字符串或 `Promise<string>`，因为某些调用方在记录日志时可能还在异步解析文件语言类型。

### 2. 异步发送逻辑

`logUnaryEvent` 是 `async` 函数：

1. 调用 `await event.metadata.language_name` 解析可能的 Promise。
2. 将各字段通过类型断言转换为 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`（这是项目内部的 analytics 类型标记，用于静态检查确保不泄露代码或路径）。
3. 调用 `logEvent('tengu_unary_event', { ... })`。
4. `hasFeedback` 仅在显式定义时才展开到 payload 中（避免发送 `undefined`）。

### 3. 调用链路

该模块被 `src/components/permissions/` 下的多个权限相关组件和 hook 调用：

- `FilePermissionDialog` 及其 `useFilePermissionDialog`、`usePermissionHandler`
- `SkillPermissionRequest`
- `FallbackPermissionRequest`
- `permissions/hooks.ts`、`permissions/utils.ts`

## 关键代码路径与文件引用

- **主实现**：`src/utils/unaryLogging.ts`（39 行）
- **Analytics 核心**：`src/services/analytics/index.ts`（`logEvent`）
- **调用方（文件权限）**：`src/components/permissions/FilePermissionDialog/`、`src/components/permissions/hooks.ts`
- **调用方（技能权限）**：`src/components/permissions/SkillPermissionRequest/SkillPermissionRequest.tsx`
- **调用方（兜底权限）**：`src/components/permissions/FallbackPermissionRequest.tsx`

## 依赖与外部交互

- **`src/services/analytics/index.js`**：`logEvent` 与 analytics 元数据类型。
- 无外部 npm 依赖。

## 风险、边界与改进建议

### 风险

1. **未捕获的 Promise 异常**：`await event.metadata.language_name` 若被拒绝（reject），会导致 `logUnaryEvent` 抛出异常。虽然该函数通常由事件处理器 fire-and-forget 调用，但若调用方使用了 `await logUnaryEvent(...)` 且未包裹 `try/catch`，可能中断上层逻辑（如权限对话框的状态更新）。
2. **类型覆盖面不足**：`CompletionType` 仅枚举了 4 种操作。随着新工具（如 NotebookEdit、MCP 工具自定义操作）的引入，可能需要不断扩展该联合类型，否则这些新操作的 analytics 会被归入模糊的 `tool_use_single` 或完全无法记录。
3. **无采样或去重**：每次用户交互都会独立发送事件。若用户快速连续点击接受/拒绝，可能产生大量重复事件，增加 analytics 后端压力。

### 边界

- **仅记录用户决策**：模块本身不决定用户是否接受或拒绝，只负责在决策发生后格式化并上报。
- **语言名称可延迟解析**：这是该模块与大多数 analytics 封装不同的地方——它主动 `await` 一个可能为 Promise 的字段，简化了调用方的生命周期管理。
- **无本地持久化**：事件直接通过 `logEvent` 发往内存中的 analytics provider，若进程在 `await` 期间崩溃，事件可能丢失。

### 改进建议

1. **增加异常兜底**：在 `await event.metadata.language_name` 外包裹 `try/catch`，若 Promise 失败则将 `language_name` 设为 `'unknown'` 而不是抛出，保证日志记录不中断主流程。
2. **扩展 CompletionType 或泛化**：考虑将 `CompletionType` 扩展为包含更多编辑工具（`notebook_edit_single`、`multi_edit_single`），或改为字符串类型并在调用层通过常量约束，减少类型变更频率。
3. **引入去重窗口**：在模块内维护一个最近已发送事件的 `Set`（如 1 秒窗口），对相同的 `(message_id, completion_type, event)` 组合跳过重复发送，避免快速连击导致的数据噪声。
4. **增加耗时指标**：在 `metadata` 中补充 `decisionDurationMs`（从请求展示到用户决策的时间），帮助分析权限请求的易理解性。
