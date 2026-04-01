# HighlightedCode.tsx 研究文档

## 场景与职责

HighlightedCode 是 Claude Code CLI 的语法高亮代码显示组件。它负责将源代码渲染为带有语法高亮的终端输出，支持多种编程语言的语法解析和主题渲染。

**核心职责：**
- 提供代码语法高亮显示
- 支持终端自适应宽度
- 提供行号 gutter（在全屏模式下）
- 优雅降级（当语法高亮不可用时）
- 响应主题变化和用户设置

## 功能点目的

### 1. 语法高亮
- 基于 `color-diff-napi` 原生模块实现高性能语法解析
- 支持多种编程语言的语法高亮
- 支持亮色/暗色主题切换

### 2. 自适应布局
- 支持固定宽度或自动测量容器宽度
- 默认宽度 80 字符
- 通过 `measureElement` 自动适应容器

### 3. 行号显示
- 全屏模式下显示行号 gutter
- 通过 `isFullscreenEnvEnabled()` 检测
- 行号宽度根据代码行数动态计算

### 4. 降级处理
- 当语法高亮被禁用或不可用时，使用 `HighlightedCodeFallback`
- 支持 `syntaxHighlightingDisabled` 设置项
- 支持环境变量 `CLAUDE_CODE_SYNTAX_HIGHLIGHT` 控制

### 5. 视觉样式
- 支持 `dim` 属性降低显示亮度
- 使用 ANSI 转义码渲染颜色
- 支持 `NoSelect` 组件防止行号被选中

## 具体技术实现

### 关键数据结构

```typescript
type Props = {
  code: string;       // 源代码内容
  filePath: string;   // 文件路径（用于语法识别）
  width?: number;     // 指定宽度（可选）
  dim?: boolean;      // 是否降低亮度
};
```

### 核心常量

```typescript
const DEFAULT_WIDTH = 80;  // 默认显示宽度
```

### 技术实现细节

#### 1. 语法高亮初始化流程

```
检查 syntaxHighlightingDisabled 设置
    ↓
如果启用 → 调用 expectColorFile() 获取 ColorFile 类
    ↓
创建 ColorFile 实例（code, filePath）
    ↓
调用 colorFile.render(theme, measuredWidth, dim) 生成高亮行
```

#### 2. 宽度测量机制

```typescript
useEffect(() => {
  if (!width && ref.current) {
    const { width: elementWidth } = measureElement(ref.current);
    if (elementWidth > 0) {
      setMeasuredWidth(elementWidth - 2);  // 留出边距
    }
  }
}, [width]);
```

#### 3. 行号 gutter 计算

```typescript
const gutterWidth = isFullscreenEnvEnabled() 
  ? (countCharInString(code, "\n") + 1).toString().length + 2  // 行号宽度 + 边距
  : 0;
```

#### 4. 行渲染组件（CodeLine）

```typescript
function CodeLine({ line, gutterWidth }) {
  const gutter = sliceAnsi(line, 0, gutterWidth);      // 提取行号部分
  const content = sliceAnsi(line, gutterWidth);        // 提取代码内容
  
  return (
    <Box flexDirection="row">
      <NoSelect fromLeftEdge={true}>
        <Text><Ansi>{gutter}</Ansi></Text>  {/* 行号（不可选中） */}
      </NoSelect>
      <Text><Ansi>{content}</Ansi></Text>   {/* 代码内容 */}
    </Box>
  );
}
```

### 依赖模块详解

#### color-diff-napi 模块

通过 `expectColorFile()` 获取：

```typescript
// 来自 components/StructuredDiff/colorDiff.ts
export function expectColorFile(): typeof ColorFile | null {
  return getColorModuleUnavailableReason() === null ? ColorFile : null;
}
```

**可用性检查：**
- 环境变量 `CLAUDE_CODE_SYNTAX_HIGHLIGHT` 不为 falsy

**ColorFile API：**
```typescript
class ColorFile {
  constructor(code: string, filePath: string);
  render(theme: SyntaxTheme, width: number, dim: boolean): string[];
}
```

#### sliceAnsi 工具

用于安全地截取 ANSI 转义码字符串：
```typescript
sliceAnsi(line, 0, gutterWidth);      // 从开头截取
sliceAnsi(line, gutterWidth);         // 从指定位置截取到末尾
```

## 关键代码路径与文件引用

### 当前文件
- `/src/components/HighlightedCode.tsx` - 主组件实现

### 依赖文件

| 文件路径 | 用途 |
|---------|------|
| `/src/components/HighlightedCode/Fallback.tsx` | 语法高亮不可用的降级组件 |
| `/src/components/StructuredDiff/colorDiff.ts` | color-diff-napi 模块封装 |
| `/src/hooks/useSettings.ts` | 用户设置（syntaxHighlightingDisabled） |
| `/src/ink.ts` | Ink 组件（Ansi, Box, Text, measureElement, NoSelect, useTheme） |
| `/src/utils/fullscreen.ts` | 全屏环境检测 |
| `/src/utils/sliceAnsi.ts` | ANSI 字符串安全截取 |
| `/src/utils/stringUtils.ts` | 字符串工具（countCharInString） |

### 被调用方

该组件被多处使用：
- 代码预览（如 GlobalSearchDialog 的预览面板）
- Diff 显示
- 文件内容展示
- 代码块渲染

## 依赖与外部交互

### 外部模块依赖

**color-diff-napi**：
- Rust 编写的原生 Node.js 模块
- 提供高性能语法高亮
- 支持多种语言和主题

### 配置依赖

**设置项：**
```typescript
const settings = useSettings();
const syntaxHighlightingDisabled = settings.syntaxHighlightingDisabled ?? false;
```

**环境变量：**
```typescript
// 禁用语法高亮
CLAUDE_CODE_SYNTAX_HIGHLIGHT=false
```

**主题：**
```typescript
const [theme] = useTheme();  // 来自 Ink 主题系统
```

### 渲染流程

```
HighlightedCode
    ├── syntaxHighlightingDisabled ? 
    │       └── HighlightedCodeFallback
    └── colorFile ?
            ├── colorFile.render() → lines[]
            │       └── lines.map(line => 
            │               gutterWidth > 0 ? 
            │                   <CodeLine /> : 
            │                   <Text><Ansi>{line}</Ansi></Text>
            │           )
            └── null → HighlightedCodeFallback
```

## 风险、边界与改进建议

### 已知风险

1. **原生模块依赖**
   - 风险：`color-diff-napi` 是原生模块，可能在某些平台无法加载
   - 缓解：有完整的降级路径（HighlightedCodeFallback）

2. **内存使用**
   - 风险：大文件语法高亮可能消耗大量内存
   - 缓解：通过 `width` 参数限制每行长度

3. **性能问题**
   - 风险：频繁重新渲染大代码块可能影响性能
   - 缓解：React Compiler 自动记忆化（`memo` 包装）

4. **ANSI 截断**
   - 风险：不正确的字符串截断可能破坏 ANSI 转义序列
   - 缓解：使用专门的 `sliceAnsi` 工具函数

### 边界情况

| 场景 | 行为 |
|------|------|
| 语法高亮禁用 | 使用 Fallback 组件，纯文本显示 |
| color-diff 模块不可用 | 使用 Fallback 组件 |
| 未指定宽度 | 自动测量容器宽度 |
| 测量失败 | 使用默认宽度 80 |
| 非全屏模式 | 不显示行号 |
| 空代码 | 正常渲染（无行） |
| 单行代码 | gutter 宽度根据行号位数计算 |

### 改进建议

1. **虚拟滚动**
   - 当前：大文件全部渲染
   - 建议：添加虚拟滚动支持，只渲染可见行

2. **增量高亮**
   - 当前：每次重新高亮整个文件
   - 建议：缓存高亮结果，只更新变化部分

3. **更多主题**
   - 当前：依赖 color-diff-napi 内置主题
   - 建议：支持自定义主题配置

4. **行高亮**
   - 当前：统一渲染所有行
   - 建议：支持高亮指定行（如当前光标行）

5. **代码折叠**
   - 当前：平铺显示所有代码
   - 建议：支持函数/类级别的代码折叠

6. **搜索高亮**
   - 当前：仅语法高亮
   - 建议：集成搜索关键词高亮（与 GlobalSearchDialog 协同）

7. **错误恢复**
   - 当前：模块失败时完全降级
   - 建议：尝试重新加载原生模块或显示警告
