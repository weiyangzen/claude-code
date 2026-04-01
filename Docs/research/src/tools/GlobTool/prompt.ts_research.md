# GlobTool/prompt.ts 研究文档

## 场景与职责

prompt.ts 是 GlobTool 的提示词和常量定义模块，负责定义：
1. 工具的常量标识（工具名称）
2. 工具的描述文本（用于 LLM 的 system prompt）

该模块是 GlobTool 的元数据层，为工具提供身份标识和能力描述，使 LLM 能够理解何时以及如何使用 GlobTool。

## 功能点目的

### 1. GLOB_TOOL_NAME
定义工具的内部标识名称 `"Glob"`，用于：
- 工具注册和查找
- 权限规则匹配
- 日志和遥测追踪
- API 调用中的工具标识

### 2. DESCRIPTION
为 LLM 提供工具的能力描述，包含以下关键信息：
- **功能定位**: 快速的文件模式匹配工具，适用于任何规模的代码库
- **模式支持**: 支持 glob 模式如 `"**/*.js"` 或 `"src/**/*.ts"`
- **返回内容**: 返回匹配的文件路径，按修改时间排序
- **使用场景**: 按文件名模式查找文件时使用
- **限制说明**: 对于开放式搜索（需要多轮 glob 和 grep），建议使用 Agent 工具

## 具体技术实现

### 关键常量定义

```typescript
// 行 1: 工具名称常量
export const GLOB_TOOL_NAME = 'Glob'

// 行 3-7: 工具描述（Markdown 格式，用于 LLM prompt）
export const DESCRIPTION = `- Fast file pattern matching tool that works with any codebase size
- Supports glob patterns like "**/*.js" or "src/**/*.ts"
- Returns matching file paths sorted by modification time
- Use this tool when you need to find files by name patterns
- When you are doing an open ended search that may require multiple rounds of globbing and grepping, use the Agent tool instead`
```

### 描述文本解析

DESCRIPTION 采用 Markdown 列表格式，包含 5 个要点：

| 要点 | 内容 | 目的 |
|-----|------|------|
| 1 | Fast file pattern matching tool... | 强调性能和适用规模 |
| 2 | Supports glob patterns like... | 说明支持的语法 |
| 3 | Returns matching file paths... | 说明输出格式和排序 |
| 4 | Use this tool when... | 明确使用场景 |
| 5 | When you are doing... | 限制范围，引导使用 Agent |

## 关键代码路径与文件引用

### 当前文件

```typescript
// src/tools/GlobTool/prompt.ts
export const GLOB_TOOL_NAME = 'Glob'

export const DESCRIPTION = `- Fast file pattern matching tool that works with any codebase size
- Supports glob patterns like "**/*.js" or "src/**/*.ts"
- Returns matching file paths sorted by modification time
- Use this tool when you need to find files by name patterns
- When you are doing an open ended search that may require multiple rounds of globbing and grepping, use the Agent tool instead`
```

### 导入方文件

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `src/tools/GlobTool/GlobTool.ts` | `GLOB_TOOL_NAME, DESCRIPTION` | 工具定义中使用 |
| `src/utils/messages.ts` | `GLOB_TOOL_NAME` | 消息处理中的工具识别 |

### GlobTool.ts 中的使用

```typescript
// GlobTool.ts 行 17
import { DESCRIPTION, GLOB_TOOL_NAME } from './prompt.js'

// GlobTool.ts 行 57-63
export const GlobTool = buildTool({
  name: GLOB_TOOL_NAME,  // 使用 'Glob'
  async description() {
    return DESCRIPTION  // 返回描述文本
  },
  // ...
})
```

## 依赖与外部交互

### 无外部依赖

prompt.ts 是一个纯常量定义模块，不导入任何外部依赖：
- 无运行时依赖
- 无类型依赖
- 无工具函数依赖

### 被依赖关系

```
prompt.ts
├── GlobTool.ts (导入 GLOB_TOOL_NAME, DESCRIPTION)
└── messages.ts (导入 GLOB_TOOL_NAME)
```

## 风险、边界与改进建议

### 潜在风险

1. **工具名称硬编码**
   - `GLOB_TOOL_NAME = 'Glob'` 是硬编码字符串
   - 如果重命名工具，需要同步修改所有引用处
   - 风险较低，因为工具名称通常稳定

2. **描述文本维护**
   - DESCRIPTION 是多行字符串，维护时需注意格式
   - 修改描述可能影响 LLM 对工具的理解和使用
   - 建议通过 A/B 测试验证描述变更的效果

3. **与 Agent 工具的边界模糊**
   - 描述中提到"多轮搜索使用 Agent 工具"
   - 实际边界可能因场景而异，LLM 可能难以判断

### 边界情况

1. **描述长度**: 当前描述约 400 字符，在合理范围内
2. **国际化**: 当前为英文描述，无多语言支持
3. **动态描述**: description() 是异步函数，但当前返回静态字符串

### 改进建议

1. **动态描述增强**
   ```typescript
   // 建议：根据上下文提供动态描述
   export async function getDescription(context: ToolContext): Promise<string> {
     const baseDesc = DESCRIPTION
     if (context.isLargeRepo) {
       return baseDesc + '\n- Note: This is a large repository, searches may take longer'
     }
     return baseDesc
   }
   ```

2. **添加使用示例**
   ```typescript
   export const EXAMPLES = `
   Example usage:
   - pattern: "**/*.test.ts" → finds all test files
   - pattern: "src/**/*.tsx", path: "src/components" → finds React components
   `
   ```

3. **版本控制**
   ```typescript
   export const TOOL_VERSION = '1.0.0'
   export const DESCRIPTION_VERSION = '2024-01'
   ```

4. **与 GrepTool 描述的一致性**
   - GrepTool 也有类似的描述结构
   - 建议统一搜索类工具的描述格式
   - 可以考虑提取共享的搜索工具描述模板

### 对比 GrepTool/prompt.ts

| 特性 | GlobTool | GrepTool |
|-----|----------|----------|
| 常量名 | GLOB_TOOL_NAME | GREP_TOOL_NAME |
| 描述长度 | ~400 字符 | ~600 字符（动态生成） |
| 描述类型 | 静态字符串 | 异步函数 getDescription() |
| 复杂度 | 简单 | 较复杂（支持配置） |

GlobTool 的 prompt.ts 设计更简洁，适合功能相对单一的文件搜索工具。GrepTool 由于功能更复杂（多种输出模式、参数等），需要动态生成描述。

### 维护建议

1. **变更管理**: 修改 DESCRIPTION 时，需评估对 LLM 行为的影响
2. **文档同步**: 确保描述与 README、用户文档保持一致
3. **测试覆盖**: 添加测试验证工具描述被正确包含在 system prompt 中
