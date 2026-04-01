# src/utils/cleanup.ts 深度研究文档

## 场景与职责

`cleanup.ts` 是 Claude Code 的磁盘清理模块，负责定期清理过期的临时文件、日志文件和缓存数据，防止磁盘空间无限增长。清理任务主要在后台运行，不会阻塞用户交互。

清理的数据类型包括：
- 消息和错误日志文件
- 会话存储文件（JSONL、asciicast）
- MCP 日志目录
- 计划文件（plans）
- 文件历史备份
- 会话环境目录
- 调试日志
- 镜像缓存
- 粘贴存储
- npm 缓存（Anthropic 包）
- 旧版本安装文件
- 过时的 agent worktrees

## 功能点目的

### 1. 基于时间的清理策略
- 默认保留 30 天的数据（`DEFAULT_CLEANUP_PERIOD_DAYS = 30`）
- 用户可通过 `cleanupPeriodDays` 设置自定义保留期
- 计算截止时间（cutoff date）进行文件筛选

### 2. 多类型文件清理
- **消息文件**：错误日志、MCP 日志
- **会话文件**：项目目录下的会话数据、工具结果
- **计划文件**：`~/.claude/plans/*.md`
- **文件历史**：`~/.claude/file-history/`
- **会话环境**：`~/.claude/session-env/`
- **调试日志**：`~/.claude/debug/*.txt`

### 3. 节流机制
- npm 缓存清理：每天最多一次（通过 marker 文件 + 锁）
- 版本清理：每天最多一次（throttled wrapper）
- 使用 `lockfile` 模块防止并发执行

### 4. 后台清理协调
- `cleanupOldMessageFilesInBackground()`：协调所有清理任务
- 在设置验证错误时跳过（防止意外删除）
- 记录遥测事件（worktree 清理数量）

## 具体技术实现

### 核心数据结构

```typescript
export type CleanupResult = {
  messages: number  // 成功清理的文件数
  errors: number    // 清理失败的计数
}

// 文件名日期转换（ISO 格式替换为可解析格式）
// 例如：2024-01-15T10-30-00-000Z -> 2024-01-15T10:30:00.000Z
```

### 关键流程

#### 1. 消息文件清理 (`cleanupOldMessageFiles`)

```
1. 获取截止时间（getCutoffDate）
2. 清理错误日志目录（CACHE_PATHS.errors()）
3. 遍历 MCP 日志目录（mcp-logs-*）：
   - 清理每个目录中的旧文件
   - 尝试删除空目录
4. 返回清理结果统计
```

#### 2. 会话文件清理 (`cleanupOldSessionFiles`)

```
1. 获取项目目录列表（~/.claude/projects/）
2. 对每个项目目录：
   - 遍历项目下的所有条目
   - 如果是文件（.jsonl, .cast）：按 mtime 删除
   - 如果是目录（会话目录）：
     * 遍历 tool-results/ 下的工具目录
     * 删除旧文件，尝试删除空目录
     * 尝试删除空会话目录
3. 尝试删除空项目目录
```

#### 3. npm 缓存清理 (`cleanupNpmCacheForAnthropicPackages`)

```
1. 检查 marker 文件（~/.claude/.npm-cache-cleanup）
2. 如果 24 小时内已运行，跳过
3. 获取锁（防止并发）
4. 使用 cacache 流式读取 npm 缓存索引
5. 收集所有 @anthropic-ai/claude-* 包条目
6. 按包名分组，每包保留最新的 5 个版本
7. 删除超过 1 天或超出保留数量的条目
8. 更新 marker 文件
9. 释放锁
```

### 关键代码路径

#### 文件名日期解析
```typescript
export function convertFileNameToDate(filename: string): Date {
  const isoStr = filename
    .split('.')[0]!
    .replace(/T(\d{2})-(\d{2})-(\d{2})-(\d{3})Z/, 'T$1:$2:$3.$4Z')
  return new Date(isoStr)
}
```

#### 条件删除辅助函数
```typescript
async function unlinkIfOld(
  filePath: string,
  cutoffDate: Date,
  fsImpl: FsOperations,
): Promise<boolean> {
  const stats = await fsImpl.stat(filePath)
  if (stats.mtime < cutoffDate) {
    await fsImpl.unlink(filePath)
    return true
  }
  return false
}
```

#### 节流清理包装器
```typescript
export async function cleanupNpmCacheForAnthropicPackages(): Promise<void> {
  const markerPath = join(getClaudeConfigHomeDir(), '.npm-cache-cleanup')
  
  // 检查上次运行时间
  try {
    const stat = await fs.stat(markerPath)
    if (Date.now() - stat.mtimeMs < ONE_DAY_MS) return
  } catch {
    // 文件不存在，继续执行
  }
  
  // 获取锁
  try {
    await lockfile.lock(markerPath, { retries: 0, realpath: false })
  } catch {
    return  // 锁被占用，跳过
  }
  
  try {
    // 执行清理...
    await fs.writeFile(markerPath, new Date().toISOString())
  } finally {
    await lockfile.unlock(markerPath, { realpath: false }).catch(() => {})
  }
}
```

### 清理结果合并

```typescript
export function addCleanupResults(
  a: CleanupResult,
  b: CleanupResult,
): CleanupResult {
  return {
    messages: a.messages + b.messages,
    errors: a.errors + b.errors,
  }
}
```

## 依赖与外部交互

### 外部依赖

| 依赖 | 用途 |
|------|------|
| `cacache` | npm 缓存操作（流式读取、删除条目） |

### 内部模块依赖

| 模块 | 用途 |
|------|------|
| `src/utils/cachePaths.ts` | 获取缓存目录路径（CACHE_PATHS） |
| `src/utils/fsOperations.ts` | 文件系统操作抽象（getFsImplementation） |
| `src/utils/sessionStorage.ts` | 获取项目目录（getProjectsDir） |
| `src/utils/settings/settings.ts` | 获取清理周期设置 |
| `src/utils/settings/allErrors.ts` | 检查设置验证错误 |
| `src/utils/lockfile.ts` | 文件锁实现 |
| `src/utils/imageStore.ts` | 清理镜像缓存 |
| `src/utils/pasteStore.ts` | 清理粘贴存储 |
| `src/utils/worktree.ts` | 清理过时 worktrees |
| `src/utils/nativeInstaller/index.ts` | 清理旧版本 |
| `src/utils/toolResultStorage.ts` | 工具结果子目录常量 |

### 调用方

| 调用方 | 用途 |
|--------|------|
| `src/utils/backgroundHousekeeping.ts` | 启动后台清理任务 |
| `src/cli/print.ts` | 打印模式后的清理 |
| `src/main.tsx` | 启动时的版本清理 |
| `src/utils/cleanup.ts` 自身 | 各种专门的清理函数 |

## 风险、边界与改进建议

### 已知风险

1. **设置验证错误时的行为**
   - 如果设置有验证错误且用户显式设置了 `cleanupPeriodDays`，会完全跳过清理
   - 这是为了防止在配置错误时使用默认的 30 天周期意外删除文件
   - 但这也可能导致磁盘空间持续增长

2. **并发清理竞争**
   - 使用文件锁（lockfile）防止并发，但锁失败时静默跳过
   - 长时间运行的清理可能阻塞其他进程的清理尝试

3. **npm 缓存清理性能**
   - 使用 `cacache.ls.stream` 流式读取，但大型缓存仍可能耗时
   - 之前的实现使用 `cacache.verify()` 会阻塞事件循环 60+ 秒

4. **错误处理粒度**
   - 单个文件删除失败会继续处理其他文件
   - 但错误计数可能不准确（某些错误被静默忽略）

### 边界情况

1. **不存在的目录**
   - 所有清理函数都处理 `ENOENT` 错误，返回空结果
   - 首次运行时所有目录都不存在，不会报错

2. **文件时间戳**
   - 依赖文件系统的 `mtime`（修改时间）
   - 如果系统时间被修改，可能导致意外删除或保留

3. **文件名格式**
   - `convertFileNameToDate` 假设特定格式（ISO 时间戳）
   - 非标准文件名的文件不会被处理

4. **权限错误**
   - `EACCES` 错误会增加 error 计数，但继续处理
   - 某些权限错误可能被记录到遥测

### 改进建议

1. **可配置性**
   - 允许按数据类型配置不同的保留期
   - 添加排除模式（某些文件/目录永不清理）
   - 支持清理前确认（交互模式）

2. **可观测性**
   - 添加清理报告（清理了什么、释放了多少空间）
   - 记录每个清理类型的详细遥测
   - 添加清理历史记录

3. **健壮性**
   - 添加磁盘空间检查（低空间时更激进的清理）
   - 实现清理任务队列（避免同时运行多个清理）
   - 添加清理预览模式（dry-run）

4. **性能优化**
   - 对大型目录使用并行删除（控制并发数）
   - 考虑使用原生 find 命令进行批量删除
   - 添加清理进度指示（长时间清理时）

5. **代码质量**
   - 统一错误处理（有些函数使用 try/catch，有些检查错误码）
   - 提取通用的目录遍历逻辑
   - 添加更多单元测试（特别是边界情况）

6. **安全性**
   - 添加路径遍历防护（确保只删除预期目录下的文件）
   - 验证文件所有权后再删除
   - 考虑使用回收站/垃圾桶而非直接删除（平台支持）
