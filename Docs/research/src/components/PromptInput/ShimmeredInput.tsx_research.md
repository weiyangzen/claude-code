# ShimmeredInput.tsx 研究文档

## 场景与职责

导出 `HighlightedInput` 组件，被 `BaseTextInput.tsx` 调用，用于在终端输入框中渲染带颜色、反色、闪烁（shimmer）效果的文本。典型场景包括 `ultrathink` 彩虹高亮、`/command` 蓝色高亮、`[Image #N]` 反色选中态。

## 功能点目的

- 按 `TextHighlight[]` 分段并独立着色
- 正确处理 `\n` 多行
- 对带 `shimmerColor` 的区域以 20fps 驱动扫过光点动画
- React Compiler memo 缓存避免每帧重算分段

## 具体技术实现

### 关键流程

1. **分段计算**：调用 `segmentTextByHighlights(text, highlights)` 得 `TextSegment[]`，再按 `\n` 拆分为 `LinePart[]`，记录 `start` 偏移。结果 memo 缓存。
2. **Shimmer 范围**：扫描 `shimmerColor`，取最小 `start` 为 `lo`，最大 `end` 为 `hi`；`sweepStart = lo - 10`，`cycleLength = hi - lo + 20`。
3. **动画驱动**：`useAnimationFrame(hasShimmer ? 50 : null)` 得 `time`，计算 `glimmerIndex = sweepStart + Math.floor(time / 50) % cycleLength`。
4. **渲染**：有 `shimmerColor` 的 part 逐字符渲染为 `<ShimmerChar index={part.start + charIndex} glimmerIndex={glimmerIndex} ... />`；无 shimmer 的直接用 `<Text>` 渲染。

### 数据结构

```ts
type Props = { text: string; highlights: TextHighlight[] }
type LinePart = { text: string; highlight?: TextHighlight; start: number }
```

- `TextHighlight` 来自 `src/utils/textHighlighting.ts`

### 协议/命令

- `useAnimationFrame` → `src/ink.js`
- 颜色值为 `keyof Theme` → `src/utils/theme.ts`

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/components/PromptInput/ShimmeredInput.tsx` | 本组件 |
| `src/components/BaseTextInput.tsx` | 调用方 |
| `src/utils/textHighlighting.ts` | 分段逻辑与类型 |
| `src/components/Spinner/ShimmerChar.tsx` | 单字符 shimmer 子组件 |
| `src/utils/theme.ts` | 主题色定义 |
| `src/ink.js` | UI 组件与动画 hook |

## 依赖与外部交互

- React Compiler 编译，使用 `_c(23)` 缓存槽位
- `segmentTextByHighlights` 内部用 `@alcalzone/ansi-tokenize` 处理 ANSI 转义
- `ShimmerChar` 对 `index === glimmerIndex` 或相邻位置使用 `shimmerColor`

## 风险、边界与改进建议

1. **长文本性能**：输入极长且 shimmer 密集时，逐字符拆分产生大量 React 节点
2. **空行占位**：空行渲染 `<Text> </Text>` 维持布局高度，但复制粘贴可能多出一个空格
3. **越界防御**：若 shimmer 范围异常，`lo/hi` 可能为 `Infinity/-Infinity`，建议加防御校验
