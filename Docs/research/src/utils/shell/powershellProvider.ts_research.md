# powershellProvider.ts 研究文档

## 场景与职责

`powershellProvider.ts` 是 Claude Code CLI 的 **PowerShell Shell 执行 provider**，与 `bashProvider.ts` 对应，实现 `ShellProvider` 接口的 PowerShell 侧逻辑。它负责将用户输入的 PowerShell 命令字符串组装成可安全执行的命令串，处理 sandbox 模式下的编码与包装，并管理 cwd 追踪与 session 环境变量注入。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `createPowerShellProvider(shellPath)` | 工厂函数，返回 `ShellProvider` 的 PowerShell 实现。 |
| `buildPowerShellArgs(cmd)` | 生成标准 PowerShell 启动参数：`['-NoProfile', '-NonInteractive', '-Command', cmd]`。 |
| `buildExecCommand(command, opts)` | 组装带 cwd 追踪与退出码捕获的 PowerShell 命令串；sandbox 模式下返回自调用 pwsh 的 shell 命令。 |
| `getSpawnArgs(commandString)` | 直接调用 `buildPowerShellArgs`，保持参数生成逻辑单一来源。 |
| `getEnvironmentOverrides()` | 注入 session env vars 与 sandbox TMPDIR。 |

## 具体技术实现

### 1. CWD 追踪与退出码捕获

```ts
const cwdTracking = `
; $_ec = if ($null -ne $LASTEXITCODE) { $LASTEXITCODE } elseif ($?) { 0 } else { 1 }
; (Get-Location).Path | Out-File -FilePath '${escapedCwdFilePath}' -Encoding utf8 -NoNewline
; exit $_ec`
```

- 优先使用 `$LASTEXITCODE`（原生可执行程序返回码）。
- 若 `$LASTEXITCODE` 为 `$null`（仅运行 cmdlet），回退到 `$?` 布尔状态转换。
- 注释详细说明了 PS 5.1 的 stderr 重定向 bug：当原生命令写入 stderr 且被 PS 重定向时，`$?` 会被设为 `$false` 即使 exe 返回 0。因此优先 `$LASTEXITCODE` 可减少误报。

### 2. Sandbox 模式下的自调用包装

当 `useSandbox === true` 时，sandbox runtime（`SandboxManager.wrapWithSandbox`）会硬编码把命令串包成 `<binShell> -c '<cmd>'`。这会导致 pwsh 失去 `-NoProfile -NonInteractive`。解决方案：

```ts
const commandString = opts.useSandbox
  ? [
      `'${shellPath.replace(/'/g, `'\''`)}'`,
      '-NoProfile',
      '-NonInteractive',
      '-EncodedCommand',
      encodePowerShellCommand(psCommand),
    ].join(' ')
  : psCommand
```

- 返回的命令串本身是一个调用 `pwsh -NoProfile -NonInteractive -EncodedCommand <base64>` 的 shell 命令。
- `Shell.ts` 在 sandbox 模式下会把该命令串再传给 `/bin/sh -c`，最终形成：
  `bwrap ... sh -c 'pwsh -NoProfile -NonInteractive -EncodedCommand ...'`
- 使用 Base64 UTF-16LE 编码（`-EncodedCommand`）是为了避免 sandbox runtime 再次调用 `shellquote.quote()` 时破坏特殊字符（如 `!` 被转义为 `\!`）。

### 3. 路径处理

- `cwdFilePath` 在非 sandbox 模式下使用 `os.tmpdir()`；sandbox 模式下写入 `sandboxTmpDir`（Linux/macOS 可写）。
- 路径中的单引号通过 `replace(/'/g, "''")` 做 PowerShell 字符串转义。
- shellPath 中的单引号在 sandbox 模式下通过 POSIX 单引号转义规则处理：`'\''`。

### 4. 环境变量覆盖

```ts
for (const [key, value] of getSessionEnvVars()) {
  env[key] = value
}
if (currentSandboxTmpDir) {
  env.TMPDIR = currentSandboxTmpDir
  env.CLAUDE_CODE_TMPDIR = currentSandboxTmpDir
}
```

- session env vars 先应用，随后 sandbox TMPDIR 覆盖，防止 `/env TMPDIR=...` 突破沙箱隔离。
- 注释提到 `bashProvider.ts` 中顺序相反（历史遗留），但此处认为沙箱隔离应获胜。

## 关键代码路径与文件引用

| 文件 | 关系 | 说明 |
|------|------|------|
| `src/utils/shell/powershellProvider.ts` | 本文件 | Provider 实现。 |
| `src/utils/shell/shellProvider.ts` | 被实现 | `ShellProvider` 类型定义。 |
| `src/utils/Shell.ts` | 调用方 | `getPsProvider()` 调用 `createPowerShellProvider()`；sandbox 时传 `/bin/sh` 作为 binShell。 |
| `src/utils/hooks.ts` | 调用方 | PowerShell hook 直接调用 `buildPowerShellArgs()` 生成 spawn 参数。 |
| `src/utils/sessionEnvVars.ts` | 被调用 | 读取 `/env` 设置的会话级环境变量。 |
| `src/utils/sandbox/sandbox-adapter.ts` | 外部交互 | `SandboxManager.wrapWithSandbox()` 对命令做 sandbox 包装。 |

## 依赖与外部交互

- **Node.js 内置**: `os`（`tmpdir`）、`path`、`path/posix`。
- **内部模块**: `../sessionEnvVars.js`、`./shellProvider.js`。
- **外部进程**: 通过 `spawn` 启动 pwsh/powershell；通过 sandbox runtime 做隔离执行。

## 风险、边界与改进建议

### 风险

1. **Base64 编码体积膨胀**: UTF-16LE + Base64 使命令串体积膨胀约 2.67 倍，对于极大脚本可能接近命令行长度限制（Windows 约 32767 字符）。
2. **$LASTEXITCODE 与 cmdlet 混合行为的权衡**: `native-ok; cmdlet-fail` 现在返回 0（旧逻辑返回 1），虽然注释认为这种情况比 git stderr bug 更罕见，但仍可能在某些链式命令中导致误判。
3. **Sandbox 模式硬编码 `/bin/sh`**: `Shell.ts` 假设 `/bin/sh` 一定存在，这在某些极简容器（如基于 Alpine 的自定义镜像）中可能不成立，虽然当前支持的 sandbox 平台均满足该假设。

### 边界

- PowerShell provider 的 `detached` 设为 `false`（与 bash 的 `true` 不同），因为 Windows 进程树杀除机制与 POSIX 不同。
- 不支持 `CLAUDE_CODE_SHELL_PREFIX` 环境变量（与 bashProvider 不同），PowerShell hook 也明确忽略该前缀。
- 当前未实现 snapshot 机制（无 `.sh` 环境快照注入），PowerShell 命令每次都在干净进程中运行。

### 改进建议

1. **命令长度检测与降级**: 在 `encodePowerShellCommand` 前检测原始命令长度，若超过阈值则改用临时 `.ps1` 文件执行，避免命令行截断。
2. **统一退出码策略**: 考虑暴露更细粒度的退出码信息（如区分 native exit code 与 pipeline 状态），让上游 `Shell.ts` 能更准确地判断命令成败。
3. **支持 PowerShell 环境快照**: 研究是否可以通过导出/导入 `clixml` 或 `$env:` 快照来复现 Bash 的 snapshot 机制，提升跨命令状态一致性。
4. **增加对 `-File` 路径的 sandbox 适配**: 若用户传入 `.ps1` 脚本路径，sandbox 可能需要额外挂载该脚本文件，当前未做特殊处理。
