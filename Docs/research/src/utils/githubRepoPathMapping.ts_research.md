# githubRepoPathMapping.ts 深度研究

## 场景与职责

本模块负责管理 GitHub 仓库与本地文件系统路径之间的映射关系，核心用途是支持 "teleport" 功能——允许用户在不同仓库之间快速切换工作目录。当用户启动 Claude Code 时，系统会自动检测当前 GitHub 仓库并记录其本地路径，后续可通过仓库名快速定位到已知的本地克隆位置。

**关键场景：**
1. **启动时路径追踪**：fire-and-forget 方式在后台更新映射，不阻塞启动流程
2. **Teleport 目录切换**：根据仓库名查找本地路径，实现跨仓库快速导航
3. **路径验证与清理**：验证路径是否仍指向预期的仓库，清理无效映射

## 功能点目的

### 1. 路径映射维护
- **目的**：建立 `owner/repo` → 本地路径数组的映射
- **机制**：使用全局配置 (`~/.claude.json`) 持久化存储
- **LRU 特性**：最近使用的路径被提升到数组首位

### 2. 路径规范化
- 使用 `realpath` 解析符号链接，确保路径一致性
- NFC 规范化处理（Unicode 兼容性）
- 使用 git 根目录而非当前工作目录，支持从子目录启动

### 3. 大小写不敏感匹配
- 仓库键统一转为小写存储
- 适配 GitHub 的大小写不敏感特性

## 具体技术实现

### 核心数据结构
```typescript
// 存储在 GlobalConfig.githubRepoPaths 中
type GithubRepoPaths = Record<string, string[]>  // key: "owner/repo" (lowercase)
```

### 关键流程

#### updateGithubRepoPathMapping() - 更新映射
```
1. 检测当前仓库 (detectCurrentRepository)
2. 获取 git 根目录 (findGitRoot)
3. 解析符号链接 (realpath + NFC normalize)
4. 检查是否已在首位
5. 移除旧条目 +  prepend 新路径
6. 保存全局配置
```

**错误处理策略**：
- 所有错误静默处理（非阻塞操作）
- 调试日志通过 `logForDebugging` 记录

#### getKnownPathsForRepo() - 查询路径
- 直接从内存中的全局配置读取
- 返回路径数组（按使用频率排序）

#### filterExistingPaths() - 过滤有效路径
- 并行检查路径存在性
- 用于 teleport 前验证路径是否仍有效

#### validateRepoAtPath() - 验证路径
- 读取指定路径的 git remote URL
- 解析并比对仓库名
- 大小写不敏感比较

#### removePathFromRepo() - 移除无效路径
- 从映射中删除指定路径
- 如无剩余路径则删除整个仓库键

## 关键代码路径与文件引用

### 本文件导出函数
| 函数 | 用途 | 调用方 |
|------|------|--------|
| `updateGithubRepoPathMapping` | 启动时更新映射 | `main.tsx`, `interactiveHelpers.tsx` |
| `getKnownPathsForRepo` | 获取仓库的已知路径 | `protocolHandler.ts` (deep link 处理) |
| `filterExistingPaths` | 过滤存在的路径 | `TeleportRepoMismatchDialog.tsx` |
| `validateRepoAtPath` | 验证路径正确性 | `TeleportRepoMismatchDialog.tsx` |
| `removePathFromRepo` | 移除无效路径 | `TeleportRepoMismatchDialog.tsx` |

### 依赖模块
```typescript
import { realpath } from 'fs/promises'
import { getOriginalCwd } from '../bootstrap/state.js'
import { getGlobalConfig, saveGlobalConfig } from './config.js'
import { detectCurrentRepository, parseGitHubRepository } from './detectRepository.js'
import { pathExists } from './file.js'
import { getRemoteUrlForDir } from './git/gitFilesystem.js'
import { findGitRoot } from './git.js'
```

### 配置存储位置
- **文件**: `~/.claude.json`
- **字段**: `githubRepoPaths`
- **类型**: `Record<string, string[]>`

## 依赖与外部交互

### 上游依赖
1. **detectRepository.ts**: 仓库检测逻辑
   - `detectCurrentRepository()`: 异步检测当前 git 仓库
   - `parseGitHubRepository()`: 解析 remote URL 为 owner/repo 格式

2. **git.ts**: Git 基础操作
   - `findGitRoot()`: 向上查找 .git 目录

3. **git/gitFilesystem.ts**: 文件系统级 git 操作
   - `getRemoteUrlForDir()`: 读取指定目录的 remote URL

4. **config.ts**: 全局配置管理
   - `getGlobalConfig()`: 读取配置
   - `saveGlobalConfig()`: 原子写入配置

### 下游调用方
1. **main.tsx**: 启动时调用 `updateGithubRepoPathMapping()`
2. **interactiveHelpers.tsx**: 启动时更新映射
3. **protocolHandler.ts**: 处理 `claude-cli://` deep links 时查询路径
4. **TeleportRepoMismatchDialog.tsx**: Teleport 时验证和清理路径

## 风险、边界与改进建议

### 已知风险

1. **并发写入冲突**
   - 风险：多进程同时更新 `githubRepoPaths` 可能丢失更新
   - 缓解：`saveGlobalConfig` 使用文件锁机制

2. **路径失效**
   - 风险：用户删除或移动仓库后，映射仍指向旧路径
   - 缓解：`filterExistingPaths` 和 `validateRepoAtPath` 提供验证机制

3. **大小写敏感文件系统**
   - 风险：在大小写敏感的文件系统上，大小写不敏感的键可能导致意外行为
   - 缓解：统一使用小写键，但保留原始路径

4. **符号链接循环**
   - 风险：`realpath` 可能遇到循环链接
   - 缓解：`realpath` 系统调用会自动处理并抛出错误

### 边界情况

1. **非 GitHub 仓库**：`detectCurrentRepository` 过滤非 github.com 主机
2. **工作树 (worktree)**：使用 `findGitRoot` 确保存储的是主仓库路径
3. **空路径数组**：`removePathFromRepo` 会自动清理空数组的仓库键
4. **权限问题**：路径不可读时静默失败

### 改进建议

1. **路径过期机制**
   - 建议：添加时间戳，定期清理长期未访问的路径
   - 实现：在路径对象中添加 `lastAccessed` 字段

2. **路径评分机制**
   - 建议：不仅按时间排序，还考虑访问频率
   - 实现：引入加权评分算法

3. **跨设备同步**
   - 建议：考虑将路径映射与账户同步
   - 注意：需要处理不同设备上的路径差异

4. **模糊匹配**
   - 建议：支持仓库名的模糊搜索
   - 场景：用户只记得部分仓库名时

5. **路径验证优化**
   - 建议：延迟验证或后台验证，避免阻塞 teleport 操作
   - 实现：先返回候选路径，异步验证并更新 UI

### 测试要点

1. 符号链接解析正确性
2. 大小写不敏感匹配
3. 并发更新安全性
4. 无效路径清理
5. 工作树场景处理
