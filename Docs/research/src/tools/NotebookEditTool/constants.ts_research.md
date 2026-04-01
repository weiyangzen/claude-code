# constants.ts 研究文档

## 场景与职责

constants.ts 是 NotebookEditTool 模块的常量定义文件。该文件采用极简设计，仅包含工具名称常量，用于避免循环依赖问题。这种设计模式在 Claude Code 的工具模块中较为常见，将名称常量独立出来供其他模块引用，而不必导入完整的工具实现。

## 功能点目的

### 1. 循环依赖避免
- 将 `NOTEBOOK_EDIT_TOOL_NAME` 独立定义，避免其他模块导入完整的 NotebookEditTool.ts
- 其他模块（如权限系统、日志系统）仅需引用工具名称时，无需加载工具完整实现

### 2. 名称集中管理
- 提供单一可信来源（Single Source of Truth）用于工具名称
- 避免魔法字符串分散在代码库各处

## 具体技术实现

### 代码内容
```typescript
// In its own file to avoid circular dependencies
export const NOTEBOOK_EDIT_TOOL_NAME = 'NotebookEdit'
```

### 设计说明
- **文件注释**：明确说明分离目的——避免循环依赖
- **导出方式**：命名导出（named export），便于 tree-shaking
- **命名规范**：全大写 SNAKE_CASE，符合常量命名惯例

## 关键代码路径与文件引用

### 被引用位置

| 文件路径 | 用途 |
|---------|------|
| `src/tools/NotebookEditTool/NotebookEditTool.ts` | 工具名称定义源 |
| 权限系统（推测） | 工具权限规则匹配 |
| 日志/分析系统（推测） | 工具使用事件追踪 |

### 引用方式
```typescript
// NotebookEditTool.ts 中的使用
import { NOTEBOOK_EDIT_TOOL_NAME } from './constants.js'

export const NotebookEditTool = buildTool({
  name: NOTEBOOK_EDIT_TOOL_NAME,
  // ...
})
```

## 依赖与外部交互

### 无外部依赖
- 该文件为零依赖纯常量定义
- 不导入任何其他模块

### 与 NotebookEditTool.ts 的关系
```
constants.ts ──exported──> NOTEBOOK_EDIT_TOOL_NAME
                              │
                              └──imported──> NotebookEditTool.ts
```

## 风险、边界与改进建议

### 风险评估

| 风险项 | 等级 | 说明 |
|-------|------|------|
| 名称变更影响 | 低 | 名称变更需同步更新所有引用点，但 TypeScript 会捕获错误 |
| 循环依赖 | 已解决 | 当前设计正是为避免此问题 |
| 命名冲突 | 极低 | 模块级导出，命名空间隔离 |

### 改进建议

1. **添加工具标识元数据**
   ```typescript
   export const NOTEBOOK_EDIT_TOOL_NAME = 'NotebookEdit'
   export const NOTEBOOK_EDIT_TOOL_ID = 'notebook_edit' // 用于日志/分析
   export const NOTEBOOK_EDIT_TOOL_VERSION = '1.0.0'    // 版本追踪
   ```

2. **添加文件扩展名常量**
   ```typescript
   export const NOTEBOOK_EDIT_SUPPORTED_EXTENSIONS = ['.ipynb'] as const
   ```

3. **添加默认配置常量**
   ```typescript
   export const NOTEBOOK_EDIT_DEFAULTS = {
     EDIT_MODE: 'replace' as const,
     CELL_TYPE: 'code' as const,
     INDENT: 1,  // JSON 缩进
   }
   ```

4. **JSDoc 文档增强**
   ```typescript
   /**
    * NotebookEdit 工具的标识名称
    * @see src/tools/NotebookEditTool/NotebookEditTool.ts
    * @usedBy PermissionSystem, Logging, Analytics
    */
   export const NOTEBOOK_EDIT_TOOL_NAME = 'NotebookEdit'
   ```

### 设计模式对比

与其他工具常量变量的对比：

| 工具 | 常量文件内容 | 设计一致性 |
|-----|-------------|-----------|
| NotebookEditTool | 仅名称 | 基准 |
| FileEditTool | 名称 + 模式常量 | 扩展 |
| BashTool | 名称 + 工具名常量 | 扩展 |

建议保持当前极简设计，除非有明确的跨模块共享需求再扩展。
