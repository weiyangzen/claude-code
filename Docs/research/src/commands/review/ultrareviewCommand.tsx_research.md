# ultrareviewCommand.tsx 深度研究文档

> 文件路径：`src/commands/review/ultrareviewCommand.tsx`  
> 文件大小：9,855 bytes（58 行，含 source map）  
> 研究日期：2026-04-01  
> 执行器：kimi (k2p5)

---

## 一、场景与职责

### 1.1 模块定位

`ultrareviewCommand.tsx` 是 Claude Code CLI 中 `/ultrareview` 命令的 **LocalJSX 命令入口模块**。它作为命令调度层与远程执行层之间的桥梁，负责：

1. 接收用户输入和 CLI 上下文
2. 调用计费门控检查
3. 根据门控结果决定是直接启动、显示确认对话框，还是返回错误提示
4. 将远程审查的启动结果回写到本地对话流

### 1.2 核心职责

| 职责 | 说明 |
|------|------|
| **命令入口** | 实现 `LocalJSXCommandCall` 接口，作为 `/ultrareview` 的懒加载模块 |
| **计费门控调度** | 调用 `checkOverageGate()` 并根据返回状态分发处理 |
| **对话框渲染** | 在需要用户确认时返回 `UltrareviewOverageDialog` JSX |
| **结果回写** | 通过 `onDone` 回调将启动成功/失败消息注入对话 |
| **取消保护** | 处理用户按 Escape 取消时的状态清理 |

### 1.3 调用场景

该模块通过 `src/commands/review.ts` 中的懒加载配置被调用：

```typescript
const ultrareview: Command = {
  type: 'local-jsx',
  name: 'ultrareview',
  description: `~10–20 min · Finds and verifies bugs in your branch. Runs in Claude Code on the web. See ${CCR_TERMS_URL}`,
  isEnabled: () => isUltrareviewEnabled(),
  load: () => import('./review/ultrareviewCommand.js'),
}
```

---

## 二、功能点目的

### 2.1 为什么使用 LocalJSX 类型

`/ultrareview` 被定义为 `local-jsx` 类型而非 `prompt` 或 `local` 类型，原因是：

- **需要交互式 UI**：免费额度耗尽时必须显示确认对话框
- **需要异步状态管理**：启动过程涉及网络请求、配额检查、会话创建
- **需要条件渲染**：根据 `checkOverageGate()` 的结果动态决定渲染内容

### 2.2 功能矩阵

| 函数 | 签名 | 目的 |
|------|------|------|
| `contentBlocksToString` | `(blocks) => string` | 将 Anthropic SDK 的 `ContentBlockParam[]` 转换为纯文本 |
| `launchAndDone` | `(args, context, onDone, billingNote, signal?) => Promise<void>` | 调用 `launchRemoteReview` 并将结果通过 `onDone` 回写 |
| `call` | `LocalJSXCommandCall` | 命令主入口，处理门控和分发 |

---

## 三、具体技术实现

### 3.1 类型与接口

```typescript
// src/types/command.ts:131-135
export type LocalJSXCommandCall = (
  onDone: LocalJSXCommandOnDone,
  context: ToolUseContext & LocalJSXCommandContext,
  args: string,
) => Promise<React.ReactNode>
```

```typescript
// src/types/command.ts:117-126
export type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: CommandResultDisplay
    shouldQuery?: boolean
    metaMessages?: string[]
    nextInput?: string
    submitNextInput?: boolean
  },
) => void
```

### 3.2 contentBlocksToString

```typescript
function contentBlocksToString(blocks: ContentBlockParam[]): string {
  return blocks
    .map(b => (b.type === 'text' ? b.text : ''))
    .filter(Boolean)
    .join('\n')
}
```

- 将 `launchRemoteReview` 返回的 `ContentBlockParam[]` 转换为可显示的字符串
- 仅提取 `type === 'text'` 的块，其他类型（如 `image`）被忽略
- `.filter(Boolean)` 移除空字符串

### 3.3 launchAndDone

```typescript
async function launchAndDone(
  args: string,
  context: Parameters<LocalJSXCommandCall>[1],
  onDone: LocalJSXCommandOnDone,
  billingNote: string,
  signal?: AbortSignal,
): Promise<void> {
  const result = await launchRemoteReview(args, context, billingNote)

  // 用户按 Escape 取消 → 跳过 onDone（避免写入已卸载的 transcript slot）
  if (signal?.aborted) return

  if (result) {
    onDone(contentBlocksToString(result), { shouldQuery: true })
  } else {
    onDone(
      'Ultrareview failed to launch the remote session. Check that this is a GitHub repo and try again.',
      { display: 'system' }
    )
  }
}
```

**设计要点**：

1. **取消保护**（line 11-14）：
   - 注释说明："User hit Escape during the ~5s launch — the dialog already showed 'cancelled' and unmounted, so skip onDone (would write to a dead transcript slot)"
   - 这是防止 React 组件卸载后仍调用状态更新导致的内存泄漏或异常

2. **结果处理**（line 15-18）：
   - `result` 为真值时，转换为字符串并设置 `shouldQuery: true`
   - `shouldQuery: true` 表示命令执行完毕后，CLI 应继续向模型发送消息，让模型向用户汇报启动结果

3. **失败兜底**（line 19-25）：
   - `result` 为 `null` 时，显示系统级错误消息（`display: 'system'`）
   - 错误场景：PR 模式下的 teleport 失败、非 GitHub 仓库等

### 3.4 call — 命令主入口

```typescript
export const call: LocalJSXCommandCall = async (onDone, context, args) => {
  const gate = await checkOverageGate()

  // 1. Extra Usage 未启用
  if (gate.kind === 'not-enabled') {
    onDone(
      'Free ultrareviews used. Enable Extra Usage at https://claude.ai/settings/billing to continue.',
      { display: 'system' }
    )
    return null
  }

  // 2. 余额不足
  if (gate.kind === 'low-balance') {
    onDone(
      `Balance too low to launch ultrareview ($${gate.available.toFixed(2)} available, $10 minimum). Top up at https://claude.ai/settings/billing`,
      { display: 'system' }
    )
    return null
  }

  // 3. 需要用户确认计费
  if (gate.kind === 'needs-confirm') {
    return (
      <UltrareviewOverageDialog
        onProceed={async signal => {
          await launchAndDone(
            args,
            context,
            onDone,
            ' This review bills as Extra Usage.',
            signal,
          )
          // 仅在非取消情况下持久化确认状态
          if (!signal.aborted) confirmOverage()
        }}
        onCancel={() =>
          onDone('Ultrareview cancelled.', { display: 'system' })
        }
      />
    )
  }

  // 4. 直接启动（proceed）
  await launchAndDone(args, context, onDone, gate.billingNote)
  return null
}
```

**状态机设计**：

```
checkOverageGate()
    │
    ├── not-enabled ──→ 系统错误消息 ──→ return null
    ├── low-balance ──→ 系统错误消息 ──→ return null
    ├── needs-confirm ──→ 返回 <UltrareviewOverageDialog />
    └── proceed ──→ launchAndDone() ──→ return null
```

**关键设计决策**：

- `confirmOverage()` 只在 `!signal.aborted` 时调用：
  - 如果用户在启动过程中按 Escape 取消，不应将 `sessionOverageConfirmed` 设为 true
  - 否则下次调用会跳过确认对话框，造成未经同意的计费

- 所有错误消息都使用 `display: 'system'`：
  - 系统消息以灰色/特殊样式显示，不触发模型查询
  - 成功启动时使用 `shouldQuery: true`，让模型主动向用户说明

---

## 四、关键代码路径与文件引用

### 4.1 调用链路

```
用户输入 /ultrareview [args]
    │
    ├──→ src/commands.ts
    │    └── import review, { ultrareview } from './commands/review.js'
    │
    ├──→ src/commands/review.ts:48-54
    │    └── ultrareview: Command { type: 'local-jsx', load: () => import('./review/ultrareviewCommand.js') }
    │
    ├──→ src/commands/review/ultrareviewCommand.tsx:28
    │    └── export const call: LocalJSXCommandCall = async (onDone, context, args) => { ... }
    │
    ├──→ src/commands/review/reviewRemote.ts
    │    ├── checkOverageGate() ──→ line 52
    │    ├── confirmOverage() ──→ line 38
    │    └── launchRemoteReview() ──→ line 128
    │
    ├──→ src/commands/review/UltrareviewOverageDialog.tsx
    │    └── <UltrareviewOverageDialog />
    │
    └──→ src/tasks/RemoteAgentTask/RemoteAgentTask.tsx
         └── registerRemoteAgentTask()
```

### 4.2 核心文件清单

| 文件路径 | 行数 | 职责 |
|----------|------|------|
| `src/commands/review/ultrareviewCommand.tsx` | 58 | **本文件**：LocalJSX 命令入口 |
| `src/commands/review.ts` | 57 | 命令注册表，定义 `ultrareview` 命令元数据 |
| `src/commands/review/reviewRemote.ts` | 316 | 远程审查核心逻辑 |
| `src/commands/review/UltrareviewOverageDialog.tsx` | 96 | Extra Usage 确认对话框 |
| `src/types/command.ts` | 216 | `LocalJSXCommandCall`、`LocalJSXCommandOnDone` 类型定义 |

### 4.3 代码位置速查

| 元素 | 行号 |
|------|------|
| `contentBlocksToString` | 6-8 |
| `launchAndDone` | 9-27 |
| `call` | 28-57 |
| `not-enabled` 处理 | 30-34 |
| `low-balance` 处理 | 36-40 |
| `needs-confirm` 处理 | 42-51 |
| `proceed` 处理 | 54-55 |

---

## 五、依赖与外部交互

### 5.1 内部依赖图谱

```
ultrareviewCommand.tsx
├── @anthropic-ai/sdk/resources/messages.js
│   └── ContentBlockParam
├── react
│   └── React (用于 JSX）
├── ../../types/command.js
│   ├── LocalJSXCommandCall
│   └── LocalJSXCommandOnDone
├── ./reviewRemote.js
│   ├── checkOverageGate
│   ├── confirmOverage
│   └── launchRemoteReview
└── ./UltrareviewOverageDialog.js
    └── UltrareviewOverageDialog
```

### 5.2 无直接外部 API 交互

本模块不直接调用任何外部 API，所有网络请求均通过 `reviewRemote.ts` 间接完成：
- `checkOverageGate()` → `fetchUltrareviewQuota()` / `fetchUtilization()`
- `launchRemoteReview()` → `teleportToRemote()` → `POST /v1/sessions`

### 5.3 与命令框架的交互

`call` 函数的返回值决定了 CLI 的行为：

| 返回值 | CLI 行为 |
|--------|----------|
| `null` | 命令执行完毕，无额外 UI，根据 `onDone` 参数决定是否查询模型 |
| `<UltrareviewOverageDialog />` | 渲染 JSX 组件到终端，等待用户交互 |

---

## 六、风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 `launchRemoteReview` 返回 `null` 时的错误信息不够精确

```typescript
if (result) {
  onDone(contentBlocksToString(result), { shouldQuery: true })
} else {
  onDone(
    'Ultrareview failed to launch the remote session. Check that this is a GitHub repo and try again.',
    { display: 'system' }
  )
}
```

- **问题**：`reviewRemote.ts` 中 PR 模式非 GitHub 仓库返回 `null`，但分支模式 bundle 失败也返回 `ContentBlockParam[]`（有明确错误文本）
- **现状**：`null` 的兜底消息假设原因是 "not a GitHub repo"，但未来可能还有其他 `null` 场景
- **建议**：在 `reviewRemote.ts` 中统一返回 `ContentBlockParam[]`，消除 `null` 的歧义

#### 6.1.2 `contentBlocksToString` 忽略非文本块

```typescript
.map(b => (b.type === 'text' ? b.text : ''))
```

- 如果 `launchRemoteReview` 未来返回 `image` 块，该函数会静默丢弃
- **建议**：增加断言或日志，发现非预期块类型时报警

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| `args` 为空 | 进入分支模式，由 `reviewRemote.ts` 处理 |
| `args` 为纯数字 | 进入 PR 模式，由 `reviewRemote.ts` 处理 |
| 用户连续快速输入 `/ultrareview` | 第一次确认后 `sessionOverageConfirmed` 为 true，后续直接启动 |
| 对话框显示后用户按 Escape | `signal.aborted` 为 true，`launchAndDone` 提前返回，不调用 `confirmOverage()` |
| `launchRemoteReview` 抛出异常 | `UltrareviewOverageDialog` 的 `.catch()` 重置 `isLaunching`，允许重试 |

### 6.3 改进建议

#### 6.3.1 错误信息精确化

```typescript
// 建议修改
} else {
  // result 为 null 时，reviewRemote.ts 应提供具体原因
  // 或在此处根据 args 和上下文推断更精确的错误提示
  onDone(
    'Ultrareview failed to launch. Common causes: not a GitHub repo (required for PR mode), repo too large for bundle mode, or network issues.',
    { display: 'system' }
  )
}
```

#### 6.3.2 增加重试机制

当前失败后需要用户手动重新输入 `/ultrareview`。可考虑：
- 在 `launchAndDone` 中捕获特定可恢复错误（如网络超时）
- 自动重试 1-2 次，减少用户操作

#### 6.3.3 类型安全增强

```typescript
// 建议为 contentBlocksToString 增加更严格的类型
function contentBlocksToString(blocks: Array<{ type: 'text'; text: string }>): string {
  return blocks.map(b => b.text).join('\n')
}
```

或在调用处使用类型断言并增加运行时检查。

#### 6.3.4 支持命令参数提示

当前 `src/commands/review.ts` 中 `ultrareview` 命令没有 `argumentHint`。建议增加：

```typescript
const ultrareview: Command = {
  // ...
  argumentHint: '[PR number]',
}
```

这样用户在输入 `/ultrareview ` 后，自动补全会显示参数提示。

#### 6.3.5 测试覆盖建议

| 测试场景 | 验证点 |
|----------|--------|
| `gate.kind === 'proceed'` | 直接调用 `launchAndDone`，返回 `null` |
| `gate.kind === 'needs-confirm'` | 返回 `UltrareviewOverageDialog` JSX |
| 用户确认后启动成功 | `onDone` 被调用，`shouldQuery: true` |
| 用户确认后按 Escape | `signal.aborted` 为 true，`confirmOverage` 不被调用 |
| `launchRemoteReview` 返回 `null` | 显示系统错误消息 |
