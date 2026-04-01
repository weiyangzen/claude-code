# prompt.ts 研究文档

## 场景与职责

`prompt.ts` 是 `ListMcpResourcesTool` 的**配置与提示信息模块**，负责定义工具的元数据、描述信息和提示模板。它是工具与模型交互的"说明书"，帮助模型理解工具的用途和调用方式。

### 核心职责

1. **工具标识**：定义工具的唯一名称常量
2. **功能描述**：提供工具用途的详细说明（`DESCRIPTION`）
3. **使用提示**：为模型提供调用指导（`PROMPT`）
4. **文档生成**：作为工具文档的单一事实来源

### 使用场景

- 模型决策时了解工具能力
- 自动生成工具文档
- 帮助用户理解工具用途

---

## 功能点目的

### 1. LIST_MCP_RESOURCES_TOOL_NAME - 工具标识

```typescript
export const LIST_MCP_RESOURCES_TOOL_NAME = 'ListMcpResourcesTool'
```

**目的**：
- 提供类型安全的工具名称引用
- 避免在多处硬编码字符串导致的维护问题
- 支持 IDE 自动补全和重构

**使用位置**：
```typescript
// ListMcpResourcesTool.ts
import { LIST_MCP_RESOURCES_TOOL_NAME } from './prompt.js'

buildTool({
  name: LIST_MCP_RESOURCES_TOOL_NAME,
  // ...
})
```

### 2. DESCRIPTION - 工具描述

```typescript
export const DESCRIPTION = `
Lists available resources from configured MCP servers.
Each resource object includes a 'server' field indicating which server it's from.

Usage examples:
- List all resources from all servers: \`listMcpResources\`
- List resources from a specific server: \`listMcpResources({ server: "myserver" })\`
`
```

**目的**：
- 向模型说明工具的核心功能
- 提供使用示例帮助模型理解调用方式
- 强调关键字段（`server` 字段的重要性）

**特点**：
- 使用模板字符串支持多行
- 包含 Markdown 格式的代码示例
- 强调资源对象的 `server` 字段

### 3. PROMPT - 模型提示

```typescript
export const PROMPT = `
List available resources from configured MCP servers.
Each returned resource will include all standard MCP resource fields plus a 'server' field 
indicating which server the resource belongs to.

Parameters:
- server (optional): The name of a specific MCP server to get resources from. If not provided,
  resources from all servers will be returned.
`
```

**目的**：
- 更详细地说明工具行为
- 明确参数定义和可选性
- 说明返回数据的结构特点

**与 DESCRIPTION 的区别**：

| 属性 | DESCRIPTION | PROMPT |
|------|-------------|--------|
| 受众 | 用户和模型 | 主要是模型 |
| 长度 | 简洁，含示例 | 详细，含参数说明 |
| 用途 | 快速了解 | 详细指导 |
| 格式 | 含 Markdown | 纯文本，结构化 |

---

## 具体技术实现

### 代码结构

```typescript
// 1. 工具名称常量
export const LIST_MCP_RESOURCES_TOOL_NAME = 'ListMcpResourcesTool'

// 2. 工具描述（面向用户和模型）
export const DESCRIPTION = `...`

// 3. 模型提示（面向模型）
export const PROMPT = `...`
```

### 导出方式

使用命名导出（Named Exports），便于按需导入：

```typescript
// 支持选择性导入
import { LIST_MCP_RESOURCES_TOOL_NAME } from './prompt.js'
import { DESCRIPTION } from './prompt.js'
import { PROMPT } from './prompt.js'

// 或一次性导入
import {
  LIST_MCP_RESOURCES_TOOL_NAME,
  DESCRIPTION,
  PROMPT,
} from './prompt.js'
```

### 字符串模板技术

使用 JavaScript 模板字符串（Template Literals）：

1. **多行支持**：无需 `\n` 转义，直接换行
2. **代码示例**：使用反引号包裹示例代码，避免转义
3. **缩进处理**：注意模板字符串会保留缩进空格

```typescript
// 实际输出包含缩进空格
export const DESCRIPTION = `
Lists available resources...
Each resource...
`
// 输出: "\nLists available resources...\nEach resource...\n"
```

---

## 关键代码路径与文件引用

### 导出使用位置

| 常量 | 使用文件 | 使用方式 |
|------|---------|---------|
| `LIST_MCP_RESOURCES_TOOL_NAME` | `ListMcpResourcesTool.ts` | `name: LIST_MCP_RESOURCES_TOOL_NAME` |
| `DESCRIPTION` | `ListMcpResourcesTool.ts` | `async description() { return DESCRIPTION }` |
| `PROMPT` | `ListMcpResourcesTool.ts` | `async prompt() { return PROMPT }` |

### 在 Tool 定义中的集成

```typescript
// ListMcpResourcesTool.ts
import {
  DESCRIPTION,
  LIST_MCP_RESOURCES_TOOL_NAME,
  PROMPT,
} from './prompt.js'

export const ListMcpResourcesTool = buildTool({
  name: LIST_MCP_RESOURCES_TOOL_NAME,
  async description() {
    return DESCRIPTION
  },
  async prompt() {
    return PROMPT
  },
  // ...
})
```

### 与其他工具的一致性

项目中的其他 MCP 工具遵循相同的模式：

```typescript
// ReadMcpResourceTool/prompt.ts 示例
export const READ_MCP_RESOURCE_TOOL_NAME = 'ReadMcpResourceTool'
export const DESCRIPTION = `...`
export const PROMPT = `...`
```

---

## 依赖与外部交互

### 无运行时依赖

`prompt.ts` 是纯常量定义文件：
- 无 `import` 语句
- 无外部库依赖
- 无运行时计算

### 编译时依赖

| 类型 | 说明 |
|------|------|
| TypeScript 编译器 | 类型检查和转译 |
| 模板字符串语法 | ES6+ 特性 |

### 被依赖关系

```
prompt.ts
└── ListMcpResourcesTool.ts
    └── buildTool()
        └── Tool 注册系统
            └── 模型工具选择逻辑
```

---

## 风险、边界与改进建议

### 已知风险

1. **字符串维护问题**
   - 风险：描述文本分散在多个文件中，更新时可能遗漏
   - 缓解：所有工具遵循统一的 `prompt.ts` 模式
   - 改进：考虑使用 i18n 框架集中管理

2. **文档与代码不同步**
   - 风险：参数变更后描述未及时更新
   - 缓解：代码审查时检查对应 `prompt.ts`
   - 改进：考虑使用 JSDoc 从代码生成描述

3. **国际化缺失**
   - 风险：仅支持英文，非英语用户理解困难
   - 现状：CLI 整体为英文界面，此问题影响较小

### 边界情况

| 场景 | 行为 |
|------|------|
| 空字符串 | 有效，但不推荐 |
| 超长描述 | 无限制，但可能影响模型理解效率 |
| 特殊字符 | 模板字符串正确处理 |
| 动态内容 | 不支持，纯静态字符串 |

### 改进建议

1. **添加版本信息**
   ```typescript
   export const VERSION = '1.0.0'
   export const CHANGELOG = `
   - 1.0.0: Initial version
   - 1.1.0: Added server parameter
   `
   ```

2. **结构化参数定义**
   ```typescript
   export const PARAMETERS = [
     {
       name: 'server',
       type: 'string',
       required: false,
       description: 'Specific MCP server name to filter by',
     },
   ] as const
   ```

3. **自动生成文档**
   ```typescript
   // 添加 JSDoc 注释支持文档生成
   /**
    * Lists available resources from configured MCP servers.
    * @see {@link https://docs.anthropic.com/mcp} MCP Documentation
    * @example
    * // List all resources
    * listMcpResources()
    * 
    * // List from specific server
    * listMcpResources({ server: "myserver" })
    */
   export const DESCRIPTION = `...`
   ```

4. **添加使用统计提示**
   ```typescript
   export const TIPS = `
   Tips:
   - Use without parameters to discover all available resources
   - Use with 'server' parameter when you know which server has the resource
   - Combine with ReadMcpResourceTool to access resource content
   `
   ```

5. **参数验证提示**
   ```typescript
   export const ERRORS = `
   Common errors:
   - Server not found: Check server name spelling
   - No resources: Server may not expose any resources
   - Connection failed: Server may be offline or misconfigured
   `
   ```

6. **与 Schema 联动**
   ```typescript
   // 考虑从 Zod Schema 生成参数描述
   import { inputSchema } from './ListMcpResourcesTool.js'
   
   export const PROMPT = generatePromptFromSchema(inputSchema, {
     description: 'List available resources from configured MCP servers',
   })
   ```

### 测试建议

1. **快照测试**
   ```typescript
   test('prompt.ts exports', () => {
     expect(DESCRIPTION).toMatchSnapshot()
     expect(PROMPT).toMatchSnapshot()
   })
   ```

2. **内容检查**
   ```typescript
   test('DESCRIPTION mentions server field', () => {
     expect(DESCRIPTION).toContain('server')
     expect(DESCRIPTION).toContain('resources')
   })
   ```

3. **长度检查**
   ```typescript
   test('prompts are reasonable length', () => {
     expect(DESCRIPTION.length).toBeLessThan(1000)
     expect(PROMPT.length).toBeLessThan(2000)
   })
   ```

### 维护检查清单

当修改此文件时，请检查：

- [ ] 描述是否准确反映工具当前行为
- [ ] 示例代码是否可运行
- [ ] 参数说明是否与 Schema 一致
- [ ] 拼写和语法是否正确
- [ ] 是否包含必要的上下文信息
