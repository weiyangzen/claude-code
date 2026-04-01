# 研究报告：src/commands/terminalSetup/index.ts

## 场景与职责

`src/commands/terminalSetup/index.ts` 是 Claude Code CLI 中 `/terminal-setup` 命令的**入口清单文件（Command Manifest）**。它的核心职责是将终端配置命令注册到全局命令体系中，并决定该命令在用户面前的可见性与描述文案。该文件本身不包含任何业务逻辑，而是作为命令系统的"门面"，通过延迟加载（lazy-load）将真正的实现委托给同目录下的 `terminalSetup.tsx`。

## 功能点目的

1. **命令注册**：将 `terminal-setup` 注册为一个内置的 `local-jsx` 类型命令，使其可以通过用户输入 `/terminal-setup` 被调用。
2. **动态描述**：根据当前检测到的终端类型（`env.terminal`），动态切换命令描述：
   - 在 `Apple_Terminal` 下显示 "Enable Option+Enter key binding for newlines and visual bell"
   - 在其他终端下显示 "Install Shift+Enter key binding for newlines"
3. **可见性控制**：对于已经原生支持 CSI u / Kitty 键盘协议的终端（Ghostty、Kitty、iTerm2、WezTerm），该命令在帮助和自动补全中**隐藏**（`isHidden: true`），因为这些终端无需额外配置即可使用 Shift+Enter。
4. **延迟加载**：通过 `load: () => import('./terminalSetup.js')` 避免在启动时将 77KB 的实现模块加载到内存中。

## 具体技术实现

### 关键数据结构

```typescript
const NATIVE_CSIU_TERMINALS: Record<string, string> = {
  ghostty: 'Ghostty',
  kitty: 'Kitty',
  'iTerm.app': 'iTerm2',
  WezTerm: 'WezTerm',
}
```

该字典定义了哪些终端标识符属于"原生支持"阵营。注意：在 `terminalSetup.tsx` 的实现文件中，此字典还额外包含了 `WarpTerminal: 'Warp'`，但入口文件中未包含 Warp（历史遗留或有意为之的细微差异）。

### 命令定义

```typescript
const terminalSetup = {
  type: 'local-jsx',
  name: 'terminal-setup',
  description: /* 动态文案 */,
  isHidden: env.terminal !== null && env.terminal in NATIVE_CSIU_TERMINALS,
  load: () => import('./terminalSetup.js'),
} satisfies Command
```

- `type: 'local-jsx'` 表示该命令会渲染 React/Ink UI 组件（尽管实际实现中 `call` 函数仅返回 `null` 并通过 `onDone` 输出文本）。
- `satisfies Command` 确保类型安全，与 `src/types/command.ts` 中定义的 `Command` 接口兼容。

## 关键代码路径与文件引用

| 路径 | 关系 | 说明 |
|------|------|------|
| `src/commands/terminalSetup/index.ts` | 本文件 | 命令入口清单 |
| `src/commands/terminalSetup/terminalSetup.tsx` | 被延迟加载 | 真正的业务实现（77KB） |
| `src/commands.ts` | 调用方 | 第 55 行 `import terminalSetup from './commands/terminalSetup/index.js'`，第 312 行将其加入 `COMMANDS` 数组 |
| `src/utils/env.ts` | 依赖 | 通过 `env.terminal` 获取当前终端类型 |
| `src/types/command.ts` | 类型依赖 | `Command` 类型定义 |

## 依赖与外部交互

- **读取**：`env.terminal`（来自 `src/utils/env.ts` 的 `detectTerminal()` 结果）。
- **不直接写入**：本文件为纯只读清单，不修改文件系统、不修改配置、不执行子进程。
- **加载契约**：通过 `import('./terminalSetup.js')` 加载的模块必须满足 `LocalJSXCommandModule` 接口，即导出 `call: LocalJSXCommandCall` 函数。

## 风险、边界与改进建议

1. **Warp 终端的可见性不一致**：`index.ts` 中的 `NATIVE_CSIU_TERMINALS` 缺少 `WarpTerminal`，而 `terminalSetup.tsx` 中却包含它。这导致在 Warp 中 `/terminal-setup` 命令**不会隐藏**，但执行后 `call` 函数会提示 "Shift+Enter is natively supported in Warp. No configuration needed."。建议统一两个字典。
2. **描述与实现可能脱节**：`index.ts` 只根据 `Apple_Terminal` 与否二分化描述，但 `terminalSetup.tsx` 实际支持 vscode/cursor/windsurf/alacritty/zed 等多种终端。用户在非 Apple Terminal 下看到的描述永远是 "Install Shift+Enter key binding for newlines"，虽然功能正确，但描述不够精确。
3. **无测试覆盖**：项目中未找到针对 `terminalSetup` 的单元测试或集成测试文件。
4. **改进建议**：将 `NATIVE_CSIU_TERMINALS` 提取到同目录的共享常量文件中，确保入口清单与实现模块使用同一份数据源，消除字典漂移风险。
