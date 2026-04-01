# prompt.ts 研究文档

## 场景与职责

prompt.ts 是 FileReadTool 的提示词模板模块，负责定义工具的描述文本和使用说明。它提供了静态常量、动态模板渲染函数，以及与 PDF 支持相关的条件提示词生成。

核心职责：
1. **定义工具常量** - 工具名称、描述、最大行数等
2. **渲染提示词模板** - 根据运行时配置生成完整的工具说明
3. **支持条件提示词** - 根据 PDF 支持状态调整提示词内容
4. **避免循环依赖** - 使用字符串常量代替直接导入其他工具

该模块在以下场景发挥关键作用：
- 系统生成工具列表时提供 Read 工具的描述
- 向模型说明 Read 工具的使用方法和限制
- 根据当前模型能力调整 PDF 相关说明

## 功能点目的

### 1. 工具标识常量

```typescript
export const FILE_READ_TOOL_NAME = 'Read'  // 工具显示名称
export const DESCRIPTION = 'Read a file from the local filesystem.'  // 简短描述
```

### 2. 文件未变更提示

```typescript
export const FILE_UNCHANGED_STUB = 
  'File unchanged since last read. The content from the earlier Read tool_result in this conversation is still current — refer to that instead of re-reading.'
```

用途：当文件读取去重功能检测到文件未变更时，返回此提示。

### 3. 最大行数限制

```typescript
export const MAX_LINES_TO_READ = 2000
```

在提示词中告知模型默认最多读取 2000 行。

### 4. 行号格式说明

```typescript
export const LINE_FORMAT_INSTRUCTION = 
  '- Results are returned using cat -n format, with line numbers starting at 1'
```

告知模型返回结果包含行号，从 1 开始。

### 5. 偏移量说明变体

提供两种偏移量使用策略：

**默认策略** - 鼓励读取整个文件：
```typescript
export const OFFSET_INSTRUCTION_DEFAULT = 
  "- You can optionally specify a line offset and limit (especially handy for long files), but it's recommended to read the whole file by not providing these parameters"
```

**针对性策略** - 鼓励只读取需要的部分：
```typescript
export const OFFSET_INSTRUCTION_TARGETED = 
  '- When you already know which part of the file you need, only read that part. This can be important for larger files.'
```

### 6. 动态提示词模板

根据运行时条件生成完整提示词：
- 是否包含文件大小限制说明
- 使用哪种偏移量策略
- 是否包含 PDF 支持说明

## 具体技术实现

### 核心常量

| 常量 | 值 | 用途 |
|------|-----|------|
| `FILE_READ_TOOL_NAME` | `'Read'` | 工具名称，避免循环依赖 |
| `FILE_UNCHANGED_STUB` | 长文本 | 文件未变更时的返回消息 |
| `MAX_LINES_TO_READ` | `2000` | 默认最大读取行数 |
| `DESCRIPTION` | `'Read a file...'` | 工具简短描述 |
| `LINE_FORMAT_INSTRUCTION` | 行号格式说明 | 告知模型输出格式 |
| `OFFSET_INSTRUCTION_DEFAULT` | 默认偏移策略 | 鼓励完整读取 |
| `OFFSET_INSTRUCTION_TARGETED` | 针对性偏移策略 | 鼓励部分读取 |

### 提示词模板函数

```typescript
export function renderPromptTemplate(
  lineFormat: string,           // 行号格式说明
  maxSizeInstruction: string,   // 文件大小限制说明（可为空）
  offsetInstruction: string,    // 偏移量使用策略
): string {
  return `Reads a file from the local filesystem. You can access any file directly by using this tool.
Assume this tool is able to read all files on the machine. If the User provides a path to a file assume that path is valid. It is okay to read a file that does not exist; an error will be returned.

Usage:
- The file_path parameter must be an absolute path, not a relative path
- By default, it reads up to ${MAX_LINES_TO_READ} lines starting from the beginning of the file${maxSizeInstruction}
${offsetInstruction}
${lineFormat}
- This tool allows Claude Code to read images (eg PNG, JPG, etc). When reading an image file the contents are presented visually as Claude Code is a multimodal LLM.${
    isPDFSupported()
      ? '\n- This tool can read PDF files (.pdf). For large PDFs (more than 10 pages), you MUST provide the pages parameter to read specific page ranges (e.g., pages: "1-5"). Reading a large PDF without the pages parameter will fail. Maximum 20 pages per request.'
      : ''
  }
- This tool can read Jupyter notebooks (.ipynb files) and returns all cells with their outputs, combining code, text, and visualizations.
- This tool can only read files, not directories. To read a directory, use an ls command via the ${BASH_TOOL_NAME} tool.
- You will regularly be asked to read screenshots. If the user provides a path to a screenshot, ALWAYS use this tool to view the file at the path. This tool will work with all temporary file paths.
- If you read a file that exists but has empty contents you will receive a system reminder warning in place of file contents.`
}
```

### 关键代码路径

| 功能 | 代码路径 | 行号 |
|------|----------|------|
| 工具名称常量 | `FILE_READ_TOOL_NAME` | 5 |
| 未变更提示 | `FILE_UNCHANGED_STUB` | 7-8 |
| 最大行数 | `MAX_LINES_TO_READ` | 10 |
| 工具描述 | `DESCRIPTION` | 12 |
| 行号格式 | `LINE_FORMAT_INSTRUCTION` | 14-15 |
| 默认偏移策略 | `OFFSET_INSTRUCTION_DEFAULT` | 17-18 |
| 针对性偏移策略 | `OFFSET_INSTRUCTION_TARGETED` | 20-21 |
| 模板渲染 | `renderPromptTemplate(lineFormat, maxSizeInstruction, offsetInstruction)` | 27-49 |

### 依赖与外部交互

#### 直接依赖模块

| 模块 | 用途 |
|------|------|
| `../../utils/pdfUtils.js` | `isPDFSupported()` 检测 PDF 支持 |
| `../BashTool/toolName.js` | `BASH_TOOL_NAME` 字符串常量 |

#### 依赖设计

**使用字符串常量的原因**：
```typescript
// 使用字符串常量避免循环依赖
import { BASH_TOOL_NAME } from '../BashTool/toolName.js'

// 而不是
// import { BashTool } from '../BashTool/BashTool.js'  // 可能导致循环依赖
```

`toolName.js` 模块只导出字符串常量，不导出工具定义，确保不会引入循环依赖。

### 条件提示词生成

PDF 相关提示词根据 `isPDFSupported()` 动态生成：

```typescript
isPDFSupported()
  ? '\n- This tool can read PDF files (.pdf). For large PDFs (more than 10 pages), you MUST provide the pages parameter to read specific page ranges (e.g., pages: "1-5"). Reading a large PDF without the pages parameter will fail. Maximum 20 pages per request.'
  : ''
```

**PDF 支持检测逻辑**（在 `pdfUtils.js` 中）：
```typescript
export function isPDFSupported(): boolean {
  return !getMainLoopModel().toLowerCase().includes('claude-3-haiku')
}
```

只有 Claude 3 Haiku 模型不支持 PDF，其他模型都支持。

## 风险、边界与改进建议

### 已知风险

1. **提示词长度**
   - 风险：完整的提示词较长，占用模型上下文
   - 当前：约 1.5K 字符
   - 缓解：内容必要，无法大幅缩减

2. **PDF 支持检测时机**
   - 风险：`isPDFSupported()` 在渲染时调用，依赖当前模型设置
   - 注意：如果模型在会话中切换，提示词不会自动更新
   - 缓解：模型切换时会重新初始化工具

3. **硬编码数值**
   - 风险：`MAX_LINES_TO_READ = 2000` 和 `PDF_MAX_PAGES_PER_READ = 20` 硬编码
   - 影响：调整需要代码变更和重新部署
   - 建议：考虑移至 GrowthBook 配置

4. **循环依赖风险**
   - 当前：通过 `toolName.js` 避免循环依赖
   - 风险：如果未来 `toolName.js` 导入其他模块，可能引入循环
   - 缓解：保持 `toolName.js` 仅导出字符串常量

### 边界情况

| 场景 | 处理方式 |
|------|----------|
| maxSizeInstruction 为空 | 模板中不显示大小限制说明 |
| PDF 不支持 | 不显示 PDF 相关说明 |
| 模型切换 | 新会话使用新模型的提示词 |

### 改进建议

1. **配置化数值**
   - 建议：将 `MAX_LINES_TO_READ` 和 `PDF_MAX_PAGES_PER_READ` 移至配置
   - 来源：GrowthBook 或环境变量
   - 好处：无需部署即可调整限制

2. **提示词模板引擎**
   - 当前：使用字符串模板拼接
   - 建议：考虑使用轻量级模板引擎（如 handlebars）
   - 好处：更清晰的模板结构，支持条件、循环

3. **国际化支持**
   - 当前：所有提示词为英文
   - 建议：增加多语言支持
   - 注意：模型主要使用英文，优先级可能不高

4. **提示词版本控制**
   - 建议：增加提示词版本号，用于遥测分析
   - 用途：分析不同提示词版本的效果

5. **动态示例**
   - 建议：根据用户历史行为生成个性化示例
   - 示例：如果用户经常读取特定类型文件，在提示词中增加相关示例

6. **A/B 测试支持**
   - 建议：提示词模板支持多版本，通过 GrowthBook 控制
   - 用途：测试不同提示词表述的效果

### 与 FileReadTool 的协作

prompt.ts 被 FileReadTool.ts 在以下位置使用：

1. **导入常量**
   ```typescript
   import {
     DESCRIPTION,
     FILE_READ_TOOL_NAME,
     FILE_UNCHANGED_STUB,
     LINE_FORMAT_INSTRUCTION,
     OFFSET_INSTRUCTION_DEFAULT,
     OFFSET_INSTRUCTION_TARGETED,
     renderPromptTemplate,
   } from './prompt.js'
   ```

2. **描述方法**
   ```typescript
   async description() {
     return DESCRIPTION
   }
   ```

3. **提示词方法**
   ```typescript
   async prompt() {
     const limits = getDefaultFileReadingLimits()
     const maxSizeInstruction = limits.includeMaxSizeInPrompt
       ? `. Files larger than ${formatFileSize(limits.maxSizeBytes)} will return an error...`
       : ''
     const offsetInstruction = limits.targetedRangeNudge
       ? OFFSET_INSTRUCTION_TARGETED
       : OFFSET_INSTRUCTION_DEFAULT
     return renderPromptTemplate(
       pickLineFormatInstruction(),
       maxSizeInstruction,
       offsetInstruction,
     )
   }
   ```

4. **未变更返回**
   ```typescript
   case 'file_unchanged':
     return {
       tool_use_id: toolUseID,
       type: 'tool_result',
       content: FILE_UNCHANGED_STUB,
     }
   ```

### 测试注意事项

- 需要 mock `isPDFSupported()` 以测试 PDF 提示词
- 模板渲染测试需要验证完整输出字符串
- 注意字符串中的换行符和空格
- 测试不同组合：有/无 maxSizeInstruction，有/无 PDF 支持

### 相关文件

| 文件 | 关系 |
|------|------|
| `src/tools/FileReadTool/FileReadTool.ts` | 主要调用方 |
| `src/utils/pdfUtils.js` | `isPDFSupported()` 函数 |
| `src/tools/BashTool/toolName.js` | `BASH_TOOL_NAME` 常量 |
| `src/tools/FileReadTool/limits.ts` | `getDefaultFileReadingLimits()` 提供配置 |
