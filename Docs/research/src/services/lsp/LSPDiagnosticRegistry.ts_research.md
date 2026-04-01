# LSPDiagnosticRegistry.ts 深度研究文档

## 场景与职责

`LSPDiagnosticRegistry.ts` 是 LSP 诊断信息的中央注册表，负责接收、去重、限流和分发来自 LSP 服务器的异步诊断通知（`textDocument/publishDiagnostics`）。

**核心职责：**
1. **诊断收集**：接收来自多个 LSP 服务器的诊断通知
2. **去重处理**：防止同一诊断在多次查询中重复发送
3. **容量限制**：限制每文件和总诊断数量，防止信息过载
4. **跨轮次追踪**：使用 LRU 缓存追踪已交付的诊断，实现跨查询轮次的去重
5. **异步交付**：与 `AsyncHookRegistry` 模式一致，支持异步附件交付

**在系统中的位置：**
- 被 `passiveFeedback.ts` 调用，注册来自 LSP 服务器的诊断
- 被附件系统调用（通过 `checkForLSPDiagnostics`）获取待处理的诊断
- 与 `diagnosticTracking.ts` 中的 `DiagnosticTrackingService` 互补（后者处理 IDE 诊断）

---

## 功能点目的

### 1. PendingLSPDiagnostic 类型
定义待处理诊断的数据结构：
```typescript
export type PendingLSPDiagnostic = {
  serverName: string      // 发送诊断的服务器
  files: DiagnosticFile[] // 诊断文件列表
  timestamp: number       // 接收时间戳
  attachmentSent: boolean // 是否已发送附件
}
```

### 2. 全局状态管理

**两个核心存储：**
```typescript
// 待处理诊断（当前轮次）
const pendingDiagnostics = new Map<string, PendingLSPDiagnostic>()

// 已交付诊断（跨轮次去重，LRU 限制内存）
const deliveredDiagnostics = new LRUCache<string, Set<string>>({
  max: MAX_DELIVERED_FILES,  // 500
})
```

### 3. 诊断注册流程

**`registerPendingLSPDiagnostic()`：**
1. 生成 UUID 作为诊断 ID（处理快速连续注册）
2. 存储到 `pendingDiagnostics` Map
3. 标记为未发送 (`attachmentSent: false`)

### 4. 诊断检索与处理流程

**`checkForLSPDiagnostics()` - 核心处理流程：**

1. **收集**：从所有待处理诊断中收集文件
2. **去重**：`deduplicateDiagnosticFiles()` - 基于内容哈希的去重
3. **限流**：
   - 每文件最多 10 条诊断 (`MAX_DIAGNOSTICS_PER_FILE`)
   - 总共最多 30 条诊断 (`MAX_TOTAL_DIAGNOSTICS`)
4. **排序**：按严重程度排序（Error < Warning < Info < Hint）
5. **追踪**：记录已交付诊断用于跨轮次去重
6. **清理**：从 pending 中移除已发送的诊断

### 5. 去重算法

**基于内容的去重键：**
```typescript
function createDiagnosticKey(diag): string {
  return jsonStringify({
    message: diag.message,
    severity: diag.severity,
    range: diag.range,
    source: diag.source || null,
    code: diag.code || null,
  })
}
```

**两级去重：**
- **批次内去重**：同一批次中相同文件+相同内容的诊断
- **跨轮次去重**：使用 `deliveredDiagnostics` LRU 缓存追踪历史诊断

---

## 具体技术实现

### 关键常量

```typescript
const MAX_DIAGNOSTICS_PER_FILE = 10   // 单文件诊断上限
const MAX_TOTAL_DIAGNOSTICS = 30      // 总诊断上限
const MAX_DELIVERED_FILES = 500       // LRU 缓存大小限制
```

### 严重程度映射

```typescript
function severityToNumber(severity: string | undefined): number {
  switch (severity) {
    case 'Error': return 1
    case 'Warning': return 2
    case 'Info': return 3
    case 'Hint': return 4
    default: return 4
  }
}
```

### 去重实现细节

```typescript
function deduplicateDiagnosticFiles(allFiles: DiagnosticFile[]): DiagnosticFile[] {
  const fileMap = new Map<string, Set<string>>()  // 文件 -> 已见诊断键
  const dedupedFiles: DiagnosticFile[] = []

  for (const file of allFiles) {
    // 获取该文件的历史已交付诊断
    const previouslyDelivered = deliveredDiagnostics.get(file.uri) || new Set()
    
    for (const diag of file.diagnostics) {
      const key = createDiagnosticKey(diag)
      
      // 跳过：本批次已见 或 历史已交付
      if (seenDiagnostics.has(key) || previouslyDelivered.has(key)) {
        continue
      }
      
      seenDiagnostics.add(key)
      dedupedFile.diagnostics.push(diag)
    }
  }
}
```

### 容量限制实现

```typescript
// 按严重程度排序（Error 优先）
file.diagnostics.sort((a, b) => severityToNumber(a.severity) - severityToNumber(b.severity))

// 单文件限制
if (file.diagnostics.length > MAX_DIAGNOSTICS_PER_FILE) {
  file.diagnostics = file.diagnostics.slice(0, MAX_DIAGNOSTICS_PER_FILE)
}

// 总数量限制
const remainingCapacity = MAX_TOTAL_DIAGNOSTICS - totalDiagnostics
if (file.diagnostics.length > remainingCapacity) {
  file.diagnostics = file.diagnostics.slice(0, remainingCapacity)
}
```

### 状态清理 API

```typescript
// 清理待处理诊断（关闭/测试时使用）
export function clearAllLSPDiagnostics(): void

// 重置所有状态（会话重置时使用）
export function resetAllLSPDiagnosticState(): void

// 文件编辑后清除该文件的已交付记录
export function clearDeliveredDiagnosticsForFile(fileUri: string): void
```

---

## 关键代码路径与文件引用

### 内部依赖

| 文件 | 用途 |
|------|------|
| `crypto` (Node.js) | `randomUUID()` - 生成诊断 ID |
| `lru-cache` (npm) | LRU 缓存实现 |
| `../../utils/debug.js` | `logForDebugging()` - 调试日志 |
| `../../utils/errors.js` | `toError()` - 错误转换 |
| `../../utils/log.js` | `logError()` - 错误日志 |
| `../../utils/slowOperations.js` | `jsonStringify()` - JSON 序列化 |
| `../diagnosticTracking.js` | `DiagnosticFile` 类型 |

### 被调用方

| 文件 | 调用函数 |
|------|----------|
| `passiveFeedback.ts` | `registerPendingLSPDiagnostic()` - 注册新诊断 |
| 附件系统 | `checkForLSPDiagnostics()` - 获取待处理诊断 |
| 测试文件 | `clearAllLSPDiagnostics()`, `resetAllLSPDiagnosticState()` |

### 关键代码行

| 行号 | 功能 |
|------|------|
| 12-21 | `PendingLSPDiagnostic` 类型定义 |
| 41-46 | 容量限制常量 |
| 49-56 | 全局状态定义（pending + delivered）|
| 65-85 | `registerPendingLSPDiagnostic()` 实现 |
| 91-104 | 严重程度映射函数 |
| 110-124 | 诊断键生成（去重基础）|
| 136-184 | 去重核心算法 |
| 193-338 | `checkForLSPDiagnostics()` 完整流程 |
| 256-288 | 容量限制与排序 |
| 291-312 | 已交付诊断追踪 |
| 346-386 | 状态清理 API |

---

## 依赖与外部交互

### 外部依赖

**Node.js 内置：**
- `crypto.randomUUID` - UUID 生成

**第三方库：**
- `lru-cache` - LRU 缓存实现（控制内存增长）

### 数据流

```
LSP Server → passiveFeedback.ts → registerPendingLSPDiagnostic()
                                              ↓
                                     pendingDiagnostics (Map)
                                              ↓
         Attachment System ← checkForLSPDiagnostics() ← 查询时
                                              ↓
                              去重 → 限流 → 交付
                                              ↓
                                     deliveredDiagnostics (LRU)
```

### 与 diagnosticTracking.ts 的关系

| | `LSPDiagnosticRegistry` | `DiagnosticTrackingService` |
|--|--------------------------|----------------------------|
| **来源** | LSP 服务器（插件） | IDE（MCP 连接） |
| **触发** | 异步通知 | 主动查询 |
| **用途** | 代码分析、智能提示 | 文件编辑前后对比 |
| **协议** | LSP `publishDiagnostics` | MCP `getDiagnostics` |

---

## 风险、边界与改进建议

### 已知风险

**1. 内存增长风险**
- `deliveredDiagnostics` 使用 LRU 限制为 500 文件
- 但每个文件的诊断集合可能无限增长（如果诊断不断变化）
- 缓解：LRU 的 `max` 限制文件数量，但单个文件的诊断集合无上限

**2. JSON 序列化错误**
```typescript
try {
  const key = createDiagnosticKey(diag)
} catch (error) {
  // 失败时仍包含诊断（避免信息丢失）
  dedupedFile.diagnostics.push(diag)
}
```
- 某些诊断可能包含循环引用，导致 `jsonStringify` 失败
- 当前处理是保守的：失败时仍包含诊断

**3. 时间戳未使用**
- `timestamp` 字段被记录但未用于过期逻辑
- 理论上可以添加 TTL 机制清理过旧诊断

### 边界情况

**1. 空诊断处理**
```typescript
if (firstFile.diagnostics.length === 0) {
  return []  // 跳过空诊断
}
```

**2. 所有诊断都被去重**
```typescript
if (finalCount === 0) {
  logForDebugging(`No new diagnostics to deliver (all filtered by deduplication)`)
  return []
}
```

**3. 文件编辑后重新显示**
- `clearDeliveredDiagnosticsForFile()` 允许文件编辑后重新显示相同诊断
- 当前由调用方（如 FileEditTool）决定何时调用

### 改进建议

**1. 添加诊断 TTL**
```typescript
// 建议：添加配置选项
const DIAGNOSTIC_TTL_MS = 5 * 60 * 1000  // 5 分钟

// 在 checkForLSPDiagnostics 中过滤过期诊断
if (Date.now() - diagnostic.timestamp > DIAGNOSTIC_TTL_MS) {
  pendingDiagnostics.delete(id)
  continue
}
```

**2. 更智能的容量限制**
- 当前按文件顺序截断，可能导致重要文件被忽略
- 建议：按严重程度全局排序后再截断

**3. 诊断来源追踪增强**
- 当前仅记录 `serverName`
- 建议：添加 LSP 版本、服务器 capabilities 等元数据

**4. 批量注册优化**
```typescript
// 当前：每个诊断单独注册（UUID 生成）
// 建议：支持批量注册接口
export function registerPendingLSPDiagnostics(
  items: Array<{serverName, files}>
): void
```

**5. 监控指标**
- 添加计数器：去重率、限流率、平均诊断数
- 有助于优化容量限制常量

### 测试建议

**关键测试场景：**
1. 同一诊断多次注册（去重验证）
2. 超过 500 个不同文件的诊断（LRU 淘汰）
3. 包含循环引用的诊断对象（JSON 序列化错误处理）
4. 大量诊断的容量限制（排序和截断逻辑）
5. 并发注册和检索（竞态条件）
6. 文件编辑后 `clearDeliveredDiagnosticsForFile` 效果

### 相关配置

当前容量限制为硬编码，建议未来可配置：
```typescript
// 建议添加配置接口
interface LSPDiagnosticConfig {
  maxPerFile: number      // 默认 10
  maxTotal: number        // 默认 30
  maxDeliveredFiles: number  // 默认 500
  enableCrossTurnDedup: boolean  // 默认 true
}
```
