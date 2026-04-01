# windowsPaths.ts 研究文档

## 场景与职责

`windowsPaths.ts` 是 Claude Code CLI 的 Windows 平台专用工具模块，提供以下核心功能：

1. **Git Bash 路径查找**：在 Windows 上定位 Git Bash 安装位置
2. **SHELL 环境设置**：为 BashTool 和 Shell.ts 设置正确的 shell 路径
3. **路径格式转换**：Windows 路径与 POSIX 路径互转（支持 Git Bash/Cygwin/MSYS2 格式）

**核心使用场景：**
- Windows 启动时设置 `SHELL` 环境变量指向 Git Bash
- 路径转换以支持 Windows 上的 POSIX 兼容 shell
- 安全地查找 git 可执行文件（防止当前目录恶意程序）

## 功能点目的

### 1. Git Bash 路径查找 (`findGitBashPath`)
- **目的**：找到 Windows 上 Git Bash 的 `bash.exe` 路径
- **实现**：
  1. 优先检查 `CLAUDE_CODE_GIT_BASH_PATH` 环境变量
  2. 查找 git 安装位置，推导 `../bin/bash.exe`
  3. 找不到则退出进程并显示安装指引

### 2. SHELL 环境设置 (`setShellIfWindows`)
- **目的**：在 Windows 上设置 `SHELL` 环境变量
- **实现**：调用 `findGitBashPath()` 并将结果设置到 `process.env.SHELL`

### 3. 可执行文件查找 (`findExecutable`)
- **目的**：安全地查找可执行文件（特别是 git）
- **安全特性**：
  - 优先检查默认安装位置（`C:\Program Files\Git\cmd\git.exe`）
  - 使用 `where.exe` 时过滤掉当前目录的结果（防止恶意程序）

### 4. 路径格式转换
- **`windowsPathToPosixPath`**：Windows 路径 → POSIX 路径（Git Bash 格式）
- **`posixPathToWindowsPath`**：POSIX 路径 → Windows 路径

## 具体技术实现

### 关键流程

#### findGitBashPath 流程
```
findGitBashPath() → string (memoized)
├── 检查 CLAUDE_CODE_GIT_BASH_PATH 环境变量
│   └── 如果存在且有效 → 返回该路径
│   └── 如果存在但无效 → 报错退出
├── 调用 findExecutable('git')
│   └── 优先检查默认位置（64位 → 32位）
│   └── 回退到 where.exe
├── 从 git 路径推导 bash 路径
│   └── gitPath/../bin/bash.exe
└── 验证 bash 存在 → 返回路径
    └── 不存在 → 报错退出
```

#### 路径转换规则

**Windows → POSIX：**
| 输入 | 输出 | 说明 |
|------|------|------|
| `\\server\share` | `//server/share` | UNC 路径 |
| `C:\Users\foo` | `/c/Users/foo` | 盘符路径 |
| `path\to\file` | `path/to/file` | 相对路径 |

**POSIX → Windows：**
| 输入 | 输出 | 说明 |
|------|------|------|
| `//server/share` | `\\server\share` | UNC 路径 |
| `/cygdrive/c/...` | `C:\...` | Cygwin 格式 |
| `/c/...` | `C:\...` | MSYS2/Git Bash 格式 |

### 数据结构

```typescript
// 路径转换函数使用 LRU 缓存（容量 500）
export const windowsPathToPosixPath = memoizeWithLRU(
  (windowsPath: string): string => { ... },
  (p: string) => p,
  500,
)

export const posixPathToWindowsPath = memoizeWithLRU(
  (posixPath: string): string => { ... },
  (p: string) => p,
  500,
)
```

### 安全机制

```typescript
// findExecutable 中的安全检查
for (const candidatePath of paths) {
  const normalizedPath = path.resolve(candidatePath).toLowerCase()
  const pathDir = path.dirname(normalizedPath).toLowerCase()
  
  // 跳过当前目录中的可执行文件（防止恶意程序）
  if (pathDir === cwd || normalizedPath.startsWith(cwd + path.sep)) {
    logForDebugging(`Skipping potentially malicious executable...`)
    continue
  }
  return candidatePath
}
```

## 关键代码路径与文件引用

### 导出函数
- `src/utils/windowsPaths.ts:87` - `setShellIfWindows()`
- `src/utils/windowsPaths.ts:98` - `findGitBashPath()` (memoized)
- `src/utils/windowsPaths.ts:128` - `windowsPathToPosixPath()` (LRU memoized)
- `src/utils/windowsPaths.ts:148` - `posixPathToWindowsPath()` (LRU memoized)

### 内部函数
- `src/utils/windowsPaths.ts:15` - `checkPathExists()` - 使用 `dir` 命令检查路径
- `src/utils/windowsPaths.ts:29` - `findExecutable()` - 安全查找可执行文件

### 依赖文件
| 文件 | 导入内容 |
|------|----------|
| `src/utils/cwd.ts` | `getCwd()` |
| `src/utils/debug.ts` | `logForDebugging()` |
| `src/utils/execSyncWrapper.ts` | `execSync_DEPRECATED` |
| `src/utils/memoize.ts` | `memoizeWithLRU()` |
| `src/utils/platform.ts` | `getPlatform()` |

### 调用方
| 文件 | 使用方式 |
|------|----------|
| `src/utils/init.ts` | 启动时调用 `setShellIfWindows()` |
| `src/utils/hooks.ts` | Hook 执行时的路径转换 |

## 依赖与外部交互

### 外部依赖
| 依赖 | 用途 |
|------|------|
| `lodash-es/memoize.js` | 基础 memoization |
| `path` | 路径操作 |
| `path/win32` | Windows 专用路径操作 |

### 环境变量
| 变量 | 用途 |
|------|------|
| `CLAUDE_CODE_GIT_BASH_PATH` | 用户指定的 Git Bash 路径 |
| `SHELL` | 被设置的 shell 路径 |

## 风险、边界与改进建议

### 已知风险

1. **硬编码路径依赖**
   - 默认 git 路径硬编码为 `C:\Program Files\Git` 和 `C:\Program Files (x86)\Git`
   - 如果安装在其他位置，依赖 `where.exe` 查找

2. **进程退出风险**
   - `findGitBashPath` 找不到 bash 时会调用 `process.exit(1)`
   - 这是设计行为，但会强制终止整个 CLI

3. **大小写敏感问题**
   - 路径比较使用 `toLowerCase()`，在土耳其语等 locale 下可能有问题

### 边界情况

1. **Git 安装但 Bash 未安装**
   - 某些精简版 Git 安装可能不包含 Bash
   - 会正确检测并提示用户

2. **路径包含特殊字符**
   - 路径转换使用纯 JS 正则，未考虑所有边缘情况

3. **UNC 路径处理**
   - 正确保留 UNC 路径格式（`\\server\share`）

### 改进建议

1. **配置化默认路径**
   - 允许通过配置文件指定额外的 git 安装位置搜索路径

2. **优雅降级**
   - 考虑在找不到 Bash 时使用 cmd.exe 作为降级方案（功能受限但可用）

3. **缓存持久化**
   - 考虑将找到的 bash 路径缓存到配置文件中，避免每次启动重新查找

4. **路径验证增强**
   - 添加对找到的 bash.exe 的实际执行测试，确保其可用

5. **WSL 支持**
   - 考虑检测 WSL 环境，优先使用 WSL bash（如果可用）
