# cwd.ts 深度研究文档

## 场景与职责

`cwd.ts` 是 Claude Code 的 **工作目录管理模块**，提供了对当前工作目录（CWD）的抽象和覆盖机制。它支持在异步上下文中临时覆盖工作目录，使并发代理能够各自看到不同的工作目录而不相互影响。

### 核心职责

1. **CWD 获取**：提供获取当前工作目录的统一接口
2. **CWD 覆盖**：支持在异步上下文中临时覆盖工作目录
3. **并发支持**：通过 `AsyncLocalStorage` 实现上下文隔离
4. **容错处理**：在获取失败时回退到原始工作目录

### 使用场景

- **多代理并发**：不同代理同时操作不同目录时，各自看到正确的 CWD
- **工具执行**：工具在执行时需要知道正确的工作目录
- **路径解析**：相对路径需要基于正确的 CWD 解析
- **会话恢复**：恢复会话后切换到正确的项目目录

---

## 功能点目的

### 1. CWD 覆盖 (`runWithCwdOverride`)

**目的**：在指定的异步上下文中临时改变 `pwd()` 和 `getCwd()` 的返回值。

**实现机制**：
- 使用 Node.js `AsyncLocalStorage` 存储覆盖的 CWD
- 在 `fn` 及其所有异步后代中，`pwd()` 返回覆盖值
- 不影响其他异步上下文

**使用示例**：
```typescript
await runWithCwdOverride('/other/project', async () => {
  // 在此函数及其异步后代中：
  console.log(pwd()) // '/other/project'
  await someAsyncOperation()
  console.log(pwd()) // 仍然是 '/other/project'
})
// 外部：
console.log(pwd()) // 原始 CWD
```

### 2. CWD 获取 (`pwd` / `getCwd`)

**目的**：获取当前工作目录，考虑可能的覆盖。

**区别**：
- `pwd()`：直接返回当前上下文的工作目录（可能被覆盖）
- `getCwd()`：在 `pwd()` 失败时回退到 `getOriginalCwd()`

**实现逻辑**：
```typescript
function pwd(): string {
  return cwdOverrideStorage.getStore() ?? getCwdState()
}

function getCwd(): string {
  try {
    return pwd()
  } catch {
    return getOriginalCwd()
  }
}
```

---

## 具体技术实现

### 核心数据结构

```typescript
const cwdOverrideStorage = new AsyncLocalStorage<string>()
```

### 函数实现

```typescript
/**
 * 在指定的异步上下文中运行函数，使用覆盖的工作目录
 */
export function runWithCwdOverride<T>(cwd: string, fn: () => T): T {
  return cwdOverrideStorage.run(cwd, fn)
}

/**
 * 获取当前工作目录（可能被覆盖）
 */
export function pwd(): string {
  return cwdOverrideStorage.getStore() ?? getCwdState()
}

/**
 * 获取当前工作目录，失败时回退到原始目录
 */
export function getCwd(): string {
  try {
    return pwd()
  } catch {
    return getOriginalCwd()
  }
}
```

### AsyncLocalStorage 工作原理

```
runWithCwdOverride('/tmp/project', fn)
    ↓
AsyncLocalStorage.run('/tmp/project', fn)
    ↓
[在 fn 内部]
    ↓
cwdOverrideStorage.getStore() → '/tmp/project'
    ↓
[在 fn 的异步后代中]
    ↓
cwdOverrideStorage.getStore() → '/tmp/project'（上下文继承）
    ↓
[fn 返回后]
    ↓
cwdOverrideStorage.getStore() → undefined（上下文恢复）
```

---

## 关键代码路径与文件引用

### 核心导出函数

| 函数 | 行号 | 用途 |
|------|------|------|
| `runWithCwdOverride` | 12-14 | 在覆盖的 CWD 下运行函数 |
| `pwd` | 19-21 | 获取当前工作目录 |
| `getCwd` | 26-32 | 获取当前工作目录（带容错） |

### 依赖文件

```
cwd.ts
├── 被调用方（上游）
│   ├── src/cli/handlers/agents.ts           # 代理处理
│   ├── src/cli/print.ts                     # CLI 输出
│   ├── src/services/vcr.ts                  # VCR 服务
│   ├── src/constants/prompts.ts             # 提示词
│   ├── src/services/api/filesApi.ts         # 文件 API
│   ├── src/constants/outputStyles.ts        # 输出样式
│   ├── src/main.tsx                         # 主入口
│   ├── src/tools/BashTool/bashPermissions.ts # Bash 权限
│   ├── src/services/lsp/LSPServerInstance.ts # LSP 服务
│   ├── src/services/mcp/utils.ts            # MCP 工具
│   ├── src/services/mcp/config.ts           # MCP 配置
│   ├── src/tools/BashTool/utils.ts          # Bash 工具
│   ├── src/tools/BriefTool/attachments.ts   # Brief 附件
│   ├── src/utils/filePersistence/filePersistence.ts # 文件持久化
│   └── ...（大量其他模块）
├── 被依赖模块（下游）
│   └── src/bootstrap/state.js               # getCwdState, getOriginalCwd
└── Node.js 内置
    └── async_hooks                          # AsyncLocalStorage
```

---

## 依赖与外部交互

### 运行时依赖

| 模块 | 用途 |
|------|------|
| `async_hooks` | `AsyncLocalStorage` 实现上下文隔离 |

### 内部模块依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `src/bootstrap/state.js` | `getCwdState`, `getOriginalCwd` | 获取全局 CWD 状态 |

### 与 bootstrap/state.js 的关系

```
cwd.ts                    bootstrap/state.js
   │                            │
   ├── getCwdState() ──────────→│ 获取当前 CWD（可能被 EnterWorktreeTool 修改）
   │                            │
   └── getOriginalCwd() ───────→│ 获取原始 CWD（启动时设置，永不改变）
```

---

## 风险、边界与改进建议

### 已知风险

1. **AsyncLocalStorage 限制**
   - 只在异步上下文中有效
   - 同步代码块中无法保持覆盖
   - 某些边界情况（如 `Promise.then` 链）可能丢失上下文

2. **覆盖嵌套**
   - 嵌套调用 `runWithCwdOverride` 时，内层覆盖外层
   - 返回外层后恢复，但如果内层抛出异常可能状态不一致

3. **全局状态依赖**
   - `getCwdState()` 依赖全局状态，可能被其他代码修改
   - 与 `EnterWorktreeTool` 等工具存在隐式耦合

### 边界情况

| 场景 | 处理 |
|------|------|
| 无覆盖时调用 `pwd()` | 返回 `getCwdState()` 的值 |
| `getCwdState()` 抛出异常 | `getCwd()` 回退到 `getOriginalCwd()` |
| 嵌套覆盖 | 内层覆盖外层，返回后恢复 |
| 异步边界 | AsyncLocalStorage 自动处理上下文传递 |

### 改进建议

1. **类型安全**
   - 考虑使用 branded type 区分原始路径和被覆盖的路径
   - 添加编译时检查防止路径混淆

2. **调试支持**
   - 添加 `getCwdStack()` 函数查看覆盖堆栈
   - 在调试日志中记录 CWD 变化

3. **文档完善**
   - 添加更多使用示例
   - 记录与 `EnterWorktreeTool` 的交互

4. **测试覆盖**
   - 添加嵌套覆盖测试
   - 添加异步边界测试
   - 添加异常恢复测试

5. **替代方案考虑**
   - 评估是否需要显式传递 CWD 而非全局覆盖
   - 考虑使用 DI（依赖注入）模式管理 CWD

### 相关 Issue/PR 参考

- 本模块支持多代理并发和 EnterWorktreeTool 功能
- 与 `EnterWorktreeTool` 紧密相关，该工具修改 `getCwdState()` 的值
