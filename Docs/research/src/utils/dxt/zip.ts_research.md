# zip.ts 深度研究文档

## 1. 场景与职责

### 1.1 模块定位

`zip.ts` 是 Claude Code 插件系统中 ZIP 文件处理的核心安全模块，负责 **ZIP 文件解压**、**ZIP 炸弹防护** 以及 **Unix 文件权限恢复**。该模块是 DXT/MCPB 扩展包格式和插件 ZIP 缓存功能的基础设施。

### 1.2 使用场景

| 场景 | 描述 |
|------|------|
| MCPB 扩展包解压 | 用户加载 `.mcpb` 文件时解压内容到缓存目录 |
| 官方市场 GCS 下载 | 从 GCS 下载官方市场 ZIP 并解压 |
| 插件 ZIP 缓存 | 将插件以 ZIP 形式存储在挂载目录，会话时解压到临时目录 |
| ZIP 权限恢复 | 恢复可执行文件的 `+x` 权限（fflate 默认丢失此信息） |

### 1.3 核心职责

1. **安全解压**：防止 ZIP 炸弹、路径遍历等攻击
2. **权限恢复**：从 ZIP 中央目录解析 Unix 文件模式，恢复可执行权限
3. **资源限制**：文件大小、数量、压缩比的硬性限制
4. **文件系统抽象**：通过 `FsOperations` 接口支持不同运行环境

---

## 2. 功能点目的

### 2.1 unzipFile - 安全解压

**目的**：将 ZIP 数据解压为文件路径到内容的映射，同时实施安全防护。

**安全特性**：
- 路径遍历检测（`../` 等）
- 绝对路径拒绝
- 文件数量限制（100,000）
- 单文件大小限制（512MB）
- 总解压大小限制（1GB）
- 压缩比检测（50:1 为可疑，0.5:1 为可能恶意）

**技术选择**：
- 使用 `fflate` 库的 `unzipSync` 而非异步版本
- 原因：避免 Bun 环境下 worker 终止导致的崩溃（见注释）

### 2.2 readAndUnzipFile - 文件系统解压

**目的**：从磁盘读取 ZIP 文件并解压，提供统一的错误处理。

**错误处理**：
- `ENOENT`：文件不存在，原样抛出
- 其他错误：包装为 "Failed to read or unzip file: {message}"

### 2.3 parseZipModes - Unix 权限解析

**目的**：从 ZIP 中央目录解析 Unix 文件模式（特别是可执行位）。

**背景问题**：
- `fflate` 的 `unzipSync` 只返回 `Record<string, Uint8Array>`
- 不暴露中央目录中的 `external file attributes`
- 导致可执行位丢失（所有文件变为 0644）

**解决方案**：
- 手动解析 ZIP 中央目录
- 提取 `versionMadeBy` 和 `externalAttr` 字段
- 对于 Unix 创建的 ZIP（host OS = 3），从高 16 位提取 `st_mode`

### 2.4 isPathSafe - 路径安全检测

**目的**：防止 ZIP 路径遍历攻击。

**检测规则**：
1. 包含 `../` 或 `..\` 模式
2. 路径规范化后为绝对路径

### 2.5 validateZipFile - 单文件验证

**目的**：在解压过程中实时验证每个文件，实施资源限制。

---

## 3. 具体技术实现

### 3.1 ZIP 炸弹防护机制

#### 3.1.1 限制常量

```typescript
const LIMITS = {
  MAX_FILE_SIZE: 512 * 1024 * 1024,      // 512MB 单文件
  MAX_TOTAL_SIZE: 1024 * 1024 * 1024,    // 1GB 总解压大小
  MAX_FILE_COUNT: 100000,                // 10万文件
  MAX_COMPRESSION_RATIO: 50,             // 50:1 压缩比上限
  MIN_COMPRESSION_RATIO: 0.5,            // 0.5:1 压缩比下限
}
```

#### 3.1.2 压缩比计算

```typescript
const currentRatio = state.totalUncompressedSize / state.compressedSize
if (currentRatio > LIMITS.MAX_COMPRESSION_RATIO) {
  throw new Error(`Suspicious compression ratio...`)
}
```

**攻击场景**：
- **ZIP 炸弹**：恶意构造的高压缩比文件（如 42.zip 可达到数百万:1）
- **预压缩内容**：攻击者将已压缩内容（如 JPEG）放入 ZIP，低压缩比可能隐藏其他恶意行为

#### 3.1.3 fflate Filter 机制

```typescript
const result = unzipSync(new Uint8Array(zipData), {
  filter: file => {
    const validationResult = validateZipFile(file, state)
    if (!validationResult.isValid) {
      throw new Error(validationResult.error!)
    }
    return true
  },
})
```

- `filter` 回调在每个文件解压前调用
- 可访问文件的元数据（名称、原始大小）
- 抛出错误可中止整个解压过程

### 3.2 ZIP 中央目录解析

#### 3.2.1 ZIP 文件结构

```
[Local File Header 1][File Data 1]
[Local File Header 2][File Data 2]
...
[Central Directory Header 1]
[Central Directory Header 2]
...
[End of Central Directory Record]
```

#### 3.2.2 中央目录条目结构（简化）

```
Offset  Size  Description
  0      4    Signature (0x02014b50)
  4      2    Version made by
  ...
  28     2    File name length
  30     2    Extra field length
  32     2    Comment length
  38     4    External file attributes
  42     4    Relative offset of local header
  46     N    File name
```

#### 3.2.3 权限提取算法

```typescript
const versionMadeBy = buf.readUInt16LE(off + 4)     // 创建者版本
const externalAttr = buf.readUInt32LE(off + 38)     // 外部属性

// versionMadeBy 高字节 = 主机 OS
// 3 = Unix, 0 = DOS, 6 = OS/2, etc.
if (versionMadeBy >> 8 === 3) {
  // Unix: 高 16 位是 st_mode
  const mode = (externalAttr >>> 16) & 0xffff
  if (mode) modes[name] = mode
}
```

**权限位布局**：
```
External Attributes (32-bit)
[st_mode (16-bit)][st_uid/st_gid (16-bit)]

st_mode 布局（标准 Unix）：
[file type (4-bit)][special (3-bit)][user (3-bit)][group (3-bit)][other (3-bit)]
```

### 3.3 路径安全检测

#### 3.3.1 路径遍历检测

```typescript
export function containsPathTraversal(path: string): boolean {
  return /(?:^|[\\/])\.\.(?:[\\/]|$)/.test(path)
}
```

**匹配模式**：
- `../file` - 开头遍历
- `dir/../file` - 中间遍历
- `file/..` - 结尾遍历
- Windows 风格 `..\file`

#### 3.3.2 绝对路径拒绝

```typescript
if (isAbsolute(normalized)) {
  return false
}
```

**原因**：ZIP 中的绝对路径（如 `/etc/passwd` 或 `C:\Windows\system32`）可能导致：
- 系统文件覆盖
- 敏感信息泄露
- 跨目录写入

### 3.4 延迟加载优化

```typescript
export async function unzipFile(zipData: Buffer): Promise<Record<string, Uint8Array>> {
  const { unzipSync } = await import('fflate')
  // ...
}
```

**优化背景**：
- `fflate` 包含约 196KB 的查找表
- `Int32Array(32769)` 和 `Uint16Array(32768)` 等
- 延迟加载避免启动时内存占用

---

## 4. 关键代码路径与文件引用

### 4.1 调用链

#### 4.1.1 MCPB 文件加载

```
loadMcpbFile (mcpbHandler.ts:698)
├── unzipFile (zip.ts:113)
│   ├── validateZipFile (zip.ts:63)
│   │   ├── isPathSafe (zip.ts:44)
│   │   └── [limits check]
│   └── fflate.unzipSync
└── parseZipModes (zip.ts:160)
    └── [central directory parsing]
```

#### 4.1.2 官方市场 GCS 下载

```
fetchOfficialMarketplaceFromGcs (officialMarketplaceGcs.ts:47)
├── unzipFile (zip.ts:113)
└── parseZipModes (zip.ts:160)
    └── [chmod extracted files]
```

#### 4.1.3 插件 ZIP 缓存

```
extractZipToDirectory (zipCache.ts:331)
├── unzipFile (zip.ts:113)
└── parseZipModes (zip.ts:160)
```

### 4.2 文件引用关系

| 文件 | 引用方式 | 用途 |
|------|----------|------|
| `path` | 静态 import | `isAbsolute`, `normalize` |
| `../debug.js` | `logForDebugging` | 调试日志 |
| `../errors.js` | `isENOENT` | ENOENT 错误检测 |
| `../fsOperations.js` | `getFsImplementation` | 文件系统抽象 |
| `../path.js` | `containsPathTraversal` | 路径遍历检测 |
| `fflate` | 动态 import | ZIP 解压 |

### 4.3 被引用位置

| 文件 | 引用内容 | 用途 |
|------|----------|------|
| `mcpbHandler.ts` | `unzipFile`, `parseZipModes` | MCPB 解压和权限恢复 |
| `zipCache.ts` | `unzipFile`, `parseZipModes` | 插件缓存解压 |
| `officialMarketplaceGcs.ts` | `unzipFile`, `parseZipModes` | GCS 市场下载解压 |

---

## 5. 依赖与外部交互

### 5.1 外部包依赖

| 包名 | 用途 | 加载方式 |
|------|------|----------|
| `fflate` | ZIP 压缩/解压 | 动态导入 |

### 5.2 内部工具依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `path` | `isAbsolute`, `normalize` | 路径处理 |
| `../debug.js` | `logForDebugging` | 调试日志 |
| `../errors.js` | `isENOENT` | 错误分类 |
| `../fsOperations.js` | `getFsImplementation` | 文件系统抽象 |
| `../path.js` | `containsPathTraversal` | 路径安全 |

### 5.3 依赖关系图

```
zip.ts
├── fflate (动态)
├── path (Node.js 内置)
├── ../debug.js
│   └── (多层依赖，见 debug.ts 研究)
├── ../errors.js
│   └── (基础错误工具)
├── ../fsOperations.js
│   ├── fs (Node.js 内置)
│   └── ../slowOperations.js
└── ../path.js
    └── (路径工具链)
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| ZIP64 不支持 | 大于 4GB 或超过 65535 个条目的 ZIP 无法解析权限 | 文档说明，返回空对象（使用默认权限） |
| 压缩比误报 | 某些合法高压缩内容（如日志文件）可能触发 50:1 限制 | 可调阈值或白名单机制 |
| 符号链接攻击 | ZIP 中的符号链接可能指向敏感路径 | 当前未处理，依赖路径遍历检测 |
| 内存耗尽 | `unzipSync` 将所有内容加载到内存 | 大文件需要流式解压 |
| fflate 漏洞 | 依赖第三方库的解压安全性 | 定期更新依赖 |

### 6.2 边界条件

| 场景 | 行为 |
|------|------|
| 空 ZIP 文件 | `unzipSync` 返回空对象，无错误 |
| 损坏的 ZIP | fflate 抛出错误，包装后向上传播 |
| 无中央目录 | `parseZipModes` 返回空对象（无法解析权限） |
| 非 Unix ZIP | `parseZipModes` 返回空对象（无可执行权限信息） |
| 恰好 100,000 文件 | 允许，100,001 拒绝 |
| 压缩比恰好 50:1 | 允许，50.0001:1 拒绝 |

### 6.3 改进建议

#### 6.3.1 短期改进

1. **符号链接处理**：
   ```typescript
   // 在 filter 中检测并拒绝符号链接条目
   if (file.name.endsWith('/') && isSymlink(file)) {
     throw new Error('Symlinks in ZIP are not supported')
   }
   ```

2. **更详细的错误信息**：
   ```typescript
   // 当前只报告第一个错误，可收集所有违规项
   errors.push(`File ${i}: ${error}`)
   ```

3. **压缩比白名单**：
   ```typescript
   // 对于已知高压缩类型（.txt, .csv）放宽限制
   const isTextFile = /\.(txt|csv|log)$/i.test(file.name)
   const maxRatio = isTextFile ? 100 : LIMITS.MAX_COMPRESSION_RATIO
   ```

#### 6.3.2 中期改进

1. **流式解压**：
   ```typescript
   // 对于大文件，使用流式 API 避免内存占用
   export async function* unzipFileStreaming(zipData: Buffer): AsyncGenerator<ZipEntry>
   ```

2. **ZIP64 支持**：
   - 扩展 `parseZipModes` 支持 ZIP64 扩展字段
   - 处理 8 字节的大小和偏移量字段

3. **异步 Worker 解压**：
   - 对于 Bun 环境，修复 worker 终止问题后使用异步 API
   - 提高解压大文件时的响应性

#### 6.3.3 长期改进

1. **内容类型验证**：
   - 基于魔数（magic number）验证文件类型
   - 防止扩展名欺骗攻击

2. **沙箱解压**：
   - 在 chroot 或容器环境中解压
   - 进一步隔离潜在风险

3. **增量更新支持**：
   - 支持 ZIP 差异更新
   - 减少市场插件更新时的下载量

### 6.4 测试建议

当前 `src/utils/dxt/` 目录下无测试文件，建议补充：

1. **单元测试**：
   - `isPathSafe` 的各种路径遍历尝试
   - `validateZipFile` 的边界条件（恰好达到限制）
   - `parseZipModes` 的各种 ZIP 格式（Unix/DOS/Windows）

2. **安全测试**：
   - 路径遍历攻击 ZIP（`../../../etc/passwd`）
   - ZIP 炸弹（高压缩比文件）
   - 符号链接攻击

3. **集成测试**：
   - 与 `mcpbHandler.ts` 的完整加载流程
   - 与 `zipCache.ts` 的缓存解压流程

---

## 附录

### A.1 代码统计

- 文件大小：7,704 bytes
- 代码行数：226 行
- 导出函数：4 个
- 类型定义：3 个
- 依赖模块：6 个（1 个外部动态，5 个内部静态）

### A.2 ZIP 格式参考

- PKZIP APPNOTE.TXT §4.3.12（中央目录）
- PKZIP APPNOTE.TXT §4.3.16（EOCD）
- 外部属性布局：
  - Unix: `(st_mode << 16) | (st_uid << 8) | st_gid`
  - DOS: `(attributes << 16)`

### A.3 相关 Issue/PR

- Bun worker 终止问题：使用 `unzipSync` 替代异步 API
- Windows CI 精度问题：`bigint: true` 用于 inode 检测（见 zipCache.ts）
- 权限恢复需求：GCS 下载的市场插件需要可执行权限
