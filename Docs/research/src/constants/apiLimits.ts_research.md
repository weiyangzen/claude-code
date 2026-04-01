# apiLimits.ts 深度研究文档

## 场景与职责

`apiLimits.ts` 是 Claude Code CLI 中定义 Anthropic API 服务端限制的核心常量文件。它集中管理所有与 API 请求相关的硬性限制，包括图片、PDF、媒体文件的大小、尺寸和数量限制。该文件的设计目标是**保持零依赖（dependency-free）**，以防止循环导入问题，确保在模块加载时即可安全使用这些常量。

### 核心使用场景
1. **图片上传预处理**：在用户粘贴或上传图片前，客户端根据这些限制进行预验证和压缩
2. **PDF 文件处理**：决定 PDF 是直接作为 base64 文档块发送，还是需要提取为页面图片
3. **请求构建**：在构造 API 请求时确保不超出媒体数量限制
4. **错误预防**：提前拦截可能触发 API 错误的请求，提供清晰的客户端错误信息

---

## 功能点目的

### 1. 图片限制 (Image Limits)

| 常量 | 值 | 用途 |
|------|-----|------|
| `API_IMAGE_MAX_BASE64_SIZE` | 5 MB | API 硬性限制：base64 编码后的图片大小上限 |
| `IMAGE_TARGET_RAW_SIZE` | 3.75 MB | 客户端目标：原始图片大小上限（考虑 base64 编码膨胀 33%） |
| `IMAGE_MAX_WIDTH` / `IMAGE_MAX_HEIGHT` | 2000px | 客户端尺寸限制，略大于 API 内部限制(1568px)以保留质量 |

**设计原理**：
- Base64 编码会使数据膨胀约 33%（4/3 倍率）
- 公式：`raw_size = base64_size * 3/4`
- API 内部会在 1568px 处进行服务器端调整，但客户端设置 2000px 以在有益时保留更高质量

### 2. PDF 限制 (PDF Limits)

| 常量 | 值 | 用途 |
|------|-----|------|
| `PDF_TARGET_RAW_SIZE` | 20 MB | 原始 PDF 大小上限，编码后约 27MB，为对话上下文留出余量 |
| `API_PDF_MAX_PAGES` | 100 | API 接受的最大页数 |
| `PDF_EXTRACT_SIZE_THRESHOLD` | 3 MB | 超过此大小的 PDF 将提取为页面图片而非直接发送 |
| `PDF_MAX_EXTRACT_SIZE` | 100 MB | 页面提取路径的最大文件大小 |
| `PDF_MAX_PAGES_PER_READ` | 20 | Read 工具单次调用中 pages 参数的最大页数 |
| `PDF_AT_MENTION_INLINE_THRESHOLD` | 10 | @提及时，超过此页数的 PDF 采用引用处理而非内联 |

**处理策略**：
- 小 PDF (<3MB)：直接作为 base64 文档块发送
- 大 PDF (3MB-100MB)：提取为页面图片序列
- 超大 PDF (>100MB)：拒绝处理

### 3. 媒体限制 (Media Limits)

| 常量 | 值 | 用途 |
|------|-----|------|
| `API_MAX_MEDIA_PER_REQUEST` | 100 | 单次 API 请求中图片+PDF 的最大数量 |

**重要性**：API 对超出此限制的请求会返回令人困惑的错误，客户端预验证可提供清晰的错误信息。

---

## 具体技术实现

### 数据结构

所有常量均为简单的 `number` 类型导出：

```typescript
// 图片限制
export const API_IMAGE_MAX_BASE64_SIZE = 5 * 1024 * 1024      // 5 MB
export const IMAGE_TARGET_RAW_SIZE = (API_IMAGE_MAX_BASE64_SIZE * 3) / 4  // 3.75 MB
export const IMAGE_MAX_WIDTH = 2000
export const IMAGE_MAX_HEIGHT = 2000

// PDF 限制
export const PDF_TARGET_RAW_SIZE = 20 * 1024 * 1024           // 20 MB
export const API_PDF_MAX_PAGES = 100
export const PDF_EXTRACT_SIZE_THRESHOLD = 3 * 1024 * 1024     // 3 MB
export const PDF_MAX_EXTRACT_SIZE = 100 * 1024 * 1024         // 100 MB
export const PDF_MAX_PAGES_PER_READ = 20
export const PDF_AT_MENTION_INLINE_THRESHOLD = 10

// 媒体限制
export const API_MAX_MEDIA_PER_REQUEST = 100
```

### 关键代码路径

#### 1. 图片处理路径

```
用户粘贴/上传图片
    ↓
src/utils/imagePaste.ts 或 src/utils/imageResizer.ts
    ↓
检查 API_IMAGE_MAX_BASE64_SIZE / IMAGE_MAX_WIDTH / IMAGE_MAX_HEIGHT
    ↓
必要时进行压缩/调整尺寸
    ↓
构造 API 请求
```

**关键文件引用**：
- `src/utils/imagePaste.ts`: 处理剪贴板图片粘贴，使用 `API_IMAGE_MAX_BASE64_SIZE`
- `src/utils/imageResizer.ts`: 图片尺寸调整，使用 `IMAGE_MAX_WIDTH` / `IMAGE_MAX_HEIGHT`
- `src/utils/imageValidation.ts`: 图片验证逻辑
- `src/utils/attachments.ts`: 附件处理，使用 `API_MAX_MEDIA_PER_REQUEST`

#### 2. PDF 处理路径

```
用户上传 PDF
    ↓
src/utils/pdf.ts
    ↓
检查文件大小与 PDF_EXTRACT_SIZE_THRESHOLD 对比
    ↓
[<3MB] 直接作为文档块 ──→ 构造 base64 请求
[>3MB] 提取为页面图片 ──→ pdf2pic 转换 → 多图片请求
    ↓
src/tools/FileReadTool/FileReadTool.ts 处理分页读取
    ↓
应用 PDF_MAX_PAGES_PER_READ 限制
```

**关键文件引用**：
- `src/utils/pdf.ts`: PDF 核心处理逻辑，使用 `PDF_EXTRACT_SIZE_THRESHOLD`、`PDF_MAX_EXTRACT_SIZE`
- `src/tools/FileReadTool/FileReadTool.ts`: 文件读取工具，使用 `PDF_MAX_PAGES_PER_READ`
- `src/services/api/claude.ts`: API 请求构造，使用 `API_PDF_MAX_PAGES`

#### 3. 错误处理路径

```
构造请求时
    ↓
src/services/api/errors.ts
    ↓
验证媒体数量是否超过 API_MAX_MEDIA_PER_REQUEST
    ↓
超出限制时抛出清晰的客户端错误（而非等待 API 返回模糊错误）
```

---

## 依赖与外部交互

### 内部依赖

**零依赖设计**：该文件明确不导入任何其他模块，注释中强调：
> "Keep this file dependency-free to prevent circular imports."

### 被依赖方

| 文件 | 使用的常量 | 用途 |
|------|-----------|------|
| `src/utils/imagePaste.ts` | `API_IMAGE_MAX_BASE64_SIZE` | 图片粘贴验证 |
| `src/utils/imageResizer.ts` | `IMAGE_MAX_WIDTH`, `IMAGE_MAX_HEIGHT` | 图片尺寸调整 |
| `src/utils/imageValidation.ts` | `API_IMAGE_MAX_BASE64_SIZE` | 图片验证 |
| `src/utils/pdf.ts` | `PDF_EXTRACT_SIZE_THRESHOLD`, `PDF_MAX_EXTRACT_SIZE` | PDF 处理策略 |
| `src/utils/attachments.ts` | `API_MAX_MEDIA_PER_REQUEST` | 附件数量验证 |
| `src/tools/FileReadTool/FileReadTool.ts` | `PDF_MAX_PAGES_PER_READ`, `PDF_AT_MENTION_INLINE_THRESHOLD` | PDF 分页读取 |
| `src/services/api/claude.ts` | `API_PDF_MAX_PAGES`, `API_IMAGE_MAX_BASE64_SIZE` | API 请求构造 |
| `src/services/api/errors.ts` | `API_MAX_MEDIA_PER_REQUEST` | 错误信息生成 |

### 外部 API 依赖

这些常量与 Anthropic API 的服务端限制保持同步：

- **来源**：`api/api/schemas/messages/blocks/` 和 `api/api/config.py`
- **最后验证日期**：2025-12-22
- **未来计划**：Issue #13240 提议从服务器动态获取限制值

---

## 风险、边界与改进建议

### 当前风险

1. **硬编码限制过时风险**
   - API 服务端限制可能变更，但客户端常量需要手动更新
   - 注释中提到未来会通过 Issue #13240 实现动态获取

2. **base64 计算精度**
   - `IMAGE_TARGET_RAW_SIZE` 使用理论计算 `(API_IMAGE_MAX_BASE64_SIZE * 3) / 4`
   - 实际 base64 编码可能因填充略有差异

3. **PDF 提取阈值策略**
   - `PDF_EXTRACT_SIZE_THRESHOLD` (3MB) 是一个经验值
   - 不同 PDF 复杂度可能导致提取后总大小差异很大

### 边界情况

| 场景 | 处理方式 |
|------|---------|
| 图片恰好 5MB base64 | 边界通过，但建议留余量 |
| PDF 恰好 100 页 | 允许，但可能触发其他限制 |
| 100+ 媒体项 | 客户端拒绝，提示分批处理 |
| 多工作目录 | 每个目录独立计算限制 |

### 改进建议

1. **动态限制获取**（已计划）
   ```typescript
   // 未来实现方向
   export async function getDynamicLimits(): Promise<ApiLimits> {
     // 从服务器获取最新限制
   }
   ```

2. **更精确的大小预估**
   - 对 PDF 提取路径，可在提取前预估最终图片总大小
   - 避免提取后发现超出限制

3. **分层限制策略**
   - 当前是硬限制，可考虑添加软限制（警告）和硬限制（拒绝）
   - 给用户更灵活的控制

4. **监控与告警**
   - 记录限制触发的频率和类型
   - 帮助识别是否需要调整阈值

### 相关测试建议

应确保以下场景有测试覆盖：
- 各种边界值（恰好等于限制值）
- 略超限制值的优雅降级
- 多文件组合触发 `API_MAX_MEDIA_PER_REQUEST`
- 不同 PDF 大小走不同处理路径
