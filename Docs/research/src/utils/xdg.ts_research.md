# xdg.ts 研究文档

## 场景与职责

`xdg.ts` 实现 XDG Base Directory 规范，用于在类 Unix 系统上组织应用程序数据、配置和缓存文件。这是 Claude Code CLI 原生安装器（native installer）的组成部分。

**XDG Base Directory 规范：**
- 定义了标准目录结构，避免在用户主目录下创建大量隐藏文件（dotfiles）
- 通过环境变量允许用户自定义位置
- 被大多数现代 Linux 桌面应用遵循

**核心使用场景：**
- 确定状态文件存储位置（`~/.local/state`）
- 确定缓存目录位置（`~/.cache`）
- 确定数据文件位置（`~/.local/share`）
- 确定用户二进制文件位置（`~/.local/bin`）

## 功能点目的

### 1. XDG State Home (`getXDGStateHome`)
- **默认**：`~/.local/state`
- **环境变量**：`$XDG_STATE_HOME`
- **用途**：存储应用程序状态文件（非日志、非配置）

### 2. XDG Cache Home (`getXDGCacheHome`)
- **默认**：`~/.cache`
- **环境变量**：`$XDG_CACHE_HOME`
- **用途**：存储可重新生成的缓存数据

### 3. XDG Data Home (`getXDGDataHome`)
- **默认**：`~/.local/share`
- **环境变量**：`$XDG_DATA_HOME`
- **用途**：存储应用程序数据文件

### 4. User Bin Directory (`getUserBinDir`)
- **默认**：`~/.local/bin`
- **用途**：存储用户级可执行文件（非 XDG 标准，但遵循相同约定）

## 具体技术实现

### 类型定义

```typescript
type EnvLike = Record<string, string | undefined>

type XDGOptions = {
  env?: EnvLike    // 可选的环境变量覆盖（用于测试）
  homedir?: string // 可选的主目录覆盖（用于测试）
}
```

### 选项解析

```typescript
function resolveOptions(options?: XDGOptions): { env: EnvLike; home: string } {
  return {
    env: options?.env ?? process.env,
    home: options?.homedir ?? process.env.HOME ?? osHomedir(),
  }
}
```

### 目录解析流程

```
getXDGStateHome(options?) → string
├── resolveOptions(options)
│   ├── env = options.env ?? process.env
│   └── home = options.homedir ?? process.env.HOME ?? os.homedir()
└── return env.XDG_STATE_HOME ?? join(home, '.local', 'state')

getXDGCacheHome(options?) → string
└── return env.XDG_CACHE_HOME ?? join(home, '.cache')

getXDGDataHome(options?) → string
└── return env.XDG_DATA_HOME ?? join(home, '.local', 'share')

getUserBinDir(options?) → string
└── return join(home, '.local', 'bin')
```

## 关键代码路径与文件引用

### 导出函数
- `src/utils/xdg.ts:32` - `getXDGStateHome(options?)`
- `src/utils/xdg.ts:42` - `getXDGCacheHome(options?)`
- `src/utils/xdg.ts:52` - `getXDGDataHome(options?)`
- `src/utils/xdg.ts:62` - `getUserBinDir(options?)`

### 依赖
| 依赖 | 用途 |
|------|------|
| `os` (Node.js) | `homedir()` 函数 |
| `path` (Node.js) | `join()` 函数 |

## 依赖与外部交互

### 外部依赖
```typescript
import { homedir as osHomedir } from 'os'
import { join } from 'path'
```

### 环境变量
| 变量 | 用途 |
|------|------|
| `XDG_STATE_HOME` | 状态目录覆盖 |
| `XDG_CACHE_HOME` | 缓存目录覆盖 |
| `XDG_DATA_HOME` | 数据目录覆盖 |
| `HOME` | 主目录（fallback） |

### 规范参考
- [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/)

## 风险、边界与改进建议

### 已知风险

1. **Windows 兼容性**
   - XDG 规范主要针对类 Unix 系统
   - Windows 上默认路径（`~/.local/state` 等）可能不符合用户预期
   - 调用方需要根据平台选择使用 XDG 路径或 Windows 标准路径

2. **路径不存在**
   - 函数只返回路径字符串，不保证目录存在
   - 调用方需要自行创建目录

3. **HOME 未设置**
   - 如果 `HOME` 环境变量未设置且 `os.homedir()` 失败，会抛出错误
   - 这在某些容器环境中可能发生

### 边界情况

1. **空环境变量**
   - 如果 `XDG_*` 环境变量设置为空字符串，会被视为有效值
   - 可能导致返回空字符串或相对路径

2. **相对路径**
   - 如果 `XDG_*` 环境变量设置为相对路径，会直接返回
   - 调用方需要处理路径解析

3. **测试注入**
   - `XDGOptions` 允许注入自定义 `env` 和 `homedir`
   - 便于单元测试，但生产代码不应滥用

### 改进建议

1. **目录自动创建**
   ```typescript
   export async function ensureXDGStateHome(options?: XDGOptions): Promise<string> {
     const dir = getXDGStateHome(options)
     await mkdir(dir, { recursive: true })
     return dir
   }
   ```

2. **路径验证**
   - 验证返回的路径是绝对路径
   - 处理空字符串和相对路径的情况

3. **Windows 支持**
   - 考虑添加 `getWindowsAppDataDir()` 等函数
   - 或者提供跨平台的统一接口

4. **XDG Config Home**
   - 添加 `getXDGConfigHome()` 函数（`~/.config`）
   - 这是 XDG 规范的重要组成部分

5. **XDG 运行时目录**
   - 添加 `getXDGRuntimeDir()` 函数（`$XDG_RUNTIME_DIR`）
   - 用于存储运行时文件（如 socket、pid 文件）

6. **规范合规检查**
   - 添加测试验证默认路径符合 XDG 规范
   - 验证环境变量优先级正确
