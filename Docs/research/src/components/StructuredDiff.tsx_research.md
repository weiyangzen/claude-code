# StructuredDiff.tsx 研究文档

## 场景与职责

StructuredDiff 是 Claude Code CLI 的代码差异高亮组件，负责渲染带有语法高亮的代码 diff。它是文件编辑、代码审查功能的核心 UI 组件，为用户提供专业的代码变更可视化。

### 核心职责
1. **语法高亮 Diff 渲染**：使用 Rust NAPI 模块 (`color-diff-napi`) 进行高性能语法高亮
2. **性能优化渲染**：通过多级缓存和列分割优化大量 diff 的渲染性能
3. **降级方案**：当语法高亮不可用时自动降级到纯文本 diff
4. **终端宽度适配**：动态计算 gutter 宽度，适应不同终端尺寸

### 使用场景
- 显示文件编辑的 diff（如 Bash 工具输出）
- 代码审查界面
- 任何需要展示代码变更的场景

### 架构设计
组件采用"缓存优先"的设计哲学：
- 使用 WeakMap 缓存 NAPI 渲染结果
- 预分割 gutter 和内容列，避免重复计算
- 支持 React remount 场景下的缓存命中

---

## 功能点目的

### 1. 缓存渲染结果 (`RENDER_CACHE`)
- **目的**：避免重复调用昂贵的 NAPI 语法高亮
- **实现**：
  - `WeakMap<StructuredPatchHunk, Map<string, CachedRender>>` 双层缓存结构
  - 外层：以 patch 对象为 key（WeakMap 允许垃圾回收）
  - 内层：以渲染参数组合为 key，限制最多 4 个条目（防止窗口调整大小时无限增长）

### 2. 智能列分割 (`computeGutterWidth`, `renderColorDiff`)
- **目的**：在全屏模式下实现可选择文本的 diff 显示
- **实现**：
  - Gutter 包含：标记符（+/-）+ 行号 + 间距
  - 计算最大行号宽度，右对齐行号
  - 使用 `sliceAnsi` 精确分割 ANSI 转义序列

### 3. 语法高亮降级
- **目的**：确保在任何环境下都能显示 diff
- **实现**：
  - 首先尝试 `renderColorDiff`（NAPI 语法高亮）
  - 如果返回 `null`，降级到 `StructuredDiffFallback`（纯文本 + 简单着色）

### 4. 终端宽度保护
- **目的**：防止极窄终端导致渲染错误
- **实现**：
  - `safeWidth = Math.max(1, Math.floor(width))`
  - gutter 宽度检查：`rawGutterWidth > 0 && rawGutterWidth < width`

---

## 具体技术实现

### 关键数据结构

```typescript
// Props 定义
type Props = {
  patch: StructuredPatchHunk;           // diff 数据块
  dim: boolean;                         // 是否使用暗色调
  filePath: string;                     // 文件路径（用于语言检测）
  firstLine: string | null;             // 文件首行（用于 shebang 检测）
  fileContent?: string;                 // 完整文件内容（用于多行字符串等上下文）
  width: number;                        // 可用渲染宽度
  skipHighlighting?: boolean;           // 强制跳过语法高亮
};

// 缓存的渲染结果
interface CachedRender {
  lines: string[];                      // 高亮后的行数组
  gutterWidth: number;                  // gutter 列宽度
  gutters: string[] | null;             // 预分割的 gutter 内容
  contents: string[] | null;            // 预分割的内容列
}

// 缓存结构
const RENDER_CACHE = new WeakMap<StructuredPatchHunk, Map<string, CachedRender>>();
```

### 缓存 Key 生成
```typescript
const key = `${theme}|${width}|${dim ? 1 : 0}|${gutterWidth}|${firstLine ?? ''}|${filePath}`;
```
包含所有影响渲染结果的参数，确保缓存命中准确性。

### 关键流程

#### 1. 渲染流程
```
StructuredDiff (memo 包裹)
    ↓
获取当前 theme: useTheme()
    ↓
获取 settings: useSettings()
    ↓
计算 safeWidth = Math.max(1, Math.floor(width))
    ↓
检查 skipHighlighting || syntaxHighlightingDisabled
    ↓
调用 renderColorDiff(patch, firstLine, filePath, fileContent, theme, safeWidth, dim, splitGutter)
    ↓
缓存命中？
  ├─ 是：返回缓存的 CachedRender
  └─ 否：执行 NAPI 渲染并缓存
    ↓
缓存结果存在？
  ├─ 是：
  │   ├─ gutterWidth > 0？
  │   │   ├─ 是：渲染两列布局（NoSelect gutter + RawAnsi content）
  │   │   └─ 否：渲染单列 RawAnsi
  └─ 否：渲染 StructuredDiffFallback
```

#### 2. NAPI 渲染流程 (`renderColorDiff`)
```typescript
function renderColorDiff(patch, firstLine, filePath, fileContent, theme, width, dim, splitGutter): CachedRender | null {
  // 1. 获取 ColorDiff 类（检查环境变量是否禁用）
  const ColorDiff = expectColorDiff();
  if (!ColorDiff) return null;
  
  // 2. 计算 gutter 宽度
  const rawGutterWidth = splitGutter ? computeGutterWidth(patch) : 0;
  const gutterWidth = rawGutterWidth > 0 && rawGutterWidth < width ? rawGutterWidth : 0;
  
  // 3. 生成缓存 key
  const key = `${theme}|${width}|${dim ? 1 : 0}|${gutterWidth}|${firstLine ?? ''}|${filePath}`;
  
  // 4. 检查缓存
  let perHunk = RENDER_CACHE.get(patch);
  const hit = perHunk?.get(key);
  if (hit) return hit;
  
  // 5. 执行 NAPI 渲染
  const lines = new ColorDiff(patch, firstLine, filePath, fileContent).render(theme, width, dim);
  if (lines === null) return null;
  
  // 6. 预分割 gutter 和内容
  let gutters: string[] | null = null;
  let contents: string[] | null = null;
  if (gutterWidth > 0) {
    gutters = lines.map(l => sliceAnsi(l, 0, gutterWidth));
    contents = lines.map(l => sliceAnsi(l, gutterWidth));
  }
  
  // 7. 存入缓存
  const entry: CachedRender = { lines, gutterWidth, gutters, contents };
  if (!perHunk) {
    perHunk = new Map();
    RENDER_CACHE.set(patch, perHunk);
  }
  if (perHunk.size >= 4) perHunk.clear();  // 防止无限增长
  perHunk.set(key, entry);
  
  return entry;
}
```

#### 3. Gutter 宽度计算
```typescript
function computeGutterWidth(patch: StructuredPatchHunk): number {
  const maxLineNumber = Math.max(
    patch.oldStart + patch.oldLines - 1,
    patch.newStart + patch.newLines - 1,
    1
  );
  return maxLineNumber.toString().length + 3; // marker + 2 padding spaces
}
```
- marker: 1 字符（+/-/空格）
- space: 1 字符
- line number: 最大行号宽度
- trailing space: 1 字符

### 渲染布局

#### 两列布局（全屏模式）
```jsx
<Box flexDirection="row">
  <NoSelect fromLeftEdge={true}>
    <RawAnsi lines={gutters} width={gutterWidth} />
  </NoSelect>
  <RawAnsi lines={contents} width={safeWidth - gutterWidth} />
</Box>
```
- **Gutter 列**：使用 `NoSelect` 包装，全屏文本选择时不包含行号和标记
- **内容列**：普通 `RawAnsi`，支持文本选择

#### 单列布局（非全屏或窄终端）
```jsx
<Box>
  <RawAnsi lines={lines} width={safeWidth} />
</Box>
```

### React Compiler 优化

组件使用 React Compiler 自动优化：
- 26 个缓存槽（`$[0]` 到 `$[25]`）
- 条件缓存比较：
  ```typescript
  if ($[0] !== dim || $[1] !== fileContent || ... || $[8] !== theme) {
    // 重新计算
  } else {
    return cached;
  }
  ```

---

## 关键代码路径与文件引用

### 主要文件
- `/src/components/StructuredDiff.tsx` - 主组件实现
- `/src/components/StructuredDiff/colorDiff.ts` - ColorDiff NAPI 模块封装
- `/src/components/StructuredDiff/Fallback.tsx` - 降级渲染组件

### 依赖文件

| 文件路径 | 用途 |
|---------|------|
| `diff` | `StructuredPatchHunk` 类型 |
| `src/hooks/useSettings.ts` | `useSettings` |
| `src/ink.ts` | `Box`, `NoSelect`, `RawAnsi`, `useTheme` |
| `src/utils/fullscreen.ts` | `isFullscreenEnvEnabled` |
| `src/utils/sliceAnsi.ts` | `sliceAnsi` - ANSI 安全字符串分割 |
| `color-diff-napi` | `ColorDiff`, `ColorFile`, `getSyntaxTheme` |
| `src/utils/envUtils.ts` | `isEnvDefinedFalsy` |

### colorDiff.ts 模块
```typescript
export function getColorModuleUnavailableReason(): 'env' | null {
  if (isEnvDefinedFalsy(process.env.CLAUDE_CODE_SYNTAX_HIGHLIGHT)) {
    return 'env';  // 通过环境变量禁用
  }
  return null;
}

export function expectColorDiff(): typeof ColorDiff | null {
  return getColorModuleUnavailableReason() === null ? ColorDiff : null;
}
```

### Fallback.tsx 降级组件
- 使用 `diffWordsWithSpace` 进行词级 diff
- 支持 40% 变化阈值（变化过大时显示整行 diff）
- 手动文本换行处理
- 行号对齐和背景色着色

### 调用方
- `src/components/StructuredDiffList.tsx` - 多 hunk 列表渲染
- 可能在消息渲染组件中调用（如显示 Bash 工具输出）

---

## 依赖与外部交互

### 外部模块依赖

#### color-diff-napi (Rust NAPI 模块)
- **ColorDiff 类**：核心语法高亮引擎
  - 构造函数：`new ColorDiff(patch, firstLine, filePath, fileContent)`
  - 渲染方法：`.render(theme, width, dim)` → `string[]`
- **ColorFile 类**：完整文件语法高亮（本组件未使用）
- **getSyntaxTheme 函数**：获取主题配置

#### diff (npm 包)
- `StructuredPatchHunk` 类型定义
- `diffWordsWithSpace`（在 Fallback 中使用）

### 环境变量
- `CLAUDE_CODE_SYNTAX_HIGHLIGHT`：设置为 falsy 值时禁用语法高亮

### Settings
- `syntaxHighlightingDisabled`：用户设置禁用语法高亮

---

## 风险、边界与改进建议

### 已知风险

1. **NAPI 模块依赖**
   - 依赖 Rust 编译的 NAPI 模块，在某些平台可能不可用
   - **缓解措施**：有完整的 Fallback 降级方案

2. **内存泄漏风险**
   - `RENDER_CACHE` 是模块级 WeakMap，理论上不会泄漏
   - 但内层 Map 限制 4 个条目，防止窗口调整大小时无限增长
   - **潜在问题**：如果 patch 对象被长期持有，缓存也不会释放

3. **竞态条件**
   - 组件使用 memo 和缓存，但父组件可能传递新的 patch 对象
   - **缓解措施**：WeakMap 使用对象引用作为 key，相同内容的不同对象会创建不同缓存条目

4. **sliceAnsi 性能**
   - 每行调用两次 `sliceAnsi`（冷缓存时）
   - 对于大 diff（数百行），这可能成为瓶颈
   - **缓解措施**：结果缓存，只计算一次

### 边界情况

1. **极窄终端**
   - `width <= gutterWidth` 时自动禁用 gutter 分割
   - `safeWidth` 确保至少为 1

2. **空 patch**
   - NAPI 渲染可能返回 `null`
   - 组件正确处理并降级到 Fallback

3. **文件路径为空**
   - `filePath` 用于语言检测，为空时可能无法正确高亮
   - Fallback 机制确保仍能显示

4. **主题变化**
   - 主题是缓存 key 的一部分，切换主题会触发重新渲染

5. **Remount 场景**
   - 注释提到："ctrl+o unmounts/remounts the entire message tree"
   - 模块级缓存确保 remount 时快速恢复

### 改进建议

1. **缓存优化**
   - 考虑使用 LRU 缓存替代简单的 Map
   - 当前 4 条目限制在极端窗口调整场景可能不足
   - 可以添加缓存命中率的调试日志

2. **错误处理**
   - NAPI 模块加载失败时当前静默降级
   - 考虑在开发模式下显示警告

3. **性能监控**
   - 添加 NAPI 渲染耗时监控
   - 大 diff 检测和警告（如 >1000 行）

4. **可访问性**
   - 当前没有 ARIA 标签
   - 考虑添加 `role="img"` 和 `aria-label` 描述 diff 内容

5. **代码组织**
   - `renderColorDiff` 函数较长，可以拆分为更小的函数
   - 缓存 key 生成逻辑可以提取为独立函数

6. **测试覆盖**
   - 建议添加单元测试：
     - 缓存命中/未命中场景
     - gutter 宽度计算边界值
     - 降级场景
     - 极窄终端处理

7. **国际化**
   - 当前无国际化支持
   - 如果 diff 包含非 ASCII 字符，`sliceAnsi` 的双宽度字符处理是否正确？

8. **功能增强**
   - 支持代码折叠（隐藏未变更的大段代码）
   - 支持行内 diff（字符级高亮）
   - 支持自定义主题
