# hash.ts 深度研究

## 场景与职责

本模块提供字符串和内容哈希功能，用于缓存键生成、变化检测和内容去重。它抽象了不同运行时的哈希实现差异（Bun vs Node.js），提供统一的哈希 API。

**核心场景：**
1. **缓存键生成**：为文件内容、配置等生成唯一标识
2. **变化检测**：比较内容哈希判断是否需要更新
3. **内容去重**：识别重复内容
4. **磁盘路径生成**：生成稳定的缓存目录名

## 功能点目的

### 1. djb2 哈希
- **目的**：快速、确定性的字符串哈希
- **特点**：
  - 非加密安全
  - 跨运行时确定性（与 Bun.hash 不同）
  - 返回有符号 32 位整数
- **场景**：缓存目录名（需要运行时升级后仍稳定）

### 2. 内容哈希
- **目的**：通用内容变化检测
- **实现**：
  - Bun: 使用 `Bun.hash()`（wyhash，约 100x SHA-256 速度）
  - Node.js: 回退到 SHA-256
- **场景**：文件内容缓存、配置变化检测

### 3. 成对哈希
- **目的**：哈希两个字符串而不分配临时连接字符串
- **实现**：
  - Bun: 种子链式 wyhash
  - Node.js: 增量 SHA-256 更新
- **优势**：避免大字符串拼接的内存分配

## 具体技术实现

### djb2Hash(str: string): number
```typescript
export function djb2Hash(str: string): number {
  let hash = 0
  for (let i = 0; i < str.length; i++) {
    hash = ((hash << 5) - hash + str.charCodeAt(i)) | 0
  }
  return hash
}
```

**算法说明：**
- 经典 djb2 算法（Dan Bernstein）
- `hash * 33 + char` 的位运算版本
- `| 0` 确保 32 位有符号整数溢出行为

**特性：**
- 快速（简单循环和位运算）
- 确定性（相同输入始终产生相同输出）
- 分布均匀性一般（非加密用途足够）

### hashContent(content: string): string
```typescript
export function hashContent(content: string): string {
  if (typeof Bun !== 'undefined') {
    return Bun.hash(content).toString()
  }
  const crypto = require('crypto') as typeof import('crypto')
  return crypto.createHash('sha256').update(content).digest('hex')
}
```

**运行时适配：**
| 运行时 | 算法 | 输出格式 |
|--------|------|----------|
| Bun | wyhash | 十进制字符串 |
| Node.js | SHA-256 | 十六进制字符串 |

**设计决策：**
- 输出格式不一致是刻意为之（简化实现）
- 调用方不应依赖具体格式，仅用于相等性比较

### hashPair(a: string, b: string): string
```typescript
export function hashPair(a: string, b: string): string {
  if (typeof Bun !== 'undefined') {
    return Bun.hash(b, Bun.hash(a)).toString()
  }
  const crypto = require('crypto') as typeof import('crypto')
  return crypto
    .createHash('sha256')
    .update(a)
    .update('\0')
    .update(b)
    .digest('hex')
}
```

**种子链式（Bun）：**
- `Bun.hash(a)` 的结果作为 `Bun.hash(b, seed)` 的种子
- 自然区分 `("ts", "code")` vs `("tsc", "ode")`

**分隔符（Node.js）：**
- `\0` 分隔符确保边界清晰
- 无分隔符时 `"ab" + "c"` 和 `"a" + "bc"` 会产生相同哈希

## 关键代码路径与文件引用

### 本文件导出
| 导出 | 类型 | 用途 |
|------|------|------|
| `djb2Hash` | 函数 | djb2 字符串哈希 |
| `hashContent` | 函数 | 通用内容哈希 |
| `hashPair` | 函数 | 成对字符串哈希 |

### 调用方
1. **cachePaths.ts**: 缓存路径生成
2. **Fallback.tsx**: 代码高亮回退组件
3. **Markdown.tsx**: Markdown 内容处理
4. **sessionStoragePortable.ts**: 会话存储
5. **teamMemorySync/index.ts**: 团队内存同步
6. **promptCacheBreakDetection.ts**: 提示缓存破坏检测
7. **perfettoTracing.ts**: 性能追踪

### 依赖模块
无外部依赖（仅使用内置模块）。

## 依赖与外部交互

### 运行时检测
```typescript
if (typeof Bun !== 'undefined') {
  // Bun 运行时
} else {
  // Node.js 运行时
}
```

**检测原理：**
- Bun 定义全局 `Bun` 对象
- `typeof` 检查在编译时优化

### Node.js 动态导入
```typescript
const crypto = require('crypto') as typeof import('crypto')
```

**设计决策：**
- 使用 `require` 而非 `import` 实现条件加载
- 避免在 Bun 运行时加载 crypto 模块
- `as typeof import('crypto')` 提供类型安全

## 风险、边界与改进建议

### 已知风险

1. **哈希碰撞**
   - 风险：不同内容产生相同哈希（djb2 尤其明显）
   - 缓解：仅用于缓存/变化检测，不用于安全场景
   - 说明：wyhash 和 SHA-256 碰撞概率极低

2. **输出格式不一致**
   - 风险：调用方依赖具体格式（十六进制 vs 十进制）
   - 缓解：文档明确说明仅用于相等性比较

3. **Bun.hash API 变化**
   - 风险：Bun 的 wyhash 实现可能变化
   - 缓解：Bun 承诺哈希稳定性

4. **大内容性能**
   - 风险：超大字符串哈希可能阻塞事件循环
   - 现状：同步 API，大内容需调用方控制

### 边界情况

1. **空字符串**：
   - `djb2Hash('')` → `0`
   - `hashContent('')` → 有效哈希值

2. **Unicode**：
   - djb2 按 UTF-16 码元处理
   - 代理对（surrogate pairs）被当作两个字符

3. **长字符串**：
   - djb2 可能溢出（32 位限制）
   - 溢出是预期行为（` | 0` 强制）

### 改进建议

1. **异步哈希**
   - 建议：为大内容提供异步 API
   - 实现：使用 Worker 或 setImmediate 分片

2. **流式哈希**
   - 建议：支持增量内容哈希
   - 场景：大文件分块读取时

3. **哈希算法选择**
   - 建议：允许调用方指定算法
   - 场景：需要特定碰撞抵抗级别时

4. **xxHash 支持**
   - 建议：在 Node.js 中优先使用 xxHash（如可用）
   - 收益：接近 wyhash 的性能

5. **Base64 输出选项**
   - 建议：可选 Base64 编码输出
   - 收益：更短的哈希字符串（vs 十六进制）

### 测试要点

1. 跨运行时输出一致性（相同输入）
2. 空字符串处理
3. Unicode 字符处理
4. 大字符串性能
5. hashPair 的边界区分能力
6. 碰撞概率（统计测试）
