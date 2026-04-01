# gitDiff.ts 深度研究文档

## 场景与职责

`gitDiff.ts` 提供 Git 差异（diff）分析和解析功能，用于：

1. **工作区变更统计**：获取文件变更数量、增删行数统计
2. **差异分块解析**：解析统一差异格式（unified diff）为结构化数据
3. **单文件差异**：获取单个文件的 PR 风格差异
4. **性能优化**：支持快速探测和按需加载，避免大数据集时的性能问题

该模块是差异显示（DiffDialog）和工具使用报告的数据源。

## 功能点目的

### 1. 差异统计获取
- `fetchGitDiff`: 获取工作区与 HEAD 的差异统计
- `parseGitNumstat`: 解析 `git diff --numstat` 输出
- `parseShortstat`: 解析 `git diff --shortstat` 输出

### 2. 差异分块解析
- `fetchGitDiffHunks`: 获取结构化差异分块
- `parseGitDiff`: 解析统一差异格式为 `StructuredPatchHunk[]`

### 3. 单文件差异
- `fetchSingleFileGitDiff`: 获取单个文件的 PR 风格差异（基于 merge-base）
- `generateSyntheticDiff`: 为未跟踪文件生成合成差异

### 4. 暂态状态检测
- `isInTransientGitState`: 检测合并/变基/拣选/还原状态

## 具体技术实现

### 关键数据结构

```typescript
export type GitDiffStats = {
  filesCount: number   // 变更文件数
  linesAdded: number   // 新增行数
  linesRemoved: number // 删除行数
}

export type PerFileStats = {
  added: number
  removed: number
  isBinary: boolean
  isUntracked?: boolean  // 未跟踪文件标记
}

export type GitDiffResult = {
  stats: GitDiffStats
  perFileStats: Map<string, PerFileStats>
  hunks: Map<string, StructuredPatchHunk[]>  // 来自 'diff' 库
}

export type ToolUseDiff = {
  filename: string
  status: 'modified' | 'added'
  additions: number
  deletions: number
  changes: number
  patch: string
  repository: string | null  // GitHub owner/repo
}
```

### 关键流程

#### 差异获取流程（性能优化）
```typescript
export async function fetchGitDiff(): Promise<GitDiffResult | null> {
  // 1. 检查 Git 状态和暂态状态
  const isGit = await getIsGit()
  if (!isGit || await isInTransientGitState()) return null

  // 2. 快速探测：使用 --shortstat 获取总计
  const { stdout: shortstatOut, code: shortstatCode } = await execFileNoThrow(
    gitExe(),
    ['--no-optional-locks', 'diff', 'HEAD', '--shortstat'],
    { timeout: GIT_TIMEOUT_MS },
  )

  if (shortstatCode === 0) {
    const quickStats = parseShortstat(shortstatOut)
    if (quickStats && quickStats.filesCount > MAX_FILES_FOR_DETAILS) {
      // 文件过多，跳过详细统计
      return { stats: quickStats, perFileStats: new Map(), hunks: new Map() }
    }
  }

  // 3. 获取详细统计
  const { stdout: numstatOut, code: numstatCode } = await execFileNoThrow(
    gitExe(),
    ['--no-optional-locks', 'diff', 'HEAD', '--numstat'],
    { timeout: GIT_TIMEOUT_MS },
  )

  // 4. 添加未跟踪文件（仅文件名）
  const untrackedStats = await fetchUntrackedFiles(remainingSlots)

  return { stats, perFileStats, hunks: new Map() }  // hunks 按需获取
}
```

#### 差异分块解析
```typescript
export function parseGitDiff(
  stdout: string,
): Map<string, StructuredPatchHunk[]> {
  const result = new Map<string, StructuredPatchHunk[]>()
  const fileDiffs = stdout.split(/^diff --git /m).filter(Boolean)

  for (const fileDiff of fileDiffs) {
    if (result.size >= MAX_FILES) break  // 文件数限制
    if (fileDiff.length > MAX_DIFF_SIZE_BYTES) continue  // 大小限制

    const lines = fileDiff.split('\n')
    // 解析文件名："a/path b/path"
    const headerMatch = lines[0]?.match(/^a\/(.+?) b\/(.+)$/)
    if (!headerMatch) continue
    const filePath = headerMatch[2] ?? headerMatch[1] ?? ''

    // 解析分块头：@@ -oldStart,oldLines +newStart,newLines @@
    const fileHunks: StructuredPatchHunk[] = []
    let currentHunk: StructuredPatchHunk | null = null
    let lineCount = 0

    for (let i = 1; i < lines.length; i++) {
      const line = lines[i] ?? ''

      const hunkMatch = line.match(/^@@ -(\d+)(?:,(\d+))? \+(\d+)(?:,(\d+))? @@/)
      if (hunkMatch) {
        // 开始新分块
        currentHunk = {
          oldStart: parseInt(hunkMatch[1] ?? '0', 10),
          oldLines: parseInt(hunkMatch[2] ?? '1', 10),
          newStart: parseInt(hunkMatch[3] ?? '0', 10),
          newLines: parseInt(hunkMatch[4] ?? '1', 10),
          lines: [],
        }
        continue
      }

      // 跳过元数据行
      if (line.startsWith('index ') || line.startsWith('---') || ...) continue

      // 添加差异行（带行数限制）
      if (currentHunk && lineCount < MAX_LINES_PER_FILE) {
        currentHunk.lines.push('' + line)  // 强制扁平字符串，避免切片引用
        lineCount++
      }
    }

    if (fileHunks.length > 0) {
      result.set(filePath, fileHunks)
    }
  }

  return result
}
```

#### 单文件 PR 风格差异
```typescript
export async function fetchSingleFileGitDiff(
  absoluteFilePath: string,
): Promise<ToolUseDiff | null> {
  const gitRoot = findGitRoot(dirname(absoluteFilePath))
  if (!gitRoot) return null

  const gitPath = relative(gitRoot, absoluteFilePath).split(sep).join('/')

  // 检查文件是否被跟踪
  const { code: lsFilesCode } = await execFileNoThrowWithCwd(
    gitExe(),
    ['--no-optional-locks', 'ls-files', '--error-unmatch', gitPath],
    { cwd: gitRoot },
  )

  if (lsFilesCode === 0) {
    // 已跟踪文件：与 merge-base 比较
    const diffRef = await getDiffRef(gitRoot)  // 优先 CLAUDE_CODE_BASE_REF
    const { stdout } = await execFileNoThrowWithCwd(
      gitExe(),
      ['--no-optional-locks', 'diff', diffRef, '--', gitPath],
      { cwd: gitRoot },
    )
    return { ...parseRawDiffToToolUseDiff(gitPath, stdout, 'modified'), repository }
  }

  // 未跟踪文件：生成合成差异
  const syntheticDiff = await generateSyntheticDiff(gitPath, absoluteFilePath)
  return { ...syntheticDiff, repository }
}
```

### 性能优化策略

1. **快速探测**：先使用 `--shortstat` 检查文件数量，避免大数据集时的内存问题
2. **分页加载**：`hunks` 在 `fetchGitDiff` 中返回空 Map，通过 `fetchGitDiffHunks` 按需加载
3. **文件限制**：`MAX_FILES` (50)、`MAX_FILES_FOR_DETAILS` (500)、`MAX_DIFF_SIZE_BYTES` (1MB)
4. **行数限制**：`MAX_LINES_PER_FILE` (400，GitHub 的自动加载限制)
5. **内存优化**：使用 `'' + line` 强制创建扁平字符串，避免 V8 切片字符串引用大内存

### 暂态状态检测
```typescript
async function isInTransientGitState(): Promise<boolean> {
  const gitDir = await getGitDir(getCwd())
  if (!gitDir) return false

  const transientFiles = [
    'MERGE_HEAD',      // 合并中
    'REBASE_HEAD',     // 变基中
    'CHERRY_PICK_HEAD', // 拣选中
    'REVERT_HEAD',     // 还原中
  ]

  const results = await Promise.all(
    transientFiles.map(file =>
      access(join(gitDir, file))
        .then(() => true)
        .catch(() => false),
    ),
  )
  return results.some(Boolean)
}
```

## 关键代码路径与文件引用

### 核心导出
- `fetchGitDiff()` - 获取差异统计
- `fetchGitDiffHunks()` - 获取差异分块
- `parseGitNumstat()` / `parseShortstat()` - 解析统计输出
- `parseGitDiff()` - 解析差异格式
- `fetchSingleFileGitDiff()` - 单文件差异
- `ToolUseDiff` / `GitDiffStats` / `PerFileStats` / `GitDiffResult` - 类型定义

### 依赖关系

**被以下模块导入**：
- `src/hooks/useDiffData.ts` - 差异数据 Hook
- `src/tools/FileWriteTool/FileWriteTool.ts` - 文件写入工具
- `src/tools/FileEditTool/FileEditTool.ts` - 文件编辑工具

**依赖的模块**：
- `src/utils/cwd.ts` - 当前工作目录
- `src/utils/detectRepository.ts` - 仓库检测
- `src/utils/execFileNoThrow.ts` - 命令执行
- `src/utils/file.ts` - 文件工具
- `src/utils/git.ts` - Git 操作

### 文件位置
- 源码：`src/utils/gitDiff.ts` (532 行)

## 依赖与外部交互

### Node.js 内置模块
- `fs/promises` - `access`, `readFile`
- `path` - 路径操作

### 项目内部依赖
- `src/utils/cwd.ts`
- `src/utils/detectRepository.ts`
- `src/utils/execFileNoThrow.ts`
- `src/utils/file.ts`
- `src/utils/git.ts`

### 外部依赖
- `diff` - `StructuredPatchHunk` 类型

### 外部工具
- Git 命令行工具

## 风险、边界与改进建议

### 已知风险

1. **内存使用**：解析大差异文件时可能占用大量内存
2. **编码问题**：假设差异输出为 UTF-8，可能无法正确处理其他编码
3. **二进制文件**：二进制文件在 numstat 中显示为 `-`，需要特殊处理

### 边界情况

1. **暂态状态**：合并/变基/拣选/还原期间返回 null
2. **未跟踪文件**：仅记录文件名，不读取内容（性能考虑）
3. **大文件**：超过 1MB 的差异文件被跳过
4. **多文件**：超过 500 个文件时跳过详细统计
5. **空仓库**：某些命令可能失败

### 改进建议

1. **流式解析**：对大差异使用流式解析减少内存占用
2. **增量加载**：支持差异的增量加载和分页
3. **缓存机制**：添加差异缓存避免重复计算
4. **测试覆盖**：当前没有专门的测试文件，建议添加单元测试
5. **配置化**：允许通过配置自定义限制值
6. **二进制处理**：改进二进制文件的差异显示

### 性能考虑

1. **--no-optional-locks**：避免触发不必要的 Git 锁
2. **5 秒超时**：防止慢速 Git 操作阻塞
3. **V8 字符串优化**：`'' + line` 技巧避免内存泄漏
