# UI.tsx 研究文档

## 场景与职责

UI.tsx 是 FileReadTool 的专用 UI 渲染模块，负责将文件读取工具的使用过程和结果以用户友好的方式展示在终端界面中。它采用 React + Ink 技术栈，为 Claude Code 的交互式命令行界面提供富文本渲染能力。

核心职责：
1. **工具使用消息渲染** - 显示正在读取的文件路径和参数
2. **工具结果消息渲染** - 显示读取完成后的摘要信息
3. **工具使用标签渲染** - 显示额外的元数据标签（如 Agent 任务 ID）
4. **错误消息渲染** - 处理并美化文件读取错误提示
5. **用户可见名称生成** - 根据上下文提供描述性名称

该模块在以下场景发挥关键作用：
- 用户执行文件读取操作时显示进度和结果
- Agent 子任务读取输出文件时的特殊显示
- 读取计划文件（plans）时的特殊标识
- 错误发生时提供清晰的错误提示

## 功能点目的

### 1. 工具使用消息渲染 (`renderToolUseMessage`)

显示文件读取操作的参数信息：
- **文件路径**: 支持 verbose 模式显示完整路径或简化路径
- **页码范围**: PDF 读取时显示 `pages 1-5`
- **行号范围**: 部分读取时显示 `lines 10-20` 或 `from line 10`
- **Agent 输出文件**: 特殊处理，不显示路径（由上层组件显示任务 ID）

### 2. 工具结果消息渲染 (`renderToolResultMessage`)

根据读取的文件类型显示不同的结果摘要：
- **图片**: "Read image (42KB)"
- **Notebook**: "Read 5 cells"
- **PDF**: "Read PDF (1.2MB)"
- **PDF 页面**: "Read 3 pages (1.2MB)"
- **文本**: "Read 100 lines"
- **未变更**: "Unchanged since last read"

### 3. 工具使用标签渲染 (`renderToolUseTag`)

为特定场景添加标签：
- **Agent 任务输出**: 显示任务 ID（如 `task_abc123`）

### 4. 错误消息渲染 (`renderToolUseErrorMessage`)

处理文件读取错误：
- **文件未找到**: 显示 "File not found"
- **读取错误**: 显示 "Error reading file"
- **详细模式**: 回退到通用错误组件显示完整错误

### 5. 用户可见名称 (`userFacingName`)

根据文件类型提供描述性名称：
- 计划文件: "Reading Plan"
- Agent 输出: "Read agent output"
- 默认: "Read"

### 6. 工具使用摘要 (`getToolUseSummary`)

生成工具使用的简短摘要：
- Agent 任务: 返回任务 ID
- 其他: 返回显示路径

## 具体技术实现

### 核心数据结构

```typescript
// 输入类型（来自 FileReadTool.ts）
interface Input {
  file_path: string
  offset?: number
  limit?: number
  pages?: string
}

// 输出类型（来自 FileReadTool.ts）
type Output = 
  | { type: 'text'; file: { numLines: number } }
  | { type: 'image'; file: { originalSize: number } }
  | { type: 'notebook'; file: { cells: unknown[] } }
  | { type: 'pdf'; file: { originalSize: number } }
  | { type: 'parts'; file: { count: number; originalSize: number } }
  | { type: 'file_unchanged' }
```

### 关键流程

#### 1. Agent 输出文件检测

```typescript
function getAgentOutputTaskId(filePath: string): string | null {
  const prefix = `${getTaskOutputDir()}/`  // e.g., /tmp/project/tasks/
  const suffix = '.output'
  if (filePath.startsWith(prefix) && filePath.endsWith(suffix)) {
    const taskId = filePath.slice(prefix.length, -suffix.length)
    // 验证任务 ID 格式（字母数字，1-20 字符）
    if (taskId.length > 0 && taskId.length <= 20 && /^[a-zA-Z0-9_-]+$/.test(taskId)) {
      return taskId
    }
  }
  return null
}
```

#### 2. 工具使用消息渲染流程

```
renderToolUseMessage({ file_path, offset, limit, pages }, { verbose }):
    1. 如果是 Agent 输出文件 → 返回空字符串（上层显示任务 ID）
    2. 获取显示路径（verbose ? 完整路径 : 简化路径）
    3. 如果提供 pages → 显示 "path · pages X-Y"
    4. 如果 verbose 且提供 offset/limit → 显示行号范围
    5. 默认 → 显示文件路径链接
```

#### 3. 结果消息渲染流程

```
renderToolResultMessage(output):
    switch (output.type):
        case 'image':
            格式化文件大小 → "Read image (42KB)"
        case 'notebook':
            统计 cell 数量 → "Read N cells"
        case 'pdf':
            格式化文件大小 → "Read PDF (1.2MB)"
        case 'parts':
            统计页数 + 大小 → "Read N pages (1.2MB)"
        case 'text':
            统计行数 → "Read N lines"
        case 'file_unchanged':
            灰色显示 → "Unchanged since last read"
```

### 关键代码路径

| 功能 | 代码路径 | 行号 |
|------|----------|------|
| Agent 输出检测 | `getAgentOutputTaskId(filePath)` | 18-29 |
| 工具使用消息 | `renderToolUseMessage({ file_path, offset, limit, pages }, { verbose })` | 30-65 |
| 工具使用标签 | `renderToolUseTag({ file_path })` | 66-76 |
| 工具结果消息 | `renderToolResultMessage(output)` | 77-143 |
| 错误消息 | `renderToolUseErrorMessage(result, { verbose })` | 144-164 |
| 用户可见名称 | `userFacingName(input)` | 165-173 |
| 工具使用摘要 | `getToolUseSummary(input)` | 174-184 |

### 依赖与外部交互

#### 直接依赖模块

| 模块 | 用途 |
|------|------|
| `@anthropic-ai/sdk/resources/index.mjs` | `ToolResultBlockParam` 类型定义 |
| `react` | React 组件框架 |
| `src/utils/messages.js` | `extractTag` 工具函数 |
| `../../components/FallbackToolUseErrorMessage.js` | 通用错误消息组件 |
| `../../components/FilePathLink.js` | 可点击的文件路径链接组件 |
| `../../components/MessageResponse.js` | 消息响应容器组件 |
| `../../ink.js` | `Text` 组件（Ink 终端渲染）|
| `../../utils/file.js` | `FILE_NOT_FOUND_CWD_NOTE`, `getDisplayPath` |
| `../../utils/format.js` | `formatFileSize` 文件大小格式化 |
| `../../utils/plans.js` | `getPlansDirectory` 计划目录路径 |
| `../../utils/task/diskOutput.js` | `getTaskOutputDir` 任务输出目录 |
| `./FileReadTool.js` | `Input`, `Output` 类型定义 |

#### 组件层次结构

```
renderToolUseMessage
├── FilePathLink (可点击路径)
└── Text (Ink 文本)

renderToolResultMessage
└── MessageResponse (容器)
    └── Text (Ink 文本)
        └── Text bold (高亮数字)

renderToolUseErrorMessage
├── MessageResponse (文件未找到/读取错误)
│   └── Text color="error"
└── FallbackToolUseErrorMessage (详细模式回退)
```

### 样式与主题

| 元素 | 样式 |
|------|------|
| 行数/页数/单元格数 | `bold` 加粗 |
| 未变更提示 | `dimColor` 暗淡颜色 |
| 错误文本 | `color="error"` 错误色 |
| 消息容器 | `height={1}` 固定高度 |

## 风险、边界与改进建议

### 已知风险

1. **Agent 输出文件识别依赖路径约定**
   - 风险：如果任务输出目录结构变更，识别可能失效
   - 缓解：路径通过 `getTaskOutputDir()` 统一获取，变更只需修改一处

2. **任务 ID 验证正则表达式**
   - 当前：`/^[a-zA-Z0-9_-]+$/`
   - 限制：20 字符长度限制可能不适用于所有任务 ID 生成策略

3. **文件路径显示安全性**
   - 已处理：通过 `getDisplayPath` 将绝对路径转为相对路径或 ~ 表示
   - 注意：verbose 模式仍可能暴露完整路径

### 边界情况

| 场景 | 处理方式 |
|------|----------|
| file_path 为空 | 返回 null，不渲染 |
| cells 为空数组 | 显示 "No cells found in notebook"（红色错误）|
| numLines = 1 | 使用单数形式 "line" |
| count = 1 | 使用单数形式 "page" |
| 非详细模式错误 | 简化显示 "File not found" 或 "Error reading file" |
| 详细模式错误 | 回退到 FallbackToolUseErrorMessage 显示完整错误 |

### 改进建议

1. **国际化支持**
   - 当前：所有文本硬编码为英文
   - 建议：添加 i18n 框架支持多语言

2. **可访问性**
   - 当前：依赖颜色和加粗区分信息
   - 建议：增加图标或前缀符号辅助色盲用户

3. **Agent 输出显示优化**
   - 当前：仅显示任务 ID
   - 建议：增加任务状态图标（进行中/完成/失败）

4. **文件类型图标**
   - 当前：纯文本显示
   - 建议：根据文件扩展名显示不同图标（📄 📊 🖼️）

5. **大数字格式化**
   - 当前：直接显示数字（如 10000 lines）
   - 建议：大数字使用 K/M 缩写（如 10K lines）

6. **路径截断策略**
   - 当前：verbose 模式显示完整路径可能换行
   - 建议：增加智能截断（中间省略）保持单行显示

### 与 FileReadTool.ts 的协作

UI.tsx 与 FileReadTool.ts 通过以下方式协作：

1. **类型共享**: UI.tsx 从 FileReadTool.ts 导入 `Input` 和 `Output` 类型
2. **渲染调用**: FileReadTool.ts 在 `buildTool` 中注册 UI 函数：
   ```typescript
   renderToolUseMessage,
   renderToolUseTag,
   renderToolResultMessage,
   renderToolUseErrorMessage,
   userFacingName,
   getToolUseSummary,
   ```
3. **数据流**: FileReadTool.ts 的 `call` 方法返回 `Output` 数据，UI.tsx 负责渲染

### 测试注意事项

- Ink 组件需要在终端环境中测试，Jest 需要特殊配置
- 文件路径测试需要考虑不同操作系统（Windows/Unix）
- Agent 输出检测测试需要模拟 `getTaskOutputDir()` 返回值
- 错误消息测试需要覆盖 verbose 和非 verbose 模式
