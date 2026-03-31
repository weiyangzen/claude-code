# 研究文档：src/commands/plan/index.ts

## 场景与职责

`src/commands/plan/index.ts` 是 Claude Code 中 `/plan` 斜杠命令的**注册入口文件**。它的职责非常单一：以符合项目命令系统规范的方式，声明并导出一个名为 `plan` 的内置命令对象。该命令属于 `local-jsx` 类型，意味着它会在本地终端中执行，并可以渲染 React/JSX UI（通过 Ink）。文件本身不包含任何业务逻辑，而是通过懒加载（lazy-load）将实际实现委托给同目录下的 `plan.tsx`。

## 功能点目的

1. **命令注册**：向全局命令表注册 `/plan` 命令，使其在 REPL 输入解析、自动补全、帮助文档中可见。
2. **懒加载优化**：通过 `load: () => import('./plan.js')` 延迟加载 `plan.tsx` 的实现，避免在应用启动时就将计划模式相关的 UI 组件和工具函数全部加载到内存中，从而缩短启动时间。
3. **元信息声明**：提供命令名称、描述、参数提示等用户-facing 的元数据。

## 具体技术实现

### 关键数据结构

```typescript
import type { Command } from '../../commands.js'

const plan = {
  type: 'local-jsx',
  name: 'plan',
  description: 'Enable plan mode or view the current session plan',
  argumentHint: '[open|<description>]',
  load: () => import('./plan.js'),
} satisfies Command
```

- `type: 'local-jsx'`：表明该命令的执行结果可以在终端中渲染 React 节点（区别于纯文本 `local` 或 prompt 扩展型 `prompt` 命令）。
- `satisfies Command`：使用 TypeScript 的 `satisfies` 运算符进行类型检查，确保对象结构符合 `Command` 联合类型，同时保留对象字面量的具体类型推断。
- `load` 函数：返回 `Promise<LocalJSXCommandModule>`，模块预期导出 `call` 函数，签名如下：
  ```typescript
  type LocalJSXCommandCall = (
    onDone: LocalJSXCommandOnDone,
    context: LocalJSXCommandContext,
    args: string,
  ) => Promise<React.ReactNode>
  ```

### 编译产物引用

注意 `load` 中导入的是 `./plan.js`，而非 `./plan.tsx`。这是 TypeScript/Bun 构建后的产物引用约定：源码使用 `.ts/.tsx`，运行时/构建后使用 `.js`。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/commands.ts` | 导入本文件 (`import plan from './commands/plan/index.js'`)，并将其加入 `COMMANDS` 数组与 `REMOTE_SAFE_COMMANDS` 集合中。 |
| `src/commands/plan/plan.tsx` | 实际业务逻辑与 UI 渲染实现，由 `load()` 懒加载。 |
| `src/types/command.ts` | 定义 `Command`、`LocalJSXCommand` 等类型。 |

在 `src/commands.ts` 中，`plan` 被明确标记为 `REMOTE_SAFE_COMMANDS` 之一，说明在 `--remote` 远程模式下，用户仍然可以执行 `/plan` 命令来切换计划模式或查看计划。

## 依赖与外部交互

- **零运行时依赖**：本文件不依赖任何外部工具、文件系统或网络服务。
- **类型依赖**：仅依赖 `../../commands.js` 中的 `Command` 类型。
- **调用关系**：
  - **被调用方**：`src/commands.ts` 静态导入本模块。
  - **调用方**：本模块通过动态 `import()` 调用 `plan.tsx`。

## 风险、边界与改进建议

### 风险与边界

1. **构建一致性风险**：`load` 硬编码为 `./plan.js`。如果构建系统未正确将 `.tsx` 编译为 `.js`，或文件路径在打包后发生变化，动态导入会在运行时抛出 `Module not found` 错误。项目使用 Bun 打包，目前这是约定俗成的做法，但仍需确保构建流程与此假设一致。
2. **类型与运行时不完全对齐**：`satisfies Command` 保证了结构符合 `Command`，但 `load` 返回的模块内容（`call` 函数是否存在、签名是否正确）在编译期无法被本文件验证，错误会延迟到运行时暴露。

### 改进建议

1. **模块加载错误处理**：当前 `commands.ts` 的调用方（如 REPL 中的命令分发器）在调用 `load()` 时已有 `try/catch`，但本文件本身可以考虑在 `load` 失败时提供更友好的错误信息。
2. **无其他显著改进点**：该文件职责单一、代码极简，符合项目命令系统的统一模式，过度重构反而会增加不必要的复杂度。
