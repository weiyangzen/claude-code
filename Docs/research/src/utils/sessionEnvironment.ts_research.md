# 研究文档：src/utils/sessionEnvironment.ts

## 场景与职责

`sessionEnvironment.ts` 负责加载和管理**由用户 hook（如 `SessionStart`、`Setup`、`CwdChanged`、`FileChanged`）产生的环境变量脚本**。这些 hook 可以将环境变量定义写入 `.sh` 文件，而本模块负责在后续 Bash 命令执行前将这些脚本内容读取、拼接并注入到 shell 环境中。

它与 `sessionEnvVars.ts` 的区别：
- `sessionEnvVars.ts`：用户通过 `/env` 命令在内存中直接设置，即时生效。
- `sessionEnvironment.ts`：从磁盘上的 hook 输出文件加载，支持更复杂的多行 shell 脚本（如 `export FOO=bar && source venv/bin/activate`）。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `getSessionEnvDirPath()` | 返回并确保存在当前会话的 hook 环境脚本目录：`~/.claude/session-env/<sessionId>/`。 |
| `getHookEnvFilePath(hookEvent, hookIndex)` | 为指定 hook 事件和索引生成标准文件名（如 `sessionstart-hook-0.sh`）。 |
| `clearCwdEnvFiles()` | 清空 `cwdchanged-hook-*` 和 `filechanged-hook-*` 对应的环境文件（写入空字符串），用于目录切换或文件变更后重置旧环境。 |
| `invalidateSessionEnvCache()` | 将内存缓存标记为失效，强制下次读取时重新扫描磁盘。 |
| `getSessionEnvironmentScript()` | 核心加载逻辑：读取 `CLAUDE_ENV_FILE`（父进程传入）+ 所有 hook 环境文件，按优先级排序后拼接为单个大脚本字符串。 |

---

## 具体技术实现

### 1. 缓存策略

```ts
let sessionEnvScript: string | null | undefined = undefined
// undefined = 未加载（需检查磁盘）
// null = 已检查，无文件
// string = 已缓存
```

- `invalidateSessionEnvCache()` 将状态重置为 `undefined`。
- `getSessionEnvironmentScript()` 首次调用或缓存失效后才会执行磁盘 I/O。

### 2. 脚本来源与优先级

加载顺序（最终脚本按此顺序拼接）：

1. **`CLAUDE_ENV_FILE`**（若存在）
   - 用于父进程传递环境（如 HFI trajectory runner）。
   - 读取文件内容，trim 后若非空则加入 `scripts` 数组。

2. **Hook 环境文件**
   - 目录：`~/.claude/session-env/<sessionId>/`
   - 文件名匹配正则：`/^(setup|sessionstart|cwdchanged|filechanged)-hook-(\d+)\.sh$/`
   - 排序规则：
     - 先按 hook 类型优先级：`setup(0) < sessionstart(1) < cwdchanged(2) < filechanged(3)`
     - 同类型按 `hookIndex` 数字升序
   - 逐个读取、trim、非空则加入 `scripts`。

3. **拼接**
   - `scripts.join('\n')` 形成最终脚本。
   - 若 `scripts` 为空，缓存为 `null`。

### 3. Windows 不支持

```ts
if (getPlatform() === 'windows') {
  logForDebugging('Session environment not yet supported on Windows')
  return null
}
```

当前仅支持 POSIX shell 环境脚本，Windows 直接返回 `null`。

### 4. `clearCwdEnvFiles`

```ts
export async function clearCwdEnvFiles(): Promise<void> {
  const dir = await getSessionEnvDirPath()
  const files = await readdir(dir)
  await Promise.all(
    files
      .filter(f => (f.startsWith('filechanged-hook-') || f.startsWith('cwdchanged-hook-')) && HOOK_ENV_REGEX.test(f))
      .map(f => writeFile(join(dir, f), ''))
  )
}
```

- 通过写入空字符串来“清空”文件，而不是删除。这样保留了文件存在性，避免某些文件监控或排序逻辑因文件消失而异常。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/sessionEnvironment.ts:13-23` | `getSessionEnvDirPath` 与会话目录创建。 |
| `src/utils/sessionEnvironment.ts:25-31` | `getHookEnvFilePath` 文件名生成。 |
| `src/utils/sessionEnvironment.ts:33-53` | `clearCwdEnvFiles` 实现。 |
| `src/utils/sessionEnvironment.ts:55-144` | `getSessionEnvironmentScript` 核心加载与缓存逻辑。 |
| `src/utils/sessionEnvironment.ts:146-166` | `sortHookEnvFiles` 排序规则。 |
| `src/utils/shell/bashProvider.ts:169-173` | 调用方：Bash provider 在构建命令时 `source` 会话环境脚本。 |
| `src/utils/hooks.ts:925` | 调用方：hook 执行前设置 `CLAUDE_ENV_FILE` 环境变量指向本模块管理的文件。 |
| `src/utils/hooks/fileChangedWatcher.ts` | 调用方：文件变更 watcher 可能触发 `invalidateSessionEnvCache`。 |
| `src/utils/hooks/AsyncHookRegistry.ts` | 调用方：异步 hook 注册表可能涉及环境缓存失效。 |
| `src/utils/Shell.ts` | 可能通过 shell 命令触发环境变更。 |
| `src/commands/env/current_folder_research.md` | 相关命令文档。 |

---

## 依赖与外部交互

- **Node.js 内置模块**：`fs/promises`（`mkdir`, `readdir`, `readFile`, `writeFile`）、`path`（`join`）。
- **内部依赖**：
  - `../bootstrap/state.js`：`getSessionId`
  - `./debug.js`：`logForDebugging`
  - `./envUtils.js`：`getClaudeConfigHomeDir`
  - `./errors.js`：`errorMessage`, `getErrnoCode`
  - `./platform.js`：`getPlatform`
- **调用方**：
  - `src/utils/shell/bashProvider.ts`（Bash 命令构建）
  - `src/utils/hooks.ts`（hook 执行环境变量设置）
  - `src/utils/hooks/fileChangedWatcher.ts`（缓存失效）

---

## 风险、边界与改进建议

### 风险与边界

1. **Windows 完全不支持**：当前直接返回 `null`，意味着 Windows 用户无法通过 hook 持久化环境变量。随着 PowerShell hook 支持的推进，这块需要补齐对应的 `.ps1` 环境脚本加载逻辑。

2. **`clearCwdEnvFiles` 的竞态**：该函数使用 `Promise.all` 并发写入空字符串。若此时 `getSessionEnvironmentScript` 正在读取同一批文件，可能读到部分已清空、部分仍含内容的混合状态。虽然概率低，但在高频文件变更场景下存在竞态窗口。

3. **缓存失效粒度粗**：`invalidateSessionEnvCache` 将整个缓存置为失效，无法针对单个 hook 文件做细粒度更新。考虑到 hook 文件数量通常 <10，这不是性能瓶颈。

4. **脚本内容无校验**：直接读取 `.sh` 文件内容并拼接到 Bash 命令中执行。若文件被恶意篡改（如注入 `; rm -rf /`），会造成命令注入。但这些文件位于用户的 `~/.claude/session-env/` 目录下，且由用户自己的 hook 写入，信任边界与用户本身一致。

5. **`CLAUDE_ENV_FILE` 的读取失败静默处理**：若 `CLAUDE_ENV_FILE` 指向的文件不存在（`ENOENT`），仅记录 debug 日志并继续。这是合理的容错，但如果父进程期望该文件被读取，失败信息可能被淹没在 debug 日志中。

6. **Hook 文件排序的稳定性**：排序依赖文件名中的数字前缀。若用户手动创建文件时使用了非数字前缀（如 `setup-hook-abc.sh`），正则匹配会失败，文件被忽略。这是有意的设计，但缺少对用户的可见提示。

### 改进建议

1. **Windows PowerShell 支持**：为 Windows 增加 `.ps1` 环境脚本支持。可在 `getSessionEnvironmentScript` 中根据平台返回不同格式：
   - POSIX → 拼接 `.sh` 内容
   - Windows → 读取 `.ps1` 文件并转换为 PowerShell 语法（或保持原样由 `powershellProvider.ts` 处理）

2. **增加文件锁或原子读写**：在 `clearCwdEnvFiles` 和 `getSessionEnvironmentScript` 之间引入简单的文件级锁（如通过 `lockfile` 包或自定义 lock 文件），避免读写竞态。

3. **缓存失效事件驱动化**：当前由调用方显式调用 `invalidateSessionEnvCache`。可改为使用 `fs.watch` 监控 `session-env/<sessionId>` 目录，文件变化时自动失效缓存，减少调用方负担。

4. **增加脚本内容摘要日志**：在 `getSessionEnvironmentScript` 返回非空脚本时，记录加载了哪些文件、总字符数，帮助用户排查环境未生效的问题（如 hook 写入了空文件或写到了错误路径）。

5. **统一 `sessionEnvVars` 与 `sessionEnvironment` 的暴露接口**：考虑在 `bashProvider.ts` 中增加一层统一的环境变量合并逻辑，明确优先级：
   - Hook 环境脚本（最低优先级，可被覆盖）
   - `/env` 设置的 `sessionEnvVars`（较高优先级）
   - 命令级显式 `env` 覆盖（最高优先级）
   当前这些合并逻辑分散在 `bashProvider.ts` 和 `powershellProvider.ts` 中，建议抽象到独立模块。
