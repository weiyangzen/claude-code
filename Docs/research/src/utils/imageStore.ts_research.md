# src/utils/imageStore.ts 研究文档

## 场景与职责

`imageStore.ts` 负责将用户粘贴或拖拽的图像持久化到本地磁盘，并为当前会话维护一个内存中的图像路径缓存。这在以下场景中至关重要：

1. **图像附件持久化**：当用户在终端中粘贴图像时，图像数据以 base64 形式存在于 `PastedContent` 中。为了支持后续操作（如点击引用、重新加载会话、跨进程访问），需要将图像写入磁盘缓存。
2. **会话隔离**：图像按 `sessionId` 分目录存储，避免不同会话之间的图像混淆。
3. **缓存清理**：提供旧会话图像缓存的清理能力，防止磁盘无限增长。

该模块是 `src/utils/config.ts` 中 `PastedContent` 类型的消费方之一，主要被 `src/utils/cleanup.ts`（清理旧缓存）和 `src/components/ClickableImageRef.tsx`（获取已存储图像路径）调用。

## 功能点目的

### 1. `storeImage`
将单个 `PastedContent`（类型为 `image`）写入磁盘。文件路径基于 `content.id` 和 `mediaType` 的扩展名，存储在 `~/.claude/image-cache/{sessionId}/{id}.{ext}`。写入时使用 `O_EXCL` 等价的原子创建语义（通过 `open` + `fh.writeFile` + `fh.datasync` + `fh.close`），并以 `0o600` 权限创建，确保只有当前用户可读。

### 2. `storeImages`
批量存储 `pastedContents` 字典中的所有图像，返回 `Map<id, path>`。

### 3. `cacheImagePath`
纯内存操作，无需文件 I/O。当图像已经以其他方式存储（如外部 URL 或已由调用方保证存在）时，仅将路径预注册到内存缓存中，供后续快速查询。

### 4. `getStoredImagePath`
根据 `imageId` 查询内存缓存中的路径。若未命中，返回 `null`（不触发磁盘 I/O）。

### 5. `clearStoredImagePaths`
清空内存缓存。通常在会话切换或退出时调用。

### 6. `cleanupOldImageCaches`
清理非当前会话的旧图像缓存目录。遍历 `~/.claude/image-cache/` 下的所有子目录，删除非当前 `sessionId` 的目录，若清理后根目录为空则一并删除。

## 具体技术实现

### 存储路径结构
```
~/.claude/image-cache/
  └── {sessionId}/
        ├── 1.png
        ├── 2.jpg
        └── 3.webp
```

路径通过 `getClaudeConfigHomeDir()` 获取配置主目录，再拼接 `IMAGE_STORE_DIR`（固定为 `'image-cache'`）和 `getSessionId()`。

### 内存缓存机制
```typescript
const storedImagePaths = new Map<number, string>()
const MAX_STORED_IMAGE_PATHS = 200
```

缓存采用简单的 `Map`，利用其按插入顺序维护键的特性实现 LRU 淘汰：
```typescript
function evictOldestIfAtCap(): void {
  while (storedImagePaths.size >= MAX_STORED_IMAGE_PATHS) {
    const oldest = storedImagePaths.keys().next().value
    if (oldest !== undefined) {
      storedImagePaths.delete(oldest)
    } else {
      break
    }
  }
}
```

> 注意：这里只从内存 Map 中删除最旧的条目，**不会删除磁盘上的实际文件**。磁盘文件的生命周期由 `cleanupOldImageCaches` 管理。

### 文件写入流程
```typescript
const fh = await open(imagePath, 'w', 0o600)
try {
  await fh.writeFile(content.content, { encoding: 'base64' })
  await fh.datasync()
} finally {
  await fh.close()
}
```

- `open` 的 `w` 标志会截断已存在文件。由于 `content.id` 在会话内通常是唯一的，这可以接受。
- `0o600` 权限确保图像文件不会被其他用户读取，保护隐私。
- `datasync()` 确保数据落盘，防止崩溃后图像损坏。

### 清理旧缓存
`cleanupOldImageCaches` 使用 `getFsImplementation()` 获取文件系统抽象，而非直接使用 `fs/promises`。这是为了兼容测试环境（可以注入 mock fs）。清理逻辑：
1. 读取 `image-cache` 根目录。
2. 跳过当前 `sessionId`。
3. 对每个旧会话目录调用 `fsImpl.rm(sessionPath, { recursive: true, force: true })`。
4. 最后检查根目录是否为空，空则 `rmdir`。

所有错误都被静默捕获（`try/catch`），避免清理失败影响主流程。

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/imageStore.ts:54-79` | `storeImage` 核心写入逻辑 |
| `src/utils/imageStore.ts:84-99` | `storeImages` 批量存储 |
| `src/utils/imageStore.ts:41-49` | `cacheImagePath` 内存缓存 |
| `src/utils/imageStore.ts:104-106` | `getStoredImagePath` 查询 |
| `src/utils/imageStore.ts:129-167` | `cleanupOldImageCaches` 旧缓存清理 |
| `src/utils/config.ts` | `PastedContent` 类型定义 |
| `src/bootstrap/state.ts` | `getSessionId` 获取当前会话 ID |
| `src/utils/envUtils.ts` | `getClaudeConfigHomeDir` 配置目录 |
| `src/utils/fsOperations.ts` | `getFsImplementation` 文件系统抽象 |
| `src/utils/debug.ts` | `logForDebugging` 调试日志 |

## 依赖与外部交互

### 内部依赖
- `../bootstrap/state.js`：`getSessionId`
- `./config.js`：`PastedContent` 类型
- `./debug.js`：`logForDebugging`
- `./envUtils.js`：`getClaudeConfigHomeDir`
- `./fsOperations.js`：`getFsImplementation`
- `fs/promises`：`mkdir`, `open`
- `path`：`join`

### 调用方
- `src/utils/cleanup.ts`：调用 `cleanupOldImageCaches`
- `src/components/ClickableImageRef.tsx`：调用 `getStoredImagePath`

## 风险、边界与改进建议

### 风险与边界
1. **文件覆盖风险**：`storeImage` 使用 `open(path, 'w')`，若同一 `content.id` 被重复存储，会静默覆盖旧文件。虽然 `id` 在会话内通常是自增的，但异常情况下可能丢失旧图像。
2. **内存缓存与磁盘不一致**：`clearStoredImagePaths` 只清内存，不清磁盘；反之 `cleanupOldImageCaches` 只清磁盘，不更新内存。若两者在运行时被交叉调用，可能导致 `getStoredImagePath` 返回已删除的路径。
3. **无大小限制**：`storeImage` 不对 `content.content` 的大小做限制，若用户粘贴了一个 100MB 的 base64 图像，会直接写入磁盘。
4. **权限掩码问题**：`0o600` 会被 umask 影响，在某些系统上可能实际创建为 `0o644`。
5. **旧缓存清理的并发安全**：`cleanupOldImageCaches` 在遍历目录时，若另一个进程正在同一目录中创建文件，可能导致 `readdir` 结果不一致或 `rmdir` 失败（已被静默捕获）。

### 改进建议
1. **原子写入**：使用 `writeFile(path + '.tmp')` + `rename` 模式替代直接 `open(path, 'w')`，避免写入过程中崩溃导致文件半写。
2. **磁盘配额/大小限制**：在 `storeImage` 前检查 base64 解码后的大小，超过某个阈值（如 10MB）时拒绝存储或记录警告。
3. **同步内存与磁盘**：在 `cleanupOldImageCaches` 完成后，检查 `storedImagePaths` 中是否有已被删除的路径并一并清除。
4. **缓存持久化**：考虑将会话关闭时的图像路径列表写入一个索引文件，以便在异常重启后仍能恢复 `storedImagePaths` 缓存。
5. **定期清理调度**：目前 `cleanupOldImageCaches` 似乎只在启动或特定时机调用，可以考虑在图像缓存目录超过一定大小时触发清理。
