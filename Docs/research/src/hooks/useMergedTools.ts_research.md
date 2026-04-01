# useMergedTools.ts 深度研究文档

## 场景与职责

`useMergedTools` 是一个用于组装和合并工具池（Tool Pool）的 React 钩子。它是 REPL 和 Agent 运行时的核心工具管理组件，负责组合内置工具、MCP 工具，并应用权限过滤和协调模式过滤。

### 核心场景

1. **工具池组装**：组合内置工具和 MCP 工具
2. **权限过滤**：根据权限上下文过滤工具
3. **协调模式过滤**：在协调器模式下限制可用工具
4. **去重和排序**：确保工具列表唯一且有序

### 与其他组件的关系

- 被 `REPL.tsx` 使用，获取可用工具列表
- 与 `assembleToolPool` 配合，共享工具组装逻辑
- 与 `mergeAndFilterTools` 配合，应用额外过滤

---

## 功能点目的

### 1. 工具池组装

使用 `assembleToolPool` 函数组装工具池：
- 获取内置工具
- 添加 MCP 工具
- 应用 deny 规则过滤
- 去重处理

### 2. 权限过滤

根据 `toolPermissionContext` 过滤工具：
- 应用权限规则
- 根据模式限制工具可用性

### 3. 协调器模式过滤

当启用 `COORDINATOR_MODE` 时：
- 只允许特定的协调工具
- 过滤掉非必要的工具

### 4. 工具合并

使用 `mergeAndFilterTools` 合并：
- `initialTools`: 初始工具（内置 + 启动 MCP）
- `assembled`: 组装的工具池
- 应用去重和排序

---

## 具体技术实现

### 关键数据结构

```typescript
import type { Tools, ToolPermissionContext } from '../Tool.js'

// 工具定义（简化）
interface Tool {
  name: string
  description: string
  parameters: object
  execute: Function
}

type Tools = Tool[]

interface ToolPermissionContext {
  mode: 'normal' | 'plan' | 'coordinator' | ...
  // 其他权限相关属性...
}
```

### 核心实现

```typescript
export function useMergedTools(
  initialTools: Tools,
  mcpTools: Tools,
  toolPermissionContext: ToolPermissionContext,
): Tools {
  let replBridgeEnabled = false
  let replBridgeOutboundOnly = false
  
  return useMemo(() => {
    // 组装工具池（共享逻辑）
    const assembled = assembleToolPool(toolPermissionContext, mcpTools)
    
    // 合并并过滤
    return mergeAndFilterTools(
      initialTools,
      assembled,
      toolPermissionContext.mode,
    )
  }, [
    initialTools,
    mcpTools,
    toolPermissionContext,
    replBridgeEnabled,
    replBridgeOutboundOnly,
  ])
}
```

### 依赖函数

#### `assembleToolPool`

位于 `src/tools.ts`：

```typescript
export function assembleToolPool(
  toolPermissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  // 1. 获取内置工具
  const builtInTools = getTools()
  
  // 2. 应用 deny 规则过滤 MCP 工具
  const filteredMcpTools = mcpTools.filter(tool => 
    !isToolDenied(tool, toolPermissionContext)
  )
  
  // 3. 合并并去重
  const allTools = uniqBy([...builtInTools, ...filteredMcpTools], 'name')
  
  // 4. 排序：内置工具在前（提示缓存稳定性）
  const [mcp, builtIn] = partition(allTools, isMcpTool)
  const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
  return [...builtIn.sort(byName), ...mcp.sort(byName)]
}
```

#### `mergeAndFilterTools`

位于 `src/utils/toolPool.ts`：

```typescript
export function mergeAndFilterTools(
  initialTools: Tools,
  assembled: Tools,
  mode: ToolPermissionContext['mode'],
): Tools {
  // 1. 合并 initialTools（优先）
  const [mcp, builtIn] = partition(
    uniqBy([...initialTools, ...assembled], 'name'),
    isMcpTool
  )
  const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
  const tools = [...builtIn.sort(byName), ...mcp.sort(byName)]
  
  // 2. 应用协调器模式过滤
  if (feature('COORDINATOR_MODE') && isCoordinatorMode()) {
    return applyCoordinatorToolFilter(tools)
  }
  
  return tools
}
```

---

## 关键代码路径与文件引用

```
src/hooks/useMergedTools.ts
├── Tools 类型                     # 来自 Tool.js
├── ToolPermissionContext 类型     # 来自 Tool.js
├── useMergedTools()               # 行 20-44: 主钩子
│   └── useMemo                    # 行 27-43: 缓存组装结果
```

### 依赖文件

```
src/tools.ts
├── assembleToolPool()             # 工具池组装
├── getTools()                     # 获取内置工具
└── isToolDenied()                 # 检查工具是否被拒绝

src/utils/toolPool.ts
├── mergeAndFilterTools()          # 合并和过滤
├── applyCoordinatorToolFilter()   # 协调器模式过滤
└── isPrActivitySubscriptionTool() # PR 活动工具检查

src/Tool.ts
├── Tools 类型
├── ToolPermissionContext 类型
└── Tool 接口

src/services/mcp/utils.ts
└── isMcpTool()                    # MCP 工具检查

src/constants/tools.ts
└── COORDINATOR_MODE_ALLOWED_TOOLS # 协调器模式允许的工具
```

---

## 依赖与外部交互

### React Hooks 使用

- `useMemo`: 缓存组装结果

### 外部依赖

```typescript
import { assembleToolPool } from '../tools.js'
import { mergeAndFilterTools } from '../utils/toolPool.js'
```

### 特性标志

```typescript
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('../coordinator/coordinatorMode.js')
  : null
```

---

## 风险、边界与改进建议

### 已知风险

1. **工具冲突**
   - 同名工具可能导致意外行为
   - 去重保留前者可能不是期望行为

2. **性能问题**
   - 每次权限上下文变化都重新组装
   - 大工具集时可能影响性能

3. **协调器模式硬编码**
   - 允许的工具列表是硬编码的
   - 需要代码变更才能扩展

### 边界情况

| 场景 | 行为 |
|-----|------|
| initialTools=[] | 只返回 assembled 工具 |
| mcpTools=[] | 只返回内置工具 |
| 工具名冲突 | 保留 initialTools 中的 |
| 协调器模式 | 过滤到允许的工具子集 |

### 改进建议

1. **工具命名空间**
   - 为 MCP 工具添加命名空间
   - 避免与内置工具冲突

2. **动态协调器配置**
   - 允许配置协调器模式允许的工具
   - 不硬编码工具列表

3. **工具缓存**
   - 缓存工具组装结果
   - 只在必要时重新组装

4. **工具依赖**
   - 支持工具依赖声明
   - 自动包含依赖工具

5. **工具版本**
   - 支持工具版本管理
   - 处理版本冲突

### 测试建议

1. **单元测试**：
   - 工具组装逻辑
   - 过滤规则
   - 去重行为

2. **集成测试**：
   - 与 MCP 系统集成
   - 协调器模式场景

3. **性能测试**：
   - 大工具集组装性能
   - 频繁更新场景
