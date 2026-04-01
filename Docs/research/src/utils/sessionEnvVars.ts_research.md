# 研究文档：src/utils/sessionEnvVars.ts

## 场景与职责

`sessionEnvVars.ts` 是 Claude Code 中用于管理**会话级环境变量**的最小化状态模块。用户可以通过 `/env` 命令在会话期间设置环境变量，这些变量仅应用于后续通过 Bash/PowerShell 工具派生的子进程，而**不会**影响 REPL 进程本身。

模块的设计哲学是**极简封装**：用一个模块级 `Map` 存储键值对，暴露 CRUD 接口。所有复杂逻辑（如持久化、与 shell provider 的集成、与 hook 环境脚本的合并）都在调用方处理。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| `getSessionEnvVars()` | 返回当前会话环境变量的只读视图（`ReadonlyMap<string, string>`）。 |
| `setSessionEnvVar(name, value)` | 设置（或覆盖）一个会话环境变量。 |
| `deleteSessionEnvVar(name)` | 删除指定环境变量。 |
| `clearSessionEnvVars()` | 清空所有会话环境变量。 |

---

## 具体技术实现

### 1. 实现代码

```ts
const sessionEnvVars = new Map<string, string>()

export function getSessionEnvVars(): ReadonlyMap<string, string> {
  return sessionEnvVars
}

export function setSessionEnvVar(name: string, value: string): void {
  sessionEnvVars.set(name, value)
}

export function deleteSessionEnvVar(name: string): void {
  sessionEnvVars.delete(name)
}

export function clearSessionEnvVars(): void {
  sessionEnvVars.clear()
}
```

### 2. 设计特点

- **模块级单例**：`sessionEnvVars` 在进程生命周期内持久存在，随会话切换不会自动清空（若需要清空，调用方需显式调用 `clearSessionEnvVars`）。
- **只读暴露**：`getSessionEnvVars()` 返回 `ReadonlyMap`，防止外部直接修改内部状态。但注意 `ReadonlyMap` 只是 TypeScript 类型约束，运行时仍可通过 `Map.prototype.set.call()` 绕过，属于常规的信任边界设计。
- **无序列化/无验证**：模块本身不检查变量名合法性（如是否包含 `=`、`\n` 等），也不做大小写规范化（Windows 环境变量通常大小写不敏感，但此处完全交给调用方处理）。

---

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/sessionEnvVars.ts:6-22` | 模块完整实现。 |
| `src/utils/shell/bashProvider.ts:249-251` | 调用方：Bash provider 在 `getEnvironmentOverrides` 中遍历 `getSessionEnvVars()` 并注入到子进程环境。 |
| `src/utils/shell/powershellProvider.ts` | 调用方：PowerShell provider 同样遍历并注入。 |
| `src/commands/env/index.js` | 调用方：`/env` 命令的实现，用户通过 CLI 交互设置/查看/删除会话环境变量。 |
| `src/commands/clear/caches.ts` | 调用方：清除缓存时可能调用 `clearSessionEnvVars()`。 |

---

## 依赖与外部交互

- **无第三方依赖**。
- **无 Node.js 内置模块依赖**。
- **调用方**：
  - `src/utils/shell/bashProvider.ts`
  - `src/utils/shell/powershellProvider.ts`
  - `src/commands/env/index.js`
  - `src/commands/clear/caches.ts`

---

## 风险、边界与改进建议

### 风险与边界

1. **会话切换不清空**：`sessionEnvVars` 是进程级变量，若一个进程内先后服务多个会话（如守护进程模式），旧会话的环境变量会泄漏到新会话。当前 Claude Code 的架构是一个进程对应一个会话，所以这不是问题，但未来若支持多会话并发，需要重构。

2. **无持久化**：会话环境变量仅在内存中，进程重启后丢失。这是设计上的选择（“会话级”即当前进程生命周期），但用户可能误以为 `/env` 设置的变量会持久化到 `~/.bashrc` 或项目配置中。

3. **与 `process.env` 完全隔离**：`setSessionEnvVar` 不会修改 `process.env`，这意味着：
   - 子进程能拿到这些变量（通过 shell provider 注入）。
   - REPL 进程本身及其直接调用的 Node.js 代码看不到这些变量。
   这是正确的设计，但某些调用方若误用 `process.env.FOO` 而非 `getSessionEnvVars().get('FOO')`，会产生困惑。

4. **无并发保护**：虽然 `Map` 本身是线程安全的（Node.js 单线程），但若多个异步操作同时读写，逻辑上仍可能产生竞态。当前调用方（如 `/env` 命令和 `clear` 命令）都是用户交互触发，并发概率极低。

5. **变量名未做大小写折叠**：在 Windows 上，`PATH` 和 `path` 是同一个环境变量。当前模块不做任何规范化，若用户在 Windows 上通过 `/env` 设置了 `path=xxx`，而 Bash provider 又注入了 `PATH=yyy`，子进程实际行为取决于 shell 的合并策略。

### 改进建议

1. **增加变量名校验**：在 `setSessionEnvVar` 中检查变量名是否包含非法字符（如 `=`, `\0`, `\n`），并抛出清晰的错误。参考 POSIX 环境变量命名规范（`[A-Za-z_][A-Za-z0-9_]*`）。

2. **增加 Windows 大小写规范化（可选）**：若检测到 `process.platform === 'win32'`，在设置和读取时统一将 key 转为大写（或提供大小写不敏感的 `get` 方法），避免 Windows 上的重复/覆盖问题。

3. **暴露 `hasSessionEnvVar(name)`**：当前调用方需要写 `getSessionEnvVars().has(name)`，增加一个便捷方法可提升 API 一致性：
   ```ts
   export function hasSessionEnvVar(name: string): boolean {
     return sessionEnvVars.has(name)
   }
   ```

4. **文档化生命周期**：在 `/env` 命令的帮助文本和 `AGENTS.md` 中明确说明“会话环境变量仅对当前进程有效，不会写入磁盘，也不会影响 REPL 进程本身”。

5. **考虑与 `sessionEnvironment.ts` 的命名区分**：`sessionEnvVars.ts` 和 `sessionEnvironment.ts` 名称非常相似，容易混淆。可考虑将本模块重命名为 `sessionEnvVarStore.ts` 或 `sessionEnvOverrides.ts`，提高代码可读性。
