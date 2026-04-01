# src/utils/claudeDesktop.ts 深入研究

## 场景与职责

`claudeDesktop.ts` 实现 Claude Code CLI 与 **Claude Desktop App** 的 MCP（Model Context Protocol）服务器配置互通。它允许 CLI 读取 Claude Desktop 已配置的 MCP stdio 服务器列表，从而支持：
- 在 CLI 中复用 Desktop 已添加的 MCP 工具。
- 降低用户重复配置 MCP 服务器的成本。

当前仅支持 **macOS** 与 **WSL**（Windows Subsystem for Linux）环境，因为 Claude Desktop 本身仅发布 macOS 与 Windows 版本。

## 功能点目的

| 功能 | 目的 |
|------|------|
| `getClaudeDesktopConfigPath()` | 定位 `claude_desktop_config.json` 的绝对路径 |
| `readClaudeDesktopMcpServers()` | 读取并解析 Desktop 配置中的 `mcpServers` 字段，返回经过 Zod 校验的服务器配置映射 |

## 具体技术实现

### 配置文件路径
#### macOS
```ts
join(homedir(), 'Library', 'Application Support', 'Claude', 'claude_desktop_config.json')
```

#### WSL
WSL 路径解析较为复杂，因为 Desktop 运行在 Windows 宿主机上，而 CLI 运行在 Linux 子系统中：
1. **优先尝试 `USERPROFILE`**：
   ```ts
   const windowsHome = process.env.USERPROFILE?.replace(/\\/g, '/')
   const wslPath = windowsHome.replace(/^[A-Z]:/, '')
   const configPath = `/mnt/c${wslPath}/AppData/Roaming/Claude/claude_desktop_config.json`
   ```
   - 将 Windows 反斜杠转为正斜杠，去掉盘符前缀，拼接为 WSL 的 `/mnt/c/...` 路径。
   - 使用 `stat` 验证文件是否存在；存在则直接返回。

2. **回退遍历 `/mnt/c/Users`**：
   - 若 `USERPROFILE` 未设置或指向路径不存在，则遍历 `/mnt/c/Users` 下的所有用户目录。
   - 跳过 `Public`、`Default`、`Default User`、`All Users` 等系统目录。
   - 对每个候选路径执行 `stat`，首个存在的即返回。

3. **最终失败**：若仍找不到，抛出 `Error`。

### 配置解析
```ts
export async function readClaudeDesktopMcpServers(): Promise<Record<string, McpServerConfig>>
```
1. 调用 `getClaudeDesktopConfigPath()` 获取路径。
2. `readFile` 读取 JSON 内容；若文件不存在（`ENOENT`），返回 `{}`。
3. 使用 `safeParseJSON` 解析；解析失败或结果非对象，返回 `{}`。
4. 提取 `mcpServers` 字段；对每个 server config 使用 `McpStdioServerConfigSchema().safeParse()` 进行 Zod 校验。
5. 仅返回校验通过的配置项。

### 平台校验
- 两个函数均先检查 `getPlatform()` 是否在 `SUPPORTED_PLATFORMS` 中（macOS / WSL）。
- 非支持平台直接抛出 `Error`。

## 关键代码路径与文件引用

```
src/cli/handlers/mcp.tsx
  └── readClaudeDesktopMcpServers()
      [MCP 命令处理：展示从 Desktop 导入的服务器列表]

src/components/MCPServerDesktopImportDialog.tsx
  └── readClaudeDesktopMcpServers()
      [UI 对话框：允许用户选择要导入的 Desktop MCP 服务器]
```

### 依赖模块
- `node:fs/promises` — `readdir`, `readFile`, `stat`
- `node:os` — `homedir`
- `node:path` — `join`
- `src/services/mcp/types.ts` — `McpServerConfig`, `McpStdioServerConfigSchema`
- `src/utils/errors.ts` — `getErrnoCode`
- `src/utils/json.ts` — `safeParseJSON`
- `src/utils/log.ts` — `logError`
- `src/utils/platform.ts` — `getPlatform`, `SUPPORTED_PLATFORMS`

## 依赖与外部交互

| 外部实体 | 交互方式 | 说明 |
|---------|---------|------|
| Claude Desktop App | 读取其配置文件 | `claude_desktop_config.json` |
| Windows 文件系统（WSL） | `/mnt/c/...` | 通过 WSL 的 drvfs 挂载读取 Windows 用户目录 |
| Zod schema | `McpStdioServerConfigSchema` | 校验每个 server 配置的结构正确性 |
| 用户环境变量 | `process.env.USERPROFILE` | WSL 下快速定位 Windows 用户主目录 |

## 风险、边界与改进建议

### 风险
1. **WSL 路径硬编码**：`/mnt/c` 是 WSL 默认挂载点，但用户可能自定义了 `wsl.conf` 的 `automount.root`（如 `/windrives/c`），此时路径解析会完全失败。
2. **`USERPROFILE` 回退遍历的权限问题**：遍历 `/mnt/c/Users` 需要 WSL 对 Windows 目录的读取权限；在某些企业环境中，该目录可能被 ACL 限制，导致 `readdir` 抛错并进入最终失败分支。
3. **无写入能力**：当前模块只读 Desktop 配置，不支持将 CLI 添加的 MCP 服务器同步回写 Desktop，互通是单向的。
4. **平台检测粒度不足**：`getPlatform()` 返回 `wsl` 时才支持；若用户在纯 Linux（非 WSL）上通过 Wine 运行 Desktop，或未来 Desktop 发布 Linux 版，当前逻辑无法适配。

### 边界
- 仅解析 `mcpServers` 中的 **stdio** 类型服务器（通过 `McpStdioServerConfigSchema` 校验）。
- SSE / WebSocket 等远程 MCP 服务器当前不在导入范围内（Desktop 配置中若有此类服务器，会被 Zod 校验过滤掉）。
- 配置文件不存在时返回 `{}` 而非抛错，这是有意设计的容错行为。
- 解析失败的服务器被静默跳过，不会中断其他服务器的导入。

### 改进建议
1. **支持自定义 WSL 挂载根**：读取 `/etc/wsl.conf` 或检查 `mount` 输出，动态确定 Windows 盘符挂载前缀，而非硬编码 `/mnt/c`。
2. **增加 Windows 原生支持**：若未来 CLI 直接运行在 Windows PowerShell/CMD 中（非 WSL），应直接读取 `%APPDATA%\Claude\claude_desktop_config.json`。
3. **双向同步**：评估是否提供将 CLI 的 MCP 配置回写到 Desktop 的能力，实现真正的配置互通（需处理权限、格式差异、冲突解决）。
4. **导入前差异提示**：在 UI 对话框中展示 "Desktop 有但 CLI 没有" 的服务器列表，并允许用户选择性导入，避免一次性全量导入造成的配置膨胀。
5. **增强错误提示**：当 WSL 路径解析失败时，给出更具体的排查指引（如检查 `USERPROFILE`、确认 Desktop 已安装、检查 WSL 挂载权限）。
