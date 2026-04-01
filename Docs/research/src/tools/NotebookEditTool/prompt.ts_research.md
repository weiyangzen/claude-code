# prompt.ts 研究文档

## 场景与职责

prompt.ts 是 NotebookEditTool 的提示词定义文件，包含工具的描述文本（DESCRIPTION）和给模型的详细指令（PROMPT）。这些文本在工具被调用时提供给 AI 模型，指导其正确使用 NotebookEdit 工具。

## 功能点目的

### 1. 工具描述（DESCRIPTION）
- **用途**：在工具搜索结果中显示，帮助模型识别何时使用该工具
- **内容**：简洁说明工具的核心功能——替换 Jupyter Notebook 单元格内容

### 2. 模型指令（PROMPT）
- **用途**：提供给 AI 模型的详细使用说明
- **内容**：包含工具功能说明、参数要求、使用场景和编辑模式说明

## 具体技术实现

### 代码内容
```typescript
export const DESCRIPTION =
  'Replace the contents of a specific cell in a Jupyter notebook.'

export const PROMPT = `Completely replaces the contents of a specific cell in a Jupyter notebook (.ipynb file) with new source. Jupyter notebooks are interactive documents that combine code, text, and visualizations, commonly used for data analysis and scientific computing. The notebook_path parameter must be an absolute path, not a relative path. The cell_number is 0-indexed. Use edit_mode=insert to add a new cell at the index specified by cell_number. Use edit_mode=delete to delete the cell at the index specified by cell_number.`
```

### 内容分析

#### DESCRIPTION 分析
- **长度**：10 个单词
- **关键词**：Replace、contents、specific cell、Jupyter notebook
- **动作导向**：以动词开头，明确工具的操作类型

#### PROMPT 详细解析

| 段落 | 内容 | 目的 |
|-----|------|------|
| 功能定义 | "Completely replaces the contents of a specific cell..." | 明确工具的核心能力 |
| 文件格式 | "(.ipynb file)" | 指定支持的文件格式 |
| 应用场景 | "Jupyter notebooks are interactive documents..." | 帮助模型理解何时使用 |
| 路径要求 | "The notebook_path parameter must be an absolute path..." | 关键参数约束 |
| 索引说明 | "The cell_number is 0-indexed" | 索引约定 |
| 插入模式 | "Use edit_mode=insert to add a new cell..." | 扩展功能说明 |
| 删除模式 | "Use edit_mode=delete to delete the cell..." | 扩展功能说明 |

## 关键代码路径与文件引用

### 被引用位置

| 文件路径 | 用途 |
|---------|------|
| `src/tools/NotebookEditTool/NotebookEditTool.ts` | 工具描述和提示词源 |

### 引用方式
```typescript
// NotebookEditTool.ts 中的使用
import { DESCRIPTION, PROMPT } from './prompt.js'

export const NotebookEditTool = buildTool({
  async description() {
    return DESCRIPTION
  },
  async prompt() {
    return PROMPT
  },
  // ...
})
```

## 依赖与外部交互

### 无外部依赖
- 该文件为零依赖纯字符串定义
- 不导入任何其他模块

### 与 NotebookEditTool.ts 的关系
```
prompt.ts ──exported──> DESCRIPTION, PROMPT
                          │
                          └──imported──> NotebookEditTool.ts
                              ├──description() 方法
                              └──prompt() 方法
```

## 风险、边界与改进建议

### 已知问题

1. **文档与实现不一致**
   - PROMPT 中提到 `cell_number` 参数
   - 实际实现使用 `cell_id` 参数
   - 这是一个文档与代码不一致的问题

2. **缺少 cell_type 说明**
   - PROMPT 未提及 `cell_type` 参数
   - 但 `edit_mode=insert` 时该参数是必需的

3. **缺少 cell_id 格式说明**
   - 未说明 `cell_id` 可以是单元格 ID 或 `cell-N` 格式
   - 可能导致模型使用困惑

### 改进建议

1. **修正参数名称**
   ```typescript
   export const PROMPT = `Completely replaces the contents of a specific cell in a Jupyter notebook (.ipynb file) with new source. Jupyter notebooks are interactive documents that combine code, text, and visualizations, commonly used for data analysis and scientific computing. The notebook_path parameter must be an absolute path, not a relative path. The cell_id parameter identifies the target cell (can be the cell's ID or cell-N format for 0-indexed position). Use edit_mode=insert to add a new cell after the cell with the specified cell_id (requires cell_type parameter). Use edit_mode=delete to delete the cell at the specified cell_id.`
   ```

2. **结构化提示词**
   ```typescript
   export const PROMPT = `
   Edit a Jupyter notebook (.ipynb) cell.

   Parameters:
   - notebook_path: Absolute path to the .ipynb file
   - cell_id: Target cell identifier (cell ID or cell-N format)
   - new_source: New source code/content for the cell
   - cell_type: 'code' or 'markdown' (required for insert mode)
   - edit_mode: 'replace' (default), 'insert', or 'delete'

   Notes:
   - Cell indices are 0-indexed
   - Insert mode adds cell AFTER the specified cell_id
   - Insert mode requires cell_type to be specified
   `.trim()
   ```

3. **添加使用示例**
   ```typescript
   export const PROMPT_EXAMPLES = `
   Example 1: Replace cell content
   { "notebook_path": "/home/user/analysis.ipynb", "cell_id": "cell-0", "new_source": "import pandas as pd" }

   Example 2: Insert new code cell
   { "notebook_path": "/home/user/analysis.ipynb", "cell_id": "cell-0", "new_source": "print('hello')", "cell_type": "code", "edit_mode": "insert" }

   Example 3: Delete cell
   { "notebook_path": "/home/user/analysis.ipynb", "cell_id": "cell-2", "edit_mode": "delete" }
   `
   ```

4. **国际化支持准备**
   ```typescript
   // 为未来多语言支持预留结构
   export const DESCRIPTIONS = {
     en: 'Replace the contents of a specific cell in a Jupyter notebook.',
     zh: '替换 Jupyter Notebook 中特定单元格的内容。',
     // ...
   }
   ```

### 最佳实践对比

与其他工具提示词的对比：

| 工具 | 描述长度 | 详细程度 | 示例 |
|-----|---------|---------|------|
| NotebookEditTool | 中等 | 基础 | 无 |
| FileEditTool | 较长 | 详细 | 有 |
| BashTool | 长 | 非常详细 | 有 |

建议参考 FileEditTool 的提示词风格，增加结构化说明和示例。

### 维护建议

1. **版本控制**：提示词变更可能影响模型行为，应记录变更历史
2. **A/B 测试**：重大提示词修改应通过特性开关进行灰度测试
3. **文档同步**：确保提示词与代码实现保持同步更新
