# 研究文档：src/utils/fileReadCache.ts

## 场景与职责

`fileReadCache.ts` 提供了一个基于 **mtime（修改时间）自动失效** 的内存文件内容缓存，用于消除 `FileEditTool` 等高频文件操作中的冗余磁盘读取。它是 `src/utils/file.ts` 中 `fileReadCache` 单例的来源，直接服务于工具层而非 UI 层。

## 功能点目的

| 导出项 | 目的 |
|--------|------|
| `fileReadCache` (singleton) | 全局可用的文件读取缓存实例。 |
| `FileReadCache.readFile(filePath)` | 返回 `{ content, encoding }`，自动根据 `mtimeMs` 判断缓存是否有效。 |
| `FileReadCache.clear()` | 清空整个缓存，用于测试或内存管理。 |
| `FileReadCache.invalidate(filePath)` | 移除指定路径的缓存项。 |
| `FileReadCache.getStats()` | 返回缓存大小和 key 列表，用于调试。 |

## 具体技术实现

### 缓存结构

```ts
type CachedFileData = {
  content: string
  encoding: BufferEncoding
  mtime: number
}

class FileReadCache {
  private cache = new Map<string, CachedFileData>()
  private readonly maxCacheSize = 1000
}
```

### 读取流程

1. `fs.statSync(filePath)` 获取 `mtimeMs`。
2. 若文件不存在，从缓存中 `delete` 并抛出原始错误。
3. 查找 `cache.get(filePath)`：
   - 命中且 `cachedData.mtime === stats.mtimeMs` → 直接返回缓存内容。
   - 未命中或 mtime 不一致 → 重新读取文件。
4. 读取时：
   - 调用 `detectFileEncoding(filePath)`（来自 `file.ts`）获取编码。
   - `fs.readFileSync(filePath, { encoding })`。
   - 将 `\r\n` 规范化为 `\n`。
5. 写入缓存；若超出 `maxCacheSize`，删除 `Map.keys().next().value`（FIFO 驱逐）。

## 关键代码路径与文件引用

### 调用方

| 文件 | 导入内容 | 说明 |
|------|----------|------|
| `src/utils/file.ts:25` | `fileReadCache` | `file.ts` 在 `readFileWithCache` 等函数中使用该单例。 |

### 被调用方

- `src/utils/file.js`：`detectFileEncoding`
- `src/utils/fsOperations.js`：`getFsImplementation`

## 依赖与外部交互

- 无网络依赖。
- 无配置项，上限硬编码为 1000 条。
- 依赖 `file.ts` 的 `detectFileEncoding`，因此虽然自身是缓存模块，但间接拉入了 `file.ts` 的少量依赖。

## 风险、边界与改进建议

### 风险

1. **FIFO 非 LRU**：超限时使用 `Map.keys().next().value` 驱逐最早插入项，而非最近最少使用项。在高并发交替读取两个大文件且总 key 超过 1000 时，可能产生抖动。
2. **内存无上限**：`maxCacheSize` 是条目数限制，不是字节数限制。若缓存 1000 个 10MB 文件，内存占用可达 10GB。
3. **mtime 精度问题**：某些文件系统（如 FAT32、网络共享）的 mtime 精度为 2 秒，快速连续写入可能导致缓存未失效而返回旧内容。
4. **路径 key 未规范化**：直接使用传入的 `filePath` 作为 key，若调用方交替使用相对路径和绝对路径，会产生重复缓存。

### 边界

- 仅缓存文本文件内容（因为 `detectFileEncoding` 和 `readFileSync` 都按文本处理）。
- 不缓存二进制文件或图片。
- `clear` 和 `getStats` 主要用于测试和调试，生产主循环不常调用。

### 改进建议

1. **LRU 替换**：将 `Map` 替换为 `lru-cache`（项目中已有依赖，见 `fileStateCache.ts`），实现真正的 LRU 驱逐。
2. **字节上限**：增加 `maxCacheSizeBytes` 限制，按 `Buffer.byteLength(content)` 计算条目大小，防止大文件撑爆内存。
3. **路径规范化**：使用 `path.normalize` 或 `path.resolve` 统一 key，避免重复缓存。
4. **异步接口**：提供 `readFileAsync` 变体，在异步工具链中避免同步 `statSync`/`readFileSync` 阻塞事件循环。
