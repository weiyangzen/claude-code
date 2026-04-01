# 研究文档：src/utils/fileStateCache.ts

## 场景与职责

`fileStateCache.ts` 实现了 Claude Code REPL 核心中的**文件状态缓存（FileStateCache）**，用于保存模型“已经看到过”的文件内容。这是 `ToolUseContext.readFileState` 的类型定义和实现来源，直接影响：
- `FileReadTool` 是否重复读取同一文件
- `getChangedFiles` 如何检测磁盘变更
- 自动注入（CLAUDE.md）内容的偏移/截断管理

与 `fileReadCache.ts` 不同，`fileStateCache` 缓存的是**对话上下文中已读文件的逻辑状态**，而非单纯的磁盘内容副本。

## 功能点目的

| 导出项 | 目的 |
|--------|------|
| `FileStateCache` | 基于 `lru-cache` 的缓存类，key 自动 `path.normalize`。 |
| `FileState` | 缓存值类型，包含 `content`、`timestamp`、`offset`、`limit`、`isPartialView?`。 |
| `createFileStateCacheWithSizeLimit(maxEntries, maxSizeBytes?)` | 工厂函数，默认字节上限 25MB。 |
| `cacheToObject(cache)` | 将缓存导出为普通对象（用于 `compact.ts` 序列化）。 |
| `cacheKeys(cache)` | 获取所有 key 数组。 |
| `cloneFileStateCache(cache)` | 深拷贝缓存（用于 forked agent 隔离状态）。 |
| `mergeFileStateCaches(first, second)` | 合并两个缓存，以较新的 `timestamp` 为准。 |
| `READ_FILE_STATE_CACHE_SIZE` | 默认最大条目数 100。 |

## 具体技术实现

### 缓存值结构

```ts
export type FileState = {
  content: string
  timestamp: number
  offset: number | undefined
  limit: number | undefined
  isPartialView?: boolean  // 自动注入且内容被截断/修改过
}
```

- `isPartialView` 为 `true` 时，表示模型看到的不是磁盘原始内容（如 CLAUDE.md 被剥掉了 HTML 注释、frontmatter，或 MEMORY.md 被截断）。此时 `EditTool`/`WriteTool` 必须要求显式 `Read` 后才能操作。
- `content` 在这里保存的是**原始磁盘字节**，用于 `getChangedFiles` 做 diff，而不是模型看到的视图。

### 缓存类实现

```ts
export class FileStateCache {
  private cache: LRUCache<string, FileState>

  constructor(maxEntries: number, maxSizeBytes: number) {
    this.cache = new LRUCache<string, FileState>({
      max: maxEntries,
      maxSize: maxSizeBytes,
      sizeCalculation: value => Math.max(1, Buffer.byteLength(value.content)),
    })
  }
}
```

- 所有 key 经过 `path.normalize()` 处理，保证相对/绝对路径、冗余段、`/` vs `\` 的一致性。
- 使用 `lru-cache` 的 `sizeCalculation` 按 `Buffer.byteLength(content)` 计算真实字节占用，实现**基于内存大小的 LRU 驱逐**。

### 辅助函数

- `cloneFileStateCache`：新建同配置缓存，然后 `dump()` / `load()` 实现深拷贝。
- `mergeFileStateCaches`：遍历两个缓存的条目，以 `timestamp` 较新的覆盖较旧的。

## 关键代码路径与文件引用

### 调用方

| 文件 | 导入内容 | 说明 |
|------|----------|------|
| `src/utils/queryHelpers.ts:29` | `createFileStateCacheWithSizeLimit`, `FileStateCache` | 创建 ask/read 操作的小型缓存。 |
| `src/utils/forkedAgent.ts:29` | `cloneFileStateCache` | 子 agent 隔离父状态。 |
| `src/utils/queryContext.ts:21` | `FileStateCache` (type) | 类型引用。 |
| `src/utils/claudemd.ts:62` | `cacheKeys`, `FileStateCache` | CLAUDE.md 注入时操作缓存。 |
| `src/utils/attachments.ts:112` | `cacheKeys`, `FileStateCache` | 附件处理时读取缓存 key。 |
| `src/Tool.ts:59` | `FileStateCache` (type) | `ToolUseContext.readFileState` 的类型。 |
| `src/QueryEngine.ts` / `src/screens/REPL.tsx` | 通过 `getToolUseContext` 传递实例 | 主循环持有缓存实例。 |
| `src/services/compact/compact.ts` | `cacheToObject` | 压缩会话时序列化缓存。 |
| `src/tools/AgentTool/runAgent.ts` | `cloneFileStateCache` | Agent 运行时克隆缓存。 |

### 被调用方

- `lru-cache`：第三方包，提供 LRU + size-based eviction。
- `path.normalize`：Node.js 内置。

## 依赖与外部交互

- 无网络依赖。
- 无持久化（由 `compact.ts` 在序列化时间接落盘）。
- 默认配置：
  - `READ_FILE_STATE_CACHE_SIZE = 100` 条
  - `DEFAULT_MAX_CACHE_SIZE_BYTES = 25 * 1024 * 1024`（25MB）

## 风险、边界与改进建议

### 风险

1. **内存估算偏差**：`sizeCalculation` 只计算 `content` 的字节长度，未计算 LRUCache 内部元数据、key 字符串、以及 `timestamp`/`offset`/`limit` 等字段的内存开销。实际占用略高于 `calculatedSize`。
2. **timestamp 冲突**：`mergeFileStateCaches` 完全依赖 `timestamp` 判定新旧。若两个缓存来自同一毫秒的操作，后遍历的会覆盖先遍历的，可能产生非确定性结果。
3. **isPartialView 语义泄漏**：`isPartialView` 是一个关键安全闸门，但它在缓存层只是一个布尔标记，实际强制逻辑分散在 `FileEditTool`/`FileWriteTool` 中，存在遗漏风险。

### 边界

- 图片文件不会被放入 `readFileState`（由 `FileReadTool` 显式跳过），因此 25MB 上限主要面向大文本、notebook、JSON 等可编辑内容。
- `path.normalize` 不会解析符号链接，因此 `a/b` 和 `a/c/../b` 会被视为同一 key，但 `a/b` 和 `realpath(a/b)` 若结果不同则仍是两个 key。

### 改进建议

1. **统一 isPartialView 检查**：将“partial view 禁止编辑”的校验逻辑集中到 `fileStateCache.ts` 或一个单独的 guard 函数中，减少在多个 tool 文件中的重复实现。
2. **更精确的 merge 策略**：在 `timestamp` 相等时，增加确定性决胜规则（如优先保留 `isPartialView === false` 的条目，或按缓存来源优先级）。
3. **监控与告警**：暴露 `calculatedSize` 和 `size` 的指标到 analytics，帮助发现异常大文件导致的缓存驱逐。
4. **测试覆盖**：补充对 `mergeFileStateCaches` 在边界条件（空缓存、相同 timestamp、超大 content）下的单元测试。
