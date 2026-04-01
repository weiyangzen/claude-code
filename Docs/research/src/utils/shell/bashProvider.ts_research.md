# bashProvider.ts 研究文档

## 场景与职责

`bashProvider.ts` 是 Claude Code CLI 的 **Bash  Shell 执行 provider**，负责将用户输入的裸命令字符串组装成可在 bash/zsh 进程中安全执行的完整命令串。它是 `ShellProvider` 接口的 bash 实现，被 `src/utils/Shell.ts` 的 `getShellConfig()` 统一调用。核心职责包括：

- 构建带环境快照（snapshot）、会话环境变量、extglob 安全关闭、eval 包装、cwd 追踪的完整命令串。
- 为子进程生成 spawn 参数（`-c` / `-l` 动态决策）。
- 提供环境变量覆盖（tmux socket 隔离、sandbox TMPDIR、session env vars）。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `createBashShellProvider(shellPath)` | 工厂函数，返回一个符合 `ShellProvider` 接口的 bash provider 对象。 |
| `buildExecCommand(command, opts)` | 把用户命令包装成可在目标 shell 中执行的完整字符串，并返回 cwd 追踪文件路径。 |
| `getSpawnArgs(commandString)` | 根据是否使用了 snapshot 决定加不加 `-l`（login shell）。 |
| `getEnvironmentOverrides(command)` | 返回需要在子进程中额外设置的环境变量。 |
| `getDisableExtglobCommand(shellPath)` | 安全防御：关闭 bash extglob / zsh EXTENDED_GLOB，防止恶意文件名利用 glob 扩展绕过权限校验。 |

## 具体技术实现

### 1. Snapshot 回退与 TOCTOU 处理

```ts
const snapshotPromise = options?.skipSnapshot
  ? Promise.resolve(undefined)
  : createAndSaveSnapshot(shellPath).catch(...)
```

- 首次创建 provider 时异步生成环境快照文件（`ShellSnapshot.ts`）。
- 每次 `buildExecCommand` 时用 `fs.access()` 检查快照是否仍存在；若被清理则清空 `lastSnapshotFilePath`，`getSpawnArgs` 会补回 `-l` 让 shell 走登录初始化流程。
- 注释明确说明：这不是纯 TOCTOU 修复，而是 fallback 决策点；`source ... || true` 仍保护真正的竞态窗口。

### 2. 命令组装流程

生成的命令串结构如下（以有 snapshot 为例）：

```bash
source <snapshot> 2>/dev/null || true && \
<sessionEnvScript> && \
<disableExtglob> && \
eval <quotedCommand> && \
pwd -P >| <cwdFilePath>
```

- `quoteShellCommand` 来自 `shellQuoting.js`，会对命令做 POSIX 安全引号处理。
- `rewriteWindowsNullRedirect` 防御模型误发 Windows 风格 `2>nul` 导致在 Git Bash 中生成名为 `nul` 的文件（issue #4928）。
- `rearrangePipeCommand` 解决 pipe 场景下 stdin redirect 被放到 eval 之后的问题：`rg foo | wc -l < /dev/null` 会导致 rg 永远等待。

### 3. Windows 路径处理

- `shellCwdFilePath` 使用 POSIX 路径（`path/posix`），供 bash 内部 `pwd -P >| ...` 使用。
- `cwdFilePath` 使用原生 OS 路径（`path`），供 Node.js 的 `readFileSync` / `unlinkSync` 使用。
- snapshot 路径在 Windows 上通过 `windowsPathToPosixPath` 转换后再 quote。

### 4. Tmux Socket 隔离（Deferred）

```ts
if (process.env.USER_TYPE === 'ant' && (hasTmuxToolBeenUsed() || commandUsesTmux)) {
  await ensureSocketInitialized()
}
```

- 只有 ant 用户且首次使用 Tmux 工具或命令含 "tmux" 时才初始化隔离 socket，避免普通会话承担启动成本。
- 通过覆盖 `TMUX` 环境变量，让所有 bash 子进程的 tmux 命令指向 Claude 私有 socket，防止误杀用户 tmux 会话。

### 5. Sandbox TMPDIR

- 若 `opts.sandboxTmpDir` 存在，设置 `TMPDIR`、`CLAUDE_CODE_TMPDIR`，并为 zsh 额外设置 `TMPPREFIX`（zsh heredoc 临时文件不尊守 TMPDIR）。

## 关键代码路径与文件引用

| 文件 | 关系 | 说明 |
|------|------|------|
| `src/utils/shell/bashProvider.ts` | 本文件 | Provider 实现。 |
| `src/utils/shell/shellProvider.ts` | 被实现 | `ShellProvider` 类型定义。 |
| `src/utils/Shell.ts` | 调用方 | `getShellConfig()` → `createBashShellProvider()` → `exec()` 中使用 provider 方法。 |
| `src/utils/bash/ShellSnapshot.ts` | 被调用 | 创建并保存 shell 环境快照。 |
| `src/utils/bash/shellQuoting.ts` | 被调用 | `quoteShellCommand`、`rewriteWindowsNullRedirect`、`shouldAddStdinRedirect`。 |
| `src/utils/bash/bashPipeCommand.ts` | 被调用 | `rearrangePipeCommand` 处理 pipe 重定向。 |
| `src/utils/bash/shellPrefix.ts` | 被调用 | `formatShellPrefixCommand` 处理 `CLAUDE_CODE_SHELL_PREFIX`。 |
| `src/utils/tmuxSocket.ts` | 被调用 | Tmux 隔离 socket 初始化与状态查询。 |
| `src/utils/sessionEnvVars.ts` | 被调用 | 读取 `/env` 设置的会话级环境变量。 |
| `src/utils/sessionEnvironment.ts` | 被调用 | 获取会话启动 hook 注入的环境脚本。 |

## 依赖与外部交互

- **Node.js 内置**: `fs/promises` (`access`), `os` (`tmpdir`), `path`, `path/posix`。
- **Bun 特性**: `import { feature } from 'bun:bundle'` 用于 `COMMIT_ATTRIBUTION` 调试日志开关。
- **环境变量**: `CLAUDE_CODE_SHELL_PREFIX`、`USER_TYPE`、`CLAUDE_CODE_TMPDIR`。
- **外部进程**: 通过 `spawn` 启动 bash/zsh；通过 `tmux` 命令维护隔离 socket。

## 风险、边界与改进建议

### 风险

1. **Snapshot 竞态**: `access()` 检查与 `source` 之间仍有微小窗口，虽然加了 `|| true`，但如果 snapshot 在窗口内消失且未触发 access 失败，命令将无环境变量运行。当前实现通过 `|| true` 保证不中断命令执行。
2. **Windows `nul` 文件污染**: 模型可能输出 `2>nul`，`rewriteWindowsNullRedirect` 已做防御，但仍需持续观察新出现的 Windows CMD 重定向语法。
3. **Pipe + stdin redirect 组合**: `rearrangePipeCommand` 只处理含 `|` 且需要 stdin redirect 的场景，若未来出现更复杂的嵌套 eval，可能需要更完整的 shell parser。

### 边界

- 仅支持 `bash` 和 `zsh`；其他 shell（fish、tcsh 等）不做 extglob 处理。
- `getDisableExtglobCommand` 通过字符串包含判断 `'bash'` / `'zsh'`，对如 `/usr/local/bin/bash` 有效，但对自定义命名的 shell 二进制可能失效。
- Tmux socket 隔离仅对 `USER_TYPE === 'ant'` 启用，普通用户不享受该隔离。

### 改进建议

1. **更精确的 shell 检测**: 可用 `basename(shellPath)` 或执行 `$0 --version` 判断 shell 类型，而不是字符串包含。
2. **Snapshot 持久化**: 考虑将 snapshot 写到项目临时目录（`CLAUDE_CODE_TMPDIR`）而非系统 `tmpdir`，降低被系统清理的概率。
3. **Pipe 处理通用化**: 当前 `rearrangePipeCommand` 是特化修复，若引入完整 shlex parser 可一劳永逸解决重定向位置问题。
4. **测试覆盖**: 当前仓库中未找到针对 `bashProvider.ts` 的独立单元测试，建议补充 snapshot 缺失回退、Windows 路径转换、pipe 重定向等场景的测试。
