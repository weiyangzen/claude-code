# src/utils/claudemd.ts 深度研究文档

## 场景与职责

`claudemd.ts` 是 Claude Code 的记忆系统核心模块，负责发现、加载和管理 CLAUDE.md 系列记忆文件。这些记忆文件为用户提供项目特定的指令和上下文，帮助 Claude 更好地理解和处理代码库。

记忆文件加载优先级（从低到高）：
1. **Managed memory** (`/etc/claude-code/CLAUDE.md`) - 全局系统级指令
2. **User memory** (`~/.claude/CLAUDE.md`) - 用户私有全局指令
3. **Project memory** (`CLAUDE.md`, `.claude/CLAUDE.md`, `.claude/rules/*.md`) - 项目级指令
4. **Local memory** (`CLAUDE.local.md`) - 项目私有指令

文件越靠近当前工作目录，优先级越高（后加载的覆盖先加载的）。

## 功能点目的

### 1. 记忆文件发现与加载
- 从多个来源发现记忆文件（系统、用户、项目、本地）
- 支持目录向上遍历，收集所有相关记忆文件
- 处理嵌套 worktree 场景，避免重复加载

### 2. 条件规则系统 (Conditional Rules)
- 支持通过 frontmatter 中的 `paths` 字段定义 glob 模式
- 只有匹配目标路径的规则才会被加载
- 用于大型项目中按文件类型/路径应用不同规则

### 3. @include 指令支持
- 记忆文件可以包含其他文件使用 `@path` 语法
- 支持相对路径 (`@./file.md`)、家目录路径 (`@~/file.md`)、绝对路径 (`@/file.md`)
- 自动防止循环引用（通过 `processedPaths` Set 追踪）
- 最大嵌套深度限制（`MAX_INCLUDE_DEPTH = 5`）

### 4. 内容处理
- **Frontmatter 解析**：提取 YAML 前置元数据
- **HTML 注释剥离**：使用 marked lexer 识别并移除块级 HTML 注释
- **AutoMem/TeamMem 截断**：对大型记忆文件应用内容大小限制
- **内容差异追踪**：标记内容是否经过转换（用于缓存和变更检测）

### 5. 排除模式支持
- 通过 `claudeMdExcludes` 设置排除特定文件
- 支持 glob 模式和绝对路径
- 处理符号链接解析（如 macOS `/tmp` -> `/private/tmp`）

## 具体技术实现

### 核心数据结构

```typescript
export type MemoryFileInfo = {
  path: string           // 文件绝对路径
  type: MemoryType       // 'User' | 'Project' | 'Local' | 'Managed' | 'AutoMem' | 'TeamMem'
  content: string        // 处理后的内容
  parent?: string        // 包含此文件的父文件路径
  globs?: string[]       // frontmatter 中的路径模式
  contentDiffersFromDisk?: boolean  // 内容是否经过转换
  rawContent?: string    // 原始磁盘内容（当内容被转换时）
}
```

### 关键流程

#### 1. 记忆文件加载流程 (`getMemoryFiles`)

```
1. 初始化 processedPaths Set 用于去重和循环检测
2. 加载 Managed 文件（系统级，始终加载）
3. 加载 Managed .claude/rules/*.md 文件
4. 如果启用 userSettings，加载 User 文件和规则
5. 从当前目录向上遍历到根目录：
   - 处理嵌套 worktree 检测（避免重复加载）
   - 加载 Project 文件（CLAUDE.md, .claude/CLAUDE.md）
   - 加载 Project 规则文件
   - 加载 Local 文件（CLAUDE.local.md）
6. 如果启用额外目录支持，加载指定目录的记忆文件
7. 如果启用 AutoMem，加载记忆目录入口点
8. 如果启用 TeamMem，加载团队记忆入口点
9. 记录遥测数据，触发 InstructionsLoaded hooks
```

#### 2. 单个记忆文件处理 (`processMemoryFile`)

```
1. 检查是否已处理或超过最大深度
2. 检查是否被 claudeMdExcludes 排除
3. 解析符号链接路径
4. 读取文件内容（safelyReadMemoryFileAsync）
5. 解析内容（parseMemoryFileContent）：
   - 检查文件扩展名（TEXT_FILE_EXTENSIONS）
   - 解析 frontmatter 提取 paths
   - 使用 marked lexer 处理 HTML 注释
   - 提取 @include 路径
   - 对 AutoMem/TeamMem 应用截断
6. 递归处理所有 @include 引用
7. 返回包含主文件和所有引用的数组
```

#### 3. 条件规则处理 (`processConditionedMdRules`)

```
1. 调用 processMdRules 获取所有带 frontmatter paths 的规则文件
2. 对每个文件，使用 ignore 库检查目标路径是否匹配 globs
3. Project 规则：glob 相对于 .claude 的父目录
4. Managed/User 规则：glob 相对于原始 CWD
```

### 关键代码路径

#### 文件扩展名白名单
```typescript
const TEXT_FILE_EXTENSIONS = new Set([
  '.md', '.txt', '.json', '.yaml', '.yml', '.toml',
  '.js', '.ts', '.tsx', '.jsx', '.py', '.go', '.rs',
  // ... 100+ 扩展名
])
```

#### HTML 注释剥离
```typescript
export function stripHtmlComments(content: string): {
  content: string
  stripped: boolean
} {
  if (!content.includes('<!--')) return { content, stripped: false }
  // 使用 marked Lexer 识别块级 HTML 注释
  return stripHtmlCommentsFromTokens(new Lexer({ gfm: false }).lex(content))
}
```

#### @include 路径提取正则
```typescript
const includeRegex = /(?:^|\s)@((?:[^\s\\]|\\ )+)/g
// 匹配 @path, @./path, @~/path, @/path
// 支持转义空格：\ 
// 支持片段标识符剥离（#heading）
```

### 缓存机制

`getMemoryFiles` 使用 `lodash-es/memoize` 进行缓存：
- 缓存键：无参数（单例缓存）
- 通过 `clearMemoryFileCaches()` 清除缓存（用于工作树切换、设置同步等）
- 通过 `resetGetMemoryFilesCache(reason)` 重置并触发 hook 报告

## 依赖与外部交互

### 外部依赖

| 依赖 | 用途 |
|------|------|
| `marked` (Lexer) | Markdown 解析，用于 HTML 注释剥离和 @include 提取 |
| `ignore` | Glob 模式匹配，用于条件规则和排除模式 |
| `picomatch` | 高级 glob 匹配，用于 claudeMdExcludes |
| `lodash-es/memoize` | 记忆文件列表缓存 |

### 内部模块依赖

| 模块 | 用途 |
|------|------|
| `src/utils/config.ts` | 获取项目配置（getCurrentProjectConfig） |
| `src/utils/settings/settings.ts` | 检查设置源是否启用 |
| `src/utils/frontmatterParser.ts` | 解析 YAML frontmatter |
| `src/utils/hooks.ts` | 触发 InstructionsLoaded hooks |
| `src/utils/fsOperations.ts` | 文件系统操作抽象 |
| `src/utils/git.ts` | 检测嵌套 worktree |
| `src/utils/memory/types.ts` | MemoryType 类型定义 |
| `src/memdir/paths.ts` | AutoMem 路径获取 |
| `src/memdir/teamMemPaths.ts` | TeamMem 路径获取（条件加载） |

### 调用方

| 调用方 | 用途 |
|--------|------|
| `src/context.ts` | 构建系统上下文和用户上下文 |
| `src/utils/status.tsx` | 显示记忆文件状态 |
| `src/commands/memory/memory.tsx` | /memory 命令实现 |
| `src/services/compact/postCompactCleanup.ts` | 压缩后重新加载记忆 |
| `src/services/settingsSync/index.ts` | 设置同步后刷新记忆 |
| `src/components/ClaudeMdExternalIncludesDialog.tsx` | 检查外部包含警告 |

## 风险、边界与改进建议

### 已知风险

1. **循环引用风险**
   - 通过 `processedPaths` Set 和 `MAX_INCLUDE_DEPTH` 限制缓解
   - 但深层嵌套仍可能导致性能问题

2. **符号链接处理**
   - 使用 `safeResolvePath` 解析符号链接
   - 在 `processedPaths` 中同时添加原始路径和解析后路径
   - 但仍可能存在边缘情况（如符号链接循环）

3. **大文件处理**
   - `MAX_MEMORY_CHARACTER_COUNT = 40000` 是建议值，非强制限制
   - AutoMem/TeamMem 有截断逻辑，但其他类型没有
   - 超大文件可能导致内存问题

4. **并发修改**
   - 文件可能在读取过程中被修改
   - 没有文件锁定机制

### 边界情况

1. **空文件/目录**
   - 空文件被静默跳过（`!memoryFile.content.trim()`）
   - 不存在的目录在 processMdRules 中返回空数组

2. **权限错误**
   - EACCES 错误被记录到遥测
   - ENOENT/EISDIR 被静默忽略

3. **Windows 路径**
   - 使用 `normalizePathForComparison` 处理驱动器大小写差异
   - 路径分隔符统一处理

4. **嵌套 Worktree**
   - 通过 `findGitRoot` 和 `findCanonicalGitRoot` 检测
   - 跳过主仓库中但在 worktree 外的 Project 文件

### 改进建议

1. **性能优化**
   - 考虑使用文件系统监视（fs.watch）替代每次遍历
   - 对大型规则目录实现增量更新
   - 添加文件内容哈希缓存，避免重复解析未变更文件

2. **可观测性**
   - 添加更详细的加载时间指标（每个文件类型的耗时）
   - 记录被排除的文件（用于调试排除模式问题）
   - 添加内存使用监控（大文件检测）

3. **功能增强**
   - 支持异步/延迟加载大型记忆文件
   - 添加记忆文件热重载（开发模式）
   - 支持记忆文件版本控制（与 git 集成）

4. **健壮性**
   - 添加文件锁定或原子读取机制
   - 改进 YAML 解析错误恢复
   - 添加 @include 循环检测的详细错误信息

5. **代码质量**
   - `getMemoryFiles` 函数过长（~300行），建议拆分为更小函数
   - 减少条件编译（feature('TEAMMEM')）的重复代码
   - 统一错误处理模式（有些抛错，有些静默忽略）
