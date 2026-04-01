# constants.ts 研究文档

## 场景与职责

`constants.ts` 是 `ExitWorktreeTool` 模块的常量定义文件，职责单一且明确：导出工具的规范名称字符串 `EXIT_WORKTREE_TOOL_NAME`。该常量作为工具在系统中的唯一标识符，被用于工具注册、常量聚合、权限规则匹配以及异步 agent 的工具白名单等场景。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `EXIT_WORKTREE_TOOL_NAME` | 提供单一可信来源（Single Source of Truth）的字符串常量 `'ExitWorktree'`，避免工具名称在多个文件中以魔法字符串形式散落 |

## 具体技术实现

### 代码内容

```ts
export const EXIT_WORKTREE_TOOL_NAME = 'ExitWorktree'
```

- 仅一行导出，无运行时副作用
- 采用 `SCREAMING_SNAKE_CASE` 命名，符合项目常量命名规范
- 字符串值 `'ExitWorktree'` 与 `EnterWorktreeTool` 的 `ENTER_WORKTREE_TOOL_NAME = 'EnterWorktree'` 形成对称命名

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/tools/ExitWorktreeTool/constants.ts` | 本文件，定义并导出 `EXIT_WORKTREE_TOOL_NAME` |
| `src/tools/ExitWorktreeTool/ExitWorktreeTool.ts` | 导入 `EXIT_WORKTREE_TOOL_NAME` 作为 `buildTool({ name: EXIT_WORKTREE_TOOL_NAME })` |
| `src/constants/tools.ts` | 导入 `EXIT_WORKTREE_TOOL_NAME` 并加入 `ASYNC_AGENT_ALLOWED_TOOLS` 集合，允许异步 agent 调用 |
| `src/tools.ts` | 通过 `ExitWorktreeTool.name`（即本常量）间接参与工具池组装与 deny-rule 过滤 |

## 依赖与外部交互

### 与工具注册系统的关系

在 `src/tools.ts` 的 `getAllBaseTools()` 中：

```ts
import { ExitWorktreeTool } from './tools/ExitWorktreeTool/ExitWorktreeTool.js'
// ...
...(isWorktreeModeEnabled() ? [EnterWorktreeTool, ExitWorktreeTool] : []),
```

`ExitWorktreeTool` 对象的 `name` 属性即来源于本常量。该名称决定了：
- 模型在 `tool_use` 中引用的工具名
- 权限系统（`permissions.ts`）中的规则匹配键
- 工具搜索（`ToolSearchTool`）的索引关键字
- 分析日志（analytics）中的工具标识

### 与异步 Agent 白名单的关系

在 `src/constants/tools.ts` 中：

```ts
export const ASYNC_AGENT_ALLOWED_TOOLS = new Set([
  // ...
  ENTER_WORKTREE_TOOL_NAME,
  EXIT_WORKTREE_TOOL_NAME,
])
```

这意味着异步 agent（后台子代理）被允许调用 `ExitWorktree`。考虑到 worktree 操作涉及全局 CWD 和会话状态，这一权限需要谨慎评估（见风险部分）。

## 风险、边界与改进建议

### 风险与边界

1. **名称变更的级联影响**
   - 虽然本常量提供了单一来源，但如果修改 `'ExitWorktree'` 的值，会波及权限规则、持久化 transcript 中的历史 `tool_use` 记录、以及可能的外部 SDK 集成。此类变更属于破坏性变更（breaking change）。

2. **常量文件过于单薄**
   - 当前文件仅包含一个字符串常量。虽然职责单一，但与 `EnterWorktreeTool/constants.ts` 等文件相比，未能集中更多与工具相关的常量（如默认参数、最大长度限制等），导致相关常量分散在其他实现文件中。

3. **无类型安全约束**
   - 常量的类型为 `string` 而非字面量类型 `'ExitWorktree'`，这意味着任何接受 `string` 的函数都不会在编译期阻止错误的传入。虽然这在 JavaScript/TypeScript 项目中很常见，但如果配合 `as const` 可以获得更强的类型安全。

### 改进建议

1. **使用 `as const` 增强类型安全**
   ```ts
   export const EXIT_WORKTREE_TOOL_NAME = 'ExitWorktree' as const
   ```
   这样导出类型为字面量 `'ExitWorktree'`，在需要严格工具名称类型的场景（如配置对象键、联合类型）中可提供编译期检查。

2. **考虑合并对称常量**
   - 如果项目风格允许，可以将 `ENTER_WORKTREE_TOOL_NAME` 和 `EXIT_WORKTREE_TOOL_NAME` 放在同一个 `worktreeConstants.ts` 中，减少文件数量。不过当前"每个工具一个 constants.ts"的风格也具有良好的局部性和可维护性。

3. **补充工具元信息常量（可选）**
   - 若未来需要为 `ExitWorktreeTool` 增加版本化或别名支持，可以在本文件中补充：
     ```ts
     export const EXIT_WORKTREE_TOOL_ALIASES = [] as const
     export const EXIT_WORKTREE_TOOL_VERSION = '1' as const
     ```
