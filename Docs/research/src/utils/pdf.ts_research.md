# pdf.ts 研究文档

## 场景与职责

本模块提供 PDF 文件的读取、验证和处理功能。核心职责包括：

1. **PDF 读取**：读取 PDF 文件并返回 base64 编码数据
2. **页数统计**：使用 `pdfinfo` 工具获取 PDF 页数
3. **页面提取**：使用 `pdftoppm` 将 PDF 页面提取为 JPEG 图片
4. **错误处理**：提供结构化的 PDF 错误类型（空文件、过大、密码保护、损坏等）

该模块是 FileReadTool 的底层支持，使 Claude 能够读取和处理 PDF 文件，支持直接 base64 编码和页面提取两种模式。

## 功能点目的

### 1. `readPDF()` - PDF 读取
- **目的**：读取 PDF 文件并返回 base64 编码数据
- **验证步骤**：
  1. 检查文件大小（非空）
  2. 检查文件大小不超过 `PDF_TARGET_RAW_SIZE`（20MB）
  3. 验证 PDF 魔数（`%PDF-` 头）
- **返回值**：成功时返回 base64 数据和文件信息，失败时返回结构化错误

### 2. `getPDFPageCount()` - 页数统计
- **目的**：获取 PDF 文件的总页数
- **工具**：使用 `pdfinfo`（poppler-utils 包）
- **容错**：工具不可用或失败时返回 `null`

### 3. `extractPDFPages()` - 页面提取
- **目的**：将 PDF 页面提取为 JPEG 图片
- **工具**：使用 `pdftoppm`（poppler-utils 包）
- **选项**：
  - 支持指定页面范围（firstPage/lastPage）
  - 输出质量：100 DPI JPEG
- **输出**：生成 `page-01.jpg`、`page-02.jpg` 等文件
- **错误识别**：通过 stderr 识别密码保护、损坏等错误

### 4. `isPdftoppmAvailable()` - 工具可用性检查
- **目的**：检查 `pdftoppm` 是否可用
- **缓存**：结果缓存于进程生命周期
- **检测方式**：运行 `pdftoppm -v`，检查退出码和 stderr

## 具体技术实现

### 关键流程

#### PDF 读取流程

```
readPDF(filePath)
    ↓
stat() 获取文件大小
    ↓
大小 = 0? → 返回 empty 错误
    ↓
大小 > PDF_TARGET_RAW_SIZE? → 返回 too_large 错误
    ↓
读取文件内容到 Buffer
    ↓
检查前 5 字节是否为 "%PDF-"
    否 → 返回 corrupted 错误
    ↓
转换为 base64
    ↓
返回成功结果
```

#### 页面提取流程

```
extractPDFPages(filePath, options?)
    ↓
stat() 获取文件大小
    ↓
大小 = 0? → 返回 empty 错误
    ↓
大小 > PDF_MAX_EXTRACT_SIZE? → 返回 too_large 错误
    ↓
isPdftoppmAvailable()?
    否 → 返回 unavailable 错误
    ↓
创建输出目录（tool-results/pdf-${uuid}/）
    ↓
构建 pdftoppm 参数（-jpeg -r 100 [-f first] [-l last]）
    ↓
执行 pdftoppm（超时 120s）
    ↓
检查退出码和 stderr
    密码保护? → 返回 password_protected 错误
    损坏? → 返回 corrupted 错误
    其他错误? → 返回 unknown 错误
    ↓
读取输出目录中的 .jpg 文件
    ↓
返回成功结果（文件路径、大小、输出目录、页数）
```

### 数据结构

```typescript
// PDF 错误类型
export type PDFError = {
  reason: 'empty' | 'too_large' | 'password_protected' | 'corrupted' | 'unknown' | 'unavailable'
  message: string
}

// PDF 结果包装器
export type PDFResult<T> =
  | { success: true; data: T }
  | { success: false; error: PDFError }

// 读取成功数据
{
  type: 'pdf'
  file: {
    filePath: string
    base64: string
    originalSize: number
  }
}

// 提取成功数据
{
  type: 'parts'
  file: {
    filePath: string
    originalSize: number
    outputDir: string
    count: number
  }
}
```

### 大小限制常量

| 常量 | 值 | 用途 |
|------|-----|------|
| `PDF_TARGET_RAW_SIZE` | 20MB | `readPDF` 大小限制（考虑 base64 编码后 ~27MB） |
| `PDF_MAX_EXTRACT_SIZE` | 100MB | `extractPDFPages` 大小限制 |

### 错误识别模式

```typescript
// 密码保护
if (/password/i.test(stderr)) → password_protected

// 文件损坏
if (/damaged|corrupt|invalid/i.test(stderr)) → corrupted
```

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `crypto` | `randomUUID()` 生成输出目录名 |
| `fs/promises` | 文件系统操作 |
| `path` | 路径拼接 |
| `../constants/apiLimits.js` | PDF 大小限制常量 |
| `./errors.js` | 错误消息提取 |
| `./execFileNoThrow.js` | 执行外部命令 |
| `./format.js` | 文件大小格式化 |
| `./fsOperations.js` | 文件系统抽象 |
| `./toolResultStorage.js` | 工具结果目录获取 |

### 外部工具依赖

| 工具 | 包 | 用途 |
|------|-----|------|
| `pdfinfo` | poppler-utils | 获取 PDF 页数 |
| `pdftoppm` | poppler-utils | PDF 转图片 |

### 调用方

| 调用方 | 用途 |
|--------|------|
| `src/tools/FileReadTool/FileReadTool.ts` | 读取 PDF 文件 |

### 输出位置

- **页面提取输出**：`~/.claude/projects/{cwd}/tool-results/pdf-{uuid}/`
- **命名格式**：`page-01.jpg`、`page-02.jpg`、...

## 风险、边界与改进建议

### 已知风险

1. **外部工具依赖**
   - 风险：`pdfinfo` 和 `pdftoppm` 可能未安装
   - 缓解：`isPdftoppmAvailable()` 检查，返回 `unavailable` 错误
   - 建议：提供清晰的安装指引

2. **命令注入**
   - 风险：`filePath` 可能包含恶意字符
   - 现状：直接传递给 `execFileNoThrow`
   - 缓解：`execFileNoThrow` 使用 `execa`，自动转义参数

3. **资源耗尽**
   - 风险：大 PDF 提取可能消耗大量磁盘空间
   - 缓解：`PDF_MAX_EXTRACT_SIZE` 限制（100MB）
   - 潜在问题：提取后的图片可能远超原文件大小

4. **临时文件清理**
   - 风险：提取的图片目录可能残留
   - 现状：依赖外部清理机制
   - 建议：添加自动清理或引用计数

5. **超时处理**
   - 当前：`pdftoppm` 超时 120 秒
   - 风险：超大或复杂 PDF 可能超时
   - 建议：可配置超时或流式处理

### 边界情况

| 场景 | 行为 |
|------|------|
| 文件不存在 | 由 `fs.stat()` 抛出错误，包装为 unknown 错误 |
| 空文件（0 字节） | 返回 `empty` 错误 |
| 非 PDF 文件（假 PDF） | 魔数检查失败，返回 `corrupted` 错误 |
| 密码保护 | 返回 `password_protected` 错误 |
| 损坏的 PDF | 返回 `corrupted` 错误 |
| 单页 PDF | 正常处理，count = 1 |
| 页面范围超出 | pdftoppm 自动处理（提取到最后一页） |
| 工具未安装 | 返回 `unavailable` 错误 |
| 并发提取 | 每个调用独立 UUID 目录，无冲突 |

### 改进建议

1. **进度报告**
   - 当前：无进度反馈
   - 建议：
     - 大文件读取进度
     - 页面提取进度（每页回调）

2. **格式支持扩展**
   - 当前：仅 JPEG 输出
   - 建议：
     - PNG（无损）
     - WebP（更好压缩）
     - 可选质量/DPI 设置

3. **文本提取**
   - 建议：添加 `pdftotext` 支持
   - 用途：文本搜索、索引

4. **元数据提取**
   - 建议：扩展 `pdfinfo` 使用
   - 提取：标题、作者、创建日期等

5. **缩略图生成**
   - 建议：生成低分辨率预览
   - 用途：快速浏览，延迟加载高清

6. **缓存机制**
   - 当前：每次重新提取
   - 建议：
     - 基于文件哈希缓存提取结果
     - 配置缓存大小限制

7. **安全性增强**
   - 建议：
     - PDF 内容扫描（恶意脚本）
     - 沙箱化外部工具执行
     - 资源限制（CPU、内存）

8. **错误恢复**
   - 当前：单一错误返回
   - 建议：
     - 部分页面提取失败时返回成功页面
     - 错误详情和恢复建议

9. **流式处理**
   - 当前：完整文件读取
   - 建议：
     - 流式 base64 编码
     - 分块读取超大文件

10. **遥测集成**
    - 建议：
      - 记录 PDF 处理频率
      - 记录大小分布
      - 记录错误类型分布
