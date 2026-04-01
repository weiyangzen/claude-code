# prompt.ts 研究文档

## 场景与职责

prompt.ts 是 MCP 工具的**提示词占位模块**，定义了 MCPTool 的基础 `PROMPT` 和 `DESCRIPTION` 常量。这些值在 MCPTool.ts 中被引用，但实际在运行时被 `src/services/mcp/client.ts` 动态覆盖。

### 核心职责

1. **占位符定义**：提供空的 `PROMPT` 和 `DESCRIPTION` 常量
2. **模块接口**：为 MCPTool.ts 提供必需的导入目标

### 使用场景

- MCPTool.ts 需要导入 `DESCRIPTION` 和 `PROMPT` 以满足 `buildTool` 的接口要求
- 实际值在 MCP 服务器连接时，由 client.ts 从服务器获取并覆盖

---

## 功能点目的

### 1. 占位符常量

```typescript
// Actual prompt and description are overridden in mcpClient.ts
export const PROMPT = ''
export const DESCRIPTION = ''
```

**设计意图**：
- MCPTool.ts 作为工具模板需要这些字段
- 实际值无法静态确定（取决于连接的 MCP 服务器）
- 使用空字符串作为安全默认值

### 2. 注释说明

文件中的注释明确指出了实际逻辑位置：
```typescript
// Actual prompt and description are overridden in mcpClient.ts
```

---

## 具体技术实现

### 代码结构

```typescript
// 第 1 行：注释说明
// Actual prompt and description are overridden in mcpClient.ts

// 第 2 行：PROMPT 常量
export const PROMPT = ''

// 第 3 行：DESCRIPTION 常量
export const DESCRIPTION = ''
```

### 文件大小

- **总行数**：3 行
- **总字符数**：约 119 字节
- **代码复杂度**：极低（纯常量定义）

---

## 关键代码路径与文件引用

### 直接依赖

```
prompt.ts
└── 无外部依赖
```

### 被引用位置

```
src/tools/MCPTool/MCPTool.ts
├── import { DESCRIPTION, PROMPT } from './prompt.js'
├── async description() { return DESCRIPTION }  // 返回空字符串
└── async prompt() { return PROMPT }            // 返回空字符串
```

### 覆盖位置

```
src/services/mcp/client.ts
└── fetchToolsForClient()
    └── 工具定义：
        async description() { return tool.description ?? '' },  // 从服务器获取
        async prompt() { ... }  // 从服务器获取并截断
```

---

## 依赖与外部交互

### 无依赖

该模块不依赖任何外部模块。

### 被依赖

| 模块 | 用途 |
|------|------|
| `MCPTool.ts` | 导入 `PROMPT` 和 `DESCRIPTION` 作为默认值 |

---

## 风险、边界与改进建议

### 当前风险

1. **空值风险**：
   - 如果 client.ts 未正确覆盖，工具将没有描述
   - 风险：模型无法了解工具用途
   - 缓解：client.ts 中始终覆盖这些值

2. **维护困惑**：
   - 新开发者可能不理解为什么存在空常量
   - 风险：误修改或删除
   - 缓解：注释明确说明用途

### 边界情况

无特殊边界情况，该模块非常简单。

### 改进建议

1. **添加警告**：
   ```typescript
   export const PROMPT = ''
   export const DESCRIPTION = ''
   
   // 添加开发时警告
   if (process.env.NODE_ENV === 'development') {
     console.warn('prompt.ts 的值应在运行时被 client.ts 覆盖')
   }
   ```

2. **使用 Symbol**：
   ```typescript
   export const PROMPT = Symbol('PLACEHOLDER_PROMPT') as unknown as string
   export const DESCRIPTION = Symbol('PLACEHOLDER_DESCRIPTION') as unknown as string
   ```
   这样如果未覆盖会导致类型错误。

3. **合并到 MCPTool.ts**：
   由于文件非常简单，可以考虑直接内联到 MCPTool.ts：
   ```typescript
   // MCPTool.ts
   const PLACEHOLDER_PROMPT = ''
   const PLACEHOLDER_DESCRIPTION = ''
   ```

4. **添加 JSDoc**：
   ```typescript
   /**
    * @deprecated 此值在运行时被 mcpClient.ts 覆盖
    */
   export const PROMPT = ''
   ```

---

## 附录：相关代码对比

### prompt.ts（占位符）

```typescript
export const PROMPT = ''
export const DESCRIPTION = ''
```

### client.ts（实际实现）

```typescript
async description() {
  return tool.description ?? ''
},
async prompt() {
  const desc = tool.description ?? ''
  return desc.length > MAX_MCP_DESCRIPTION_LENGTH
    ? desc.slice(0, MAX_MCP_DESCRIPTION_LENGTH) + '… [truncated]'
    : desc
},
```

### 对比说明

| 特性 | prompt.ts | client.ts |
|------|-----------|-----------|
| 值来源 | 静态空字符串 | 动态从 MCP 服务器获取 |
| 截断处理 | 无 | 超过 2048 字符截断 |
| 调用时机 | 模块加载 | 每次需要描述时 |
| 实际使用 | 否（被覆盖） | 是 |
