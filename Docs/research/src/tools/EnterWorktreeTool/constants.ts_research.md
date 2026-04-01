# constants.ts 研究文档

## 场景与职责

constants.ts 是 EnterWorktreeTool 的常量定义模块，遵循项目工具模块的组织规范，将工具名称等常量集中管理。

**核心场景：**
1. 提供工具的唯一标识名称
2. 确保工具名称在定义和使用处保持一致
3. 支持工具名称的静态分析和重构

**职责边界：**
- 仅包含常量定义，不包含逻辑
- 作为工具模块的单一事实来源

## 功能点目的

### 1. 工具名称常量
- **目的**：定义 EnterWorktreeTool 的公开名称
- **值**：`'EnterWorktree'`
- **用途**：
  - 工具注册时的标识
  - 模型调用工具时的名称匹配
  - 分析事件中的工具标识

## 具体技术实现

### 代码实现

```typescript
export const ENTER_WORKTREE_TOOL_NAME = 'EnterWorktree'
```

### 设计特点

| 特点 | 说明 |
|-----|------|
| 命名规范 | UPPER_SNAKE_CASE，符合 TypeScript 常量命名惯例 |
| 导出方式 | 命名导出（named export），便于 tree-shaking |
| 类型推断 | TypeScript 自动推断为 `'EnterWorktree'` 字面量类型 |

## 关键代码路径与文件引用

### 当前文件
- `/src/tools/EnterWorktreeTool/constants.ts` - 常量定义

### 使用位置

| 文件路径 | 使用方式 |
|---------|---------|
| `/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts` | `import { ENTER_WORKTREE_TOOL_NAME } from './constants.js'` |

### 使用代码

```typescript
// EnterWorktreeTool.ts
import { ENTER_WORKTREE_TOOL_NAME } from './constants.js'

export const EnterWorktreeTool: Tool<InputSchema, Output> = buildTool({
  name: ENTER_WORKTREE_TOOL_NAME,
  // ...
})
```

## 依赖与外部交互

### 依赖关系

```
constants.ts
└── (无依赖，纯常量定义)
```

### 被依赖关系

```
EnterWorktreeTool.ts
└── constants.ts (ENTER_WORKTREE_TOOL_NAME)
```

## 风险、边界与改进建议

### 已知风险

1. **命名一致性**
   - 工具名称必须与模型训练时的名称保持一致
   - 修改此常量需要同步更新模型配置和文档

2. **命名冲突**
   - 工具名称在全局工具表中必须唯一
   - 建议添加前缀避免冲突（如 `claude_enter_worktree`）

### 边界情况

1. **大小写敏感**
   - 工具名称区分大小写
   - 模型调用时必须完全匹配 `'EnterWorktree'`

2. **字符限制**
   - 工具名称只能包含字母、数字和下划线
   - 当前名称符合要求

### 改进建议

1. **添加类型约束**
   ```typescript
   // 建议：使用类型确保工具名称符合规范
   export const ENTER_WORKTREE_TOOL_NAME: ToolName = 'EnterWorktree'
   ```

2. **添加注释文档**
   ```typescript
   /**
    * Tool name for EnterWorktreeTool.
    * Used when creating isolated git worktrees for parallel development.
    * @see EnterWorktreeTool
    */
   export const ENTER_WORKTREE_TOOL_NAME = 'EnterWorktree'
   ```

3. **考虑命名空间**
   ```typescript
   // 建议：使用命名空间避免全局冲突
   export const ToolNames = {
     EnterWorktree: 'EnterWorktree',
     ExitWorktree: 'ExitWorktree',
   } as const
   ```

4. **版本控制**
   ```typescript
   // 建议：如果工具版本化，可添加版本信息
   export const ENTER_WORKTREE_TOOL_NAME = 'EnterWorktree'
   export const ENTER_WORKTREE_TOOL_VERSION = '1.0.0'
   ```

### 与 ExitWorktreeTool 的关系

ExitWorktreeTool 也有对应的常量定义：

```typescript
// ExitWorktreeTool/constants.ts
export const EXIT_WORKTREE_TOOL_NAME = 'ExitWorktree'
```

两个工具名称形成对称关系：
- EnterWorktree / ExitWorktree
- 便于用户理解和记忆
- 符合工具配对的命名惯例
