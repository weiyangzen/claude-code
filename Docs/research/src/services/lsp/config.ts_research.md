# config.ts 深度研究文档

## 场景与职责

`config.ts` 是 LSP 子系统的配置加载入口，负责从插件系统中提取所有 LSP 服务器配置，并进行聚合和错误处理。

**核心职责：**
1. **配置聚合**：从所有启用的插件中收集 LSP 服务器配置
2. **并行加载**：并行加载多个插件的 LSP 配置以提高性能
3. **错误隔离**：单个插件配置错误不影响其他插件
4. **结果合并**：将多个插件的配置合并为统一的配置对象

**在系统中的位置：**
- 被 `LSPServerManager.ts` 调用，在初始化时获取服务器配置
- 依赖 `lspPluginIntegration.ts` 提供的插件 LSP 加载功能
- 属于 LSP 架构的配置层（配置加载 → LSPServerManager → LSPServerInstance）

---

## 功能点目的

### 1. getAllLspServers() 主函数

**函数签名：**
```typescript
export async function getAllLspServers(): Promise<{
  servers: Record<string, ScopedLspServerConfig>
}>
```

**返回值：**
- `servers`: 以服务器名称（含插件作用域）为键的配置对象
- 例如：`{"plugin:typescript:typescript-language-server": {...}}`

### 2. 并行加载策略

**核心逻辑（Lines 27-43）：**
```typescript
const results = await Promise.all(
  plugins.map(async plugin => {
    const errors: PluginError[] = []
    try {
      const scopedServers = await getPluginLspServers(plugin, errors)
      return { plugin, scopedServers, errors }
    } catch (e) {
      // 防御性：单个插件失败不丢失其他结果
      logForDebugging(`Failed to load LSP servers for plugin ${plugin.name}: ${e}`)
      return { plugin, scopedServers: undefined, errors }
    }
  })
)
```

**设计决策：**
- 使用 `Promise.all` 并行加载所有插件
- 保持原始顺序，确保配置合并时的优先级一致（后加载的覆盖先加载的）
- 单个插件失败返回 `undefined`，不影响整体流程

### 3. 结果合并

**合并逻辑（Lines 45-62）：**
```typescript
for (const { plugin, scopedServers, errors } of results) {
  const serverCount = scopedServers ? Object.keys(scopedServers).length : 0
  if (serverCount > 0) {
    Object.assign(allServers, scopedServers)  // 合并配置
    logForDebugging(`Loaded ${serverCount} LSP server(s) from plugin: ${plugin.name}`)
  }
  
  if (errors.length > 0) {
    logForDebugging(`${errors.length} error(s) loading LSP servers from plugin: ${plugin.name}`)
  }
}
```

**合并规则：**
- 使用 `Object.assign` 进行浅合并
- 后加载插件的配置优先级更高（同名服务器被覆盖）
- 服务器名称已包含插件作用域（`plugin:${pluginName}:${serverName}`），冲突概率低

### 4. 错误处理策略

**两层错误处理：**

1. **单个插件加载失败**：
   - 捕获异常，记录日志
   - 返回 `undefined`，继续处理其他插件

2. **配置验证错误**：
   - 由 `getPluginLspServers` 收集到 `errors` 数组
   - 记录错误数量，但不阻断流程

3. **整体加载失败**：
   - 最外层 try-catch 捕获
   - 记录错误，返回空配置（LSP 功能降级，不阻断应用）

---

## 具体技术实现

### 依赖模块

```typescript
import type { PluginError } from '../../types/plugin.js'
import { logForDebugging } from '../../utils/debug.js'
import { errorMessage, toError } from '../../utils/errors.js'
import { logError } from '../../utils/log.js'
import { getPluginLspServers } from '../../utils/plugins/lspPluginIntegration.js'
import { loadAllPluginsCacheOnly } from '../../utils/plugins/pluginLoader.js'
import type { ScopedLspServerConfig } from './types.js'
```

### 加载流程

```
getAllLspServers()
    ↓
loadAllPluginsCacheOnly()  →  获取所有启用的插件
    ↓
Promise.all(
  plugins.map(plugin => 
    getPluginLspServers(plugin, errors)  →  从单个插件加载
  )
)
    ↓
合并所有结果  →  Object.assign(allServers, scopedServers)
    ↓
返回 { servers: allServers }
```

### 关键代码行

| 行号 | 功能 |
|------|------|
| 15-17 | 函数返回类型定义 |
| 18 | `allServers` 初始化 |
| 22 | 加载所有启用的插件 |
| 27-43 | 并行加载所有插件的 LSP 配置 |
| 32 | 调用 `getPluginLspServers` 加载单个插件 |
| 45-62 | 合并结果并记录日志 |
| 49 | `Object.assign` 合并配置 |
| 67-74 | 整体错误处理 |

---

## 关键代码路径与文件引用

### 内部依赖

| 文件 | 用途 |
|------|------|
| `../../types/plugin.js` | `PluginError` 类型 |
| `../../utils/debug.js` | `logForDebugging()` - 调试日志 |
| `../../utils/errors.js` | `errorMessage()`, `toError()` - 错误处理 |
| `../../utils/log.js` | `logError()` - 错误日志 |
| `../../utils/plugins/lspPluginIntegration.js` | `getPluginLspServers()` - 插件 LSP 加载 |
| `../../utils/plugins/pluginLoader.js` | `loadAllPluginsCacheOnly()` - 插件加载 |
| `./types.js` | `ScopedLspServerConfig` 类型 |

### 被调用方

| 文件 | 调用方式 |
|------|----------|
| `LSPServerManager.ts` | `getAllLspServers()` - 初始化时获取配置 |

### 依赖关系图

```
config.ts
    ├── lspPluginIntegration.ts
    │       ├── schemas.ts (LspServerConfigSchema)
    │       ├── pluginOptionsStorage.ts
    │       └── pluginDirectories.ts
    ├── pluginLoader.ts
    │       └── (插件系统核心)
    └── types.js (类型定义)
```

---

## 依赖与外部交互

### 外部依赖

**插件系统：**
- `loadAllPluginsCacheOnly()` - 获取缓存的插件列表
- `getPluginLspServers()` - 从单个插件提取 LSP 配置

### 配置来源

**插件配置来源（由 `lspPluginIntegration.ts` 处理）：**
1. `.lsp.json` 文件（插件根目录）
2. `manifest.lspServers` 字段（内联或引用外部文件）

**配置处理流程：**
```
插件目录
    ↓
.lsp.json 或 manifest.lspServers
    ↓
环境变量解析 (${CLAUDE_PLUGIN_ROOT}, ${user_config.X})
    ↓
添加插件作用域 (plugin:${pluginName}:${serverName})
    ↓
ScopedLspServerConfig
```

### 错误类型

**PluginError（来自 types/plugin.js）：**
```typescript
type PluginError = {
  type: string           // 错误类型，如 'lsp-config-invalid'
  plugin: string         // 插件名称
  serverName?: string    // 服务器名称
  validationError?: string  // 验证错误信息
  source: string         // 错误来源
}
```

---

## 风险、边界与改进建议

### 已知风险

**1. 配置合并冲突**
```typescript
Object.assign(allServers, scopedServers)
```
- 同名服务器配置会被覆盖
- 虽然服务器名称包含插件作用域，但用户可能手动配置冲突名称

**2. 无配置验证**
- 当前仅依赖 `lspPluginIntegration.ts` 的验证
- 无统一的配置后验证（如检查命令是否存在）

**3. 日志信息有限**
- 仅记录加载的服务器数量
- 未记录具体加载了哪些服务器（调试用）

### 边界情况

**1. 无插件场景**
```typescript
const { enabled: plugins } = await loadAllPluginsCacheOnly()
// plugins 为空数组，results 为空，返回空 servers
```

**2. 所有插件加载失败**
```typescript
try {
  // ...
} catch (error) {
  logError(toError(error))
  logForDebugging(`Error loading LSP servers: ${errorMessage(error)}`)
}
// 返回空配置，LSP 功能静默禁用
```

**3. 插件返回空配置**
```typescript
const serverCount = scopedServers ? Object.keys(scopedServers).length : 0
if (serverCount > 0) {
  Object.assign(allServers, scopedServers)
}
// 空配置被跳过
```

### 改进建议

**1. 配置冲突检测**
```typescript
// 建议：检测同名服务器配置
for (const serverName of Object.keys(scopedServers)) {
  if (allServers[serverName]) {
    logForDebugging(`Warning: LSP server '${serverName}' configuration overridden by ${plugin.name}`)
  }
}
```

**2. 详细日志记录**
```typescript
// 建议：记录每个加载的服务器
for (const serverName of Object.keys(scopedServers)) {
  logForDebugging(`LSP server loaded: ${serverName} (command: ${scopedServers[serverName].command})`)
}
```

**3. 配置预验证**
```typescript
// 建议：验证命令可执行
import { which } from 'which'

async function validateServerConfig(config: ScopedLspServerConfig): Promise<boolean> {
  const exists = await which(config.command).catch(() => null)
  if (!exists) {
    logError(new Error(`LSP server command not found: ${config.command}`))
    return false
  }
  return true
}
```

**4. 性能优化**
```typescript
// 建议：缓存配置结果
let cachedServers: Record<string, ScopedLspServerConfig> | undefined
let cacheTimestamp: number

export async function getAllLspServers(): Promise<{servers: Record<string, ScopedLspServerConfig>}> {
  if (cachedServers && Date.now() - cacheTimestamp < CACHE_TTL) {
    return { servers: cachedServers }
  }
  // ... 重新加载
}
```

**5. 配置来源追踪**
```typescript
// 建议：添加来源信息到配置
interface ScopedLspServerConfigWithMeta extends ScopedLspServerConfig {
  _meta: {
    pluginName: string
    source: 'lsp.json' | 'manifest'
    loadedAt: string
  }
}
```

### 测试建议

**关键测试场景：**
1. 多个插件提供 LSP 配置的合并
2. 单个插件配置错误的隔离
3. 无插件/无 LSP 配置的场景
4. 同名服务器配置的覆盖行为
5. 插件加载失败的降级处理
6. 大量插件的并行加载性能

### 相关文件

| 文件 | 关系 |
|------|------|
| `lspPluginIntegration.ts` | 被调用，处理单个插件的配置加载 |
| `pluginLoader.ts` | 被调用，获取插件列表 |
| `schemas.ts` | 被 `lspPluginIntegration.ts` 使用，配置验证 Schema |
| `LSPServerManager.ts` | 调用方，使用配置创建服务器实例 |
