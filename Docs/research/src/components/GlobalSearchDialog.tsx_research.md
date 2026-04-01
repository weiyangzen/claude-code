# GlobalSearchDialog.tsx 研究文档

## 场景与职责

GlobalSearchDialog 是 Claude Code CLI 的全局搜索对话框组件，对应快捷键 `Ctrl+Shift+F` / `Cmd+Shift+F`。它提供跨工作区的文本搜索能力，基于 ripgrep 实现高性能文件内容检索。

**核心职责：**
- 提供交互式全局搜索界面，支持实时模糊匹配
- 集成 ripgrep 进行高效的跨文件内容搜索
- 支持搜索结果预览和快速跳转
- 提供多种结果操作（打开文件、插入路径/Mention）
- 与外部编辑器集成，支持直接打开文件到指定行

## 功能点目的

### 1. 全局搜索界面
- **输入框**：实时输入搜索关键词，支持防抖处理（100ms）
- **结果列表**：显示匹配的文件路径、行号和匹配文本
- **预览面板**：选中项时异步加载文件内容预览（上下4行上下文）
- **响应式布局**：根据终端宽度自动调整布局（≥140列时预览在右侧）

### 2. 搜索功能
- **防抖搜索**：避免频繁触发 ripgrep，提升性能
- **结果限制**：
  - 每文件最多10个匹配（`MAX_MATCHES_PER_FILE`）
  - 总计最多500个匹配（`MAX_TOTAL_MATCHES`）
  - 超过限制时显示截断提示
- **增量过滤**：在已有结果中快速过滤，减少重复搜索

### 3. 结果操作
- **Enter**：在外部编辑器中打开文件并定位到匹配行
- **Tab**：插入 Mention 格式（`@file#Lline `）
- **Shift+Tab**：插入路径格式（`file:line `）

### 4. 预览功能
- 异步读取选中匹配项的文件内容
- 显示匹配行上下4行上下文
- 高亮显示匹配关键词

## 具体技术实现

### 关键数据结构

```typescript
// 匹配项结构
type Match = {
  file: string;    // 文件路径（相对或绝对）
  line: number;    // 行号
  text: string;    // 匹配行文本
};

// 预览内容结构
type Preview = {
  file: string;
  line: number;
  content: string;  // 文件内容（多行）
};
```

### 关键常量

```typescript
const VISIBLE_RESULTS = 12;           // 可见结果数量
const DEBOUNCE_MS = 100;              // 搜索防抖时间
const PREVIEW_CONTEXT_LINES = 4;      // 预览上下文行数
const MAX_MATCHES_PER_FILE = 10;      // 每文件最大匹配数
const MAX_TOTAL_MATCHES = 500;        // 总匹配数上限
```

### 核心流程

#### 1. 搜索流程（handleQueryChange → _temp4）

```
用户输入 → 清除上一次的 timeout 和 abortController
        → 如果查询为空：清空结果
        → 否则：设置 isSearching=true
        → 在现有结果中过滤（快速响应）
        → 延迟100ms后执行 ripgrep 搜索
```

**ripgrep 参数：**
```javascript
[
  '-n',              // 显示行号
  '--no-heading',    // 不显示文件头
  '-i',              // 忽略大小写
  '-m', '10',        // 每文件最多10个匹配
  '-F',              // 固定字符串搜索（非正则）
  '-e', query        // 搜索模式
]
```

#### 2. 结果解析（parseRipgrepLine）

使用正则表达式解析 ripgrep 输出：
```typescript
/^(.*?):(\d+):(.*)$/
```
- 捕获组1：文件路径（处理 Windows 盘符路径）
- 捕获组2：行号
- 捕获组3：匹配文本

#### 3. 预览加载流程

```
选中项变化 → 创建 AbortController
          → 计算读取范围（line - 4 到 line + 4）
          → 调用 readFileInRange 读取文件
          → 成功后更新 preview 状态
          → 失败时显示 "(preview unavailable)"
```

#### 4. 结果去重与合并

```typescript
setMatches(prev => {
  const seen = new Set(prev.map(matchKey));
  const fresh = parsed.filter(p => !seen.has(matchKey(p)));
  const next = prev.concat(fresh);
  return next.length > MAX_TOTAL_MATCHES 
    ? next.slice(0, MAX_TOTAL_MATCHES) 
    : next;
});
```

### React Compiler 优化

代码使用 React Compiler（`_c` 函数）进行自动记忆化：
- 依赖数组缓存（`$[n]`）
- 条件性重新计算，避免不必要的渲染

## 关键代码路径与文件引用

### 当前文件
- `/src/components/GlobalSearchDialog.tsx` - 主组件实现

### 依赖文件

| 文件路径 | 用途 |
|---------|------|
| `/src/components/design-system/FuzzyPicker.tsx` | 模糊选择器基础组件 |
| `/src/context/overlayContext.tsx` | 覆盖层管理（Escape键协调） |
| `/src/hooks/useTerminalSize.ts` | 终端尺寸监听 |
| `/src/utils/ripgrep.ts` | ripgrep 封装（ripGrepStream） |
| `/src/utils/readFileInRange.ts` | 范围读取文件内容 |
| `/src/utils/editor.ts` | 外部编辑器集成 |
| `/src/utils/format.ts` | 文本格式化（truncatePathMiddle, truncateToWidth） |
| `/src/utils/highlightMatch.ts` | 匹配高亮 |
| `/src/utils/permissions/filesystem.ts` | 相对路径计算 |
| `/src/utils/cwd.ts` | 获取当前工作目录 |
| `/src/services/analytics/index.ts` | 分析事件上报 |
| `/src/ink.ts` | Ink 渲染组件（Text） |

### 调用方
- 通过快捷键 `Ctrl+Shift+F` / `Cmd+Shift+F` 触发
- 由主应用路由/命令系统调用

## 依赖与外部交互

### 外部工具依赖

**ripgrep (rg)**：
- 系统 rg（优先，如果 `USE_BUILTIN_RIPGREP` 为 falsy）
- 内置 rg（npm 构建模式，位于 `vendor/ripgrep`）
- 嵌入式 rg（Bun 打包模式，通过 argv0 切换）

### 配置依赖

- **终端尺寸**：通过 `useTerminalSize` 动态响应
- **覆盖层管理**：注册为 `global-search` 覆盖层

### 分析事件

```typescript
// 选择结果时上报
logEvent("tengu_global_search_select", {
  result_count: matches.length,
  opened_editor: opened
});

// 插入路径时上报
logEvent("tengu_global_search_insert", {
  result_count: matches.length,
  mention
});
```

## 风险、边界与改进建议

### 已知风险

1. **Windows 路径解析**
   - 风险：Windows 盘符路径（如 `C:\path`）中的冒号可能与行号分隔符混淆
   - 缓解：使用正则表达式 `^(.*?):(\d+):(.*)$` 正确处理

2. **大文件预览**
   - 风险：预览大文件时可能造成内存压力
   - 缓解：使用 `readFileInRange` 限制读取范围（最多9行）

3. **搜索取消竞争**
   - 风险：快速输入时可能产生多个并行的 ripgrep 进程
   - 缓解：使用 `AbortController` 取消前一个搜索

4. **结果截断**
   - 风险：大量匹配时只显示前500个，可能遗漏重要结果
   - 缓解：显示截断提示（`truncated ? "+" : ""`），建议用户优化搜索词

### 边界情况

| 场景 | 行为 |
|------|------|
| 空查询 | 显示 "Type to search…" |
| 无匹配 | 显示 "No matches" |
| 搜索中 | 显示 "Searching…" 和加载状态 |
| 文件读取失败 | 预览显示 "(preview unavailable)" |
| 终端宽度 < 140 | 预览在底部显示 |
| 终端宽度 ≥ 140 | 预览在右侧显示 |

### 改进建议

1. **正则搜索支持**
   - 当前仅支持固定字符串搜索（`-F` 参数）
   - 建议：添加选项支持正则表达式搜索

2. **文件类型过滤**
   - 当前搜索所有文件
   - 建议：添加文件类型/Glob过滤选项

3. **搜索结果排序**
   - 当前按 ripgrep 输出顺序
   - 建议：支持按文件名、匹配数等排序

4. **搜索历史**
   - 当前不保存搜索历史
   - 建议：添加最近搜索记录

5. **预览语法高亮**
   - 当前仅高亮匹配关键词
   - 建议：集成 HighlightedCode 组件提供完整语法高亮

6. **并发控制优化**
   - 当前每个字符输入都可能触发搜索
   - 建议：增加更智能的防抖策略（如等待用户停止输入）
