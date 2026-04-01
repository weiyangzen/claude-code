# src/utils/claudeCodeHints.ts 深入研究

## 场景与职责

`claudeCodeHints.ts` 实现了 **Claude Code Hints 协议**的解析器与状态存储。该协议允许在 Claude Code 下运行的 CLI 或 SDK 向 stderr 输出自关闭标签 `<claude-code-hint />`，Claude Code 的 shell 工具（Bash/PowerShell）捕获并剥离这些标签后，向用户展示安装提示（当前仅支持 plugin 类型提示）。

核心职责：
- **解析**：从 shell 输出中扫描、提取并剥离 hint 标签。
- **状态管理**：维护一个单槽（single-slot）的 pending hint 存储，供 React UI 订阅并展示提示对话框。
- **会话控制**：保证每个会话最多只展示一次 hint 提示，避免频繁打扰用户。

## 功能点目的

| 功能 | 目的 |
|------|------|
| `extractClaudeCodeHints(output, command)` | 扫描命令输出，返回解析出的 hints 列表和剥离标签后的干净输出 |
| `setPendingHint(hint)` | 将解析出的 hint 写入 pending 槽（受 gate 控制，每个会话最多一次） |
| `clearPendingHint()` | 清除 pending 槽（用户拒绝 hint 时使用） |
| `markShownThisSession()` | 标记当前会话已展示过 hint 对话框，后续 `setPendingHint` 变为 no-op |
| `getPendingHintSnapshot()` | 供 `useSyncExternalStore` 读取当前 pending hint |
| `subscribeToPendingHint` | 供 React 订阅 pending hint 变化 |
| `hasShownHintThisSession()` | 查询会话展示状态 |
| `_resetClaudeCodeHintStore()` | 测试专用重置函数 |

## 具体技术实现

### 标签解析
#### 快速路径
```ts
if (!output.includes('<claude-code-hint')) {
  return { hints: [], stripped: output }
}
```
- 绝大多数输出不含 hint，通过 `includes` 快速短路，避免正则开销与内存分配。

#### 正则匹配
```ts
const HINT_TAG_RE = /^[ \t]*<claude-code-hint\s+([^>]*?)\s*\/>[ \t]*$/gm
```
- 使用 `^...$` 锚定整行（`m` 多行模式），防止日志语句中**引用** hint 标签的行被误匹配。
- 允许行首行尾空白（兼容某些 SDK 对 stderr 的填充）。

#### 属性解析
```ts
const ATTR_RE = /(\w+)=(?:"([^"]*)"|([^\s/>]+))/g
```
- 支持 `key="value"` 和 `key=value` 两种形式。
- 不支持转义序列（注释说明：若需要则提升 spec version）。

#### 过滤与验证
- `v` 必须在 `SUPPORTED_VERSIONS`（当前仅 `{1}`）中。
- `type` 必须在 `SUPPORTED_TYPES`（当前仅 `{'plugin'}`）中。
- `value` 不能为空。
- 不支持的 hint 被丢弃并记录 `logForDebugging`。

#### 空白折叠
```ts
const collapsed = stripped.replace(/\n{3,}/g, '\n\n')
```
- 删除 hint 行后可能留下多余的空行，将连续 3 个及以上换行折叠为 2 个，保持输出整洁。

### Pending Hint 存储
- **单槽设计**：`let pendingHint: ClaudeCodeHint | null = null`
- **会话级限制**：`let shownThisSession = false`
- 一旦 `shownThisSession = true`，后续所有 `setPendingHint` 直接忽略。
- 基于 `createSignal` 实现订阅/通知。

### Hint 数据结构
```ts
type ClaudeCodeHint = {
  v: number        // 协议版本
  type: 'plugin'   // hint 类型
  value: string    // plugin slug，如 "eslint@marketplace"
  sourceCommand: string // 产生该 hint 的命令首 token
}
```

## 关键代码路径与文件引用

```
src/tools/BashTool/BashTool.tsx
  └── extractClaudeCodeHints(output, command)
      [bash 工具执行后解析输出中的 hint 标签]

src/tools/PowerShellTool/PowerShellTool.tsx
  └── extractClaudeCodeHints(output, command)
      [PowerShell 工具执行后解析输出中的 hint 标签]

src/utils/plugins/hintRecommendation.ts
  └── setPendingHint, hasShownHintThisSession
      [plugin hint 的 gate 逻辑：检查是否已安装、是否官方 marketplace 等]

src/hooks/useClaudeCodeHintRecommendation.tsx
  └── clearPendingHint, getPendingHintSnapshot, markShownThisSession, subscribeToPendingHint
      [React Hook 层：订阅 pending hint 并触发 UI 对话框]
```

### 依赖模块
- `src/utils/debug.ts` — `logForDebugging`
- `src/utils/signal.ts` — `createSignal`

## 依赖与外部交互

| 外部实体 | 交互方式 | 说明 |
|---------|---------|------|
| Bash/PowerShell 工具输出 | `extractClaudeCodeHints` | 从 stdout+stderr 混合输出中扫描标签 |
| Plugin 推荐系统 | `hintRecommendation.ts` | 对 `type='plugin'` 的 hint 做前置过滤（marketplace 校验、已安装检查等） |
| React UI | `useSyncExternalStore` (via hook) | 展示 hint 安装提示对话框 |
| 调试日志 | `logForDebugging` | 记录丢弃的不支持 hint |

## 风险、边界与改进建议

### 风险
1. **正则性能**：`HINT_TAG_RE` 使用 `gm` 标志在完整输出上全局匹配；若恶意输出包含大量 `<claude-code-hint` 字符串（即使不匹配整行），快速路径会失效，正则回溯可能消耗较多 CPU。
2. **属性值无引号时的字符限制**：`ATTR_RE` 的无引号值形式 `[^\s/>]+` 不支持含空格的值；虽然当前 `plugin` 类型的 slug 不含空格，但未来扩展可能受限。
3. **单槽丢失信息**：若同一命令输出多个 hint，只有最后一个（或第一个，取决于 `setPendingHint` 的调用时机）能被展示，其余被静默丢弃。
4. **`sourceCommand` 解析简单**：仅取 `command.trim().split(/\s/)[0]`，对复杂 shell 结构（如 `cd /tmp && eslint`）可能记录 `cd` 而非实际产生 hint 的 `eslint`。

### 边界
- 当前 spec version 为 `1`，仅定义 `plugin` 类型。
- hint 标签必须独占一行；嵌入在普通日志行中的标签会被忽略（这是安全特性，防止误解析）。
- `shownThisSession` 是进程级状态，CLI 重启后会重置，符合“每个会话最多一次”的产品设计。
- 该模块**不直接操作 UI**，所有展示逻辑由 `useClaudeCodeHintRecommendation.tsx` 消费。

### 改进建议
1. **正则性能加固**：在快速路径中增加更严格的预筛（如检查 `/<claude-code-hint\s/`），进一步减少正则触发概率；或限制单次扫描的最大字符数。
2. **支持多 hint 队列**：将单槽扩展为有限队列（如最多保留 3 个），允许同一命令输出多个 plugin 推荐时依次展示。
3. **改进 `sourceCommand` 提取**：使用更智能的 shell 命令解析（如取最后一个 `&&`/`||`/`|` 后的第一个 token），更准确地标识产生 hint 的实际程序。
4. **属性转义支持**：若未来 hint payload 需要包含引号或特殊字符，提升 spec version 并引入 `"` 转义序列支持。
5. **增加 hint 来源白名单**：当前任何子进程输出都可 emit hint；可考虑增加受信任的命令白名单，降低恶意 CLI 滥用 hint 协议诱导安装插件的风险。
