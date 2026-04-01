# crypto.ts 深度研究文档

## 场景与职责

`crypto.ts` 是 Claude Code 的 **加密工具抽象层**，作为 `package.json` 中 `"browser"` 字段的间接点。它提供了跨平台（Node.js 和浏览器）的加密功能统一接口，目前主要导出 `randomUUID` 函数。

### 核心职责

1. **跨平台兼容**：在 Node.js/Bun 和浏览器环境中提供统一的加密 API
2. **包体积优化**：避免在浏览器构建中引入庞大的 crypto-browserify polyfill
3. **构建系统适配**：通过 `"browser"` 字段实现构建时的文件替换

### 使用场景

- **会话 ID 生成**：`bootstrap/state.ts` 使用 `randomUUID` 生成新会话 ID
- **任务 ID 生成**：`cronTasks.ts` 使用 `randomUUID` 生成定时任务 ID
- **各种 UUID 需求**：整个代码库中需要唯一标识符的场景

---

## 功能点目的

### 1. UUID 生成 (`randomUUID`)

**目的**：生成符合 RFC 4122 标准的版本 4 UUID。

**实现细节**：
- Node.js/Bun 环境：直接导出 `node:crypto` 的 `randomUUID`
- 浏览器环境：通过 `crypto.browser.ts` 使用 Web Crypto API

### 2. 构建时替换

**目的**：在浏览器构建中避免引入 ~500KB 的 crypto-browserify polyfill。

**实现机制**：
```json
// package.json
{
  "browser": {
    "src/utils/crypto.ts": "src/utils/crypto.browser.ts"
  }
}
```

当 Bun 构建 `--target browser` 时，自动将 `crypto.ts` 替换为 `crypto.browser.ts`。

---

## 具体技术实现

### 文件内容

```typescript
import { randomUUID } from 'crypto'
export { randomUUID }
```

### 浏览器替代实现

```typescript
// crypto.browser.ts
export function randomUUID(): string {
  return crypto.randomUUID()
}
```

### 技术细节

**注意**：文件使用显式的 `import-then-export` 模式，而非 re-export 语法：

```typescript
// 正确（本文件使用）
import { randomUUID } from 'crypto'
export { randomUUID }

// 错误（会导致 Bun bytecode 编译问题）
export { randomUUID } from 'crypto'
```

**原因**：Bun 的内部 bytecode 编译在处理 re-export 语法时存在 bug，会导致 `ReferenceError: randomUUID is not defined`。显式导入再导出产生正确的实时绑定。

---

## 关键代码路径与文件引用

### 核心导出

| 导出 | 行号 | 用途 |
|------|------|------|
| `randomUUID` | 12-13 | UUID 生成函数 |

### 依赖文件

```
crypto.ts
├── 被调用方（上游）
│   ├── src/bootstrap/state.ts               # 会话 ID 生成
│   ├── src/services/oauth/index.ts          # OAuth 相关
│   └── 其他需要 UUID 的模块
├── 被依赖模块（下游）
│   └── node:crypto                          # Node.js 加密模块
└── 浏览器替代
    └── src/utils/crypto.browser.ts          # Web Crypto API 实现
```

---

## 依赖与外部交互

### 运行时依赖

| 模块 | 用途 |
|------|------|
| `crypto` (Node.js) | `randomUUID` 函数 |

### 内部模块依赖

无内部模块依赖。

### 构建系统交互

| 构建目标 | 使用的文件 | 实现 |
|----------|-----------|------|
| Node.js/Bun | `crypto.ts` | `node:crypto` |
| Browser | `crypto.browser.ts` | `crypto.randomUUID()` |

---

## 风险、边界与改进建议

### 已知风险

1. **Bun Bytecode 编译**
   - 已知的 Bun 内部 bug 要求使用显式导入导出模式
   - 如果未来 Bun 修复此问题，可以考虑简化代码

2. **浏览器兼容性**
   - `crypto.randomUUID()` 需要较新的浏览器版本
   - 旧浏览器可能需要 polyfill

3. **功能单一**
   - 当前仅导出 `randomUUID`，其他加密功能直接依赖 `node:crypto`
   - 如果需要更多跨平台加密功能，需要扩展此模块

### 边界情况

| 场景 | 处理 |
|------|------|
| 浏览器不支持 crypto.randomUUID | 需要应用层提供 polyfill |
| Bun bytecode 编译 | 使用显式导入导出模式规避 |

### 改进建议

1. **功能扩展**
   - 考虑添加更多常用的加密工具函数（如哈希、加密等）
   - 统一跨平台的加密 API

2. **文档完善**
   - 添加关于 `"browser"` 字段工作原理的注释
   - 记录 Bun bytecode 问题的背景

3. **测试覆盖**
   - 添加浏览器环境的集成测试
   - 验证 UUID 格式正确性

4. **未来优化**
   - 关注 Bun 的 bytecode 编译问题修复
   - 考虑使用标准的 Web Crypto API 统一实现

### 相关 Issue/PR 参考

- PR `#20957` 和 `#21178`：涉及 Bun bytecode 编译问题
- `integration-tests-ant-native`：相关测试失败案例
