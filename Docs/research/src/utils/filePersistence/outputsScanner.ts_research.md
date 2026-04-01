# 研究文档：src/utils/filePersistence/outputsScanner.ts

## 场景与职责

`outputsScanner.ts` 是 Claude Code 项目中**文件持久化子系统的扫描器模块**。它的核心职责是：
1. **检测运行环境类型**（BYOC / Cloud）
2. **扫描本地 `outputs` 目录**，找出自当前 turn 开始以来被修改过的文件
3. 为上游的 `filePersistence.ts` 提供干净、安全的文件列表

该模块是文件持久化流程的"数据源"，本身不执行任何网络 I/O 或上传操作，只专注于本地文件系统的安全扫描和过滤。

---

## 功能点目的

### 1. `logDebug(message: string)`
模块级调试日志包装器。统一添加 `[file-persistence]` 前缀，通过 `src/utils/debug.js` 的 `logForDebugging()` 输出。便于在大量调试日志中快速定位文件持久化相关的问题。

### 2. `getEnvironmentKind(): EnvironmentKind | null`
从环境变量 `CLAUDE_CODE_ENVIRONMENT_KIND` 读取当前运行环境类型：
- `'byoc'`：BYOC（Bring Your Own Compute）模式
- `'anthropic_cloud'`：Anthropic 托管的 Cloud/1P 模式
- 其他值或未设置：返回 `null`

该函数被 `filePersistence.ts` 和 `plans.ts` 共同使用，用于判断当前是否处于需要特殊文件处理的远程会话环境。

### 3. `findModifiedFiles(turnStartTime, outputsDir): Promise<string[]>`
**核心扫描函数**，递归遍历 `outputsDir`，返回所有 `mtime >= turnStartTime` 的普通文件路径列表。

关键行为：
- 使用 `fs.readdir(..., { recursive: true, withFileTypes: true })` 一次性递归列出所有条目
- **跳过符号链接**（安全考量）
- 兼容 Node 20+ 的 `entry.parentPath` 和旧版本的 `entry.path` 回退
- 并行 `lstat` 所有文件获取 `mtimeMs`
- 在 `readdir` 和 `lstat` 之间再次检查符号链接，防护竞态条件
- 静默处理目录不存在或文件在扫描期间被删除的情况

---

## 具体技术实现

### 关键流程

```
filePersistence.ts
  └─> findModifiedFiles(turnStartTime, outputsDir)
        ├─> fs.readdir(outputsDir, { recursive: true, withFileTypes: true })
        ├─> 过滤：跳过 symlink，只保留 isFile()
        ├─> 构建完整路径（兼容 parentPath / path 回退）
        ├─> Promise.all(filePaths.map(async filePath => fs.lstat(filePath)))
        ├─> 二次过滤：跳过 lstat 期间变成 symlink 的文件
        ├─> 过滤：mtimeMs >= turnStartTime
        └─> 返回 modifiedFiles: string[]
```

### 数据结构

```typescript
// 来自 teleport/environments.js
export type EnvironmentKind = 'anthropic_cloud' | 'byoc' | 'bridge'

// 来自 ./types.js
type TurnStartTime = number  // 毫秒时间戳（Date.now()）
```

### Node 版本兼容性处理

`fs.readdir` 在 Node 20+ 返回的 `Dirent` 对象包含 `parentPath` 属性，而旧版本使用 `path`。模块中通过两个类型守卫函数处理兼容性：

```typescript
function hasParentPath(entry: object): entry is { parentPath: string; name: string } {
  return 'parentPath' in entry && typeof entry.parentPath === 'string'
}

function hasPath(entry: object): entry is { path: string; name: string } {
  return 'path' in entry && typeof entry.path === 'string'
}

function getEntryParentPath(entry: object, fallback: string): string {
  if (hasParentPath(entry)) return entry.parentPath
  if (hasPath(entry)) return entry.path
  return fallback
}
```

### 安全设计

1. **符号链接双重检查**
   - 第一次：在 `readdir` 循环中，`entry.isSymbolicLink()` 直接跳过
   - 第二次：在 `lstat` 结果中，`stat.isSymbolicLink()` 再次跳过
   这防止了 TOCTOU（Time-of-check to time-of-use）竞态攻击，即文件在扫描期间被替换为符号链接指向敏感文件。

2. **静默容错**
   - `readdir` 失败（目录不存在/无权限）→ 返回空数组
   - `lstat` 失败（文件在 `readdir` 后被删除）→ 返回 `null`，不影响其他文件

---

## 关键代码路径与文件引用

### 本文件内部

| 函数 | 行号 | 说明 |
|------|------|------|
| `logDebug` | 17 | 调试日志前缀包装器 |
| `getEnvironmentKind` | 25 | 环境类型检测 |
| `hasParentPath` | 33 | Node 20+ `parentPath` 类型守卫 |
| `hasPath` | 39 | 旧版 `path` 类型守卫 |
| `getEntryParentPath` | 43 | 兼容性路径提取 |
| `findModifiedFiles` | 62 | 核心扫描函数 |

### 上游调用方

| 文件 | 行号 | 调用方式 |
|------|------|----------|
| `src/utils/filePersistence/filePersistence.ts` | 158 | `const modifiedFiles = await findModifiedFiles(turnStartTime, outputsDir)` |
| `src/utils/plans.ts` | 189 | `if (getEnvironmentKind() === null) { return false }` |
| `src/utils/plans.ts` | 361 | `if (getEnvironmentKind() === null) { return }` |

在 `plans.ts` 中，`getEnvironmentKind()` 用于判断是否需要执行计划文件的恢复（resume）和快照（snapshot）逻辑。只有在远程会话（BYOC/Cloud）中，本地文件才不会持久化，因此才需要从消息历史或文件快照中恢复计划文件。

### 下游依赖方

| 文件 | 导入内容 | 作用 |
|------|----------|------|
| `fs/promises` | `readdir`, `lstat` | Node.js 原生异步文件系统操作 |
| `path` | `join` | 路径拼接 |
| `src/utils/debug.js` | `logForDebugging` | 调试日志基础设施 |
| `src/utils/teleport/environments.js` | `EnvironmentKind` | 环境类型定义 |
| `src/utils/filePersistence/types.js` | `TurnStartTime` | 时间戳类型别名 |

---

## 依赖与外部交互

### 环境变量依赖

| 环境变量 | 用途 | 消费方 |
|----------|------|--------|
| `CLAUDE_CODE_ENVIRONMENT_KIND` | 判断环境类型 | `getEnvironmentKind()` |

### 本地文件系统交互

- **读取目录**：`fs.readdir(outputsDir, { recursive: true, withFileTypes: true })`
  - `recursive: true` 会一次性展开所有子目录，返回扁平化的 `Dirent[]` 列表
  - 这是 Node.js v20.1.0 / v18.17.0 起支持的特性
- **获取文件状态**：`fs.lstat(filePath)`
  - 使用 `lstat` 而非 `stat`，确保不跟随符号链接
- **时间比较**：`stat.mtimeMs >= turnStartTime`
  - `mtimeMs` 是毫秒级时间戳，与 `Date.now()` 生成的 `turnStartTime` 精度一致

---

## 风险、边界与改进建议

### 已知风险

1. **`fs.readdir` 的 `recursive` 选项兼容性**
   `recursive: true` 在 Node.js 20.1.0 / 18.17.0 才稳定支持。如果项目在更旧的 Node 版本上运行，该选项会被静默忽略，导致只扫描顶层目录，子目录中的文件被遗漏。虽然项目使用 Bun 运行时，但如果在纯 Node 环境下测试或降级运行，可能存在兼容性风险。

2. **大量文件时的内存和 I/O 压力**
   `recursive: true` 会一次性将所有 `Dirent` 对象加载到内存中。如果 `outputs` 目录包含数万个小文件，`entries` 数组可能占用大量内存。随后的 `Promise.all(lstat(...))` 会同时发起大量系统调用，可能触发操作系统的文件描述符限制或导致 I/O 抖动。

3. **目录本身的修改时间不被考虑**
   `findModifiedFiles` 只检查普通文件的 `mtime`，不检查子目录的修改时间。如果某个 turn 中只创建了一个空子目录，该变更不会被检测到（因为没有文件）。这在大多数场景下是可接受的，但如果业务逻辑需要追踪目录结构变化，则存在遗漏。

4. **`turnStartTime` 精度边界**
   `Date.now()` 和 `fs.stat().mtimeMs` 的精度都是毫秒级。如果文件在 turn 开始后的同一毫秒内被修改，条件 `mtimeMs >= turnStartTime` 能正确包含。但如果文件在 `turnStartTime` 记录前的同一毫秒内被修改，也可能被误包含（因为 mtime 精度限制）。

5. **没有文件大小预检查**
   `outputsScanner.ts` 只返回文件路径，不返回文件大小。上游 `filePersistence.ts` 也无法在调用 `filesApi.ts` 之前知道文件大小，导致大文件的上传失败成本较高。

6. **隐藏文件和临时文件的处理**
   当前实现不会过滤隐藏文件（如 `.DS_Store`、`.gitignore`）或临时文件（如 `*.tmp`、编辑器交换文件）。这些文件如果位于 `outputs` 目录中，且 mtime 符合条件，会被无条件上传，浪费带宽和 API 配额。

### 改进建议

1. **增加文件大小和元信息返回**
   将 `findModifiedFiles` 的返回类型从 `string[]` 扩展为包含 `size` 和 `mtimeMs` 的对象数组，使上游能够：
   - 在上传前过滤超大文件
   - 更精确地记录 analytics（如总字节数）
   - 优化并发调度（按文件大小排序，避免大文件阻塞队列尾部）

2. **增加忽略规则（.gitignore 风格）**
   引入一个可配置的忽略列表（如 `.claude_persistence_ignore` 或硬编码的常见临时文件模式），过滤掉不需要上传的文件：
   - 隐藏文件（`.*`）
   - 临时文件（`*.tmp`, `*.swp`, `*~`）
   - 版本控制元数据（`.git/`, `.svn/`）

3. **分批/流式扫描替代一次性递归**
   对于可能包含大量文件的 `outputs` 目录，可以考虑：
   - 使用 `fs.opendir()` 进行流式遍历，减少内存峰值
   - 或者限制递归深度，避免意外扫描过深的目录树
   - 对 `lstat` 调用进行分批处理（如每批 1000 个），避免并发系统调用过多

4. **增加目录存在性缓存**
   `findModifiedFiles` 在每次 turn 结束时都会被调用。如果 `outputs` 目录不存在，每次都会触发一次失败的 `readdir` 调用。可以在模块级别缓存 "目录不存在" 的状态，减少不必要的 I/O。

5. **补充单元测试**
   当前代码库中没有找到针对 `outputsScanner.ts` 的单元测试。建议补充以下场景的测试：
   - 正常扫描：混合文件、子目录、符号链接
   - `turnStartTime` 边界：刚好等于 mtime、小于 mtime、大于 mtime
   - 竞态条件模拟：文件在 `readdir` 后变成符号链接或被删除
   - Node 版本兼容性：`parentPath` 和 `path` 回退逻辑
   - 空目录/不存在的目录：返回空数组
   - 大量文件场景：性能基准测试

6. **考虑使用 `fs.stat` 替代 `fs.lstat` 的风险再评估**
   当前使用 `lstat` 是正确的安全选择，因为它不跟随符号链接。但需要确保所有调用方都理解这一点：如果 `outputs` 目录中的合法文件本身是一个指向其他位置的硬链接（hard link），`lstat` 会正常返回其元数据，这是期望行为。只有符号链接被跳过。
