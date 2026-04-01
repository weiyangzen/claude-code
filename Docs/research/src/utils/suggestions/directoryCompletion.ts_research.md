# directoryCompletion.ts 深度研究文档

## 场景与职责

`directoryCompletion.ts` 是 Claude Code CLI 的**文件路径自动补全引擎**，专门处理目录和文件路径的智能提示功能。该模块在用户输入文件路径时提供实时的补全建议，是终端文件操作体验的核心组件：

1. **目录补全**：提供子目录名称的补全建议（用于 `cd` 类操作）
2. **文件路径补全**：提供文件和目录的混合补全建议
3. **路径解析**：处理各种路径格式（相对路径、绝对路径、home 目录 `~`）
4. **智能缓存**：使用 LRU 缓存避免重复的磁盘 I/O 操作
5. **路径类型检测**：识别输入是否为路径类 token（如 `./`, `../`, `~/`）

该模块在命令参数补全、文件选择器等场景中发挥关键作用。

## 功能点目的

### 1. 目录补全 (`getDirectoryCompletions`)
- **目的**：根据部分路径输入返回匹配的子目录列表
- **使用场景**：用户输入 `cd src/co` 时提示 `components/`, `config/` 等
- **限制**：仅返回目录，不包含文件

### 2. 文件路径补全 (`getPathCompletions`)
- **目的**：返回文件和目录的混合补全建议
- **使用场景**：文件选择、路径参数输入
- **可配置**：支持包含/排除隐藏文件、仅目录模式

### 3. 路径解析 (`parsePartialPath`)
- **目的**：将部分路径解析为目录和前缀组件
- **处理逻辑**：
  - 空输入 → 使用 basePath 或当前工作目录
  - 以分隔符结尾 → 视为目录，前缀为空
  - 其他 → 分割为 `dirname` 和 `basename`

### 4. 目录扫描 (`scanDirectory`, `scanDirectoryForPaths`)
- **目的**：读取目录内容并过滤/格式化
- **缓存**：LRU 缓存，500 条目，5 分钟 TTL
- **限制**：最多返回 100 个结果（MVP 限制）

### 5. 路径类型检测 (`isPathLikeToken`)
- **目的**：判断 token 是否为路径类输入
- **识别模式**：`~/`, `/`, `./`, `../`, `~`, `.`, `..`

## 具体技术实现

### 关键数据结构

```typescript
// 目录条目
type DirectoryEntry = {
  name: string      // 目录名
  path: string      // 完整路径
  type: 'directory' // 固定为目录
}

// 路径条目（文件或目录）
type PathEntry = {
  name: string                    // 文件名/目录名
  path: string                    // 完整路径
  type: 'directory' | 'file'      // 类型
}

// 解析后的路径
type ParsedPath = {
  directory: string  // 目录部分
  prefix: string     // 前缀（用于匹配）
}

// 补全选项
type CompletionOptions = {
  basePath?: string   // 基础路径（默认当前工作目录）
  maxResults?: number // 最大结果数（默认 10）
}

type PathCompletionOptions = CompletionOptions & {
  includeFiles?: boolean   // 是否包含文件（默认 true）
  includeHidden?: boolean  // 是否包含隐藏文件（默认 false）
}
```

### 核心算法流程

#### 1. 路径解析流程
```
parsePartialPath(partialPath, basePath)
├── 空输入处理
│   └── 返回 { directory: basePath || getCwd(), prefix: '' }
├── 路径扩展（expandPath）
│   ├── 处理 ~ → home 目录
│   ├── 处理相对路径 → 绝对路径
│   └── 处理平台特定分隔符
├── 以分隔符结尾？
│   └── 返回 { directory: resolved, prefix: '' }
└── 分割目录和文件名
    ├── directory = dirname(resolved)
    └── prefix = basename(partialPath)
```

#### 2. 目录扫描流程
```
scanDirectory(dirPath)
├── 缓存检查（LRU）
│   └── 命中 → 返回缓存结果
├── 磁盘读取
│   ├── fs.readdir(dirPath, { withFileTypes: true })
│   └── 过滤：仅目录 + 非隐藏
├── 格式化结果
│   └── 限制 100 条
└── 缓存结果
```

#### 3. 路径补全流程
```
getPathCompletions(partialPath, options)
├── 解析路径 → { directory, prefix }
├── 扫描目录（含缓存）
├── 过滤匹配
│   ├── 前缀匹配（大小写不敏感）
│   ├── 类型过滤（includeFiles）
│   └── 隐藏文件过滤（includeHidden）
├── 限制结果数（默认 10）
└── 格式化输出
    ├── 目录：添加尾部 /
    └── 文件：保持原样
```

### 缓存策略

```typescript
// 目录缓存（仅目录）
const directoryCache = new LRUCache<string, DirectoryEntry[]>({
  max: 500,
  ttl: 5 * 60 * 1000, // 5 分钟
})

// 路径缓存（文件+目录）
const pathCache = new LRUCache<string, PathEntry[]>({
  max: 500,
  ttl: 5 * 60 * 1000,
})

// 缓存键：对于路径缓存，包含 includeHidden 标志
const cacheKey = `${dirPath}:${includeHidden}`
```

### 路径格式化逻辑

```typescript
// 构造相对路径显示
const hasSeparator = partialPath.includes('/') || partialPath.includes(sep)
let dirPortion = ''
if (hasSeparator) {
  const lastSeparatorPos = Math.max(
    partialPath.lastIndexOf('/'),
    partialPath.lastIndexOf(sep)
  )
  dirPortion = partialPath.substring(0, lastSeparatorPos + 1)
}
// 去除开头的 ./（仅用于当前目录搜索）
if (dirPortion.startsWith('./') || dirPortion.startsWith('.' + sep)) {
  dirPortion = dirPortion.slice(2)
}

// 最终显示：dirPortion + entry.name
```

### 关键代码路径

| 功能 | 函数 | 行号 |
|------|------|------|
| 目录补全 | `getDirectoryCompletions` | 121-140 |
| 路径补全 | `getPathCompletions` | 208-255 |
| 路径解析 | `parsePartialPath` | 55-78 |
| 目录扫描 | `scanDirectory` | 84-116 |
| 路径扫描 | `scanDirectoryForPaths` | 168-203 |
| 路径类型检测 | `isPathLikeToken` | 152-162 |
| 缓存清除 | `clearDirectoryCache`, `clearPathCache` | 145-148, 260-263 |

## 依赖与外部交互

### 直接依赖模块

```typescript
import { LRUCache } from 'lru-cache'  // LRU 缓存库
import { basename, dirname, join, sep } from 'path'  // 路径操作
import type { SuggestionItem } from 'src/components/PromptInput/PromptInputFooterSuggestions.js'  // UI 类型
import { getCwd } from 'src/utils/cwd.js'  // 获取当前工作目录
import { getFsImplementation } from 'src/utils/fsOperations.js'  // 文件系统操作
import { logError } from 'src/utils/log.js'  // 错误日志
import { expandPath } from 'src/utils/path.js'  // 路径扩展（~ 等）
```

### 依赖详解

1. **lru-cache** (外部库)
   - 用途：高效的 LRU 缓存实现
   - 配置：最大 500 条目，5 分钟 TTL

2. **cwd.ts** (内部模块)
   - `getCwd()`: 获取当前工作目录
   - 支持 AsyncLocalStorage 的上下文覆盖

3. **fsOperations.ts** (内部模块)
   - `getFsImplementation()`: 获取文件系统实现
   - 支持抽象文件系统（用于测试、虚拟文件系统等）
   - 返回的 `fs.readdir()` 返回 `Dirent[]` 对象

4. **path.ts** (内部模块)
   - `expandPath()`: 扩展路径（处理 `~`, 相对路径等）
   - 支持跨平台路径格式

5. **log.ts** (内部模块)
   - `logError()`: 记录错误信息

### 调用方

- `shellCompletion.ts`: Shell 命令补全
- `fileSuggestions.ts`: 文件建议处理
- `unifiedSuggestions.ts`: 统一建议处理
- `PromptInput.tsx`: 输入处理

## 风险、边界与改进建议

### 已知风险

1. **缓存一致性问题**
   - 文件系统变更（创建/删除/重命名）不会自动使缓存失效
   - **当前处理**：5 分钟 TTL 后自动过期
   - **风险**：用户可能看到过时的目录列表

2. **符号链接循环**
   - 目录扫描不处理符号链接循环
   - **当前处理**：依赖操作系统的 `readdir` 行为
   - **风险**：可能导致无限递归（虽然有限制 100 条）

3. **大目录性能**
   - 扫描包含数千文件的目录时可能阻塞
   - **当前处理**：限制 100 条结果，异步读取
   - **风险**：首次扫描仍可能耗时较长

4. **权限错误**
   - 无权限访问的目录会返回空数组
   - **当前处理**：try-catch 捕获，记录错误日志

### 边界情况

| 场景 | 处理方式 |
|------|----------|
| 空输入 | 返回 basePath 下的目录 |
| 以 `/` 结尾 | 视为目录，列出其子目录 |
| 不存在的路径 | `dirname` 部分可能不存在，返回空数组 |
| 权限拒绝 | 捕获错误，返回空数组，记录日志 |
| 隐藏文件 | 默认排除，可通过 `includeHidden` 包含 |
| 仅文件匹配 | 目录优先排序，文件在后 |
| Windows 路径 | 支持 `\` 和 `/` 分隔符 |

### 改进建议

1. **缓存优化**
   - 添加文件系统监视（`fs.watch`）主动使缓存失效
   - 实现分层缓存（内存 + 磁盘）
   - 添加缓存统计和监控

2. **性能优化**
   - 使用流式读取处理大目录
   - 实现增量扫描（仅读取变更部分）
   - 添加并发控制（避免同时扫描多个目录）

3. **功能增强**
   - 支持模糊匹配（类似命令补全）
   - 添加最近使用路径的记忆功能
   - 支持路径别名（如 `@src` → `./src`）
   - 添加文件类型图标/颜色提示

4. **可配置性**
   - 允许用户配置缓存 TTL
   - 支持自定义忽略模式（如 `.gitignore`）
   - 可配置的结果数量限制

5. **错误处理**
   - 更细粒度的错误分类（权限 vs 不存在 vs 其他）
   - 用户友好的错误提示
   - 网络文件系统的特殊处理

### 测试要点

- 各种路径格式的解析正确性
- 缓存命中/未命中的行为
- 大目录的性能表现
- 权限错误的处理
- 跨平台路径分隔符处理
- 隐藏文件包含/排除逻辑
- 并发访问的安全性
