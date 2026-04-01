# 研究文档：src/utils/findExecutable.ts

## 场景与职责

`findExecutable.ts` 是一个极简的 PATH 查找包装器，用于替代 `spawn-rx` 中的 `findActualExecutable` 函数。引入它的直接动机是：**避免为了一个简单的 `which` 功能而拉入整个 `rxjs` 库（约 313 KB）**。

该模块在需要按名称查找系统可执行文件时被调用，典型场景包括：
- `ripgrep.ts` 查找系统 `rg`
- `env.ts` 查找 `npm` 以判断 WSL 中是否使用了 Windows 路径的 npm

## 功能点目的

| 导出项 | 目的 |
|--------|------|
| `findExecutable(exe, args)` | 在 PATH 中查找可执行文件，返回 `{ cmd, args }` 以兼容 `spawn-rx` API。 |

- `cmd`：若找到则为解析后的绝对路径，否则回退为原始名称。
- `args`：原样透传。

## 具体技术实现

```ts
import { whichSync } from './which.js'

export function findExecutable(
  exe: string,
  args: string[],
): { cmd: string; args: string[] } {
  const resolved = whichSync(exe)
  return { cmd: resolved ?? exe, args }
}
```

- 完全委托给 `src/utils/which.js` 的 `whichSync` 实现。
- 自身无额外缓存、无环境变量读取、无平台特殊处理。

## 关键代码路径与文件引用

### 调用方

| 文件 | 调用点 | 说明 |
|------|--------|------|
| `src/utils/ripgrep.ts:38` | `findExecutable('rg', [])` | 判断系统是否安装了 ripgrep。 |
| `src/utils/env.ts:97` | `findExecutable('npm', [])` | WSL 环境下检测 npm 来源路径。 |

### 被调用方

- `src/utils/which.js`：`whichSync`

## 依赖与外部交互

- 无网络依赖。
- 无配置项。
- 不直接调用操作系统 API，所有平台逻辑下沉到 `which.js`。

## 风险、边界与改进建议

### 风险

1. **PATH 劫持**：`whichSync` 按 PATH 顺序查找，若当前目录在 PATH 中且存在同名恶意可执行文件，可能产生安全问题。调用方（如 `ripgrep.ts`）已通过显式使用命令名 `'rg'` 而非解析出的 `systemPath` 来规避 `NoDefaultCurrentDirectoryInExePath` 未启用时的风险。
2. **无缓存**：每次调用都重新扫描 PATH。在 `ripgrep.ts` 中通过 `memoize` 包裹了上层配置函数，因此实际查找次数有限；但 `env.ts` 中的 WSL 检测也是 `memoize` 的，影响可控。

### 边界

- 只查找**可执行文件**，不验证文件权限是否允许当前用户执行。
- 在 Windows 上，`whichSync` 会尝试常见扩展名（`.exe`, `.cmd`, `.bat` 等）。
- 不处理 ` PATHEXT` 之外的特殊脚本类型（如 `.ps1`）。

### 改进建议

1. **安全加固**：可考虑增加对返回路径的验证（如拒绝当前工作目录下的相对路径），但当前调用方已在更高层处理。
2. **合并到 which.ts**：由于该模块仅有一行有效逻辑，若未来无更多 `spawn-rx` 兼容需求，可直接在调用方使用 `whichSync`，进一步减少模块数量。
3. **异步变体**：当前只有同步版本。若未来需要在异步路径中避免阻塞，可补充基于 `which()`（异步）的 `findExecutableAsync`。
