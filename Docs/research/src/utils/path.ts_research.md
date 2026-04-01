# path.ts 研究文档

## 场景与职责

本模块提供路径处理和扩展功能。核心职责包括：

1. **路径扩展**：支持 `~` 家目录展开、相对路径解析、Windows POSIX 路径转换
2. **路径归一化**：统一路径格式，支持 JSON 配置键的跨平台一致性
3. **路径安全检查**：检测路径遍历攻击模式
4. **目录获取**：获取文件或目录的父目录路径
5. **相对路径转换**：将绝对路径转换为相对于 CWD 的路径以节省 token

该模块是文件系统操作的基础工具，被几乎所有涉及路径处理的组件使用。

## 功能点目的

### 1. `expandPath()` - 路径扩展
- **目的**：将各种形式的路径转换为规范化的绝对路径
- **支持形式**：
  - `~` → 家目录
  - `~/path` → 家目录下的路径
  - `./path` 或 `path` → 相对于 baseDir 的路径
  - `/absolute/path` → 已绝对路径，仅规范化
  - `/c/Users/...`（Windows POSIX）→ 转换为 `C:\Users\...`
- **安全特性**：
  - 检查 null 字节（`\0`）
  - NFC Unicode 规范化

### 2. `toRelativePath()` - 相对路径转换
- **目的**：将绝对路径转换为相对于 CWD 的路径
- **用途**：节省 tool output 中的 token
- **边界处理**：如果相对路径会跳出 CWD（以 `..` 开头），保留绝对路径

### 3. `getDirectoryForPath()` - 目录获取
- **目的**：获取文件或目录的所在目录路径
- **逻辑**：
  - 如果是目录，返回自身
  - 如果是文件或不存在，返回父目录
- **安全**：跳过 UNC 路径的文件系统检查（防止 NTLM 凭证泄漏）

### 4. `containsPathTraversal()` - 路径遍历检测
- **目的**：检测路径中的目录遍历模式（`../`、`..\`）
- **正则**：`/(?:^|[\\/])\.\.(?:[\\/]|$)/`
- **用途**：安全检查，防止路径遍历攻击

### 5. `normalizePathForConfigKey()` - 配置键路径归一化
- **目的**：将路径归一化为适合作为 JSON 配置键的格式
- **处理**：
  - 使用 `normalize()` 解析 `.` 和 `..` 段
  - 将所有反斜杠替换为正斜杠
- **用途**：Windows 路径在 JSON 中的跨平台一致性

### 6. `sanitizePath` - 路径清理（重导出）
- **来源**：`./sessionStoragePortable.js`
- **用途**：会话存储相关的路径清理

## 具体技术实现

### 关键流程

#### expandPath 执行流程

```
输入 path, baseDir?
    ↓
baseDir 默认 = getCwd() ?? fs.cwd()
    ↓
安全检查（null 字节）
    ↓
trim() 处理空白
    ↓
处理 ~ → homedir()
    ↓
Windows? 且匹配 /^\/[a-z]\//i ?
    是 → posixPathToWindowsPath() 转换
    ↓
isAbsolute()?
    是 → normalize()
    否 → resolve(baseDir, path)
    ↓
NFC Unicode 规范化
```

### 数据结构

```typescript
// 无自定义类型，使用 string 表示路径

// 关键常量
// Windows POSIX 路径模式：/c/、/d/ 等
/^\/[a-z]\//i

// 路径遍历检测正则
/(?:^|[\\/])\.\.(?:[\\/]|$)/
```

### 跨平台处理

| 平台 | 处理方式 |
|------|----------|
| Unix/Linux | 原生路径处理 |
| macOS | 原生路径处理 |
| Windows | 支持 POSIX 风格路径（/c/Users）转换 |
| WSL | 原生 Linux 路径处理 |

### Windows POSIX 路径转换

```typescript
if (getPlatform() === 'windows' && trimmedPath.match(/^\/[a-z]\//i)) {
  try {
    processedPath = posixPathToWindowsPath(trimmedPath)
    // /c/Users/foo → C:\Users\foo
  } catch {
    processedPath = trimmedPath  // 转换失败使用原路径
  }
}
```

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `os` | `homedir()` 获取家目录 |
| `path` | Node.js 路径操作 |
| `./cwd.js` | 获取当前工作目录 |
| `./fsOperations.js` | 文件系统操作 |
| `./platform.js` | 平台检测 |
| `./windowsPaths.js` | Windows/POSIX 路径转换 |
| `./sessionStoragePortable.js` | `sanitizePath` 重导出 |

### 调用方

| 调用方 | 用途 |
|--------|------|
| `src/memdir/paths.ts` | 内存目录路径处理 |
| `src/tools/BashTool/BashTool.tsx` | Bash 工具路径验证 |
| `src/tools/FileReadTool/FileReadTool.ts` | 文件读取路径处理 |
| `src/tools/FileEditTool/FileEditTool.ts` | 文件编辑路径处理 |
| `src/tools/FileWriteTool/FileWriteTool.ts` | 文件写入路径处理 |
| `src/services/compact/compact.ts` | 压缩功能路径处理 |
| `src/utils/notebook.ts` | Notebook 路径扩展 |
| 以及 20+ 其他模块 | 各种路径处理场景 |

### 相关工具

- `./windowsPaths.ts`：Windows 特定的路径转换
- `./sessionStoragePortable.js`：会话存储路径处理

## 风险、边界与改进建议

### 已知风险

1. **路径遍历攻击**
   - 缓解：`containsPathTraversal()` 检测
   - 限制：仅检测，不阻止，需调用方处理
   - 潜在问题：复杂编码可能绕过

2. **符号链接安全**
   - 风险：`expandPath` 不解析符号链接
   - 影响：可能访问预期外的文件
   - 建议：敏感操作前使用 `fs.realpath()`

3. **Unicode 规范化**
   - 使用 NFC 规范化
   - 风险：某些文件系统使用 NFD（如 macOS HFS+）
   - 潜在问题：相同视觉字符不同编码导致不匹配

4. **UNC 路径处理**
   - `getDirectoryForPath` 跳过 UNC 路径检查
   - 风险：UNC 路径可能触发 NTLM 认证
   - 缓解：明确跳过文件系统操作

### 边界情况

| 场景 | 行为 |
|------|------|
| 空字符串 | 返回 baseDir 的规范化路径 |
| 仅空白字符 | 同空字符串 |
| null 字节 | 抛出错误 "Path contains null bytes" |
| 不存在的路径 | 正常解析，不检查存在性 |
| Windows 驱动器根 | `C:` → `C:\`（normalize 处理） |
| UNC 路径 | `\\server\share` 正常处理 |
| 相对路径跳出 | `../../etc/passwd` 被解析，不阻止 |
| 超长路径 | 依赖 Node.js 处理（>260 字符 Windows 问题） |

### 改进建议

1. **路径遍历自动阻止**
   - 当前：仅检测，返回 boolean
   - 建议：提供 `expandPathSafe()` 自动抛出或清理

2. **符号链接解析选项**
   - 建议：添加 `resolveSymlinks` 参数
   - 实现：使用 `fs.realpath()`

3. **路径长度检查**
   - 建议：添加 Windows 长路径支持（`\\?\` 前缀）
   - 检查：超出平台限制时警告

4. **更多路径形式**
   - 建议：
     - 支持 `%VAR%` 环境变量展开（Windows）
     - 支持 `$VAR` 环境变量展开（Unix）
     - 支持 `file://` URL 转换

5. **缓存优化**
   - 当前：每次调用重新计算
   - 建议：使用 `memoizeWithLRU` 缓存结果
   - 注意：CWD 变化时需失效

6. **类型安全增强**
   - 建议：使用 branded type 区分绝对/相对路径
   - 示例：`type AbsolutePath = string & { __brand: 'absolute' }`

7. **配置键规范化选项**
   - 当前：固定正斜杠
   - 建议：
     - 保留大小写选项（某些文件系统敏感）
     - 驱动器字母大小写统一

8. **遥测集成**
   - 建议：
     - 记录路径扩展频率
     - 记录平台分布
     - 记录异常路径模式

9. **测试覆盖**
   - 建议：
     - 跨平台路径测试矩阵
     - Unicode 路径测试
     - 符号链接场景测试

10. **文档增强**
    - 建议：
      - 添加路径转换示例表格
      - 说明各函数适用场景
      - 安全使用指南
