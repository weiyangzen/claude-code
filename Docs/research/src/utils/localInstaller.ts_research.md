# src/utils/localInstaller.ts 研究文档

## 场景与职责

`localInstaller.ts` 管理 Claude Code CLI 的本地安装机制。与全局 `npm install -g @anthropic-ai/claude-code` 不同，本地安装将 CLI 包安装到 `~/.claude/local/` 目录下，并创建一个包装脚本 `~/.claude/local/claude`。这种方式有以下优势：

1. **避免权限问题**：不需要 `sudo` 或全局 npm 目录写权限。
2. **隔离版本**：每个用户有自己的本地版本，不影响系统其他部分。
3. **自动更新**：可以通过本地 npm 命令更新，而不依赖全局包管理器。

该模块被自动更新器、设置向导和诊断工具调用。

## 功能点目的

### 1. `ensureLocalPackageEnvironment`
确保本地安装环境已就绪：
- 创建 `~/.claude/local/` 目录。
- 创建 `package.json`（`{ name: 'claude-local', version: '0.0.1', private: true }`）。
- 创建包装脚本 `~/.claude/local/claude`：
  ```sh
  #!/bin/sh
  exec "~/.claude/local/node_modules/.bin/claude" "$@"
  ```
- 设置脚本权限为 `0o755`。

使用 `writeIfMissing` 原子创建（`O_EXCL` 语义），避免覆盖用户已有的自定义配置。

### 2. `installOrUpdateClaudePackage`
在本地目录中安装或更新 Claude CLI 包：
- 先调用 `ensureLocalPackageEnvironment`。
- 根据 `channel`（`stable` 或 `latest`）或 `specificVersion` 构建版本规范。
- 执行 `npm install ${MACRO.PACKAGE_URL}@${versionSpec}`。
- 安装成功后，将全局配置的 `installMethod` 设为 `'local'`。
- 返回状态：`'in_progress'`、`'success'` 或 `'install_failed'`。

### 3. `localInstallationExists`
检查本地安装是否已存在（通过检查 `~/.claude/local/node_modules/.bin/claude` 是否存在）。

### 4. `isRunningFromLocalInstallation`
判断当前进程是否就是从本地安装目录运行的：
```typescript
export function isRunningFromLocalInstallation(): boolean {
  const execPath = process.argv[1] || ''
  return execPath.includes('/.claude/local/node_modules/')
}
```

### 5. `getShellType`
根据 `process.env.SHELL` 推断用户使用的 shell 类型（`zsh`、`bash`、`fish` 或 `unknown`）。

## 具体技术实现

### 路径延迟求值
```typescript
function getLocalInstallDir(): string {
  return join(getClaudeConfigHomeDir(), 'local')
}
```

使用函数而非模块级常量，是因为 `getClaudeConfigHomeDir()` 读取 `process.env.CLAUDE_CONFIG_DIR`，而某些入口点（如 `hfi.tsx`）需要在 `main()` 中设置该变量。如果在模块加载时就求值，会捕获到旧值。

### 原子写入
```typescript
async function writeIfMissing(path: string, content: string, mode?: number): Promise<boolean> {
  try {
    await writeFile(path, content, { encoding: 'utf8', flag: 'wx', mode })
    return true
  } catch (e) {
    if (getErrnoCode(e) === 'EEXIST') return false
    throw e
  }
}
```

- `flag: 'wx'` 确保文件不存在时才创建，防止 TOCTOU 竞态。
- 对于包装脚本，虽然 `writeFile` 可以指定 `mode`，但 umask 可能屏蔽执行位，因此额外调用 `chmod(wrapperPath, 0o755)`。

### npm 安装错误处理
```typescript
if (result.code !== 0) {
  const error = new Error(`Failed to install Claude CLI package: ${result.stderr}`)
  logError(error)
  return result.code === 190 ? 'in_progress' : 'install_failed'
}
```

- 退出码 190 被特殊处理为 `'in_progress'`。这可能是 npm 的某种特定退出码（如操作正在进行中），但注释中没有详细说明。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/localInstaller.ts:19-24` | `getLocalInstallDir` / `getLocalClaudePath` 延迟路径求值 |
| `src/utils/localInstaller.ts:29-32` | `isRunningFromLocalInstallation` 运行环境检测 |
| `src/utils/localInstaller.ts:38-50` | `writeIfMissing` 原子写入 |
| `src/utils/localInstaller.ts:56-90` | `ensureLocalPackageEnvironment` 环境初始化 |
| `src/utils/localInstaller.ts:97-138` | `installOrUpdateClaudePackage` 安装/更新 |
| `src/utils/localInstaller.ts:144-151` | `localInstallationExists` 存在性检查 |
| `src/utils/localInstaller.ts:156-162` | `getShellType` shell 类型推断 |
| `src/utils/config.ts` | `ReleaseChannel`, `saveGlobalConfig` |
| `src/utils/envUtils.ts` | `getClaudeConfigHomeDir` |
| `src/utils/execFileNoThrow.ts` | `execFileNoThrowWithCwd` |
| `src/utils/fsOperations.ts` | `getFsImplementation` |
| `src/utils/log.ts` | `logError` |
| `src/utils/slowOperations.ts` | `jsonStringify` |

## 依赖与外部交互

### 内部依赖
- `./config.js`：`ReleaseChannel`, `saveGlobalConfig`
- `./envUtils.js`：`getClaudeConfigHomeDir`
- `./errors.js`：`getErrnoCode`
- `./execFileNoThrow.js`：`execFileNoThrowWithCwd`
- `./fsOperations.js`：`getFsImplementation`
- `./log.js`：`logError`
- `./slowOperations.js`：`jsonStringify`
- `fs/promises`：`access`, `chmod`, `writeFile`
- `path`：`join`

### 外部依赖
- `npm` 命令行工具（通过 `execFileNoThrowWithCwd` 调用）

### 调用方
- `src/components/AutoUpdater.tsx`
- `src/utils/shellConfig.ts`
- `src/utils/doctorDiagnostic.ts`
- `src/utils/localInstaller.ts`（自引用，无实际递归）

## 风险、边界与改进建议

### 风险与边界
1. **`isRunningFromLocalInstallation` 的硬编码路径**：检查 `/.claude/local/node_modules/` 是否存在于 `process.argv[1]` 中。如果用户通过符号链接、自定义 `CLAUDE_CONFIG_DIR` 或其他非标准方式运行，该检测可能失效。
2. **npm 依赖**：`installOrUpdateClaudePackage` 直接调用系统 `npm` 命令。如果用户环境中没有 npm（如只安装了 Bun 或 pnpm），安装会失败。虽然大多数 Node.js 用户都有 npm，但这并非绝对保证。
3. **网络与权限失败**：`npm install` 可能因网络问题、代理配置、磁盘空间不足等原因失败。当前错误处理只记录了 stderr 和退出码，没有提供具体的故障排除建议。
4. **`MACRO.PACKAGE_URL` 的构建时依赖**：包 URL 是通过构建宏注入的。如果构建配置发生变化或宏未定义，可能导致安装错误的包或空字符串。
5. **shell 类型推断过于简单**：`getShellType` 只检查 `process.env.SHELL` 是否包含 `zsh`/`bash`/`fish`。对于使用其他 shell（如 `nushell`、`xonsh`）或 Windows（无 `SHELL` 环境变量）的用户，会返回 `'unknown'`。

### 改进建议
1. **支持多种包管理器**：检测环境中可用的包管理器（npm、pnpm、yarn、bun），优先使用与当前运行时一致的包管理器安装本地包。
2. **增强 `isRunningFromLocalInstallation`**：除了检查路径字符串，还可以检查 `require.resolve('@anthropic-ai/claude-code')` 是否指向 `~/.claude/local/node_modules` 下。
3. **安装失败的用户提示**：在返回 `'install_failed'` 时，附带常见的故障排除链接或建议（如检查网络连接、npm 配置、磁盘空间）。
4. **离线安装支持**：考虑缓存最近下载的包 tarball，在网络不可用时尝试从本地缓存安装。
5. **Windows 兼容性**：`getShellType` 在 Windows 上通常返回 `'unknown'`。可以考虑检查 `process.env.COMSPEC` 或 `process.env.PATHEXT` 来推断 PowerShell/CMD 使用。
