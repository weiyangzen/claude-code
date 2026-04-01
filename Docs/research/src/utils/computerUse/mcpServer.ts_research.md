# mcpServer.ts 研究文档

## 场景与职责

本文件实现 **Computer Use MCP 服务器的创建和管理**，提供两种运行模式：
1. **进程内服务器**：供 CLI 主进程使用（默认模式）
2. **子进程服务器**：通过 `--computer-use-mcp` 参数启动的独立进程

核心职责：
1. **服务器工厂**：创建配置好的 MCP 服务器实例
2. **应用枚举**：获取已安装应用列表（带超时保护）
3. **工具列表定制**：替换 ListTools 处理器以包含应用名称
4. **子进程入口**：支持独立 MCP 服务器模式

## 功能点目的

### 1. 应用枚举 (`tryGetInstalledAppNames`)

功能：
- 枚举已安装应用用于 `request_access` 工具描述
- 1 秒超时保护（`APP_ENUM_TIMEOUT_MS = 1000`）
- 失败软处理：超时或失败时工具描述省略列表

流程：
1. 调用 `adapter.executor.listInstalledApps()`
2. 与 1 秒超时竞争
3. 失败时吞后续拒绝，避免 unhandledRejection
4. 成功时通过 `filterAppsForDescription` 过滤和净化

### 2. 服务器创建 (`createComputerUseMcpServerForCli`)

流程：
1. 获取 host adapter
2. 获取坐标模式
3. 创建基础服务器（`createComputerUseMcpServer`）
4. 异步枚举应用（不阻塞启动）
5. 构建工具列表（`buildComputerUseTools`）
6. **替换 ListTools 处理器** - 包含应用名称

**设计说明**：
- 异步函数，1 秒枚举超时不阻塞启动
- 从 `client.ts` 的 `await import()` 调用，不在 `main.tsx` 中
- 真实分发通过 `wrapper.tsx` 的 `.call()` 覆盖

### 3. 子进程入口 (`runComputerUseMcpServer`)

用途：
- `--computer-use-mcp` 命令行参数的入口点
- 镜像 `runClaudeInChromeMcpServer` 模式
- stdio 传输，stdin 关闭时退出
- 退出前刷新分析数据

流程：
1. 启用配置（`enableConfigs`）
2. 初始化分析接收器
3. 创建服务器
4. 设置 stdio 传输
5. 注册 stdin 关闭/错误处理程序
6. 连接并运行

## 具体技术实现

### 核心常量

```typescript
const APP_ENUM_TIMEOUT_MS = 1000
```

### 应用枚举实现

```typescript
async function tryGetInstalledAppNames(): Promise<string[] | undefined> {
  const adapter = getComputerUseHostAdapter()
  const enumP = adapter.executor.listInstalledApps()
  
  let timer: ReturnType<typeof setTimeout> | undefined
  const timeoutP = new Promise<undefined>(resolve => {
    timer = setTimeout(resolve, APP_ENUM_TIMEOUT_MS, undefined)
  })
  
  const installed = await Promise.race([enumP, timeoutP])
    .catch(() => undefined)
    .finally(() => clearTimeout(timer))
  
  if (!installed) {
    void enumP.catch(() => {})  // 吞后续拒绝
    logForDebugging('[Computer Use MCP] app enumeration exceeded...')
    return undefined
  }
  
  return filterAppsForDescription(installed, homedir())
}
```

### 服务器创建

```typescript
export async function createComputerUseMcpServerForCli() {
  const adapter = getComputerUseHostAdapter()
  const coordinateMode = getChicagoCoordinateMode()
  const server = createComputerUseMcpServer(adapter, coordinateMode)

  const installedAppNames = await tryGetInstalledAppNames()
  const tools = buildComputerUseTools(
    adapter.executor.capabilities,
    coordinateMode,
    installedAppNames,
  )
  
  server.setRequestHandler(ListToolsRequestSchema, async () =>
    adapter.isDisabled() ? { tools: [] } : { tools },
  )

  return server
}
```

### 子进程入口

```typescript
export async function runComputerUseMcpServer(): Promise<void> {
  enableConfigs()
  initializeAnalyticsSink()

  const server = await createComputerUseMcpServerForCli()
  const transport = new StdioServerTransport()

  let exiting = false
  const shutdownAndExit = async (): Promise<void> => {
    if (exiting) return
    exiting = true
    await Promise.all([shutdown1PEventLogging(), shutdownDatadog()])
    process.exit(0)
  }
  
  process.stdin.on('end', () => void shutdownAndExit())
  process.stdin.on('error', () => void shutdownAndExit())

  await server.connect(transport)
}
```

## 关键代码路径与文件引用

### 本文件导出
- `createComputerUseMcpServerForCli()` - 创建进程内服务器
- `runComputerUseMcpServer()` - 子进程入口

### 调用方
- `src/services/mcp/client.ts:933` - `createComputerUseMcpServerForCli`
- `src/entrypoints/cli.tsx` - `runComputerUseMcpServer`（通过 `--computer-use-mcp`）

### 依赖文件
- `src/utils/computerUse/appNames.ts` - `filterAppsForDescription`
- `src/utils/computerUse/gates.ts` - `getChicagoCoordinateMode`
- `src/utils/computerUse/hostAdapter.ts` - `getComputerUseHostAdapter`
- `src/services/analytics/datadog.ts` - `shutdownDatadog`
- `src/services/analytics/firstPartyEventLogger.ts` - `shutdown1PEventLogging`
- `src/services/analytics/sink.ts` - `initializeAnalyticsSink`
- `src/utils/config.ts` - `enableConfigs`
- `src/utils/debug.ts` - `logForDebugging`

### 外部包
- `@ant/computer-use-mcp` - `createComputerUseMcpServer`, `buildComputerUseTools`
- `@modelcontextprotocol/sdk` - `StdioServerTransport`, `ListToolsRequestSchema`

## 依赖与外部交互

### MCP SDK
- 使用官方 MCP SDK 创建服务器
- stdio 传输用于子进程模式
- 自定义 ListTools 处理器

### 原生模块
- 通过 `hostAdapter.ts` 间接依赖
- 服务器创建时加载原生模块

### 分析系统
- 子进程模式需要初始化分析接收器
- 退出前刷新 Datadog 和 1P 事件日志

### 配置系统
- 子进程模式需要启用配置
- 读取 GrowthBook 等功能标志

## 风险、边界与改进建议

### 已知风险

1. **应用枚举延迟**：
   - 1 秒超时可能不够慢速系统
   - 模型无法获得应用提示

2. **子进程模式**：
   - 独立进程增加内存占用
   - 当前默认使用进程内模式

3. **服务器状态同步**：
   - `isDisabled()` 检查在 ListTools 时
   - 可能与会话中途的 GrowthBook 切换不一致

### 边界情况

1. **枚举超时**：
   - 返回 undefined，工具描述省略列表
   - 分辨率仍在后台继续

2. **功能禁用**：
   - 返回空工具列表
   - 模型看不到 CU 工具

3. **重复创建**：
   - 每次调用创建新服务器实例
   - 适配器是单例，但服务器是新的

### 改进建议

1. **枚举优化**：
   - 缓存应用列表（变化不频繁）
   - 后台定期刷新

2. **超时配置**：
   - 允许环境变量调整 1 秒超时
   - 根据系统性能自适应

3. **错误恢复**：
   - 枚举失败时重试
   - 提供部分列表而非完全省略

4. **可观测性**：
   - 记录枚举时间和成功率
   - 监控工具列表请求

5. **子进程优化**：
   - 评估是否需要保留子进程模式
   - 进程内模式更高效

6. **动态更新**：
   - 支持会话中途更新工具列表
   - 新安装应用后立即可用
