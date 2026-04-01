# prompt.ts 研究文档

## 场景与职责

prompt.ts 是 LSPTool 的提示词定义文件，负责提供工具的静态描述信息。这是一个极简的常量定义模块，用于向 AI 模型说明 LSP 工具的功能和使用方法。

**主要职责：**
- 定义工具的显示名称常量
- 提供详细的工具功能描述（prompt）
- 说明支持的操作类型和参数要求

**使用场景：**
- AI 模型初始化时了解可用工具
- 工具描述动态生成时作为基础文本
- 帮助文档生成

---

## 功能点目的

### 1. 工具名称常量

```typescript
export const LSP_TOOL_NAME = 'LSP' as const
```

使用 `as const` 确保类型为字面量 `'LSP'`，便于类型推断和检查。

### 2. 工具描述文本

```typescript
export const DESCRIPTION = `Interact with Language Server Protocol (LSP) servers to get code intelligence features.

Supported operations:
- goToDefinition: Find where a symbol is defined
- findReferences: Find all references to a symbol
- hover: Get hover information (documentation, type info) for a symbol
- documentSymbol: Get all symbols (functions, classes, variables) in a document
- workspaceSymbol: Search for symbols across the entire workspace
- goToImplementation: Find implementations of an interface or abstract method
- prepareCallHierarchy: Get call hierarchy item at a position (functions/methods)
- incomingCalls: Find all functions/methods that call the function at a position
- outgoingCalls: Find all functions/methods called by the function at a position

All operations require:
- filePath: The file to operate on
- line: The line number (1-based, as shown in editors)
- character: The character offset (1-based, as shown in editors)

Note: LSP servers must be configured for the file type. If no server is available, an error will be returned.`
```

**描述内容结构：**
1. **概述**：说明工具与 LSP 服务器交互获取代码智能功能
2. **支持的操作**：9 种操作及其简要说明
3. **必需参数**：所有操作共用的参数要求
4. **注意事项**：LSP 服务器配置要求

---

## 具体技术实现

### 代码结构

```typescript
// 行 1: 工具名称常量
export const LSP_TOOL_NAME = 'LSP' as const

// 行 3-21: 工具描述
export const DESCRIPTION = `...`
```

### 设计特点

1. **纯常量导出**：无函数、无逻辑，纯数据
2. **TypeScript 类型安全**：使用 `as const` 提供字面量类型
3. **自包含文档**：描述文本完整说明使用方式
4. **多行模板字符串**：使用反引号支持换行，提高可读性

---

## 关键代码路径与文件引用

### 被引用位置

| 文件 | 引用方式 | 用途 |
|------|----------|------|
| `LSPTool.ts` | `import { DESCRIPTION, LSP_TOOL_NAME } from './prompt.js'` | 工具名称和描述 |

### 在 LSPTool.ts 中的使用

```typescript
import { DESCRIPTION, LSP_TOOL_NAME } from './prompt.js'

export const LSPTool = buildTool({
  name: LSP_TOOL_NAME,  // 'LSP'
  
  async description() {
    return DESCRIPTION  // 完整描述文本
  },
  
  async prompt() {
    return DESCRIPTION  // 同上
  },
  // ...
})
```

---

## 依赖与外部交互

### 无外部依赖

这是一个零依赖的纯常量模块：
- 无 npm 包导入
- 无本地模块导入
- 无 Node.js 内置模块使用

### 导出项

| 导出项 | 类型 | 值 |
|--------|------|-----|
| `LSP_TOOL_NAME` | `const` | `'LSP'` |
| `DESCRIPTION` | `const` | 多行描述字符串 |

---

## 风险、边界与改进建议

### 风险分析

1. **描述过时风险**
   - 当添加新操作或修改参数时，需要同步更新描述
   - 建议：在添加新操作时检查并更新此文件

2. **国际化缺失**
   - 当前为英文描述
   - 如需多语言支持，需要重构为可配置结构

### 改进建议

1. **动态描述生成**
   - 当前：静态文本
   - 建议：可考虑从 schemas.ts 动态生成操作列表，避免重复维护

   ```typescript
   // 示例改进
   import { LSP_OPERATIONS } from './schemas.js'
   
   export const DESCRIPTION = generateDescription(LSP_OPERATIONS)
   ```

2. **参数详细说明**
   - 当前：仅说明必需参数
   - 建议：可为每个操作添加特定参数说明

3. **使用示例**
   - 当前：无示例
   - 建议：添加典型使用示例，帮助模型理解

   ```typescript
   export const DESCRIPTION = `...

Examples:
- Find definition: { operation: "goToDefinition", filePath: "src/main.ts", line: 10, character: 5 }
- Find references: { operation: "findReferences", filePath: "src/main.ts", line: 10, character: 5 }
`
   ```

4. **版本信息**
   - 建议：添加支持的 LSP 协议版本说明

### 维护检查清单

当修改 LSP 工具时，检查此文件是否需要更新：

- [ ] 添加新操作类型
- [ ] 修改操作名称
- [ ] 添加/删除必需参数
- [ ] 修改坐标系统（1-based vs 0-based）
- [ ] 修改 LSP 服务器配置方式

### 文件大小

- 原始大小：约 1.1 KB
- 行数：21 行
- 属于极小的配置/常量模块
