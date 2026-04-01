# memoryScan.ts 研究文档

## 场景与职责

`memoryScan.ts` 是记忆系统的**目录扫描基础设施**，提供记忆文件的发现和头部信息提取功能。它是记忆召回和提取功能的底层支撑模块。

### 核心职责
1. **递归目录扫描**：扫描记忆目录中的所有 `.md` 文件（排除 `MEMORY.md`）
2. **头部信息提取**：读取文件前30行，解析 frontmatter
3. **元数据收集**：收集文件名、路径、修改时间、描述、类型等信息
4. **结果排序和截断**：按修改时间排序（最新优先），最多返回200个

### 使用场景
- `findRelevantMemories.ts`：查询时召回相关记忆
- `extractMemories.ts`：预注入记忆列表，避免提取 agent 花费一轮 `ls`
- 任何需要获取记忆目录文件清单的场景

### 设计动机
该模块从 `findRelevantMemories.ts` 中分离出来，目的是：
- 打破循环依赖（`findRelevantMemories` → `sideQuery` → API 客户端链 → `memdir.ts`）
- 允许 `extractMemories` 导入扫描功能而不引入 `sideQuery`

---

## 功能点目的

### 1. `scanMemoryFiles()` - 记忆文件扫描
**目的**：扫描记忆目录，返回带元数据的记忆头部列表

**关键参数**：
- `memoryDir`: 要扫描的目录路径
- `signal`: 中止信号

**返回值**：`MemoryHeader[]` - 按修改时间排序（最新优先），最多200个

**实现细节**：
- 使用 `fs.readdir` 递归扫描（`{ recursive: true }`）
- 过滤 `.md` 文件，排除 `MEMORY.md`
- 使用 `Promise.allSettled` 并行读取文件头部
- 每个文件读取前30行（`FRONTMATTER_MAX_LINES`）

### 2. `formatMemoryManifest()` - 记忆清单格式化
**目的**：将记忆头部列表格式化为文本清单，用于提示词

**输出格式**：
```
- [type] filename (2026-04-01T12:00:00.000Z): description
- [type] filename (2026-04-01T12:00:00.000Z)
```

**使用场景**：
- 相关性选择器的提示词
- 提取 agent 的提示词

---

## 具体技术实现

### 关键流程

```
scanMemoryFiles(memoryDir, signal)
  ↓
readdir(memoryDir, { recursive: true })  // 递归列出所有文件
  ↓
过滤：.md 文件 && basename !== 'MEMORY.md'
  ↓
Promise.allSettled(
  每个文件 → readFileInRange(filePath, 0, 30)
    ↓
    parseFrontmatter(content, filePath)
      ↓
    构造 MemoryHeader
)
  ↓
过滤 fulfilled 结果
  ↓
按 mtimeMs 降序排序
  ↓
截取前 200 个
```

### 数据结构

#### MemoryHeader
```typescript
export type MemoryHeader = {
  filename: string           // 相对路径（如 "user/preferences.md"）
  filePath: string          // 绝对路径
  mtimeMs: number           // 修改时间戳（毫秒）
  description: string | null // frontmatter 中的描述
  type: MemoryType | undefined // 解析后的记忆类型
}
```

### 常量定义

```typescript
const MAX_MEMORY_FILES = 200        // 最大返回文件数
const FRONTMATTER_MAX_LINES = 30    // 读取的前几行数
```

### 性能优化

1. **单次读取优化**：
   - `readFileInRange` 内部会 stat 文件并返回 `mtimeMs`
   - 避免单独的 stat 轮询
   - 对于常见情况（N ≤ 200），相比 stat-sort-read 模式减少一半系统调用

2. **并行处理**：
   - 使用 `Promise.allSettled` 并行读取所有文件头部
   - 单个文件失败不影响其他文件

3. **快速失败**：
   - 目录读取失败时返回空数组（优雅降级）
   - 不抛出异常中断调用链

---

## 关键代码路径与文件引用

### 内部依赖
| 文件 | 用途 |
|------|------|
| `memoryTypes.ts` | `MemoryType`, `parseMemoryType()` |

### 外部依赖
| 文件 | 用途 |
|------|------|
| `../utils/frontmatterParser.ts` | `parseFrontmatter()` - 解析 YAML frontmatter |
| `../utils/readFileInRange.ts` | `readFileInRange()` - 行范围文件读取 |

### 调用方
| 文件 | 用途 |
|------|------|
| `src/memdir/findRelevantMemories.ts` | `scanMemoryFiles()` - 查询时召回 |
| `src/services/extractMemories/extractMemories.ts` | `scanMemoryFiles()`, `formatMemoryManifest()` - 记忆提取 |

---

## 依赖与外部交互

### 文件系统操作
- `fs/promises.readdir`: 递归目录扫描
- `path.basename`, `path.join`: 路径处理

### 外部工具函数
- `readFileInRange`: 高效的行范围读取，返回内容和 mtime
- `parseFrontmatter`: YAML frontmatter 解析

### 错误处理策略
```typescript
try {
  // ... 扫描逻辑
} catch {
  return []  // 任何错误都返回空数组
}
```

---

## 风险、边界与改进建议

### 已知风险

1. **递归扫描性能**
   - `fs.readdir` 的 `recursive: true` 在大型目录树中可能较慢
   - 没有深度限制，可能意外扫描深层嵌套目录

2. **内存占用**
   - 并行读取所有文件头部可能占用大量内存
   - 每个文件读取30行，200个文件 = 最多6000行内容同时驻留

3. **Frontmatter 解析失败**
   - `parseFrontmatter` 失败不会中断整个扫描
   - 但失败文件的 `description` 和 `type` 将为空

### 边界情况

| 场景 | 行为 |
|------|------|
| 目录不存在 | 返回空数组 |
| 目录为空 | 返回空数组 |
| 所有文件读取失败 | 返回空数组 |
| 文件超过200个 | 按时间排序，只返回最新的200个 |
| Frontmatter 超过30行 | 只解析前30行，可能解析不完整 |
| 文件无 frontmatter | `description: null`, `type: undefined` |

### 改进建议

1. **流式处理**
   ```typescript
   // 当前：一次性读取所有文件
   // 改进：使用异步生成器，按需产出
   export async function* scanMemoryFilesStream(memoryDir: string): AsyncGenerator<MemoryHeader>
   ```

2. **深度限制**
   ```typescript
   const MAX_DEPTH = 3  // 限制递归深度
   ```

3. **增量扫描**
   ```typescript
   // 支持基于 mtime 的增量更新
   export function scanMemoryFilesIncremental(
     memoryDir: string, 
     sinceMtimeMs: number
   ): Promise<MemoryHeader[]>
   ```

4. **更好的错误报告**
   ```typescript
   // 返回部分结果和错误列表
   export type ScanResult = {
     headers: MemoryHeader[]
     errors: Array<{ path: string; error: Error }>
   }
   ```

5. **缓存机制**
   ```typescript
   // 使用文件系统监视器（watcher）缓存扫描结果
   // 仅在文件变化时重新扫描
   ```

6. **可配置限制**
   ```typescript
   export interface ScanOptions {
     maxFiles?: number        // 默认 200
     maxLinesPerFile?: number // 默认 30
     includeMemoryMd?: boolean // 默认 false
   }
   ```

### 测试建议

```typescript
// 应该测试的场景
describe('scanMemoryFiles', () => {
  it('returns empty array for non-existent directory')
  it('excludes MEMORY.md from results')
  it('sorts results by mtime descending')
  it('limits results to MAX_MEMORY_FILES')
  it('handles files without frontmatter')
  it('handles parse errors gracefully')
  it('respects abort signal')
})

describe('formatMemoryManifest', () => {
  it('includes type tag when present')
  it('omits type tag when undefined')
  it('includes description when present')
  it('omits description when null')
  it('formats ISO timestamp correctly')
})
```
