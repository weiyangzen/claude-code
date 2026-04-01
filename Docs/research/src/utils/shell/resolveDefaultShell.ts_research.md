# resolveDefaultShell.ts 研究文档

## 场景与职责

`resolveDefaultShell.ts` 是 Claude Code CLI 中 **默认 Shell 解析** 的极简入口。它决定当用户在输入框中使用 `!` 快捷执行命令时，应该使用 `bash` 还是 `powershell`。该模块被 `src/utils/promptShellExecution.ts` 调用，是交互式 REPL 输入框与底层 Shell 工具之间的桥接层。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `resolveDefaultShell()` | 返回当前会话应使用的默认 shell 类型：`'bash'` 或 `'powershell'`。 |

## 具体技术实现

### 1. 解析顺序

```ts
export function resolveDefaultShell(): 'bash' | 'powershell' {
  return getInitialSettings().defaultShell ?? 'bash'
}
```

- 第一优先级：用户设置中的 `defaultShell`（来自 `settings.json` 或配置快照）。
- 第二优先级：硬编码默认值 `'bash'`。

### 2. 平台策略

- 注释明确说明：**所有平台默认都是 `'bash'`**，包括 Windows。
- 不会在 Windows 上自动回退到 PowerShell，因为那会破坏已有 Windows 用户的 bash hook 配置。
- 该行为对应设计文档 `docs/design/ps-shell-selection.md §4.2`。

## 关键代码路径与文件引用

| 文件 | 关系 | 说明 |
|------|------|------|
| `src/utils/shell/resolveDefaultShell.ts` | 本文件 | 默认 shell 解析实现。 |
| `src/utils/settings/settings.ts` | 被调用 | `getInitialSettings()` 读取启动时的用户设置快照。 |
| `src/utils/promptShellExecution.ts` | 调用方 | 输入框 `!` 命令执行前调用本函数决定 shell 类型。 |
| `src/utils/hooks.ts` | 相关 | Hook 的 shell 选择逻辑独立，使用 `hook.shell ?? DEFAULT_HOOK_SHELL`，不经过本模块。 |
| `src/utils/shell/shellProvider.ts` | 相关 | 定义 `DEFAULT_HOOK_SHELL = 'bash'`。 |

## 依赖与外部交互

- **内部模块**: `../settings/settings.js`。
- **无外部网络/进程交互**。

## 风险、边界与改进建议

### 风险

1. **设置漂移**: `getInitialSettings()` 返回的是启动时的设置快照，若用户在会话中途修改 `defaultShell`，本函数不会感知到变化，直到重启 CLI。
2. **功能范围狭窄**: 本函数仅服务于输入框 `!` 命令，而 `BashTool` / `PowerShellTool` 的显式调用、Hook 的 shell 选择均走各自独立逻辑，导致“默认 shell”概念在代码库中存在多个定义点，一致性风险较高。

### 边界

- 返回值类型被严格约束为 `'bash' | 'powershell'`，不支持 `zsh`、`fish` 等其他 shell。
- 若 `getInitialSettings().defaultShell` 为非法值（如 `'fish'`），由于 TypeScript 类型声明为 `'bash' | 'powershell'`，运行时可能返回非法字符串，但调用方若未做运行时校验可能引发下游错误。

### 改进建议

1. **运行时校验**: 在返回值前做一次白名单校验，非法值时回退到 `'bash'` 并打印警告日志，防止类型系统无法覆盖的运行时污染。
2. **统一默认 shell 概念**: 将 `resolveDefaultShell()` 提升为所有非显式指定 shell 场景的统一入口（包括 Hook 的第二阶段设计），减少概念分裂。
3. **支持动态刷新**: 若设置系统支持热重载，可让 `resolveDefaultShell` 订阅设置变更事件，或至少提供显式刷新接口。
4. **文档同步**: 在 `docs/design/ps-shell-selection.md` 中明确标注 `resolveDefaultShell.ts` 是该设计文档 §4.2 的代码实现锚点，方便新开发者追溯。
