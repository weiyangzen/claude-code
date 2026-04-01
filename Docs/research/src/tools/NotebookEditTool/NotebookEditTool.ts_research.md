# NotebookEditTool.ts 研究文档

## 场景与职责

NotebookEditTool 是 Claude Code 中专门用于编辑 Jupyter Notebook (.ipynb) 文件的工具。与通用的 FileEditTool 不同，该工具专门处理 Notebook 的特殊 JSON 结构，支持对单元格（cell）进行替换、插入和删除操作。

### 核心职责
1. **Notebook 单元格编辑**：支持对 .ipynb 文件中的代码单元格（code cell）和 Markdown 单元格进行内容修改
2. **单元格生命周期管理**：支持插入新单元格、删除现有单元格、替换单元格内容
3. **Notebook 格式兼容**：支持 nbformat 4.5+ 版本的单元格 ID 规范
4. **执行状态管理**：在修改代码单元格时自动重置执行计数和输出，保持 Notebook 状态一致性

## 功能点目的

### 1. 三种编辑模式
- **replace（默认）**：替换指定单元格的源代码内容
- **insert**：在指定单元格后插入新单元格
- **delete**：删除指定单元格

### 2. 单元格定位机制
- 优先通过单元格的 `id` 字段匹配（nbformat 4.5+）
- 回退到 `cell-N` 格式的索引解析（兼容旧版本 Notebook）
- 支持在末尾自动将 replace 模式转换为 insert 模式

### 3. 安全与权限控制
- 强制要求先读取后写入（Read-before-Edit），防止基于过时内容的意外覆盖
- 文件修改时间戳检查，检测外部修改
- UNC 路径安全拦截（防止 NTLM 凭证泄漏）
- 集成文件历史追踪（fileHistory）用于撤销/重做

## 具体技术实现

### 关键数据结构

```typescript
// 输入参数结构（Zod Schema）
{
  notebook_path: string;      // 绝对路径
  cell_id?: string;           // 单元格标识
  new_source: string;         // 新源代码
  cell_type?: 'code' | 'markdown';  // 单元格类型
  edit_mode?: 'replace' | 'insert' | 'delete';
}

// 输出结果结构
{
  new_source: string;
  cell_id?: string;
  cell_type: 'code' | 'markdown';
  language: string;
  edit_mode: string;
  error?: string;
  // 归因追踪字段
  notebook_path: string;
  original_file: string;
  updated_file: string;
}
```

### 核心流程

#### 1. 输入验证流程（validateInput）
```
1. 路径解析（支持相对路径转换为绝对路径）
2. UNC 路径安全检查（跳过以 \\ 或 // 开头的路径）
3. 文件扩展名验证（必须为 .ipynb）
4. 编辑模式验证（replace/insert/delete）
5. insert 模式强制要求 cell_type
6. Read-before-Edit 检查（必须已读取文件）
7. 文件修改时间戳检查（防止外部修改）
8. 文件存在性验证
9. JSON 有效性验证
10. 单元格存在性验证（通过 ID 或索引）
```

#### 2. 执行流程（call 方法）
```
1. 文件历史追踪记录（fileHistoryTrackEdit）
2. 读取文件元数据（内容、编码、换行符）
3. JSON 解析为 NotebookContent
4. 单元格索引解析（ID 优先，索引回退）
5. 边界处理（末尾 replace 自动转 insert）
6. 新单元格 ID 生成（nbformat 4.5+）
7. 执行编辑操作（delete/insert/replace）
8. 代码单元格状态重置（execution_count, outputs）
9. JSON 序列化（缩进为 1，符合 Jupyter 规范）
10. 文件写入（保持原编码和换行符）
11. 读取状态更新（readFileState）
12. 返回操作结果
```

### 关键代码路径

#### 单元格定位逻辑（行 350-368）
```typescript
let cellIndex
if (!cell_id) {
  cellIndex = 0 // 无 ID 时默认在开头插入
} else {
  // 优先通过 ID 查找
  cellIndex = notebook.cells.findIndex(cell => cell.id === cell_id)
  // ID 未找到时，尝试解析 cell-N 格式索引
  if (cellIndex === -1) {
    const parsedCellIndex = parseCellId(cell_id)
    if (parsedCellIndex !== undefined) {
      cellIndex = parsedCellIndex
    }
  }
  // insert 模式：在找到的位置后插入
  if (originalEditMode === 'insert') {
    cellIndex += 1
  }
}
```

#### 单元格创建逻辑（行 396-428）
```typescript
if (edit_mode === 'delete') {
  notebook.cells.splice(cellIndex, 1)
} else if (edit_mode === 'insert') {
  // 根据 cell_type 创建对应类型的单元格
  if (cell_type === 'markdown') {
    new_cell = { cell_type: 'markdown', id: new_cell_id, source: new_source, metadata: {} }
  } else {
    new_cell = { 
      cell_type: 'code', 
      id: new_cell_id, 
      source: new_source, 
      metadata: {},
      execution_count: null,  // 代码单元格特有
      outputs: []             // 代码单元格特有
    }
  }
  notebook.cells.splice(cellIndex, 0, new_cell)
} else {
  // replace 模式
  targetCell.source = new_source
  if (targetCell.cell_type === 'code') {
    targetCell.execution_count = null
    targetCell.outputs = []
  }
  if (cell_type && cell_type !== targetCell.cell_type) {
    targetCell.cell_type = cell_type
  }
}
```

#### 新单元格 ID 生成（行 381-390）
```typescript
if (notebook.nbformat > 4 || (notebook.nbformat === 4 && notebook.nbformat_minor >= 5)) {
  if (edit_mode === 'insert') {
    new_cell_id = Math.random().toString(36).substring(2, 15)
  } else if (cell_id !== null) {
    new_cell_id = cell_id
  }
}
```

## 依赖与外部交互

### 直接依赖模块

| 模块路径 | 用途 |
|---------|------|
| `src/Tool.ts` | ToolDef 类型定义、buildTool 工厂函数 |
| `src/types/notebook.ts` | NotebookCell、NotebookContent 类型（编译时类型） |
| `src/utils/notebook.ts` | parseCellId 函数（解析 cell-N 格式） |
| `src/utils/file.ts` | getFileModificationTime、writeTextContent |
| `src/utils/fileRead.ts` | readFileSyncWithMetadata |
| `src/utils/fileHistory.ts` | fileHistoryEnabled、fileHistoryTrackEdit |
| `src/utils/json.ts` | safeParseJSON |
| `src/utils/slowOperations.ts` | jsonParse、jsonStringify（带性能监控） |
| `src/utils/lazySchema.ts` | lazySchema（延迟加载 Zod Schema） |
| `src/utils/permissions/filesystem.ts` | checkWritePermissionForTool |
| `./constants.ts` | NOTEBOOK_EDIT_TOOL_NAME |
| `./prompt.ts` | DESCRIPTION、PROMPT |
| `./UI.tsx` | UI 渲染函数 |

### 工具注册与配置

```typescript
export const NotebookEditTool = buildTool({
  name: NOTEBOOK_EDIT_TOOL_NAME,  // 'NotebookEdit'
  searchHint: 'edit Jupyter notebook cells (.ipynb)',
  maxResultSizeChars: 100_000,
  shouldDefer: true,  // 延迟加载工具
  // ... 其他配置
})
```

### 与 FileReadTool 的协作
- NotebookEditTool 要求文件必须先被 FileReadTool 读取（通过 `readFileState` 检查）
- FileReadTool 提供 `readNotebook` 和 `mapNotebookCellsToToolResult` 用于 Notebook 读取

## 风险、边界与改进建议

### 已知风险

1. **JSON 缓存污染风险**
   - 代码中明确注释：必须使用非缓存的 `jsonParse` 而非 `safeParseJSON`
   - 原因：`safeParseJSON` 按内容字符串缓存，返回共享对象引用，而代码会就地修改 Notebook 对象
   - 位置：行 326-330

2. **时间戳精度问题**
   - 使用 `Math.floor(fs.statSync(filePath).mtimeMs)` 确保跨操作的一致性
   - 防止亚毫秒级精度差异导致的误判

3. **UNC 路径安全风险**
   - 明确跳过 UNC 路径（`\\` 或 `//` 开头）的文件系统操作
   - 防止 NTLM 凭证泄漏攻击

### 边界情况处理

| 场景 | 处理方式 |
|-----|---------|
| 文件不存在 | 返回 errorCode 1 |
| 非 .ipynb 文件 | 返回 errorCode 2，建议使用 FileEditTool |
| 无效编辑模式 | 返回 errorCode 4 |
| insert 模式缺少 cell_type | 返回 errorCode 5 |
| 无效 JSON | 返回 errorCode 6 |
| 单元格不存在 | 返回 errorCode 7 或 8 |
| 未先读取文件 | 返回 errorCode 9 |
| 文件已被外部修改 | 返回 errorCode 10 |
| replace 超出末尾 | 自动转换为 insert 模式 |

### 改进建议

1. **单元格 ID 生成增强**
   - 当前使用 `Math.random()` 生成 ID，建议改用更可靠的 UUID 或确定性 ID 生成策略
   - 考虑使用 crypto.randomUUID() 或基于内容的哈希

2. **批量操作支持**
   - 当前仅支持单单元格操作，考虑支持多单元格批量编辑
   - 可减少多次文件 I/O 和 JSON 解析开销

3. **Notebook 格式验证**
   - 当前仅验证 JSON 有效性，建议增加 nbformat 结构验证
   - 可使用 JSON Schema 验证 Notebook 结构完整性

4. **冲突解决机制**
   - 当前仅检测外部修改并拒绝，可考虑提供合并/冲突解决选项
   - 类似于 Git 的三方合并策略

5. **单元格元数据保留**
   - 当前在 replace 时保留 metadata，但可能某些场景需要重置
   - 考虑增加 metadata 处理选项

### 测试关注点

1. **单元格定位准确性**：测试 ID 匹配、索引匹配、cell-N 格式解析
2. **边界条件**：空 Notebook、单单元格、末尾插入
3. **格式兼容性**：nbformat 4.0、4.5、5.0 的不同行为
4. **并发安全**：外部修改检测、时间戳精度
5. **编码和换行符**：保持原文件编码（UTF-8/UTF-16）和换行符风格（LF/CRLF）
