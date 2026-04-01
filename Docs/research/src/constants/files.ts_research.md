# files.ts 深度研究文档

## 场景与职责

`files.ts` 是 Claude Code CLI 中定义文件类型检测和处理策略的核心模块。它提供二进制文件识别、二进制内容检测功能，帮助 CLI 避免对非文本文件执行文本操作（如 diff、搜索、编辑）。

### 核心使用场景
1. **文件类型过滤**：在文本搜索、diff 比较前过滤掉二进制文件
2. **内容安全检测**：通过读取文件内容判断是否为二进制，而非仅依赖扩展名
3. **Git 操作优化**：避免将二进制文件纳入文本比较操作
4. **文件读取决策**：决定如何处理用户请求读取的文件

---

## 功能点目的

### 1. 二进制扩展名集合 (`BINARY_EXTENSIONS`)

**功能**：预定义的已知二进制文件扩展名集合，用于快速文件类型判断。

**覆盖类别**：

| 类别 | 扩展名示例 |
|------|-----------|
| **图片** | `.png`, `.jpg`, `.jpeg`, `.gif`, `.bmp`, `.ico`, `.webp`, `.tiff` |
| **视频** | `.mp4`, `.mov`, `.avi`, `.mkv`, `.webm`, `.wmv`, `.flv` |
| **音频** | `.mp3`, `.wav`, `.ogg`, `.flac`, `.aac`, `.m4a`, `.opus` |
| **压缩包** | `.zip`, `.tar`, `.gz`, `.bz2`, `.7z`, `.rar`, `.xz` |
| **可执行文件** | `.exe`, `.dll`, `.so`, `.dylib`, `.bin`, `.app` |
| **文档** | `.pdf`, `.doc`, `.docx`, `.xls`, `.xlsx`, `.ppt`, `.pptx` |
| **字体** | `.ttf`, `.otf`, `.woff`, `.woff2`, `.eot` |
| **字节码** | `.pyc`, `.class`, `.jar`, `.wasm`, `.node` |
| **数据库** | `.sqlite`, `.sqlite3`, `.db`, `.mdb` |
| **设计/3D** | `.psd`, `.ai`, `.sketch`, `.blend`, `.3ds` |
| **其他** | `.swf`, `.lockb`, `.dat` |

**特殊说明**：
- `.pdf` 虽在列表中，但 `FileReadTool` 在调用点单独排除（支持 PDF 读取）

### 2. 二进制扩展名检测 (`hasBinaryExtension`)

**功能**：根据文件路径的扩展名判断是否为二进制文件。

**实现逻辑**：
```typescript
export function hasBinaryExtension(filePath: string): boolean {
  const ext = filePath.slice(filePath.lastIndexOf('.')).toLowerCase()
  return BINARY_EXTENSIONS.has(ext)
}
```

**边界处理**：
- 提取扩展名时包含点号（如 `.png`）
- 统一转换为小写进行大小写不敏感比较

### 3. 二进制内容检测 (`isBinaryContent`)

**功能**：通过分析文件内容判断是否为二进制文件，作为扩展名检测的补充。

**检测策略**：

| 检测方法 | 说明 |
|---------|------|
| **空字节检测** | 发现空字节（`0x00`）即判定为二进制 |
| **非打印字符比例** | 非打印字符超过 10% 判定为二进制 |
| **样本大小** | 最多检查前 8192 字节 |

**可打印字符定义**：
- 可打印 ASCII：32-126
- 常见空白字符：Tab (9), LF (10), CR (13)

**实现逻辑**：
```typescript
export function isBinaryContent(buffer: Buffer): boolean {
  const checkSize = Math.min(buffer.length, BINARY_CHECK_SIZE) // 8192
  let nonPrintable = 0
  
  for (let i = 0; i < checkSize; i++) {
    const byte = buffer[i]!
    
    // 空字节 = 立即判定为二进制
    if (byte === 0) return true
    
    // 统计非打印字符
    if (byte < 32 && byte !== 9 && byte !== 10 && byte !== 13) {
      nonPrintable++
    }
  }
  
  // 超过 10% 非打印字符 = 二进制
  return nonPrintable / checkSize > 0.1
}
```

---

## 具体技术实现

### 数据结构

```typescript
// 二进制扩展名集合（大小写不敏感）
export const BINARY_EXTENSIONS = new Set([
  '.png', '.jpg', '.jpeg', '.gif', '.bmp', '.ico', '.webp', '.tiff', '.tif',
  '.mp4', '.mov', '.avi', '.mkv', '.webm', '.wmv', '.flv', '.m4v', '.mpeg', '.mpg',
  '.mp3', '.wav', '.ogg', '.flac', '.aac', '.m4a', '.wma', '.aiff', '.opus',
  '.zip', '.tar', '.gz', '.bz2', '.7z', '.rar', '.xz', '.z', '.tgz', '.iso',
  '.exe', '.dll', '.so', '.dylib', '.bin', '.o', '.a', '.obj', '.lib', '.app', '.msi', '.deb', '.rpm',
  '.pdf', '.doc', '.docx', '.xls', '.xlsx', '.ppt', '.pptx', '.odt', '.ods', '.odp',
  '.ttf', '.otf', '.woff', '.woff2', '.eot',
  '.pyc', '.pyo', '.class', '.jar', '.war', '.ear', '.node', '.wasm', '.rlib',
  '.sqlite', '.sqlite3', '.db', '.mdb', '.idx',
  '.psd', '.ai', '.eps', '.sketch', '.fig', '.xd', '.blend', '.3ds', '.max',
  '.swf', '.fla',
  '.lockb', '.dat', '.data',
])

// 检测函数
export function hasBinaryExtension(filePath: string): boolean
export function isBinaryContent(buffer: Buffer): boolean
```

### 关键代码路径

#### 1. Git 差异检测路径

```
执行 Git 差异比较
    ↓
src/utils/git.ts
    ↓
调用 hasBinaryExtension() 过滤文件
    ↓
跳过二进制文件，仅对文本文件执行 diff
```

**关键文件引用**：
- `src/utils/git.ts`: Git 工具函数，使用 `hasBinaryExtension`

#### 2. 文件读取路径

```
用户请求读取文件
    ↓
src/tools/FileReadTool/FileReadTool.ts
    ↓
检查文件扩展名（.pdf 特殊处理）
    ↓
[非 .pdf 且二进制扩展名] → 拒绝或警告
[.pdf 或其他] → 继续处理
```

**关键文件引用**：
- `src/tools/FileReadTool/FileReadTool.ts`: 文件读取工具

#### 3. Web 获取路径

```
获取远程文件
    ↓
src/tools/WebFetchTool/utils.ts
    ↓
使用 isBinaryContent() 检测内容类型
    ↓
决定如何处理响应内容
```

**关键文件引用**：
- `src/tools/WebFetchTool/utils.ts`: Web 获取工具

#### 4. MCP 输出存储路径

```
存储 MCP 工具输出
    ↓
src/utils/mcpOutputStorage.ts
    ↓
使用 hasBinaryExtension() 判断存储策略
    ↓
选择合适的存储格式
```

**关键文件引用**：
- `src/utils/mcpOutputStorage.ts`: MCP 输出存储

---

## 依赖与外部交互

### 内部依赖

**零依赖**：此文件不导入任何其他模块。

### 被依赖方

| 文件 | 使用的函数/常量 | 用途 |
|------|----------------|------|
| `src/utils/git.ts` | `hasBinaryExtension` | Git 差异过滤 |
| `src/tools/FileReadTool/FileReadTool.ts` | `BINARY_EXTENSIONS` | 文件读取决策 |
| `src/tools/WebFetchTool/utils.ts` | `isBinaryContent` | 远程内容检测 |
| `src/utils/mcpOutputStorage.ts` | `hasBinaryExtension` | MCP 输出存储 |

---

## 风险、边界与改进建议

### 当前风险

1. **扩展名欺骗**
   - 仅依赖扩展名可能被欺骗（如将 `.exe` 重命名为 `.txt`）
   - `isBinaryContent()` 是更好的检测方式，但并非所有调用点都使用

2. **Unicode 文本误判**
   - UTF-16 或其他多字节编码的文本文件可能含有大量"非打印"字节
   - 10% 阈值可能过于激进或保守

3. **扩展名遗漏**
   - 新文件类型不断出现，列表需要持续维护
   - 某些小众格式可能未被覆盖

4. **PDF 特殊处理分散**
   - `.pdf` 在 `BINARY_EXTENSIONS` 中，但 `FileReadTool` 单独排除
   - 这种特殊处理分散在调用点，容易遗漏或冲突

### 边界情况

| 场景 | 行为 |
|------|------|
| 无扩展名文件 | `hasBinaryExtension` 返回 true（`slice` 返回整个路径） |
| 隐藏文件（如 `.gitignore`） | 正确识别，点号在开头不是扩展名分隔符 |
| 空文件 | `isBinaryContent` 返回 false（0% 非打印字符） |
| 全空字节文件 | `isBinaryContent` 在第一个字节返回 true |
| UTF-8 BOM | 取决于 BOM 字节是否被识别为"非打印" |

### 改进建议

1. **MIME 类型检测**
   ```typescript
   // 建议添加基于内容的 MIME 检测
   import { fileTypeFromBuffer } from 'file-type'
   
   export async function getFileType(buffer: Buffer): Promise<FileTypeResult> {
     return await fileTypeFromBuffer(buffer)
   }
   ```

2. **分层检测策略**
   ```typescript
   // 建议实现优先级检测
   export async function detectFileType(filePath: string): Promise<FileType> {
     // 1. 检查扩展名（最快）
     if (hasBinaryExtension(filePath)) return 'binary'
     
     // 2. 读取内容样本
     const buffer = await readFileChunk(filePath, BINARY_CHECK_SIZE)
     
     // 3. 内容分析
     if (isBinaryContent(buffer)) return 'binary'
     
     // 4. 编码检测
     return detectEncoding(buffer)
   }
   ```

3. **扩展名列表维护**
   ```typescript
   // 建议按类别组织，便于维护
   export const BINARY_EXTENSIONS_BY_CATEGORY = {
     images: ['.png', '.jpg', '.jpeg', '.gif', '.bmp', '.ico', '.webp', '.tiff', '.tif'],
     videos: ['.mp4', '.mov', '.avi', '.mkv', '.webm', '.wmv', '.flv', '.m4v', '.mpeg', '.mpg'],
     // ...
   }
   
   // 自动生成完整集合
   export const BINARY_EXTENSIONS = new Set(
     Object.values(BINARY_EXTENSIONS_BY_CATEGORY).flat()
   )
   ```

4. **PDF 处理统一**
   ```typescript
   // 建议明确定义可读取的二进制类型
   export const READABLE_BINARY_EXTENSIONS = new Set(['.pdf'])
   
   export function isBinaryButReadable(filePath: string): boolean {
     return READABLE_BINARY_EXTENSIONS.has(getExtension(filePath))
   }
   ```

5. **性能优化**
   ```typescript
   // 建议缓存频繁访问的文件类型判断
   const fileTypeCache = new Map<string, boolean>()
   
   export function hasBinaryExtensionCached(filePath: string): boolean {
     if (fileTypeCache.has(filePath)) {
       return fileTypeCache.get(filePath)!
     }
     const result = hasBinaryExtension(filePath)
     fileTypeCache.set(filePath, result)
     return result
   }
   ```

6. **测试覆盖**
   - 各种编码格式的文本文件（UTF-8, UTF-16, GBK, etc.）
   - 边界大小的文件（恰好 8192 字节）
   - 各种二进制文件头（magic numbers）

### 与 Git 的集成

```
files.ts (文件类型检测)
    ↓ 被使用
utils/git.ts (Git 操作)
    ↓ 调用
Git 命令（diff, show, etc.）
    ↓
过滤后的文本输出
```

文件类型检测是 Git 集成的重要前置步骤，确保不会尝试对二进制文件执行文本操作。
