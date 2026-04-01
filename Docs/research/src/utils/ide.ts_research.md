# ide.ts 研究文档

## 场景与职责

`ide.ts` 是 Claude Code CLI 的 **IDE 集成核心模块**，负责检测、连接和管理与各种 IDE（VS Code、Cursor、Windsurf、JetBrains 系列等）的集成。该模块实现了完整的 IDE 生命周期管理，包括自动发现、扩展安装、连接管理和路径转换等功能。

### 核心使用场景

1. **IDE 自动检测**：扫描系统中运行的 IDE 进程，识别支持的 IDE 类型
2. **IDE 扩展管理**：自动安装/更新 Claude Code 扩展（VS Code）或检测插件（JetBrains）
3. **MCP 服务器集成**：通过 SSE/WebSocket 与 IDE 扩展建立 MCP 连接
4. **工作区匹配**：验证当前工作目录是否在 IDE 打开的工作区内
5. **路径转换**：处理 WSL 与 Windows 之间的路径转换

### 支持的 IDE 类型

| 类别 | IDE |
|------|-----|
| VS Code 系列 | VS Code、Cursor、Windsurf |
| JetBrains 系列 | IntelliJ IDEA、PyCharm、WebStorm、PhpStorm、RubyMine、CLion、GoLand、Rider、DataGrip、AppCode、DataSpell、Aqua、Gateway、Fleet、Android Studio |

---

## 功能点目的

### 1. IDE 类型检测与分类

**分类逻辑**：
```typescript
type IdeType = 'cursor' | 'windsurf' | 'vscode' | 'pycharm' | 'intellij' | ...
type IdeKind = 'vscode' | 'jetbrains'
```

**平台特定的进程关键词**：
- macOS：检查 `ps aux` 输出中的进程名（如 "Visual Studio Code", "IntelliJ IDEA"）
- Windows：检查 `tasklist` 输出中的可执行文件名（如 `code.exe`, `idea64.exe`）
- Linux：检查 `ps aux` 输出中的进程名（如 `code`, `idea`）

### 2. IDE 锁文件管理

**锁文件位置**：`~/.claude/ide/{port}.lock`

**锁文件内容**（JSON 格式）：
```typescript
type LockfileJsonContent = {
  workspaceFolders?: string[]  // IDE 打开的工作区文件夹
  pid?: number                 // IDE 进程 ID
  ideName?: string             // IDE 名称
  transport?: 'ws' | 'sse'     // 传输协议
  runningInWindows?: boolean   // 是否在 Windows 中运行（WSL 场景）
  authToken?: string           // 认证令牌
}
```

**锁文件清理**：定期清理过期的锁文件（进程不存在或端口无响应）

### 3. IDE 发现流程 (`findAvailableIDE`)

**轮询机制**：
- 最多轮询 30 秒
- 每秒检查一次
- 滚动 drain 期间暂停（避免与滚动帧竞争事件循环）
- 当且仅当检测到**恰好一个** IDE 时返回（避免歧义）

### 4. 扩展安装管理

**VS Code 扩展**：
- 扩展 ID：`anthropic.claude-code`（普通用户）或 `anthropic.claude-code-internal`（内部员工）
- 自动检测已安装版本
- 版本比较：如果已安装版本低于 Claude Code 版本，强制更新
- 安装延迟：`sleep(500)` 避免 `code` 命令连续调用崩溃

**JetBrains 插件**：
- 插件目录扫描检测
- 不支持自动安装（需手动从 Marketplace 安装）
- 显示安装提示

### 5. WSL 路径转换

**场景**：WSL 中的 Claude Code 与 Windows 中的 IDE 通信

**转换逻辑**：
- Windows 路径 → WSL 路径：使用 `wslpath -u` 或手动转换（`C:\` → `/mnt/c/`）
- WSL 路径 → Windows 路径：使用 `wslpath -w`
- 分发版匹配：检查 `\\wsl$\{distro}` 路径是否匹配当前分发版

---

## 具体技术实现

### IDE 检测核心算法

```typescript
export async function detectIDEs(includeInvalid: boolean): Promise<DetectedIDEInfo[]> {
  const detectedIDEs: DetectedIDEInfo[] = []
  
  // 1. 获取排序后的锁文件列表
  const lockfiles = await getSortedIdeLockfiles()
  const lockfileInfos = await Promise.all(lockfiles.map(readIdeLockfile))
  
  // 2. 延迟加载祖先 PID 集合（性能优化）
  const getAncestors = makeAncestorPidLookup()
  const needsAncestryCheck = getPlatform() !== 'wsl' && isSupportedTerminal()
  
  // 3. 遍历锁文件进行验证
  for (const lockfileInfo of lockfileInfos) {
    if (!lockfileInfo) continue
    
    // 3.1 工作区匹配检查
    let isValid = checkWorkspaceMatch(lockfileInfo, cwd)
    
    // 3.2 PID 祖先检查（避免多 IDE 窗口歧义）
    if (needsAncestryCheck && !isValidByPort) {
      if (!lockfileInfo.pid || !isProcessRunning(lockfileInfo.pid)) continue
      if (process.ppid !== lockfileInfo.pid) {
        const ancestors = await getAncestors()
        if (!ancestors.has(lockfileInfo.pid)) continue
      }
    }
    
    // 3.3 构建 IDE 信息
    const host = await detectHostIP(lockfileInfo.runningInWindows, lockfileInfo.port)
    const url = lockfileInfo.useWebSocket 
      ? `ws://${host}:${lockfileInfo.port}`
      : `http://${host}:${lockfileInfo.port}/sse`
    
    detectedIDEs.push({ url, name, workspaceFolders, port, isValid, ... })
  }
  
  return detectedIDEs
}
```

### 父进程检测（macOS 专用）

```typescript
function getVSCodeIDECommandByParentProcess(): string | null {
  if (getPlatform() !== 'macos') return null
  
  let pid = process.ppid
  for (let i = 0; i < 10; i++) {  // 最多向上追溯 10 层
    if (!pid || pid === 0 || pid === 1) break
    
    const command = execSyncWithDefaults_DEPRECATED(
      `ps -o command= -p ${pid}`,
    )?.trim()
    
    // 检查已知的 VS Code 变体
    const appNames = {
      'Visual Studio Code.app': 'code',
      'Cursor.app': 'cursor',
      'Windsurf.app': 'windsurf',
      // ...
    }
    
    for (const [appName, executableName] of Object.entries(appNames)) {
      const appIndex = command.indexOf(appName + '/Contents/MacOS/Electron')
      if (appIndex !== -1) {
        // 构建 CLI 命令路径
        return command.substring(0, appIndex + appName.length) +
               '/Contents/Resources/app/bin/' + executableName
      }
    }
    
    // 获取父 PID 继续追溯
    pid = parseInt(execSyncWithDefaults_DEPRECATED(`ps -o ppid= -p ${pid}`)?.trim())
  }
  
  return null
}
```

### Windows 命令包装器（Windows 路径问题修复）

```typescript
// VS Code 1.110.0 开始将安装根目录添加到 PATH 前面
// 导致 'code' 解析为 Code.exe（GUI）而非 code.cmd（CLI）
function getVSCodeIDECommand(ideType: IdeType): Promise<string | null> {
  // ... 父进程检测逻辑 ...
  
  // Windows 上显式请求 .cmd 包装器
  const ext = getPlatform() === 'windows' ? '.cmd' : ''
  switch (ideType) {
    case 'vscode': return 'code' + ext
    case 'cursor': return 'cursor' + ext
    case 'windsurf': return 'windsurf' + ext
  }
}
```

### WSL 主机 IP 检测

```typescript
const detectHostIP = memoize(async (isIdeRunningInWindows: boolean, port: number) => {
  if (process.env.CLAUDE_CODE_IDE_HOST_OVERRIDE) {
    return process.env.CLAUDE_CODE_IDE_HOST_OVERRIDE
  }
  
  if (getPlatform() !== 'wsl' || !isIdeRunningInWindows) {
    return '127.0.0.1'
  }
  
  // WSL2 VM 中运行，IDE 在 Windows 中
  // 使用默认网关 IP 连接 Windows 主机
  try {
    const routeResult = await execa('ip route show | grep -i default', { shell: true })
    const gatewayMatch = routeResult.stdout.match(/default via (\d+\.\d+\.\d+\.\d+)/)
    if (gatewayMatch) {
      const gatewayIP = gatewayMatch[1]
      if (await checkIdeConnection(gatewayIP, port)) {
        return gatewayIP
      }
    }
  } catch { /* 忽略错误 */ }
  
  return '127.0.0.1'
}, (isIdeRunningInWindows, port) => `${isIdeRunningInWindows}:${port}`)
```

---

## 关键代码路径与文件引用

### 导出位置
- **文件**：`src/utils/ide.ts`（1494 行，约 46KB）
- **主要导出**：
  - `calculateHorizontalScrollWindow` - 水平滚动计算
  - `detectIDEs()` / `findAvailableIDE()` - IDE 检测
  - `initializeIdeIntegration()` - 初始化集成
  - `isVSCodeIde()` / `isJetBrainsIde()` - IDE 类型判断
  - `isIDEExtensionInstalled()` / `maybeInstallIDEExtension()` - 扩展管理
  - `toIDEDisplayName()` - 显示名称转换
  - `getConnectedIdeClient()` / `closeOpenDiffs()` - MCP 交互

### 调用方分布

| 文件路径 | 使用场景 |
|---------|---------|
| `src/hooks/useIDEIntegration.tsx` | React Hook 封装 IDE 初始化 |
| `src/hooks/useIdeSelection.ts` | IDE 选择逻辑 |
| `src/hooks/useDiffInIDE.ts` | IDE 中显示 diff |
| `src/hooks/useIdeLogging.ts` | IDE 日志记录 |
| `src/hooks/notifs/useIDEStatusIndicator.tsx` | IDE 状态指示器 |
| `src/services/mcp/client.ts` | MCP 服务器管理 |
| `src/services/diagnosticTracking.ts` | 诊断跟踪 |
| `src/services/tips/tipRegistry.ts` | 提示注册 |
| `src/commands/ide/ide.tsx` | `/ide` 命令实现 |
| `src/components/IdeOnboardingDialog.tsx` | IDE 引导对话框 |
| `src/components/IdeAutoConnectDialog.tsx` | 自动连接对话框 |
| `src/utils/status.tsx` | 状态显示 |
| `src/utils/statusNoticeDefinitions.tsx` | 状态通知定义 |
| `src/utils/jetbrains.ts` | JetBrains 插件检测 |
| `src/utils/idePathConversion.ts` | 路径转换 |

### 依赖导入

```typescript
// 核心依赖
import type { Client } from '@modelcontextprotocol/sdk/client/index.js'
import axios from 'axios'
import { execa } from 'execa'
import { createConnection } from 'net'
import memoize from 'lodash-es/memoize.js'

// 内部工具
import { callIdeRpc } from '../services/mcp/client.js'
import { getGlobalConfig, saveGlobalConfig } from './config.js'
import { isJetBrainsPluginInstalledCached } from './jetbrains.js'
import { WindowsToWSLConverter, checkWSLDistroMatch } from './idePathConversion.js'
// ... 其他工具导入
```

---

## 依赖与外部交互

### 外部依赖

| 包名 | 用途 |
|------|------|
| `@modelcontextprotocol/sdk` | MCP 客户端类型定义 |
| `axios` | 内部扩展下载（Artifactory） |
| `execa` | 进程执行 |
| `lodash-es/memoize` | 函数记忆化 |

### Node.js 内置模块

| 模块 | 用途 |
|------|------|
| `net` | `createConnection` - 端口连通性检测 |
| `os` | 平台检测、临时目录 |
| `path` | 路径操作 |

### 内部依赖

| 模块 | 用途 |
|------|------|
| `services/mcp/client.ts` | `callIdeRpc` - IDE RPC 调用 |
| `services/mcp/types.ts` | MCP 类型定义 |
| `utils/config.ts` | 全局配置读写 |
| `utils/jetbrains.ts` | JetBrains 插件检测 |
| `utils/idePathConversion.ts` | WSL 路径转换 |
| `utils/execFileNoThrow.ts` | 安全执行外部命令 |
| `utils/fsOperations.ts` | 文件系统操作 |
| `utils/platform.ts` | 平台检测 |

### 环境变量

| 环境变量 | 用途 |
|---------|------|
| `CLAUDE_CODE_IDE_HOST_OVERRIDE` | 强制指定 IDE 主机 IP |
| `CLAUDE_CODE_IDE_SKIP_VALID_CHECK` | 跳过工作区验证 |
| `CLAUDE_CODE_IDE_SKIP_AUTO_INSTALL` | 跳过自动安装扩展 |
| `CLAUDE_CODE_AUTO_CONNECT_IDE` | 强制自动连接 IDE |
| `CLAUDE_CODE_SSE_PORT` | IDE 扩展端口（环境注入） |
| `WSL_DISTRO_NAME` | WSL 分发版名称 |

---

## 风险、边界与改进建议

### 已知风险

1. **进程检测的不可靠性**
   - 基于进程名的检测可能被伪装
   - WSL 中 PID 可能不可靠
   - **缓解**：结合端口连通性检查、工作区匹配、PID 祖先链多重验证

2. **扩展安装的竞态条件**
   - 连续调用 `code --install-extension` 可能导致崩溃
   - **缓解**：使用 `sleep(500)` 延迟，但非根本解决

3. **WSL 网络复杂性**
   - WSL2 与 Windows 主机通信涉及虚拟网络
   - 防火墙可能阻止连接
   - **缓解**：尝试默认网关 IP，失败回退到 `127.0.0.1`

4. **锁文件残留**
   - IDE 崩溃时锁文件可能未清理
   - **缓解**：启动时清理过期锁文件，基于 PID 和端口检测

5. **内部员工扩展源依赖**
   - `installFromArtifactory` 依赖内部 Artifactory 服务
   - 需要 `~/.npmrc` 中的认证令牌
   - **风险**：外部构建可能无法访问

### 边界情况

| 场景 | 处理 |
|------|------|
| 多个 IDE 窗口打开 | 优先匹配工作区，其次匹配 PID 祖先链，最后返回所有匹配 |
| 无 IDE 运行 | 返回空数组，引导用户安装 |
| IDE 扩展未安装 | 自动安装（VS Code）或提示安装（JetBrains） |
| WSL 分发版不匹配 | 拒绝连接，避免路径混乱 |
| 锁文件格式不兼容 | 尝试旧格式解析（纯文本路径列表） |
| 端口被占用 | 清理过期锁文件，跳过该端口 |

### 改进建议

1. **扩展安装可靠性**
   ```typescript
   // 实现指数退避重试
   async function installWithRetry(command: string, args: string[], maxRetries = 3) {
     for (let i = 0; i < maxRetries; i++) {
       const result = await execFileNoThrowWithCwd(command, args)
       if (result.code === 0) return result
       await sleep(500 * Math.pow(2, i))  // 指数退避
     }
     throw new Error('Installation failed after retries')
   }
   ```

2. **IDE 检测缓存**
   - 当前每次调用 `detectIDEs()` 都重新扫描
   - 可添加短期缓存（如 5 秒）减少系统调用
   - 需注意 IDE 启动/退出的实时性

3. **更精确的工作区匹配**
   ```typescript
   // 使用 Git 仓库根目录辅助匹配
   const gitRoot = await findGitRoot(cwd)
   const isValid = lockfileInfo.workspaceFolders.some(folder => 
     cwd.startsWith(folder) || gitRoot === await findGitRoot(folder)
   )
   ```

4. **支持更多 IDE**
   - NeoVim/Vim（通过插件）
   - Emacs
   - Sublime Text

5. **单元测试覆盖**
   - 锁文件解析测试
   - 路径转换测试（各种 WSL 场景）
   - 扩展版本比较测试
   - PID 祖先链检测测试

6. **性能优化**
   - `detectRunningIDEs` 使用 `ps`/`tasklist` 命令，每次约 150ms
   - 可考虑使用原生 Node.js 进程枚举（如 `process-list` 包）
   - 权衡：原生模块依赖 vs 命令兼容性
