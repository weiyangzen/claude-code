# notebook.ts 研究文档

## 场景与职责

本模块提供 Jupyter Notebook（.ipynb）文件的读取、解析和处理功能。核心职责包括：

1. **Notebook 读取**：读取并解析 .ipynb 文件为结构化数据
2. **单元格处理**：处理代码单元格、Markdown 单元格及其输出
3. **输出处理**：处理多种输出类型（stream、execute_result、display_data、error）
4. **工具结果映射**：将 Notebook 内容转换为 Claude API 的 ToolResultBlockParam 格式
5. **图片提取**：从输出数据中提取 base64 编码的图片

该模块是 FileReadTool 和 NotebookEditTool 的底层支持，使 Claude 能够理解和操作 Jupyter Notebook 文件。

## 功能点目的

### 1. `readNotebook()` - Notebook 读取
- **目的**：读取并解析 Jupyter Notebook 文件
- **功能**：
  - 支持读取整个 notebook 或特定单元格（通过 cellId）
  - 处理单元格输出大小限制（`LARGE_OUTPUT_THRESHOLD = 10000`）
  - 返回处理后的 `NotebookCellSource` 数组

### 2. `mapNotebookCellsToToolResult()` - 工具结果映射
- **目的**：将 Notebook 单元格数据映射为 Claude API 格式
- **处理逻辑**：
  - 将单元格内容和输出转换为 `TextBlockParam` 和 `ImageBlockParam`
  - 合并相邻的文本块以优化 token 使用
  - 生成 `ToolResultBlockParam` 结构

### 3. `parseCellId()` - 单元格 ID 解析
- **目的**：解析 `cell-N` 格式的单元格 ID
- **返回值**：单元格索引（number）或 undefined

### 4. 内部处理函数

#### `processCell()` - 单元格处理
- 处理单元格元数据（类型、执行次数、语言）
- 处理单元格输出（大小限制检查）
- 返回 `NotebookCellSource` 结构

#### `processOutput()` - 输出处理
支持四种输出类型：
- `stream`：标准输出/错误流
- `execute_result` / `display_data`：执行结果和显示数据（含图片提取）
- `error`：错误信息（ename、evalue、traceback）

#### `extractImage()` - 图片提取
- 从 `image/png` 或 `image/jpeg` 数据中提取 base64 图片
- 去除空白字符

## 具体技术实现

### 关键流程

#### Notebook 读取流程

```
readNotebook(notebookPath, cellId?)
    ↓
expandPath() 解析路径
    ↓
读取文件内容
    ↓
jsonParse() 解析 JSON
    ↓
提取语言信息 (metadata.language_info.name)
    ↓
如果指定 cellId:
    查找对应单元格 → processCell() → 返回单元素数组
否则:
    所有单元格 map(processCell) → 返回数组
```

#### 工具结果映射流程

```
mapNotebookCellsToToolResult(data, toolUseID)
    ↓
flatMap(getToolResultFromCell)
    ↓
reduce 合并相邻文本块
    ↓
返回 ToolResultBlockParam
```

### 数据结构

```typescript
// Notebook 单元格（处理后）
interface NotebookCellSource {
  cellType: 'code' | 'markdown' | 'raw'
  source: string           // 单元格源代码
  execution_count?: number // 执行次数（仅代码单元格）
  cell_id: string          // 单元格 ID
  language?: string        // 编程语言（仅代码单元格）
  outputs?: NotebookCellSourceOutput[]
}

// 单元格输出（处理后）
interface NotebookCellSourceOutput {
  output_type: 'stream' | 'execute_result' | 'display_data' | 'error'
  text?: string
  image?: NotebookOutputImage
}

// 输出图片
interface NotebookOutputImage {
  image_data: string       // base64 编码
  media_type: 'image/png' | 'image/jpeg'
}
```

### 大输出处理

```typescript
const LARGE_OUTPUT_THRESHOLD = 10000

function isLargeOutputs(outputs): boolean {
  let size = 0
  for (const o of outputs) {
    size += (o.text?.length ?? 0) + (o.image?.image_data.length ?? 0)
    if (size > LARGE_OUTPUT_THRESHOLD) return true
  }
  return false
}

// 大输出替换为提示信息
{
  output_type: 'stream',
  text: `Outputs are too large to include. Use BashTool with: cat <notebook_path> | jq '.cells[${index}].outputs'`
}
```

### 文本块合并优化

```typescript
// 合并相邻文本块以减少 token 使用
content.reduce((acc, curr) => {
  if (acc.length === 0) return [curr]
  const prev = acc[acc.length - 1]
  if (prev?.type === 'text' && curr.type === 'text') {
    prev.text += '\n' + curr.text  // 合并
    return acc
  }
  acc.push(curr)
  return acc
}, [])
```

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `@anthropic-ai/sdk/resources/index.mjs` | SDK 类型定义 |
| `../tools/BashTool/toolName.js` | Bash 工具名称常量 |
| `../tools/BashTool/utils.js` | 输出格式化 |
| `../types/notebook.js` | Notebook 类型定义 |
| `./fsOperations.js` | 文件系统操作 |
| `./path.js` | 路径扩展 |
| `./slowOperations.js` | JSON 解析 |

### 调用方

| 调用方 | 用途 |
|--------|------|
| `src/tools/FileReadTool/FileReadTool.ts` | 读取 notebook 文件 |
| `src/tools/NotebookEditTool/NotebookEditTool.ts` | 编辑 notebook 单元格 |

### 类型定义来源

- `NotebookContent`、`NotebookCell` 等类型定义在 `../types/notebook.js`
- 实际运行时由调用方提供类型定义

## 风险、边界与改进建议

### 已知风险

1. **Notebook 格式兼容性**
   - 风险：Jupyter Notebook 格式版本差异
   - 现状：假设 nbformat 4.x 格式
   - 潜在问题：旧版本或未来版本可能不兼容

2. **大输出处理**
   - 风险：`LARGE_OUTPUT_THRESHOLD` 固定为 10000 字符
   - 可能问题：对于某些场景可能过小或过大
   - 建议：基于 token 数而非字符数

3. **图片格式支持**
   - 当前：仅支持 PNG 和 JPEG
   - 风险：其他格式（SVG、GIF）被忽略
   - 建议：添加更多格式支持或转换

4. **错误输出处理**
   - 当前：简单拼接 traceback
   - 风险：长 traceback 可能超出限制
   - 建议：添加 traceback 截断和格式化

### 边界情况

| 场景 | 行为 |
|------|------|
| Notebook 格式无效 | 依赖 jsonParse() 抛出错误 |
| 单元格无 ID | 使用 `cell-${index}` 生成 |
| 输出为空数组 | 返回空 outputs 数组 |
| 图片数据含空白 | extractImage() 去除空白 |
| 未知输出类型 | TypeScript 编译时检查 |
| 代码单元格无语言 | 默认 'python' |
| 指定 cellId 不存在 | 抛出错误 "Cell with ID ... not found" |

### 改进建议

1. **格式版本检测**
   - 当前：无版本检查
   - 建议：
     - 读取 `nbformat` 和 `nbformat_minor`
     - 根据版本调整解析逻辑
     - 警告不兼容版本

2. **输出大小动态限制**
   - 当前：固定 10000 字符阈值
   - 建议：
     - 基于 token 估算
     - 可配置阈值
     - 按输出类型差异化处理

3. **更多图片格式**
   - 当前：仅 PNG/JPEG
   - 建议：
     - 支持 SVG（转换为 PNG 或文本描述）
     - 支持 GIF（首帧或完整动画）
     - 支持 WebP

4. **交互式输出**
   - 当前：忽略交互式输出（widgets）
   - 建议：
     - 记录 widget 状态
     - 提供 widget 存在提示

5. **Markdown 渲染**
   - 当前：Markdown 单元格作为纯文本
   - 建议：
     - 可选渲染为 HTML
     - 保留原始 Markdown

6. **执行环境信息**
   - 当前：仅提取语言名称
   - 建议：
     - 提取内核信息
     - 提取依赖包版本
     - 用于复现环境

7. **增量读取**
   - 当前：读取整个 notebook
   - 建议：
     - 支持仅读取元数据
     - 支持流式读取大 notebook

8. **单元格输出缓存**
   - 当前：每次重新处理
   - 建议：
     - 缓存处理后的输出
     - 基于 mtime 失效

9. **安全性增强**
   - 风险：Notebook 可能包含恶意代码
   - 建议：
     - 扫描代码单元格中的危险模式
     - 警告用户潜在风险

10. **遥测集成**
    - 建议：
      - 记录 notebook 读取频率
      - 记录单元格类型分布
      - 记录输出大小分布
