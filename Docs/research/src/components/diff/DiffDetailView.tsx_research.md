# DiffDetailView.tsx 研究文档

## 场景与职责

`DiffDetailView` 是 Claude Code 中用于显示单个文件 diff 内容的展示组件。它是 diff 查看功能的核心渲染组件，负责：

1. **单个文件 diff 内容展示** - 接收文件的 hunks 数据，渲染结构化的 diff 视图
2. **特殊文件状态处理** - 处理未跟踪文件(untracked)、二进制文件(binary)、大文件(large file)等特殊情况
3. **文件内容读取与语法高亮支持** - 读取原始文件内容，为 `StructuredDiff` 组件提供语法高亮所需的上下文
4. **终端宽度适配** - 响应终端尺寸变化，动态调整渲染宽度

该组件在 `DiffDialog` 的 detail 视图模式下被使用，当用户从文件列表中选择某个文件后，通过此组件查看具体的代码变更。

## 功能点目的

### 1. 文件内容读取与首行提取
- **目的**：为语法高亮引擎提供文件首行内容（用于 shebang 检测）和完整文件内容（用于多行字符串等上下文感知）
- **实现**：使用 `readFileSafe` 工具函数安全读取文件，提取第一行用于语言检测

### 2. 多状态文件展示
- **未跟踪文件(untracked)**：显示提示信息，引导用户使用 `git add` 查看行数统计
- **二进制文件(binary)**：显示"Binary file - cannot display diff"提示
- **大文件(large file)**：显示"Large file - diff exceeds 1 MB limit"提示
- **截断文件(truncated)**：在标题旁显示"(truncated)"标记，并在底部显示截断提示

### 3. Diff Hunks 渲染
- 使用 `StructuredDiff` 组件进行词级 diff 和语法高亮
- 支持最多 400 行解析限制（与 gitDiff.ts 中的 `MAX_LINES_PER_FILE` 保持一致）
- 无滚动设计 - 渲染所有行（受限于解析上限）

## 具体技术实现

### Props 接口定义

```typescript
type Props = {
  filePath: string;           // 文件路径（用于展示和语法检测）
  hunks: StructuredPatchHunk[]; // diff 的 hunk 数组
  isLargeFile?: boolean;      // 是否为大文件（>1MB）
  isBinary?: boolean;         // 是否为二进制文件
  isTruncated?: boolean;      // 是否被截断（>400行）
  isUntracked?: boolean;      // 是否为未跟踪文件
};
```

### 核心数据结构

**StructuredPatchHunk**（来自 'diff' 库）：
```typescript
interface StructuredPatchHunk {
  oldStart: number;    // 旧文件起始行号
  oldLines: number;    // 旧文件行数
  newStart: number;    // 新文件起始行号
  newLines: number;    // 新文件行数
  lines: string[];     // 行内容数组（以 +/-/space 开头）
}
```

### 关键渲染流程

1. **文件内容准备阶段**（useMemo 优化）：
   - 解析 `filePath` 为绝对路径：`resolve(getCwd(), filePath)`
   - 调用 `readFileSafe(fullPath)` 读取文件内容
   - 提取首行：`content?.split("\n")[0]`

2. **状态分支渲染**：
   - 优先级顺序：`isUntracked` → `isBinary` → `isLargeFile` → 正常 diff
   - 每个分支返回预定义的 UI 结构

3. **正常 Diff 渲染**：
   - 文件路径标题（bold）+ 截断标记（如适用）
   - `Divider` 分隔线（padding=4）
   - 条件渲染：
     - `hunks.length === 0`：显示 "No diff content"
     - 否则：遍历 hunks，为每个 hunk 渲染 `StructuredDiff` 组件
   - 截断提示（如 `isTruncated` 为 true）

### React Compiler 优化

代码经过 React Compiler（19+）编译，使用 `_c(n)` 创建 memoization cache：
- 每个条件分支和 JSX 元素都有独立的 cache slot
- 使用 `Symbol.for("react.memo_cache_sentinel")` 作为初始标记
- 通过比较依赖项决定是否复用缓存的 JSX 元素

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 用途 |
|---------|------|
| `src/components/StructuredDiff.tsx` | 核心 diff 渲染组件，提供语法高亮和词级 diff |
| `src/components/design-system/Divider.tsx` | 水平分隔线组件 |
| `src/hooks/useTerminalSize.ts` | 获取终端尺寸（columns）用于宽度计算 |
| `src/utils/file.ts` | `readFileSafe` 安全文件读取 |
| `src/utils/cwd.ts` | `getCwd` 获取当前工作目录 |
| `src/ink.ts` | Ink 组件（Box, Text） |

### 被调用方

| 文件路径 | 调用场景 |
|---------|---------|
| `src/components/diff/DiffDialog.tsx` | DiffDialog 在 viewMode='detail' 时渲染 DiffDetailView |

### 类型依赖

| 来源 | 类型 |
|-----|------|
| `diff` 包 | `StructuredPatchHunk` |

## 依赖与外部交互

### 文件系统交互
- **读取操作**：通过 `readFileSafe` 读取文件内容用于语法高亮
- **路径解析**：使用 Node.js `path.resolve` 结合 `getCwd()` 解析相对路径

### 与 StructuredDiff 的协作
`DiffDetailView` 作为容器组件，`StructuredDiff` 作为展示组件：
- `DiffDetailView` 负责：文件 I/O、状态判断、布局包裹
- `StructuredDiff` 负责：语法高亮、diff 算法、ANSI 渲染

参数传递：
```typescript
<StructuredDiff 
  key={index}
  patch={hunk}
  filePath={filePath}
  firstLine={firstLine}
  fileContent={fileContent}
  dim={false}
  width={columns - 2 - 2}  // 终端宽度减去边距
/>
```

### 终端尺寸响应
通过 `useTerminalSize` hook 监听终端宽度变化，动态调整 `StructuredDiff` 的 `width` 属性。

## 风险、边界与改进建议

### 已知风险

1. **文件读取失败**：
   - `readFileSafe` 在读取失败时返回 `null`，组件已处理此情况
   - 风险：文件在 diff 生成后被删除或权限变更

2. **大文件内存占用**：
   - 虽然组件本身不直接限制文件大小，但依赖的 `readFileSafe` 会读取完整文件内容
   - 风险：超大文件（如几百 MB）可能导致内存压力

3. **终端宽度计算**：
   - `width={columns - 2 - 2}` 的硬编码边距假设可能与其他组件不一致
   - 风险：布局偏移或内容截断

### 边界情况

| 场景 | 处理行为 |
|-----|---------|
| `filePath` 为空字符串 | 提前返回，不渲染任何内容 |
| `hunks` 为空数组 | 显示 "No diff content" |
| 文件读取失败 | `fileContent` 为 `undefined`，`firstLine` 为 `null` |
| 终端宽度极窄 | `width` 可能为负数，但 `StructuredDiff` 内部有 `Math.max(1, ...)` 保护 |

### 改进建议

1. **性能优化**：
   - 考虑对 `fileContent` 进行大小限制，避免读取超大文件
   - 对于仅需要首行 shebang 检测的场景，可以只读取文件头部

2. **可访问性**：
   - 当前使用颜色区分添加/删除，建议增加符号指示器（+/-）的冗余提示

3. **代码结构**：
   - 特殊状态（untracked/binary/large）的渲染逻辑可以提取为独立子组件
   - 减少编译后代码的重复模式（React Compiler 生成的 cache 检查代码）

4. **错误处理**：
   - 增加文件读取失败的显式错误提示，而非静默忽略
   - 考虑添加文件路径不存在时的降级展示

5. **类型安全**：
   - 当前 Props 使用可选标记（`?`），建议明确哪些字段是互斥的（如 isBinary 和 isLargeFile 不应同时为 true）
