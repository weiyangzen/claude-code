# 研究文档：src/native-ts/yoga-layout/enums.ts

## 场景与职责

`enums.ts` 是项目内纯 TypeScript 版 yoga-layout 的**枚举常量定义文件**。它的核心职责是为 flexbox 布局引擎提供一组与上游 Meta `yoga-layout` C++/WASM 版本数值完全兼容的常量定义。该文件被设计为"零运行时开销"的类型安全枚举源，供同目录下的 `index.ts`（布局引擎主实现）以及 Ink 渲染层的布局适配器使用。

项目选择自研 TS 版 yoga 而非引入官方 WASM 包，主要动机是：
- 避免 WASM 线性内存增长与手动释放的复杂性；
- 消除异步 `loadYoga()` 带来的预加载/切换/重置机制；
- 在 Node.js 终端 UI（Ink）场景下获得更可控的性能与调试能力。

## 功能点目的

| 枚举 | 用途 |
|------|------|
| `Align` | 控制交叉轴对齐（`align-items` / `align-self` / `align-content`），含 `Stretch`、`Baseline`、`SpaceBetween` 等 9 种值 |
| `BoxSizing` | 盒模型模式（`BorderBox`、`ContentBox`），目前 Ink 只使用 `BorderBox` |
| `Dimension` | 区分宽高维度（`Width`、`Height`），用于内部轴切换计算 |
| `Direction` | 文本方向（`Inherit`、`LTR`、`RTL`），Ink 固定传入 `LTR` |
| `Display` | 显示模式（`Flex`、`None`、`Contents`），支持 CSS `display: contents` 的透明提升行为 |
| `Edge` | 9 边模型（`Left`/`Top`/`Right`/`Bottom`/`Start`/`End`/`Horizontal`/`Vertical`/`All`），用于 margin/padding/border/position 的级联回退解析 |
| `Errata` | Yoga 兼容性标志位（位掩码），如 `StretchFlexBasis`、`Classic` |
| `ExperimentalFeature` | 实验特性开关，当前仅 `WebFlexBasis` |
| `FlexDirection` | 主轴方向（`Column`、`ColumnReverse`、`Row`、`RowReverse`） |
| `Gutter` | 间隙轴（`Column`、`Row`、`All`），对应 CSS `gap` |
| `Justify` | 主轴内容分布（`FlexStart`、`Center`、`FlexEnd`、`SpaceBetween`、`SpaceAround`、`SpaceEvenly`） |
| `MeasureMode` | 尺寸约束模式（`Undefined`、`Exactly`、`AtMost`），决定子节点在测量阶段如何解释可用空间 |
| `Overflow` | 溢出处理（`Visible`、`Hidden`、`Scroll`），`Scroll` 会影响容器在 `AtMost` 模式下的尺寸 clamp 行为 |
| `PositionType` | 定位类型（`Static`、`Relative`、`Absolute`） |
| `Unit` | 值单位（`Undefined`、`Point`、`Percent`、`Auto`），配合 `Value` 类型解析为实际像素值 |
| `Wrap` | 换行模式（`NoWrap`、`Wrap`、`WrapReverse`） |

## 具体技术实现

### const 对象 + 类型别名模式

文件严格遵循仓库约定，**不使用 TypeScript `enum`**，而是采用 `as const` 对象配合索引类型别名：

```ts
export const Align = {
  Auto: 0,
  FlexStart: 1,
  // ...
} as const
export type Align = (typeof Align)[keyof typeof Align]
```

这种模式的优势：
1. **Tree-shaking 友好**：编译后生成普通对象，可被 ESM 摇树优化；
2. **类型安全**：`Align` 类型只能取对象值的联合，避免魔法数字；
3. **无 TS enum 的反向映射污染**：不会生成额外的对象属性；
4. **运行时调试友好**：在 DevTools 中可直接看到 `Align.FlexStart` 而非裸数字。

### 数值与上游严格对齐

文件头注释明确说明 "Values match upstream exactly so callers don't change"。这意味着：
- 任何修改枚举数值的行为都会破坏与上游 yoga C++ 版本的二进制/逻辑兼容性；
- `index.ts` 内部大量直接使用裸数字做快速路径判断（如 `v.unit === 3` 代表 `Auto`），因此枚举值变更必须同步修改引擎内部硬编码。

## 关键代码路径与文件引用

### 被导入方

- **`src/native-ts/yoga-layout/index.ts`**（行 41-58）
  - `index.ts` 将 `enums.ts` 中除 `BoxSizing` 外的全部枚举导入，并在行 60-77 重新导出，作为公共 API 的一部分。
  - 引擎内部大量条件分支直接比较这些枚举值（如 `if (v.unit === Unit.Auto)`、`if (style.flexWrap !== Wrap.NoWrap)`）。

- **`src/ink/layout/yoga.ts`**（行 1-14）
  - Ink 的布局适配器从 `index.ts` 批量导入枚举，用于将 Ink 的字符串风格 API（如 `'flex-start'`）映射为 yoga 的数值枚举。
  - 具体映射逻辑分散在 `YogaLayoutNode` 的各个 setter 中（如 `setAlignItems`、`setJustifyContent`、`setFlexWrap` 等）。

### 无外部依赖

`enums.ts` 是纯粹的常量声明文件，不依赖任何其他模块，也不包含副作用。

## 依赖与外部交互

| 方向 | 实体 | 关系说明 |
|------|------|----------|
| 被依赖 | `src/native-ts/yoga-layout/index.ts` | 导入并重新导出全部枚举，作为布局引擎公共 API |
| 被依赖 | `src/ink/layout/yoga.ts` | 通过 `index.ts` 的重新导出使用枚举，完成字符串风格到数值枚举的映射 |
| 被依赖 | `src/ink/reconciler.ts` | 仅使用 `getYogaCounters`，不直接使用枚举 |
| 被依赖 | `src/ink/ink.tsx` | 同上，仅用于性能计数器读取 |

## 风险、边界与改进建议

### 风险

1. **数值硬编码耦合**：`index.ts` 内部存在多处直接写死枚举数值的优化（如 `resolveEdges4Into` 中 `v.unit === 0` 代表 `Undefined`、`_hasAutoMargin` 检查中 `unit === 3` 代表 `Auto`）。若 `enums.ts` 的值被修改而 `index.ts` 未同步更新，会导致静默逻辑错误。
2. **无自动化同步机制**：文件注释说明是 "ported from yoga-layout/src/generated/YGEnums.ts"，但仓库内没有脚本或 CI 步骤校验与上游 yoga 的枚举值一致性。上游新增枚举值时可能遗漏同步。
3. **BoxSizing 的悬空实现**：`BoxSizing` 枚举已导出，但 `index.ts` 中 `setBoxSizing` 是空实现（注释 "Not implemented — Ink doesn't use content-box"）。若未来 Ink 使用 `content-box`，需要补全引擎逻辑。

### 边界

- `Errata.Classic` 值为 `2147483646`（`INT_MAX - 1`），`Errata.All` 为 `2147483647`（`INT_MAX`），这是 Yoga 的位掩码设计，在 32 位有符号整数范围内安全。
- `Edge` 的 `Start`/`End` 在 `index.ts` 中被硬编码映射为 `Left`/`Right`（LTR 假设），RTL 场景未实现。

### 改进建议

1. **增加静态断言**：在 `index.ts` 顶部增加编译期或启动期断言，校验关键枚举值（如 `Unit.Undefined === 0`、`Unit.Auto === 3`），防止人为修改 `enums.ts` 后破坏引擎。
2. **文档化裸数字含义**：`index.ts` 中大量 `v.unit === 0`、`v.unit === 1` 等快速路径应补充行内注释或改为引用 `Unit.Undefined` 等常量，在牺牲极小性能的前提下提升可维护性。
3. **同步脚本**：若项目长期维护自研 yoga，建议增加一个轻量脚本，对比上游 `YGEnums.ts` 的生成结果与本文件，检测新增/变更枚举。
