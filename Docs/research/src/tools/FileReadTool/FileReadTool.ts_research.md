# FileReadTool.ts 研究文档

## 场景与职责

FileReadTool 是 Claude Code 的核心文件读取工具，负责从本地文件系统读取各种类型的文件内容并返回给模型。它是整个系统中最基础、使用频率最高的工具之一，支持以下文件类型：

1. **文本文件** - 代码、配置文件、文档等
2. **图片文件** - PNG、JPEG、GIF、WebP，支持自动压缩和尺寸调整
3. **PDF 文件** - 支持整本读取或按页码范围提取
4. **Jupyter Notebook** - 读取 `.ipynb` 文件并解析单元格内容

该工具在以下场景发挥关键作用：
- 用户请求查看代码文件内容
- 分析截图或图片
- 阅读 PDF 文档
- 查看 Notebook 中的代码和输出
- 自动记忆文件（如 CLAUDE.md）的加载

## 功能点目的

### 1. 多类型文件支持
- **文本读取**：支持大文件分片读取（offset/limit 参数）
- **图片处理**：自动压缩、调整尺寸以适应 API 限制
- **PDF 处理**：支持整本读取或页码范围提取，大 PDF 自动转为图片
- **Notebook 解析**：提取代码单元格、输出和元数据

### 2. 安全与权限控制
- 检查文件系统权限规则（allow/deny/ask）
- 阻止读取危险设备文件（/dev/zero, /dev/random 等）
- 验证 PDF 文件头防止伪造
- 网络安全风险提醒（CYBER_RISK_MITIGATION_REMINDER）

### 3. 性能优化
- **重复读取去重**：通过 `readFileState` 缓存避免重复读取未变更文件
- **大文件流式读取**：使用 `readFileInRange` 的 streaming 路径处理大文件
- **图片压缩**：多策略压缩（PNG palette、JPEG quality 降级）
- **Token 限制**：预估并验证内容 token 数，防止超出模型上下文

### 4. 智能路径处理
- 支持 macOS 截图文件名空格变体（普通空格 vs 窄空格 U+202F）
- 文件不存在时提供相似文件建议
- 路径展开（~ 展开为 home 目录）

### 5. 技能自动发现
- 读取文件时自动发现并加载相关 skill 目录
- 激活条件匹配的技能

## 具体技术实现

### 核心数据结构

```typescript
// 输入参数 Schema
interface Input {
  file_path: string    // 绝对路径
  offset?: number      // 起始行号（1-indexed）
  limit?: number       // 读取行数
  pages?: string       // PDF 页码范围（如 "1-5"）
}

// 输出类型（Discriminated Union）
type Output = 
  | { type: 'text'; file: { filePath, content, numLines, startLine, totalLines } }
  | { type: 'image'; file: { base64, type, originalSize, dimensions? } }
  | { type: 'notebook'; file: { filePath, cells } }
  | { type: 'pdf'; file: { filePath, base64, originalSize } }
  | { type: 'parts'; file: { filePath, originalSize, count, outputDir } }
  | { type: 'file_unchanged'; file: { filePath } }
```

### 关键流程

#### 1. 文件读取主流程 (`call` 方法)

```
1. 获取文件读取限制（maxSizeBytes, maxTokens）
2. 检查是否已缓存且未变更（readFileState 去重）
3. 发现相关 skills 并异步加载
4. 根据文件扩展名分发到不同处理器：
   - .ipynb → readNotebook → notebook 类型
   - 图片扩展名 → readImageWithTokenBudget → image 类型
   - .pdf → PDF 处理流程
   - 其他 → readFileInRange → text 类型
5. 记录文件操作日志和遥测数据
6. 更新 readFileState 缓存
```

#### 2. PDF 处理流程

```
if (pages 参数提供):
    1. 解析页码范围
    2. 使用 pdftoppm 提取指定页面为 JPEG
    3. 压缩图片并返回 image blocks
else:
    1. 获取 PDF 页数
    2. 如果页数 > 10: 报错要求提供 pages 参数
    3. 如果 PDF 不支持或大小 > 3MB: 提取为图片
    4. 否则: 读取为 base64 document block
```

#### 3. 图片处理流程 (`readImageWithTokenBudget`)

```
1. 读取图片文件到 buffer（单次读取）
2. 检测图片格式（magic bytes）
3. 尝试标准压缩（maybeResizeAndDownsampleImageBuffer）
   - 检查尺寸和大小限制
   - PNG: 尝试 palette 压缩
   - JPEG: 尝试 quality 80/60/40/20 降级
   - 必要时调整尺寸到 2000x2000
4. 如果超出 token 预算:
   - 使用 compressImageBufferWithTokenLimit 进一步压缩
   - 极端情况: 400x400 JPEG quality 20
5. 返回 base64 编码图片
```

### 关键代码路径

| 功能 | 代码路径 | 行号 |
|------|----------|------|
| 工具定义 | `buildTool({ name: FILE_READ_TOOL_NAME, ... })` | 337-718 |
| 主调用入口 | `call(file_path, offset, limit, pages)` | 496-651 |
| 内部实现 | `callInner(...)` | 804-1086 |
| 图片读取 | `readImageWithTokenBudget(filePath, maxTokens)` | 1097-1183 |
| Token 验证 | `validateContentTokens(content, ext, maxTokens)` | 755-772 |
| 重复读取检测 | `readFileState.get(fullFilePath)` | 540-573 |
| 网络安全提醒 | `CYBER_RISK_MITIGATION_REMINDER` | 729-730 |
| 文件读取监听 | `registerFileReadListener(listener)` | 165-173 |

### 依赖与外部交互

#### 直接依赖模块

| 模块 | 用途 |
|------|------|
| `../../Tool.js` | `buildTool`, `ToolDef` 基础工具框架 |
| `./prompt.js` | 工具描述、提示模板常量 |
| `./UI.tsx` | 渲染工具使用和结果消息 |
| `./limits.js` | 文件读取限制配置 |
| `./imageProcessor.js` | 图片处理（sharp/native）|
| `../../utils/readFileInRange.js` | 文本文件范围读取 |
| `../../utils/imageResizer.js` | 图片压缩、调整尺寸 |
| `../../utils/pdf.js` | PDF 读取和页码提取 |
| `../../utils/pdfUtils.js` | PDF 工具函数（parsePDFPageRange, isPDFSupported）|
| `../../utils/notebook.js` | Notebook 解析 |
| `../../utils/file.js` | 文件工具函数（addLineNumbers, findSimilarFile）|
| `../../utils/fileStateCache.js` | 文件状态缓存 |
| `../../skills/loadSkillsDir.js` | Skill 发现和加载 |
| `../../services/analytics/index.js` | 遥测事件上报 |

#### 外部系统交互

1. **文件系统**: 通过 `getFsImplementation()` 获取 fs 实现（支持真实/sandbox/测试环境）
2. **PDF 工具**: 调用 `pdftoppm`（poppler-utils）提取 PDF 页面
3. **图片处理**: 使用 sharp 库或原生 image-processor-napi 模块
4. **API 限制**: 遵循 Anthropic API 的图片/PDF 大小限制

### 配置与常量

```typescript
// 被阻止的设备文件路径（无限输出或阻塞输入）
const BLOCKED_DEVICE_PATHS = new Set([
  '/dev/zero', '/dev/random', '/dev/urandom', '/dev/full',
  '/dev/stdin', '/dev/tty', '/dev/console',
  '/dev/stdout', '/dev/stderr',
  '/dev/fd/0', '/dev/fd/1', '/dev/fd/2'
])

// 支持的图片扩展名
const IMAGE_EXTENSIONS = new Set(['png', 'jpg', 'jpeg', 'gif', 'webp'])

// 网络安全提醒（附加到文本文件内容）
export const CYBER_RISK_MITIGATION_REMINDER = 
  "Whenever you read a file, you should consider whether it would be considered malware..."
```

## 风险、边界与改进建议

### 已知风险

1. **PDF 伪造风险**
   - 已缓解：验证 PDF 文件头 `%PDF-`，防止 HTML 重命名为 PDF 导致 API 错误

2. **设备文件阻塞**
   - 已缓解：路径级检查阻止读取 /dev/zero 等设备文件

3. **Token 溢出**
   - 已缓解：预估 + API 精确计算 token 数，超出时抛出 MaxFileReadTokenExceededError

4. **图片处理失败**
   - 已缓解：多层级 fallback（标准压缩 → 激进压缩 → 极端压缩 → 原始数据）

5. **重复读取浪费 Token**
   - 已缓解：通过 readFileState 缓存检测文件未变更，返回 file_unchanged 类型

### 边界情况

| 场景 | 处理方式 |
|------|----------|
| 文件不存在 | 尝试 macOS 截图路径变体 → 查找相似文件 → 返回友好错误 |
| 空文件 | 返回警告："the file exists but the contents are empty" |
| offset 超出文件长度 | 返回警告："the file exists but is shorter than the provided offset" |
| 大文件 (>10MB) | 使用 streaming 路径，避免内存溢出 |
| 超大单行文件 | streaming 路径下丢弃范围外内容，防止内存无限增长 |
| PDF 密码保护 | 检测 stderr 中的 "password" 关键字，返回友好错误 |
| PDF 损坏 | 检测 stderr 中的 "damaged/corrupt/invalid" 关键字 |

### 改进建议

1. **PDF 处理优化**
   - 当前：依赖外部 pdftoppm 工具，需要用户手动安装 poppler-utils
   - 建议：考虑集成纯 JavaScript PDF 解析库（如 pdfjs）减少外部依赖

2. **图片压缩策略**
   - 当前：固定 quality 值序列（80/60/40/20）
   - 建议：实现二分查找最优 quality，减少压缩尝试次数

3. **缓存策略**
   - 当前：仅基于 mtime 判断文件是否变更
   - 建议：增加文件内容 hash 校验，处理 mtime 精度问题

4. **大文件处理**
   - 当前：maxSizeBytes 检查整个文件而非读取范围
   - 建议：对于显式 offset/limit 请求，允许读取大文件的指定部分

5. **错误处理细化**
   - 当前：图片处理错误分类依赖字符串匹配
   - 建议：与 sharp 库维护者协商暴露结构化错误代码

6. **性能监控**
   - 当前：有基础遥测（tengu_file_read_dedup, tengu_image_resize_failed）
   - 建议：增加读取耗时、压缩耗时、缓存命中率等指标

### 测试注意事项

- 图片处理依赖 sharp 或原生模块，测试环境需要相应配置
- PDF 测试需要 poppler-utils 安装
- 文件系统权限测试需要模拟不同的 ToolPermissionContext
- 大文件测试需要准备特定大小的测试文件
