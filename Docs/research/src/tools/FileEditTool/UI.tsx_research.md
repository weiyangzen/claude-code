# UI.tsx 研究文档

## 场景与职责

`UI.tsx` 是 `FileEditTool`（`Edit` 工具）的 React UI 渲染层，负责在终端/REPL 界面中呈现工具使用请求、执行结果、用户拒绝后的 diff 预览，以及各类错误信息的降级展示。该文件与 `src/components/FileEditToolUpdatedMessage.tsx`、`src/components/FileEditToolUseRejectedMessage.tsx`、`src/components/FileEditToolDiff.tsx` 等组件协同工作，构成文件编辑操作的完整可视化链路。

核心职责包括：
- `userFacingName`：根据输入判断操作类型（`Create` / `Update` / `Updated plan`）。
- `getToolUseSummary`：提取并显示文件路径摘要。
- `renderToolUseMessage`：渲染工具调用时的文件路径链接（`FilePathLink`）。
- `renderToolResultMessage`：渲染编辑成功后的结果消息（委托 `FileEditToolUpdatedMessage`）。
- `renderToolUseRejectedMessage`：渲染用户拒绝编辑后的 diff/内容预览（支持新建文件与修改文件两种场景）。
- `renderToolUseErrorMessage`：渲染编辑失败时的降级错误信息（如“File must be read first”、“File not found”）。

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `userFacingName` | 为工具调用提供人类可读的操作名称，用于消息列表标题。 |
| `getToolUseSummary` | 返回被编辑文件的显示路径（相对或绝对），作为摘要。 |
| `renderToolUseMessage` | 在模型发起工具调用时，显示目标文件路径链接。 |
| `renderToolResultMessage` | 编辑完成后展示 diff 统计与结构化 patch（`StructuredDiffList`）。 |
| `renderToolUseRejectedMessage` | 用户拒绝后，仍展示拟编辑内容的 diff 或代码预览，帮助用户理解拒绝了什么。 |
| `renderToolUseErrorMessage` | 将底层错误转化为更温和的终端文案，避免暴露过多技术细节。 |

## 具体技术实现

### 1. React Compiler 缓存模式

整个文件已被 React Compiler（`react/compiler-runtime`）转换，所有函数组件内部使用 `_c(N)` 创建 memo cache，并通过条件比较（如 `$[0] !== filePath`）决定是否重新计算 JSX 节点。这种编译后代码的特点是：
- 大量条件分支用于判断 props/state 是否变化。
- 数组索引 `$[i]` 作为稳定缓存槽位。
- 返回值通常是提前计算好的 React 元素引用，避免无意义重渲染。

### 2. `userFacingName`

逻辑：
- `input == null` → `"Update"`
- `file_path.startsWith(getPlansDirectory())` → `"Updated plan"`
- `input.edits != null`（hashline 编辑）→ `"Update"`
- `old_string === ''` → `"Create"`
- 其他 → `"Update"`

### 3. `renderToolUseMessage`

接收 `file_path` 和 `{ verbose }`：
- 若路径缺失返回 `null`。
- 计划文件（`startsWith(getPlansDirectory())`）返回空字符串，因为路径已在 `userFacingName` 中体现。
- 其他情况返回 `<FilePathLink filePath={file_path}>{verbose ? file_path : getDisplayPath(file_path)}</FilePathLink>`。

### 4. `renderToolResultMessage`

接收 `FileEditOutput` 和 `ProgressMessage[]`，返回：
```tsx
<FileEditToolUpdatedMessage
  filePath={filePath}
  structuredPatch={structuredPatch}
  firstLine={originalFile.split('\n')[0] ?? null}
  fileContent={originalFile}
  style={style}
  verbose={verbose}
  previewHint={isPlanFile ? '/plan to preview' : undefined}
/>
```
- `isPlanFile` 用于在计划文件上显示 `/plan to preview` 提示。

### 5. `renderToolUseRejectedMessage`

这是本文件最复杂的渲染逻辑，分为三种展示形态：

#### A. Hashline / 未知形态
若 `input.edits != null`，直接渲染：
```tsx
<FileEditToolUseRejectedMessage file_path={filePath} operation="update" firstLine={null} verbose={verbose} />
```

#### B. 新建文件（`oldString === ''`）
渲染内容预览而非 diff：
```tsx
<FileEditToolUseRejectedMessage
  file_path={filePath}
  operation="write"
  content={newString}
  firstLine={firstLineOf(newString)}
  verbose={verbose}
/>
```

#### C. 修改文件（默认）
通过内部组件 `EditRejectionDiff` 异步加载 diff 数据：
- 使用 `useState` 懒初始化一个 Promise：`() => loadRejectionDiff(filePath, oldString, newString, replaceAll)`。
- 外层用 `<Suspense fallback={...}>` 包裹，fallback 为静态拒绝消息骨架。
- 实际渲染由 `EditRejectionBody` 通过 `use(promise)` 解包后完成。

### 6. `loadRejectionDiff` 异步加载逻辑

```ts
async function loadRejectionDiff(
  filePath: string,
  oldString: string,
  newString: string,
  replaceAll: boolean,
): Promise<RejectionDiffData>
```

流程：
1. 调用 `readEditContext(filePath, oldString, CONTEXT_LINES)` 进行**分块读取**（8KB chunk），定位第一次出现 `oldString` 的上下文窗口。
2. 若 `ctx === null`（ENOENT）、`ctx.truncated`（未在 10MB 扫描上限内找到）或 `content === ''`，则回退到“仅 diff 工具输入”模式：
   - 以 `oldString` 作为整个 `fileContents`，调用 `getPatchForEdit` 生成 patch。
   - 返回 `{ patch, firstLine: null, fileContent: undefined }`。
3. 若成功找到上下文：
   - `findActualString(ctx.content, oldString)` 做引号归一化匹配。
   - `preserveQuoteStyle` 保持文件引号风格。
   - `getPatchForEdit` 基于上下文切片生成 patch。
   - `adjustHunkLineNumbers(patch, ctx.lineOffset - 1)` 将切片相对行号转换为文件绝对行号。
   - `firstLine` 仅在 `ctx.lineOffset === 1` 时取 `firstLineOf(ctx.content)`，否则为 `null`（避免展示截断文件的首行）。
4. 任何异常（如用户手动应用了变更导致 diff 失败）被 `catch` 后返回空 patch，避免崩溃。

### 7. `renderToolUseErrorMessage`

对常见错误进行文案降级：
- `"File has not been read yet"` → `<Text dimColor>File must be read first</Text>`
- 包含 `FILE_NOT_FOUND_CWD_NOTE` 的错误 → `<Text color="error">File not found</Text>`
- 其他非 verbose 模式 → `<Text color="error">Error editing file</Text>`
- verbose 模式或无法识别 → 回退到 `<FallbackToolUseErrorMessage />`

## 关键代码路径与文件引用

- **组件依赖**：
  - `src/components/FileEditToolUpdatedMessage.js`（结果展示）
  - `src/components/FileEditToolUseRejectedMessage.js`（拒绝展示）
  - `src/components/FilePathLink.js`（可点击路径）
  - `src/components/FallbackToolUseErrorMessage.js`（通用错误回退）
  - `src/components/MessageResponse.js`（消息容器）
- **工具函数**：
  - `src/utils/diff.js`：`adjustHunkLineNumbers`、`CONTEXT_LINES`
  - `src/utils/file.js`：`FILE_NOT_FOUND_CWD_NOTE`、`getDisplayPath`
  - `src/utils/plans.js`：`getPlansDirectory`
  - `src/utils/readEditContext.js`：`readEditContext`
  - `src/utils/stringUtils.js`：`firstLineOf`
- **同目录工具**：
  - `./types.js`：`FileEditOutput`
  - `./utils.js`：`findActualString`、`getPatchForEdit`、`preserveQuoteStyle`
- **调用方**：
  - `FileEditTool.ts` 在 `buildTool` 中直接引用本文件的全部渲染函数。
  - `src/tools/BashTool/BashTool.tsx` 引用 `userFacingName` 作为别名逻辑参考。

## 依赖与外部交互

| 依赖模块 | 交互方式 | 说明 |
|----------|----------|------|
| `FileEditToolUpdatedMessage` | JSX 组件 | 成功结果的结构化 diff + 统计展示。 |
| `FileEditToolUseRejectedMessage` | JSX 组件 | 拒绝后的代码/diff 预览。 |
| `FilePathLink` | JSX 组件 | 工具调用时的可点击文件路径。 |
| `readEditContext` | 异步函数 | 拒绝 diff 的分块文件读取，避免加载超大文件。 |
| `getPatchForEdit` / `findActualString` / `preserveQuoteStyle` | 函数调用 | 生成准确的 diff patch 并处理引号风格。 |
| `useTerminalSize` | React Hook | 获取终端宽度，用于 `StructuredDiffList` 自适应布局。 |

## 风险、边界与改进建议

### 风险与边界

1. **React Compiler 生成代码的可维护性**：源码经过编译器转换后，充斥着 `_c(N)`、`$[i]` 等机器生成模式，人工阅读和调试成本极高。任何手动修改都极易破坏 memo 语义。
2. **`loadRejectionDiff` 的异步异常静默**：当用户手动应用了变更，diff 生成可能失败，此时返回空 patch，用户看到的拒绝消息将缺少 diff 细节，体验降级。
3. **分块读取的行号偏移计算**：`adjustHunkLineNumbers(patch, ctx.lineOffset - 1)` 假设 patch 内部行号与切片起点一致。若 `getPatchForEdit` 内部逻辑变化（如处理 `replace_all` 多匹配时），行号对齐可能出错。
4. **`firstLine` 的语义限制**：仅当 `ctx.lineOffset === 1` 时才展示首行，否则为 `null`。这在某些边缘场景（如文件极短、上下文从第 2 行开始）可能导致 `StructuredDiffList` 缺少文件头信息。
5. **`edits` 字段的防御性处理**：`renderToolUseRejectedMessage` 对 `input.edits` 做了特殊处理，但 `FileEditTool.ts` 的输入 schema 实际上并不包含 `edits` 字段（那是 `FileEditToolDiff` 组件或未来 hashline 编辑的预留）。这种不一致增加了理解成本。

### 改进建议

1. **保留原始 TSX 源码与编译产物分离**：当前仓库中 `.tsx` 文件似乎已经是编译后产物（包含 source map）。建议建立清晰的源码 → 编译产物工作流，避免在版本控制中直接维护编译后代码。
2. **为 `loadRejectionDiff` 增加重试与更细粒度错误提示**：当 diff 失败时，可尝试仅展示 `oldString`/`newString` 的文本对比，而不是完全空 patch。
3. **统一 `edits` 与 `old_string/new_string` 的 UI 处理**：若 hashline 编辑尚未正式启用，应移除或注释掉 `edits` 分支，减少死代码；若已启用，应确保类型定义与 schema 同步。
4. **引入 Skeleton/Placeholder 的渐进加载体验**：`Suspense fallback` 当前仅显示静态文本，可考虑展示简化版 diff 骨架（如文件路径 + “Loading diff…”），提升感知性能。
5. **将 `CONTEXT_LINES` 与 `MAX_LINES_TO_RENDER` 等常量抽离为工具级配置**：方便后续根据终端尺寸或用户偏好动态调整上下文行数。
