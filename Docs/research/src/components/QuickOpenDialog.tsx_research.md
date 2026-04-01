# QuickOpenDialog.tsx 研究文档

## 场景与职责

`QuickOpenDialog.tsx` 是 Claude Code 的快速文件打开对话框组件，实现了类似 VS Code "Quick Open"（Ctrl+P）的功能。该组件在以下场景触发：

1. **快捷键触发**: 用户按下 `Ctrl+Shift+P` / `Cmd+Shift+P`（可配置）
2. **命令触发**: 用户输入 `/quick-open` 或相关命令
3. **文件引用**: 在对话中快速引用文件时使用

组件的核心职责：
- 提供模糊搜索界面查找项目文件
- 实时预览选中文件的内容
- 支持多种操作：在编辑器打开、插入路径、@提及文件
- 智能布局（根据终端宽度调整预览位置）

## 功能点目的

### 1. 模糊文件搜索
集成 `fileSuggestions.ts` 提供的文件索引：
```typescript
generateFileSuggestions(q, true).then(items => {
  const paths = items
    .filter(i => i.id.startsWith("file-"))
    .map(i => i.displayText)
    .filter(p => !p.endsWith(path.sep))
    .map(p => p.split(path.sep).join("/"));
  setResults(paths);
});
```

特点：
- 实时搜索（输入即搜索）
- 使用 Rust 实现的文件索引（通过 `FileIndex`）
- 支持模糊匹配算法

### 2. 智能预览系统
根据终端宽度决定预览布局：

**宽终端（≥120 列）**: 预览在右侧
```
[文件列表 (40%)] [预览 (60%)]
```

**窄终端（<120 列）**: 预览在底部
```
[文件列表]
[预览区域]
```

预览内容：
- 文件路径（中间截断）
- 前 20 行内容（语法高亮）
- 搜索词高亮显示

### 3. 多操作支持
用户可以通过不同按键执行不同操作：

| 操作 | 按键 | 功能 |
|------|------|------|
| 打开 | Enter | 在外部编辑器打开文件 |
| 提及 | Tab | 插入 `@filepath ` 到输入框 |
| 插入路径 | Shift+Tab | 插入 `filepath ` 到输入框 |
| 取消 | Esc | 关闭对话框 |

### 4. 防抖与取消机制
使用 `AbortController` 管理并发请求：
```typescript
const controller = new AbortController();
readFileInRange(absolute, 0, effectivePreviewLines, undefined, controller.signal)
  .then(r => { /* 更新预览 */ })
  .catch(() => { /* 预览不可用 */ });

// 切换文件时取消上一个请求
return () => controller.abort();
```

## 具体技术实现

### 关键流程

#### 初始化流程
```
1. 组件挂载
2. 注册 overlay 状态 (useRegisterOverlay)
3. 获取终端尺寸 (useTerminalSize)
4. 计算可见结果数：Math.min(VISIBLE_RESULTS, Math.max(4, rows - 14))
5. 初始化空结果列表和空查询
```

#### 搜索流程
```
1. 用户输入查询
2. 更新 query 状态
3. 递增 queryGenRef（用于取消旧请求）
4. 如果查询为空，清空结果
5. 调用 generateFileSuggestions(query, true)
6. 过滤 file- 类型的结果
7. 转换路径分隔符为 /
8. 更新 results 状态
```

#### 预览加载流程
```
1. 用户聚焦某个文件（onFocus）
2. 设置 focusedPath 状态
3. useEffect 触发预览加载
4. 创建 AbortController
5. 解析绝对路径
6. 调用 readFileInRange(absolute, 0, effectivePreviewLines)
7. 成功后更新 preview 状态
8. 组件卸载或路径变化时取消请求
```

#### 文件打开流程
```
1. 用户按 Enter 选择文件
2. 调用 openFileInExternalEditor(path.resolve(getCwd(), filepath))
3. 记录分析事件 tengu_quick_open_select
4. 调用 onDone() 关闭对话框
```

### 数据结构

#### Props 定义
```typescript
type Props = {
  onDone: () => void;              // 对话框关闭回调
  onInsert: (text: string) => void; // 插入文本回调（用于 @提及/路径插入）
};
```

#### 状态定义
```typescript
const [results, setResults] = useState<string[]>([]);        // 搜索结果
const [query, setQuery] = useState("");                      // 搜索查询
const [focusedPath, setFocusedPath] = useState<string>();    // 当前聚焦路径
const [preview, setPreview] = useState<PreviewState>(null);  // 预览内容
const queryGenRef = useRef(0);                               // 请求世代计数器
```

#### 预览状态
```typescript
type PreviewState = {
  path: string;      // 预览的文件路径
  content: string;   // 文件内容（前 N 行）
} | null;
```

### 布局计算
```typescript
// 是否右侧预览
const previewOnRight = columns >= 120;

// 预览行数
const effectivePreviewLines = previewOnRight 
  ? VISIBLE_RESULTS - 1  // 右侧：较少行数
  : PREVIEW_LINES;       // 底部：20 行

// 路径显示宽度
const maxPathWidth = previewOnRight 
  ? Math.max(20, Math.floor((columns - 10) * 0.4))
  : Math.max(20, columns - 8);

// 预览区域宽度
const previewWidth = previewOnRight 
  ? Math.max(40, columns - maxPathWidth - 14)
  : columns - 6;
```

## 关键代码路径与文件引用

### 本文件关键代码
| 行号 | 功能 |
|------|------|
| 17-20 | Props 类型定义 |
| 21-22 | 常量定义（VISIBLE_RESULTS, PREVIEW_LINES） |
| 28-225 | QuickOpenDialog 主组件 |
| 34 | useRegisterOverlay 注册 |
| 39 | 可见结果数计算 |
| 51 | queryGenRef 创建 |
| 71-90 | handleQueryChange 搜索处理 |
| 93-129 | 预览加载 useEffect |
| 133-147 | handleOpen 打开处理 |
| 150-166 | handleInsert 插入处理 |
| 208-224 | FuzzyPicker 渲染 |

### 依赖文件引用

| 导入路径 | 用途 |
|----------|------|
| `../context/overlayContext.js` | useRegisterOverlay |
| `../hooks/fileSuggestions.js` | generateFileSuggestions |
| `../hooks/useTerminalSize.js` | useTerminalSize |
| `../ink.js` | Text 组件 |
| `../services/analytics/index.js` | logEvent |
| `../utils/cwd.js` | getCwd |
| `../utils/editor.js` | openFileInExternalEditor |
| `../utils/format.js` | truncatePathMiddle, truncateToWidth |
| `../utils/highlightMatch.js` | highlightMatch |
| `../utils/readFileInRange.js` | readFileInRange |
| `./design-system/FuzzyPicker.js` | FuzzyPicker 组件 |
| `./design-system/LoadingState.js` | LoadingState 组件 |

### 依赖的依赖

```
QuickOpenDialog.tsx
├── fileSuggestions.ts
│   ├── FileIndex (Rust 实现的模糊搜索索引)
│   ├── git ls-files (获取 tracked 文件)
│   └── ripgrep (fallback 文件搜索)
├── readFileInRange.ts
│   ├── 快速路径：readFile + 内存分割 (<10MB)
│   └── 流式路径：createReadStream (大文件)
├── editor.ts
│   └── 外部编辑器检测和启动
└── FuzzyPicker.ts
    └── 模糊选择 UI 组件
```

## 依赖与外部交互

### 外部依赖
1. **React Compiler Runtime**: 自动记忆化
2. **Ink**: 终端 UI 组件
3. **Node.js path**: 路径处理

### 文件索引系统
`generateFileSuggestions` 使用混合策略：

1. **Git 优先**: 在 Git 仓库中使用 `git ls-files`
   - 速度快（读取索引）
   - 自动尊重 .gitignore
   - 包含子模块

2. **Ripgrep 回退**: 非 Git 项目使用 `rg --files`
   - 跨平台文件搜索
   - 可配置忽略模式

3. **后台刷新**: 文件列表在后台定期更新
   - 检测新文件（untracked）
   - 5 秒节流

### 分析事件
```typescript
// 打开文件时
logEvent("tengu_quick_open_select", {
  result_count: results.length,
  opened_editor: opened  // 是否成功打开编辑器
});

// 插入路径时
logEvent("tengu_quick_open_insert", {
  result_count: results.length,
  mention  // 是否是 @提及
});
```

### 外部编辑器集成
`openFileInExternalEditor` 支持：
- **GUI 编辑器**: VS Code、Cursor、Sublime 等（分离式启动）
- **终端编辑器**: vim、nano 等（alt-screen 切换）
- **自动检测**: 通过 `$VISUAL`、`$EDITOR` 或默认配置

## 风险、边界与改进建议

### 已知风险

1. **大文件预览**
   - `readFileInRange` 有 10MB 快速路径限制
   - 超过 10MB 使用流式读取，但预览仍可能慢
   - 二进制文件可能产生乱码预览

2. **搜索延迟**
   - `generateFileSuggestions` 可能阻塞 UI
   - 在 100k+ 文件的项目中可能明显

3. **并发请求竞争**
   ```typescript
   // 虽然有 queryGenRef，但以下情况仍可能有问题：
   if (gen !== queryGenRef.current) {
     return;  // 取消旧请求结果
   }
   ```
   - 如果请求完成顺序错乱，可能显示错误结果

4. **预览取消**
   - `AbortController` 在文件读取开始后取消
   - 但 `readFileInRange` 内部可能不立即响应取消信号

### 边界情况

1. **空查询**
   - 查询为空时显示空结果
   - 不显示任何文件（不同于 VS Code 显示最近文件）

2. **无结果**
   - 显示 "No matching files"
   - 用户可继续输入或取消

3. **终端尺寸变化**
   - 使用 `useTerminalSize` 监听变化
   - 布局自动调整，但预览内容不重新加载

4. **特殊字符路径**
   - 路径中的特殊字符可能影响显示
   - `truncatePathMiddle` 处理截断

5. **权限不足文件**
   - `readFileInRange` 可能抛出权限错误
   - 显示 "(preview unavailable)"

### 改进建议

1. **最近文件列表**
   ```typescript
   // 空查询时显示最近打开的文件
   const [recentFiles, setRecentFiles] = useState(() => 
     getGlobalConfig().recentQuickOpenFiles || []
   );
   ```

2. **文件类型图标**
   - 根据扩展名显示文件类型图标
   - 增强视觉识别

3. **预览语法高亮增强**
   - 当前使用简单的高亮
   - 可集成 tree-sitter 提供更准确的语法高亮

4. **搜索历史**
   - 保存搜索历史，支持上下箭头浏览
   - 类似 shell 历史功能

5. **文件夹导航**
   - 支持输入文件夹路径后按 `/` 进入
   - 类似模糊查找器的文件夹导航

6. **性能优化**
   ```typescript
   // 添加防抖，避免每输入一个字符都搜索
   const debouncedQuery = useDebounce(query, 50);
   
   useEffect(() => {
     if (debouncedQuery) {
       handleSearch(debouncedQuery);
     }
   }, [debouncedQuery]);
   ```

7. **二进制文件检测**
   - 检测二进制文件，显示 "Binary file" 而非乱码
   - 可使用 `is-binary-path` 或类似库

8. **预览行号**
   - 在预览中显示行号
   - 帮助用户定位内容

9. **多选支持**
   - 支持选择多个文件批量打开
   - 使用 Space 键切换选择

10. **模糊匹配高亮**
    - 当前只高亮精确匹配的查询词
    - 改进模糊匹配算法的高亮显示

### 测试建议

1. **单元测试**
   - 布局计算逻辑
   - 路径处理函数

2. **集成测试**
   - 文件搜索端到端流程
   - 预览加载和取消

3. **性能测试**
   - 大项目（100k+ 文件）搜索性能
   - 大文件预览性能

4. **手动测试场景**
   - 极窄终端（<80 列）
   - 极高终端（>200 列）
   - 包含特殊字符的文件名
   - 无权限文件

### 相关文件
- `src/hooks/fileSuggestions.ts`: 文件搜索核心逻辑
- `src/utils/readFileInRange.ts`: 文件读取优化
- `src/utils/editor.ts`: 外部编辑器集成
- `src/components/design-system/FuzzyPicker.tsx`: 选择器 UI
- `src/utils/highlightMatch.tsx`: 搜索高亮
