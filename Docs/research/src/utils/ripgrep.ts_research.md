# 研究文档：src/utils/ripgrep.ts

## 场景与职责

`ripgrep.ts` 是 Claude Code 中所有基于 `ripgrep`（`rg`）的文件搜索、索引、 glob 与计数的**统一封装层**。它屏蔽了不同运行环境下的 ripgrep 来源差异（系统安装版、内置 vendor 版、Bun embedded 版），并提供了：

1. **基础搜索** `ripGrep()`：返回匹配行数组，带 EAGAIN 重试、超时处理、部分结果返回。
2. **流式搜索** `ripGrepStream()`：逐 chunk 回调结果，用于交互式实时搜索（如 `fzf change:reload` 模式）。
3. **文件计数** `ripGrepFileCount()`：仅统计 `rg --files` 的行数，用于遥测（仓库文件量估算）。
4. **配置探测** `ripgrepCommand()` / `getRipgrepStatus()`：决定使用 system / builtin / embedded 哪种模式。
5. **macOS 代码签名** `codesignRipgrepIfNecessary()`：对 vendor 版二进制做 ad-hoc 签名与隔离属性移除，防止 Gatekeeper 拦截。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `ripgrepCommand()` | 根据环境决定 ripgrep 可执行文件路径与参数。支持 system（用户强制）、embedded（Bun bundled）、builtin（vendor 预编译）三种模式。 |
| `ripGrepRaw()` | 底层进程启动：embedded 模式用 `spawn`（支持 `argv0`），非 embedded 用 `execFile`；统一超时、SIGKILL 升级、stdout/stderr 截断保护。 |
| `ripGrep()` | 高层封装：处理 exit code 1（无匹配视为成功）、EAGAIN 单线程重试、关键错误码（ENOENT/EACCES/EPERM）直接抛错、超时/缓冲区溢出时返回部分结果。 |
| `ripGrepStream()` | 流式接口：按 stdout chunk 回调完整行，支持 early abort，适合交互式 UI 实时渲染。 |
| `ripGrepFileCount()` | 超轻量计数：不缓存完整 stdout，只按 chunk 数换行符，峰值内存 ~64KB。 |
| `countFilesRoundedRg()` | 对指定目录做文件计数，并按最近 10 的幂取整（隐私保护），结果 memoize。 |
| `testRipgrepOnFirstUse()` | 首次使用时跑 `--version` 测试，缓存可用性状态并上报 telemetry。 |
| `codesignRipgrepIfNecessary()` | macOS 上为 vendor 版 rg 做 ad-hoc codesign 并移除 quarantine xattr，解决 Gatekeeper 阻止问题。 |

---

## 具体技术实现

### 1. ripgrep 来源配置：`getRipgrepConfig`

使用 `lodash-es/memoize` 缓存，决策逻辑：

```ts
const getRipgrepConfig = memoize((): RipgrepConfig => {
  // 1. 用户强制使用系统版
  if (userWantsSystemRipgrep) {
    const { cmd: systemPath } = findExecutable('rg', [])
    if (systemPath !== 'rg') {
      // SECURITY: 使用 'rg' 而非绝对路径，防止当前目录下的 ./rg.exe 被恶意执行
      return { mode: 'system', command: 'rg', args: [] }
    }
  }

  // 2. Bundled (Bun native模式) — rg 静态编译在 bun-internal 中，通过 argv0='rg' 分发
  if (isInBundledMode()) {
    return {
      mode: 'embedded',
      command: process.execPath,
      args: ['--no-config'],
      argv0: 'rg',
    }
  }

  // 3. 内置 vendor 二进制
  const rgRoot = path.resolve(__dirname, 'vendor', 'ripgrep')
  const command = process.platform === 'win32'
    ? path.resolve(rgRoot, `${process.arch}-win32`, 'rg.exe')
    : path.resolve(rgRoot, `${process.arch}-${process.platform}`, 'rg')
  return { mode: 'builtin', command, args: [] }
})
```

**安全设计**：system 模式下即使找到了系统路径，也使用命令名 `'rg'` 让 OS 通过 `NoDefaultCurrentDirectoryInExePath` 解析，避免 PATH 劫持。

### 2. 底层进程启动：`ripGrepRaw`

**Embedded 分支（`spawn`）**：
- 必须支持 `argv0`，`execFile` 对 `argv0` 支持不佳，因此使用 `spawn`。
- 手动拼接 `stdout` / `stderr`，超过 `MAX_BUFFER_SIZE = 20_000_000`（20MB）时截断。
- 超时后先 `SIGTERM`，5s 未结束则升级 `SIGKILL`；Windows 下只能调用默认 `kill()`。
- `'close'` 与 `'error'` 事件做 `settled` 去重，防止 Windows 上同一进程同时触发两个事件导致双回调。

**非 Embedded 分支（`execFile`）**：
- 直接使用 `execFile`，利用其内置的 `maxBuffer` 与 `timeout`。
- `killSignal` 在非 Windows 平台设为 `'SIGKILL'`，因为 ripgrep 可能在不可中断 I/O 中阻塞，`SIGTERM` 杀不掉。

**超时默认值**：
- WSL 文件读取慢（3-5× 性能惩罚），默认 60s；其他平台 20s。
- 可通过 `CLAUDE_CODE_GLOB_TIMEOUT_SECONDS` 环境变量覆盖。

### 3. 高层错误处理：`ripGrep`

```ts
export async function ripGrep(args, target, abortSignal): Promise<string[]>
```

处理矩阵：

| 场景 | 行为 |
|------|------|
| `error === null` (exit 0) | 返回过滤后的行数组 |
| `error.code === 1` | rg 正常“无匹配”，返回 `[]` |
| `error.code ∈ {ENOENT, EACCES, EPERM}` | ripgrep 本身不可用，直接 `reject` |
| `stderr` 含 EAGAIN | 单线程重试一次（`-j 1`），仅本次调用生效，不持久化 |
| 超时 / 缓冲区溢出 | 若有部分 stdout，去掉最后一行（可能不完整）后返回；无结果则抛 `RipgrepTimeoutError` |
| 其他错误 | `logError` 后返回部分结果或 `[]` |

`RipgrepTimeoutError` 是自定义 Error 子类，携带 `partialResults`，允许调用方区分“无匹配”与“搜索未完成”。

### 4. 流式接口：`ripGrepStream`

- 使用 `spawn` + `stdio: ['ignore', 'pipe', 'ignore']`。
- 维护 `remainder` 变量处理跨 chunk 的不完整行。
- 若 `abortSignal.aborted`，在 `'close'` 中丢弃 `remainder`（避免 killed process 的撕裂尾部被误 flush）。
- 无内部超时、无 EAGAIN 重试、stderr 忽略；调用方（交互式 UI）自行负责恢复策略。

### 5. 文件计数：`ripGrepFileCount`

- 同样使用 `spawn` + `stdio` 忽略 stdin/stderr。
- 不缓存 stdout，只计数 `\n`：
  ```ts
  child.stdout?.on('data', (chunk: Buffer) => {
    lines += countCharInString(chunk, '\n')
  })
  ```
- 峰值内存仅一个 stream chunk（默认 ~64KB）。

### 6. macOS 代码签名：`codesignRipgrepIfNecessary`

仅在 `process.platform === 'darwin'` 且 `mode === 'builtin'` 时执行一次：

1. `codesign -vv -d <builtinPath>` 检查签名状态。
2. 若输出包含 `linker-signed`，则执行 ad-hoc 签名：
   ```
   codesign --sign - --force --preserve-metadata=entitlements,requirements,flags,runtime <path>
   ```
3. 移除 quarantine xattr：
   ```
   xattr -d com.apple.quarantine <path>
   ```
4. 任何步骤失败都仅 `logError`，不阻塞启动。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/ripgrep.ts:31-65` | `getRipgrepConfig` 配置决策。 |
| `src/utils/ripgrep.ts:108-232` | `ripGrepRaw` 底层进程启动（spawn / execFile 双分支）。 |
| `src/utils/ripgrep.ts:246-281` | `ripGrepFileCount` 轻量计数。 |
| `src/utils/ripgrep.ts:295-343` | `ripGrepStream` 流式接口。 |
| `src/utils/ripgrep.ts:345-463` | `ripGrep` 高层封装与错误处理。 |
| `src/utils/ripgrep.ts:476-522` | `countFilesRoundedRg` 隐私取整计数。 |
| `src/utils/ripgrep.ts:551-617` | `testRipgrepOnFirstUse` 首次可用性测试。 |
| `src/utils/ripgrep.ts:620-679` | `codesignRipgrepIfNecessary` macOS 签名。 |
| `src/tools/GrepTool/GrepTool.ts` | 主要调用方：Grep 工具实现。 |
| `src/tools/GlobTool/GlobTool.ts` | Glob 工具内部使用 `ripGrep` / `ripGrepStream`。 |
| `src/utils/glob.ts` | 文件 glob 使用 `ripGrep`。 |
| `src/main.tsx:414` | `countFilesRoundedRg` 在启动 prefetch 中被调用。 |

---

## 依赖与外部交互

- **第三方库**：`lodash-es/memoize`。
- **Node.js 内置模块**：`child_process`（`execFile`, `spawn`）、`os`（`homedir`）、`path`、`url`。
- **内部依赖**：
  - `./bundledMode.js`：`isInBundledMode`
  - `./debug.js`：`logForDebugging`
  - `./envUtils.js`：`isEnvDefinedFalsy`
  - `./execFileNoThrow.js`：`execFileNoThrow`
  - `./findExecutable.js`：`findExecutable`
  - `./log.js`：`logError`
  - `./platform.js`：`getPlatform`
  - `./stringUtils.js`：`countCharInString`
  - `src/services/analytics/index.js`：`logEvent`
- **外部二进制**：系统 `rg`、vendor 目录下的 `rg`/`rg.exe`、macOS `codesign`/`xattr`。
- **调用方**：GrepTool、GlobTool、file-index、doctor、permissions、bash 快照、settings、markdownConfigLoader 等 20+ 处。

---

## 风险、边界与改进建议

### 风险与边界

1. **Windows `killSignal: undefined` 的语义**：非 embedded 分支在 Windows 上 `killSignal` 设为 `undefined`，此时 `execFile` 默认行为是 `SIGTERM`。但 Windows 没有 POSIX 信号，`child_process` 会调用 `TerminateProcess`，效果等同于强制结束。注释中提到“SIGKILL throws”可能是历史遗留，实际上 Node.js 在 Windows 上处理 `SIGKILL` 通常是允许的（通过 `taskkill /F` 或 `TerminateProcess`）。若实际测试中 `SIGKILL` 不再抛错，可统一行为。

2. **EAGAIN 重试的竞态**：`isEagainError` 只检查 stderr 字符串是否包含 `os error 11` 或 `Resource temporarily unavailable`。如果 stderr 为空或语言本地化导致消息不同，重试不会触发。此外，重试时使用的是同一 `abortSignal`，若原信号已触发超时，重试无意义。

3. **`countFilesRoundedRg` 的 memoize 缓存**：使用 lodash 的 `memoize`，自定义 resolver 排除了 `abortSignal`。这是正确的，但 memoize 的缓存永远不会被清除，在长时间运行的进程（如 REPL）中，若目录内容变化，计数结果会 stale。当前仅用于启动 telemetry，影响有限。

4. **vendor 目录路径硬编码**：`__dirname` 的计算在 `NODE_ENV === 'test'` 时做了特殊偏移（`'../../../'` vs `'../'`）。若测试运行器的工作目录或打包工具（如 esbuild）改变目录结构，此路径可能失效。

5. **`codesignRipgrepIfNecessary` 的单例状态**：`alreadyDoneSignCheck` 是模块级布尔值，仅保证同进程内执行一次。多进程并发启动时可能同时执行 codesign，虽然 macOS 的 `codesign` 本身对同一文件并发调用通常是安全的，但日志可能产生噪音。

### 改进建议

1. **统一 Windows kill 信号**：验证 Node.js 当前版本在 Windows 上 `execFile` 对 `SIGKILL` 的支持，若已支持则移除平台分支，统一使用 `SIGKILL`，减少认知负担。

2. **EAGAIN 检测增强**：除 stderr 文本外，可结合 `error.code === 'EAGAIN'` 做双重判断，提高在资源受限环境中的检测率。

3. **`ripGrepStream` 增加 `stderr` 透传选项**：当前流式接口完全忽略 stderr，某些调用方（如调试模式）可能希望看到 rg 的 warning。可新增可选的 `onStderr` 回调。

4. **代码签名步骤异步化**：`codesignRipgrepIfNecessary` 被 `await` 在每次 `ripGrep` / `ripGrepStream` / `ripGrepFileCount` 之前。虽然首次之后是 no-op，但函数调用开销仍然存在。可将签名检查提前到进程启动时（如 `main.tsx`），使搜索路径零额外开销。

5. **增加 `rg` 版本兼容性检查**：`testRipgrepOnFirstUse` 只检查 stdout 是否以 `ripgrep ` 开头。未来若 vendor 的 rg 版本与调用方使用的 flags 不兼容（如旧版不支持某些 flag），应在测试阶段捕获并降级或报错。
