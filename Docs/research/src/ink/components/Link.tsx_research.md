# Link.tsx 研究文档

## 场景与职责

`Link` 是 Ink 框架中的超链接组件，提供终端中的可点击链接功能：

1. **OSC 8 超链接**: 使用 ANSI OSC 8 转义序列创建终端原生超链接
2. **终端兼容性检测**: 检测终端是否支持超链接，不支持时显示回退内容
3. **灵活的内容**: 支持自定义链接文本或使用 URL 作为文本

Link 组件使得在终端应用中可以直接嵌入可点击的 URL，用户可以通过 Cmd/Ctrl+点击在浏览器中打开链接。

## 功能点目的

### 1. 超链接支持检测
- **supportsHyperlinks()**: 检测当前终端是否支持 OSC 8 超链接
- **扩展终端支持**: 除了 supports-hyperlinks 库检测的终端外，额外支持 Ghostty、Hyper、kitty、alacritty、iTerm 等

### 2. 链接渲染
- **支持超链接**: 渲染为 `<ink-link>` 元素，包含 href 属性
- **不支持超链接**: 渲染为普通文本，可选显示 fallback 内容

### 3. 内容灵活性
- **children**: 自定义链接显示文本
- **url**: 链接目标 URL
- **fallback**: 不支持超链接时的回退内容，未提供时使用 children 或 url

## 具体技术实现

### 关键数据结构

```typescript
export type Props = {
  readonly children?: ReactNode;  // 链接显示文本
  readonly url: string;           // 链接目标（必需）
  readonly fallback?: ReactNode;  // 不支持时的回退内容
};
```

### 关键流程

1. **内容确定**
   ```typescript
   const content = children ?? url;
   ```
   - 优先使用 children 作为显示文本
   - 如果没有 children，使用 url 作为显示文本

2. **终端能力检测**
   ```typescript
   if (supportsHyperlinks()) {
     // 渲染为 ink-link
   }
   ```

3. **条件渲染**
   ```typescript
   if (supportsHyperlinks()) {
     return (
       <Text>
         <ink-link href={url}>{content}</ink-link>
       </Text>
     );
   }
   return <Text>{fallback ?? content}</Text>;
   ```

### 代码路径

```
Link.tsx
├── 导入依赖
│   ├── React
│   ├── supportsHyperlinks: 终端能力检测
│   └── Text: 文本容器
├── Props 类型定义
├── Link 函数组件
│   ├── 确定 content (children ?? url)
│   ├── 检测 supportsHyperlinks()
│   ├── 支持: 渲染 <ink-link href={url}>{content}</ink-link>
│   └── 不支持: 渲染 <Text>{fallback ?? content}</Text>
└── 默认导出
```

## 关键代码路径与文件引用

### 核心依赖

| 文件 | 用途 |
|------|------|
| `src/ink/supports-hyperlinks.ts` | 终端超链接支持检测 |
| `src/ink/components/Text.tsx` | 文本容器组件 |

### supportsHyperlinks 实现

```typescript
// src/ink/supports-hyperlinks.ts
import supportsHyperlinksLib from 'supports-hyperlinks';

export const ADDITIONAL_HYPERLINK_TERMINALS = [
  'ghostty', 'Hyper', 'kitty', 'alacritty', 'iTerm.app', 'iTerm2',
];

export function supportsHyperlinks(options?: SupportsHyperlinksOptions): boolean {
  // 1. 检查 supports-hyperlinks 库的结果
  if (supportsHyperlinksLib.stdout) return true;
  
  // 2. 检查 TERM_PROGRAM
  const termProgram = env['TERM_PROGRAM'];
  if (ADDITIONAL_HYPERLINK_TERMINALS.includes(termProgram)) return true;
  
  // 3. 检查 LC_TERMINAL（tmux 中保留的原始终端）
  const lcTerminal = env['LC_TERMINAL'];
  if (ADDITIONAL_HYPERLINK_TERMINALS.includes(lcTerminal)) return true;
  
  // 4. 检查 TERM（kitty 设置 xterm-kitty）
  const term = env['TERM'];
  if (term?.includes('kitty')) return true;
  
  return false;
}
```

### 渲染管线

```
Link 组件
├── <Text> 容器（确保在文本上下文中）
└── <ink-link href={url}>
    └── content

reconciler.ts
├── 识别 ink-link 元素
├── 设置 href 属性
└── 存储 hyperlink 信息

render-node-to-output.ts
├── 处理 ink-text 的子节点
├── 提取 hyperlink 属性
└── 使用 wrapWithOsc8Link 包装文本

termio/osc.ts
└── 生成 OSC 8 转义序列
    OSC 8 ;; params BEL text OSC 8 ;; BEL
```

### ink-link 渲染

在 `src/ink/render-node-to-output.ts` 中：

```typescript
// OSC 8 超链接转义序列
const OSC = '\u001B]';
const BEL = '\u0007';

function wrapWithOsc8Link(text: string, url: string): string {
  return `${OSC}8;;${url}${BEL}${text}${OSC}8;;${BEL}`;
}
```

## 依赖与外部交互

### 运行时依赖

1. **supports-hyperlinks**: 检测终端超链接支持
2. **React Compiler**: 使用 `_c` 函数进行自动记忆化
3. **Text 组件**: 作为容器确保在文本上下文中

### Props 交互

| Prop | 说明 |
|------|------|
| url | 链接目标 URL（必需） |
| children | 链接显示文本（可选，默认使用 url） |
| fallback | 不支持超链接时的回退内容（可选，默认使用 children 或 url） |

### 终端兼容性

| 终端 | 支持状态 |
|------|----------|
| iTerm2 | 支持（原生 + LC_TERMINAL 检测） |
| kitty | 支持（原生 + TERM 检测） |
| Ghostty | 支持（ADDITIONAL_HYPERLINK_TERMINALS） |
| alacritty | 支持（ADDITIONAL_HYPERLINK_TERMINALS） |
| Hyper | 支持（ADDITIONAL_HYPERLINK_TERMINALS） |
| VS Code | 部分支持（xterm.js 有独立处理） |
| tmux | 依赖底层终端（LC_TERMINAL 检测） |

### 与 VS Code 的交互

```typescript
// App.tsx 中的注释说明
// xterm.js (VS Code, Cursor, Windsurf, etc.) has its own OSC 8 link
// handler that fires on Cmd+click without consuming the mouse event.
// Let xterm.js own link-opening; Cmd+click is the native UX there anyway.
if (url && process.env.TERM_PROGRAM !== 'vscode' && !isXtermJs()) {
  // 设置点击打开浏览器的 timer
}
```

## 风险、边界与改进建议

### 已知风险

1. **VS Code 双重打开**: xterm.js 有自己的链接处理器，可能导致链接被打开两次
   - 已在 App.tsx 中通过检测 TERM_PROGRAM 和 isXtermJs() 避免

2. **双击选择冲突**: 双击链接会选择单词而不是打开链接
   - 已在 App.tsx 中通过 MULTI_CLICK_TIMEOUT_MS 延迟处理

3. **终端兼容性**: 不同终端对 OSC 8 的支持程度不同

### 边界情况

1. **空 url**: 组件没有验证 url 是否为空，可能导致无效链接
2. **特殊字符**: url 中的特殊字符可能需要编码
3. **长 URL**: 非常长的 URL 可能影响布局

### 改进建议

1. **URL 验证**: 添加对 url 的基本验证
   ```typescript
   if (!url || !isValidURL(url)) {
     console.warn('Invalid URL provided to Link component');
     return <Text>{fallback ?? children ?? url}</Text>;
   }
   ```

2. **URL 编码**: 自动对 url 进行编码处理
   ```typescript
   const encodedUrl = encodeURI(url);
   ```

3. **样式支持**: 允许为链接设置样式（颜色、下划线等）
   ```typescript
   type Props = {
     // ... 现有 props
     readonly color?: Color;
     readonly underline?: boolean;
   };
   ```

4. **点击反馈**: 添加点击时的视觉反馈

5. **链接预览**: 在悬停时显示链接目标（如果终端支持）

6. **无障碍支持**: 添加 aria-label 或类似的无障碍属性

### 代码质量

- 组件简单清晰，职责单一
- 使用 React Compiler 进行记忆化
- 条件渲染逻辑明确

### 测试建议

- 测试在不同终端下的渲染行为
- 测试 supportsHyperlinks 的检测逻辑
- 测试 fallback 内容的显示
- 测试 URL 编码处理
