# 研究文档：src/commands/reload-plugins/index.ts

## 场景与职责

`src/commands/reload-plugins/index.ts` 是 `/reload-plugins` 命令的**注册入口与懒加载声明文件**。它属于 Claude Code 插件体系中的 Layer-3（运行时会话层）刷新入口之一，职责非常单一：

1. **声明命令元数据**：向全局命令注册表暴露一个名为 `reload-plugins` 的本地命令（`type: 'local'`）。
2. **懒加载实现**：通过 `load: () => import('./reload-plugins.js')` 将真正的执行逻辑延迟到用户首次触发该命令时才加载，避免启动时引入不必要的依赖。
3. **区分交互式与非交互式场景**：显式标记 `supportsNonInteractive: false`，表示该命令仅在交互式 TUI/REPL 中可用；SDK/headless 场景不会通过文本 prompt 触发此命令，而是调用 `query.reloadPlugins()` 控制请求（返回结构化数据供 UI 更新）。

该文件在 `src/commands.ts` 的 `COMMANDS` 数组中被静态导入（`import reloadPlugins from './commands/reload-plugins/index.js'`），因此只要命令系统初始化完成，`reload-plugins` 就会出现在命令列表与自动补全中。

## 功能点目的

- **提供用户显式刷新插件状态的 slash 命令入口**：当用户在会话中安装/启用/禁用插件、修改插件配置，或通过 `/plugin` 菜单变更插件后，需要一条明确的命令将会话内的 `AppState` 与磁盘上的插件实际状态同步。
- **保持启动性能**：将重逻辑（插件缓存清理、重新加载所有插件、MCP/LSP 重连、设置同步等）隔离到 `reload-plugins.ts`，本文件只做轻量元数据声明。
- **防止 headless/SDK 误用**：通过 `supportsNonInteractive: false` 与注释明确告知调用方，非交互式路径应使用控制请求 API，而不是把 `/reload-plugins` 当作文本 prompt 发送。

## 具体技术实现

### 数据结构

文件导出一个 `Command` 类型的默认对象（使用 `satisfies Command` 进行类型约束）：

```ts
const reloadPlugins = {
  type: 'local',
  name: 'reload-plugins',
  description: 'Activate pending plugin changes in the current session',
  supportsNonInteractive: false,
  load: () => import('./reload-plugins.js'),
} satisfies Command
```

字段说明：
- `type: 'local'`：本地命令，执行后返回文本结果，不展开为模型 prompt。
- `name: 'reload-plugins'`：命令唯一标识，也是用户输入的 `/reload-plugins` 名称。
- `description`：在帮助系统与自动补全中显示的描述。
- `supportsNonInteractive: false`：禁止在非交互式环境（如 headless 子进程、SDK）中通过 slash command 方式调用。
- `load`：懒加载函数，返回 `Promise<LocalCommandModule>`，模块中需导出 `call` 函数（即 `reload-plugins.ts` 中的 `export const call: LocalCommandCall`）。

### 关键流程

1. **启动阶段**：`src/commands.ts` 静态导入本文件，`reloadPlugins` 对象被加入 `COMMANDS` 数组。
2. **命令匹配阶段**：当用户在 REPL 输入 `/reload-plugins` 时，`findCommand` 命中该对象。
3. **懒加载阶段**：命令调度器调用 `reloadPlugins.load()`，动态导入 `./reload-plugins.js`，随后执行其导出的 `call` 函数。

## 关键代码路径与文件引用

| 路径 | 关系 | 说明 |
|------|------|------|
| `src/commands.ts` | 调用方/注册方 | 第 136 行静态导入 `reloadPlugins`，并在 `COMMANDS` 数组（第 296 行）中注册。 |
| `src/types/command.ts` | 类型依赖 | 提供 `Command`、`LocalCommandModule`、`LocalCommandCall` 等类型定义。 |
| `src/commands/reload-plugins/reload-plugins.ts` | 被加载方 | 真正的命令实现，由 `load()` 懒加载。 |

## 依赖与外部交互

- **无运行时外部 IO**：本文件仅做对象声明与类型约束，不直接访问文件系统、网络、状态管理或子进程。
- **编译时依赖**：
  - `../../commands.js`（实际为 `src/commands.ts`）提供 `Command` 类型。
  - Bun 的打包系统处理 `bun:bundle` 相关特性标记（本文件未直接使用，但整个命令系统通过 `feature()` 做条件编译）。

## 风险、边界与改进建议

### 风险与边界

1. **命名耦合**：`name: 'reload-plugins'` 在多处硬编码匹配（如 `src/hooks/useManagePlugins.ts` 的通知文本、`src/utils/plugins/refresh.ts` 的注释引用）。若重命名命令，需要全局搜索替换。
2. **懒加载路径稳定性**：`load: () => import('./reload-plugins.js')` 使用 `.js` 扩展名（TypeScript 项目的 ESM 兼容写法），若目标文件被重命名或移动，编译期不会报错，运行时会抛出模块找不到错误。
3. **SDK 与本地命令语义分裂**：注释提到 SDK 使用 `query.reloadPlugins()`，但本文件并未对 SDK 路径做任何显式约束，完全依赖调用方自觉遵守约定。若未来 SDK 错误地通过文本发送 `/reload-plugins`，会因 `supportsNonInteractive: false` 被拒绝。

### 改进建议

1. **集中命令名称常量**：建议将 `'reload-plugins'` 提取为 `src/commands/reload-plugins/constants.ts` 中的常量，供通知系统、测试、文档引用，减少魔法字符串。
2. **增加类型级懒加载校验**：可考虑在 `Command` 类型中为 `load` 增加返回模块的泛型约束，确保 `./reload-plugins.js` 必须导出 `call` 函数，提升重构安全性。
3. **注释同步到文档**：文件中关于 SDK 调用方式的注释很有价值，建议同步到 `AGENTS.md` 或 SDK 集成文档，降低新开发者误用概率。
