# constants.ts 研究文档

## 场景与职责

`constants.ts` 是 `TodoWriteTool` 模块的常量定义文件，仅包含一个导出常量：`TODO_WRITE_TOOL_NAME`。该文件遵循项目中工具模块的命名规范，将工具名称集中定义以便在多个文件中统一引用。

## 功能点目的

### 1. 工具名称集中定义
- 提供单一可信来源（Single Source of Truth）用于 TodoWrite 工具名称
- 避免在代码中硬编码字符串导致的拼写错误或不一致

### 2. 跨模块引用
- 被 `TodoWriteTool.ts` 导入用于工具定义
- 被 `sessionRestore.ts` 导入用于从 transcript 中识别 TodoWrite 工具调用

## 具体技术实现

### 代码内容

```typescript
export const TODO_WRITE_TOOL_NAME = 'TodoWrite'
```

### 设计模式

这是典型的**常量提取模式**（Constants Extraction Pattern）：
- 将魔术字符串（Magic String）提取为命名常量
- 便于 IDE 自动完成和重构
- 类型安全（TypeScript 会在拼写错误时提示）

## 关键代码路径与文件引用

### 本文件

| 行号 | 代码 | 说明 |
|------|------|------|
| 1 | `TODO_WRITE_TOOL_NAME` | 工具名称常量定义 |

### 引用方

| 文件路径 | 用途 |
|----------|------|
| `src/tools/TodoWriteTool/TodoWriteTool.ts` | 工具定义中使用 |
| `src/utils/sessionRestore.ts` | 从 transcript 提取 todos 时识别工具名称 |

## 依赖与外部交互

- **无外部依赖** - 纯常量定义
- **无运行时交互** - 编译时常量

## 风险、边界与改进建议

### 风险

1. **命名一致性**
   - 常量值为 `'TodoWrite'`，但文件名和目录使用 `TodoWriteTool`
   - 这种不一致可能导致混淆
   - 建议：保持命名一致性，或添加注释说明历史原因

2. **V2 迁移风险**
   - 随着 Task V2 系统的推广，此常量可能逐渐失去用途
   - 但 `sessionRestore.ts` 仍需它来恢复旧会话

### 改进建议

1. **添加 JSDoc 注释**
   ```typescript
   /**
    * Tool name for the legacy TodoWrite tool (V1).
    * Used for session restore from transcript.
    * @deprecated Task V2 system uses different tool names.
    */
   export const TODO_WRITE_TOOL_NAME = 'TodoWrite'
   ```

2. **考虑合并**
   - 如果此常量仅被少数文件使用，可考虑直接内联
   - 但当前跨模块引用使其有独立存在的价值

3. **类型安全增强**
   - 可考虑使用 const assertion 或字面量类型
   ```typescript
   export const TODO_WRITE_TOOL_NAME = 'TodoWrite' as const
   ```
