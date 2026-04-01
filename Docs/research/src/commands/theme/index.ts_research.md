# 研究文档：src/commands/theme/index.ts

## 场景与职责

`src/commands/theme/index.ts` 是 `/theme`  slash 命令的**注册入口与懒加载声明文件**。它的唯一职责是将 "theme" 命令以符合项目命令体系规范的方式暴露给命令注册表，使得用户在 REPL 中输入 `/theme` 时，系统能够定位并延迟加载该命令的 JSX 实现。该文件本身不包含任何 UI 或业务逻辑，属于命令的"元数据层"。

## 功能点目的

1. **命令注册**：声明一个 `type: 'local-jsx'` 的命令对象，名称为 `theme`，描述为 `Change the theme`。
2. **懒加载（Lazy Loading）**：通过 `load: () => import('./theme.js')` 将实际的 React/Ink 组件实现延迟到用户首次调用 `/theme` 时才加载，避免启动时引入不必要的依赖。
3. **类型安全**：使用 TypeScript 的 `satisfies Command` 约束，确保命令对象结构符合 `src/types/command.ts` 中定义的 `Command` 联合类型，同时保留对象字面量的具体推断信息。

## 具体技术实现

### 关键数据结构

```typescript
const theme = {
  type: 'local-jsx',
  name: 'theme',
  description: 'Change the theme',
  load: () => import('./theme.js'),
} satisfies Command
```

- `type: 'local-jsx'`：表明该命令会在本地终端渲染 Ink/React UI，而非发送 prompt 给模型或返回纯文本。
- `load`：返回 `Promise<LocalJSXCommandModule>`，即 `theme.tsx` 中导出的 `{ call: LocalJSXCommandCall }`。

### 编译与导入路径

源码中使用 `.js` 扩展名（`./theme.js`），这是 TypeScript/Bun 项目的约定：源码写 `.js`，构建时解析到 `.ts`/`.tsx`。`theme.tsx` 经过 React Compiler 编译后生成带有 memo cache 的代码，但这对入口文件透明。

## 关键代码路径与文件引用

### 上游调用方

| 文件 | 作用 |
|------|------|
| `src/commands.ts:57` | `import theme from './commands/theme/index.js'`，将 theme 命令引入全局命令列表。 |
| `src/commands.ts:306` | 在 `COMMANDS()` memoized 数组中显式插入 `theme`，使其成为内置命令之一。 |
| `src/commands.ts:624` | `REMOTE_SAFE_COMMANDS` 集合包含 `theme`，表示在 `--remote` 模式下该命令仍然可用（仅影响本地 TUI 状态）。 |
| `src/screens/REPL.tsx` | 处理 `local-jsx` 命令的执行路径（`mod.call(onDone, context, commandArgs)`），负责渲染和生命周期管理。 |

### 下游被调用方

| 文件 | 作用 |
|------|------|
| `src/commands/theme/theme.tsx` | 懒加载目标，包含 `ThemePickerCommand` 组件和 `call` 导出。 |

## 依赖与外部交互

- **类型依赖**：`../../commands.js` 中的 `Command` 类型（实际来自 `src/types/command.ts` 的 re-export）。
- **运行时依赖**：无直接运行时依赖，所有交互通过 `load()` 委托给 `theme.tsx`。
- **命令系统交互**：
  - 注册阶段：`commands.ts` 的 `getCommands()` / `loadAllCommands()` 将其纳入可用命令列表。
  - 执行阶段：REPL 的 `onSubmit` 逻辑通过 `findCommand` 匹配到 `theme`，调用 `command.load()` 获取模块，再执行 `mod.call()`。

## 风险、边界与改进建议

### 风险

1. **路径漂移**：如果 `theme.tsx` 被重命名或移动而 `load` 路径未同步更新，用户调用 `/theme` 时将抛出模块找不到的错误（`Error: Cannot find module`）。由于该错误发生在懒加载时，启动阶段不会暴露。
2. **类型与运行时脱节**：`satisfies Command` 在编译时保证结构，但无法保证 `load()` 返回的模块确实包含 `call` 函数。若 `theme.tsx` 意外删除 `call` 导出，会在运行时触发 `mod.call is not a function`。

### 边界

- 该文件**不处理**命令的可用性（availability）、权限或 feature flag；这些由 `commands.ts` 中的 `isCommandEnabled()` 和 `meetsAvailabilityRequirement()` 统一处理。
- `theme` 命令没有 `aliases`、`argumentHint` 或 `isHidden` 等扩展属性，在命令列表中显示为最简单的形式。

### 改进建议

1. **增加单元测试覆盖**：目前该文件仅做对象声明，但可添加一个轻量测试验证 `theme.load()` 能成功解析且返回的模块包含 `call` 函数，防止重构时破坏懒加载契约。
2. **统一入口模式**：项目内大量命令采用相同的 `index.ts` + `实现.tsx` 结构，可考虑用代码生成或 lint 规则强制 `load` 路径与实现文件一致。
3. **考虑添加 `isEnabled`**：如果未来某些构建环境需要禁用主题切换（如纯 headless 模式），可在此添加 `isEnabled: () => !isHeadless()` 等条件开关，而不必修改 `commands.ts`。
