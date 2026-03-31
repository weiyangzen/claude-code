# 研究文档：src/commands/permissions/index.ts

## 场景与职责

`src/commands/permissions/index.ts` 是 `/permissions`（别名 `/allowed-tools`）斜杠命令的**命令清单（Command Manifest）**。它的唯一职责是将该命令注册进 Claude Code 的全局命令体系，使用户在 REPL 中输入 `/permissions` 或 `/allowed-tools` 时，系统能够识别并懒加载对应的交互式 UI 实现。

该文件本身不包含任何业务逻辑或 UI 渲染代码，仅作为命令的元数据入口点（metadata entrypoint）。它声明了命令的类型为 `local-jsx`，这意味着该命令会渲染一个 Ink/React 终端 UI，而不是直接返回文本或 prompt 扩展。

## 功能点目的

1. **命令注册**：向 `src/commands.ts` 中的内置命令列表注册 `permissions` 命令，使其出现在自动补全、帮助文档和斜杠命令解析器中。
2. **懒加载隔离**：通过 `load: () => import('./permissions.js')` 实现按需加载。Permission 管理 UI 依赖大量 React 组件和权限工具函数，懒加载可以避免在应用启动时引入这些重依赖。
3. **别名支持**：提供 `allowed-tools` 别名，兼容用户可能使用的旧称或更直观的叫法。

## 具体技术实现

### 关键流程

```ts
import type { Command } from '../../commands.js'

const permissions = {
  type: 'local-jsx',
  name: 'permissions',
  aliases: ['allowed-tools'],
  description: 'Manage allow & deny tool permission rules',
  load: () => import('./permissions.js'),
} satisfies Command

export default permissions
```

- **`type: 'local-jsx'`**：标识这是一个本地 JSX 命令。`processSlashCommand.tsx` 在处理此类命令时，会调用 `load()` 获取模块，然后执行其导出的 `call` 函数，并将返回值（React 节点）交给 Ink 渲染。
- **`satisfies Command`**：TypeScript 类型约束，确保对象符合 `Command` 联合类型中的 `LocalJSXCommand` 分支。
- **`load: () => import('./permissions.js')`**：使用动态 import 实现懒加载。注意导入路径使用 `.js` 扩展名，这是项目 TypeScript/ESM 配置下的惯例（编译后保持 ESM 兼容性）。

### 数据结构

该文件导出的对象符合 `CommandBase & LocalJSXCommand` 结构：

| 字段 | 值 | 说明 |
|------|-----|------|
| `type` | `'local-jsx'` | 命令类型 |
| `name` | `'permissions'` | 主命令名 |
| `aliases` | `['allowed-tools']` | 别名数组 |
| `description` | `'Manage allow & deny tool permission rules'` | 帮助文本和自动补全描述 |
| `load` | `() => Promise<LocalJSXCommandModule>` | 懒加载工厂函数 |

## 关键代码路径与文件引用

### 上游调用方

- **`src/commands.ts`**（第 126 行导入，第 331 行注入 `COMMANDS` 数组）：
  ```ts
  import permissions from './commands/permissions/index.js'
  ```
  在 `COMMANDS` memoized 数组中，`permissions` 被排在 `thinkbackPlay` 之后、`plan` 之前。这决定了它在 `/` 自动补全列表中的出现顺序。

- **`src/utils/processUserInput/processSlashCommand.tsx`**：
  当用户输入 `/permissions` 时，斜杠命令解析器通过 `findCommand()` 在 `getCommands()` 返回的列表中定位到该命令对象，随后根据 `type === 'local-jsx'` 进入 JSX 命令执行分支，调用 `command.load()` 获取模块并执行 `call()`。

### 下游被调用方

- **`src/commands/permissions/permissions.tsx`**：
  `load()` 动态导入的目标文件，包含实际的 `LocalJSXCommandCall` 实现和 `PermissionRuleList` UI 渲染。

## 依赖与外部交互

| 依赖 | 路径 | 作用 |
|------|------|------|
| `Command` 类型 | `../../commands.js` | 提供命令对象的 TypeScript 类型定义 |
| `permissions.tsx` | `./permissions.js`（运行时） | 实际的命令执行与 UI 逻辑 |

该文件**零运行时外部副作用**，不依赖 React、不读写状态、不发起网络请求，是一个纯元数据模块。

## 风险、边界与改进建议

### 风险与边界

1. **Bridge/Remote 模式不可用**：由于 `type === 'local-jsx'`，该命令在 `isBridgeSafeCommand()` 判断中会被明确返回 `false`（见 `src/commands.ts` 第 673 行）。这意味着通过 Remote Control bridge（手机/网页客户端）无法调用 `/permissions`，这是设计上的安全边界，因为该命令渲染的是交互式 TUI，不适合纯文本流式响应。
2. **非 REMOTE_SAFE_COMMANDS**：在 `--remote` 模式下，`filterCommandsForRemoteMode()` 会过滤掉不在 `REMOTE_SAFE_COMMANDS` 白名单中的命令，`permissions` 不在白名单中，因此远程模式下该命令对用户不可见。
3. **无可用性限制（availability）**：该命令未设置 `availability` 字段，因此对所有用户可见（包括 Bedrock/Vertex/Foundry 等第三方服务用户和 Console API 用户）。

### 改进建议

1. **测试覆盖**：目前项目中未找到针对 `permissions` 命令的单元测试或集成测试。建议至少添加：
   - 命令注册测试（验证 `COMMANDS()` 包含 `permissions` 且 `isCommandEnabled` 为 true）。
   - 懒加载测试（验证 `load()` 能正确解析到 `permissions.tsx` 模块）。
2. **文档同步**：README 或用户文档中若提到 `/allowed-tools` 旧别名，应确保与 `aliases` 数组保持一致。
3. **类型安全**：当前使用 `satisfies Command` 已足够，但如果未来需要区分 `local-jsx` 与 `local` 命令的特定字段，可考虑使用更精确的类型断言。
