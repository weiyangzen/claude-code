# FileEditToolDiff.tsx 研究文档

## 场景与职责

`FileEditToolDiff` 是 Claude Code 中用于**文件编辑权限请求界面**的核心差异展示组件。当 AI 需要编辑文件并请求用户确认时，该组件负责渲染一个可视化的 diff 视图，让用户在批准前能够清楚地看到将要做的修改。

主要使用场景：
1. **文件编辑权限请求** (`FileEditPermissionRequest`) - 显示 AI 提议的编辑内容
2. **Sed 编辑权限请求** (`SedEditPermissionRequest`) - 显示批量替换操作的差异
3. **用户拒绝后的 diff 展示** - 在 `FileEditTool/UI.tsx` 中用于展示被拒绝的编辑

## 功能点目的

### 1. 异步 Diff 数据加载
- 使用 React 的 `useState` + `Suspense` 模式实现异步数据加载
- 避免在渲染时阻塞 UI，提供流畅的用户体验
- 加载期间显示占位符 `DiffFrame`（显示省略号）

### 2. 智能文件读取策略
组件实现了多种文件读取策略以优化性能和内存使用：

| 策略 | 适用场景 | 说明 |
|------|----------|------|
| **整块读取** (`readCapped`) | 小文件或空 `old_string` | 读取完整文件内容用于 diff |
| **分块扫描** (`scanForContext`) | 大文件中的单次编辑 | 只读取匹配位置周围的上下文 |
| **仅 diff 工具输入** (`diffToolInputsOnly`) | 超大文件或匹配失败 | 直接对比 `old_string` 和 `new_string` |

### 3. 大文件优化
- 当 `old_string` 长度超过 `CHUNK_SIZE` (8KB) 时，跳过文件读取，直接 diff 输入内容
- 避免为大文件分配大量内存用于重叠缓冲区
- 扫描上限 `MAX_SCAN_BYTES` (10MB) 防止无限制读取

### 4. 行号调整
- 使用 `adjustHunkLineNumbers` 将切片相对行号转换为文件绝对行号
- 确保 diff 显示的行号与实际文件行号一致

## 具体技术实现

### 关键数据结构

```typescript
// 组件 Props
type Props = {
  file_path: string;    // 目标文件的绝对路径
  edits: FileEdit[];    // 编辑操作数组
};

// Diff 数据类型
type DiffData = {
  patch: StructuredPatchHunk[];  // diff 块数组
  firstLine: string | null;      // 文件第一行（用于 shebang 检测）
  fileContent: string | undefined; // 文件内容（可能未加载）
};

// FileEdit 类型（来自 FileEditTool/types.js）
type FileEdit = {
  old_string: string;   // 要替换的文本
  new_string: string;   // 新文本
  replace_all: boolean; // 是否替换所有匹配项
};
```

### 核心流程

```
FileEditToolDiff(props)
├── 创建 dataPromise (useState 延迟初始化)
├── 渲染 Suspense + DiffFrame(placeholder)
└── DiffBody (异步)
    ├── use(promise) 等待数据
    ├── useTerminalSize() 获取终端宽度
    └── 渲染 StructuredDiffList
```

### 数据加载流程 (`loadDiffData`)

```typescript
async function loadDiffData(file_path: string, edits: FileEdit[]): Promise<DiffData> {
  // 1. 过滤无效编辑
  const valid = edits.filter(e => e.old_string != null && e.new_string != null);
  
  // 2. 超大 old_string 优化路径
  if (single && single.old_string.length >= CHUNK_SIZE) {
    return diffToolInputsOnly(file_path, [single]);
  }
  
  // 3. 尝试打开文件
  const handle = await openForScan(file_path);
  if (handle === null) return diffToolInputsOnly(file_path, valid);
  
  try {
    // 4. 多编辑或空 old_string：需要完整文件
    if (!single || single.old_string === '') {
      const file = await readCapped(handle);
      // ... 生成完整文件 diff
    }
    
    // 5. 单次编辑：扫描上下文
    const ctx = await scanForContext(handle, single.old_string, CONTEXT_LINES);
    // ... 生成上下文切片 diff
  } finally {
    await handle.close();
  }
}
```

### 编辑规范化 (`normalizeEdit`)

```typescript
function normalizeEdit(fileContent: string, edit: FileEdit): FileEdit {
  // 1. 查找实际匹配的字符串（处理引号规范化）
  const actualOld = findActualString(fileContent, edit.old_string) || edit.old_string;
  
  // 2. 保留文件中的引号风格
  const actualNew = preserveQuoteStyle(edit.old_string, actualOld, edit.new_string);
  
  return { ...edit, old_string: actualOld, new_string: actualNew };
}
```

## 关键代码路径与文件引用

### 本文件关键函数

| 函数 | 行号 | 职责 |
|------|------|------|
| `FileEditToolDiff` | 23-52 | 主组件，管理 promise 和 Suspense |
| `DiffBody` | 53-80 | 异步渲染主体，使用 `use()` 解包 promise |
| `DiffFrame` | 81-105 | 边框容器，支持 placeholder 模式 |
| `loadDiffData` | 106-160 | 核心数据加载逻辑 |
| `diffToolInputsOnly` | 161-171 | 仅基于输入生成 diff（无文件读取） |
| `normalizeEdit` | 172-180 | 编辑规范化（引号处理） |

### 依赖文件

| 文件路径 | 用途 |
|----------|------|
| `src/utils/diff.js` | `getPatchForDisplay`, `adjustHunkLineNumbers`, `CONTEXT_LINES` |
| `src/utils/readEditContext.js` | `openForScan`, `readCapped`, `scanForContext`, `CHUNK_SIZE` |
| `src/tools/FileEditTool/utils.js` | `findActualString`, `preserveQuoteStyle` |
| `src/tools/FileEditTool/types.js` | `FileEdit` 类型定义 |
| `src/utils/stringUtils.js` | `firstLineOf` |
| `src/utils/log.js` | `logError` |
| `src/hooks/useTerminalSize.js` | 终端尺寸获取 |
| `src/components/StructuredDiffList.js` | diff 渲染组件 |

### 调用方文件

| 文件路径 | 使用场景 |
|----------|----------|
| `src/components/permissions/FileEditPermissionRequest/FileEditPermissionRequest.tsx` | 文件编辑权限请求 |
| `src/components/permissions/SedEditPermissionRequest/SedEditPermissionRequest.tsx` | Sed 编辑权限请求 |
| `src/tools/FileEditTool/UI.tsx` | 被拒绝的编辑 diff 展示 |

## 依赖与外部交互

### React 特性使用

1. **React Compiler 优化**: 代码经过 React Compiler 编译，使用 `_c(n)` 进行缓存优化
2. **Suspense + use**: 使用 React 18+ 的 `use()` API 处理异步数据
3. **useState 延迟初始化**: `useState(() => loadDiffData(...))` 避免重复创建 promise

### 文件系统交互

通过 `readEditContext.js` 提供的函数进行文件操作：
- `openForScan`: 以只读模式打开文件，ENOENT 返回 null
- `readCapped`: 读取整个文件（上限 10MB）
- `scanForContext`: 8KB 分块扫描，查找匹配位置并提取上下文

### Diff 生成

使用 `diff` 库的 `structuredPatch` 函数（通过 `getPatchForDisplay` 封装）：
- 自动处理 `&` 和 `$` 字符的转义（diff 库的 bug workaround）
- 将前导制表符转换为空格用于显示
- 3 行上下文 (`CONTEXT_LINES = 3`)
- 5 秒超时 (`DIFF_TIMEOUT_MS = 5000`)

## 风险、边界与改进建议

### 已知风险

1. **大文件处理**
   - 超过 10MB 的文件会被截断处理
   - 超过 8KB 的 `old_string` 会直接 diff 输入，可能丢失上下文

2. **多编辑场景**
   - 多编辑操作需要完整文件内容
   - 大文件上的多编辑可能导致内存压力

3. **编码问题**
   - 假设文件为 UTF-8 编码
   - 二进制文件可能产生乱码

### 边界情况

| 场景 | 处理方式 |
|------|----------|
| 文件不存在 | 回退到 `diffToolInputsOnly` |
| 匹配失败 | 回退到 `diffToolInputsOnly` |
| 扫描超时/截断 | 回退到 `diffToolInputsOnly` |
| 空 `old_string` | 读取完整文件，视为新内容插入 |
| `replace_all` | 在上下文窗口内显示首次匹配 |

### 改进建议

1. **性能优化**
   - 考虑添加文件内容缓存，避免短时间内重复读取同一文件
   - 对于频繁编辑的文件，可维护内存缓存

2. **用户体验**
   - 添加加载进度指示（当前仅显示省略号）
   - 对于超大文件，显示警告提示用户正在显示部分差异

3. **错误处理**
   - 当前错误仅记录到日志，可考虑在 UI 中显示友好的错误信息
   - 区分"文件不存在"和"匹配失败"的不同提示

4. **代码结构**
   - `loadDiffData` 函数较长，可拆分为更小的策略函数
   - 考虑使用策略模式替代条件分支

5. **测试覆盖**
   - 需要针对大文件、多编码、特殊字符等场景的单元测试
   - 建议添加性能基准测试
