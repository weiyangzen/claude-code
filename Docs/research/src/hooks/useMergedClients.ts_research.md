# useMergedClients.ts 深度研究文档

## 场景与职责

`useMergedClients` 是一个用于合并 MCP（Model Context Protocol）客户端列表的工具钩子。它将初始客户端列表与动态发现的 MCP 客户端合并，确保去重并缓存结果。

### 核心场景

1. **MCP 客户端合并**：合并初始配置和动态发现的 MCP 服务器连接
2. **去重处理**：基于 `name` 字段去重
3. **性能优化**：使用 `useMemo` 缓存合并结果

### 与其他组件的关系

- 被 `REPL.tsx` 使用，管理 MCP 客户端列表
- 与 MCP 连接管理器配合，处理服务器连接
- 简单的工具函数包装，无复杂业务逻辑

---

## 功能点目的

### 1. 客户端合并

合并两个 MCP 客户端列表：
- `initialClients`: 启动时配置的 MCP 服务器
- `mcpClients`: 动态发现的 MCP 服务器

### 2. 去重策略

使用 `lodash-es/uniqBy` 基于 `name` 字段去重：
- 如果两个列表有同名客户端，保留 `initialClients` 中的（因为它在前面）
- 确保每个 MCP 服务器只出现一次

### 3. 缓存优化

使用 `useMemo` 缓存合并结果：
- 只有当 `initialClients` 或 `mcpClients` 变化时才重新计算
- 避免不必要的重渲染

---

## 具体技术实现

### 关键数据结构

```typescript
import type { MCPServerConnection } from '../services/mcp/types.js'

// MCPServerConnection 定义
interface MCPServerConnection {
  name: string           // 服务器名称（唯一标识）
  // 其他连接属性...
}
```

### 核心实现

```typescript
// 纯合并函数
export function mergeClients(
  initialClients: MCPServerConnection[] | undefined,
  mcpClients: readonly MCPServerConnection[] | undefined,
): MCPServerConnection[] {
  if (initialClients && mcpClients && mcpClients.length > 0) {
    return uniqBy([...initialClients, ...mcpClients], 'name')
  }
  return initialClients || []
}

// React 钩子包装
export function useMergedClients(
  initialClients: MCPServerConnection[] | undefined,
  mcpClients: MCPServerConnection[] | undefined,
): MCPServerConnection[] {
  return useMemo(
    () => mergeClients(initialClients, mcpClients),
    [initialClients, mcpClients],
  )
}
```

### 合并逻辑

```
initialClients: [A, B]
mcpClients: [B, C]
          ↓ mergeClients
result: [A, B, C]  // B 来自 initialClients（去重保留前者）
```

---

## 关键代码路径与文件引用

```
src/hooks/useMergedClients.ts
├── MCPServerConnection 类型       # 来自 services/mcp/types.js
├── mergeClients()                 # 行 5-13: 纯合并函数
└── useMergedClients()             # 行 15-23: React 钩子
    └── useMemo                    # 行 19-22: 缓存合并结果
```

### 依赖文件

```
src/services/mcp/types.ts
└── MCPServerConnection 类型定义

lodash-es/uniqBy.js
└── uniqBy 函数
```

---

## 依赖与外部交互

### React Hooks 使用

- `useMemo`: 缓存合并结果

### 外部依赖

```typescript
import uniqBy from 'lodash-es/uniqBy.js'
```

### 类型依赖

```typescript
import type { MCPServerConnection } from '../services/mcp/types.js'
```

---

## 风险、边界与改进建议

### 已知风险

1. **去重策略简单**
   - 仅基于 `name` 去重
   - 如果同名但配置不同，可能产生意外行为

2. **无验证**
   - 不验证客户端连接状态
   - 可能包含已断开的连接

3. **顺序依赖**
   - `initialClients` 优先，但可能不是期望的行为

### 边界情况

| 场景 | 行为 |
|-----|------|
| initialClients=undefined | 返回 [] |
| mcpClients=undefined | 返回 initialClients |
| mcpClients=[] | 返回 initialClients |
| 两者都有内容 | 合并并去重 |
| 同名不同配置 | 保留 initialClients 中的 |

### 改进建议

1. **智能合并**
   - 比较配置内容，不只是名称
   - 处理配置冲突（如不同的命令路径）

2. **状态感知**
   - 过滤掉已断开的连接
   - 标记连接状态

3. **优先级配置**
   - 允许配置合并优先级
   - 例如：动态发现优先或初始配置优先

4. **变更检测**
   - 通知客户端列表变化
   - 支持订阅变更事件

### 测试建议

1. **单元测试**：
   - 各种边界情况
   - 去重逻辑验证

2. **集成测试**：
   - 与 MCP 系统集成
   - 动态发现场景
