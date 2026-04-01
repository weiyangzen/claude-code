# shellProvider.ts 研究文档

## 场景与职责

`shellProvider.ts` 是 Claude Code CLI 中 **Shell 抽象层** 的核心契约文件。它定义了 `ShellProvider` 类型、`ShellType` 联合类型、支持的 shell 常量列表 `SHELL_TYPES`，以及 Hook 的默认 shell 常量 `DEFAULT_HOOK_SHELL`。该模块是 `bashProvider.ts`、`powershellProvider.ts`、`Shell.ts`、`hooks.ts` 等多个模块的公共依赖，起到接口统一与类型约束的作用。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `SHELL_TYPES` | 编译期常量数组 `['bash', 'powershell']`，用于类型推导与运行时遍历。 |
| `ShellType` | 从 `SHELL_TYPES` 推导出的联合类型：`'bash' | 'powershell'`。 |
| `DEFAULT_HOOK_SHELL` | Hook 系统默认使用的 shell：`'bash'`。 |
| `ShellProvider` | 接口定义：任何具体 shell 实现（bash / powershell）都必须提供 `type`、`shellPath`、`detached` 以及三个方法：`buildExecCommand`、`getSpawnArgs`、`getEnvironmentOverrides`。 |

## 具体技术实现

### 1. 类型定义

```ts
export const SHELL_TYPES = ['bash', 'powershell'] as const
export type ShellType = (typeof SHELL_TYPES)[number]
export const DEFAULT_HOOK_SHELL: ShellType = 'bash'

export type ShellProvider = {
  type: ShellType
  shellPath: string
  detached: boolean

  buildExecCommand(
    command: string,
    opts: {
      id: number | string
      sandboxTmpDir?: string
      useSandbox: boolean
    },
  ): Promise<{ commandString: string; cwdFilePath: string }>

  getSpawnArgs(commandString: string): string[]

  getEnvironmentOverrides(command: string): Promise<Record<string, string>>
}
```

### 2. 设计意图

- **统一抽象**: `Shell.ts` 的 `exec()` 函数无需关心底层是 bash 还是 PowerShell，只需通过 `resolveProvider[shellType]()` 获取 provider，然后调用统一接口。
- **最小契约**: 仅暴露命令执行所必需的三阶段生命周期：
  1. `buildExecCommand` — 构建命令串与 cwd 追踪文件路径。
  2. `getSpawnArgs` — 生成子进程 spawn 参数。
  3. `getEnvironmentOverrides` — 提供环境变量覆盖。
- **`detached` 标志**: 区分 POSIX（bash 用 `detached: true` 配合进程树杀除）与 Windows（PowerShell 用 `detached: false`）的进程生命周期管理策略。

## 关键代码路径与文件引用

| 文件 | 关系 | 说明 |
|------|------|------|
| `src/utils/shell/shellProvider.ts` | 本文件 | 接口与常量定义。 |
| `src/utils/shell/bashProvider.ts` | 实现方 | `createBashShellProvider()` 返回的对象符合 `ShellProvider`。 |
| `src/utils/shell/powershellProvider.ts` | 实现方 | `createPowerShellProvider()` 返回的对象符合 `ShellProvider`。 |
| `src/utils/Shell.ts` | 调用方 | `exec()` 通过 `resolveProvider[shellType]()` 获取 provider 并调用其方法。 |
| `src/utils/hooks.ts` | 调用方 | 读取 `DEFAULT_HOOK_SHELL`；通过 `shellType` 决定 spawn 路径。 |
| `src/utils/hooks/hooksSettings.ts` | 相关 | 可能在 hook 配置解析中引用 `ShellType`。 |

## 依赖与外部交互

- **无外部依赖**: 纯类型/常量文件，不导入任何内部或外部模块。
- **无网络/进程交互**。

## 风险、边界与改进建议

### 风险

1. **扩展性瓶颈**: `ShellType` 目前仅支持 `bash` 和 `powershell`。若未来需要支持 `zsh` 作为一等公民（而非通过 bash provider 兼容运行），需要修改本文件并级联更新所有实现方与调用方。
2. **`detached` 语义平台耦合**: `detached` 布尔值过于简单，无法表达 Windows 上需要的 `windowsHide`、UAC 提升、ConPTY 等复杂行为；这些目前散落在 `Shell.ts` 和 `hooks.ts` 中，未纳入 provider 契约。
3. **缺少错误/取消契约**: `ShellProvider` 未定义如何向 provider 传播 `AbortSignal`，信号处理完全由 `Shell.ts` 的 `wrapSpawn` 负责，provider 无法参与优雅取消或资源清理。

### 边界

- `getEnvironmentOverrides` 接收原始 `command: string`，但 bash provider 实际用它判断命令是否包含 "tmux"，以决定是否延迟初始化 tmux socket；PowerShell provider 目前忽略该参数。这是接口设计上的“可选使用”约定，非强制。
- `buildExecCommand` 的返回值包含 `cwdFilePath`，但不同 provider 对该路径的格式约定不同：bash 在 Windows 上返回 POSIX 路径（供 bash 内部使用），而 `Shell.ts` 负责在读取/删除时做平台转换。

### 改进建议

1. **增强 provider 契约**: 
   - 增加 `supportsSandbox: boolean` 标志，替代 `Shell.ts` 中硬编码的 `isSandboxedPowerShell` 判断。
   - 增加 `getCancellationStrategy()` 或让 `buildExecCommand` 接收 `AbortSignal`，使 provider 能执行更细粒度的取消逻辑。
2. **平台特定能力接口**: 将 `windowsHide`、是否需要 `posixPathToWindowsPath` 转换等抽成 provider 上的方法或属性，而不是让 `Shell.ts` 根据 `shellType` 硬编码分支。
3. **文档化扩展指南**: 若计划支持新 shell（如 `zsh` 独立 provider 或 `fish`），在本文件中添加注释说明需要实现的最小方法集合及常见陷阱（如路径格式、detached 行为、编码）。
4. **类型安全增强**: 考虑将 `ShellProvider` 拆分为 `BashShellProvider` 与 `PowerShellShellProvider` 两个更具体的子类型，在需要平台特定能力时做区分使用，同时保留一个公共基类型用于通用场景。
