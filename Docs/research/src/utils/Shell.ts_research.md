# `src/utils/Shell.ts` 深度研究文档

> 文件规模：约 474 行  
> 核心定位：Claude Code CLI 的主 shell 执行模块，负责所有外部命令的派生、生命周期管理、CWD 追踪与沙箱集成。

---

## 1. 场景与职责

`Shell.ts` 是 Claude Code CLI 与操作系统之间的“命令执行网关”。无论是用户通过 Bash/PowerShell 工具提交的命令、后台的 git 操作、安全审查流程，还是 skill 加载时的脚本执行，最终都会通过本模块的 `exec()` 函数落地为真实的子进程。

其主要职责包括：

1. **Shell 发现与选择**：自动检测系统上可用的 `bash`/`zsh`，支持环境变量覆盖，并做可执行性校验。
2. **命令构建与封装**：通过 `ShellProvider` 抽象层，为 bash 和 PowerShell 分别生成包含环境快照、会话变量、安全加固（关闭 extglob）、CWD 追踪等前缀的完整命令字符串。
3. **子进程派生**：使用 Node.js 的 `child_process.spawn()` 创建进程，处理超时、中断、后台化、输出重定向。
4. **CWD 追踪与恢复**：命令执行前后通过临时文件同步当前工作目录；若 CWD 被命令本身删除，则自动回退到启动目录。
5. **沙箱集成**：与 `SandboxManager` 协作，对命令进行 `bwrap`（Linux/macOS）级别的隔离，并处理沙箱化 PowerShell 的特殊路径。
6. **输出管理**：通过 `TaskOutput` 统一接管 stdout/stderr，支持“文件模式”（高吞吐量、无 JS 介入）和“管道模式”（实时回调、用于 hooks）。

---

## 2. 功能点目的

### 2.1 `findSuitableShell()` — 可靠的 Shell 发现

优先级设计（第 73–137 行）：

1. `CLAUDE_CODE_SHELL` 环境变量显式覆盖（仅限 bash/zsh，且需通过 `isExecutable()` 校验）。
2. `SHELL` 环境变量（同样仅限 bash/zsh）。
3. `which('zsh')` / `which('bash')` 的异步探测结果。
4. 固定 fallback 路径：`/bin`、`/usr/bin`、`/usr/local/bin`、`/opt/homebrew/bin`。

`isExecutable()`（第 50–68 行）采用双重校验：先 `accessSync(X_OK)`，在 Nix 等环境中可能失败，于是 fallback 到 `execFileSync(shellPath, ['--version'])` 快速验证。若全部失败，则抛出明确错误，避免在后续执行阶段出现难以定位的 `ENOENT`。

### 2.2 `getShellConfig()` / `getPsProvider()` — 缓存的 Provider 获取

- `getShellConfig`（第 146 行）通过 `lodash-es/memoize` 缓存 `findSuitableShell()` 与 `createBashShellProvider()` 的结果，保证每会话只初始化一次 bash provider。
- `getPsProvider`（第 148–154 行）同样 memoized，缓存 PowerShell 路径探测与 provider 创建。

### 2.3 `exec()` — 主执行函数

`exec(command, abortSignal, shellType, options?)`（第 181–442 行）是整个文件的核心。其设计目标是：

- **安全**：沙箱隔离、敏感环境变量剥离、`O_NOFOLLOW` 防止符号链接攻击。
- **可靠**：CWD 恢复、超时处理、后台化、进程树清理（tree-kill）。
- **高效**：文件模式绕过 JS 事件循环，直接让内核把 stdout/stderr 刷到同一 fd；管道模式则在需要实时回调时启用。
- **可观测**：进度回调、分析日志、调试日志。

### 2.4 `setCwd()` — 带符号链接解析的目录切换

`setCwd(path, relativeTo?)`（第 447–474 行）不仅做路径拼接，还会调用 `realpathSync` 解析符号链接，确保内部状态始终保存物理路径（与 `pwd -P` 行为一致）。若路径不存在，则抛出友好错误而非裸 `ENOENT`。

---

## 3. 具体技术实现（关键流程/数据结构/协议/命令）

### 3.1 ShellProvider 抽象与命令组装

`ShellProvider` 类型定义于 `src/utils/shell/shellProvider.ts`，包含三个方法：

- `buildExecCommand(command, opts)`：返回 `{ commandString, cwdFilePath }`。
- `getSpawnArgs(commandString)`：返回传给 `spawn()` 的 args 数组。
- `getEnvironmentOverrides(command)`：返回需要额外注入的环境变量。

#### Bash Provider（`src/utils/shell/bashProvider.ts`）

`buildExecCommand` 的组装顺序（第 77–198 行）：

1. **Shell 快照恢复**：若存在由 `createAndSaveSnapshot()` 生成的快照文件，先 `source` 它；若快照在会话期间被清理（如 tmpdir 回收），则回退到 `-l`（login shell）初始化。
2. **会话环境脚本**：`getSessionEnvironmentScript()` 注入 `/env` 等会话级变量。
3. **安全加固**：`getDisableExtglobCommand()` 关闭 bash 的 `extglob` 和 zsh 的 `EXTENDED_GLOB`，防止恶意文件名在命令验证后发生二次扩展。
4. **命令包装**：
   - 对 Windows 的 `2>nul` 做防御性重写（防止在 Git Bash 中创建名为 `nul` 的字面文件）。
   - 若命令包含管道且需要 stdin 重定向，则调用 `rearrangePipeCommand()` 把 `< /dev/null` 移到第一个命令之后，避免 `rg | wc -l < /dev/null` 导致 rg 永远等待 stdin。
   - 使用 `eval` 执行已引用的命令，确保 alias 在 `source` 后能被二次解析。
5. **CWD 追踪**：追加 `pwd -P >| ${shellCwdFilePath}`，把物理路径写入临时文件。
6. **前缀包装**：若设置了 `CLAUDE_CODE_SHELL_PREFIX`，则把整个命令串再包一层。

`getSpawnArgs`（第 200–206 行）在有快照时返回 `['-c', commandString]`，无快照时返回 `['-c', '-l', commandString]`，确保环境初始化不丢失。

#### PowerShell Provider（`src/utils/shell/powershellProvider.ts`）

`buildExecCommand`（第 35–97 行）的关键设计：

- **退出码捕获**：优先使用 `$LASTEXITCODE`（原生可执行程序返回值），仅在 `$null` 时 fallback 到 `$?`（cmdlet 状态）。这解决了 PS 5.1 中 `git push 2>&1` 因 stderr 被重定向导致 `$? = $false` 的误报问题。
- **CWD 追踪**：使用 `(Get-Location).Path | Out-File -FilePath '${escapedCwdFilePath}' -Encoding utf8 -NoNewline`。
- **沙箱特殊处理**：非沙箱路径直接返回裸 PS 命令，由 `getSpawnArgs()` 追加 `-NoProfile -NonInteractive -Command`；沙箱路径则预先把命令编码为 Base64 UTF-16LE（`-EncodedCommand`），并自行拼接成 `pwsh -NoProfile -NonInteractive -EncodedCommand ...` 的字符串。原因见下文“沙箱化 PowerShell 特殊 case”。

### 3.2 文件模式 vs 管道模式（File Mode vs Pipe Mode）

这是 `exec()` 中输出重定向的核心分支（第 281–313 行、第 330–332 行）。

#### 文件模式（默认，无 `onStdout` 回调时）

```ts
const usePipeMode = !!onStdout  // 第 284 行
const taskOutput = new TaskOutput(taskId, onProgress ?? null, !usePipeMode)  // 第 286 行
```

- `stdoutToFile = true`。
- `open(taskOutput.path, ...)` 打开一个磁盘文件，获得 `FileHandle`。
- `spawn()` 的 `stdio` 配置为 `['pipe', outputHandle?.fd, outputHandle?.fd]`，即 stdout 和 stderr 都指向同一个文件 fd。
- **性能优势**：数据完全不经过 Node.js 主线程，由内核直接写入磁盘；`TaskOutput` 只在需要进度时通过 `tailFile()` 轮询文件尾部（`POLL_INTERVAL_MS = 1000`）。
- **原子性**：POSIX 上通过 `O_APPEND` 保证每次写入都是原子 seek-to-end + write，stdout 与 stderr 按时间顺序交错且不会撕裂（tearing）。

#### 管道模式（提供 `onStdout` 回调时）

- `stdoutToFile = false`。
- `spawn()` 的 `stdio` 为 `['pipe', 'pipe', 'pipe']`。
- `ShellCommandImpl` 内部创建 `StreamWrapper`（定义于 `src/utils/ShellCommand.ts` 第 66–104 行），把 `childProcess.stdout` 和 `childProcess.stderr` 的数据事件实时写入 `TaskOutput` 的内存缓冲区。
- 内存上限 `DEFAULT_MAX_MEMORY = 8MB`，超出后触发 `spillToDisk()`，把内容转存到磁盘文件。
- 在 `exec()` 第 364–368 行，额外为 `childProcess.stdout` 挂载 `onStdout` 回调，让调用方（如 hooks）能实时拿到输出分片。

### 3.3 O_NOFOLLOW 与 Windows 'w' 标志的安全考量

第 299–313 行的 `open()` 调用包含平台差异化处理：

```ts
const O_NOFOLLOW = fsConstants.O_NOFOLLOW ?? 0
outputHandle = await open(
  taskOutput.path,
  process.platform === 'win32'
    ? 'w'
    : fsConstants.O_WRONLY |
        fsConstants.O_CREAT |
        fsConstants.O_APPEND |
        O_NOFOLLOW,
)
```

- **POSIX（Linux/macOS）**：使用数值标志 `O_WRONLY | O_CREAT | O_APPEND | O_NOFOLLOW`。
  - `O_NOFOLLOW` 的作用是：如果 `taskOutput.path` 是一个符号链接，则 `open()` 失败而非跟随链接。这能防止沙箱内的恶意进程提前创建一个指向敏感文件（如 `/etc/passwd`）的符号链接，导致子进程的 stdout 实际覆盖到目标文件上。
- **Windows**：使用字符串标志 `'w'` 而非数值标志。
  - 注释（第 292–298 行）解释了原因：在 Windows 上，若使用 `'a'`（append）模式，libuv 的 `fs__open` 会剥离 `FILE_WRITE_DATA` 权限、仅保留 `FILE_APPEND_DATA`；而 MSYS2/Cygwin 在探测继承句柄时，会把缺少 `FILE_WRITE_DATA` 的句柄视为只读，从而静默丢弃所有输出。使用 `'w'` 则授予 `FILE_GENERIC_WRITE`。由于 `spawn()` 时 dup 的句柄共享同一个 `FILE_OBJECT` 且带有 `FILE_SYNCHRONOUS_IO_NONALERT`，内核级单锁仍然能保证 I/O 串行化，原子性不会因此丢失。

### 3.4 沙箱化 PowerShell 特殊 Case（使用 /bin/sh 作为内层 shell）

第 247–278 行专门处理 `shouldUseSandbox && shellType === 'powershell'` 的情况：

```ts
const isSandboxedPowerShell = shouldUseSandbox && shellType === 'powershell'
const sandboxBinShell = isSandboxedPowerShell ? '/bin/sh' : binShell
// ...
const spawnBinary = isSandboxedPowerShell ? '/bin/sh' : binShell
const shellArgs = isSandboxedPowerShell
  ? ['-c', commandString]
  : provider.getSpawnArgs(commandString)
```

**原因**：`SandboxManager.wrapWithSandbox()` 的底层实现硬编码了 `<binShell> -c '<cmd>'` 的包装方式。如果直接把 `pwsh` 作为 `binShell` 传进去，`-NoProfile -NonInteractive` 等关键标志会丢失，导致：

1. PowerShell 加载用户 profile，产生额外延迟和不可控输出。
2. profile 中可能包含交互式提示，在沙箱内挂起。

**解决方案**：

- `powershellProvider.buildExecCommand({ useSandbox: true })` 预先把最终命令包装成 `pwsh -NoProfile -NonInteractive -EncodedCommand <base64>` 的形式（第 86–93 行）。
- `Shell.ts` 把 `/bin/sh` 作为沙箱的 `binShell` 传入，于是沙箱内部执行的是 `/bin/sh -c 'pwsh -NoProfile ... -EncodedCommand ...'`。
- 外层 `spawn()` 同样使用 `/bin/sh -c` 来解析沙箱返回的 POSIX 格式命令串。
- `/bin/sh` 被注释明确说明“存在于所有支持沙箱的平台上”。

### 3.5 CWD 恢复逻辑及其重要性

第 220–238 行：

```ts
let cwd = pwd()
try {
  await realpath(cwd)
} catch {
  const fallback = getOriginalCwd()
  logForDebugging(`Shell CWD "${cwd}" no longer exists, recovering to "${fallback}"`)
  try {
    await realpath(fallback)
    setCwdState(fallback)
    cwd = fallback
  } catch {
    return createFailedCommand(
      `Working directory "${cwd}" no longer exists. Please restart Claude from an existing directory.`,
    )
  }
}
```

**触发场景**：用户执行了类似 `rm -rf $(pwd)` 或 `cd` 进了一个被删除的临时目录。此时若直接 `spawn()`，子进程会因 CWD 不存在而立即失败。

**恢复策略**：

1. 先用 `realpath(cwd)` 探测当前目录是否还在磁盘上。
2. 若已消失，回退到 `getOriginalCwd()`（即 Claude 启动时的目录）。
3. 若启动目录也消失了（极端情况），则直接返回 `createFailedCommand()`，给出明确的用户提示，避免抛出难以理解的系统错误。

这保证了 CLI 不会因为一次“自毁式”命令而彻底丧失执行能力。

### 3.6 `.then()` 微任务中的同步 `readFileSync` / `unlinkSync`

第 371–421 行：

```ts
void shellCommand.result.then(async result => {
  // ...
  if (result && !preventCwdChanges && !result.backgroundTaskId) {
    try {
      let newCwd = readFileSync(nativeCwdFilePath, { encoding: 'utf8' }).trim()
      // ...
      if (newCwd.normalize('NFC') !== cwd) {
        setCwd(newCwd, cwd)
        invalidateSessionEnvCache()
        void onCwdChangedForHooks(cwd, newCwd)
      }
    } catch {
      logEvent('tengu_shell_set_cwd', { success: false })
    }
  }
  try {
    unlinkSync(nativeCwdFilePath)
  } catch {
    // File may not exist if command failed before pwd -P ran
  }
})
```

注释（第 371–375 行）明确解释了**必须使用同步 API** 的原因：

> “NOTE: readFileSync/unlinkSync are intentional here — these must complete synchronously within the .then() microtask so that callers who `await shellCommand.result` see the updated cwd immediately after. Using async readFile would introduce a microtask boundary, causing a race where cwd hasn't been updated yet when the caller continues.”

换句话说，如果这里使用 `readFile()`（返回 Promise），那么 `await shellCommand.result` 的调用方在继续执行时，CWD 的读取和更新可能还排在下一个微任务里，导致后续命令在错误的目录下执行。同步 API 消除了这个竞态。

此外，Windows 路径转换也在此处处理：`nativeCwdFilePath` 在 Windows 上会被 `posixPathToWindowsPath()` 转换，因为 Node.js 的 `readFileSync/unlinkSync` 需要原生 Windows 路径，而 bash 里的 `pwd -P` 输出的是 POSIX 路径。

### 3.7 环境变量注入

第 317–328 行：

```ts
env: {
  ...subprocessEnv(),
  SHELL: shellType === 'bash' ? binShell : undefined,
  GIT_EDITOR: 'true',
  CLAUDECODE: '1',
  ...envOverrides,
  ...(process.env.USER_TYPE === 'ant'
    ? { CLAUDE_CODE_SESSION_ID: getSessionId() }
    : {}),
},
```

- `subprocessEnv()`（`src/utils/subprocessEnv.ts`）：在 GitHub Actions 环境下会剥离 `ANTHROPIC_API_KEY`、`AWS_SECRET_ACCESS_KEY` 等敏感变量，防止 prompt injection 通过 shell 扩展泄露密钥；在非 GHA 环境下几乎无开销。
- `GIT_EDITOR: 'true'`：让 git 子命令在需要编辑器时直接成功退出（`true` 命令永远返回 0），避免挂起等待用户输入。
- `CLAUDECODE: '1'`：供子进程识别自己运行在 Claude Code 环境中。
- `envOverrides`：由 Provider 提供，bash 中可能包含 `TMUX`（隔离 socket）、`TMPDIR`（沙箱临时目录）、会话级 `/env` 变量；PowerShell 中同样包含会话变量和沙箱 `TMPDIR`。
- `CLAUDE_CODE_SESSION_ID`：仅对 `USER_TYPE === 'ant'` 的用户注入，用于内部追踪。

---

## 4. 关键代码路径与文件引用

### 4.1 本文件内的关键函数与行号

| 函数/变量 | 行号 | 说明 |
|-----------|------|------|
| `DEFAULT_TIMEOUT` | 44 | 30 分钟（30 * 60 * 1000） |
| `isExecutable()` | 50–68 | 双重可执行性校验 |
| `findSuitableShell()` | 73–137 | Shell 发现主逻辑 |
| `getShellConfigImpl()` | 139–143 | 异步获取 bash provider |
| `getShellConfig` | 146 | memoized 导出 |
| `getPsProvider` | 148–154 | memoized 获取 PowerShell provider |
| `resolveProvider` | 156–159 | 按 `ShellType` 映射到 provider getter |
| `ExecOptions` | 161–175 | `timeout`、`onProgress`、`onStdout` 等选项类型 |
| `exec()` | 181–442 | 主执行函数 |
| `setCwd()` | 447–474 | 设置并解析物理工作目录 |

### 4.2 `exec()` 内部的关键分支行号

| 逻辑 | 行号 | 说明 |
|------|------|------|
| CWD 恢复 | 220–238 | `realpath(cwd)` 失败后的 fallback |
| 中止信号前置检查 | 240–243 | `abortSignal.aborted` 时直接 `createAbortedCommand()` |
| 沙箱化 PowerShell 判断 | 256 | `isSandboxedPowerShell` |
| `SandboxManager.wrapWithSandbox()` | 260–265 | 沙箱包装 |
| 文件模式 / 管道模式判断 | 284 | `const usePipeMode = !!onStdout` |
| `O_NOFOLLOW` / Windows 'w' 打开文件 | 301–313 | 安全与兼容性处理 |
| `spawn()` 调用 | 316–337 | 子进程创建 |
| `outputHandle.close()` | 352–358 | 关闭父进程持有的 fd 副本 |
| `onStdout` 实时回调挂载 | 364–368 | 管道模式专属 |
| 同步 CWD 读取与清理 | 385–421 | `readFileSync` + `unlinkSync` |
| spawn 失败后的清理 | 424–441 | `createAbortedCommand(code: 126)` |

### 4.3 直接依赖的文件

| 依赖文件 | 被引用的符号 | 作用 |
|----------|--------------|------|
| `src/utils/ShellCommand.ts` | `wrapSpawn`, `createAbortedCommand`, `createFailedCommand`, `ShellCommand`, `ExecResult` | 子进程包装、结果 Promise、流生命周期 |
| `src/utils/task/TaskOutput.ts` | `TaskOutput` | 输出缓冲、进度轮询、文件/内存双模式 |
| `src/utils/task/diskOutput.ts` | `getTaskOutputDir` | 获取输出文件磁盘目录 |
| `src/utils/shell/bashProvider.ts` | `createBashShellProvider` | Bash 命令构建、快照恢复、环境变量 |
| `src/utils/shell/powershellProvider.ts` | `createPowerShellProvider` | PowerShell 命令构建、退出码处理、Base64 编码 |
| `src/utils/shell/shellProvider.ts` | `ShellProvider`, `ShellType` | Provider 抽象类型 |
| `src/utils/shell/powershellDetection.ts` | `getCachedPowerShellPath` | 探测 pwsh/powershell 安装位置 |
| `src/utils/sandbox/sandbox-adapter.ts` | `SandboxManager` | 沙箱配置、包装、清理 |
| `src/utils/subprocessEnv.ts` | `subprocessEnv` | GHA 环境下的敏感变量剥离 |
| `src/utils/which.ts` | `which` | 跨平台命令路径查找（优先 `Bun.which`） |
| `src/utils/cwd.ts` | `pwd` | 获取当前工作目录（支持 AsyncLocalStorage 覆盖） |
| `src/bootstrap/state.js` | `getOriginalCwd`, `getSessionId`, `setCwdState` | 会话级 CWD 与 ID 状态 |
| `src/utils/sessionEnvironment.js` | `invalidateSessionEnvCache` | CWD 变化后使环境缓存失效 |
| `src/utils/hooks/fileChangedWatcher.js` | `onCwdChangedForHooks` | CWD 变化时触发文件监听 hook |
| `src/utils/permissions/filesystem.ts` | `getClaudeTempDirName` | 获取按用户隔离的临时目录名 |
| `src/utils/platform.ts` | `getPlatform` | 平台判断（windows/linux/macos/wsl） |
| `src/utils/windowsPaths.ts` | `posixPathToWindowsPath` | Windows 路径格式转换 |
| `src/services/analytics/index.js` | `logEvent` | 分析事件上报 |

### 4.4 主要调用方

| 调用方文件 | 调用方式 | 场景 |
|------------|----------|------|
| `src/tools/BashTool/BashTool.tsx` | `exec(command, abortSignal, 'bash', options)` | 用户 Bash 命令 |
| `src/tools/PowerShellTool/PowerShellTool.tsx` | `exec(command, abortSignal, 'powershell', options)` | 用户 PowerShell 命令 |
| `src/commands/commit.ts` | `exec(...)` | Git 提交辅助命令 |
| `src/commands/security-review.ts` | `exec(...)` | 安全审查脚本执行 |
| `src/skills/loadSkillsDir.ts` | `exec(...)` | Skill 目录加载时的 shell 调用 |
| 其他大量模块 | 直接 `import { exec } from '../utils/Shell.js'` | 各类需要派生子进程的功能 |

---

## 5. 依赖与外部交互

### 5.1 Node.js 内置模块

- `child_process`：`spawn`（异步派生子进程）、`execFileSync`（`isExecutable` 的 fallback 校验）。
- `fs`：`readFileSync`、`unlinkSync`（CWD 追踪的同步清理）、`accessSync`（`X_OK` 校验）、`constants`（`O_WRONLY | O_CREAT | O_APPEND | O_NOFOLLOW`）。
- `fs/promises`：`mkdir`、`open`、`realpath`（异步文件操作）。

### 5.2 第三方库

- `lodash-es/memoize.js`：用于 `getShellConfig` 和 `getPsProvider` 的每会话缓存。
- `tree-kill`：在 `ShellCommandImpl.#doKill()` 中按 PID 递归杀死整个进程树，确保后台任务不会残留孤儿进程。

### 5.3 外部包 `@anthropic-ai/sandbox-runtime`

通过 `src/utils/sandbox/sandbox-adapter.ts` 间接依赖。`SandboxManager.wrapWithSandbox()` 会把命令字符串转换为包含 `bwrap`（bubblewrap）调用的完整命令。该包还负责：

- 网络隔离与域名白名单/黑名单。
- 文件系统读写限制（`allowWrite`/`denyWrite`/`denyRead`）。
- 违规事件收集（`SandboxViolationStore`）。
- 沙箱命令结束后的清理（`cleanupAfterCommand()`）以及裸 git repo 文件擦除（`scrubBareGitRepoFiles()`）。

### 5.4 平台特定行为

- **POSIX（macOS/Linux/WSL）**：支持沙箱；使用数值 `fsConstants` 打开输出文件；`detached: true`（bash provider）。
- **Windows**：不支持沙箱（`isSupportedPlatform()` 返回 false）；使用 `'w'` 字符串标志打开文件；PowerShell 路径探测需要处理 `pwsh.exe` / `powershell.exe`；路径需要在 POSIX 与 Windows 格式间转换。

---

## 6. 风险、边界与改进建议

### 6.1 已识别的风险与边界

#### 6.1.1 CWD 恢复的 TOCTOU 竞态

`realpath(cwd)` 检查与后续 `spawn()` 之间仍存在时间窗口：如果目录在检查通过后、spawn 前被删除，子进程仍然可能失败。不过这种情况概率极低，且 `spawn()` 本身的错误会被 `catch` 块捕获并返回 `createAbortedCommand(code: 126)`。

#### 6.1.2 文件模式下的输出文件被并发清理

`TaskOutput.#readStdoutFromFile()`（`src/utils/task/TaskOutput.ts` 第 297–326 行）处理了 `ENOENT` 的情况：若另一个 Claude Code 进程在启动清理时删除了输出文件，会返回一段诊断字符串而非空字符串，避免下游因空输出产生困惑。这是一个已知的跨进程边界问题。

#### 6.1.3 同步 I/O 阻塞事件循环

`readFileSync` / `unlinkSync` 在 `.then()` 微任务中虽然消除了 CWD 更新的竞态，但如果临时文件所在的磁盘 I/O 极慢（如网络文件系统），会短暂阻塞 Node.js 事件循环。考虑到 CWD 追踪文件通常只有几十字节且位于本地 `/tmp`，实际影响可忽略。

#### 6.1.4 沙箱化 PowerShell 的硬编码 `/bin/sh`

虽然注释说明“`/bin/sh` 存在于所有支持沙箱的平台上”，但如果未来支持某个 POSIX 不兼容的沙箱平台（如某些容器环境精简到没有 `/bin/sh`），这里会硬失败。当前这是一个可接受的假设，因为沙箱底层 `bwrap` 本身就依赖 POSIX。

#### 6.1.5 `O_NOFOLLOW` 在 Windows 上的缺失

Windows 分支使用字符串 `'w'` 标志，无法传递 `O_NOFOLLOW`。虽然 Windows 沙箱当前不被支持，但如果未来引入 Windows 沙箱方案，需要额外确保输出文件不会成为符号链接攻击的目标。

#### 6.1.6 `which()` 的 `Bun.which` 与 Node 回退差异

`src/utils/which.ts` 优先使用 `Bun.which`（同步底层实现），回退时则通过 `execa` 或 `execSync_DEPRECATED`  spawning `which` / `where.exe`。在极端 PATH 场景下，Bun 与系统 `which` 的结果可能不一致，但通常不会导致功能问题。

### 6.2 改进建议

#### 6.2.1 将 CWD 追踪文件放入 `getClaudeTempDir()` 而非系统 `/tmp`

当前非沙箱模式下，CWD 追踪文件直接放在 `os.tmpdir()`（`/tmp` 或 `%TEMP%`）。虽然文件名包含随机 ID 且及时清理，但统一放到 Claude 专属的临时目录（如 `getClaudeTempDirName()` 生成的按用户隔离目录）可以进一步降低权限冲突和信息泄露风险。

#### 6.2.2 为 `setCwd()` 增加 Unicode 规范化说明文档

`setCwd()` 内部没有显式调用 `normalize('NFC')`，而 `exec()` 的 CWD 比较逻辑（第 406 行）做了 NFC 比较。如果调用方直接调用 `setCwd()` 传入 NFD 路径，可能导致后续命令的 `newCwd.normalize('NFC') !== cwd` 误判为“已变化”。建议在 `setCwd()` 内部也统一做 NFC 规范化，保持行为一致。

#### 6.2.3 考虑为 `outputHandle.close()` 增加更明确的错误分类

当前 `outputHandle.close()` 的 catch 块是空的（第 355–357 行、第 428–431 行）。虽然常见错误是 fd 已被子进程关闭，但 `EIO` 等异常可能暗示底层文件系统问题。增加 `logForDebugging` 记录具体错误码有助于排查罕见故障。

#### 6.2.4 统一 `isExecutable()` 的 timeout 为可配置参数

当前 `execFileSync(shellPath, ['--version'])` 的 timeout 硬编码为 1000ms（第 60 行）。在极端负载或网络文件系统（NFS home）环境下，shell 的 `--version` 可能超过 1 秒，导致误判为不可执行。建议提取为常量或配置项。

#### 6.2.5 增加对 `CLAUDE_CODE_SHELL` 指向非 bash/zsh shell 的降级提示

当用户设置了 `CLAUDE_CODE_SHELL=/usr/bin/fish` 时，`findSuitableShell()` 会静默忽略并 fallback 到自动探测。可以在日志中增加 `logError` 级别的提示，告知用户“已忽略不支持的 shell 覆盖”，减少配置困惑。

---

## 7. 总结

`src/utils/Shell.ts` 是 Claude Code CLI 的命令执行中枢。它通过 `ShellProvider` 抽象统一了 Bash 与 PowerShell 的语义差异，在 `exec()` 中整合了沙箱隔离、CWD 追踪、超时控制、后台化、输出文件管理等一系列复杂能力。其设计在性能（文件模式绕过 JS）、安全（`O_NOFOLLOW`、环境变量剥离、沙箱）、可靠性（CWD 恢复、同步 CWD 更新）之间做了精细权衡。理解该文件，对于排查命令执行失败、沙箱行为异常、输出丢失或 CWD 漂移等问题至关重要。
