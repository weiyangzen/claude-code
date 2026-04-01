# bidi.ts 研究文档

## 场景与职责

`bidi.ts` 实现双向文本（Bidirectional Text）重排序功能，解决终端中 RTL（从右到左）语言（如希伯来语、阿拉伯语）的显示问题。

### 问题背景

- **macOS 终端**（Terminal.app、iTerm2）原生支持 Unicode Bidi 算法
- **Windows 终端**（包括 Windows Terminal、conhost、WSL）不实现 Bidi 支持
- 没有软件级 Bidi 处理时，RTL 文本会反向显示

### 核心职责

1. 检测是否需要软件级 Bidi 重排序
2. 对 RTL 文本应用 Unicode Bidi 算法
3. 将逻辑顺序的字符数组转换为视觉顺序

## 功能点目的

### 1. 平台检测
自动检测运行环境，判断是否需要软件 Bidi 支持：
- Windows 平台（`process.platform === 'win32'`）
- Windows Terminal（`WT_SESSION` 环境变量）
- VS Code 集成终端（`TERM_PROGRAM === 'vscode'`）

### 2. RTL 字符检测
快速检测文本中是否包含 RTL 字符，避免对纯 LTR 文本运行完整的 Bidi 算法：
- 希伯来语：U+0590-U+05FF, U+FB1D-U+FB4F
- 阿拉伯语：U+0600-U+06FF, U+0750-U+077F, U+08A0-U+08FF, U+FB50-U+FDFF, U+FE70-U+FEFF
- 塔纳文（Thaana）：U+0780-U+07BF
- 叙利亚文（Syriac）：U+0700-U+074F

### 3. Bidi 重排序
使用 `bidi-js` 库实现标准的 Unicode Bidi 算法：
- 获取嵌入层级（embedding levels）
- 按层级反转连续片段
- 保持 `ClusteredChar` 数据结构完整

## 具体技术实现

### 核心数据结构

```typescript
// 聚簇字符 - Ink 渲染管线的基本单元
type ClusteredChar = {
  value: string;      // 字符值（可能多字节）
  width: number;      // 显示宽度（CJK/emoji 为 2）
  styleId: number;    // 样式 ID（用于颜色/样式）
  hyperlink: string | undefined;  // 超链接 URL
};
```

### 平台检测逻辑

```typescript
function needsBidi(): boolean {
  if (needsSoftwareBidi === undefined) {
    needsSoftwareBidi =
      process.platform === 'win32' ||
      typeof process.env['WT_SESSION'] === 'string' ||
      process.env['TERM_PROGRAM'] === 'vscode'
  }
  return needsSoftwareBidi
}
```

### Bidi 重排序算法

```typescript
export function reorderBidi(characters: ClusteredChar[]): ClusteredChar[] {
  // 1. 快速退出检查
  if (!needsBidi() || characters.length === 0) return characters
  
  // 2. 构建纯文本字符串
  const plainText = characters.map(c => c.value).join('')
  
  // 3. RTL 字符快速检测
  if (!hasRTLCharacters(plainText)) return characters
  
  // 4. 获取 Bidi 层级
  const bidi = getBidi()
  const { levels } = bidi.getEmbeddingLevels(plainText, 'auto')
  
  // 5. 映射层级到 ClusteredChar 索引
  const charLevels: number[] = []
  let offset = 0
  for (let i = 0; i < characters.length; i++) {
    charLevels.push(levels[offset]!)
    offset += characters[i]!.value.length
  }
  
  // 6. 按层级反转（标准 Bidi 重排序）
  const reordered = [...characters]
  const maxLevel = Math.max(...charLevels)
  
  for (let level = maxLevel; level >= 1; level--) {
    // 反转所有连续且层级 >= level 的片段
    // ...
  }
  
  return reordered
}
```

### 关键算法细节

**层级映射**：
- `bidi-js` 按 Unicode 码点返回层级
- `ClusteredChar` 可能包含多字节字符（如 emoji）
- 需要正确映射字符索引到层级索引

**重排序实现**：
- 使用标准 Bidi 算法：从最大层级到 1，反转所有连续且层级 >= 当前层级的片段
- 同时反转字符数组和层级数组，保持对应关系

## 关键代码路径与文件引用

### 入口与导出
- **文件**：`src/ink/bidi.ts`
- **导出函数**：`reorderBidi`

### 依赖关系

**被导入**：
- `bidi-js` - Unicode Bidi 算法实现

**导入使用**：
```typescript
import bidiFactory from 'bidi-js'
```

### 关键函数

| 函数 | 职责 | 行号 |
|------|------|------|
| `reorderBidi` | 主入口：重排序字符数组 | 53-105 |
| `needsBidi` | 检测是否需要软件 Bidi | 29-37 |
| `getBidi` | 获取/创建 bidi-js 实例 | 39-44 |
| `hasRTLCharacters` | 快速 RTL 字符检测 | 131-139 |
| `reverseRange` | 反转数组片段（泛型） | 107-115 |
| `reverseRangeNumbers` | 反转数组片段（number[]） | 117-125 |

### 相关文件

- `src/ink/output.ts` - 调用 `reorderBidi` 进行输出前的文本处理
- `src/ink/render-node-to-output.ts` - 渲染管线中可能涉及

## 依赖与外部交互

### 外部库

**bidi-js**：
- 纯 JavaScript 实现的 Unicode Bidi 算法
- 提供 `getEmbeddingLevels` 函数获取每个字符的层级
- 使用 `'auto'` 方向自动检测文本方向

### 数据结构交互

`ClusteredChar` 是 Ink 渲染管线的核心数据结构：
- 在渲染过程中维护字符的完整上下文（样式、超链接）
- Bidi 重排序保持这些属性与字符的关联

### 调用时机

```
渲染管线
    ↓
文本测量/布局
    ↓
输出生成
    ↓
reorderBidi() - 在写入屏幕缓冲区前重排序
    ↓
屏幕缓冲区
```

## 风险、边界与改进建议

### 已知风险

1. **性能开销**：
   - 每次渲染都需检测 RTL 字符
   - 完整 Bidi 算法有一定计算成本
   - 对于混合 LTR/RTL 的复杂文本，层级计算可能较重

2. **字符宽度处理**：
   - `ClusteredChar` 的 `value` 可能包含多字节字符
   - 层级映射时需要正确处理字符长度

3. **缓存策略**：
   - `bidiInstance` 和 `needsSoftwareBidi` 使用模块级缓存
   - 但重排序结果没有缓存，相同文本每次重新计算

### 边界情况

1. **空数组**：直接返回原数组
2. **纯 LTR 文本**：`hasRTLCharacters` 快速返回，避免完整 Bidi 计算
3. **单字符**：层级计算和反转逻辑正确处理
4. **环境变量变化**：`needsSoftwareBidi` 在模块加载时确定，运行时环境变化不重新检测

### 改进建议

1. **性能优化**：
   - 添加 LRU 缓存缓存重排序结果
   - 考虑使用 Web Worker 处理复杂 Bidi 文本
   
   ```typescript
   // 示例：添加缓存
   const reorderCache = new Map<string, ClusteredChar[]>()
   ```

2. **功能扩展**：
   - 支持显式方向标记（LRM、RLM、PDF）
   - 添加对竖排文本的支持

3. **配置选项**：
   - 允许用户强制启用/禁用 Bidi 处理
   - 支持自定义 RTL 字符范围

4. **测试覆盖**：
   - 添加复杂混合方向文本的测试
   - 测试各种 emoji 和组合字符的处理
   - 验证 Windows/macOS/Linux 平台行为一致性

5. **代码改进**：
   - `reverseRange` 和 `reverseRangeNumbers` 可以合并为泛型函数
   - 考虑使用 TypedArray 优化大量字符的处理
