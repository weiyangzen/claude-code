# terminalLauncher.ts 研究文档

> 文件路径：`src/utils/deepLink/terminalLauncher.ts`  
> 研究时间：2026-04-01  
> 执行器：kimi (k2p5)

---

## 场景与职责

`terminalLauncher.ts` 解决的核心问题是：**当操作系统通过 deep link 唤起 Claude 时，进程本身没有 attached 终端（headless 上下文），因此需要主动打开一个用户偏好的终端模拟器，并在其中启动新的 Claude 实例。**

该模块负责：
1. **终端检测**：按平台探测用户安装的终端模拟器；
2. **安全启动**：将 claude 二进制路径、deep link 参数（`--deep-link-origin`、`--prefill` 等）传递给终端；
3. **进程分离**：以 detached 模式启动终端，确保 trampoline 进程可立即退出，不阻塞等待终端关闭。

平台支持：
- **macOS**：Terminal.app, iTerm2, Ghostty, Kitty, Alacritty, WezTerm
- **Linux**：`$TERMINAL`, `x-terminal-emulator`, gnome-terminal, konsole, xfce4-terminal, mate-terminal, tilix, xterm 等
- **Windows**：Windows Terminal (wt.exe), PowerShell, cmd.exe

---

## 功能点目的

| 功能 | 目的 |
|------|------|
| `detectTerminal()` | 按平台返回用户首选的终端信息 `{ name, command }`。 |
| `launchInTerminal(claudePath, action)` | 组装 claude 参数，按平台调用对应的终端启动函数。 |
| `launchMacosTerminal` / `launchLinuxTerminal` / `launchWindowsTerminal` | 各平台具体的参数映射与启动逻辑。 |
| `spawnDetached` | 以 `detached: true` + `stdio: 'ignore'` 启动子进程，确保 trampoline 不等待终端。 |
| `buildShellCommand` / `shellQuote` / `appleScriptQuote` / `psQuote` / `cmdQuote` | 针对不同 shell/解释器的字符串转义，防止命令注入。 |

---

## 具体技术实现

### 关键流程

#### 1. `launchInTerminal(claudePath, action)`
组装 claude 参数列表：
```ts
const claudeArgs = ['--deep-link-origin']
if (action.repo) {
  claudeArgs.push('--deep-link-repo', action.repo)
  if (action.lastFetchMs !== undefined) {
    claudeArgs.push('--deep-link-last-fetch', String(action.lastFetchMs))
  }
}
if (action.query) {
  claudeArgs.push('--prefill', action.query)
}
```
随后按 `process.platform` 分发到 `launchMacosTerminal` / `launchLinuxTerminal` / `launchWindowsTerminal`。

#### 2. macOS 终端启动 (`launchMacosTerminal`)
分为 **Pure argv 路径** 与 **Shell-string 路径** 两大类：

| 终端 | 类型 | 启动方式 |
|------|------|----------|
| iTerm2 | Shell-string | AppleScript `write text` → 需 `shellQuote` + `appleScriptQuote` |
| Terminal.app | Shell-string | AppleScript `do script` → 同上 |
| Ghostty | Pure argv | `open -na Ghostty --args --window-save-state=never --working-directory=<cwd> -e <claudePath> ...` |
| Alacritty | Pure argv | `open -na Alacritty --args --working-directory <cwd> -e <claudePath> ...` |
| Kitty | Pure argv | `open -na kitty --args --directory <cwd> <claudePath> ...` |
| WezTerm | Pure argv | `open -na WezTerm --args start --cwd <cwd> -- <claudePath> ...` |

- **iTerm2 特殊处理**：若 iTerm 未运行，直接 `activate` 让其自身创建首个窗口；若已运行，则 `create window with default profile` 避免双窗口。
- **失败回退**：任何终端启动失败（`code !== 0`）均回退到 `Terminal.app`。

#### 3. Linux 终端启动 (`launchLinuxTerminal`)
所有 Linux 路径均为 **Pure argv**：通过各终端原生的 `--working-directory` / `--directory` / `--workdir` 等参数设置 cwd，再以 `-e`/`-x` 直接执行 `claudePath` + `claudeArgs`。

| 终端 | cwd 参数 | 执行前缀 |
|------|----------|----------|
| gnome-terminal | `--working-directory=<cwd>` | `--` |
| konsole | `--workdir <cwd>` | `-e` |
| kitty | `--directory <cwd>` | (无) |
| wezterm | `start --cwd <cwd> --` | (无) |
| alacritty | `--working-directory <cwd>` | `-e` |
| ghostty | `--working-directory=<cwd>` | `-e` |
| xfce4-terminal / mate-terminal | `--working-directory=<cwd>` | `-x` |
| tilix | `--working-directory=<cwd>` | `-e` |
| xterm / x-terminal-emulator / $TERMINAL | 无可靠 cwd 参数 | `spawn({cwd})` 设置进程 cwd |

最终统一调用 `spawnDetached(terminal.command, args, { cwd: spawnCwd })`。

#### 4. Windows 终端启动 (`launchWindowsTerminal`)
- **Windows Terminal (wt.exe)**：Pure argv
  ```
  wt.exe -d <cwd> -- <claudePath> --deep-link-origin --prefill "..."
  ```
- **PowerShell**：Shell-string
  ```
  pwsh.exe -NoExit -Command "Set-Location '<cwd>'; & '<claudePath>' '--deep-link-origin' '--prefill' '...'"
  ```
  使用 `psQuote`（单引号转义为 `''`）进行安全包装。
- **cmd.exe**：Shell-string
  ```
  cmd.exe /k "cd /d \"<cwd>\" && \"<claudePath>\" ..."
  ```
  使用 `cmdQuote`（剥离 `"`，`%` 转 `%%`，尾部反斜杠加倍）。
  - **关键细节**：`cmd.exe` 不使用 MSVCRT 风格的参数解析，因此显式设置 `windowsVerbatimArguments: true` 以绕过 libuv 的默认转义逻辑。

### 字符串转义实现

| 函数 | 用途 | 核心逻辑 |
|------|------|----------|
| `shellQuote` | POSIX sh（AppleScript 路径） | `'${s.replace(/'/g, "'\\''")}'` |
| `appleScriptQuote` | AppleScript 字符串字面量 | `"${s.replace(/\\/g, '\\\\').replace(/"/g, '\\"')}"` |
| `psQuote` | PowerShell 单引号字符串 | `'${s.replace(/'/g, "''")}'` |
| `cmdQuote` | cmd.exe 参数 | 剥离 `"`；`%` → `%%`；尾部 `\` 加倍；再包 `"..."` |

---

## 关键代码路径与文件引用

### 本文件导出
- `detectTerminal()` → 未在本批次发现外部直接调用，推测供调试或未来 CLI 命令使用。
- `launchInTerminal()` → `src/utils/deepLink/protocolHandler.ts` ~L60（`handleDeepLinkUri` 中调用）。

### 被调用方详情

| 导入来源 | 符号 | 用途 |
|----------|------|------|
| `child_process` | `spawn` | 启动终端子进程 |
| `path` | `basename` | Linux `$TERMINAL` 显示名称 |
| `../config.js` | `getGlobalConfig` | 读取 `deepLinkTerminal` 存储偏好 |
| `../debug.js` | `logForDebugging` | 调试日志 |
| `../execFileNoThrow.js` | `execFileNoThrow` | macOS `mdfind`、`ls`、`osascript` |
| `../which.js` | `which` | 探测各平台终端可执行文件 |

### 调用方详情

**`src/utils/deepLink/protocolHandler.ts`**
```ts
const launched = await launchInTerminal(process.execPath, {
  query: action.query,
  cwd,
  repo: resolvedRepo,
  lastFetchMs: lastFetch?.getTime(),
})
```
- 若 `launched` 为 `false`，向 stderr 输出错误并返回 exit code `1`。

---

## 依赖与外部交互

### 外部系统命令
| 平台 | 命令 | 作用 |
|------|------|------|
| macOS | `mdfind` | Spotlight 查询终端 app bundle 是否安装。 |
| macOS | `ls` | 当 `mdfind` 不可用时回退检查 `/Applications/*.app`。 |
| macOS | `osascript` | 执行 AppleScript 控制 iTerm2 / Terminal.app。 |
| macOS | `open` | 以 `open -na <App> --args ...` 启动纯 argv 终端。 |

### 配置依赖
- **`getGlobalConfig().deepLinkTerminal`**：存储了用户上一次交互会话中检测到的终端偏好（由 `terminalPreference.ts` 写入）。这是 macOS 检测的首要信号，因为 headless LaunchServices 环境下 `TERM_PROGRAM` 不存在。

---

## 风险、边界与改进建议

### 风险
1. **AppleScript 路径的注入风险**：iTerm2 与 Terminal.app 只能通过 AppleScript `write text` / `do script` 传递命令字符串，无法使用 argv。虽然 `shellQuote` 与 `appleScriptQuote` 已做转义，但任何转义漏洞都会直接导致命令注入。
2. **cmd.exe 的 `"` 剥离策略**：`cmdQuote` 选择直接删除输入中的 `"`，这意味着若用户 prompt 中原本包含 `"`，其语义会被改变（而非被安全保留）。这是 cmd.exe  quoting 模型的已知限制。
3. **`windowsVerbatimArguments` 的兼容性**：该标志仅对 `cmd.exe` 启用，若未来支持其他 Windows 终端（如新的第三方终端），需要重新评估 libuv 转义策略。
4. **Linux `spawnDetached` 的 cwd 继承**：对于无 cwd 参数的终端（如 xterm），依赖 `spawn({cwd})` 让终端进程继承 cwd，再由终端将 cwd 传递给子进程。这不是所有终端都保证的行为，可能导致实际启动目录与预期不符。

### 边界
- **检测偏好顺序（macOS）**：`stored config > TERM_PROGRAM > mdfind > ls /Applications > Terminal.app fallback`。这意味着首次 deep link 启动时，若用户终端未运行且 `TERM_PROGRAM` 未保留，可能选错终端。
- **Linux 检测顺序**：`$TERMINAL > x-terminal-emulator > 硬编码优先级列表`。`$TERMINAL` 并非所有发行版都设置。
- **Windows 检测顺序**：`wt.exe > pwsh.exe > powershell.exe > cmd.exe`。

### 改进建议
1. **允许用户显式指定终端**：在设置中增加 `preferredTerminal` 选项，覆盖自动检测逻辑。
2. **cmd.exe 语义保留**：研究是否可通过环境变量或临时批处理文件绕过 cmd.exe 的 quoting 限制，从而保留 `"` 字符。
3. **Linux cwd 可靠性增强**：对 xterm 等无 cwd 参数的终端，可考虑先 `cd` 再执行，例如通过 `env` 包装或生成临时脚本。
4. **测试覆盖**：当前无单元测试，建议补充：
   - 各平台 `cmdQuote` / `psQuote` / `shellQuote` 的边界注入测试；
   - `launchMacosTerminal` 的 AppleScript 生成快照测试；
   - `detectTerminal` 的 mock 测试（mock `which`、`execFileNoThrow`、`getGlobalConfig`）。
