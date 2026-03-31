# 研究文档：src/commands/help/index.ts

## 场景与职责

`src/commands/help/index.ts` 是 Claude Code 中 `/help` 命令的**注册入口与元数据定义文件**。它不承担任何 UI 渲染或交互逻辑，而是作为命令系统（command system）的"声明式配置"，向核心命令注册表描述：

- `help` 是一个**本地 JSX 命令**（`type: 'local-jsx'`），即在终端内直接渲染 React/Ink 组件的交互式弹窗；
- 命令名称为 `help`，用户通过输入 `/help` 触发；
- 描述为 `Show help and available commands`；
- 采用**懒加载**（`load: () => import('./help.js')`）策略，避免启动时拉取 `help.tsx` 及其依赖（`HelpV2` 组件树）的代码，降低首包体积。

该文件是命令发现（command discovery）链路中的静态元数据来源，被 `src/commands.ts` 在构建命令列表时直接静态导入。

## 功能点目的

1. **命令自描述**：为自动补全（autocomplete）、技能索引（skill index）、帮助列表提供名称、描述、类型等元数据。
2. **懒加载隔离**：通过 `load` 函数将命令实现与命令定义解耦，使 `src/commands.ts` 在模块初始化阶段只收集轻量元数据，实际 UI 逻辑在首次调用 `/help` 时才动态导入。
3. **类型安全**：使用 `satisfies Command`（来自 `src/commands.js` 的 `Command` 联合类型），确保该对象符合 `LocalJSXCommand` 分支的接口约束（`type`、`name`、`description`、`load` 必填）。

## 具体技术实现

### 数据结构

```ts
const help = {
  type: 'local-jsx',
  name: 'help',
  description: 'Show help and available commands',
  load: () => import('./help.js'),
} satisfies Command
```

- `type: 'local-jsx'`：标识该命令的调用签名为 `LocalJSXCommandCall`（见 `src/types/command.ts`），返回值是 `Promise<React.ReactNode>`，由 REPL 通过 `setToolJSX` 渲染到 TUI 中。
- `load`：返回 `Promise<LocalJSXCommandModule>`，模块必须导出 `call` 函数。此处指向同目录编译后的 `help.js`（源文件为 `help.tsx`）。
- `satisfies Command`：TypeScript 4.9+ 特性，既做类型检查，又保留对象字面量的精确推断（如 `type` 被窄化为字面量 `'local-jsx'`）。

### 关键流程

1. **启动阶段**：`src/commands.ts` 顶层 `import help from './commands/help/index.js'`，将 `help` 对象加入 `COMMANDS` 数组。
2. **命令发现**：`getCommands(cwd)` 被调用时，将 `help` 与技能（skills）、插件（plugins）、工作流（workflows）等来源的命令合并、去重、过滤（`meetsAvailabilityRequirement`、`isCommandEnabled`）。
3. **用户触发**：用户在 PromptInput 中输入 `/help` 或按 `?`（PromptInput 中硬编码的快捷方式）。
4. **执行阶段**：`processSlashCommand.tsx` 的 `getMessagesForSlashCommand` 匹配到 `command.type === 'local-jsx'`，调用 `command.load()` → 解析为 `mod.call(onDone, context, args)` → 渲染 `HelpV2`。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/commands/help/index.ts` | **本文件**：命令元数据与懒加载入口。 |
| `src/commands/help/help.tsx` | 命令实现，导出 `call` 函数并渲染 `HelpV2`。 |
| `src/commands.ts` | 静态导入 `help`，将其注册到内置命令列表 `COMMANDS` 中（第 23、281 行）。 |
| `src/types/command.ts` | 定义 `Command`、`LocalJSXCommand`、`LocalJSXCommandCall`、`LocalJSXCommandModule` 等类型。 |
| `src/utils/processUserInput/processSlashCommand.tsx` | `local-jsx` 分支的执行器，负责调用 `load()` 和 `mod.call()`，并通过 `setToolJSX` 将返回的 JSX 挂载到 REPL（第 551–655 行）。 |
| `src/screens/REPL.tsx` | 维护 `toolJSX` 状态，决定 local-jsx 命令的渲染位置（inline / bottom / modal）。 |
| `src/components/HelpV2/HelpV2.tsx` | `help.tsx` 导入的顶层组件，实际绘制帮助弹窗。 |

## 依赖与外部交互

### 上游依赖（调用方）

- **`src/commands.ts`**：唯一静态导入本文件的地方。`help` 被加入 `COMMANDS` memoized 数组，进而被 `getCommands()` 返回给 REPL、PromptInput、技能工具等消费者。
- **`processSlashCommand.tsx`**：运行时通过 `command.load()` 动态拉取本文件对应的 `./help.js` 模块。

### 下游依赖（被调用方）

- **`./help.js`**（源文件 `help.tsx`）：懒加载目标，导出 `call` 函数。
- **`HelpV2.tsx`**：由 `help.tsx` 导入，负责具体的帮助 UI。

### 横向交互

- **REMOTE_SAFE_COMMANDS**（`src/commands.ts` 第 619–637 行）：`help` 被显式标记为远程安全命令，意味着在 `--remote` 模式下仍然可用。
- **BRIDGE_SAFE_COMMANDS**：`help` 属于 `local-jsx` 类型，而 `isBridgeSafeCommand()` 对 `local-jsx` 统一返回 `false`，因此**不能**通过 Remote Control bridge（手机/网页客户端）执行。

## 风险、边界与改进建议

### 风险

1. **编译产物耦合**：`load: () => import('./help.js')` 硬编码了 `.js` 扩展名，依赖构建系统（Bun bundle）将 `help.tsx` 编译为 `help.js`。若构建配置变更导致输出路径或扩展名变化，懒加载会在运行时抛出模块找不到错误。
2. **无错误边界**：`index.ts` 本身没有 `catch` 或 fallback。如果 `help.js` 的动态导入失败（如磁盘损坏、打包遗漏），错误会在 `processSlashCommand.tsx` 的 `command.load().then(...)` 中被捕获，但用户只会看到空白或通用错误提示。
3. **无 `isEnabled` 门控**：`help` 命令没有设置 `isEnabled` 函数，默认始终启用。这在绝大多数场景是合理的，但若未来需要按环境（如纯 CI 非交互模式）隐藏帮助，需要在这里补充条件。

### 边界

- **非交互模式短路**：`processSlashCommand.tsx` 第 614–621 行会在 `isNonInteractiveSession` 为 `true` 时直接 `resolve({ messages: [], shouldQuery: false })`，不渲染任何 JSX。这意味着即使 `help` 命令被触发，非交互会话也不会真正执行 `load()` 后的渲染逻辑。
- **懒加载的时序**：`load()` 在每次 `/help` 被调用时都会重新 `import()`。虽然现代 ES Module 动态导入在第二次会命中缓存，但仍会创建一个新的 Promise 和模块命名空间对象。对于 `help` 这种轻量模块，开销可忽略。

### 改进建议

1. **统一扩展名抽象**：项目内大量 `load` 函数都硬编码 `.js`，可考虑在 `Command` 类型或构建层引入宏/别名，自动映射 `.ts`/`.tsx` → 编译产物，降低手动维护风险。
2. **补充 `immediate` 标记说明**：`help` 目前未设置 `immediate: true`，因此属于"非立即" local-jsx 命令。在 fullscreen 模式下，它会被渲染在 scrollable 区域（REPL.tsx 第 4599–4601 行注释）。若未来希望帮助弹窗像 `/btw` 一样悬浮在底部而不随聊天记录滚动，可在此文件增加 `immediate: true`，但需评估对焦点管理和滚动体验的副作用。
3. **增加测试覆盖**：目前未找到针对 `help` 命令的单元测试或集成测试。建议至少补充：
   - `index.ts` 的 `satisfies Command` 类型合规性测试；
   - `help.tsx` 中 `call` 函数是否正确传递 `commands` 和 `onDone` 的测试；
   - `HelpV2` 组件在接收到 `commands` 后是否正确分类（builtin / custom / ant-only）的快照测试。
