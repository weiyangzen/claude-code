# src/utils/lockfile.ts 研究文档

## 场景与职责

`lockfile.ts` 是 `proper-lockfile` 包的懒加载包装器。`proper-lockfile` 依赖 `graceful-fs`，后者在首次 require 时会 monkey-patch 所有 `fs` 方法，带来约 8ms 的启动开销。对于只需要快速响应的命令（如 `claude --help`），这种开销是不必要的。

该模块通过延迟加载 `proper-lockfile`，确保只有在真正需要文件锁功能时才承担这一成本。

调用方非常广泛，任何需要跨进程文件锁的模块都会使用它：
- `src/utils/config.ts`
- `src/utils/cleanup.ts`
- `src/utils/teammateMailbox.ts`
- `src/utils/tasks.ts`
- `src/utils/auth.ts`

## 功能点目的

### 懒加载包装
模块导出了 `proper-lockfile` 的四个核心 API：
- `lock(file, options?)`：异步获取文件锁，返回释放函数。
- `lockSync(file, options?)`：同步获取文件锁，返回释放函数。
- `unlock(file, options?)`：异步释放文件锁。
- `check(file, options?)`：异步检查文件是否被锁定。

底层实现：
```typescript
let _lockfile: Lockfile | undefined

function getLockfile(): Lockfile {
  if (!_lockfile) {
    _lockfile = require('proper-lockfile') as Lockfile
  }
  return _lockfile
}
```

## 具体技术实现

### 动态 require
使用 `require` 而非 `import`，因为：
1. `require` 是同步的，可以在首次调用时立即加载模块。
2. 模块顶部已经通过 `import type` 引入了类型定义，确保类型安全。
3. 在 ESM 环境下（Bun/Node.js ESM），`require` 仍然可用（通过 CJS 兼容层）。

### 类型定义
```typescript
import type { CheckOptions, LockOptions, UnlockOptions } from 'proper-lockfile'
type Lockfile = typeof import('proper-lockfile')
```

这种方式允许在不实际加载模块的情况下获得完整的 TypeScript 类型信息。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/lockfile.ts:18-24` | `getLockfile` 懒加载器 |
| `src/utils/lockfile.ts:26-43` | 导出的 `lock`/`lockSync`/`unlock`/`check` 包装函数 |

## 依赖与外部交互

### 外部依赖
- `proper-lockfile`（动态加载）

### 调用方
- `src/utils/config.ts`
- `src/utils/cleanup.ts`
- `src/utils/teammateMailbox.ts`
- `src/utils/tasks.ts`
- `src/utils/auth.ts`

## 风险、边界与改进建议

### 风险与边界
1. **`require` 在纯 ESM 环境中的兼容性**：虽然当前 Bun 和 Node.js 的 ESM 实现都支持 `require`，但如果未来运行时被严格限制为纯 ESM（如某些边缘的打包工具或浏览器环境），`require` 会抛出错误。不过考虑到这是一个 Node.js CLI 工具，这种风险很低。
2. **`graceful-fs` 的副作用**：即使通过懒加载避免了启动时的 monkey-patch，一旦任何代码路径触发了 `getLockfile()`，`graceful-fs` 仍然会 patch 全局 `fs` 模块。这可能导致后续依赖原生 `fs` 行为的代码出现微妙的行为变化。
3. **无缓存清理接口**：模块没有提供重置 `_lockfile` 的方法。在测试环境中，如果需要模拟或替换 `proper-lockfile` 的行为，没有官方钩子可用。
4. **类型导入与实际模块版本不一致**：`import type from 'proper-lockfile'` 在编译时解析，但如果运行时加载的 `proper-lockfile` 版本与编译时类型版本不同，可能出现运行时类型不匹配（虽然 JS 本身不检查类型，但可能导致 API 签名变化引发错误）。

### 改进建议
1. **考虑替代库**：如果 `graceful-fs` 的 monkey-patch 副作用在未来造成问题，可以考虑迁移到不依赖 `graceful-fs` 的锁库（如 `lockfile` 或 `fs-ext`），或者自己实现一个基于 `mkdir` 的轻量级跨平台文件锁。
2. **提供测试重置接口**：导出 `_resetLockfileForTesting()` 函数，允许测试在需要时清除 `_lockfile` 缓存并注入 mock。
3. **版本锁定**：在 `package.json` 中严格锁定 `proper-lockfile` 的版本范围，防止 minor 版本更新引入 API 变化。
4. **ESM 动态导入备选**：虽然当前 `require` 工作正常，但可以为未来预留一个基于 `import()` 的回退路径：
   ```typescript
   if (!_lockfile) {
     _lockfile = typeof require !== 'undefined'
       ? require('proper-lockfile')
       : await import('proper-lockfile')
   }
   ```
   不过这会使得 `lock()` 等函数变成 async（如果走 `import()` 路径），需要权衡 API 兼容性。
