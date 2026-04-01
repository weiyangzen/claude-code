# 研究文档：src/native-ts/file-index/index.ts

> 研究范围：代码、脚本、配置、测试及必要实现上下文  
> 目标对象：`src/native-ts/file-index/index.ts`  
> 文档日期：2026-04-01  
> 执行器：kimi (k2p5)

---

## 1. 场景与职责

### 1.1 模块定位

`src/native-ts/file-index/index.ts` 是 `vendor/file-index-src`（基于 Rust + NAPI 的 nucleo 模糊搜索模块）的**纯 TypeScript 移植实现**。它属于 `src/native-ts` 三大零外部运行时依赖的算法模块之一：

| 模块 | 职责 | 替代的原生模块 |
|------|------|---------------|
| `file-index` | 高性能模糊文件搜索索引 | `vendor/file-index-src` (Rust nucleo) |
| `color-diff` | 语法高亮与 diff 渲染 | `vendor/color-diff-src` (Rust syntect+bat) |
| `yoga-layout` | Flexbox 布局引擎 | `yoga-layout` (C++ WASM) |

### 1.2 核心职责

该模块为 Claude Code 提供**高性能的本地文件路径模糊搜索能力**：

1. **索引构建**：接收并去重大量文件路径（典型场景 27 万+ 条），构建可搜索的内部数据结构
2. **模糊搜索**：在毫秒级时间内响应用户的模糊查询（如 `@src/hooks` 或 Quick Open 的 `ctrl+shift+p`）
3. **智能评分**：以 nucleo/fzf-v2 风格的评分规则对结果排序，使最相关文件排在最前
4. **非阻塞构建**：通过时间片分块（chunked yield）避免大索引构建阻塞主线程事件循环
5. **渐进查询**：异步构建过程中，已处理的前缀即可被搜索，实现"边建边搜"

### 1.3 设计目标

- **零原生依赖**：移除 Rust NAPI 依赖，实现跨平台（Windows CI、WSL、嵌入式 bun 构建）的纯 JS/TS 可移植性
- **API 兼容**：保持与旧 Rust 模块一致的 API 和评分行为
- **性能等价**：在典型工作负载下达到与 nucleo 相当的搜索性能

---

## 2. 功能点目的

| 功能点 | 目的 | 对应 API |
|--------|------|----------|
| **索引构建（同步）** | 将文件路径列表转换为可搜索的内部结构 | `loadFromFileList(fileList: string[]): void` |
| **索引构建（异步）** | 非阻塞式构建，支持渐进查询 | `loadFromFileListAsync(fileList: string[]): { queryable: Promise<void>, done: Promise<void> }` |
| **模糊搜索** | 根据用户输入返回前 N 个最匹配的文件路径 | `search(query: string, limit: number): SearchResult[]` |
| **空查询兜底** | 用户未输入时返回顶层目录/文件片段 | `computeTopLevelEntries()` (内部) |
| **事件循环让出** | 支持异步分块构建的工具函数 | `yieldToEventLoop(): Promise<void>` |

---

## 3. 具体技术实现

### 3.1 关键数据结构

```typescript
// 搜索结果类型
export type SearchResult = {
  path: string   // 文件路径
  score: number  // 归一化分数 (0.0 = 最佳, 1.0 = 最差)
}

// FileIndex 类核心字段
export class FileIndex {
  private paths: string[] = []                          // 原始路径（保留大小写）
  private lowerPaths: string[] = []                     // 小写路径（大小写不敏感搜索用）
  private charBits: Int32Array = new Int32Array(0)      // 每路径的 a-z 位图（26位）
  private pathLens: Uint16Array = new Uint16Array(0)    // 每路径长度
  private topLevelCache: SearchResult[] | null = null   // 空查询缓存
  private readyCount = 0                                // 异步构建中已就绪的路径数
}

// 全局复用缓冲区，避免搜索时重复分配
const posBuf = new Int32Array(MAX_QUERY_LEN)  // MAX_QUERY_LEN = 64
```

#### charBits 位图机制

- **26 位位图**：每个 bit 代表路径中是否出现对应小写字母（`a=bit0`, `b=bit1`, ..., `z=bit25`）
- **构建过程**：遍历路径的小写形式，对每个 `a-z` 字符设置对应位
- **搜索优化**：若 `(charBits[i] & needleBitmap) !== needleBitmap`，可立即排除该路径（O(1) 剪枝）
- **命中率**：对宽泛查询（如 `"test"`）约 89% 路径通过；对罕见字符（如 `"xyz"`）可排除 90%+ 路径

### 3.2 搜索算法流程（`search` 方法）

```typescript
search(query: string, limit: number): SearchResult[]
```

#### 阶段 1: 预处理

1. **Smart Case 判定**：
   - 查询全小写 → 大小写不敏感（使用 `lowerPaths`）
   - 查询含大写 → 大小写敏感（使用原始 `paths`）

2. **Needle 位图构建**：
   ```typescript
   let needleBitmap = 0
   for (let j = 0; j < nLen; j++) {
     const cc = needleChars[j]!.charCodeAt(0)
     if (cc >= 97 && cc <= 122) needleBitmap |= 1 << (cc - 97)
   }
   ```

3. **分数上限计算**：
   ```typescript
   const scoreCeiling = nLen * (SCORE_MATCH + BONUS_BOUNDARY) + BONUS_FIRST_CHAR + 32
   ```

#### 阶段 2: 线性扫描 + 多级剪枝

```typescript
outer: for (let i = 0; i < readyCount; i++) {
  // 1. O(1) 位图剪枝
  if ((charBits[i]! & needleBitmap) !== needleBitmap) continue
  
  // 2. Fused indexOf 扫描（SIMD 加速）
  let pos = haystack.indexOf(needleChars[0]!)
  if (pos === -1) continue
  posBuf[0] = pos
  
  let gapPenalty = 0
  let consecBonus = 0
  let prev = pos
  
  for (let j = 1; j < nLen; j++) {
    pos = haystack.indexOf(needleChars[j]!, prev + 1)
    if (pos === -1) continue outer
    posBuf[j] = pos
    const gap = pos - prev - 1
    if (gap === 0) consecBonus += BONUS_CONSECUTIVE
    else gapPenalty += PENALTY_GAP_START + gap * PENALTY_GAP_EXTENSION
    prev = pos
  }
  
  // 3. Gap-bound 剪枝
  if (topK.length === limit && scoreCeiling + consecBonus - gapPenalty <= threshold) {
    continue
  }
  
  // 4. 边界/驼峰评分
  let score = nLen * SCORE_MATCH + consecBonus - gapPenalty
  score += scoreBonusAt(path, posBuf[0]!, true)   // 首字符特殊处理
  for (let j = 1; j < nLen; j++) {
    score += scoreBonusAt(path, posBuf[j]!, false)
  }
  score += Math.max(0, 32 - (hLen >> 2))  // 长度归一化
  
  // 5. Top-K 维护...
}
```

#### 阶段 3: Top-K 维护

- 维护升序数组 `topK`（大小不超过 `limit`）
- 未满时直接追加；满后使用二分插入 + `shift()` 更新
- 均摊复杂度：O(log limit) 每命中项

#### 阶段 4: 最终分数映射

```typescript
const matchCount = topK.length
const denom = Math.max(matchCount, 1)
for (let i = 0; i < matchCount; i++) {
  const path = topK[i]!.path
  const positionScore = i / denom                    // 0.0 = 最佳
  const finalScore = path.includes('test')
    ? Math.min(positionScore * 1.05, 1.0)           // test 文件轻微降权
    : positionScore
  results[i] = { path, score: finalScore }
}
```

### 3.3 评分系统详解

#### 评分常量（nucleo/fzf-v2 近似）

| 常量 | 值 | 含义 |
|------|-----|------|
| `SCORE_MATCH` | 16 | 每个匹配字符基础分 |
| `BONUS_BOUNDARY` | 8 | 边界匹配奖励（`/`、`\`、`-`、`_`、`.`、` ` 后） |
| `BONUS_CAMEL` | 6 | 驼峰边界奖励（小写后接大写） |
| `BONUS_CONSECUTIVE` | 4 | 连续字符奖励 |
| `BONUS_FIRST_CHAR` | 8 | 首字符匹配奖励 |
| `PENALTY_GAP_START` | 3 | 间隔起始惩罚 |
| `PENALTY_GAP_EXTENSION` | 1 | 间隔每字符额外惩罚 |

#### 边界检测函数

```typescript
function isBoundary(code: number): boolean {
  return (
    code === 47 ||  // /
    code === 92 ||  // \
    code === 45 ||  // -
    code === 95 ||  // _
    code === 46 ||  // .
    code === 32     // space
  )
}

function scoreBonusAt(path: string, pos: number, first: boolean): number {
  if (pos === 0) return first ? BONUS_FIRST_CHAR : 0
  const prevCh = path.charCodeAt(pos - 1)
  if (isBoundary(prevCh)) return BONUS_BOUNDARY
  if (isLower(prevCh) && isUpper(path.charCodeAt(pos))) return BONUS_CAMEL
  return 0
}
```

### 3.4 异步分块构建

#### 时间片常量

```typescript
const CHUNK_MS = 4  // 每 4ms 让出事件循环
```

#### 两阶段分块流程

```typescript
private async buildAsync(fileList: string[], markQueryable: () => void): Promise<void> {
  // 阶段 1: 去重
  let chunkStart = performance.now()
  for (let i = 0; i < fileList.length; i++) {
    // 去重逻辑...
    if ((i & 0xff) === 0xff && performance.now() - chunkStart > CHUNK_MS) {
      await yieldToEventLoop()
      chunkStart = performance.now()
    }
  }
  
  // 阶段 2: 索引构建
  this.resetArrays(paths)
  chunkStart = performance.now()
  let firstChunk = true
  for (let i = 0; i < paths.length; i++) {
    this.indexPath(i)
    if ((i & 0xff) === 0xff && performance.now() - chunkStart > CHUNK_MS) {
      this.readyCount = i + 1
      if (firstChunk) {
        markQueryable()  // 第一个分块完成即可查询
        firstChunk = false
      }
      await yieldToEventLoop()
      chunkStart = performance.now()
    }
  }
  this.readyCount = paths.length
  markQueryable()
}
```

#### 渐进查询机制

- `readyCount` 字段跟踪已就绪的路径数
- `search()` 方法只扫描 `0` 到 `readyCount` 的范围
- 第一个分块完成后外部即可发起搜索，获得部分结果
- 构建完成后触发 `indexBuildComplete` 信号，UI 可重搜升级结果

### 3.5 空查询处理

```typescript
function computeTopLevelEntries(paths: string[], limit: number): SearchResult[] {
  const topLevel = new Set<string>()
  for (const p of paths) {
    // 提取第一级路径段（/ 或 \ 分隔）
    let end = p.length
    for (let i = 0; i < p.length; i++) {
      const c = p.charCodeAt(i)
      if (c === 47 || c === 92) { end = i; break }
    }
    const segment = p.slice(0, end)
    if (segment.length > 0) {
      topLevel.add(segment)
      if (topLevel.size >= limit) break
    }
  }
  // 按 (长度升序, 字母升序) 排序
  const sorted = Array.from(topLevel)
  sorted.sort((a, b) => {
    const lenDiff = a.length - b.length
    if (lenDiff !== 0) return lenDiff
    return a < b ? -1 : a > b ? 1 : 0
  })
  return sorted.slice(0, limit).map(path => ({ path, score: 0.0 }))
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 模块自身

| 文件 | 行数 | 说明 |
|------|------|------|
| `src/native-ts/file-index/index.ts` | 370 | 唯一实现文件，导出 `FileIndex`、`SearchResult`、`yieldToEventLoop`、`CHUNK_MS` |

### 4.2 导出 API

```typescript
// 类型
export type SearchResult = { path: string; score: number }

// 类
export class FileIndex {
  loadFromFileList(fileList: string[]): void
  loadFromFileListAsync(fileList: string[]): { queryable: Promise<void>; done: Promise<void> }
  search(query: string, limit: number): SearchResult[]
}

// 工具函数
export function yieldToEventLoop(): Promise<void>
export const CHUNK_MS: 4

// 默认导出
export default FileIndex
export type { FileIndex as FileIndexType }
```

### 4.3 上游调用方（谁使用 file-index）

```
src/native-ts/file-index/index.ts
    │
    ├──► src/hooks/fileSuggestions.ts (主要调用方)
    │       ├── getFileIndex() → new FileIndex()  // 单例模式
    │       ├── getPathsForSuggestions() → index.loadFromFileListAsync()
    │       ├── findMatchingFiles() → fileIndex.search()
    │       ├── mergeUntrackedIntoNormalizedCache() → fileIndex.loadFromFileListAsync()
    │       └── getDirectoryNamesAsync() → 复用 CHUNK_MS / yieldToEventLoop
    │
    ├──► src/hooks/useTypeahead.tsx
    │       └── onIndexBuildComplete (signal) → 索引完成后重搜以升级 partial → full
    │
    ├──► src/components/QuickOpenDialog.tsx
    │       └── generateFileSuggestions(q, true) → Quick Open 弹窗
    │
    ├──► src/hooks/unifiedSuggestions.ts
    │       └── generateFileSuggestions() → 与 agent/MCP 建议统一排序
    │
    └──► src/commands/clear/caches.ts
            └── clearFileSuggestionCaches() → 重置 fileIndex 单例与所有签名缓存
```

### 4.4 fileSuggestions.ts 详细交互

```typescript
// src/hooks/fileSuggestions.ts

// 1. 单例模式管理
let fileIndex: FileIndex | null = null
function getFileIndex(): FileIndex {
  if (!fileIndex) fileIndex = new FileIndex()
  return fileIndex
}

// 2. 路径列表签名缓存（避免重复构建）
let loadedTrackedSignature: string | null = null
let loadedMergedSignature: string | null = null

// 3. 异步构建完成信号
const indexBuildComplete = createSignal()
export const onIndexBuildComplete = indexBuildComplete.subscribe

// 4. 搜索包装函数
const MAX_SUGGESTIONS = 15
function findMatchingFiles(fileIndex: FileIndex, partialPath: string): SuggestionItem[] {
  const results = fileIndex.search(partialPath, MAX_SUGGESTIONS)
  return results.map(result => createFileSuggestionItem(result.path, result.score))
}

// 5. 缓存刷新（带节流）
export function startBackgroundCacheRefresh(): void {
  // 通过 .git/index mtime 检测变更
  // 5s 节流防止频繁刷新
}
```

---

## 5. 依赖与外部交互

### 5.1 运行时依赖

`src/native-ts/file-index/index.ts` **零外部运行时依赖**，仅使用：

- TypeScript 内置类型
- 引擎原生 API：`performance.now()`、`setImmediate`
- 标准 TypedArray：`Int32Array`、`Uint16Array`

### 5.2 与 fileSuggestions 的交互契约

| 契约项 | 说明 |
|--------|------|
| **去重** | `loadFromFileList` 内部使用 `Set<string>` 去重并过滤空字符串 |
| **渐进查询** | `readyCount` 字段使外部可在异步构建期间获得部分结果 |
| **缓存签名** | `fileSuggestions.ts` 维护 `loadedTrackedSignature` 和 `loadedMergedSignature`，避免相同路径列表重复重建索引 |
| **事件通知** | `indexBuildComplete` Signal 在索引构建完成后触发，`useTypeahead.tsx` 订阅该信号以刷新建议列表 |
| **节流控制** | `CHUNK_MS` 导出供 `fileSuggestions.ts` 的 `getDirectoryNamesAsync` 复用 |

### 5.3 与 Quick Open / Typeahead 的交互

- **Quick Open** (`ctrl+shift+p`)：直接调用 `generateFileSuggestions(query, showOnEmpty=true)`，结果经 `highlightMatch` 高亮后展示，并异步读取选中文件前 20 行作为预览
- **Typeahead @ 提及**：`useTypeahead.tsx` 在检测到 `@` 或路径字符时，通过 `generateUnifiedSuggestions()` 间接调用 `generateFileSuggestions()`，50ms debounce 后更新下拉建议

---

## 6. 风险、边界与改进建议

### 6.1 已知边界与限制

| 边界 | 影响 | 代码位置 |
|------|------|----------|
| **最大查询长度 64** | 超长查询被静默截断，可能导致匹配行为与预期不符 | `MAX_QUERY_LEN = 64` |
| **位图仅覆盖 a-z** | 非 ASCII 字符（中文、带重音符号、数字）无法通过位图预过滤排除 | `charBits` 构建逻辑（行 161-166） |
| **无增量更新** | 文件增删改需重建整个索引 | `buildIndex()` 全量重建 |
| **主线程执行** | 搜索算法在主线程运行；极端大仓库 + 宽泛查询可能产生 10ms+ 阻塞 | `search()` 同步实现 |
| **Test 惩罚硬编码** | `"test"` 子串惩罚是写死的，无法配置 | `path.includes('test')`（行 283-285） |
| **无单元测试覆盖** | 仓库中未找到直接测试 `file-index` 的测试文件 | 全局搜索无 `.test.ts` |

### 6.2 潜在风险

1. **readyCount 竞态**：`buildAsync` 在 `await yieldToEventLoop()` 之前更新 `readyCount`，但 `search()` 与构建循环共享同一线程，实际竞态窗口极小；若未来迁移至 Worker，需显式同步

2. **签名哈希碰撞**：`pathListSignature()` 使用 FNV-1a 变体并采样每 500 个路径，中间文件重命名可能恰好落在采样间隙，导致缓存未失效（5s 刷新 floor 最终会覆盖）

3. **posBuf 全局复用**：`posBuf` 是模块级全局变量，若 `search()` 被并发调用（如 Worker 场景）会导致数据竞争；当前单线程环境安全

4. **topLevelCache 固定限制**：`TOP_LEVEL_CACHE_LIMIT = 100`，超大仓库的顶层目录可能被截断

### 6.3 改进建议

| 优先级 | 建议 | 预期收益 |
|--------|------|----------|
| **高** | 添加 `file-index` 核心算法的单元测试（位图过滤、评分边界、top-K 排序、空查询、smart case） | 防止回归，支持未来重构 |
| **高** | 将 `search()` 迁移至 Web Worker 或 Node Worker Threads | 彻底消除大索引搜索对主线程的阻塞 |
| **中** | 扩展位图至数字或常见非 ASCII 字符范围，或引入二级过滤器 | 提升含数字/中文路径的搜索效率 |
| **中** | 支持增量索引更新（`addPaths` / `removePaths` API） | 避免文件小变更时的全量重建 |
| **低** | 将 `"test"` 惩罚规则外化为配置项或启发式策略 | 提升不同项目（如测试框架仓库）的搜索体验 |
| **低** | 对 `computeTopLevelEntries` 的结果也应用模糊评分，而非仅按长度+字母序 | 空查询时的建议相关性更高 |

---

## 7. 附录

### 7.1 文件建议完整调用链

```
用户输入 @foo
    │
    ▼
src/hooks/useTypeahead.tsx
    ├── startBackgroundCacheRefresh() ──► src/hooks/fileSuggestions.ts
    │       ├── getFilesUsingGit() ──► git ls-files (优先)
    │       ├── getProjectFiles() ──► ripGrep() (回退)
    │       ├── getClaudeConfigFiles() ──► .claude/ markdown
    │       └── getPathsForSuggestions()
    │               └── fileIndex.loadFromFileListAsync(allPaths)
    │
    ├── generateUnifiedSuggestions() ──► src/hooks/unifiedSuggestions.ts
    │       └── generateFileSuggestions("foo")
    │               └── findMatchingFiles(fileIndex, "foo")
    │                       └── fileIndex.search("foo", 15)
    │
    └── onIndexBuildComplete → 重搜以升级 partial → full
```

### 7.2 Quick Open 调用链

```
Ctrl+Shift+P
    │
    ▼
src/components/QuickOpenDialog.tsx
    └── generateFileSuggestions(query, true)
            └── src/hooks/fileSuggestions.ts → FileIndex.search()
```

### 7.3 核心算法复杂度

| 操作 | 时间复杂度 | 空间复杂度 | 说明 |
|------|-----------|-----------|------|
| `loadFromFileList` | O(N × L) | O(N) | N=路径数, L=平均路径长度 |
| `search` | O(N × (L + log K)) | O(K) | K=limit, 最坏情况全扫描 |
| 位图剪枝 | O(1) 每路径 | - | 排除不匹配路径 |
| Top-K 维护 | O(log K) 每命中 | O(K) | 二分插入 |

### 7.4 相关文件索引

| 文件路径 | 关系 | 用途 |
|----------|------|------|
| `src/native-ts/file-index/index.ts` | 目标文件 | 模糊搜索索引实现 |
| `src/hooks/fileSuggestions.ts` | 主要调用方 | 文件建议逻辑封装 |
| `src/hooks/useTypeahead.tsx` | 调用方 | 输入框类型提示 |
| `src/components/QuickOpenDialog.tsx` | 调用方 | Quick Open 弹窗 |
| `src/hooks/unifiedSuggestions.ts` | 调用方 | 统一建议排序 |
| `src/commands/clear/caches.ts` | 调用方 | 缓存清理 |
| `src/native-ts/color-diff/index.ts` | 同级模块 | 语法高亮 |
| `src/native-ts/yoga-layout/index.ts` | 同级模块 | Flexbox 布局 |
| `src/ink/layout/yoga.ts` | yoga 调用方 | Ink 布局适配 |
