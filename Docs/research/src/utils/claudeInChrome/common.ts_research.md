# common.ts 深度研究文档

## 场景与职责

`common.ts` 是 **Claude in Chrome** 功能模块的**共享基础设施层**，为 `setup.ts`、`mcpServer.ts`、`chromeNativeHost.ts`、`toolRendering.tsx` 等上层模块提供：

1. **浏览器配置与检测**：定义 7 款 Chromium 内核浏览器的目录结构、二进制名称、注册表键值。
2. **路径解析**：计算浏览器数据目录、Native Messaging Hosts 目录、Unix socket / Windows named pipe 路径。
3. **浏览器启动**：`openInChrome(url)` 跨平台打开 URL。
4. **Tab ID 追踪**：防止重复渲染/操作同一浏览器标签页。
5. **MCP 服务器名称常量**：`claude-in-chrome` 的规范化标识。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `CHROMIUM_BROWSERS` | 静态配置表，描述 Chrome、Brave、Arc、Chromium、Edge、Vivaldi、Opera 在各平台下的路径与注册表信息。 |
| `BROWSER_DETECTION_ORDER` | 检测优先级数组，按市场占有率/常见程度排序。 |
| `getAllBrowserDataPaths()` | 返回所有支持的浏览器数据目录（用于检测扩展是否安装）。 |
| `getAllNativeMessagingHostsDirs()` | 返回所有浏览器的 Native Messaging manifest 安装目录。 |
| `getAllWindowsRegistryKeys()` | 返回 Windows 注册表键列表（供 `setup.ts` 写入 manifest 路径）。 |
| `detectAvailableBrowser()` | 按优先级检测本地实际安装了哪款浏览器，返回首个匹配项。 |
| `openInChrome(url)` | 跨平台用检测到的浏览器打开指定 URL。 |
| `getSecureSocketPath()` / `getAllSocketPaths()` | 生成/收集 Unix socket 或 Windows named pipe 路径。 |
| `trackClaudeInChromeTabId()` / `isTrackedClaudeInChromeTabId()` | 追踪已渲染过的 tabId，避免重复处理。 |
| `isClaudeInChromeMCPServer()` | 通过 MCP 名称规范化判断是否为 Claude in Chrome 服务器。 |

## 具体技术实现

### 1. 浏览器配置结构 (`BrowserConfig`)

```typescript
type BrowserConfig = {
  name: string
  macos: { appName: string; dataPath: string[]; nativeMessagingPath: string[] }
  linux: { binaries: string[]; dataPath: string[]; nativeMessagingPath: string[] }
  windows: { dataPath: string[]; registryKey: string; useRoaming?: boolean }
}
```

- `Arc` 在 Linux 无支持（`binaries: []`）。
- `Opera` 在 Windows 使用 `Roaming` AppData（`useRoaming: true`），其余浏览器多用 `Local`。

### 2. 浏览器检测逻辑 (`detectAvailableBrowser()`)

按 `BROWSER_DETECTION_ORDER` 顺序检测：

- **macOS**：检查 `/Applications/<appName>.app` 是否存在且为目录。
- **Linux / WSL**：通过 `which(binary)` 检查可执行文件是否在 PATH 中。
- **Windows**：检查 `AppData/Local`（或 `Roaming`）下的浏览器数据目录是否存在。

检测成功时通过 `logForDebugging` 输出日志，返回首个匹配的 `ChromiumBrowser` 标识。

### 3. URL 打开逻辑 (`openInChrome()`)

| 平台 | 实现 |
|------|------|
| macOS | `open -a <appName> <url>` |
| Windows | `rundll32 url,OpenURL <url>`（避免 `cmd.exe` 的元字符注入问题） |
| Linux/WSL | 遍历该浏览器的 `binaries`，逐个尝试 `execFileNoThrow(binary, [url])` |

### 4. Socket 路径管理

- **目录**：`/tmp/claude-mcp-browser-bridge-<username>/`
- **当前进程 socket**：`getSecureSocketPath()` 返回 `join(getSocketDir(), `${process.pid}.sock`)`（Unix）或固定 named pipe（Windows）。
- **兼容扫描**：`getAllSocketPaths()` 会扫描目录下所有 `*.sock`，并追加两个 legacy fallback 路径（`tmpdir()` 和 `/tmp` 下的旧版路径），供外部 MCP 客户端尝试连接。

### 5. Tab ID 追踪

```typescript
const MAX_TRACKED_TABS = 200
const trackedTabIds = new Set<number>()
```

- 当 set 达到上限且新 tabId 不在其中时，**清空整个 set** 然后添加新 ID。
- 这是一个简单的 LRU 替代策略，避免无界增长。

### 6. MCP 服务器识别

```typescript
export function isClaudeInChromeMCPServer(name: string): boolean {
  return normalizeNameForMCP(name) === CLAUDE_IN_CHROME_MCP_SERVER_NAME
}
```

- 复用 `src/services/mcp/normalization.js` 的 `normalizeNameForMCP`，确保与 MCP 配置中的命名规则一致。

## 关键代码路径与文件引用

```
src/utils/claudeInChrome/common.ts
  ├── CHROMIUM_BROWSERS 常量表
  ├── detectAvailableBrowser()
  │   └── which()               ← src/utils/which.js
  │   └── logForDebugging()     ← src/utils/debug.js
  ├── openInChrome(url)
  │   └── execFileNoThrow()     ← src/utils/execFileNoThrow.js
  ├── getSecureSocketPath()
  ├── getAllSocketPaths()
  ├── isClaudeInChromeMCPServer()
  │   └── normalizeNameForMCP() ← src/services/mcp/normalization.js
  └── trackClaudeInChromeTabId() / isTrackedClaudeInChromeTabId()
```

### 调用方分布

| 调用方 | 使用的导出项 |
|--------|-------------|
| `chromeNativeHost.ts` | `getSecureSocketPath`, `getSocketDir` |
| `setup.ts` | `CLAUDE_IN_CHROME_MCP_SERVER_NAME`, `getAllBrowserDataPaths`, `getAllNativeMessagingHostsDirs`, `getAllWindowsRegistryKeys`, `openInChrome` |
| `mcpServer.ts` | `getAllSocketPaths`, `getSecureSocketPath` |
| `toolRendering.tsx` | `trackClaudeInChromeTabId`, `isTrackedClaudeInChromeTabId` |
| `src/services/mcp/client.ts` | `isClaudeInChromeMCPServer` |
| `src/hooks/usePromptsFromClaudeInChrome.tsx` | `CLAUDE_IN_CHROME_MCP_SERVER_NAME`, `isTrackedClaudeInChromeTabId` |
| `src/commands/chrome/chrome.tsx` | `CLAUDE_IN_CHROME_MCP_SERVER_NAME`, `openInChrome` |
| `src/utils/attachments.ts` | `isTrackedClaudeInChromeTabId`（推测，用于附件去重） |

## 依赖与外部交互

| 依赖 | 用途 |
|------|------|
| `fs` / `fs/promises` | 同步/异步文件系统操作（`readdirSync`, `stat`） |
| `os` | `homedir()`, `platform()`, `tmpdir()`, `userInfo()` |
| `path` | `join()` |
| `src/services/mcp/normalization.js` | MCP 服务器名称规范化 |
| `src/utils/debug.js` | 调试日志 |
| `src/utils/errors.js` | `isFsInaccessible()` 区分权限错误与不存在 |
| `src/utils/execFileNoThrow.js` | 安全执行外部命令 |
| `src/utils/platform.js` | `getPlatform()` 返回 `'macos' \| 'linux' \| 'windows' \| 'wsl'` |
| `src/utils/which.js` | 跨平台 `which` 实现 |

## 风险、边界与改进建议

### 风险

1. **浏览器配置与 `setupPortable.ts` 的重复**：`CHROMIUM_BROWSERS` 和 `BROWSER_DETECTION_ORDER` 在 `setupPortable.ts` 中有几乎镜像的副本，若新增浏览器或路径变更，需同时修改两处，易遗漏。
2. **WSL 检测的局限性**：WSL 下 `getPlatform()` 返回 `'wsl'`，但 `openInChrome()` 将其与 `linux` 同等处理，直接调用 Linux binary。若用户希望在 WSL 中调用 Windows 版 Chrome，当前逻辑不支持。
3. **Arc on Windows 的注释与实现不一致**：代码中 Arc Windows 有 `dataPath`，但注释未特别说明；而 `setupPortable.ts` 也保留了同样配置。
4. **`getAllSocketPaths()` 的同步读取**：使用了 `readdirSync`，注释说明是因为外部包 `@ant/claude-for-chrome-mcp` 的 `ClaudeForChromeContext.getSocketPaths` 要求同步回调。

### 边界

- `MAX_TRACKED_TABS = 200` 是硬编码值，对极端多标签场景可能不够；但清空策略简单有效。
- `detectAvailableBrowser()` 只返回**第一个**检测到的浏览器，若用户同时安装 Chrome 和 Edge，永远优先 Chrome。
- Windows 下 `getAllNativeMessagingHostsDirs()` 返回空数组（因为 Windows 使用注册表而非文件路径），调用方需单独处理。

### 改进建议

1. **统一浏览器配置源**：将 `CHROMIUM_BROWSERS` 提取到单独的 JSON 或共享模块，供 `common.ts` 与 `setupPortable.ts` 共同引用，消除重复。
2. **WSL → Windows Chrome 透传**：增加检测 WSL 中 `chrome.exe` 或 `powershell.exe start chrome` 的能力，提升 WSL 用户体验。
3. **用户偏好浏览器**：允许通过环境变量（如 `CLAUDE_CHROME_BROWSER=edge`）覆盖自动检测顺序。
4. **异步 socket 扫描**：若外部包允许，逐步将 `getAllSocketPaths()` 中的 `readdirSync` 改为异步，避免阻塞事件循环。
5. **增加测试**：对 `detectAvailableBrowser()` 的 mock 测试、`openInChrome()` 的命令生成测试、`getAllSocketPaths()` 的路径拼接测试均有价值。
