# wrapAnsi.ts 研究文档

## 场景与职责

`wrapAnsi.ts` 是 `wrap-text.ts` 的底层依赖，提供 ANSI 安全的文本换行功能。该模块封装了 `wrap-ansi` npm 包和 Bun 原生实现，提供统一的换行接口。

### 核心职责
1. **平台适配**: 自动选择 Bun 原生或 npm 实现
2. **ANSI 安全**: 正确处理 ANSI 转义序列
3. **硬换行**: 在指定列强制换行
4. **修剪选项**: 支持去除行尾空格

## 功能点目的

### 1. 平台优化
检测运行环境，优先使用 Bun 原生实现：
```typescript
const wrapAnsiBun =
  typeof Bun !== 'undefined' && typeof Bun.wrapAnsi === 'function'
    ? Bun.wrapAnsi
    : null
```

**Bun 优势**:
- 原生实现，性能更高
- 与 Bun 运行时深度集成
- 减少 npm 依赖

**Node 回退**:
- 使用 `wrap-ansi` npm 包
- 保证跨平台兼容性

### 2. 统一接口
```typescript
const wrapAnsi: (
  input: string,
  columns: number,
  options?: WrapAnsiOptions,
) => string = wrapAnsiBun ?? wrapAnsiNpm
```

无论底层实现如何，提供一致的 API。

### 3. 换行选项
```typescript
type WrapAnsiOptions = {
  hard?: boolean      // 硬换行（强制在列边界换行）
  wordWrap?: boolean  // 按词边界换行
  trim?: boolean      // 去除行尾空格
}
```

## 具体技术实现

### 实现选择逻辑
```typescript
// 模块加载时确定实现
const bunStringWidth =
  typeof Bun !== 'undefined' && typeof Bun.stringWidth === 'function'
    ? Bun.stringWidth
    : null

// 运行时直接使用，无额外检查
const wrapAnsi: (...) => string = wrapAnsiBun ?? wrapAnsiNpm
```

**性能考虑**:
- 在模块加载时解析实现，避免每次调用的 `typeof` 检查
- 利用 JavaScript 引擎的属性访问优化

### Bun 原生特性
Bun 的 `wrapAnsi` 实现特点：
- 用 Zig 编写，性能优于 JavaScript
- 正确处理复杂的 ANSI 序列
- 支持所有标准选项

### npm 包特性
`wrap-ansi` 包特点：
- 成熟的 ANSI 处理
- 广泛的测试覆盖
- 与 `string-width`、`strip-ansi` 等包协同工作

## 关键代码路径与文件引用

### 依赖
```typescript
import wrapAnsiNpm from 'wrap-ansi'
```

### 调用方
- **`wrap-text.ts`**: 主要的换行调用方
- **其他模块**: 需要 ANSI 安全换行的场景

### 使用示例
```typescript
import { wrapAnsi } from './wrapAnsi.js'

// 基础换行
wrapAnsi('Hello World', 5)  // 'Hello\nWorld'

// 硬换行
wrapAnsi('Hello World', 5, { hard: true })

// 修剪空格
wrapAnsi('Hello   World', 8, { trim: true })
```

## 依赖与外部交互

| 依赖 | 用途 |
|------|------|
| `wrap-ansi` | Node 环境的换行实现 |
| `Bun.wrapAnsi` | Bun 运行时的原生实现 |

### 外部交互
- **Bun 运行时**: 检测并使用原生 API
- **Node.js**: 回退到 npm 包

## 风险、边界与改进建议

### 已知风险
1. **行为差异**: Bun 和 npm 实现在边界情况可能有细微差异
2. **版本同步**: Bun 和 npm 包版本可能不同步
3. **测试覆盖**: 需要确保两种路径都有测试

### 边界情况
1. **空字符串**: 返回空字符串
2. **零宽度**: `columns <= 0` 行为未定义
3. **纯 ANSI**: 纯转义序列的换行行为
4. **宽字符**: 在宽字符中间换行的处理

### 改进建议
1. **一致性测试**: 添加测试确保两种实现行为一致
2. **类型定义**: 为 Bun API 添加完整类型定义
3. **功能检测**: 更完善的功能检测，而非仅 `typeof` 检查
4. **选项验证**: 添加选项参数验证
5. **性能基准**: 建立性能基准测试

### 相关库
- [wrap-ansi](https://www.npmjs.com/package/wrap-ansi) - npm 包
- [Bun.wrapAnsi](https://bun.sh/docs/api/utils) - Bun 文档
- [slice-ansi](https://www.npmjs.com/package/slice-ansi) - 相关工具
