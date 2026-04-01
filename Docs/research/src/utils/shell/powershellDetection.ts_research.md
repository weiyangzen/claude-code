# powershellDetection.ts 研究文档

## 场景与职责

`powershellDetection.ts` 负责在宿主系统上 **探测 PowerShell 可执行文件的位置**，并缓存结果供整个进程生命周期复用。同时提供 `getPowerShellEdition()` 用于推断当前安装的是 PowerShell 7+（core）还是 Windows PowerShell 5.1（desktop）。该模块是 `PowerShellTool` 与 `hooks.ts` 中 PowerShell 支持的基础设施层。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `findPowerShell()` | 在 PATH 中查找 `pwsh`，回退到 `powershell`；在 Linux 上额外规避 snap 启动器挂起问题。 |
| `getCachedPowerShellPath()` | 提供带缓存的幂等接口，避免重复磁盘/PATH 探测。 |
| `getPowerShellEdition()` | 根据可执行文件名称推断版本族系（core / desktop），用于 prompt 生成不同的语法指导。 |
| `resetPowerShellCache()` | 测试专用：清空缓存 Promise。 |

## 具体技术实现

### 1. PowerShell 查找逻辑

```ts
export async function findPowerShell(): Promise<string | null> {
  const pwshPath = await which('pwsh')
  if (pwshPath) { ... }
  const powershellPath = await which('powershell')
  ...
}
```

- 优先 `pwsh`（PowerShell Core 7+），次选 `powershell`（Windows PowerShell 5.1）。
- Linux 特殊处理：若 `which('pwsh')` 指向 `/snap/...` 或通过符号链接最终解析到 `/snap/...`，则绕过 snap 启动器，直接探测 `/opt/microsoft/powershell/7/pwsh` 或 `/usr/bin/pwsh`。
  - 原因：snap 在子进程中初始化 confinement 时可能挂起，而底层二进制更可靠。
  - 实现：使用 `realpath()` 解析符号链接链，再检查是否以 `/snap/` 开头。

### 2. 缓存机制

```ts
let cachedPowerShellPath: Promise<string | null> | null = null

export function getCachedPowerShellPath(): Promise<string | null> {
  if (!cachedPowerShellPath) {
    cachedPowerShellPath = findPowerShell()
  }
  return cachedPowerShellPath
}
```

- 单 Promise 缓存：首次调用启动异步探测，后续调用直接返回同一 Promise。
- 缓存粒度为进程级，不支持按工作目录或环境变化刷新（除非调用 `resetPowerShellCache()`）。

### 3. Edition 推断

```ts
export async function getPowerShellEdition(): Promise<PowerShellEdition | null> {
  const p = await getCachedPowerShellPath()
  if (!p) return null
  const base = p.split(/[/\\]/).pop()!.toLowerCase().replace(/\.exe$/, '')
  return base === 'pwsh' ? 'core' : 'desktop'
}
```

- 不通过执行 `--version` 判断，而是基于二进制文件名：
  - `pwsh` / `pwsh.exe` → `'core'`（隐含 7+，因为 6 已 EOL）。
  - `powershell` / `powershell.exe` → `'desktop'`（5.1）。
- 该推断被 `PowerShellTool/prompt.ts` 用于生成版本相关的语法提示（如 `&&` 是否可用、默认编码等）。

## 关键代码路径与文件引用

| 文件 | 关系 | 说明 |
|------|------|------|
| `src/utils/shell/powershellDetection.ts` | 本文件 | 探测与缓存实现。 |
| `src/utils/which.ts` | 被调用 | 跨平台 `which` 实现（优先使用 `Bun.which`）。 |
| `src/utils/platform.ts` | 被调用 | `getPlatform()` 区分 windows / linux / macos。 |
| `src/utils/Shell.ts` | 调用方 | `getPsProvider()` 调用 `getCachedPowerShellPath()` 创建 PowerShell provider。 |
| `src/utils/hooks.ts` | 调用方 | PowerShell hook 执行前调用 `getCachedPowerShellPath()` 获取 pwsh 路径。 |
| `src/tools/PowerShellTool/prompt.ts` | 调用方 | `getPowerShellEdition()` 用于生成 edition-aware prompt。 |
| `src/utils/powershell/parser.ts` | 可能引用 | parser.ts 中也有 `toUtf16LeBase64`，与 powershellProvider 共享编码逻辑。 |

## 依赖与外部交互

- **Node.js 内置**: `fs/promises`（`stat`、`realpath`）。
- **内部模块**: `../platform.js`、`../which.js`。
- **外部进程**: 不直接 spawn 进程；仅通过 `which` 和 `stat` 做文件系统探测。

## 风险、边界与改进建议

### 风险

1. **缓存过期**: 若用户在会话中途安装/卸载 PowerShell，`getCachedPowerShellPath()` 仍返回旧结果。当前仅在测试入口提供 `resetPowerShellCache()`。
2. **Linux snap 绕过不完整**: 仅探测 `/opt/microsoft/powershell/7/pwsh` 和 `/usr/bin/pwsh` 两个固定路径，若用户通过其他非标准方式安装，仍可能回退到 snap 路径。
3. **Edition 推断过于简化**: 仅按文件名推断，无法区分 PowerShell 7.0 与 7.5 的功能差异；也无法处理用户将 `pwsh` 重命名为 `powershell` 的极端情况。

### 边界

- `which('pwsh')` 在 Windows 上通常返回 `C:\Program Files\PowerShell\7\pwsh.exe`。
- `which('powershell')` 在 Windows 上通常返回 `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe`。
- Linux 平台才执行 snap 规避逻辑；Windows/macOS 直接信任 PATH 结果。

### 改进建议

1. **增加缓存刷新机制**: 在 `Shell.ts` 或配置重载时提供显式刷新入口，而不仅限于测试。
2. **扩展 Linux 探测路径**: 增加 `/usr/local/bin/pwsh`、`$HOME/.dotnet/tools/pwsh` 等常见安装位置。
3. **精确版本探测**: 增加 `getPowerShellVersion()`，通过执行 `pwsh -Command "$PSVersionTable.PSVersion.ToString()"` 获取精确版本号，供 prompt 或功能开关使用。
4. **统一错误提示**: 当 `findPowerShell()` 返回 `null` 时，当前由调用方各自处理错误信息；可考虑在此模块内提供标准化错误消息模板。
