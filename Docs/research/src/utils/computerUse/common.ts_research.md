# common.ts 研究文档

## 场景与职责

本文件是 Computer Use MCP 功能的**共享常量与工具函数模块**，提供跨组件使用的通用定义和辅助函数。核心职责包括：

1. **MCP 服务器名称定义**：统一的服务器标识符
2. **终端 Bundle ID 检测**：识别当前运行的终端模拟器
3. **CLI 能力声明**：静态能力配置（平台、截图过滤）
4. **服务器名称工具函数**：MCP 服务器名称规范化比较

## 功能点目的

### 1. MCP 服务器名称 (`COMPUTER_USE_MCP_SERVER_NAME`)
- 统一标识符 `'computer-use'`
- 用于工具名称构建：`mcp__computer-use__{toolName}`
- 与 `normalizeNameForMCP` 配合进行名称比较

### 2. CLI Host Bundle ID (`CLI_HOST_BUNDLE_ID`)
- 哨兵值：`'com.anthropic.claude-code.cli-no-window'`
- 用于 frontmost gate：Claude Code 是终端应用，没有窗口
- 永远不会匹配真实的 `NSWorkspace.frontmostApplication`
- 使包的 "host is frontmost" 分支成为死代码

### 3. 终端 Bundle ID 回退表 (`TERMINAL_BUNDLE_ID_FALLBACK`)
- 当 `__CFBundleIdentifier` 环境变量未设置时的备用检测
- 支持 macOS 终端：iTerm、Apple Terminal、Ghostty、Kitty、Warp、VSCode
- Linux 条目故意省略（`createCliExecutor` 是 darwin-only）

### 4. CLI CU 能力声明 (`CLI_CU_CAPABILITIES`)
- `screenshotFiltering: 'native'` - 使用原生截图过滤
- `platform: 'darwin'` - macOS 平台
- 注意：`hostBundleId` 不在此处，由 `executor.ts` 动态添加

## 具体技术实现

### 核心常量与类型

```typescript
export const COMPUTER_USE_MCP_SERVER_NAME = 'computer-use'

export const CLI_HOST_BUNDLE_ID = 'com.anthropic.claude-code.cli-no-window'

const TERMINAL_BUNDLE_ID_FALLBACK: Readonly<Record<string, string>> = {
  'iTerm.app': 'com.googlecode.iterm2',
  Apple_Terminal: 'com.apple.Terminal',
  ghostty: 'com.mitchellh.ghostty',
  kitty: 'net.kovidgoyal.kitty',
  WarpTerminal: 'dev.warp.Warp-Stable',
  vscode: 'com.microsoft.VSCode',
}

export const CLI_CU_CAPABILITIES = {
  screenshotFiltering: 'native' as const,
  platform: 'darwin' as const,
}
```

### 关键函数

#### `getTerminalBundleId()` - 终端检测

```typescript
export function getTerminalBundleId(): string | null {
  const cfBundleId = process.env.__CFBundleIdentifier
  if (cfBundleId) return cfBundleId
  return TERMINAL_BUNDLE_ID_FALLBACK[env.terminal ?? ''] ?? null
}
```

**实现细节**：
- `__CFBundleIdentifier` 由 LaunchServices 在 .app bundle 启动进程时设置
- 被子进程继承，是精确的 bundleId，无需查找
- 在 tmux/screen 下反映启动服务器的终端（可能与当前客户端不同）
- 返回 null 时表示无法检测（ssh、清除的环境变量、未知终端）

#### `isComputerUseMCPServer()` - 服务器名称检查

```typescript
export function isComputerUseMCPServer(name: string): boolean {
  return normalizeNameForMCP(name) === COMPUTER_USE_MCP_SERVER_NAME
}
```

**使用场景**：
- `client.ts` 中用于识别 Computer Use MCP 服务器
- 决定是否启动进程内服务器（而非子进程）

## 关键代码路径与文件引用

### 本文件导出
- `COMPUTER_USE_MCP_SERVER_NAME` - 服务器名称常量
- `CLI_HOST_BUNDLE_ID` - CLI Host 哨兵 bundle ID
- `CLI_CU_CAPABILITIES` - 静态能力声明
- `getTerminalBundleId()` - 终端检测函数
- `isComputerUseMCPServer()` - 服务器名称检查

### 调用方
- `src/utils/computerUse/executor.ts:276` - `getTerminalBundleId()` 获取终端 bundle ID
- `src/utils/computerUse/hostAdapter.ts:41` - `COMPUTER_USE_MCP_SERVER_NAME` 设置 serverName
- `src/utils/computerUse/setup.ts:48` - `COMPUTER_USE_MCP_SERVER_NAME` 构建 MCP 配置
- `src/services/mcp/client.ts:928` - `isComputerUseMCPServer()` 识别 CU 服务器

### 依赖文件
- `src/services/mcp/normalization.ts` - `normalizeNameForMCP`
- `src/utils/env.ts` - `env.terminal`

## 依赖与外部交互

### 外部包依赖
- 无直接外部包依赖

### 环境变量交互
- `process.env.__CFBundleIdentifier` - LaunchServices 设置的 bundle ID
- `env.terminal` - 终端类型检测（来自 env.ts）

### 与系统交互
- 依赖 macOS LaunchServices 环境变量
- 在 tmux/screen 环境下有已知行为差异

## 风险、边界与改进建议

### 已知风险

1. **tmux/screen 环境差异**：
   - `__CFBundleIdentifier` 反映启动服务器的终端，而非当前连接的客户端
   - 缓解：无害，只是豁免一个终端窗口，截图排除也适用

2. **SSH 会话**：
   - 无法检测终端类型，返回 null
   - 调用方必须处理 null 情况

3. **新终端支持**：
   - 新终端模拟器需要手动添加到回退表
   - 如果终端设置 `__CFBundleIdentifier` 则自动支持

### 边界情况

1. **未知终端**：
   - 返回 null，调用方使用 `CLI_HOST_BUNDLE_ID` 作为回退

2. **多终端环境**：
   - 使用启动 Claude Code 的终端作为 surrogate host
   - 该终端在截图中被排除

### 改进建议

1. **终端检测增强**：
   - 添加更多终端模拟器支持（Alacritty、WezTerm 等）
   - 考虑通过父进程链遍历检测终端

2. **配置覆盖**：
   - 添加环境变量允许用户手动指定终端 bundle ID
   - 用于自定义终端或容器环境

3. **诊断信息**：
   - 在调试日志中记录终端检测过程
   - 帮助排查截图包含终端的问题

4. **跨平台考虑**：
   - 虽然当前是 darwin-only，但可为未来 Linux 支持预留接口
   - 添加平台检测和相应的回退逻辑
