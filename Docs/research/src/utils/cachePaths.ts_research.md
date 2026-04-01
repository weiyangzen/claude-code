# src/utils/cachePaths.ts 深入研究

## 场景与职责

`cachePaths.ts` 为 Claude Code 提供**跨平台、项目隔离的缓存目录路径**。它基于 `env-paths` 库生成符合各操作系统惯例的缓存根目录，并在此基础上按当前工作目录（CWD）和服务器名称进行子目录划分，确保：
- 不同项目的缓存数据互不污染。
- 路径名称在文件系统安全范围内（过滤非法字符、限制长度）。
- 缓存目录命名稳定，不因升级而改变（避免历史数据孤立）。

当前主要用于：错误日志、消息文件、MCP 服务器日志。

## 功能点目的

| 功能 | 目的 |
|------|------|
| `CACHE_PATHS.baseLogs()` | 获取当前项目的基础日志目录 |
| `CACHE_PATHS.errors()` | 获取当前项目的错误日志子目录 |
| `CACHE_PATHS.messages()` | 获取当前项目的消息缓存子目录 |
| `CACHE_PATHS.mcpLogs(serverName)` | 获取指定 MCP 服务器的日志目录 |

## 具体技术实现

### 跨平台根目录
```ts
import envPaths from 'env-paths'
const paths = envPaths('claude-cli')
```
- `env-paths` 根据操作系统返回标准路径：
  - macOS: `~/Library/Caches/claude-cli`
  - Linux: `~/.cache/claude-cli`
  - Windows: `%LOCALAPPDATA%\claude-cli\Cache`

### 路径安全化（`sanitizePath`）
```ts
function sanitizePath(name: string): string {
  const sanitized = name.replace(/[^a-zA-Z0-9]/g, '-')
  if (sanitized.length <= MAX_SANITIZED_LENGTH) {
    return sanitized
  }
  return `${sanitized.slice(0, MAX_SANITIZED_LENGTH)}-${Math.abs(djb2Hash(name)).toString(36)}`
}
```
- 将所有非字母数字字符替换为 `-`。
- 若长度超过 `200`，截断后追加 `djb2Hash(name)` 的 base36 值，保证唯一性。
- **刻意使用 `djb2Hash` 而非 `Bun.hash`**：注释说明缓存目录名必须跨升级保持稳定；`Bun.hash` 可能使用 wyhash，与本地稳定的 djb2 不同。

### 项目目录隔离
```ts
function getProjectDir(cwd: string): string {
  return sanitizePath(cwd)
}
```
- 以当前工作目录的完整路径作为项目标识，经 `sanitizePath` 后作为子目录名。

### MCP 服务器日志目录
```ts
mcpLogs: (serverName: string) =>
  join(paths.cache, getProjectDir(getFsImplementation().cwd()), `mcp-logs-${sanitizePath(serverName)}`)
```
- 对 `serverName` 再次做 `sanitizePath`，特别处理 Windows 下的冒号（驱动器号保留字符）。

## 关键代码路径与文件引用

```
src/utils/errorLogSink.ts
  └── CACHE_PATHS.errors()
      [错误日志文件落盘路径]

src/utils/log.ts
  └── CACHE_PATHS.errors(), CACHE_PATHS.messages()
      [通用日志与消息缓存路径]

src/utils/cleanup.ts
  └── CACHE_PATHS.errors(), CACHE_PATHS.messages(), CACHE_PATHS.mcpLogs(serverName)
      [清理过期缓存文件]
```

### 依赖模块
- `env-paths` — 跨平台标准路径生成
- `node:path` — `join`
- `src/utils/fsOperations.ts` — `getFsImplementation().cwd()`
- `src/utils/hash.ts`（或 `hash.js`）— `djb2Hash`

## 依赖与外部交互

| 外部实体 | 交互方式 | 说明 |
|---------|---------|------|
| 操作系统缓存目录 | `env-paths` | 按 OS 惯例生成根缓存路径 |
| 当前工作目录 | `getFsImplementation().cwd()` | 作为项目隔离键 |
| 文件系统 | 路径字符串返回 | 本模块不实际创建目录，仅返回路径 |

## 风险、边界与改进建议

### 风险
1. **CWD 变更导致路径漂移**：`baseLogs()` 等函数在每次调用时实时读取 `cwd()`；若进程在运行期间切换了工作目录，后续日志会写到另一个项目目录下，造成数据分散。
2. **`sanitizePath` 冲突**：不同 CWD 经替换后可能产生相同的 sanitized 名称（如 `/home/user/a+b` 与 `/home/user/a-b` 都变成 `home-user-a-b`），导致不同项目缓存意外合并。
3. **超长 CWD 截断后的哈希碰撞**：虽然 `djb2Hash` 降低了碰撞概率，但截断 + 哈希策略仍非绝对唯一。
4. **无目录创建逻辑**：模块仅返回路径字符串，调用方需自行 `mkdir`；若某调用方遗漏，可能导致写入失败。

### 边界
- 最大 sanitized 长度硬编码为 `200`，在大多数文件系统（Windows MAX_PATH 260）下安全，但极端嵌套路径仍可能接近上限。
- `serverName` 的冒号处理注释提到 Windows 兼容性，但实际 `sanitizePath` 对所有非字母数字统一替换为 `-`，已自然覆盖冒号。

### 改进建议
1. **冻结项目目录**：在 CLI 启动时计算一次 `projectDir` 并缓存，避免运行期间 CWD 变化导致缓存路径漂移。
2. **冲突检测与日志**：在 debug 模式下记录 `cwd -> sanitizedPath` 映射，便于排查不同项目意外合并的问题。
3. **目录自动创建**：提供 `ensureCacheDirs()` 辅助函数，统一创建 `errors`、`messages`、`mcpLogs` 等子目录，减少调用方重复实现。
4. **路径长度上限校验**：在 `sanitizePath` 后检查最终拼接路径是否超过操作系统限制（如 Windows 260），超长时改用哈希值替代完整 sanitized CWD。
