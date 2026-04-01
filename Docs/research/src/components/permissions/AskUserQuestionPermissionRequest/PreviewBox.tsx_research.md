# PreviewBox.tsx 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`PreviewBox` 是一个用于终端 UI 的预览内容展示组件，专门用于在 `AskUserQuestionPermissionRequest` 流程中显示选项的预览内容（如代码片段、设计稿、文档等）。它提供了一个带边框的等宽字体盒子，支持 Markdown 渲染和语法高亮。

### 1.2 使用场景
- **选项预览展示**：当用户在多选题中切换选项时，右侧预览面板显示对应选项的详细内容
- **代码片段展示**：支持多种编程语言的语法高亮（通过 `cliHighlight`）
- **Markdown 内容渲染**：支持标题、列表、代码块、引用等 Markdown 元素
- **终端自适应布局**：根据终端尺寸自动调整宽度和高度，支持内容截断和滚动提示

### 1.3 调用关系
- **被调用方**：`PreviewQuestionView.tsx` 是主要调用者，为每个聚焦的选项渲染预览
- **父级上下文**：`AskUserQuestionPermissionRequest.tsx` 计算全局内容尺寸并传递给子组件

---

## 2. 功能点目的

### 2.1 带边框的预览容器
- 使用 Unicode 制表符（`┌┐└┘│─├┤`）绘制美观的边框
- 支持自定义最小/最大宽度和高度
- 内容区域自动内边距处理

### 2.2 智能内容截断
- 当内容超过 `maxLines` 时显示截断指示器（`─── ✂ ─── N lines hidden`）
- 使用 `sliceAnsi` 处理 ANSI 转义序列，确保样式不中断
- 支持最小高度填充，保持布局一致性

### 2.3 Markdown 渲染与语法高亮
- 通过 `applyMarkdown` 将 Markdown 转换为带样式的终端文本
- 支持代码块的语言识别和高亮（需 `CliHighlight`）
- 可通过设置禁用语法高亮以提升性能

### 2.4 响应式布局
- 监听终端尺寸变化（`useTerminalSize`）
- 自动计算可用宽度（`terminalWidth - 4` 作为默认）
- 内容宽度自适应，考虑 CJK 字符和 Emoji 宽度

---

## 3. 具体技术实现

### 3.1 组件架构

```
PreviewBox (入口组件)
├── PreviewBoxWithHighlight (Suspense 包装器，异步加载高亮器)
└── PreviewBoxBody (核心渲染逻辑)
```

### 3.2 关键数据结构

```typescript
// 组件 Props
interface PreviewBoxProps {
  content: string;           // 预览内容（支持 Markdown）
  maxLines?: number;         // 最大显示行数（默认 20）
  minHeight?: number;        // 最小高度（行数）
  minWidth?: number;         // 最小宽度（默认 40）
  maxWidth?: number;         // 最大可用宽度
}

// Unicode 边框字符
const BOX_CHARS = {
  topLeft: '┌',
  topRight: '┐',
  bottomLeft: '└',
  bottomRight: '┘',
  horizontal: '─',
  vertical: '│',
  teeLeft: '├',
  teeRight: '┤'
};
```

### 3.3 核心渲染流程

1. **内容预处理**（`PreviewBoxBody`）
   ```javascript
   const rendered = applyMarkdown(content, theme, highlight);
   ```
   - 调用 `applyMarkdown` 解析 Markdown 并应用主题样式
   - 如果提供了 `highlight`，代码块会被语法高亮

2. **尺寸计算**
   ```javascript
   const effectiveMaxWidth = maxWidth ?? terminalWidth - 4;
   const effectiveMaxLines = maxLines ?? 20;
   const contentWidth = Math.max(minWidth, ...lines.map(stringWidth));
   const boxWidth = Math.min(contentWidth + 4, effectiveMaxWidth);
   const innerWidth = boxWidth - 4; // 减去边框和间距
   ```

3. **内容截断处理**
   ```javascript
   const contentLines = rendered.split("\n");
   const isTruncated = contentLines.length > effectiveMaxLines;
   const truncatedLines = isTruncated ? contentLines.slice(0, effectiveMaxLines) : contentLines;
   ```

4. **截断指示器渲染**（当内容被截断时）
   ```javascript
   const hiddenCount = contentLines.length - effectiveMaxLines;
   const label = `${BOX_CHARS.horizontal.repeat(3)} ✂ ${BOX_CHARS.horizontal.repeat(3)} ${hiddenCount} lines hidden `;
   ```

5. **行渲染与填充**
   ```javascript
   const paddingNeeded = Math.max(0, effectiveMinHeight - truncatedLines.length - (isTruncated ? 1 : 0));
   const lines = paddingNeeded > 0 ? [...truncatedLines, ...Array(paddingNeeded).fill("")] : truncatedLines;
   ```

### 3.4 语法高亮加载策略

```javascript
// PreviewBox 入口判断
if (settings.syntaxHighlightingDisabled) {
  return <PreviewBoxBody {...props} highlight={null} />;
}
return (
  <Suspense fallback={<PreviewBoxBody {...props} highlight={null} />}>
    <PreviewBoxWithHighlight {...props} />
  </Suspense>
);

// PreviewBoxWithHighlight 异步加载
const highlight = use(getCliHighlightPromise());
```

- 使用 React 19 的 `use()` Hook 加载异步高亮模块
- 提供无高亮的 fallback UI，避免阻塞渲染

### 3.5 宽度感知处理

```javascript
// 使用 stringWidth 计算显示宽度（正确处理 CJK 和 Emoji）
const lineWidth = stringWidth(line_0);
const displayLine = lineWidth > innerWidth ? sliceAnsi(line_0, 0, innerWidth) : line_0;
const padding = " ".repeat(Math.max(0, innerWidth - stringWidth(displayLine)));
```

---

## 4. 关键代码路径与文件引用

### 4.1 组件文件
| 文件路径 | 职责 |
|---------|------|
| `src/components/permissions/AskUserQuestionPermissionRequest/PreviewBox.tsx` | 主组件实现 |

### 4.2 依赖工具函数
| 文件路径 | 功能 |
|---------|------|
| `src/utils/markdown.ts` | `applyMarkdown()` - Markdown 解析与渲染 |
| `src/utils/cliHighlight.ts` | `getCliHighlightPromise()` - 异步加载语法高亮器 |
| `src/utils/sliceAnsi.ts` | `sliceAnsi()` - ANSI 安全的字符串切片 |
| `src/ink/stringWidth.ts` | `stringWidth()` - 终端显示宽度计算 |
| `src/hooks/useTerminalSize.ts` | `useTerminalSize()` - 终端尺寸监听 |
| `src/hooks/useSettings.ts` | `useSettings()` - 用户设置（语法高亮开关） |

### 4.3 调用链
```
AskUserQuestionPermissionRequest.tsx
├── 计算 maxHeight/maxWidth
├── 检测 hasPreview
├── 调用 applyMarkdown 预计算尺寸
└── 传递 globalContentHeight/Width 给子组件

    QuestionView.tsx
    ├── 检测 hasAnyPreview
    └── 渲染 PreviewQuestionView

        PreviewQuestionView.tsx
        ├── 管理 focusedIndex 状态
        ├── 获取 focusedOption.preview
        └── 渲染 PreviewBox
            ├── 调用 applyMarkdown(content, theme, highlight)
            ├── 计算 boxWidth/innerWidth
            ├── 截断处理
            └── 渲染边框 + 内容行
```

---

## 5. 依赖与外部交互

### 5.1 React 依赖
- **React Compiler 优化**：代码使用 `_c(n)` 缓存机制（React Compiler 编译产物）
- **Suspense + use()**：异步加载语法高亮模块
- **Ink 组件**：`Box`, `Text`, `Ansi` 用于终端 UI 渲染

### 5.2 工具模块依赖
```typescript
import { useSettings } from '../../../hooks/useSettings.js';
import { useTerminalSize } from '../../../hooks/useTerminalSize.js';
import { stringWidth } from '../../../ink/stringWidth.js';
import { Ansi, Box, Text, useTheme } from '../../../ink.js';
import { type CliHighlight, getCliHighlightPromise } from '../../../utils/cliHighlight.js';
import { applyMarkdown } from '../../../utils/markdown.js';
import sliceAnsi from '../../../utils/sliceAnsi.js';
```

### 5.3 配置依赖
- **Settings**: `syntaxHighlightingDisabled` - 控制是否禁用语法高亮
- **Theme**: 通过 `useTheme` 获取当前主题，传递给 `applyMarkdown`

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 性能风险
- **Markdown 重复解析**：每次渲染都调用 `applyMarkdown`，对于大内容可能影响性能
- **缓存粒度**：React Compiler 的缓存基于 props 引用，内容字符串变化会触发重新渲染

#### 6.1.2 布局风险
- **宽度计算偏差**：`stringWidth` 依赖 `Bun.stringWidth` 或 JS fallback，某些特殊字符可能计算不准确
- **终端兼容性**：Unicode 制表符在某些终端可能显示异常

#### 6.1.3 内容截断问题
- **ANSI 序列中断**：虽然使用 `sliceAnsi`，但在极端情况下样式可能泄漏
- **多字节字符截断**：依赖 `sliceAnsi` 正确处理 grapheme cluster

### 6.2 边界情况

| 场景 | 当前行为 | 建议 |
|-----|---------|------|
| 空内容 | 显示 "No preview available"（调用方处理） | 组件内可添加默认占位 |
| 终端宽度 < minWidth | 内容被截断 | 添加水平滚动或换行提示 |
| 内容包含控制字符 | 直接渲染 | 添加内容清理 |
| 极大内容（>1000行） | 只显示前 maxLines 行 | 考虑虚拟滚动 |

### 6.3 改进建议

#### 6.3.1 性能优化
```typescript
// 建议：添加内容哈希或 useMemo 缓存 applyMarkdown 结果
const rendered = useMemo(
  () => applyMarkdown(content, theme, highlight),
  [content, theme, highlight]
);
```

#### 6.3.2 可访问性
- 添加键盘导航支持（已在父组件 PreviewQuestionView 实现）
- 考虑添加屏幕阅读器友好的内容摘要

#### 6.3.3 功能扩展
- 支持水平滚动（当前仅垂直截断）
- 支持内容搜索/高亮
- 支持复制预览内容到剪贴板

#### 6.3.4 代码结构
- `PreviewBoxBody` 函数过长（约 140 行），可拆分为更小的子函数
- 截断逻辑和边框渲染可提取为独立组件

### 6.4 测试建议
- 单元测试：不同宽度下的内容截断
- 集成测试：与 `applyMarkdown` 和 `sliceAnsi` 的协作
- 视觉回归测试：不同终端尺寸下的布局
- 性能测试：大 Markdown 文档的渲染性能

---

## 7. 相关类型定义

```typescript
// 来自 AskUserQuestionTool.tsx
interface QuestionOption {
  label: string;
  description: string;
  preview?: string;  // 传递给 PreviewBox 的 content
}

// 来自 use-multiple-choice-state.ts
interface QuestionState {
  selectedValue?: string | string[];
  textInputValue: string;
}
```

---

## 8. 总结

`PreviewBox` 是一个功能完整的终端预览组件，通过精心设计的边框、智能截断和 Markdown 渲染，为用户提供了优雅的选项预览体验。其核心设计亮点包括：

1. **分层架构**：入口组件 + Suspense 包装器 + 核心渲染体
2. **异步高亮**：不阻塞主渲染流程
3. **宽度感知**：正确处理 CJK 和 Emoji 的显示宽度
4. **优雅降级**：设置禁用高亮时无缝切换

主要关注点在于性能优化（Markdown 重复解析）和边界情况处理（极端终端尺寸）。
