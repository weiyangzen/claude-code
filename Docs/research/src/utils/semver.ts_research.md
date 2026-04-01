# 研究文档：src/utils/semver.ts

## 场景与职责

`semver.ts` 是 Claude Code 的**语义化版本比较封装层**。项目需要同时支持两种运行环境：

1. **Bun 环境**：Bun 内置了 `Bun.semver`，其 `order()` 方法比 npm `semver` 包快约 20 倍。
2. **Node.js 环境**： fallback 到传统的 npm `semver` 包。

该模块对所有 semver 比较操作提供统一 API，并在运行时自动选择最优实现。所有 npm `semver` 的 fallback 调用都使用 `{ loose: true }` 模式，以兼容更多非严格 semver 字符串。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `gt(a, b)` | 大于比较（greater than）。 |
| `gte(a, b)` | 大于等于比较。 |
| `lt(a, b)` | 小于比较。 |
| `lte(a, b)` | 小于等于比较。 |
| `satisfies(version, range)` | 检查版本是否满足范围表达式（如 `^1.0.0`）。 |
| `order(a, b)` | 返回 `-1` / `0` / `1`，用于排序比较。 |

---

## 具体技术实现

### 1. 动态加载 npm semver

```ts
let _npmSemver: typeof import('semver') | undefined

function getNpmSemver(): typeof import('semver') {
  if (!_npmSemver) {
    // eslint-disable-next-line @typescript-eslint/no-require-imports
    _npmSemver = require('semver') as typeof import('semver')
  }
  return _npmSemver
}
```

- 使用 `require` 而非 `import`，因为该模块可能是同步调用链的一部分（如启动时的版本比较）。
- 单例缓存 `_npmSemver`，避免重复加载模块。

### 2. Bun 分支

每个比较函数都先检查 `typeof Bun !== 'undefined'`：

```ts
export function gt(a: string, b: string): boolean {
  if (typeof Bun !== 'undefined') {
    return Bun.semver.order(a, b) === 1
  }
  return getNpmSemver().gt(a, b, { loose: true })
}
```

- `Bun.semver.order(a, b)` 返回 `-1 | 0 | 1`，因此：
  - `gt` → `=== 1`
  - `gte` → `>= 0`
  - `lt` → `=== -1`
  - `lte` → `<= 0`
- `Bun.semver.satisfies(version, range)` 与 npm semver 签名一致，直接透传。

### 3. Node.js fallback

- `gt` / `gte` / `lt` / `lte` / `order`：使用 npm `semver` 的对应方法，附加 `{ loose: true }`。
- `satisfies`：使用 `semver.satisfies(version, range, { loose: true })`。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/semver.ts:11-17` | `getNpmSemver` 懒加载与缓存。 |
| `src/utils/semver.ts:19-59` | 比较函数 `gt`, `gte`, `lt`, `lte`, `order`。 |
| `src/utils/semver.ts:47-52` | `satisfies` 实现。 |
| `src/utils/releaseNotes.ts:216-229` | 调用方：release notes 版本比较。 |
| `src/utils/autoUpdater.ts` | 调用方：自动更新版本检查。 |
| `src/utils/desktopDeepLink.ts` | 调用方：桌面端深度链接版本校验。 |
| `src/utils/logoV2Utils.ts` | 调用方：Logo v2 功能版本 gate。 |
| `src/utils/ide.ts` | 调用方：IDE 集成版本检查。 |

---

## 依赖与外部交互

- **第三方库**：`semver`（npm 包，Node.js fallback 时使用）。
- **运行时全局对象**：`Bun`（Bun 环境时存在）。
- **调用方**：
  - `src/utils/releaseNotes.ts`
  - `src/utils/autoUpdater.ts`
  - `src/utils/desktopDeepLink.ts`
  - `src/utils/logoV2Utils.ts`
  - `src/utils/ide.ts`

---

## 风险、边界与改进建议

### 风险与边界

1. **`typeof Bun` 检测的可靠性**：在 Bun 的 compiled binary（standalone executable）中，`Bun` 全局对象通常存在，但某些边缘构建配置或 polyfill 环境可能导致假阴性。不过当前项目明确以 Bun 为主 runtime，此检测是社区广泛接受的模式。

2. **`require('semver')` 的 bundle 兼容性**：若项目使用 esbuild、rollup 等打包工具将代码打包为单文件，`require('semver')` 需要打包工具支持 `require` 语法或正确 externalize `semver` 包。当前项目使用 Bun 构建，对 `require` 支持良好。

3. **`loose: true` 的语义差异**：loose 模式会容忍一些非标准 semver 字符串（如 `1.2` 视为 `1.2.0`）。这在用户体验上是加分项，但在需要严格版本校验的场景（如安全更新检查）可能过于宽松。当前所有调用方都接受 loose 模式，没有提供 strict 选项。

4. **无 `eq` 函数**：模块提供了 `gt/gte/lt/lte/order/satisfies`，但没有提供 `eq(a, b)`。调用方若需要相等判断，通常使用 `order(a, b) === 0`，这略显冗余。

5. **Bun.semver 与 npm semver 的边界行为差异**：虽然 Bun 声称兼容 npm semver，但在极端输入（如空字符串、非法字符）时，两者的错误类型或返回值可能不同。当前模块没有统一错误处理，直接透传底层行为。

### 改进建议

1. **增加 `eq` 和 `neq` 辅助函数**：补充常见的相等/不等比较，减少调用方使用 `order() === 0` 的样板代码：
   ```ts
   export function eq(a: string, b: string): boolean {
     return order(a, b) === 0
   }
   ```

2. **增加输入校验**：在传入 `Bun.semver` 或 `npm semver` 之前，对 `a`、`b` 做基础校验（非空字符串），并抛出语义清晰的错误。这有助于在版本字符串意外为 `undefined` 或 `""` 时快速定位问题。

3. **提供 strict 模式选项**：为需要严格 semver 校验的场景增加可选参数：
   ```ts
   export function gt(a: string, b: string, loose = true): boolean
   ```
   保持默认 `loose = true` 以兼容现有代码。

4. **增加基准测试与兼容性测试**：在 CI 中同时跑 Bun 和 Node.js 环境，验证同一组版本字符串在两种后端下的比较结果完全一致。

5. **考虑使用 `import` 替代 `require`**：若项目全面转向 ESM 或 top-level await 被支持，可将 `getNpmSemver` 改为动态 `import()`，更符合现代 TS 工程规范。不过当前同步 API 的设计使得 `require` 是更自然的选择。
