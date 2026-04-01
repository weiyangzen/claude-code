# LSPServerManager.ts 深度研究文档

## 场景与职责

`LSPServerManager.ts` 是 LSP 子系统的核心管理层，负责多 LSP 服务器实例的协调管理、文件扩展名路由、文件生命周期同步（didOpen/didChange/didSave/didClose）。

**核心职责：**
1. **多服务器管理**：管理多个 LSP 服务器实例（如 TypeScript、Python、Go 等）
2. **文件路由**：根据文件扩展名将请求路由到对应的服务器
3. **文件同步**：维护文件在 LSP 服务器中的打开状态，发送文本同步通知
4. **生命周期协调**：初始化所有配置的服务器，优雅关闭
5. **请求代理**：将 LSP 工具请求转发到适当的服务器

**在系统中的位置：**
- 被 `manager.ts` 调用，作为单例管理器的底层实现
- 调用 `LSPServerInstance.ts` 管理单个服务器
- 被 `LSPTool.ts` 通过 `manager.ts` 间接使用
- 属于 LSP 架构的上层（LSPServerInstance ← LSPServerManager ← manager.ts）

---

## 功能点目的

### 1. LSPServerManager 接口

```typescript
export type LSPServerManager = {
  initialize(): Promise<void>           // 加载配置，创建实例
  shutdown(): Promise<void>             // 关闭所有服务器
  getServerForFile(filePath): LSPServerInstance | undefined  // 获取服务器
  ensureServerStarted(filePath): Promise<LSPServerInstance | undefined>  // 确保启动
  sendRequest<T>(filePath, method, params): Promise<T | undefined>  // 发送请求
  getAllServers(): Map<string, LSPServerInstance>  // 获取所有服务器
  openFile(filePath, content): Promise<void>   // didOpen
  changeFile(filePath, content): Promise<void> // didChange
  saveFile(filePath): Promise<void>            // didSave
  closeFile(filePath): Promise<void>           // didClose
  isFileOpen(filePath): boolean                // 检查文件状态
}
```

### 2. 扩展名路由

**路由映射构建（Lines 88-117）：**
```typescript
// extensionMap: 扩展名 → 服务器名称列表
const extensionMap: Map<string, string[]> = new Map()

for (const [serverName, config] of Object.entries(serverConfigs)) {
  const fileExtensions = Object.keys(config.extensionToLanguage)
  for (const ext of fileExtensions) {
    const normalized = ext.toLowerCase()
    if (!extensionMap.has(normalized)) {
      extensionMap.set(normalized, [])
    }
    extensionMap.get(normalized)!.push(serverName)
  }
}
```

**路由策略：**
- 多个服务器支持同一扩展名时，使用第一个注册的服务器
- 扩展名大小写不敏感（统一转为小写）

### 3. 文件同步管理

**打开文件追踪：**
```typescript
// openedFiles: URI → 服务器名称
const openedFiles: Map<string, string> = new Map()
```

**同步策略：**

| 方法 | LSP 通知 | 说明 |
|------|----------|------|
| `openFile()` | `textDocument/didOpen` | 文件首次打开，发送完整内容 |
| `changeFile()` | `textDocument/didChange` | 文件修改，发送增量变化 |
| `saveFile()` | `textDocument/didSave` | 文件保存通知 |
| `closeFile()` | `textDocument/didClose` | 文件关闭，清理追踪 |

**关键逻辑：**
- `changeFile` 时如果文件未打开，自动调用 `openFile`
- 避免重复打开同一文件（检查 `openedFiles`）
- 关闭时从追踪中移除，允许后续重新打开

### 4. 配置验证

**初始化时验证（Lines 91-104）：**
```typescript
if (!config.command) {
  throw new Error(`Server ${serverName} missing required 'command' field`)
}
if (!config.extensionToLanguage || Object.keys(config.extensionToLanguage).length === 0) {
  throw new Error(`Server ${serverName} missing required 'extensionToLanguage' field`)
}
```

**容错处理：**
- 单个服务器配置错误不阻断其他服务器初始化
- 错误被捕获、记录，继续处理其他服务器

### 5. workspace/configuration 处理

**处理服务器配置请求（Lines 124-135）：**
```typescript
instance.onRequest('workspace/configuration', (params) => {
  logForDebugging(`LSP: Received workspace/configuration request from ${serverName}`)
  // 返回空配置，避免服务器报错
  return params.items.map(() => null)
})
```

**背景：**
- 某些服务器（如 TypeScript）即使客户端声明不支持也会发送此请求
- 返回 `null` 满足协议要求，不提供实际配置

---

## 具体技术实现

### 关键数据结构

```typescript
// 私有状态
const servers: Map<string, LSPServerInstance> = new Map()        // 服务器实例
const extensionMap: Map<string, string[]> = new Map()            // 扩展名路由
const openedFiles: Map<string, string> = new Map()               // 已打开文件
```

### 文件 URI 处理

```typescript
import { pathToFileURL } from 'url'

const fileUri = pathToFileURL(path.resolve(filePath)).href
// 结果: file:///home/user/project/src/file.ts
```

### 语言 ID 映射

```typescript
const ext = path.extname(filePath).toLowerCase()
const languageId = server.config.extensionToLanguage[ext] || 'plaintext'
```

### didOpen 参数构建

```typescript
await server.sendNotification('textDocument/didOpen', {
  textDocument: {
    uri: fileUri,
    languageId,
    version: 1,  // 文档版本
    text: content,
  },
})
```

### didChange 参数构建

```typescript
await server.sendNotification('textDocument/didChange', {
  textDocument: {
    uri: fileUri,
    version: 1,  // 版本号（当前固定为1）
  },
  contentChanges: [{ text: content }],  // 完整内容替换
})
```

---

## 关键代码路径与文件引用

### 内部依赖

| 文件 | 用途 |
|------|------|
| `path` (Node.js) | 路径处理 |
| `url.pathToFileURL` | 文件路径转 URI |
| `../../utils/debug.js` | `logForDebugging()` - 调试日志 |
| `../../utils/errors.js` | `errorMessage()` - 错误处理 |
| `../../utils/log.js` | `logError()` - 错误日志 |
| `./config.js` | `getAllLspServers()` - 获取服务器配置 |
| `./LSPServerInstance.js` | `createLSPServerInstance()` - 创建实例 |
| `./types.js` | `ScopedLspServerConfig` 类型 |

### 被调用方

| 文件 | 调用方式 |
|------|----------|
| `manager.ts` | `createLSPServerManager()` - 创建管理器实例 |
| `passiveFeedback.ts` | `manager.getAllServers()` - 获取所有服务器注册处理器 |

### 关键代码行

| 行号 | 功能 |
|------|------|
| 16-43 | `LSPServerManager` 接口定义 |
| 59-64 | 私有状态定义 |
| 71-148 | `initialize()` 实现 |
| 88-117 | 扩展名路由映射构建 |
| 124-135 | workspace/configuration 处理器注册 |
| 157-185 | `shutdown()` 实现 |
| 192-207 | `getServerForFile()` 实现 |
| 215-236 | `ensureServerStarted()` 实现 |
| 244-263 | `sendRequest()` 实现 |
| 270-310 | `openFile()` 实现 |
| 312-343 | `changeFile()` 实现 |
| 349-368 | `saveFile()` 实现 |
| 377-400 | `closeFile()` 实现 |
| 402-405 | `isFileOpen()` 实现 |

---

## 依赖与外部交互

### 外部依赖

**Node.js 内置：**
- `path` - 路径解析
- `url` - URL 转换

### 配置来源

**`getAllLspServers()`（来自 config.ts）：**
- 从插件加载 LSP 服务器配置
- 支持 `.lsp.json` 文件和 manifest 声明
- 返回 `Record<string, ScopedLspServerConfig>`

### LSP 协议交互

**文本同步通知：**
| 通知 | 触发时机 | 参数 |
|------|----------|------|
| `textDocument/didOpen` | 文件首次访问 | `{textDocument: {uri, languageId, version, text}}` |
| `textDocument/didChange` | 文件内容修改 | `{textDocument: {uri, version}, contentChanges: [{text}]}` |
| `textDocument/didSave` | 文件保存 | `{textDocument: {uri}}` |
| `textDocument/didClose` | 文件关闭 | `{textDocument: {uri}}` |

**客户端请求处理：**
| 请求 | 处理方式 |
|------|----------|
| `workspace/configuration` | 返回 `null` 数组 |

### 文件追踪状态

```
openedFiles Map 状态示例:
{
  "file:///home/user/project/src/main.ts" -> "plugin:typescript:typescript-language-server",
  "file:///home/user/project/src/utils.ts" -> "plugin:typescript:typescript-language-server",
  "file:///home/user/project/src/app.py" -> "plugin:python:pyright"
}
```

---

## 风险、边界与改进建议

### 已知风险

**1. 版本号固定**
```typescript
version: 1  // 始终为1，不递增
```
- 当前实现不维护文档版本号
- 某些 LSP 服务器可能依赖版本号进行增量同步优化

**2. 全量内容替换**
```typescript
contentChanges: [{ text: content }]  // 完整内容，非增量
```
- 每次修改发送完整文件内容
- 大文件频繁修改时性能开销大

**3. 文件关闭未完全集成**
```typescript
/**
 * NOTE: Currently available but not yet integrated with compact flow.
 * TODO: Integrate with compact - call closeFile() when compact removes files
 */
```
- 文件从上下文移除时未通知 LSP 服务器
- 可能导致服务器内存持续增长

**4. 多服务器竞争**
- 多个服务器支持同一扩展名时，只使用第一个
- 无优先级或能力协商机制

### 边界情况

**1. 无服务器处理文件**
```typescript
const server = getServerForFile(filePath)
if (!server) return undefined  // 静默返回
```

**2. 服务器启动失败**
```typescript
try {
  await server.start()
} catch (error) {
  logError(...)
  throw error  // 向上传播
}
```

**3. 文件路径解析**
```typescript
const fileUri = pathToFileURL(path.resolve(filePath)).href
```
- 相对路径转为绝对路径
- 再转为 file:// URI

### 改进建议

**1. 增量同步支持**
```typescript
// 建议：维护文档版本和内容哈希
interface DocumentState {
  version: number
  content: string
  hash: string
}

// 计算差异并发送增量变化
const changes = computeDiff(oldContent, newContent)
await server.sendNotification('textDocument/didChange', {
  textDocument: { uri, version: ++docState.version },
  contentChanges: changes,  // 增量变化数组
})
```

**2. 文档版本管理**
```typescript
// 建议：维护每个打开文件的版本号
const documentVersions: Map<string, number> = new Map()

function incrementVersion(uri: string): number {
  const version = (documentVersions.get(uri) || 0) + 1
  documentVersions.set(uri, version)
  return version
}
```

**3. 服务器选择策略**
```typescript
// 建议：基于能力的服务器选择
function selectServer(servers: LSPServerInstance[], operation: string): LSPServerInstance {
  // 选择支持该操作的服务器
  return servers.find(s => s.capabilities[operation]) || servers[0]
}
```

**4. 文件关闭集成**
```typescript
// 建议：在 compact 流程中集成
// 当文件从上下文中移除时调用
function onFileRemovedFromContext(filePath: string) {
  if (isFileOpen(filePath)) {
    void closeFile(filePath)
  }
}
```

**5. 健康检查与故障转移**
```typescript
// 建议：定期检查服务器健康
async function healthCheckAll(): Promise<void> {
  for (const [name, server] of servers) {
    if (!server.isHealthy()) {
      logForDebugging(`Server ${name} unhealthy, attempting restart`)
      await server.restart().catch(err => logError(err))
    }
  }
}
```

**6. 路由缓存优化**
```typescript
// 建议：缓存文件路径到服务器的映射
const routeCache: Map<string, LSPServerInstance> = new Map()

function getServerForFile(filePath: string): LSPServerInstance | undefined {
  if (routeCache.has(filePath)) {
    return routeCache.get(filePath)
  }
  const server = computeServerForFile(filePath)
  if (server) routeCache.set(filePath, server)
  return server
}
```

### 测试建议

**关键测试场景：**
1. 多服务器配置，同一扩展名路由到正确服务器
2. 文件打开/修改/保存/关闭完整流程
3. 服务器启动失败时的错误处理
4. 大文件（>10MB）的同步性能
5. 并发文件操作（多个文件同时修改）
6. 无服务器支持文件类型的处理
7. 服务器崩溃后的恢复和重新路由

### 相关配置

**ScopedLspServerConfig 关键字段：**
```typescript
{
  command: string,                    // 启动命令
  args?: string[],                    // 参数
  extensionToLanguage: Record<string, string>,  // {".ts": "typescript"}
  env?: Record<string, string>,       // 环境变量
  workspaceFolder?: string,           // 工作区
  initializationOptions?: unknown,    // 初始化选项
  maxRestarts?: number,               // 最大重启次数
  scope: 'dynamic' | 'static',        // 作用域
  source: string                      // 来源插件名
}
```
