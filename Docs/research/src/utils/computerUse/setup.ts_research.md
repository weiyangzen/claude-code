# setup.ts 研究文档

## 场景与职责

本文件负责 **Computer Use MCP 的初始配置设置**，构建动态 MCP 配置和允许的工具列表。它是 CLI 启动时配置 Computer Use 功能的入口点。

核心职责：
1. **MCP 配置构建**：创建 `ScopedMcpServerConfig` 对象
2. **允许工具列表**：构建 `mcp__computer-use__*` 工具名称列表
3. **进程内服务器设置**：配置命令/参数（实际不生成子进程）

## 功能点目的

### 1. MCP 配置 (`mcpConfig`)

配置结构：
```typescript
{
  [COMPUTER_USE_MCP_SERVER_NAME]: {
    type: 'stdio',
    command: process.execPath,
    args: [...],
    scope: 'dynamic',
  }
}
```

**关键设计**：
- `command`/`args` 实际上永远不会生成子进程
- `client.ts` 按名称拦截并使用进程内服务器
- 配置只需要存在且 `type: 'stdio'` 以进入正确分支
- 镜像 Chrome 的 setup 模式

### 2. 允许工具列表 (`allowedTools`)

构建流程：
1. 调用 `buildComputerUseTools` 获取工具定义
2. 使用 `buildMcpToolName` 构建完整名称
3. 格式：`mcp__computer-use__{toolName}`

**目的**：
- 这些工具绕过正常权限提示
- 包的 `request_access` 处理整个会话的批准
- API 后端检测 `mcp__computer-use__*` 名称并注入 CU 可用性提示到系统提示

### 3. 参数配置

捆绑模式：
```typescript
['--computer-use-mcp']
```

开发模式：
```typescript
[
  join(fileURLToPath(import.meta.url), '..', 'cli.js'),
  '--computer-use-mcp',
]
```

## 具体技术实现

### 核心函数

```typescript
export function setupComputerUseMCP(): {
  mcpConfig: Record<string, ScopedMcpServerConfig>
  allowedTools: string[]
}
```

### 实现细节

```typescript
export function setupComputerUseMCP() {
  // 构建允许的工具列表
  const allowedTools = buildComputerUseTools(
    CLI_CU_CAPABILITIES,
    getChicagoCoordinateMode(),
  ).map(t => buildMcpToolName(COMPUTER_USE_MCP_SERVER_NAME, t.name))

  // 根据模式确定参数
  const args = isInBundledMode()
    ? ['--computer-use-mcp']
    : [
        join(fileURLToPath(import.meta.url), '..', 'cli.js'),
        '----computer-use-mcp',
      ]

  return {
    mcpConfig: {
      [COMPUTER_USE_MCP_SERVER_NAME]: {
        type: 'stdio',
        command: process.execPath,
        args,
        scope: 'dynamic',
      } as const,
    },
    allowedTools,
  }
}
```

### 常量使用

- `CLI_CU_CAPABILITIES`：来自 `common.ts`，声明平台能力
- `COMPUTER_USE_MCP_SERVER_NAME`：来自 `common.ts`，服务器标识符
- `getChicagoCoordinateMode()`：来自 `gates.ts`，坐标模式

## 关键代码路径与文件引用

### 本文件导出
- `setupComputerUseMCP()` - 返回 MCP 配置和允许工具列表

### 调用方
- `src/services/mcp/config.ts` - 整合到全局 MCP 配置

### 依赖文件
- `src/utils/computerUse/common.ts` - `CLI_CU_CAPABILITIES`, `COMPUTER_USE_MCP_SERVER_NAME`
- `src/utils/computerUse/gates.ts` - `getChicagoCoordinateMode`
- `src/services/mcp/mcpStringUtils.ts` - `buildMcpToolName`
- `src/utils/bundledMode.ts` - `isInBundledMode`

### 外部包
- `@ant/computer-use-mcp` - `buildComputerUseTools`

## 依赖与外部交互

### MCP 系统集成
- 返回的配置被合并到全局 MCP 服务器配置
- `scope: 'dynamic'` 表示动态服务器（非持久化）

### 进程内拦截
- `client.ts` 检测 `isComputerUseMCPServer(name)`
- 使用 `createComputerUseMcpServerForCli()` 创建进程内服务器
- 配置的 `command`/`args` 实际上不使用

### API 后端集成
- 工具名称格式 `mcp__computer-use__*` 被 API 识别
- 触发 `COMPUTER_USE_MCP_AVAILABILITY_HINT` 注入系统提示
- Cowork 使用相同名称实现相同目的

## 风险、边界与改进建议

### 已知风险

1. **配置与实际行为不一致**：
   - 配置声明使用 stdio 子进程
   - 实际使用进程内服务器
   - 可能造成混淆

2. **工具名称硬编码**：
   - 名称格式必须与 API 后端约定一致
   - 更改需要同步更新后端

3. **捆绑模式检测**：
   - `isInBundledMode()` 决定参数格式
   - 检测错误导致启动失败

### 边界情况

1. **功能禁用**：
   - 即使功能禁用，配置仍被创建
   - `isDisabled()` 在服务器层处理

2. **坐标模式变化**：
   - 首次读取后冻结
   - 会话期间保持一致

3. **重复调用**：
   - 每次启动时调用
   - 无缓存，但开销很小

### 改进建议

1. **配置明确化**：
   - 添加注释说明进程内拦截
   - 考虑添加 `type: 'in-process'` 明确声明

2. **工具名称常量**：
   - 将名称格式提取到共享常量
   - 避免与 API 后端不一致

3. **延迟初始化**：
   - 考虑延迟到首次使用时创建配置
   - 减少启动开销

4. **可观测性**：
   - 记录配置创建
   - 监控工具列表大小

5. **文档完善**：
   - 在代码中详细说明进程内拦截机制
   - 解释为什么需要虚拟配置

6. **测试覆盖**：
   - 测试捆绑/开发模式参数
   - 验证工具名称格式
