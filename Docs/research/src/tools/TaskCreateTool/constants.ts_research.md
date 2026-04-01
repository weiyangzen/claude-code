# constants.ts 研究文档

## 场景与职责

`constants.ts` 是 TaskCreateTool 模块的常量定义文件，负责集中管理 TaskCreate 工具的标识符常量。该文件遵循 Claude Code 工具模块的标准组织结构，将常量定义与业务逻辑分离，便于维护和引用。

## 功能点目的

### 1. 工具名称常量定义
- 定义 `TASK_CREATE_TOOL_NAME` 常量，值为 `'TaskCreate'`
- 作为工具的唯一标识符，在以下场景使用：
  - 工具注册时的名称匹配
  - 钩子系统中的工具识别
  - 日志和遥测事件中的工具标识
  - 自动分类器（auto-classifier）的输入处理

## 具体技术实现

### 代码实现
```typescript
export const TASK_CREATE_TOOL_NAME = 'TaskCreate'
```

### 设计特点
- **单一职责**: 文件仅包含常量定义，无业务逻辑
- **导出方式**: 使用命名导出（named export），便于 tree-shaking
- **命名规范**: 使用全大写蛇形命名（SNAKE_CASE）表示常量

## 关键代码路径与文件引用

### 被引用位置
| 文件 | 用途 |
|------|------|
| `src/tools/TaskCreateTool/TaskCreateTool.ts` | 导入作为 `buildTool` 的 `name` 属性 |

### 引用代码
```typescript
// TaskCreateTool.ts
import { TASK_CREATE_TOOL_NAME } from './constants.js'

export const TaskCreateTool = buildTool({
  name: TASK_CREATE_TOOL_NAME,
  // ...
})
```

## 依赖与外部交互

### 无外部依赖
- 该文件为纯常量定义，不导入任何外部模块
- 无运行时依赖，仅在编译期/静态分析期被使用

### 被依赖关系
- 仅被 `TaskCreateTool.ts` 直接导入
- 工具名称通过 `TaskCreateTool` 间接被整个任务系统使用

## 风险、边界与改进建议

### 潜在风险

1. **命名不一致风险**
   - 风险：如果工具名称需要变更，可能遗漏更新此常量
   - 缓解：工具名称变更应通过全局搜索确保一致性
   - 相关位置：
     - `src/tools/TaskCreateTool/constants.ts` - 常量定义
     - `src/tools/TaskCreateTool/TaskCreateTool.ts` - 工具注册
     - 钩子配置中的工具匹配模式

2. **循环引用风险**
   - 风险：如果常量文件导入其他业务逻辑模块，可能导致循环引用
   - 当前状态：安全，无外部导入

### 边界条件

1. **常量值唯一性**
   - 边界：`'TaskCreate'` 必须在所有工具中唯一
   - 冲突后果：工具注册失败或覆盖其他工具
   - 检查：启动时的工具注册流程会验证名称唯一性

2. **字符串不变性**
   - 边界：JavaScript 中 `const` 只保证引用不变，不保证内容不变
   - 实际：字符串为原始类型，天然不可变

### 改进建议

1. **添加类型约束**
   ```typescript
   import type { ToolName } from '../../types/tools.js'
   export const TASK_CREATE_TOOL_NAME: ToolName = 'TaskCreate'
   ```
   - 优点：编译期检查工具名称格式
   - 依赖：需要定义 `ToolName` 类型

2. **集中式工具名称管理**
   - 建议：将所有工具名称统一在 `src/tools/toolNames.ts` 管理
   - 优点：便于查看所有工具名称，避免冲突
   - 权衡：与当前分散式组织方式冲突

3. **添加 JSDoc 注释**
   ```typescript
   /**
    * TaskCreate 工具的标识符名称
    * @internal 仅在工具注册和钩子匹配时使用
    */
   export const TASK_CREATE_TOOL_NAME = 'TaskCreate'
   ```
   - 优点：提供 IDE 提示和文档生成支持

4. **工具名称版本控制**
   - 建议：如果工具名称变更，保留别名支持向后兼容
   - 实现：在 `buildTool` 中使用 `aliases` 属性
   ```typescript
   buildTool({
     name: TASK_CREATE_TOOL_NAME,
     aliases: ['LegacyTaskCreateName'], // 向后兼容
     // ...
   })
   ```

### 相关工具名称参考

Claude Code 中其他任务相关工具的命名：
- `TaskCreate` - 创建任务（当前文件）
- `TaskUpdate` - 更新任务状态/属性
- `TaskList` - 列出所有任务
- `TaskComplete` - 标记任务完成

命名规范：动词 + 名词，PascalCase 格式
