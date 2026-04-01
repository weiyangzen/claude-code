# useMergedCommands.ts 深度研究文档

## 场景与职责

`useMergedCommands` 是一个用于合并命令列表的工具钩子。它将初始命令列表与 MCP（Model Context Protocol）提供的命令合并，确保去重并缓存结果。

### 核心场景

1. **命令合并**：合并内置命令和 MCP 提供的命令
2. **去重处理**：基于 `name` 字段去重
3. **性能优化**：使用 `useMemo` 缓存合并结果

### 与其他组件的关系

- 被 `REPL.tsx` 使用，管理可用命令列表
- 与 `useMergedClients` 和 `useMergedTools` 形成工具函数家族
- 简单的工具函数包装，无复杂业务逻辑

---

## 功能点目的

### 1. 命令合并

合并两个命令列表：
- `initialCommands`: 内置命令和初始配置
- `mcpCommands`: MCP 服务器提供的命令

### 2. 去重策略

使用 `lodash-es/uniqBy` 基于 `name` 字段去重：
- 如果两个列表有同名命令，保留 `initialCommands` 中的（因为它在前面）
- 确保每个命令只出现一次

### 3. 缓存优化

使用 `useMemo` 缓存合并结果：
- 只有当 `initialCommands` 或 `mcpCommands` 变化时才重新计算
- 避免不必要的重渲染

---

## 具体技术实现

### 关键数据结构

```typescript
import type { Command } from '../commands.js'

// Command 定义（简化）
interface Command {
  name: string           // 命令名称（唯一标识）
  description?: string   // 命令描述
  handler: Function      // 命令处理函数
  // 其他属性...
}
```

### 核心实现

```typescript
export function useMergedCommands(
  initialCommands: Command[],
  mcpCommands: Command[],
): Command[] {
  return useMemo(() => {
    if (mcpCommands.length > 0) {
      return uniqBy([...initialCommands, ...mcpCommands], 'name')
    }
    return initialCommands
  }, [initialCommands, mcpCommands])
}
```

### 合并逻辑

```
initialCommands: [/help, /clear]
mcpCommands: [/clear, /mcp-tool]
          ↓ useMergedCommands
result: [/help, /clear, /mcp-tool]  // /clear 来自 initialCommands
```

---

## 关键代码路径与文件引用

```
src/hooks/useMergedCommands.ts
├── Command 类型                   # 来自 commands.js
└── useMergedCommands()            # 行 5-15: 主钩子
    └── useMemo                    # 行 9-14: 缓存合并结果
```

### 依赖文件

```
src/commands.ts
├── Command 类型定义
└── builtInCommandNames            # 内置命令名列表

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
import type { Command } from '../commands.js'
```

---

## 风险、边界与改进建议

### 已知风险

1. **命令冲突**
   - MCP 命令可能覆盖内置命令
   - 虽然去重保留前者，但可能不是期望行为

2. **无验证**
   - 不验证命令处理函数的有效性
   - 可能包含无效命令

3. **命名空间**
   - 无命名空间隔离
   - 不同来源的命令可能意外冲突

### 边界情况

| 场景 | 行为 |
|-----|------|
| mcpCommands=[] | 返回 initialCommands |
| initialCommands=[] | 返回 mcpCommands（去重后）|
| 同名命令 | 保留 initialCommands 中的 |
| 两者都为空 | 返回 [] |

### 改进建议

1. **命名空间**
   - 为 MCP 命令添加命名空间前缀
   - 例如：`mcp:server-name:command`

2. **冲突检测**
   - 检测命令冲突并警告用户
   - 提供解决冲突的选项

3. **命令验证**
   - 验证命令处理函数
   - 确保命令可执行

4. **优先级配置**
   - 允许配置命令优先级
   - 支持用户覆盖默认行为

### 测试建议

1. **单元测试**：
   - 各种边界情况
   - 去重逻辑验证

2. **集成测试**：
   - 与 MCP 系统集成
   - 命令执行测试
