# protocolHandler.ts 研究文档

> 文件路径：`src/utils/deepLink/protocolHandler.ts`  
> 研究时间：2026-04-01  
> 执行器：kimi (k2p5)

---

## 场景与职责

`protocolHandler.ts` 是 Claude Code **Deep Link 协议处理的核心入口**。当操作系统通过已注册的 URI Scheme 唤起 Claude 时（例如用户点击浏览器中的 `claude-cli://open?q=...` 链接），该模块承担以下职责：

1. **URI 解析**：调用 `parseDeepLink` 将原始 URI 转换为结构化 action；
2. **工作目录解析**：根据 `cwd` / `repo` 参数决定新会话应在哪个目录启动；
3. **Git 新鲜度预计算**：在 trampoline（无 TTY 的协议处理进程）中提前读取 `FETCH_HEAD` 时间，避免主进程启动路径阻塞；
4. **终端启动**：调用 `terminalLauncher.ts` 在用户偏好的终端模拟器中启动新的 Claude 实例，并传递 `--deep-link-origin` 等内部标志。

该模块运行于 **headless 上下文**（无终端 attached），因此所有交互都必须通过启动新终端窗口完成。

---

## 功能点目的

| 功能 | 目的 |
|------|------|
| `handleDeepLinkUri(uri)` | CLI 入口 `claude --handle-uri <url>` 的处理函数，解析 URI 并启动终端会话。返回 exit code。 |
| `handleUrlSchemeLaunch()` | macOS 专用入口：检测是否由 LaunchServices 通过 `.app` bundle 启动，若是则通过 NAPI 模块读取 Apple Event 中的 URL 并处理。 |
| `resolveCwd(action)` | 确定启动目录的优先级：显式 `cwd` > `repo` MRU 查找 > 用户主目录 (`homedir`)。 |

---

## 具体技术实现

### 关键流程

#### 1. `handleDeepLinkUri(uri: string): Promise<number>`
```
parseDeepLink(uri)
  → resolveCwd(action)
  → readLastFetchTime(cwd)   (仅当 repo 解析成功时)
  → launchInTerminal(process.execPath, { query, cwd, repo, lastFetchMs })
  → 返回 0（成功）或 1（失败）
```
- **日志**：使用 `logForDebugging` 记录原始 URI 与解析后的 action（JSON 序列化）。
- **错误处理**：解析失败时向 `stderr` 输出可读错误并返回 `1`；终端启动失败同样返回 `1`。
- **二进制路径**：始终使用 `process.execPath`，确保 trampoline 与目标实例是同一二进制文件，避免 PATH 查找带来的不确定性。

#### 2. `handleUrlSchemeLaunch(): Promise<number | null>`
- **检测信号**：检查 `process.env.__CFBundleIdentifier` 是否等于 `MACOS_BUNDLE_ID`（`com.anthropic.claude-code-url-handler`）。
  - 这是 LaunchServices 覆盖的环境变量，仅在通过 URL handler bundle 启动时才会出现，比 `!TERM` 等负向启发式更精确。
- **读取 URL**：动态 `import('url-handler-napi')`，调用 `waitForUrlEvent(5000)` 同步阻塞最多 5 秒等待 Apple Event。
- 若未拿到 URL 或 NAPI 模块不可用，返回 `null`（表示并非 URL 启动）。

#### 3. `resolveCwd(action)`
| 优先级 | 条件 | 返回值 |
|--------|------|--------|
| 1 | `action.cwd` 存在 | `{ cwd: action.cwd }` |
| 2 | `action.repo` 存在 | 调用 `getKnownPathsForRepo` → `filterExistingPaths`；若命中则 `{ cwd: existing[0], resolvedRepo: action.repo }` |
| 3 | 默认 | `{ cwd: homedir() }` |

- **容错设计**：即使 repo 未被本地克隆，也不报错，而是优雅降级到用户主目录，保证 web 链接始终能打开 Claude。

---

## 关键代码路径与文件引用

### 本文件导出
- `handleDeepLinkUri(uri)` → `src/main.tsx` ~L648-659（`--handle-uri` 分支）
- `handleUrlSchemeLaunch()` → `src/main.tsx` ~L666-676（macOS bundle 启动分支）

### 被调用方详情（本文件导入）

| 导入来源 | 符号 | 用途 |
|----------|------|------|
| `os` | `homedir` | 默认 fallback 目录 |
| `../debug.js` | `logForDebugging` | 调试日志 |
| `../githubRepoPathMapping.js` | `filterExistingPaths`, `getKnownPathsForRepo` | repo → 本地路径 MRU 解析 |
| `../slowOperations.js` | `jsonStringify` | 日志序列化 |
| `./banner.js` | `readLastFetchTime` | 预计算 FETCH_HEAD 时间 |
| `./parseDeepLink.js` | `parseDeepLink` | URI 解析 |
| `./registerProtocol.js` | `MACOS_BUNDLE_ID` | macOS bundle 识别 |
| `./terminalLauncher.ts` | `launchInTerminal` | 终端启动 |

### 调用方详情

**`src/main.tsx`**
- 在 `main()` 函数早期（`feature('LODESTOM')` 门控下）处理两种启动模式：
  1. `process.argv` 包含 `--handle-uri <url>` 时，先 `enableConfigs()`，再 `handleDeepLinkUri(url)`，最后 `process.exit(exitCode)`。
  2. macOS 上 `__CFBundleIdentifier` 匹配时，同样先 `enableConfigs()`，再 `handleUrlSchemeLaunch()`，然后 `process.exit(urlSchemeResult ?? 1)`。
- 这种提前退出的设计确保 deep link 处理路径**不加载完整的交互式初始化流程**（如 REPL、信任对话框等），保持 trampoline 轻量。

---

## 依赖与外部交互

### 外部原生模块
- **`url-handler-napi`**（动态导入）
  - 仅 macOS 使用。
  - 提供 `waitForUrlEvent(ms)` 从 Apple Event 中读取 URL。
  - 模块缺失时 catch 并返回 `null`，保证非标准构建也能启动。

### 配置/状态依赖
- **`githubRepoPathMapping.js`**：依赖全局配置中的 `githubRepoPaths` 字段，该字段由 `interactiveHelpers.tsx` 在每次交互启动时异步更新（fire-and-forget）。
- **`enableConfigs()`**：在 `main.tsx` 中于调用本模块前执行，确保配置系统可用（否则 `getKnownPathsForRepo` 可能读到空配置）。

---

## 风险、边界与改进建议

### 风险
1. **`url-handler-napi` 动态导入失败静默**：若该原生模块在 macOS 生产包中缺失，`handleUrlSchemeLaunch` 会返回 `null`，导致通过 bundle 启动的 deep link 被静默忽略，用户体验为“点击链接无反应”。
2. **`filterExistingPaths` 的并发 stat**：对 MRU 列表中的所有路径并发执行 `pathExists`，若列表较长或包含网络文件系统路径，可能造成事件循环阻塞或超时。
3. **`process.execPath` 假设**：假设 OS 注册的 handler 指向的正是当前运行二进制；在符号链接、wrapper 脚本或某些包管理器（如 Homebrew 的 shim）场景下，该假设可能不成立，导致启动的实例与预期不同。

### 边界
- **无 TTY 环境**：本模块自身不在终端中运行，因此所有用户可见输出（错误信息）都通过 `console.error` 写入 stderr；成功时则完全静默，由新启动的终端窗口接管交互。
- **repo 未克隆降级到 home**：这是有意的设计，但用户可能困惑为何点击某个 repo 链接后打开的是主目录而非该仓库。

### 改进建议
1. **增强 `url-handler-napi` 缺失的可见性**：可在 catch 分支增加 `logEvent` 或 `console.error`，提示开发者/用户 deep link 原生依赖未加载。
2. **MRU 路径超时/截断**：对 `filterExistingPaths` 增加并行度限制或单个路径超时，避免网络挂载路径拖慢启动。
3. **用户可感知降级提示**：当 `repo` 未解析到本地路径时，可考虑在新启动的 Claude 实例中显示一条信息性横幅（类似 `banner.ts` 的 stale 提示），告知用户“未找到本地克隆，已打开主目录”。
4. **测试覆盖**：当前无单元测试，建议补充：
   - `resolveCwd` 的优先级组合测试；
   - `handleDeepLinkUri` 的 mock 测试（mock `launchInTerminal` 与 `parseDeepLink`）；
   - macOS `handleUrlSchemeLaunch` 在不同 `__CFBundleIdentifier` 值下的分支测试。
