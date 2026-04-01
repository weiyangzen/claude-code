# fileSuggestions.ts 深度研究文档

## 1. 场景与职责

### 1.1 核心定位
`fileSuggestions.ts` 是 Claude Code 的文件路径智能补全系统的核心引擎，负责：
- 提供基于 `@` 符号触发的文件/目录模糊搜索建议
- 管理文件索引的构建、缓存和增量更新
- 支持 Git 仓库和非 Git 项目的文件发现
- 集成自定义命令钩子（fileSuggestion hook）

### 1.2 使用场景
| 场景 | 描述 |
|------|------|
| 用户输入 `@` | 触发文件建议下拉列表 |
| 用户输入 `@readme` | 模糊匹配文件名包含 "readme" 的文件 |
| 用户输入 `@./src/` | 路径补全，列出 src 目录下的文件 |
| 用户输入 `~` | 支持家目录展开 |
| 大型仓库 | 支持 27万+ 文件的索引和搜索 |

### 1.3 调用方
- `src/hooks/useTypeahead.tsx` - 主要的建议系统集成点
- `src/hooks/unifiedSuggestions.ts` - 统一建议生成器
- `src/components/QuickOpenDialog.tsx` - 快速打开对话框
- `src/commands/clear/caches.ts` - 缓存清理命令

---

## 2. 功能点目的

### 2.1 文件索引管理
- **目的**：为模糊搜索提供高性能的数据结构
- **实现**：使用 Rust 编写的 `FileIndex` 类（通过 `native-ts/file-index` 暴露）
- **特点**：
  - 基于 nucleo 模糊匹配引擎
  - 支持渐进式加载，构建期间可查询部分结果
  - 每 ~4ms 让出事件循环，避免阻塞 UI

### 2.2 双模式文件发现
| 模式 | 适用场景 | 性能特点 |
|------|----------|----------|
| Git 模式 | Git 仓库 | `git ls-files` 直接读取索引，极快 (~50ms) |
| Ripgrep 模式 | 非 Git 项目 | `rg --files` 扫描文件系统 |

### 2.3 智能缓存策略
- **签名机制**：使用 `pathListSignature` 计算路径列表哈希，避免重复构建索引
- **分层缓存**：
  - `cachedTrackedFiles` - 已跟踪文件缓存
  - `cachedConfigFiles` - Claude 配置目录文件缓存
  - `cachedTrackedDirs` - 目录结构缓存
- **背景刷新**：5秒节流 + `.git/index` mtime 检测

### 2.4 忽略模式支持
- 支持 `.ignore` 和 `.rgignore` 文件
- 使用 `ignore` npm 包进行模式匹配
- 缓存按 `repoRoot:cwd` 组合键存储

---

## 3. 具体技术实现

### 3.1 核心数据结构

```typescript
// 文件索引单例（延迟初始化）
let fileIndex: FileIndex | null = null

// 缓存状态
let fileListRefreshPromise: Promise<FileIndex> | null = null
let cacheGeneration = 0
let cachedTrackedFiles: string[] = []
let cachedConfigFiles: string[] = []
let cachedTrackedDirs: string[] = []

// 签名缓存
let loadedTrackedSignature: string | null = null
let loadedMergedSignature: string | null = null

// 忽略模式缓存
let ignorePatternsCache: ReturnType<typeof ignore> | null = null
let ignorePatternsCacheKey: string | null = null
```

### 3.2 路径列表签名算法

```typescript
export function pathListSignature(paths: string[]): string {
  const n = paths.length
  const stride = Math.max(1, Math.floor(n / 500))  // 采样步长
  let h = 0x811c9dc5 | 0  // FNV-1a 初始值
  
  // 采样哈希（约 700 个路径在 27万 文件列表中）
  for (let i = 0; i < n; i += stride) {
    const p = paths[i]!
    for (let j = 0; j < p.length; j++) {
      h = ((h ^ p.charCodeAt(j)) * 0x01000193) | 0
    }
    h = (h * 0x01000193) | 0
  }
  
  // 确保最后一个路径被包含（捕获尾部添加/删除）
  if (n > 0) {
    const last = paths[n - 1]!
    for (let j = 0; j < last.length; j++) {
      h = ((h ^ last.charCodeAt(j)) * 0x01000193) | 0
    }
  }
  
  return `${n}:${(h >>> 0).toString(16)}`
}
```

**设计考量**：
- 27万 文件列表哈希约 700 个路径，而非 14MB 全部数据
- 采样策略在性能和准确性之间取得平衡
- 5秒刷新兜底捕获采样遗漏的重命名

### 3.3 Git 文件获取流程

```typescript
async function getFilesUsingGit(
  abortSignal: AbortSignal,
  respectGitignore: boolean,
): Promise<string[] | null> {
  // 1. 查找 Git 根目录
  const repoRoot = findGitRoot(getCwd())
  if (!repoRoot) return null

  // 2. 获取已跟踪文件（快路径）
  const trackedResult = await execFileNoThrowWithCwd(
    gitExe(),
    ['-c', 'core.quotepath=false', 'ls-files', '--recurse-submodules'],
    { timeout: 5000, abortSignal, cwd: repoRoot },
  )

  // 3. 应用 .ignore/.rgignore 模式
  const ignorePatterns = await loadRipgrepIgnorePatterns(repoRoot, cwd)
  if (ignorePatterns) {
    normalizedTracked = ignorePatterns.filter(normalizedTracked)
  }

  // 4. 后台获取未跟踪文件
  if (!untrackedFetchPromise) {
    untrackedFetchPromise = execFileNoThrowWithCwd(
      gitExe(), 
      ['-c', 'core.quotepath=false', 'ls-files', '--others', '--exclude-standard'],
      { timeout: 10000, cwd: repoRoot }
    ).then(async untrackedResult => {
      // 合并未跟踪文件到缓存
      void mergeUntrackedIntoNormalizedCache(normalizedUntracked)
    })
  }

  return normalizedTracked
}
```

### 3.4 背景缓存刷新机制

```typescript
const REFRESH_THROTTLE_MS = 5_000

export function startBackgroundCacheRefresh(): void {
  if (fileListRefreshPromise) return

  const indexMtime = getGitIndexMtime()
  
  // 节流逻辑：已有缓存时检查 Git 状态变化
  if (fileIndex) {
    const gitStateChanged = indexMtime !== null && indexMtime !== lastGitIndexMtime
    if (!gitStateChanged && Date.now() - lastRefreshMs < REFRESH_THROTTLE_MS) {
      return  // 跳过刷新
    }
  }

  // 启动刷新
  fileListRefreshPromise = getPathsForSuggestions()
    .then(result => {
      fileListRefreshPromise = null
      indexBuildComplete.emit()  // 通知 UI 重新搜索
      lastGitIndexMtime = indexMtime
      lastRefreshMs = Date.now()
    })
}
```

### 3.5 目录名提取优化

```typescript
export function getDirectoryNames(files: string[]): string[] {
  const directoryNames = new Set<string>()
  collectDirectoryNames(files, 0, files.length, directoryNames)
  return [...directoryNames].map(d => d + path.sep)
}

function collectDirectoryNames(
  files: string[],
  start: number,
  end: number,
  out: Set<string>,
): void {
  for (let i = start; i < end; i++) {
    let currentDir = path.dirname(files[i]!)
    // 早期退出：已处理过此目录及其所有父目录
    while (currentDir !== '.' && !out.has(currentDir)) {
      const parent = path.dirname(currentDir)
      if (parent === currentDir) break  // 到达根目录
      out.add(currentDir)
      currentDir = parent
    }
  }
}
```

**优化点**：
- 避免使用 `path.parse()`（每次分配 5 字段对象）
- 使用 `while` 循环和 `Set.has()` 进行早期退出
- 异步版本每 ~10k 文件让出事件循环

### 3.6 建议生成流程

```typescript
export async function generateFileSuggestions(
  partialPath: string,
  showOnEmpty = false,
): Promise<SuggestionItem[]> {
  // 1. 检查自定义命令钩子
  if (getInitialSettings().fileSuggestion?.type === 'command') {
    const input: FileSuggestionCommandInput = {
      ...createBaseHookInput(),
      query: partialPath,
    }
    const results = await executeFileSuggestionCommand(input)
    return results.slice(0, MAX_SUGGESTIONS).map(createFileSuggestionItem)
  }

  // 2. 空路径返回顶层目录内容
  if (partialPath === '' || partialPath === '.' || partialPath === './') {
    const topLevelPaths = await getTopLevelPaths()
    startBackgroundCacheRefresh()
    return topLevelPaths.slice(0, MAX_SUGGESTIONS).map(createFileSuggestionItem)
  }

  // 3. 启动背景刷新
  const wasBuilding = fileListRefreshPromise !== null
  startBackgroundCacheRefresh()

  // 4. 处理路径前缀（./ 和 ~）
  let normalizedPath = partialPath
  if (partialPath.startsWith('.' + path.sep)) {
    normalizedPath = partialPath.substring(2)
  }
  if (normalizedPath.startsWith('~')) {
    normalizedPath = expandPath(normalizedPath)
  }

  // 5. 执行模糊搜索
  const matches = fileIndex
    ? findMatchingFiles(fileIndex, normalizedPath)
    : []

  return matches
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 核心文件依赖图

```
fileSuggestions.ts
├── native-ts/file-index/index.js  (Rust FileIndex)
├── utils/hooks.ts                 (executeFileSuggestionCommand)
├── utils/markdownConfigLoader.ts  (CLAUDE_CONFIG_DIRECTORIES)
├── utils/ripgrep.ts               (ripGrep fallback)
├── utils/git.ts                   (findGitRoot, gitExe)
├── utils/config.ts                (getGlobalConfig)
├── utils/settings/settings.ts     (getInitialSettings)
├── utils/signal.ts                (createSignal)
├── services/analytics/index.ts    (logEvent)
└── ignore (npm package)
```

### 4.2 关键函数调用链

**冷启动路径**：
```
generateFileSuggestions()
  └── startBackgroundCacheRefresh()
      └── getPathsForSuggestions()
          ├── getProjectFiles()           // Git 或 ripgrep
          │   ├── getFilesUsingGit()      // 优先尝试
          │   └── ripGrep()               // 回退
          ├── getClaudeConfigFiles()      // 配置目录
          ├── getDirectoryNamesAsync()    // 目录提取
          └── fileIndex.loadFromFileListAsync()  // 构建索引
```

**搜索路径**：
```
generateFileSuggestions("@readme")
  └── findMatchingFiles(fileIndex, "readme")
      └── fileIndex.search("readme", MAX_SUGGESTIONS)  // Rust nucleo
```

### 4.3 信号系统

```typescript
const indexBuildComplete = createSignal()
export const onIndexBuildComplete = indexBuildComplete.subscribe

// 使用方（useTypeahead.tsx）
useEffect(() => {
  startBackgroundCacheRefresh()
  return onIndexBuildComplete(() => {
    // 索引构建完成，重新触发搜索以升级部分结果
    const token = latestSearchTokenRef.current
    if (token !== null) {
      void fetchFileSuggestions(token, token === '')
    }
  })
}, [fetchFileSuggestions])
```

---

## 5. 依赖与外部交互

### 5.1 外部依赖

| 依赖 | 用途 | 类型 |
|------|------|------|
| `ignore` | .gitignore 模式匹配 | npm 包 |
| `fs` (statSync) | .git/index mtime 检测 | Node.js 内置 |
| `native-ts/file-index` | Rust 模糊匹配引擎 | 原生模块 |

### 5.2 环境变量

| 变量 | 影响 |
|------|------|
| `CLAUDE_CODE_RESPECT_GITIGNORE` | 控制是否尊重 .gitignore |
| `NODE_ENV=test` | 跳过背景缓存刷新（防止测试泄漏） |

### 5.3 Hook 集成

```typescript
// settings.json 配置示例
{
  "fileSuggestion": {
    "type": "command",
    "command": "fzf --filter"
  }
}
```

当配置自定义命令时，完全绕过内置的文件索引系统。

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| 大型仓库内存压力 | 27万+ 文件列表占用内存 | 签名采样减少比较开销 |
| Git 工作树检测 | `.git` 是文件而非目录（worktree） | `statSync` 错误处理，回退到时间节流 |
| 并发刷新 | 快速按键可能触发多次刷新 | `fileListRefreshPromise` 去重 |
| 缓存失效 | 签名碰撞导致遗漏更新 | 5秒刷新兜底 |

### 6.2 边界条件

1. **空仓库**：`git ls-files` 返回空列表，索引正常构建
2. **非 Git 项目**：自动回退到 ripgrep
3. **删除的 CWD**：使用 `getOriginalCwd()` 回退
4. **超长路径**：FNV-1a 哈希自然处理

### 6.3 改进建议

1. **Watchman 集成**：对于超大型仓库，使用 Watchman 替代轮询
2. **增量更新**：仅更新变更的文件而非重建整个索引
3. **持久化缓存**：将会话间的索引持久化到磁盘
4. **并行 Git 操作**：`git ls-files` 和 `--others` 可并行执行
5. **内存优化**：使用 Trie 结构替代字符串数组存储路径

### 6.4 性能基准

基于代码注释和实现分析：

| 操作 | 预期耗时 | 备注 |
|------|----------|------|
| Git ls-files (27万文件) | ~50ms | 直接读取索引 |
| 索引构建 | ~120ms | 含 4ms 事件循环让出 |
| 模糊搜索 | ~8-15ms | nucleo 引擎 |
| 签名计算 | <1ms | 采样 700 个路径 |

### 6.5 测试注意事项

- 测试环境 (`NODE_ENV=test`) 跳过背景刷新，避免异步泄漏
- 需要模拟 `native-ts/file-index` 模块进行单元测试
- Git 相关测试需要准备临时仓库
