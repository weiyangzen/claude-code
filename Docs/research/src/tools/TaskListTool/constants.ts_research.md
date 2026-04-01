# constants.ts 研究文档

## 场景与职责

`constants.ts` 是 TaskListTool 的常量定义文件，职责单一且明确：**定义 TaskList 工具的名称常量**。遵循 Claude Code 工具模块的命名规范，将工具名称硬编码集中管理，便于在多个模块间共享和引用。

## 功能点目的

1. **工具名称集中定义**：提供单一可信源（Single Source of Truth）用于 TaskList 工具名称
2. **跨模块共享**：允许其他模块导入使用，避免魔法字符串分散在代码各处
3. **类型安全**：TypeScript 常量提供编译时检查和自动补全

## 具体技术实现

### 代码实现

```typescript
export const TASK_LIST_TOOL_NAME = 'TaskList'
```

### 设计特点

- **简单导出**：使用命名导出（named export），便于 tree-shaking
- **大写蛇形命名**：符合常量命名惯例
- **字符串值**：`'TaskList'` 采用 PascalCase，符合工具命名规范

## 依赖与外部交互

### 被引用位置

| 文件 | 用途 |
|------|------|
| `src/tools/TaskListTool/TaskListTool.ts` | 工具定义中的 `name` 属性 |
| `src/constants/tools.ts` | 导入用于工具白名单定义 |
| `src/utils/swarm/inProcessRunner.ts` | Teammate 工具白名单注入 |
| `src/utils/permissions/classifierDecision.ts` | YOLO 自动模式白名单 |

### 引用代码示例

**TaskListTool.ts**（行 10）：
```typescript
import { TASK_LIST_TOOL_NAME } from './constants'

export const TaskListTool = buildTool({
  name: TASK_LIST_TOOL_NAME,
  // ...
})
```

**constants/tools.ts**（行 23）：
```typescript
import { TASK_LIST_TOOL_NAME } from '../tools/TaskListTool/constants'

export const IN_PROCESS_TEAMMATE_ALLOWED_TOOLS = new Set([
  TASK_CREATE_TOOL_NAME,
  TASK_GET_TOOL_NAME,
  TASK_LIST_TOOL_NAME,  //  teammate 允许使用
  TASK_UPDATE_TOOL_NAME,
  // ...
])
```

**classifierDecision.ts**（行 14, 71）：
```typescript
import { TASK_LIST_TOOL_NAME } from '../../tools/TaskListTool/constants'

const SAFE_YOLO_ALLOWLISTED_TOOLS = new Set([
  // ...
  TASK_LIST_TOOL_NAME,  // YOLO 模式免分类器检查
  // ...
])
```

## 风险、边界与改进建议

### 风险点

1. **无版本控制**：如果工具名称变更，需要同步更新所有引用处
2. **命名冲突**：`'TaskList'` 作为字符串值，与其他工具名称无命名空间隔离

### 边界情况

- 无特殊边界情况，纯常量定义

### 改进建议

1. **添加类型约束**（可选）：
   ```typescript
   import type { ToolName } from '../../Tool.js'
   export const TASK_LIST_TOOL_NAME: ToolName = 'TaskList'
   ```

2. **添加文档注释**：
   ```typescript
   /**
    * TaskList 工具名称 - 用于列出任务列表中的所有任务
    * @see src/tools/TaskListTool/TaskListTool.ts
    */
   export const TASK_LIST_TOOL_NAME = 'TaskList'
   ```

3. **考虑统一常量文件**：如果工具数量增长，可考虑将相关工具常量合并到统一文件，减少文件数量

### 相关文件引用

- 常量定义：`src/tools/TaskListTool/constants.ts`
- 工具实现：`src/tools/TaskListTool/TaskListTool.ts`
- 工具白名单：`src/constants/tools.ts`
- 权限分类器：`src/utils/permissions/classifierDecision.ts`
- Teammate 运行器：`src/utils/swarm/inProcessRunner.ts`
