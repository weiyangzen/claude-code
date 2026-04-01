# Research: src/utils/memory/versions.ts

## 场景与职责

该文件提供与 Git 仓库版本控制相关的记忆系统辅助函数。目前主要功能是检测当前工作目录是否位于 Git 仓库中，用于决定记忆文件的描述文本（如 "Checked in at ./CLAUDE.md" vs "Saved in ./CLAUDE.md"）。

这是一个轻量级的工具模块，将 Git 相关的版本控制逻辑从 UI 组件中抽离，保持关注点分离。

## 功能点目的

1. **Git 仓库状态检测**：提供同步函数 `projectIsInGitRepo(cwd: string)`，快速判断指定目录是否在 Git 仓库中。

2. **UI 文本适配**：根据 Git 状态决定记忆文件选择器中的描述文本：
   - Git 仓库中：显示 "Checked in at ./CLAUDE.md"（暗示文件可被版本控制）
   - 非 Git 目录：显示 "Saved in ./CLAUDE.md"（普通文件保存）

3. **同步检测优化**：注释明确指出该函数用于同步检查场景，优先使用 `findGitRoot`（文件系统遍历）而非异步的 `dirIsInGitRepo`。

## 具体技术实现

### 核心函数

```typescript
export function projectIsInGitRepo(cwd: string): boolean {
  return findGitRoot(cwd) !== null
}
```

### 实现细节

- **同步执行**：直接返回 `findGitRoot` 的结果比较，无 async/await 开销
- **布尔返回值**：将 `findGitRoot` 的 `string | null` 结果转换为布尔值
- **依赖注入**：通过 `cwd` 参数接收工作目录，不依赖全局状态

### 被调用方使用模式

在 `MemoryFileSelector.tsx` 中的使用：

```typescript
const isGit = projectIsInGitRepo(getOriginalCwd())
// ...
description = isGit ? "Checked in at ./CLAUDE.md" : "Saved in ./CLAUDE.md"
```

## 关键代码路径与文件引用

### 被以下文件引用

| 文件路径 | 引用方式 | 具体用途 |
|---------|---------|---------|
| `src/components/memory/MemoryFileSelector.tsx` | `import { projectIsInGitRepo } from '../../utils/memory/versions.js'` | 记忆文件选择器 UI 中根据 Git 状态显示不同描述文本 |

### 依赖的文件

| 文件路径 | 导入内容 | 用途 |
|---------|---------|------|
| `src/utils/git.ts` | `findGitRoot` | 核心 Git 仓库检测逻辑 |

### 相关函数对比

| 函数 | 位置 | 同步/异步 | 用途 |
|-----|------|----------|------|
| `projectIsInGitRepo` | 本文件 | 同步 | UI 快速检测 |
| `dirIsInGitRepo` | `src/utils/git.ts` | 异步 | 通用异步检测 |
| `findGitRoot` | `src/utils/git.ts` | 同步 | 底层实现，返回路径或 null |
| `getIsGit` | `src/utils/git.ts` | 异步 | 带缓存和诊断日志的检测 |

## 依赖与外部交互

### 直接依赖

```typescript
import { findGitRoot } from '../git.js'
```

### `findGitRoot` 实现概要

`src/utils/git.ts` 中的 `findGitRoot` 函数：

1. **LRU 缓存**：使用 `memoizeWithLRU` 缓存最多 50 个路径的结果
2. **向上遍历**：从起始路径向上遍历目录树，查找 `.git` 目录或文件
3. **Worktree 支持**：正确处理 `.git` 为文件的情况（worktree/submodule）
4. **NFC 规范化**：返回的路径进行 Unicode NFC 规范化

```typescript
const findGitRootImpl = memoizeWithLRU(
  (startPath: string): string | typeof GIT_ROOT_NOT_FOUND => {
    // 向上遍历目录树查找 .git
    while (current !== root) {
      try {
        const gitPath = join(current, '.git')
        const stat = statSync(gitPath)
        if (stat.isDirectory() || stat.isFile()) {
          return current.normalize('NFC')
        }
      } catch {
        // .git 不存在，继续向上
      }
      current = dirname(current)
    }
    return GIT_ROOT_NOT_FOUND
  },
  path => path,
  50,
)
```

## 风险、边界与改进建议

### 已知风险

1. **功能单一性**：当前文件仅包含一个简单函数，可能存在过度抽象的风险。但考虑到 Git 相关工具函数可能未来扩展，单独成文件也有合理性。

2. **同步 I/O**：`findGitRoot` 使用 `statSync` 进行同步文件系统遍历，虽然通过 LRU 缓存缓解，但在缓存未命中时仍可能阻塞事件循环。

3. **调用频率**：`MemoryFileSelector` 组件在渲染时调用此函数，如果组件频繁重渲染且缓存未命中，可能造成性能问题。

### 边界情况

1. **Worktree 场景**：`.git` 可能是文件而非目录（指向实际 git 目录），`findGitRoot` 正确处理这种情况。

2. **符号链接**：`findGitRoot` 不解析符号链接，如果 cwd 是 symlink，可能返回 symlink 所在目录而非真实路径。

3. **权限问题**：如果 `.git` 存在但无权限访问，`statSync` 会抛出异常，被 catch 块捕获后继续向上遍历。

### 改进建议

1. **函数扩展**：当前函数仅返回布尔值，可考虑扩展为返回更多版本控制信息：
   ```typescript
   interface ProjectVersionInfo {
     isGitRepo: boolean
     gitRoot: string | null
     hasUncommittedChanges: boolean
     currentBranch: string | null
   }
   ```

2. **缓存预热**：在应用启动时预热 `findGitRoot` 缓存，避免 UI 首次渲染时的同步 I/O。

3. **异步迁移**：考虑为 UI 组件提供异步版本，避免阻塞渲染：
   ```typescript
   export async function projectIsInGitRepoAsync(cwd: string): Promise<boolean> {
     return dirIsInGitRepo(cwd) // 使用 git.ts 中的异步版本
   }
   ```

4. **与 memdir 集成**：未来自动记忆（AutoMem）或团队记忆（TeamMem）功能可能需要基于 Git 状态的更复杂逻辑，此文件可作为扩展点。

5. **测试覆盖**：当前简单函数可能缺乏单元测试，建议添加：
   - Git 仓库内目录的检测
   - Git 仓库外目录的检测  
   - Worktree 场景
   - 嵌套 Git 仓库（子模块）场景
