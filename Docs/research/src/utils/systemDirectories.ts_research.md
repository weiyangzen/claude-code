# systemDirectories.ts 研究文档

## 场景与职责

`systemDirectories.ts` 提供了跨平台的系统目录路径获取功能。该模块处理 Windows、macOS、Linux 和 WSL 之间的差异，为应用提供一致的标准目录路径（如桌面、文档、下载等）。

## 功能点目的

### 跨平台目录标准化
- **问题**: 不同操作系统有不同的标准目录结构和环境变量
- **解决方案**: 提供统一的 `getSystemDirectories()` 函数，根据平台返回正确的路径
- **支持平台**: Windows、macOS、Linux、WSL

### XDG 规范支持
- **Linux/WSL**: 支持 XDG Base Directory 规范
- **环境变量**: 检查 `XDG_*_DIR` 环境变量
- **回退**: 使用默认路径作为后备

## 具体技术实现

### 核心类型定义

```typescript
export type SystemDirectories = {
  HOME: string
  DESKTOP: string
  DOCUMENTS: string
  DOWNLOADS: string
  [key: string]: string  // 索引签名用于兼容性
}

type SystemDirectoriesOptions = {
  env?: EnvLike
  homedir?: string
  platform?: Platform
}
```

### 平台特定实现

#### Windows
```typescript
case 'windows': {
  const userProfile = env.USERPROFILE || homeDir
  return {
    HOME: homeDir,
    DESKTOP: join(userProfile, 'Desktop'),
    DOCUMENTS: join(userProfile, 'Documents'),
    DOWNLOADS: join(userProfile, 'Downloads'),
  }
}
```

#### Linux/WSL
```typescript
case 'linux':
case 'wsl': {
  return {
    HOME: homeDir,
    DESKTOP: env.XDG_DESKTOP_DIR || defaults.DESKTOP,
    DOCUMENTS: env.XDG_DOCUMENTS_DIR || defaults.DOCUMENTS,
    DOWNLOADS: env.XDG_DOWNLOAD_DIR || defaults.DOWNLOADS,
  }
}
```

#### macOS/默认
```typescript
case 'macos':
default: {
  if (platform === 'unknown') {
    logForDebugging(`Unknown platform detected, using default paths`)
  }
  return defaults
}
```

### 默认路径

```typescript
const defaults: SystemDirectories = {
  HOME: homeDir,
  DESKTOP: join(homeDir, 'Desktop'),
  DOCUMENTS: join(homeDir, 'Documents'),
  DOWNLOADS: join(homeDir, 'Downloads'),
}
```

## 关键代码路径与文件引用

### 本文件导出
- `getSystemDirectories(options?)`: 获取系统目录
- `SystemDirectories`: 类型定义

### 依赖模块

| 模块 | 用途 |
|------|------|
| `os` | `homedir()` |
| `path` | `join` |
| `./debug.js` | `logForDebugging` |
| `./platform.js` | `getPlatform`, `Platform` |

### 调用方

| 文件 | 用途 |
|------|------|
| `src/utils/plugins/mcpbHandler.ts` | MCP 插件处理 |

## 依赖与外部交互

### 与平台检测的集成
- 使用 `getPlatform()` 检测当前平台
- 区分 Linux 和 WSL（通过 `/proc/version` 检查）

### 与环境变量的集成
- Windows: 使用 `USERPROFILE`
- Linux/WSL: 使用 `XDG_*_DIR` 系列变量

### 测试支持
- `SystemDirectoriesOptions` 允许注入测试值
- 可以覆盖 `env`、`homedir` 和 `platform`

## 风险、边界与改进建议

### 潜在风险

1. **路径不存在**: 返回的路径可能不存在（如用户删除了 Desktop 目录）
2. **权限问题**: 某些路径可能无法访问
3. **本地化目录名**: Windows 可能使用本地化的目录名（如 "桌面" 而非 "Desktop"）

### 边界情况

1. **未知平台**: 使用默认路径并记录调试日志
2. **缺失环境变量**: 使用默认路径作为后备
3. **空 homeDir**: 依赖 `os.homedir()` 的返回值

### 改进建议

1. **目录存在性检查**: 添加可选的存在性验证
```typescript
export async function getSystemDirectories(
  options?: SystemDirectoriesOptions & { verifyExists?: boolean }
): Promise<SystemDirectories> {
  const dirs = /* ... */
  if (options?.verifyExists) {
    for (const [key, path] of Object.entries(dirs)) {
      try {
        await access(path)
      } catch {
        dirs[key] = ''  // 或尝试创建
      }
    }
  }
  return dirs
}
```

2. **更多标准目录**: 扩展支持更多 XDG 目录
```typescript
export type SystemDirectories = {
  // ... existing
  MUSIC?: string
  PICTURES?: string
  VIDEOS?: string
  TEMPLATES?: string
  PUBLICSHARE?: string
}
```

3. **本地化支持**: 处理 Windows 本地化目录名
```typescript
// 使用 Windows API 或注册表获取实际路径
function getWindowsKnownFolder(folderId: string): string {
  // 实现...
}
```

4. **缓存机制**: 目录路径通常不变，可以缓存
```typescript
const cachedDirs = memoize(getSystemDirectories)
```

5. **类型安全**: 使用更严格的类型
```typescript
type KnownDirectory = 'HOME' | 'DESKTOP' | 'DOCUMENTS' | 'DOWNLOADS'
export type SystemDirectories = Record<KnownDirectory, string>
```
