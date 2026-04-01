# sessionStoragePortable.ts 深度研究

## 场景与职责

`sessionStoragePortable.ts` 是一个**纯 Node.js 工具模块**，设计为 CLI 和 VS Code 扩展共享的底层会话存储基础设施。其核心职责是提供与平台无关的会话文件读写、路径解析和元数据提取能力。

**关键设计目标：**
- **零内部依赖**：不依赖 logging、experiments、feature flags 等 CLI 特有模块
- **跨运行时兼容**：同时支持 Bun (CLI) 和 Node.js (SDK/VS Code 扩展)
- **高性能 I/O**：针对大文件 (>5MB) 优化，避免全量读取
- **路径可移植性**：处理不同平台的路径差异和符号链接

**主要调用方：**
- `src/utils/sessionStorage.ts` - CLI 主存储实现
- `src/utils/attribution.ts` - 会话归属统计
- `src/utils/listSessionsImpl.ts` - 会话列表实现
- `packages/claude-vscode/src/common-host/sessionStorage.ts` - VS Code 扩展

---

## 功能点目的

### 1. UUID 验证与解析
```typescript
const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i
export function validateUuid(maybeUuid: unknown): UUID | null
```
- 严格验证 UUID v4 格式
- 用于会话 ID 的合法性校验

### 2. JSON 字符串字段提取（轻量级解析）
```typescript
export function extractJsonStringField(text: string, key: string): string | undefined
export function extractLastJsonStringField(text: string, key: string): string | undefined
export function unescapeJsonString(raw: string): string
```
- **目的**：避免完整 JSON 解析的开销，直接从原始文本中提取特定字段
- **应用场景**：从会话文件尾部快速读取 `customTitle`、`tag` 等元数据
- **技术特点**：
  - 支持 `"key":"value"` 和 `"key": "value"` 两种格式
  - 处理 JSON 转义序列
  - `extractLastJsonStringField` 用于获取追加字段的最新值

### 3. 首个提示词提取
```typescript
export function extractFirstPromptFromHead(head: string): string
```
- **目的**：从会话文件头部提取用户的第一条有意义消息，用于会话列表展示
- **过滤逻辑**：
  - 跳过 `tool_result`、`isMeta`、`isCompactSummary` 消息
  - 跳过自动生成的系统消息（XML 标签开头、中断标记）
  - 提取 bash 输入并格式化为 `! <command>`
  - 截断至 200 字符

### 4. 文件头尾读取
```typescript
export const LITE_READ_BUF_SIZE = 65536  // 64KB
export async function readHeadAndTail(filePath: string, fileSize: number, buf: Buffer): Promise<{ head: string; tail: string }>
export async function readSessionLite(filePath: string): Promise<LiteSessionFile | null>
```
- **目的**：高效读取大文件的首尾部分，用于元数据扫描
- **优化点**：
  - 使用共享 Buffer 避免重复分配
  - 小文件时 `tail === head` 避免重复读取
  - 单文件描述符完成头尾读取

### 5. 路径处理与项目目录发现
```typescript
export const MAX_SANITIZED_LENGTH = 200
export function sanitizePath(name: string): string
export function getProjectsDir(): string
export function getProjectDir(projectDir: string): string
export async function canonicalizePath(dir: string): Promise<string>
export async function findProjectDir(projectPath: string): Promise<string | undefined>
```
- **路径清理**：
  - 非字母数字字符替换为连字符
  - 超长路径 (>200 字符) 截断并追加哈希后缀
  - 支持 Bun.hash 和 djb2Hash 回退
- **项目目录发现**：
  - 支持精确匹配和前缀匹配（处理 Bun/Node 哈希差异）
  - NFC Unicode 规范化处理 macOS 合成字符

### 6. 会话文件路径解析
```typescript
export async function resolveSessionFilePath(
  sessionId: string,
  dir?: string
): Promise<{ filePath: string; projectPath: string | undefined; fileSize: number } | undefined>
```
- **解析策略**：
  1. 指定目录时：规范化路径 → 查找项目目录 → 检查文件存在性
  2. 未指定目录时：扫描所有项目目录
  3. 支持 git worktree 回退查找
- **零字节文件处理**：视为不存在，继续搜索

### 7. 转录文件分块读取（压缩边界处理）
```typescript
export const SKIP_PRECOMPACT_THRESHOLD = 5 * 1024 * 1024  // 5MB
export async function readTranscriptForLoad(
  filePath: string,
  fileSize: number
): Promise<{ boundaryStartOffset: number; postBoundaryBuf: Buffer; hasPreservedSegment: boolean }>
```
- **目的**：加载会话时跳过压缩前的历史记录，减少内存占用
- **核心机制**：
  - 扫描 `"compact_boundary"` 标记
  - 处理 `attribution-snapshot` 行（占大会话文件的 84% 字节）
  - 支持 `preservedSegment` 边界（保留部分历史）
  - 1MB 分块读取，输出缓冲区动态增长

---

## 具体技术实现

### 轻量级 JSON 字段提取算法
```typescript
// 双模式匹配：紧凑格式和带空格格式
const patterns = [`"${key}":"`, `"${key}": "`]
// 逐字符扫描处理转义
while (i < text.length) {
  if (text[i] === '\\') { i += 2; continue }
  if (text[i] === '"') { return unescapeJsonString(...) }
}
```

### 分块读取状态机
```typescript
type LoadState = {
  out: Sink                    // 输出缓冲区
  boundaryStartOffset: number  // 边界起始偏移
  hasPreservedSegment: boolean // 是否保留段
  lastSnapSrc: Buffer | null   // 最后一个 attr-snap
  bufFileOff: number           // 当前缓冲区文件偏移
  carryLen: number             // 跨块携带长度
  // ...
}
```

处理流程：
1. `processStraddle` - 处理跨块边界的不完整行
2. `scanChunkLines` - 扫描块内行，过滤 attr-snap，检测边界
3. `captureSnap` - 捕获最后一个 attr-snap
4. `finalizeOutput` - 将最后一个 snap 追加到输出末尾

### 路径哈希兼容性
```typescript
function simpleHash(str: string): string {
  return Math.abs(djb2Hash(str)).toString(36)
}
// Bun 环境使用 Bun.hash，Node 环境使用 djb2Hash
const hash = typeof Bun !== 'undefined' ? Bun.hash(name).toString(36) : simpleHash(name)
```

---

## 关键代码路径与文件引用

### 核心导出
| 导出 | 用途 |
|------|------|
| `LITE_READ_BUF_SIZE` | 64KB 轻量读取缓冲区大小 |
| `validateUuid` | UUID 格式验证 |
| `extractJsonStringField` | 轻量级 JSON 字段提取 |
| `extractFirstPromptFromHead` | 首个提示词提取 |
| `readHeadAndTail` | 文件头尾读取 |
| `readSessionLite` | 轻量级会话文件读取 |
| `sanitizePath` | 路径清理 |
| `getProjectsDir` | 项目根目录 |
| `getProjectDir` | 特定项目目录 |
| `canonicalizePath` | 路径规范化 |
| `findProjectDir` | 项目目录查找 |
| `resolveSessionFilePath` | 会话文件路径解析 |
| `readTranscriptForLoad` | 转录文件加载 |
| `SKIP_PRECOMPACT_THRESHOLD` | 跳过预压缩阈值 |

### 依赖模块
- `src/utils/envUtils.ts` - `getClaudeConfigHomeDir`
- `src/utils/getWorktreePathsPortable.ts` - worktree 路径获取
- `src/utils/hash.ts` - `djb2Hash`

### 被调用方
- `src/utils/sessionStorage.ts` - 主存储实现
- `src/utils/attribution.ts` - 归属统计
- `src/utils/listSessionsImpl.ts` - 会话列表
- `src/utils/path.ts` - 路径工具

---

## 依赖与外部交互

### 外部依赖
| 模块 | 用途 |
|------|------|
| `crypto` (UUID) | 类型定义 |
| `fs/promises` | 异步文件操作 |
| `path` | 路径拼接 |

### 内部依赖
| 模块 | 用途 |
|------|------|
| `envUtils.js` | 配置目录获取 |
| `getWorktreePathsPortable.js` | worktree 检测 |
| `hash.js` | djb2 哈希 |

### 环境兼容性
- **Bun**：使用 `Bun.hash` 进行快速哈希
- **Node.js**：使用 `djb2Hash` 回退

---

## 风险、边界与改进建议

### 已知风险

1. **哈希不一致风险**
   - Bun.hash 和 djb2Hash 产生不同结果，导致长路径项目目录不匹配
   - 缓解：`findProjectDir` 实现了前缀回退匹配

2. **大文件处理**
   - 5MB 以下文件不跳过预压缩扫描，可能加载过多历史
   - 边界：`readTranscriptForLoad` 输出缓冲区上限 `fileSize + 1`

3. **字符编码**
   - 假设 UTF-8 编码，未处理其他编码
   - NFC 规范化仅应用于路径，不应用于文件内容

4. **并发安全**
   - `readSessionLite` 每个调用分配独立 Buffer，并发安全
   - `readHeadAndTail` 需要调用方提供共享 Buffer

### 边界情况

| 场景 | 处理 |
|------|------|
| 零字节文件 | 视为不存在 |
| 截断文件（无换行符结尾） | `finalizeOutput` 自动插入换行符 |
| 跨块 attr-snap | `straddleSnapCarryLen` 机制处理 |
| 多个 compact_boundary | 以最后一个为准，但保留 `hasPreservedSegment` 标记 |
| Windows 路径 | `sanitizePath` 将反斜杠替换为连字符 |

### 改进建议

1. **性能优化**
   - 考虑使用内存映射文件处理超大转录文件
   - 添加 LRU 缓存缓存频繁访问的会话元数据

2. **健壮性**
   - 添加 JSON 字段提取的模糊匹配（处理不同空白格式）
   - 增加文件编码检测和转换

3. **可观测性**
   - 添加慢操作日志（大文件读取、worktree 检测）
   - 记录哈希不匹配回退事件

4. **兼容性**
   - 考虑标准化哈希算法，消除 Bun/Node 差异
   - 添加版本标记到项目目录，支持未来迁移
