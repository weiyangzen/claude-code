# toolPool.ts 研究文档

## 场景与职责

`toolPool.ts` 是 Claude Code CLI 的工具池（Tool Pool）管理模块，负责在 Coordinator 模式下过滤和合并工具集合。该模块是 Agent 协调器架构的关键组件，确保在协调器模式下只暴露允许的工具给子 Agent。

主要使用场景：
1. **Coordinator 模式工具过滤**：在协调器模式下限制可用工具集合
2. **工具池合并**：合并初始工具集和动态组装的工具集
3. **MCP 工具特殊处理**：PR 活动订阅工具在协调器模式下始终允许

## 功能点目的

### 1. Coordinator 模式工具过滤
- **背景**：Coordinator 模式是一种特殊的 Agent 执行模式，协调器 Agent 负责任务分配，Worker Agent 负责执行
- **需求**：协调器应该只能使用特定的协调类工具，不能使用所有工具
- **实现**：通过 `COORDINATOR_MODE_ALLOWED_TOOLS` 常量定义允许的工具白名单

### 2. 工具池合并与去重
- **场景**：初始工具（built-in + startup MCP）与动态组装的工具（来自 `assembleToolPool`）可能有重叠
- **处理**：使用 `uniqBy` 去重，`initialTools` 优先
- **排序**：内置工具在前（用于 prompt cache），MCP 工具在后，各自按名称排序

### 3. 死代码消除优化
- **技术**：使用 Bun 的 `feature()` 进行条件导入
- **效果**：非 Coordinator 模式构建时，`coordinatorModeModule` 代码被完全移除

## 具体技术实现

### 核心常量

```typescript
// MCP 工具名称后缀（PR 活动订阅）
const PR_ACTIVITY_TOOL_SUFFIXES = [
  'subscribe_pr_activity',
  'unsubscribe_pr_activity',
]

// 死代码消除：条件导入
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('../coordinator/coordinatorMode.js')
  : null
```

### 核心函数

#### `isPrActivitySubscriptionTool`

```typescript
export function isPrActivitySubscriptionTool(name: string): boolean {
  return PR_ACTIVITY_TOOL_SUFFIXES.some(suffix => name.endsWith(suffix))
}
```

**设计说明**：
- 使用后缀匹配而非完整名称匹配
- 原因：MCP 服务器名称前缀可能变化（如 `github_`, `gitlab_`）
- 这些工具是轻量级编排操作，协调器可以直接调用

#### `applyCoordinatorToolFilter`

```typescript
export function applyCoordinatorToolFilter(tools: Tools): Tools {
  return tools.filter(
    t =>
      COORDINATOR_MODE_ALLOWED_TOOLS.has(t.name) ||
      isPrActivitySubscriptionTool(t.name),
  )
}
```

**过滤逻辑**：
1. 检查工具名是否在 `COORDINATOR_MODE_ALLOWED_TOOLS` 集合中
2. 或者是 PR 活动订阅工具

#### `mergeAndFilterTools`

```typescript
export function mergeAndFilterTools(
  initialTools: Tools,      // 初始工具（built-in + startup MCP）
  assembled: Tools,         // 组装工具（来自 assembleToolPool）
  mode: ToolPermissionContext['mode'],  // 权限模式
): Tools
```

**算法流程**：

1. **合并与去重**
   ```typescript
   const [mcp, builtIn] = partition(
     uniqBy([...initialTools, ...assembled], 'name'),
     isMcpTool,
   )
   ```
   - `uniqBy([...initialTools, ...assembled], 'name')`：`initialTools` 在前，所以优先
   - `partition(isMcpTool)`：分离内置工具和 MCP 工具

2. **排序**
   ```typescript
   const byName = (a: Tool, b: Tool) => a.name.localeCompare(b.name)
   const tools = [...builtIn.sort(byName), ...mcp.sort(byName)]
   ```
   - 内置工具按名称排序在前
   - MCP 工具按名称排序在后
   - **原因**：服务器的 prompt cache 策略要求内置工具保持连续前缀

3. **Coordinator 模式过滤**
   ```typescript
   if (feature('COORDINATOR_MODE') && coordinatorModeModule) {
     if (coordinatorModeModule.isCoordinatorMode()) {
       return applyCoordinatorToolFilter(tools)
     }
   }
   ```
   - 构建时检查：代码是否包含 Coordinator 模式
   - 运行时检查：当前是否处于 Coordinator 模式

## 关键代码路径与文件引用

### 调用方（被谁使用）

| 文件路径 | 使用场景 |
|---------|---------|
| `src/cli/print.ts` | CLI 打印输出时的工具过滤 |
| `src/hooks/useMergedTools.ts` | React Hook 中的工具合并 |
| `src/screens/REPL.tsx` | REPL 主界面的工具管理 |
| `src/main.tsx` | 主入口的工具初始化 |

### 依赖模块

| 模块 | 用途 |
|-----|------|
| `bun:bundle` (feature) | 构建时功能开关 |
| `lodash-es/partition.js` | 工具数组分区 |
| `lodash-es/uniqBy.js` | 按名称去重 |
| `../constants/tools.js` | `COORDINATOR_MODE_ALLOWED_TOOLS` |
| `../services/mcp/utils.js` | `isMcpTool` 检测 |
| `../Tool.js` | `Tool`, `Tools`, `ToolPermissionContext` 类型 |

## 依赖与外部交互

### 与 Coordinator 模式的集成

```typescript
// coordinatorMode.ts 伪代码
export function isCoordinatorMode(): boolean {
  return process.env.CLAUDE_CODE_COORDINATOR_MODE === 'true'
}
```

**激活方式**：通过环境变量 `CLAUDE_CODE_COORDINATOR_MODE` 激活协调器模式。

### 与 Tool 类型的集成

```typescript
// Tool.ts 中的相关类型
export type Tool = {
  name: string
  // ... 其他属性
}

export type Tools = Tool[]

export type ToolPermissionContext = {
  mode: 'normal' | 'coordinator' | 'worker'
  // ...
}
```

### 与 MCP 的集成

```typescript
// MCP 工具检测
import { isMcpTool } from '../services/mcp/utils.js'

// MCP 工具特点：
// - 名称通常包含服务器前缀（如 `github_read_file`）
// - 需要额外的权限检查
// - 在 prompt 中排在内置工具之后（cache 策略）
```

### 与 Prompt Cache 的集成

**排序的重要性**：
```typescript
const tools = [...builtIn.sort(byName), ...mcp.sort(byName)]
```

- Anthropic API 的 prompt cache 机制对工具定义有优化
- 内置工具保持连续前缀可以提高缓存命中率
- MCP 工具变化频繁，排在后面避免影响内置工具的缓存

## 风险、边界与改进建议

### 潜在风险

1. **Coordinator 模式硬编码**
   - `COORDINATOR_MODE_ALLOWED_TOOLS` 是编译时常量
   - 新增协调器工具需要修改代码并重新构建

2. **工具名称冲突**
   - `uniqBy` 使用 `initialTools` 优先
   - 如果 `assembled` 中有同名但不同实现的工具，会被静默覆盖

3. **功能开关耦合**
   ```typescript
   if (feature('COORDINATOR_MODE') && coordinatorModeModule)
   ```
   - 两个条件都需要满足，逻辑复杂
   - 如果 `feature()` 返回 true 但模块加载失败，行为不确定

4. **PR 活动工具后缀假设**
   - 假设所有 PR 活动工具都以特定后缀结尾
   - 如果 MCP 服务器使用不同命名约定，可能无法识别

### 边界条件

| 场景 | 行为 |
|-----|------|
| `initialTools` 和 `assembled` 都为空 | 返回空数组 |
| 同名工具冲突 | `initialTools` 中的版本优先 |
| 无内置工具 | 只有 MCP 工具，按名称排序 |
| 无 MCP 工具 | 只有内置工具，按名称排序 |
| Coordinator 模式未编译 | 跳过过滤，返回全部工具 |
| Coordinator 模式编译但未激活 | 返回全部工具 |
| Coordinator 模式激活 | 只返回白名单中的工具 |

### 改进建议

1. **配置化白名单**
   ```typescript
   // 从设置或环境变量读取
   const allowedTools = getCoordinatorAllowedTools() || 
     COORDINATOR_MODE_ALLOWED_TOOLS
   ```

2. **工具冲突警告**
   ```typescript
   function mergeToolsWithWarning(initial: Tools, assembled: Tools): Tools {
     const initialNames = new Set(initial.map(t => t.name))
     const conflicts = assembled.filter(t => initialNames.has(t.name))
     if (conflicts.length > 0) {
       console.warn(`Tool conflicts detected: ${conflicts.map(t => t.name).join(', ')}`)
     }
     return uniqBy([...initial, ...assembled], 'name')
   }
   ```

3. **更灵活的 PR 活动检测**
   ```typescript
   const PR_ACTIVITY_TOOL_PATTERNS = [
     /subscribe_pr_activity$/,
     /unsubscribe_pr_activity$/,
     /pr_activity_subscribe$/,  // 支持不同命名风格
     /pr_activity_unsubscribe$/,
   ]
   ```

4. **工具元数据标记**
   ```typescript
   export type Tool = {
     name: string
     metadata?: {
       category: 'builtin' | 'mcp' | 'coordinator'
       allowInCoordinatorMode?: boolean
     }
   }
   ```

5. **运行时工具过滤**
   ```typescript
   export function createToolFilter(
     predicate: (tool: Tool) => boolean
   ): (tools: Tools) => Tools {
     return tools => tools.filter(predicate)
   }
   ```

6. **工具池缓存**
   ```typescript
   // 缓存合并结果（工具集合不变时）
   const toolPoolCache = new Map<string, Tools>()
   
   export function mergeAndFilterTools(
     initialTools: Tools,
     assembled: Tools,
     mode: ToolPermissionContext['mode'],
   ): Tools {
     const cacheKey = computeHash(initialTools, assembled, mode)
     if (toolPoolCache.has(cacheKey)) {
       return toolPoolCache.get(cacheKey)!
     }
     const result = computeMergedTools(initialTools, assembled, mode)
     toolPoolCache.set(cacheKey, result)
     return result
   }
   ```

### 测试建议

应覆盖以下场景：
- 正常模式下的工具合并
- Coordinator 模式下的工具过滤
- 工具名称冲突处理
- 空工具集处理
- PR 活动工具识别（各种命名风格）
- 功能开关编译/未编译的行为差异
- 工具排序（内置 vs MCP）
