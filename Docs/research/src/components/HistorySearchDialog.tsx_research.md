# HistorySearchDialog.tsx 研究文档

## 场景与职责

HistorySearchDialog 是 Claude Code CLI 的历史记录搜索对话框组件，对应快捷键 `Ctrl+R`。它允许用户搜索和复用之前的输入历史，提高交互效率。

**核心职责：**
- 提供交互式历史记录搜索界面
- 支持模糊匹配和精确匹配
- 异步加载历史记录（懒解析粘贴内容）
- 显示相对时间（如 "2m ago", "1h ago"）
- 提供预览面板显示完整历史内容

## 功能点目的

### 1. 历史搜索界面
- **搜索框**：实时过滤历史记录
- **结果列表**：显示历史条目的首行和时间
- **预览面板**：显示完整的历史内容（多行文本）
- **响应式布局**：根据终端宽度调整布局（≥100列时预览在右侧）

### 2. 匹配算法
- **精确匹配**：查询字符串包含在历史条目中
- **模糊匹配**：查询字符串是历史条目的子序列（Subsequence）
- 精确匹配结果优先显示

### 3. 历史加载
- 异步加载历史记录
- 使用 `AsyncGenerator` 流式读取
- 支持取消操作（避免内存泄漏）
- 当前项目过滤（只显示当前项目的历史）

### 4. 时间显示
- 使用 `formatRelativeTimeAgo` 显示相对时间
- 固定宽度显示（`AGE_WIDTH = 8`）

### 5. 预览功能
- 使用 `wrapAnsi` 自动换行
- 限制显示行数（`PREVIEW_ROWS = 6`）
- 超出时显示 "… +N more lines"

## 具体技术实现

### 关键数据结构

```typescript
// 历史条目（内部使用）
type Item = {
  entry: TimestampedHistoryEntry;  // 原始历史条目
  display: string;                 // 显示文本
  lower: string;                   // 小写文本（用于匹配）
  firstLine: string;               // 首行（列表显示）
  age: string;                     // 相对时间字符串
};

// 时间戳历史条目（来自 history.ts）
type TimestampedHistoryEntry = {
  display: string;                 // 显示文本
  timestamp: number;               // 时间戳
  resolve: () => Promise<HistoryEntry>;  // 懒加载完整内容
};

// 历史条目（完整）
type HistoryEntry = {
  display: string;
  pastedContents: Record<number, PastedContent>;  // 粘贴内容
};
```

### 关键常量

```typescript
const PREVIEW_ROWS = 6;    // 预览显示行数
const AGE_WIDTH = 8;       // 时间列宽度
```

### 核心流程

#### 1. 历史加载流程

```
组件挂载 → 启动异步加载
        → 调用 getTimestampedHistory() 获取生成器
        → 遍历生成器读取条目
        → 提取首行、计算相对时间
        → 构建 Item 对象
        → 更新 items 状态
        → 支持取消（cancelled 标志）
```

**代码实现：**
```typescript
useEffect(() => {
  let cancelled = false;
  void (async () => {
    const reader = getTimestampedHistory();
    const loaded: Item[] = [];
    for await (const entry of reader) {
      if (cancelled) {
        void reader.return(undefined);
        return;
      }
      // 处理条目...
      loaded.push({
        entry,
        display,
        lower: display.toLowerCase(),
        firstLine: nl === -1 ? display : display.slice(0, nl),
        age: age + ' '.repeat(Math.max(0, AGE_WIDTH - stringWidth(age)))
      });
    }
    if (!cancelled) setItems(loaded);
  })();
  return () => { cancelled = true; };
}, []);
```

#### 2. 过滤算法

```typescript
const filtered = useMemo(() => {
  if (!items) return [];
  const q = query.trim().toLowerCase();
  if (!q) return items;
  
  const exact: Item[] = [];
  const fuzzy: Item[] = [];
  
  for (const item of items) {
    if (item.lower.includes(q)) {
      exact.push(item);           // 精确匹配
    } else if (isSubsequence(item.lower, q)) {
      fuzzy.push(item);           // 模糊匹配
    }
  }
  
  return exact.concat(fuzzy);      // 精确优先
}, [items, query]);
```

#### 3. 子序列匹配算法

```typescript
function isSubsequence(text: string, query: string): boolean {
  let j = 0;
  for (let i = 0; i < text.length && j < query.length; i++) {
    if (text[i] === query[j]) j++;
  }
  return j === query.length;
}
```

时间复杂度：O(n + m)，其中 n 是文本长度，m 是查询长度

#### 4. 预览渲染

```typescript
renderPreview={item => {
  const wrapped = wrapAnsi(item.display, previewWidth, { hard: true })
    .split('\n')
    .filter(l => l.trim() !== '');
  const overflow = wrapped.length > PREVIEW_ROWS;
  const shown = wrapped.slice(0, overflow ? PREVIEW_ROWS - 1 : PREVIEW_ROWS);
  const more = wrapped.length - shown.length;
  
  return (
    <Box flexDirection="column" borderStyle="round" borderDimColor paddingX={1} height={PREVIEW_ROWS + 2}>
      {shown.map((row, i) => <Text key={i} dimColor>{row}</Text>)}
      {more > 0 && <Text dimColor>{`… +${more} more lines`}</Text>}
    </Box>
  );
}}
```

## 关键代码路径与文件引用

### 当前文件
- `/src/components/HistorySearchDialog.tsx` - 主组件实现

### 依赖文件

| 文件路径 | 用途 |
|---------|------|
| `/src/components/design-system/FuzzyPicker.tsx` | 模糊选择器基础组件 |
| `/src/context/overlayContext.tsx` | 覆盖层管理 |
| `/src/history.ts` | 历史记录 API（getTimestampedHistory） |
| `/src/hooks/useTerminalSize.ts` | 终端尺寸监听 |
| `/src/ink/stringWidth.ts` | 字符串宽度计算（处理 Unicode） |
| `/src/ink/wrapAnsi.ts` | ANSI 文本自动换行 |
| `/src/ink.ts` | Ink 组件（Box, Text） |
| `/src/services/analytics/index.ts` | 分析事件上报 |
| `/src/utils/config.ts` | HistoryEntry 类型定义 |
| `/src/utils/format.ts` | 格式化工具（formatRelativeTimeAgo, truncateToWidth） |

### 调用方
- 通过快捷键 `Ctrl+R` 触发
- 由主应用路由/命令系统调用

## 依赖与外部交互

### 历史记录系统

**数据来源：**
```typescript
// 来自 history.ts
export async function* getTimestampedHistory(): AsyncGenerator<TimestampedHistoryEntry> {
  const currentProject = getProjectRoot();
  const seen = new Set<string>();
  
  for await (const entry of makeLogEntryReader()) {
    if (entry.project !== currentProject) continue;
    if (seen.has(entry.display)) continue;  // 去重
    
    yield {
      display: entry.display,
      timestamp: entry.timestamp,
      resolve: () => logEntryToHistoryEntry(entry),  // 懒加载
    };
    
    if (seen.size >= MAX_HISTORY_ITEMS) return;  // 最多100条
  }
}
```

**存储位置：**
- 全局历史文件：`~/.claude/history.jsonl`
- 格式：JSON Lines（每行一个 JSON 对象）

### 分析事件

```typescript
logEvent('tengu_history_picker_select', {
  result_count: filtered.length,
  query_length: query.length
});
```

### 覆盖层管理

```typescript
useRegisterOverlay('history-search');
```

注册为 `history-search` 覆盖层，用于 Escape 键协调。

## 风险、边界与改进建议

### 已知风险

1. **异步加载竞态**
   - 风险：组件快速卸载/重新挂载可能导致内存泄漏或状态不一致
   - 缓解：使用 `cancelled` 标志和 `reader.return()` 取消

2. **大历史文件**
   - 风险：历史文件过大时加载缓慢
   - 缓解：限制最多 100 条（`MAX_HISTORY_ITEMS`），流式读取

3. **子序列匹配性能**
   - 风险：历史条目过多时过滤可能卡顿
   - 缓解：使用 `useMemo` 缓存过滤结果

4. **预览换行**
   - 风险：包含 ANSI 转义码的文本换行可能出错
   - 缓解：使用 `wrapAnsi` 处理

### 边界情况

| 场景 | 行为 |
|------|------|
| 历史为空 | 显示 "No history yet" |
| 加载中 | 显示 "Loading…" |
| 无匹配 | 显示 "No matching prompts" |
| 空查询 | 显示所有历史 |
| 终端宽度 < 100 | 预览在底部 |
| 终端宽度 ≥ 100 | 预览在右侧 |
| 多行历史 | 列表只显示首行，预览显示全部 |
| 粘贴内容 | 懒加载，选中时才解析 |

### 改进建议

1. **搜索性能优化**
   - 当前：O(n) 遍历所有历史
   - 建议：使用倒排索引或 Trie 树加速搜索

2. **历史持久化**
   - 当前：只保存到本地文件
   - 建议：支持云端同步历史记录

3. **智能排序**
   - 当前：按时间倒序
   - 建议：结合使用频率和时间加权排序

4. **历史分组**
   - 当前：平铺列表
   - 建议：按日期分组（今天、昨天、上周等）

5. **历史编辑**
   - 当前：只读
   - 建议：支持删除单条历史或清空

6. **预览增强**
   - 当前：纯文本预览
   - 建议：集成语法高亮（使用 HighlightedCode 组件）

7. **跨项目历史**
   - 当前：只显示当前项目
   - 建议：可选显示所有项目历史

8. **历史统计**
   - 建议：显示使用统计（最常用命令等）
