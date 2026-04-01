# src/utils/imageResizer.ts 研究文档

## 场景与职责

`imageResizer.ts` 是 Claude Code CLI 的图像处理核心模块，负责在将图像发送到 Anthropic API 之前进行客户端预处理。由于 API 对图像有严格的尺寸限制（base64 编码后最大 5MB，建议原始大小不超过 3.75MB），该模块承担了以下职责：

1. **图像尺寸调整与降采样**：将超大图像缩放到 `IMAGE_MAX_WIDTH × IMAGE_MAX_HEIGHT`（2000×2000）以内。
2. **图像压缩**：通过格式转换（PNG → JPEG）和质量降级，确保图像原始大小不超过 `IMAGE_TARGET_RAW_SIZE`（3.75MB）。
3. **格式检测**：通过 magic bytes 检测图像真实格式，避免被文件扩展名欺骗。
4. **元数据提取与坐标映射**：记录原始尺寸和显示尺寸，为后续 UI 坐标反映射提供依据。
5. **错误分类与遥测**：对图像处理失败进行细粒度错误分类，并上报 analytics 事件。

调用方主要包括：
- `src/utils/attachments.ts`：处理附件图像
- `src/utils/imagePaste.ts`：处理粘贴图像
- `src/hooks/usePasteHandler.ts` / `src/hooks/useTextInput.ts`：输入框图像粘贴
- `src/bridge/inboundMessages.ts`：桥接消息中的图像
- `src/tools/FileReadTool/imageProcessor.ts`：底层图像处理器封装

## 功能点目的

### 1. `maybeResizeAndDownsampleImageBuffer`
这是从 `FileReadTool` 的 `readImage` 中提取出来的核心函数，目的是在保持可接受视觉质量的前提下，将任意图像 buffer 压缩到 API 限制范围内。其策略是：
- 先检查原始图像是否已经满足尺寸和大小限制，若是则直接返回。
- 若仅大小超限但尺寸合规，优先尝试 PNG 压缩（保留透明度），再尝试 JPEG 质量阶梯式降级（80 → 60 → 40 → 20）。
- 若尺寸也超限，先按比例缩放至 2000px 以内，再执行上述压缩。
- 若仍超限，则进行极限压缩（缩放到 1000px 宽 + JPEG quality 20）。

### 2. `maybeResizeAndDownsampleImageBlock`
面向 `ImageBlockParam`（Anthropic SDK 类型）的包装函数，仅处理 `source.type === 'base64'` 的图像块，解码 base64 → buffer → 处理 → 重新编码为 base64。

### 3. `compressImageBuffer` / `compressImageBufferWithTokenLimit` / `compressImageBlock`
提供另一套更激进的压缩管线（来自 `FileReadTool`），用于将图像压缩到指定的字节上限或 token 上限。策略包括：
- 渐进式缩放（1.0 → 0.75 → 0.5 → 0.25）
- PNG 调色板优化（palette: true, colors: 64）
- 中等 JPEG 转换（quality 50, 600×600）
- 极限 JPEG 转换（quality 20, 400×400）

### 4. `detectImageFormatFromBuffer` / `detectImageFormatFromBase64`
通过文件头 magic bytes 识别 PNG、JPEG、GIF、WebP，防止扩展名与真实格式不符导致处理失败。

### 5. `createImageMetadataText`
生成图像元数据文本（如 `[Image: source: path/to/file, original 3000x2000, displayed at 2000x1333. Multiply coordinates by 1.50 to map to original image.]`），供模型理解坐标映射关系。

### 6. `classifyImageError`
将图像处理错误分类为 8 种类型（模块加载、处理错误、像素限制、内存、超时、Vips、权限、未知），用于遥测上报时避免发送敏感字符串。

## 具体技术实现

### 图像处理引擎选择
模块通过 `getImageProcessor()` 动态获取底层图像处理库：
- **Bundle 模式**：优先尝试加载 `image-processor-napi`（原生高性能模块），失败则回退到 `sharp`。
- **非 Bundle 模式**：直接使用 `sharp`。

> 重要实现细节：代码中反复强调**每次操作都要创建新的 `sharp(imageBuffer)` 实例**。因为 `image-processor-napi` 在调用 `toBuffer()` 后复用实例会导致格式转换不生效，这是一个已修复的 bug（注释中明确说明）。

### 压缩策略流程图（`maybeResizeAndDownsampleImageBuffer`）
```
空 buffer? → 抛 ImageResizeError
获取 metadata → 无法获取尺寸?
  ├─ 原始大小 > 3.75MB → JPEG quality 80
  └─ 否则 → 原样返回
尺寸和大小都合规? → 原样返回
仅大小超限?
  ├─ PNG → png({compressionLevel:9, palette:true})
  └─ JPEG 质量阶梯 80/60/40/20
尺寸超限 → 按比例缩放至 2000px 内
缩放后仍超限?
  ├─ PNG → 缩放 + PNG 压缩
  ├─ JPEG 质量阶梯 80/60/40/20
  └─ 极限方案 → 1000px + JPEG 20
```

### 错误处理与降级
在 `catch` 块中：
1. 记录错误并上报 `tengu_image_resize_failed`。
2. 通过 magic bytes 检测真实格式。
3. 计算 base64 大小（`Math.ceil(originalSize * 4 / 3)`）。
4. 对于 PNG，直接读取 IHDR chunk 的宽高，检查是否超限。
5. 若 base64 大小 ≤ 5MB 且尺寸不超限，则允许原图通过（fallback）。
6. 否则抛出用户友好的 `ImageResizeError`，提示手动压缩。

### 数据结构
```typescript
interface ResizeResult {
  buffer: Buffer
  mediaType: string
  dimensions?: ImageDimensions  // { originalWidth?, originalHeight?, displayWidth?, displayHeight? }
}

interface ImageCompressionContext {
  imageBuffer: Buffer
  metadata: { width?: number; height?: number; format?: string }
  format: string
  maxBytes: number
  originalSize: number
}
```

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/imageResizer.ts:169-433` | `maybeResizeAndDownsampleImageBuffer` 主函数 |
| `src/utils/imageResizer.ts:445-481` | `maybeResizeAndDownsampleImageBlock` ImageBlockParam 包装 |
| `src/utils/imageResizer.ts:498-577` | `compressImageBuffer` 激进压缩管线 |
| `src/utils/imageResizer.ts:769-812` | `detectImageFormatFromBuffer` magic bytes 检测 |
| `src/utils/imageResizer.ts:50-124` | `classifyImageError` 错误分类器 |
| `src/constants/apiLimits.ts` | API 限制常量（5MB、3.75MB、2000px） |
| `src/tools/FileReadTool/imageProcessor.ts` | 底层图像处理器获取（sharp / image-processor-napi） |
| `src/utils/format.ts` | `formatFileSize` 用于错误消息 |
| `src/utils/debug.ts` | `logForDebugging` 调试日志 |
| `src/services/analytics/index.ts` | `logEvent` 遥测上报 |

## 依赖与外部交互

### 内部依赖
- `../constants/apiLimits.js`：`API_IMAGE_MAX_BASE64_SIZE`, `IMAGE_TARGET_RAW_SIZE`, `IMAGE_MAX_WIDTH`, `IMAGE_MAX_HEIGHT`
- `../tools/FileReadTool/imageProcessor.js`：`getImageProcessor`, `SharpFunction`, `SharpInstance`
- `../services/analytics/index.js`：`logEvent`
- `./debug.js`：`logForDebugging`
- `./errors.js`：`errorMessage`
- `./format.js`：`formatFileSize`
- `./log.js`：`logError`

### 外部依赖
- `@anthropic-ai/sdk/resources/messages.mjs`：`Base64ImageSource`, `ImageBlockParam`
- `sharp` / `image-processor-napi`：底层图像处理（动态导入）

### 调用方
- `src/utils/attachments.ts`
- `src/utils/imagePaste.ts`
- `src/hooks/usePasteHandler.ts`
- `src/hooks/useTextInput.ts`
- `src/bridge/inboundMessages.ts`
- `src/utils/mcpValidation.ts`
- `src/utils/config.ts`（类型引用 `ImageDimensions`）

## 风险、边界与改进建议

### 风险与边界
1. **原生模块加载失败**：`image-processor-napi` 在某些平台（如 ARM Linux、Alpine）可能因缺少编译产物而加载失败，虽然会回退到 `sharp`，但首次加载失败会触发 `console.warn` 并增加启动延迟。
2. **极限压缩导致图像不可读**：当图像被压缩到 1000px + JPEG quality 20 时，文字类截图可能模糊到模型无法 OCR，但用户不会收到质量警告，只会收到大小提示。
3. **PNG 尺寸绕过检测的局限性**：`overDim` 检测仅针对 PNG 格式，若超大 BMP/TIFF 被错误标记为其他格式，可能绕过检测。
4. **base64 大小计算的精度**：`Math.ceil(originalSize * 4 / 3)` 是近似值，未考虑 base64 padding，极端情况下可能有几字节的低估。
5. **内存占用**：大图像（如 50MB 原始文件）会被完整读入 Buffer，然后创建多个 `sharp` 实例，可能瞬间占用数百 MB 内存。

### 改进建议
1. **流式处理大图像**：对于明显超过某个阈值（如 20MB）的图像，可以先拒绝或采用流式压缩，避免全量加载到内存。
2. **质量降级可视化提示**：在将图像压缩到 quality 20 时，向用户显示一个警告（如 "Image quality significantly reduced"）。
3. **扩展格式支持**：考虑增加对 AVIF、HEIC 等现代格式的检测和转换支持。
4. **错误分类器国际化**：当前 `classifyImageError` 依赖英文错误消息匹配，若 sharp 未来更改错误文本，分类可能失效。建议与 sharp 版本号绑定测试。
5. **缓存压缩结果**：对于同一会话中重复使用的图像，可以缓存压缩后的 buffer，避免重复计算。
