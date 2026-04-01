# 研究文档：src/utils/fileOperationAnalytics.ts

## 场景与职责

`fileOperationAnalytics.ts` 是一个轻量级的隐私保护型分析日志模块。它的唯一职责是：在文件被读取、写入或编辑时，向 Statsig（通过 `src/services/analytics/index.js`）发送一个标准化事件 `tengu_file_operation`，同时**避免泄露原始文件路径和完整内容**。

该模块被 FileReadTool、FileWriteTool、FileEditTool 调用，是产品 telemetry 中文件操作埋点的统一收口。

## 功能点目的

| 导出项 | 目的 |
|--------|------|
| `logFileOperation(params)` | 对一次文件操作进行脱敏后上报 analytics。 |

`params` 包含：
- `operation`: `'read' | 'write' | 'edit'`
- `tool`: `'FileReadTool' | 'FileWriteTool' | 'FileEditTool'`
- `filePath`: 原始路径（会被哈希后再上报）
- `content?`: 原始内容（可选，仅在长度 ≤100KB 时会被哈希）
- `type?`: `'create' | 'update'`（可选，用于 write 操作区分新建/覆盖）

## 具体技术实现

### 哈希策略

```ts
function hashFilePath(filePath: string): string {
  return createHash('sha256').update(filePath).digest('hex').slice(0, 16)
}

function hashFileContent(content: string): string {
  return createHash('sha256').update(content).digest('hex') // 64 chars
}
```

- **路径**：截断到 16 位十六进制，足够用于聚合统计，同时不可逆。
- **内容**：完整 64 位十六进制，用于 deduplication 和变更检测分析。

### 内容大小限制

```ts
const MAX_CONTENT_HASH_SIZE = 100 * 1024 // 100KB
```

- 若 `content` 超过 100KB，直接跳过 `contentHash` 字段，防止对 base64 图片等大文件做哈希时耗尽内存。

### 类型安全

返回的 metadata 使用 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`  branded type，强制调用方在编译期确认不会把代码或路径原样传进 analytics。

## 关键代码路径与文件引用

### 调用方

| 文件 | 调用点 | 说明 |
|------|--------|------|
| `src/tools/FileEditTool/FileEditTool.ts` | `logFileOperation` | 文件编辑成功后上报。 |
| `src/tools/FileReadTool/FileReadTool.ts` | `logFileOperation` | 文件读取成功后上报。 |
| `src/tools/FileWriteTool/FileWriteTool.ts` | `logFileOperation` | 文件写入成功后上报。 |

### 被调用方

- `src/services/analytics/index.js`：`logEvent`
- Node.js `crypto`：`createHash`

## 依赖与外部交互

- **无外部网络直接调用**：所有数据通过内部的 `logEvent` 统一封装后进入 Statsig。
- **无持久化**：纯内存计算 + 即时上报。
- **无配置项**：`MAX_CONTENT_HASH_SIZE` 是硬编码常量。

## 风险、边界与改进建议

### 风险

1. **哈希冲突**：16 位截断的 SHA-256 在极大规模数据下存在理论冲突，但对 analytics 聚合而言可接受。
2. **内容缺失**：超过 100KB 的文件不报告 `contentHash`，导致大文件变更无法被 telemetry 追踪；目前这是有意为之的 trade-off。
3. **同步阻塞**：`createHash` 和 `digest` 是同步 CPU 密集型操作。在极端高频批量文件操作场景下（如一次性读取数千个小文件），可能短暂阻塞事件循环。

### 边界

- 仅处理三种工具、三种 operation；其他文件操作（如 `NotebookEditTool`、BashTool 的间接文件操作）不经过此模块。
- `content` 为可选参数，调用方可能遗漏传入，导致内容变更分析不完整。

### 改进建议

1. **异步哈希**：若未来需要处理超大内容，可考虑使用 `crypto.createHash` 的流式接口，或将哈希任务 offload 到 worker thread。
2. **可配置阈值**：将 `MAX_CONTENT_HASH_SIZE` 暴露为环境变量或配置项，便于不同部署形态调整。
3. **扩展覆盖**：考虑将 `NotebookEditTool` 也纳入 `logFileOperation` 的埋点范围，使文件操作 analytics 更完整。
