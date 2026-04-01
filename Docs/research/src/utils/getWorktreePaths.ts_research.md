# getWorktreePaths.ts 深度研究文档

## 场景与职责

`getWorktreePaths.ts` 提供 Git 工作树（worktree）路径检测功能，用于：

1. **多工作树支持**：检测当前 Git 仓库的所有工作树路径
2. **会话管理**：在会话恢复和列表显示时识别相关工作树
3. **跨工作树导航**：支持在不同工作树之间切换和导航

该模块是 CLI 版本的工作树检测实现，包含分析日志记录，依赖 CLI 的 git 解析器。

## 功能点目的

### 1. 工作树列表获取
- 使用 `git worktree list --porcelain` 获取所有工作树
- 解析 porcelain 格式输出提取路径
- 对路径进行 NFC Unicode 规范化

### 2. 工作树排序
- 当前工作树排在第一位
- 其他工作树按字母顺序排序

### 3. 分析日志
- 记录工作树检测的持续时间
- 记录工作树数量和成功状态

## 具体技术实现

### 关键流程

#### 工作树获取流程
```typescript
export async function getWorktreePaths(cwd: string): Promise<string[]> {
  const startTime = Date.now()

  const { stdout, code } = await execFileNoThrowWithCwd(
    gitExe(),  // 使用 CLI 的 git 解析器
    ['worktree', 'list', '--porcelain'],
    {
      cwd,
      preserveOutputOnError: false,
    },
  )

  const durationMs = Date.now() - startTime

  if (code !== 0) {
    logEvent('tengu_worktree_detection', {
      duration_ms: durationMs,
      worktree_count: 0,
      success: false,
    })
    return []
  }

  // 解析 porcelain 输出
  // 格式示例：
  // worktree /Users/foo/repo
  // HEAD abc123
  // branch refs/heads/main
  //
  // worktree /Users/foo/repo-wt1
  // ...
  const worktreePaths = stdout
    .split('\n')
    .filter(line => line.startsWith('worktree '))
    .map(line => line.slice('worktree '.length).normalize('NFC'))

  logEvent('tengu_worktree_detection', {
    duration_ms: durationMs,
    worktree_count: worktreePaths.length,
    success: true,
  })

  // 排序：当前工作树优先
  const currentWorktree = worktreePaths.find(
    path => cwd === path || cwd.startsWith(path + sep),
  )
  const otherWorktrees = worktreePaths
    .filter(path => path !== currentWorktree)
    .sort((a, b) => a.localeCompare(b))

  return currentWorktree ? [currentWorktree, ...otherWorktrees] : otherWorktrees
}
```

### Porcelain 格式解析

Git 的 `--porcelain` 格式是机器可读的稳定输出格式：
- 每行一个字段
- 工作树路径行格式：`worktree <path>`
- 空行分隔不同的工作树条目

## 关键代码路径与文件引用

### 核心导出
- `getWorktreePaths(cwd: string): Promise<string[]>` - 获取工作树路径列表

### 依赖关系

**被以下模块导入**：
- `src/bridge/bridgePointer.ts` - 桥接指针
- `src/main.tsx` - 主入口
- `src/components/LogSelector.tsx` - 日志选择器
- `src/commands/resume/resume.tsx` - 恢复命令
- `src/utils/listSessionsImpl.ts` - 会话列表实现
- `src/utils/sessionStoragePortable.ts` - 便携会话存储
- `src/utils/sessionStorage.ts` - 会话存储

**依赖的模块**：
- `src/services/analytics/index.ts` - `logEvent` 分析日志
- `src/utils/execFileNoThrow.ts` - `execFileNoThrowWithCwd`
- `src/utils/git.ts` - `gitExe`

### 文件位置
- 源码：`src/utils/getWorktreePaths.ts` (70 行)

## 依赖与外部交互

### Node.js 内置模块
- `path` - `sep` 用于路径分隔符

### 项目内部依赖
- `src/services/analytics/index.ts` - 分析事件记录
- `src/utils/execFileNoThrow.ts` - 安全的命令执行
- `src/utils/git.ts` - Git 可执行文件解析

### 外部依赖
- Git 命令行工具

## 风险、边界与改进建议

### 已知风险

1. **Git 版本兼容**：`git worktree list --porcelain` 需要较新版本的 Git
2. **子模块检测**：可能将子模块误识别为工作树
3. **路径规范化**：NFC 规范化在某些文件系统上可能不适用

### 边界情况

1. **非 Git 目录**：返回空数组
2. **单工作树仓库**：返回包含主工作树的单元素数组
3. **嵌套工作树**：当前工作树检测使用 `startsWith`，可能误判嵌套路径
4. **符号链接**：路径比较前未解析符号链接

### 与 getWorktreePathsPortable 的关系

该模块有一个轻量级替代版本 `getWorktreePathsPortable.ts`：
- `getWorktreePaths`: 包含分析日志，使用 CLI 的 `gitExe()` 解析器
- `getWorktreePathsPortable`: 无分析日志，无 CLI 依赖，使用原生 `child_process`

`getWorktreePathsPortable` 用于：
- SDK 场景（`listSessionsImpl.ts`）
- 需要避免 CLI 依赖链的场景（execa → cross-spawn → which）

### 改进建议

1. **缓存机制**：添加短期缓存避免重复执行 git 命令
2. **错误分类**：区分 Git 未安装、不在 Git 仓库、Git 版本过低等错误
3. **符号链接处理**：解析符号链接后再比较路径
4. **测试覆盖**：当前没有专门的测试文件，建议添加单元测试
5. **性能优化**：对于无工作树的仓库快速返回
