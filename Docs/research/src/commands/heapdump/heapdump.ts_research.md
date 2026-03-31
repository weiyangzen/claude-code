# 研究文档：src/commands/heapdump/heapdump.ts

## 场景与职责

本文件是 `/heapdump` 命令的实际执行实现，属于 Claude Code CLI 的本地命令（Local Command）模块。该命令是一个**隐藏的调试工具**，主要用于：

1. **内存问题诊断**：当应用出现内存泄漏或高内存占用时，生成 V8 堆内存快照
2. **开发调试**：为 Anthropic 内部团队（ant 用户）提供 JS 堆分析能力
3. **自动/手动触发**：支持用户手动执行，也可由系统的内存监控自动触发

该命令被设计为 `local` 类型命令，意味着它直接在本地执行而不需要模型交互，返回文本结果。

## 功能点目的

### 核心功能
- 调用 `heapDumpService.performHeapDump()` 生成堆内存快照
- 返回堆快照文件路径和诊断信息文件路径
- 错误处理：捕获并返回友好的错误信息

### 设计特点
- **简单代理模式**：本文件本身不包含复杂逻辑，而是委托给 `heapDumpService` 处理
- **统一返回格式**：遵循 `LocalCommandResult` 类型规范，返回 `{ type: 'text', value: string }`
- **支持非交互模式**：`supportsNonInteractive: true` 允许在自动化场景下调用

## 具体技术实现

### 关键流程

```
用户执行 /heapdump
    ↓
call() 函数被调用
    ↓
performHeapDump() [来自 heapDumpService]
    ↓
返回结果处理：
  - 成功：返回堆快照路径 + 诊断文件路径
  - 失败：返回错误信息
```

### 数据结构

**函数签名**：
```typescript
export async function call(): Promise<{ type: 'text'; value: string }>
```

**返回结果格式**：
- 成功时：`value` 包含两行文本：
  - 第一行：堆快照文件路径（`.heapsnapshot`）
  - 第二行：诊断信息文件路径（`-diagnostics.json`）
- 失败时：`value` 包含错误前缀和具体错误信息

### 错误处理

```typescript
if (!result.success) {
  return {
    type: 'text',
    value: `Failed to create heap dump: ${result.error}`,
  }
}
```

采用简单的布尔标志检查，错误信息直接透传给用户。

## 关键代码路径与文件引用

### 直接依赖

| 文件路径 | 依赖类型 | 说明 |
|---------|---------|------|
| `src/utils/heapDumpService.ts` | 核心依赖 | 提供 `performHeapDump()` 函数，包含实际的堆转储逻辑 |

### 调用链

```
src/commands/heapdump/heapdump.ts
  └── performHeapDump() [src/utils/heapDumpService.ts]
       ├── captureMemoryDiagnostics() [同文件]
       │   ├── process.memoryUsage()
       │   ├── v8.getHeapStatistics()
       │   ├── v8.getHeapSpaceStatistics() [Bun 环境可能不可用]
       │   ├── process._getActiveHandles()
       │   ├── process._getActiveRequests()
       │   └── /proc/self/fd, /proc/self/smaps_rollup [Linux 特有]
       └── writeHeapSnapshot() [同文件]
            ├── Bun.generateHeapSnapshot() [Bun 运行时]
            └── v8.getHeapSnapshot() + stream.pipeline() [Node 运行时]
```

### 被调用方

通过 `src/commands/heapdump/index.ts` 的 `load` 函数懒加载：

```typescript
// index.ts
load: () => import('./heapdump.js')
```

实际调用发生在命令系统的本地命令执行流程中。

## 依赖与外部交互

### 运行时依赖

1. **Node.js V8 模块**：通过 `v8` 模块获取堆统计信息和快照
2. **Bun 兼容性**：在 Bun 运行时中使用 `Bun.generateHeapSnapshot()`
3. **文件系统**：写入 `~/Desktop` 目录（通过 `getDesktopPath()` 确定）

### 输出文件

命令执行后会在用户桌面生成两个文件：

1. **`<sessionId>.heapsnapshot`**：V8 堆快照，可用 Chrome DevTools 分析
2. **`<sessionId>-diagnostics.json`**：内存诊断信息，包含：
   - 内存使用统计（heapUsed, heapTotal, rss, external, arrayBuffers）
   - V8 堆统计（heapSizeLimit, mallocedMemory, detachedContexts 等）
   - 活跃句柄和请求计数
   - 潜在内存泄漏分析
   - 平台信息（Linux smaps_rollup 等）

### 权限要求

- 需要写入用户桌面目录的权限
- Linux 下读取 `/proc/self/fd` 和 `/proc/self/smaps_rollup` 需要相应权限

## 风险、边界与改进建议

### 已知风险

1. **大堆内存崩溃风险**：
   - 注释说明 V8 堆快照序列化在非常大的堆上可能崩溃
   - 缓解措施：诊断信息在生成快照前写入，确保即使快照失败也有诊断数据

2. **Bun 运行时限制**：
   - `getHeapSpaceStatistics()` 在 Bun 中不可用
   - Bun 使用同步写入（`writeFileSync`）而非流式写入

3. **平台差异**：
   - Linux 特有功能（`/proc` 文件系统）在其他平台会静默失败
   - macOS/Windows 的桌面路径解析逻辑不同

### 边界情况

| 场景 | 行为 |
|-----|------|
| 堆内存极大（>几GB） | 快照生成可能崩溃，但诊断文件已保存 |
| Bun 运行时 | 使用替代 API，部分统计信息不可用 |
| 非 Linux 平台 | /proc 相关统计不可用，其他功能正常 |
| 桌面目录不可写 | 返回错误信息 |
| 手动触发 | dumpNumber = 0 |
| 自动触发（1.5GB阈值） | dumpNumber 递增，用于区分多次自动转储 |

### 改进建议

1. **进度指示**：当前实现无进度反馈，大堆快照时用户可能以为卡死
2. **可配置输出路径**：目前固定输出到桌面，建议支持自定义路径
3. **快照压缩**：堆快照文件通常很大，可考虑自动压缩
4. **自动清理**：添加机制自动清理旧的堆快照文件
5. **内存阈值可配置**：当前 1.5GB 自动触发阈值硬编码，建议可配置

### 安全考虑

- 堆快照包含应用内存的完整内容，可能包含敏感数据
- 文件权限设置为 `0o600`（仅所有者可读写）
- 命令被标记为 `isHidden: true`，不对普通用户显示
