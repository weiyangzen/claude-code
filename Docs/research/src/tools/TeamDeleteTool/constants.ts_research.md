# constants.ts 研究文档

## 场景与职责

`constants.ts` 是 `TeamDeleteTool` 的常量定义文件，遵循 Claude Code CLI 的工具模块架构模式。该文件负责集中管理工具名称常量，确保在代码库中引用的一致性。

### 核心职责
1. **工具名称定义**：定义 TeamDelete 工具的唯一标识符
2. **跨模块引用**：为其他模块提供类型安全的工具名称引用
3. **避免魔法字符串**：防止工具名称在代码中分散硬编码

## 功能点目的

### 1. 工具名称常量
- 定义 `TEAM_DELETE_TOOL_NAME` 常量
- 值为 `'TeamDelete'`
- 用于工具注册、权限检查、分析日志等场景

### 2. 跨模块一致性
- 被 `TeamDeleteTool.ts` 自身引用
- 被 `coordinatorMode.ts` 用于内部工具集合
- 被 `classifierDecision.ts` 用于安全工具白名单

## 具体技术实现

### 常量定义

```typescript
export const TEAM_DELETE_TOOL_NAME = 'TeamDelete'
```

### 使用模式

```typescript
// 在 TeamDeleteTool.ts 中
import { TEAM_DELETE_TOOL_NAME } from './constants.js'

export const TeamDeleteTool: Tool<InputSchema, Output> = buildTool({
  name: TEAM_DELETE_TOOL_NAME,
  // ...
})
```

```typescript
// 在 coordinatorMode.ts 中
import { TEAM_DELETE_TOOL_NAME } from '../tools/TeamDeleteTool/constants.js'

const INTERNAL_WORKER_TOOLS = new Set([
  TEAM_CREATE_TOOL_NAME,
  TEAM_DELETE_TOOL_NAME,
  SEND_MESSAGE_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
])
```

```typescript
// 在 classifierDecision.ts 中
import { TEAM_DELETE_TOOL_NAME } from '../../tools/TeamDeleteTool/constants.js'

const SAFE_YOLO_ALLOWLISTED_TOOLS = new Set([
  // ...
  TEAM_DELETE_TOOL_NAME,
  // ...
])
```

## 关键代码路径与文件引用

### 被引用方

| 路径 | 用途 |
|------|------|
| `./TeamDeleteTool.ts` | 工具定义中的 `name` 属性 |
| `../../coordinator/coordinatorMode.ts` | `INTERNAL_WORKER_TOOLS` 集合 |
| `../../utils/permissions/classifierDecision.ts` | `SAFE_YOLO_ALLOWLISTED_TOOLS` 白名单 |

### 引用关系图

```
constants.ts
  ├─ TeamDeleteTool.ts          (工具定义)
  ├─ coordinatorMode.ts         (协调器模式内部工具集合)
  └─ classifierDecision.ts      (YOLO 分类器安全工具白名单)
```

## 依赖与外部交互

### 无外部依赖

该文件是一个纯常量定义文件，不导入任何外部模块：
- 无 `import` 语句
- 无运行时依赖
- 可在任何上下文中安全导入

### 命名约定

- 使用 `SCREAMING_SNAKE_CASE`（全大写下划线分隔）
- 后缀 `_TOOL_NAME` 明确表示这是工具名称常量
- 前缀 `TEAM_DELETE_` 避免与其他工具名称冲突

## 风险、边界与改进建议

### 风险点

1. **命名变更影响**
   - 修改常量值会影响所有引用位置
   - 需要同步更新相关文档和测试

2. **工具名称冲突**
   - 当前值为 `'TeamDelete'`，需要确保全局唯一
   - 如果未来添加类似工具，命名需要区分

### 边界情况

1. **空值检查**
   - 常量为字符串字面量，不可能为 undefined/null
   - TypeScript 编译时保证类型安全

2. **字符串比较**
   - 使用 `===` 进行精确匹配
   - 大小写敏感（`'TeamDelete' !== 'teamdelete'`）

### 改进建议

1. **添加 JSDoc 注释**
   ```typescript
   /**
    * Tool name for the TeamDelete tool.
    * Used to disband a swarm team and clean up resources.
    * @see TeamDeleteTool
    */
   export const TEAM_DELETE_TOOL_NAME = 'TeamDelete'
   ```

2. **考虑使用枚举**
   ```typescript
   export enum ToolNames {
     TeamDelete = 'TeamDelete',
     TeamCreate = 'TeamCreate',
     // ...
   }
   ```

3. **集中式常量管理**
   - 考虑将所有工具名称常量集中到一个文件
   - 或者使用命名空间模式
   ```typescript
   export const ToolNames = {
     TeamDelete: 'TeamDelete',
     TeamCreate: 'TeamCreate',
   } as const
   ```

4. **运行时验证**
   - 在工具注册时验证名称唯一性
   - 防止重复注册相同名称的工具
