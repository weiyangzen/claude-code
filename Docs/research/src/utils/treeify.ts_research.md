# 研究文档：src/utils/treeify.ts

## 场景与职责

本模块是一个**终端友好的对象树形可视化工具**，用于将嵌套的 JavaScript 对象渲染为带缩进和分支符号的文本树。它在 Claude Code 中主要用于展示验证错误列表、配置结构等需要层次化呈现的数据。与通用的 `console.log` 或 `JSON.stringify` 不同，它支持 Ink 主题颜色、循环引用检测，并能优雅处理空 key、函数值等边界情况。

## 功能点目的

| 导出符号 | 目的 |
|---------|------|
| `treeify(obj, options?)` | 将 `TreeNode` 对象递归渲染为字符串树。 |
| `TreeNode` | 类型定义：任意嵌套的对象结构，叶子可以是 `string` 或 `undefined`。 |
| `TreeifyOptions` | 配置选项：是否显示值、是否隐藏函数、是否启用颜色、主题名、自定义颜色键。 |

## 具体技术实现

### 1. 树形字符与颜色

- 使用 `figures` npm 包提供跨平台的 Unicode 树形字符：
  - `├` (`lineUpDownRight`) — 非最后一个分支
  - `└` (`lineUpRight`) — 最后一个分支
  - `│` (`lineVertical`) — 垂直延续线
  - ` ` (空格) — 最后一项后的空白填充
- 颜色通过 `src/components/design-system/color.js` 的 `color(colorKey, themeName)` 函数实现，支持 `treeChar`（分支符）、`key`（属性名）、`value`（值）分别着色。

### 2. 递归渲染逻辑 `growBranch`

函数签名：`growBranch(node, prefix, _isLast, depth)`

- **字符串叶子**：直接输出 `prefix + value`。
- **非对象/null 叶子**：在 `showValues === true` 时输出 `String(node)`；否则只输出 key。
- **循环引用检测**：维护模块级 `visited = new WeakSet<object>()`。遇到已访问对象时输出 `[Circular]` 并终止递归。
- **对象节点**：
  - 过滤 key（若 `hideFunctions === true` 则跳过 `typeof value === 'function'`）。
  - 对每个 key 判断是否为最后一个，选择 `├` 或 `└`。
  - 计算下一级 `prefix`：当前 prefix + (`│` 或 ` `) + `' '`。
  - 递归调用 `growBranch`。

### 3. 特殊值处理

- **Array**：不展开，统一显示为 `[Array(N)]`，避免长数组撑爆输出。
- **函数**：若 `showValues` 为 true 则显示 `[Function]`；若 `hideFunctions` 为 true 则在 key 过滤阶段直接跳过。
- **空/空白 key**：当 key 全为空白字符时，不显示 key 本身，也不加冒号，仅输出分支符和值。这是为了支持某些以空字符串为 key 的 hacky 数据结构。

### 4. 顶层特殊处理

- 若对象为空（`Object.keys(obj).length === 0`），返回 `(empty)`。
- 若对象只有一个 key 且该 key 为空字符串、值为字符串，则直接返回 `└ value`（避免不必要的缩进）。

## 关键代码路径与文件引用

- **主实现**：`src/utils/treeify.ts`（170 行）
- **调用方（验证错误展示）**：`src/components/ValidationErrorsList.tsx`
- **颜色系统**：`src/components/design-system/color.js`
- **主题类型**：`src/utils/theme.ts`

## 依赖与外部交互

- **`figures`**：提供跨平台 Unicode 线条字符。
- **`src/components/design-system/color.js`**：`color(text, colorKey)` 着色函数。
- **`src/utils/theme.ts`**：`Theme`、`ThemeName` 类型定义。

## 风险、边界与改进建议

### 风险

1. **栈溢出**：对极端深层嵌套的对象（如递归生成的深度 > 1000 的结构），`growBranch` 的递归调用可能导致 JavaScript 调用栈溢出。当前无任何深度限制或尾递归优化。
2. **`visited` WeakSet 的共享生命周期**：`visited` 在每次 `treeify` 调用时新建，因此同一次调用内可检测循环引用，但不同调用之间不共享。这是正确的设计，因为不同调用应独立渲染。
3. **Array 信息丢失**：所有数组都被压缩为 `[Array(N)]`，对于需要查看数组前几个元素摘要的场景不够友好。

### 边界

- **仅支持 `TreeNode` 类型**：输入对象必须是普通对象或字符串；不支持 Map、Set、Date 等特殊类型的原生美化显示（Date 会被 `String(node)` 转为字符串）。
- **颜色函数依赖 Ink 运行时**：虽然 `treeify` 本身不依赖 React/Ink，但默认的 `color()` 实现是为终端 UI 设计的。若在纯 Node.js 脚本中使用，需要传入自定义的 `treeCharColors` 或空 colorize 逻辑。
- **无宽度限制**：输出字符串的长度不受控制，超宽对象可能导致终端自动换行，破坏树的视觉对齐。

### 改进建议

1. **增加 `maxDepth` 选项**：限制递归深度，超出时显示 `[Depth limit]` 或 `[Object]`，防止栈溢出和输出爆炸。
2. **数组摘要模式**：增加 `showArrayPreview: number` 选项，允许显示数组的前 N 个元素（如 `[1, 2, 3, ... + 47 more]`），在信息量和可读性之间取得平衡。
3. **支持更多原生类型**：为 `Date`、`RegExp`、`Error` 等常见类型提供默认的 `[Date: ...]`、`[RegExp: ...]` 显示。
4. **宽度感知截断**：在生成每一行时考虑终端宽度，对超长值进行尾部截断（可复用 `src/utils/truncate.ts` 的能力），保持树形结构不被换行破坏。
5. **循环引用路径显示**：当前仅显示 `[Circular]`，若能显示循环引用的路径（如 `[Circular ~.foo.bar]`），对调试更有价值。
