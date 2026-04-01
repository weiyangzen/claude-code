# idePathConversion.ts 研究文档

## 场景与职责

`idePathConversion.ts` 是 Claude Code CLI 的 **IDE 路径转换工具**，专门处理 WSL（Windows Subsystem for Linux）环境与 Windows IDE 之间的路径格式转换。该模块解决了跨平台开发中的核心痛点：WSL 中的 Linux 路径与 Windows 中的 NTFS 路径之间的互转。

### 核心使用场景

1. **WSL ↔ Windows 路径转换**：
   - IDE（运行在 Windows）报告的工作区路径是 Windows 格式（`C:\Users\...`）
   - Claude Code（运行在 WSL）需要将其转换为 Linux 格式（`/mnt/c/...`）
   - 反之，向 IDE 发送文件路径时需要转换回 Windows 格式

2. **WSL 分发版匹配**：
   - 检查 IDE 报告的路径是否来自正确的 WSL 分发版
   - 避免不同分发版之间的路径混淆（如 Ubuntu 与 Debian）

3. **Diff 显示路径转换**：
   - 在 IDE 中显示 diff 时，需要将 WSL 路径转换为 Windows 路径

---

## 功能点目的

### 1. 路径转换接口 (`IDEPathConverter`)

定义统一的路径转换契约：
```typescript
export interface IDEPathConverter {
  /**
   * 将 IDE 路径（Windows 格式）转换为本地路径（WSL 格式）
   * 用于从 IDE 锁文件读取工作区文件夹
   */
  toLocalPath(idePath: string): string

  /**
   * 将本地路径（WSL 格式）转换为 IDE 路径（Windows 格式）
   * 用于向 IDE 发送路径（如 showDiffInIDE）
   */
  toIDEPath(localPath: string): string
}
```

### 2. Windows 到 WSL 转换器 (`WindowsToWSLConverter`)

**实现策略**：
1. **首选 `wslpath` 命令**：使用 WSL 内置工具进行转换，最准确
2. **回退手动转换**：如果 `wslpath` 失败，使用正则表达式手动转换
3. **分发版匹配检查**：检测 `\\wsl$\{distro}\` 路径，确保与当前分发版匹配

**转换规则**：
- `C:\Users\name\project` → `/mnt/c/Users/name/project`
- `\\wsl$\Ubuntu\home\user` → `/home/user`（同分发版）
- `\\wsl$\Debian\home\user` → 保持原样（不同分发版，wslpath 会失败）

### 3. WSL 分发版匹配检查 (`checkWSLDistroMatch`)

**支持的 UNC 路径格式**：
- `\\wsl$\{distro}\...`（旧格式）
- `\\wsl.localhost\{distro}\...`（新格式）

**用途**：在路径转换前验证路径是否来自当前分发版，避免错误转换。

---

## 具体技术实现

### WindowsToWSLConverter 类

```typescript
export class WindowsToWSLConverter implements IDEPathConverter {
  constructor(private wslDistroName: string | undefined) {}

  toLocalPath(windowsPath: string): string {
    if (!windowsPath) return windowsPath

    // 检查是否来自不同 WSL 分发版
    if (this.wslDistroName) {
      const wslUncMatch = windowsPath.match(
        /^\\\\wsl(?:\.localhost|\$)\\([^\\]+)(.*)$/
      )
      if (wslUncMatch && wslUncMatch[1] !== this.wslDistroName) {
        // 不同分发版 - wslpath 会失败，返回原路径
        return windowsPath
      }
    }

    try {
      // 使用 wslpath 进行转换
      const result = execFileSync('wslpath', ['-u', windowsPath], {
        encoding: 'utf8',
        stdio: ['pipe', 'pipe', 'ignore'],  // 忽略 stderr
      }).trim()
      return result
    } catch {
      // wslpath 失败，回退到手动转换
      return windowsPath
        .replace(/\\/g, '/')                    // 反斜杠转斜杠
        .replace(/^([A-Z]):/i, (_, letter) => `/mnt/${letter.toLowerCase()}`)
    }
  }

  toIDEPath(wslPath: string): string {
    if (!wslPath) return wslPath

    try {
      const result = execFileSync('wslpath', ['-w', wslPath], {
        encoding: 'utf8',
        stdio: ['pipe', 'pipe', 'ignore'],
      }).trim()
      return result
    } catch {
      // wslpath 失败，返回原路径
      return wslPath
    }
  }
}
```

### 分发版匹配检查

```typescript
export function checkWSLDistroMatch(
  windowsPath: string,
  wslDistroName: string,
): boolean {
  const wslUncMatch = windowsPath.match(
    /^\\\\wsl(?:\.localhost|\$)\\([^\\]+)(.*)$/
  )
  if (wslUncMatch) {
    return wslUncMatch[1] === wslDistroName
  }
  return true  // 非 WSL UNC 路径，无分发版不匹配问题
}
```

### 使用示例（来自 ide.ts）

```typescript
// WSL 路径转换场景
if (getPlatform() === 'wsl' && lockfileInfo.runningInWindows) {
  // 检查分发版匹配
  if (!checkWSLDistroMatch(idePath, process.env.WSL_DISTRO_NAME)) {
    return false  // 不同分发版，不匹配
  }

  // 尝试原始路径匹配
  const resolvedOriginal = resolve(localPath).normalize('NFC')
  if (cwd === resolvedOriginal || cwd.startsWith(resolvedOriginal + pathSeparator)) {
    return true
  }

  // 转换为 WSL 本地路径后再次匹配
  const converter = new WindowsToWSLConverter(process.env.WSL_DISTRO_NAME)
  localPath = converter.toLocalPath(idePath)
}
```

---

## 关键代码路径与文件引用

### 导出位置
- **文件**：`src/utils/idePathConversion.ts`
- **导出接口**：
  - `IDEPathConverter` - 路径转换器接口
- **导出类**：
  - `WindowsToWSLConverter` - Windows ↔ WSL 路径转换器
- **导出函数**：
  - `checkWSLDistroMatch()` - WSL 分发版匹配检查

### 调用方分布

| 文件路径 | 使用场景 |
|---------|---------|
| `src/utils/ide.ts` | IDE 工作区路径匹配、WSL 场景处理 |
| `src/hooks/useDiffInIDE.ts` | 向 IDE 发送 diff 路径时的转换 |

### 依赖导入

```typescript
import { execFileSync } from 'child_process'  // 同步执行 wslpath
```

---

## 依赖与外部交互

### Node.js 内置模块

| 模块 | 用途 |
|------|------|
| `child_process` | `execFileSync` - 同步执行 `wslpath` 命令 |

### 外部命令依赖

| 命令 | 用途 | 可用性 |
|------|------|--------|
| `wslpath` | WSL 路径转换 | WSL 环境内置 |

### 被依赖关系

| 模块 | 导入内容 | 用途 |
|------|---------|------|
| `utils/ide.ts` | `WindowsToWSLConverter`, `checkWSLDistroMatch` | IDE 检测路径转换 |
| `hooks/useDiffInIDE.ts` | `WindowsToWSLConverter` | Diff 路径转换 |

---

## 风险、边界与改进建议

### 已知风险

1. **`wslpath` 命令依赖**
   - 仅在 WSL 环境中可用
   - 在纯 Linux 或 macOS 上调用会失败
   - **缓解**：调用前检查 `process.env.WSL_DISTRO_NAME` 或平台类型

2. **同步执行阻塞**
   - 使用 `execFileSync` 同步执行 `wslpath`
   - 如果命令挂起，会阻塞整个进程
   - **缓解**：`wslpath` 是轻量级工具，通常很快返回；超时控制由 Node.js 默认处理

3. **路径格式边缘情况**
   - 网络路径（`\\server\share`）
   - 特殊字符（Unicode、空格、符号链接）
   - **缓解**：`wslpath` 原生处理这些情况；手动回退逻辑可能不完美

4. **不同 WSL 版本差异**
   - WSL1 与 WSL2 的文件系统挂载点不同
   - `wslpath` 行为可能略有差异
   - **缓解**：依赖 `wslpath` 抽象底层差异

### 边界情况

| 场景 | 处理 |
|------|------|
| 空路径 | 直接返回原路径 |
| 非 WSL 环境调用 | 依赖调用方前置检查，模块本身不验证 |
| `wslpath` 命令不存在 | 回退到手动转换（`toLocalPath`）或返回原路径（`toIDEPath`） |
| 不同分发版路径 | `toLocalPath` 返回原路径，不进行转换 |
| 非 UNC WSL 路径 | `checkWSLDistroMatch` 返回 `true`（无冲突） |
| 盘符大小写 | 正则表达式使用 `i` 标志，不区分大小写 |

### 改进建议

1. **添加异步 API**
   ```typescript
   export class WindowsToWSLConverter implements IDEPathConverter {
     async toLocalPathAsync(windowsPath: string): Promise<string> {
       if (!windowsPath) return windowsPath
       try {
         const { stdout } = await execFile('wslpath', ['-u', windowsPath])
         return stdout.trim()
       } catch {
         return this.fallbackConvert(windowsPath)
       }
     }
   }
   ```

2. **增强错误处理**
   ```typescript
   export type PathConversionResult = 
     | { success: true; path: string }
     | { success: false; error: 'wslpath_not_found' | 'different_distro' | 'invalid_path' }
   
   export function toLocalPathSafe(windowsPath: string): PathConversionResult {
     // 详细错误分类
   }
   ```

3. **支持更多路径类型**
   ```typescript
   // 网络路径转换
   if (windowsPath.startsWith('\\\\')) {
     // 处理 UNC 路径（非 WSL）
     return convertUncPath(windowsPath)
   }
   
   // 相对路径处理
   if (!isAbsolute(windowsPath)) {
     throw new Error('Relative paths not supported')
   }
   ```

4. **缓存转换结果**
   ```typescript
   export class WindowsToWSLConverter implements IDEPathConverter {
     private cache = new Map<string, string>()
     
     toLocalPath(windowsPath: string): string {
       if (this.cache.has(windowsPath)) {
         return this.cache.get(windowsPath)!
       }
       const result = this.doConversion(windowsPath)
       this.cache.set(windowsPath, result)
       return result
     }
   }
   ```

5. **单元测试覆盖**
   - 各种 Windows 路径格式（带空格、Unicode、长路径）
   - WSL UNC 路径匹配（新旧格式）
   - `wslpath` 失败时的回退逻辑
   - 不同分发版路径的拒绝逻辑

6. **文档和示例**
   ```typescript
   /**
    * @example
    * ```typescript
    * const converter = new WindowsToWSLConverter('Ubuntu')
    * converter.toLocalPath('C:\\Users\\test')  // '/mnt/c/Users/test'
    * converter.toIDEPath('/home/user')         // '\\wsl$\Ubuntu\home\user'
    * ```
    */
   ```
