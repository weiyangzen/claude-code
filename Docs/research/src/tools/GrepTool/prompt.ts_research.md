# prompt.ts 深度研究文档

## 场景与职责

prompt.ts 是 GrepTool 的轻量级配置模块，负责：

1. **工具名称定义**：导出 `GREP_TOOL_NAME` 常量
2. **工具描述生成**：提供 `getDescription()` 函数生成工具的系统提示描述

该模块是工具与系统提示之间的桥梁，确保模型理解工具的能力和最佳实践。

## 功能点目的

### 核心功能

| 功能 | 目的 |
|------|------|
| `GREP_TOOL_NAME` | 工具名称常量，用于注册和引用 |
| `getDescription()` | 生成工具描述，指导模型正确使用 |

### 描述内容目的

工具描述不仅说明功能，更重要的是指导模型：

1. **使用规范**：明确何时使用 GrepTool，何时使用其他工具
2. **语法提示**：说明 ripgrep 与 grep 的区别
3. **最佳实践**：提供模式编写建议

## 具体技术实现

### 模块结构

```typescript
import { AGENT_TOOL_NAME } from '../AgentTool/constants.js'
import { BASH_TOOL_NAME } from '../BashTool/toolName.js'

export const GREP_TOOL_NAME = 'Grep'

export function getDescription(): string {
  return `A powerful search tool built on ripgrep

  Usage:
  - ALWAYS use ${GREP_TOOL_NAME} for search tasks...
  - Supports full regex syntax...
  - Filter files with glob parameter...
  - Output modes: "content"...
  - Use ${AGENT_TOOL_NAME} tool for open-ended searches...
  - Pattern syntax: Uses ripgrep (not grep)...
  - Multiline matching: By default patterns match within single lines only...
`
}
```

### 描述内容详解

```
A powerful search tool built on ripgrep

Usage:
- ALWAYS use Grep for search tasks. NEVER invoke `grep` or `rg` as a Bash command. 
  The Grep tool has been optimized for correct permissions and access.
  
  【目的：强制使用 GrepTool 而非 Bash 执行 grep，确保权限检查】

- Supports full regex syntax (e.g., "log.*Error", "function\\s+\\w+")
  
  【目的：说明支持的正则语法】

- Filter files with glob parameter (e.g., "*.js", "**/*.tsx") or type parameter 
  (e.g., "js", "py", "rust")
  
  【目的：说明文件过滤能力】

- Output modes: "content" shows matching lines, "files_with_matches" shows only 
  file paths (default), "count" shows match counts
  
  【目的：说明三种输出模式】

- Use Agent tool for open-ended searches requiring multiple rounds
  
  【目的：界定 GrepTool 与 AgentTool 的职责边界】

- Pattern syntax: Uses ripgrep (not grep) - literal braces need escaping 
  (use `interface\\{}` to find `interface{}` in Go code)
  
  【目的：说明 ripgrep 与 grep 的语法差异，避免转义错误】

- Multiline matching: By default patterns match within single lines only. 
  For cross-line patterns like `struct \\{[\\s\\S]*?field`, use `multiline: true`
  
  【目的：说明多行匹配的使用场景】
```

### 跨工具引用

模块导入其他工具的名称常量，用于描述中的交叉引用：

```typescript
import { AGENT_TOOL_NAME } from '../AgentTool/constants.js'  // 'Agent'
import { BASH_TOOL_NAME } from '../BashTool/toolName.js'      // 'Bash'
```

这种设计确保：
1. 工具名称变更时描述自动同步
2. 避免硬编码字符串导致的维护问题

## 关键代码路径与文件引用

### 内部依赖

| 文件 | 用途 |
|------|------|
| `src/tools/AgentTool/constants.js` | 导入 AGENT_TOOL_NAME |
| `src/tools/BashTool/toolName.js` | 导入 BASH_TOOL_NAME |

### 调用方

| 文件 | 用途 |
|------|------|
| `src/tools/GrepTool/GrepTool.ts` | 导入 GREP_TOOL_NAME 和 getDescription |
| `src/constants/prompts.ts` | 可能聚合所有工具描述 |

### 工具注册链

```
src/tools/GrepTool/prompt.ts
    │ exports GREP_TOOL_NAME, getDescription
    ▼
src/tools/GrepTool/GrepTool.ts
    │ imports for tool definition
    ▼
src/tools.ts
    │ aggregates all tools
    ▼
src/Tool.ts
    │ buildTool factory
    ▼
System Prompt Generation
```

## 依赖与外部交互

### 与 AgentTool 的关系

描述中明确建议：
```
Use Agent tool for open-ended searches requiring multiple rounds
```

这界定了职责边界：
- **GrepTool**：单次、确定性的搜索任务
- **AgentTool**：需要多轮探索的开放式搜索

### 与 BashTool 的关系

描述中明确禁止：
```
NEVER invoke `grep` or `rg` as a Bash command
```

原因：
1. GrepTool 有优化的权限检查
2. GrepTool 有结果大小限制和格式化
3. GrepTool 支持分页和输出模式

## 风险、边界与改进建议

### 已知风险

1. **描述与实现不同步**
   - 描述中提到的功能需要与实际代码保持一致
   - 如参数变更需要同步更新描述

2. **国际化缺失**
   - 当前描述为硬编码英文
   - 多语言支持需要重构

3. **过度指导风险**
   - 详细的描述可能限制模型的灵活性
   - 需要平衡指导与灵活性

### 边界情况

1. **工具名称变更**
   - GREP_TOOL_NAME 是硬编码的 'Grep'
   - 变更名称需要同步更新所有引用

2. **描述长度**
   - 当前描述适中，但随功能增加可能膨胀
   - 需要考虑描述长度对上下文的影响

### 改进建议

1. **动态描述**
   - 根据环境（如是否支持多行模式）动态调整描述
   - 示例：
   ```typescript
   export function getDescription(context?: ToolContext): string {
     const baseDesc = '...';
     if (context?.supportsMultiline) {
       return baseDesc + '\n- Multiline mode is available...';
     }
     return baseDesc;
   }
   ```

2. **使用示例**
   - 添加具体的使用示例
   - 示例：
   ```
   Examples:
   - Find all TODOs: pattern: "TODO|FIXME", glob: "*.ts"
   - Search for function: pattern: "function\\s+\\w+", output_mode: "content"
   ```

3. **版本控制**
   - 添加描述版本号，便于追踪变更
   - 示例：
   ```typescript
   export const GREP_TOOL_DESCRIPTION_VERSION = '1.2.0';
   ```

4. **配置化**
   - 将描述中的硬编码值（如默认 head_limit）改为引用常量
   - 确保描述与代码行为一致

5. **文档生成**
   - 考虑从代码注释自动生成描述
   - 或使用 schema 定义生成参数说明
