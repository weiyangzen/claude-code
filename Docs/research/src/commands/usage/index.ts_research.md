# src/commands/usage/index.ts 深度研究文档

## 场景与职责

### 1.1 功能定位
`src/commands/usage/index.ts` 是 Claude Code CLI 中 `/usage` 斜杠命令的**入口定义文件**。它不承担任何 UI 渲染或业务逻辑，而是作为命令系统的"注册卡片"，向核心命令注册表声明 `/usage` 命令的存在、类型、描述、可见性范围以及实现模块的懒加载方式。

### 1.2 使用场景
- **命令注册**：在 CLI 启动时，`src/commands.ts` 静态导入本文件，将 `usage` 命令纳入内置命令列表。
- **懒加载入口**：当用户在 REPL 中输入 `/usage` 时，命令执行器通过本文件声明的 `load` 函数动态导入真正的实现模块（`./usage.js`），避免启动时加载不必要的 React/JSX 运行时。
- **权限控制**：通过 `availability: ['claude-ai']` 明确限制该命令仅对 Claude.ai OAuth 订阅用户可见，对 Console API Key 用户、Bedrock/Vertex/Foundry 用户及未登录用户隐藏。

### 1.3 生命周期中的角色
```
CLI 启动 → commands.ts 导入 index.ts → 命令元数据进入内存
用户输入 /usage → 命令系统匹配 name='usage' → 调用 index.ts 中的 load()
                          ↓
                   懒加载 ./usage.js → 执行 call() 渲染 Settings 组件
```

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `type: 'local-jsx'` | 声明该命令为本地 JSX 命令，执行结果是一个 React 节点，由 Ink 渲染在终端内 |
| `name: 'usage'` | 用户在 REPL 中输入 `/usage` 时的匹配标识 |
| `description: 'Show plan usage limits'` | 在帮助系统、自动补全、命令列表中展示给用户的人类可读描述 |
| `availability: ['claude-ai']` | 访问控制：仅当 `isClaudeAISubscriber()` 返回 `true` 时，命令才会出现在可用命令列表中 |
| `load: () => import('./usage.js')` | 懒加载真正的命令实现，减少启动内存占用和初始化时间 |

---

## 具体技术实现

### 3.1 源码全文
```typescript
import type { Command } from '../../commands.js'

export default {
  type: 'local-jsx',
  name: 'usage',
  description: 'Show plan usage limits',
  availability: ['claude-ai'],
  load: () => import('./usage.js'),
} satisfies Command
```

### 3.2 类型系统解析
- **`satisfies Command`**：TypeScript 4.9+ 的 `satisfies` 运算符确保该对象符合 `Command` 联合类型，同时保留对象字面量的具体类型信息，使得 `load` 返回类型的推断更加精确。
- **`Command` 联合类型**（定义于 `src/types/command.ts`）：
  ```typescript
  export type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
  ```
  本文件使用的是 `LocalJSXCommand` 分支：
  ```typescript
  type LocalJSXCommand = {
    type: 'local-jsx'
    load: () => Promise<LocalJSXCommandModule>
  }
  ```
  其中 `LocalJSXCommandModule` 要求模块导出 `call: LocalJSXCommandCall`。

### 3.3 `availability` 过滤机制
`availability` 字段在 `src/commands.ts` 的 `meetsAvailabilityRequirement()` 函数中被消费：
```typescript
export function meetsAvailabilityRequirement(cmd: Command): boolean {
  if (!cmd.availability) return true
  for (const a of cmd.availability) {
    switch (a) {
      case 'claude-ai':
        if (isClaudeAISubscriber()) return true
        break
      // ...
    }
  }
  return false
}
```
对于 `usage` 命令，若用户未通过 Claude.ai OAuth 登录（或订阅状态不可用），`getCommands()` 会将其从返回列表中过滤掉，导致：
1. 自动补全中不显示 `/usage`
2. 直接输入 `/usage` 会触发 "Command usage not found"

### 3.4 懒加载实现细节
- `load` 返回 `import('./usage.js')`，这是一个动态 `import()` 表达式。
- 构建工具（如 Bun bundler）会将 `./usage.js` 及其依赖（React、Settings 组件等）拆分为独立的 chunk，实现按需加载。
- `./usage.js` 是 `./usage.tsx` 的编译产物；源码中写 `./usage.js` 是为了兼容 Node/Bun 的 ESM 解析规则（TypeScript 编译后保留 `.js` 扩展名）。

---

## 关键代码路径与文件引用

### 4.1 调用链（本文件视角）
```
src/commands.ts:56
    import usage from './commands/usage/index.js'
         ↓
src/commands.ts:317
    usage,  // 被放入 COMMANDS() 数组
         ↓
src/commands.ts:476 getCommands(cwd)
    经过 meetsAvailabilityRequirement(usage) 过滤
         ↓
用户输入 /usage → src/commands.ts:688 findCommand('usage', commands)
         ↓
匹配到 usage 对象 → 调用 usage.load()
         ↓
本文件: load: () => import('./usage.js')
         ↓
src/commands/usage/usage.tsx 被加载并执行 call()
```

### 4.2 核心文件引用

| 文件路径 | 与本文件关系 | 用途 |
|----------|-------------|------|
| `src/commands.ts` | **调用方/注册方** | 导入并注册 `usage` 命令；执行可用性过滤和命令查找 |
| `src/types/command.ts` | **类型依赖** | 提供 `Command`、`LocalJSXCommand` 等类型定义 |
| `src/commands/usage/usage.tsx` | **被懒加载的实现** | 提供 `LocalJSXCommandModule`（导出 `call` 函数） |
| `src/utils/auth.ts` | **间接依赖** | `isClaudeAISubscriber()` 决定本命令是否对用户可见 |

### 4.3 相关常量与集合
- `REMOTE_SAFE_COMMANDS`（`src/commands.ts:619`）：包含 `usage`，表示在 `--remote` 模式下该命令仍然可用，因为它只影响本地 TUI 状态，不依赖本地文件系统。
- `BRIDGE_SAFE_COMMANDS`（`src/commands.ts:651`）：**不包含** `usage`。因为 `usage` 是 `local-jsx` 类型，而 `isBridgeSafeCommand()` 对 `local-jsx` 统一返回 `false`，防止在 Remote Control bridge（手机/网页客户端）上渲染 Ink UI 导致问题。

---

## 依赖与外部交互

### 5.1 直接依赖
| 模块 | 导入内容 | 用途 |
|------|---------|------|
| `../../commands.js` | `Command` (type) | 类型约束 |

### 5.2 间接/运行时依赖
| 模块 | 作用 |
|------|------|
| `src/utils/auth.js` | `isClaudeAISubscriber()` 控制命令可见性 |
| `src/commands/usage/usage.js` | 动态导入，提供实际执行逻辑 |

### 5.3 无外部网络/IO 交互
本文件是纯元数据定义，不发起任何 HTTP 请求、不读写文件、不访问环境变量（除通过 `availability` 机制间接依赖运行时认证状态外）。

---

## 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 编译产物路径耦合
- **风险**：源码中硬编码 `import('./usage.js')`，依赖 TypeScript/Bun 编译后保留 `.js` 扩展名。若构建流程改变（如改为 `.jsx` 或引入 loader 别名），懒加载可能失败。
- **缓解**：当前项目统一使用 `.js` 扩展名进行 ESM 互操作，这是既定规范，风险可控。

#### 6.1.2 动态导入失败无本地兜底
- **风险**：若 `./usage.js` 在运行时缺失或损坏（如打包错误），`load()` 会抛出 `Error: Cannot find module`，导致用户输入 `/usage` 后 CLI 崩溃或显示未处理的异常。
- **代码位置**：本文件第 8 行 `load: () => import('./usage.js')` 没有 `catch` 或 fallback。
- **现状**：调用方（命令执行器）可能有全局错误处理，但本文件自身未做防御。

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| 非 Claude.ai 用户 | 命令在 `getCommands()` 阶段被过滤，用户看不到 `/usage` |
| 用户已登录但订阅状态变更 | `getCommands()` 每次调用都会重新评估 `availability`，状态变更会即时生效（因为 `meetsAvailabilityRequirement` 未被 memoize） |
| 动态导入时网络/磁盘异常 | `import('./usage.js')` 抛出异常，取决于上层调用者是否捕获 |

### 6.3 改进建议

#### 6.3.1 增加懒加载错误兜底
```typescript
load: () => import('./usage.js').catch(err => {
  // 返回一个显示友好错误信息的 fallback 模块
  return {
    call: async (onDone) => {
      const { Text } = await import('../../ink.js')
      onDone?.()
      return <Text color="error">Failed to load /usage command. Please try again.</Text>
    }
  } satisfies LocalJSXCommandModule
}),
```

#### 6.3.2 考虑添加 `isEnabled` 开关
当前 `usage` 命令没有 `isEnabled` 字段（默认始终启用）。如果未来需要按平台或 feature flag 灰度关闭，可添加：
```typescript
isEnabled: () => !process.env.DISABLE_USAGE_COMMAND,
```

#### 6.3.3 文档化 `availability` 扩展
如果未来 `/usage` 需要对 Console API Key 用户也可见（例如展示不同的限额信息），可将 `availability` 扩展为 `['claude-ai', 'console']`，但需同步修改 `usage.tsx` 和 API 层以适配不同身份来源的数据格式。

### 6.4 测试建议

| 测试场景 | 验证点 |
|----------|--------|
| 类型检查 | `tsc --noEmit` 确保 `satisfies Command` 通过 |
| 命令注册 | `getCommands()` 在 `isClaudeAISubscriber() === true` 时包含 `usage` |
| 命令隐藏 | `getCommands()` 在 `isClaudeAISubscriber() === false` 时不包含 `usage` |
| 懒加载 | 调用 `usage.load()` 成功返回包含 `call` 函数的模块 |
| 远程模式 | `REMOTE_SAFE_COMMANDS.has(usage)` 为 `true` |
| Bridge 安全 | `isBridgeSafeCommand(usage)` 为 `false`（因为是 `local-jsx`） |
