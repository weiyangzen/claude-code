# MCP Command Index 研究文档

## 场景与职责

`index.ts` 是 MCP 命令模块的入口文件，定义了 `mcp` 命令的元数据配置。它遵循 Claude Code 的 Command 接口规范，将命令注册到 CLI 系统中。这是一个轻量级的模块定义文件，负责延迟加载实际的命令实现。

## 功能点目的

### 1. 命令元数据定义
定义 MCP 命令的基本信息：
- **名称**: `mcp`
- **描述**: "Manage MCP servers"
- **类型**: `local-jsx` - 使用本地 JSX 组件实现的命令
- **立即执行**: `immediate: true` - 命令立即可用，不需要额外初始化
- **参数提示**: `[enable|disable [server-name]]` - 显示可用的子命令参数

### 2. 延迟加载
通过 `load: () => import('./mcp.js')` 实现代码分割和延迟加载，优化启动性能。

## 具体技术实现

### 代码结构
```typescript
import type { Command } from '../../commands.js'

const mcp = {
  type: 'local-jsx',
  name: 'mcp',
  description: 'Manage MCP servers',
  immediate: true,
  argumentHint: '[enable|disable [server-name]]',
  load: () => import('./mcp.js'),
} satisfies Command

export default mcp
```

### 关键属性说明

| 属性 | 值 | 说明 |
|------|-----|------|
| `type` | `'local-jsx'` | 命令使用本地 JSX 组件实现，而非纯 CLI 命令 |
| `name` | `'mcp'` | 命令在 CLI 中的调用名称 |
| `description` | `'Manage MCP servers'` | 帮助文本中显示的描述 |
| `immediate` | `true` | 命令立即可用，不需要等待某些初始化完成 |
| `argumentHint` | `'[enable\|disable [server-name]]'` | 参数提示，显示支持的子命令 |
| `load` | `() => import('./mcp.js')` | 延迟加载实际实现的工厂函数 |

## 关键代码路径与文件引用

### 依赖关系
```
index.ts
  → imports: '../../commands.js' (Command 类型定义)
  → lazy loads: './mcp.js' (实际实现)
    → MCPSettings 组件 [src/components/mcp/index.js]
    → MCPReconnect 组件 [src/components/mcp/MCPReconnect.js]
    → useMcpToggleEnabled hook [src/services/mcp/MCPConnectionManager.js]
```

### 调用链
```
CLI 命令解析器
  → 匹配 'mcp' 命令
    → 执行 load() → import('./mcp.js')
      → 调用 call(onDone, context, args) 函数
        → 根据 args 渲染不同组件:
          - 'no-redirect' → MCPSettings
          - 'reconnect <name>' → MCPReconnect
          - 'enable/disable [target]' → MCPToggle
          - default → MCPSettings 或 PluginSettings (ant 构建)
```

## 依赖与外部交互

### 类型依赖
- `Command` 类型来自 `../../commands.js`，定义了命令模块的契约接口

### 延迟加载目标
- `./mcp.js` (编译后的 `mcp.tsx`) 包含实际的 React 组件实现

## 风险、边界与改进建议

### 风险点

1. **类型安全**
   - 使用 `satisfies Command` 确保类型兼容
   - 如果 `Command` 接口变更，此文件需要同步更新

2. **构建依赖**
   - 依赖 `./mcp.js` 在构建时存在
   - 如果 `mcp.tsx` 编译失败，整个 MCP 命令将不可用

### 边界情况

1. **参数提示与实际功能可能不同步**
   - `argumentHint` 显示 `[enable|disable [server-name]]`
   - 但实际 `mcp.tsx` 还支持 `no-redirect` 和 `reconnect` 参数
   - **建议**: 更新 `argumentHint` 以反映所有可用参数

2. **ant 构建特殊行为**
   - 在 "ant" 构建中，基础 `/mcp` 命令会重定向到 `/plugins installed`
   - 这是硬编码的构建时行为 (`if ("external" === 'ant')`)
   - 外部构建永远不会触发此重定向

### 改进建议

1. **更新参数提示**
   ```typescript
   argumentHint: '[enable|disable [server-name]|reconnect <name>|no-redirect]'
   ```

2. **添加命令分组**（如果 CLI 框架支持）
   - 将 MCP 相关子命令分组显示，提高可发现性

3. **文档注释**
   ```typescript
   /**
    * MCP (Model Context Protocol) server management command.
    * 
    * Subcommands:
    *   - enable [server-name]: Enable MCP server(s)
    *   - disable [server-name]: Disable MCP server(s)
    *   - reconnect <name>: Reconnect to a specific MCP server
    *   - no-redirect: Open MCP settings without redirect (testing)
    * 
    * CLI subcommands (via 'claude mcp'):
    *   - add: Add a new MCP server
    *   - xaa: Manage XAA IdP connection
    */
   ```

4. **考虑拆分 immediate 标志**
   - 当前 `immediate: true` 可能意味着命令不需要等待某些初始化
   - 考虑是否需要根据具体子命令调整此行为
