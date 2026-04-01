# imagePaste.ts 研究文档

## 场景与职责

`imagePaste.ts` 是 Claude Code CLI 的 **图像剪贴板处理模块**，负责从系统剪贴板读取图像、处理图像文件路径粘贴，以及将图像转换为适合 Anthropic API 传输的格式。该模块是图像交互功能的核心，支持用户通过复制粘贴或拖拽方式将图像输入到 Claude Code。

### 核心使用场景

1. **剪贴板图像读取**：
   - 用户复制图像后，在 Claude Code 中粘贴（Cmd+V/Ctrl+V）
   - 检测剪贴板内容并提取图像数据
   - 转换为 base64 格式供 API 使用

2. **图像文件拖拽**：
   - 用户从文件管理器拖拽图像文件到终端
   - 解析文件路径并读取图像内容
   - 支持绝对路径和相对路径

3. **截图直接粘贴**：
   - macOS 截图后直接进入剪贴板
   - 检测临时截图文件并处理

4. **跨平台支持**：
   - macOS：使用 `osascript` 或原生 NSPasteboard 模块
   - Linux：使用 `xclip` 或 `wl-paste`
   - Windows：使用 PowerShell

---

## 功能点目的

### 1. 剪贴板图像检测 (`hasImageInClipboard`)

**设计目标**：快速检测剪贴板中是否包含图像，用于 UI 提示（如显示"剪贴板中有图像"提示）。

**实现策略**：
- **macOS 优先使用原生模块**：通过 `image-processor-napi` 的 `hasClipboardImage()` 方法（约 0.03ms）
- **回退到 osascript**：如果原生模块不可用，使用 AppleScript 检测
- **其他平台**：暂不支持（返回 `false`）

### 2. 剪贴板图像读取 (`getImageFromClipboard`)

**设计目标**：从剪贴板提取图像并转换为 API 可用格式。

**处理流程**：
1. **原生模块快速路径**（macOS）：
   - 使用 `image-processor-napi` 直接读取 PNG 字节
   - 通过 CoreGraphics 自动降采样（如果超过尺寸限制）
   - 约 5ms 冷启动，亚毫秒级热启动

2. **命令行回退路径**：
   - 平台特定命令提取图像到临时文件
   - 读取文件并转换为 base64
   - BMP 格式自动转换为 PNG（WSL2 默认使用 BMP）

3. **尺寸和大小优化**：
   - 最大尺寸：1568x1568（`IMAGE_MAX_WIDTH/HEIGHT`）
   - 目标大小：3.75MB 原始数据（`IMAGE_TARGET_RAW_SIZE`）
   - 超过限制时自动压缩或降采样

### 3. 图像文件路径处理 (`isImageFilePath`, `asImageFilePath`)

**支持的图像格式**：
- PNG (`.png`)
- JPEG (`.jpg`, `.jpeg`)
- GIF (`.gif`)
- WebP (`.webp`)

**路径清理逻辑**：
- 去除外层引号（单引号或双引号）
- 去除 shell 转义反斜杠（macOS/Linux）
- 保留 Windows 路径中的反斜杠

### 4. 图像文件读取 (`tryReadImageFromPath`)

**处理流程**：
1. 清理和验证路径
2. 尝试读取绝对路径
3. 如果是文件名，尝试从剪贴板获取完整路径匹配
4. BMP 自动转 PNG
5. 尺寸优化
6. 返回 base64 数据和元数据

---

## 具体技术实现

### 平台特定命令配置

```typescript
function getClipboardCommands() {
  const platform = process.platform as SupportedPlatform
  const screenshotPath = join(baseTmpDir, 'claude_cli_latest_screenshot.png')

  const commands: Record<SupportedPlatform, ClipboardCommands> = {
    darwin: {
      checkImage: `osascript -e 'the clipboard as «class PNGf»'`,
      saveImage: `osascript -e 'set png_data to (the clipboard as «class PNGf»)' -e '...'`,
      getPath: `osascript -e 'get POSIX path of (the clipboard as «class furl»)'`,
      deleteFile: `rm -f "${screenshotPath}"`,
    },
    linux: {
      checkImage: 'xclip -selection clipboard -t TARGETS -o | grep image/ || wl-paste -l | grep image/',
      saveImage: `xclip -selection clipboard -t image/png -o > "${screenshotPath}" || wl-paste --type image/png > "${screenshotPath}"`,
      getPath: 'xclip -selection clipboard -t text/plain -o || wl-paste',
      deleteFile: `rm -f "${screenshotPath}"`,
    },
    win32: {
      checkImage: 'powershell -Command "(Get-Clipboard -Format Image) -ne $null"',
      saveImage: `powershell -Command "$img = Get-Clipboard -Format Image; if ($img) { $img.Save('${screenshotPath}') }"`,
      getPath: 'powershell -Command "Get-Clipboard"',
      deleteFile: `del /f "${screenshotPath}"`,
    },
  }
}
```

### 原生模块快速路径

```typescript
export async function getImageFromClipboard(): Promise<ImageWithDimensions | null> {
  // 检查功能开关
  if (feature('NATIVE_CLIPBOARD_IMAGE') && process.platform === 'darwin') {
    try {
      const { getNativeModule } = await import('image-processor-napi')
      const readClipboard = getNativeModule()?.readClipboardImage
      if (!readClipboard) throw new Error('native clipboard reader unavailable')
      
      const native = readClipboard(IMAGE_MAX_WIDTH, IMAGE_MAX_HEIGHT)
      if (!native) return null  // 剪贴板无图像
      
      // 检查文件大小，必要时进一步压缩
      if (native.png.length > IMAGE_TARGET_RAW_SIZE) {
        const resized = await maybeResizeAndDownsampleImageBuffer(
          native.png,
          native.png.length,
          'png'
        )
        return {
          base64: resized.buffer.toString('base64'),
          mediaType: `image/${resized.mediaType}`,
          dimensions: {
            originalWidth: native.originalWidth,
            originalHeight: native.originalHeight,
            displayWidth: resized.dimensions?.displayWidth ?? native.width,
            displayHeight: resized.dimensions?.displayHeight ?? native.height,
          },
        }
      }
      
      return {
        base64: native.png.toString('base64'),
        mediaType: 'image/png',
        dimensions: { /* ... */ },
      }
    } catch (e) {
      logError(e as Error)
      // 回退到 osascript
    }
  }
  
  // 命令行回退路径...
}
```

### 路径清理实现

```typescript
function stripBackslashEscapes(path: string): string {
  const platform = process.platform as SupportedPlatform
  
  // Windows 保留反斜杠
  if (platform === 'win32') return path
  
  // 使用随机盐防止占位符注入
  const salt = randomBytes(8).toString('hex')
  const placeholder = `__DOUBLE_BACKSLASH_${salt}__`
  
  // 1. 将实际的双反斜杠替换为占位符
  const withPlaceholder = path.replace(/\\\\/g, placeholder)
  
  // 2. 去除 shell 转义反斜杠（如 `\ ` → ` `）
  const withoutEscapes = withPlaceholder.replace(/\\(.)/g, '$1')
  
  // 3. 恢复实际的双反斜杠为单反斜杠
  return withoutEscapes.replace(new RegExp(placeholder, 'g'), '\\')
}
```

### 图像文件读取

```typescript
export async function tryReadImageFromPath(
  text: string
): Promise<(ImageWithDimensions & { path: string }) | null> {
  const cleanedPath = asImageFilePath(text)
  if (!cleanedPath) return null

  let imageBuffer: Buffer | undefined

  try {
    if (isAbsolute(imagePath)) {
      imageBuffer = getFsImplementation().readFileBytesSync(imagePath)
    } else {
      // VSCode 终端只获取文件名，尝试从剪贴板匹配完整路径
      const clipboardPath = await getImagePathFromClipboard()
      if (clipboardPath && imagePath === basename(clipboardPath)) {
        imageBuffer = getFsImplementation().readFileBytesSync(clipboardPath)
      }
    }
  } catch (e) {
    logError(e as Error)
    return null
  }

  if (!imageBuffer || imageBuffer.length === 0) return null

  // BMP 转 PNG（WSL2 默认复制为 BMP）
  if (imageBuffer[0] === 0x42 && imageBuffer[1] === 0x4d) {
    const sharp = await getImageProcessor()
    imageBuffer = await sharp(imageBuffer).png().toBuffer()
  }

  // 尺寸优化
  const ext = extname(imagePath).slice(1).toLowerCase() || 'png'
  const resized = await maybeResizeAndDownsampleImageBuffer(
    imageBuffer,
    imageBuffer.length,
    ext
  )

  return {
    path: imagePath,
    base64: resized.buffer.toString('base64'),
    mediaType: detectImageFormatFromBase64(resized.buffer.toString('base64')),
    dimensions: resized.dimensions,
  }
}
```

---

## 关键代码路径与文件引用

### 导出位置
- **文件**：`src/utils/imagePaste.ts`（416 行，约 14KB）
- **导出常量**：
  - `PASTE_THRESHOLD` - 大粘贴阈值（800 字符）
  - `IMAGE_EXTENSION_REGEX` - 图像扩展名正则
- **导出函数**：
  - `hasImageInClipboard()` - 检测剪贴板是否有图像
  - `getImageFromClipboard()` - 从剪贴板读取图像
  - `getImagePathFromClipboard()` - 从剪贴板获取图像路径
  - `isImageFilePath(text)` - 检查文本是否为图像路径
  - `asImageFilePath(text)` - 清理并返回图像路径
  - `tryReadImageFromPath(text)` - 尝试从路径读取图像
- **导出类型**：
  - `ImageWithDimensions` - 带尺寸信息的图像数据

### 调用方分布

| 文件路径 | 使用场景 |
|---------|---------|
| `src/hooks/usePasteHandler.ts` | 粘贴事件处理，集成图像粘贴 |
| `src/hooks/useClipboardImageHint.ts` | 剪贴板图像提示检测 |
| `src/components/PromptInput/PromptInput.tsx` | 提示输入图像处理 |
| `src/tools/BriefTool/attachments.ts` | 附件上传图像处理 |
| `src/components/CustomSelect/select-input-option.tsx` | 选择输入选项图像支持 |

### 依赖导入

```typescript
import { feature } from 'bun:bundle'                          // 功能开关
import { randomBytes } from 'crypto'                          // 随机盐生成
import { execa } from 'execa'                                 // 命令执行
import { basename, extname, isAbsolute, join } from 'path'    // 路径操作
import { IMAGE_MAX_HEIGHT, IMAGE_MAX_WIDTH, IMAGE_TARGET_RAW_SIZE } from '../constants/apiLimits.js'
import { getFeatureValue_CACHED_MAY_BE_STALE } from '../services/analytics/growthbook.js'
import { getImageProcessor } from '../tools/FileReadTool/imageProcessor.js'
import { logForDebugging } from './debug.js'
import { execFileNoThrowWithCwd } from './execFileNoThrow.js'
import { getFsImplementation } from './fsOperations.js'
import { detectImageFormatFromBase64, type ImageDimensions, maybeResizeAndDownsampleImageBuffer } from './imageResizer.js'
import { logError } from './log.js'
```

---

## 依赖与外部交互

### 外部依赖

| 包名 | 用途 |
|------|------|
| `bun:bundle` | `feature()` - 功能开关检查 |
| `execa` | 执行剪贴板命令 |

### Node.js 内置模块

| 模块 | 用途 |
|------|------|
| `crypto` | `randomBytes` - 生成随机盐 |
| `path` | 路径操作 |

### 内部依赖

| 模块 | 用途 |
|------|------|
| `constants/apiLimits.js` | 图像尺寸和大小限制常量 |
| `services/analytics/growthbook.js` | 功能开关值获取 |
| `tools/FileReadTool/imageProcessor.js` | Sharp 图像处理 |
| `utils/debug.js` | 调试日志 |
| `utils/execFileNoThrow.js` | 安全命令执行 |
| `utils/fsOperations.js` | 文件系统操作 |
| `utils/imageResizer.js` | 图像尺寸调整 |
| `utils/log.js` | 错误日志 |

### 原生模块

| 模块 | 用途 | 加载方式 |
|------|------|---------|
| `image-processor-napi` | 原生 NSPasteboard 读取 | 动态 `import()` |

### 环境变量

| 环境变量 | 用途 |
|---------|------|
| `CLAUDE_CODE_TMPDIR` | 临时文件目录覆盖 |

### 系统命令依赖

| 平台 | 命令 | 用途 |
|------|------|------|
| macOS | `osascript` | AppleScript 执行 |
| Linux | `xclip` | X11 剪贴板访问 |
| Linux | `wl-paste` | Wayland 剪贴板访问 |
| Windows | `powershell` | 剪贴板访问 |

---

## 风险、边界与改进建议

### 已知风险

1. **原生模块加载失败**
   - `image-processor-napi` 可能未安装或平台不兼容
   - **缓解**：动态导入 + try-catch，失败回退到命令行方式

2. **剪贴板命令不可用**
   - Linux 系统可能未安装 `xclip` 或 `wl-paste`
   - **缓解**：命令执行使用 `reject: false`，失败时返回 `null`

3. **临时文件残留**
   - 图像提取到临时文件后，删除操作是 fire-and-forget
   - 进程崩溃可能导致临时文件残留
   - **缓解**：使用系统临时目录，通常会被系统自动清理

4. **安全性问题**
   - 从剪贴板读取的文件路径可能包含恶意构造的内容
   - **缓解**：`stripBackslashEscapes` 使用随机盐防止占位符注入

5. **大图像内存占用**
   - 读取大图像文件时可能占用大量内存
   - **缓解**：`maybeResizeAndDownsampleImageBuffer` 限制最大尺寸

### 边界情况

| 场景 | 处理 |
|------|------|
| 剪贴板无图像 | 返回 `null` |
| 原生模块不可用 | 回退到命令行方式 |
| 命令执行失败 | 返回 `null` |
| 图像超过尺寸限制 | 自动降采样 |
| 图像超过大小限制 | 自动压缩或转 JPEG |
| BMP 格式 | 自动转 PNG |
| 路径包含特殊字符 | 使用随机盐防止注入 |
| 相对路径 | 尝试从剪贴板匹配完整路径 |
| 空图像文件 | 记录警告，返回 `null` |

### 改进建议

1. **添加图像格式验证**
   ```typescript
   // 使用 magic bytes 验证图像格式
   function validateImageFormat(buffer: Buffer, expectedExt: string): boolean {
     const signatures = {
       png: [0x89, 0x50, 0x4e, 0x47],
       jpg: [0xff, 0xd8, 0xff],
       // ...
     }
     const sig = signatures[expectedExt]
     return buffer.slice(0, sig.length).equals(Buffer.from(sig))
   }
   ```

2. **支持更多图像格式**
   - AVIF
   - HEIC/HEIF（iOS 截图格式）
   - TIFF

3. **剪贴板监控 API**
   ```typescript
   // 提供事件驱动的剪贴板变化监听
   export function watchClipboard(callback: (hasImage: boolean) => void): () => void
   ```

4. **渐进式图像加载**
   ```typescript
   // 大图像先显示缩略图，后台加载完整图像
   export async function getImageFromClipboardProgressive(
     onProgress: (progress: number) => void
   ): Promise<ImageWithDimensions | null>
   ```

5. **图像元数据保留**
   ```typescript
   export type ImageWithMetadata = ImageWithDimensions & {
     exif?: {
       createdAt?: Date
       device?: string
       location?: { lat: number; lng: number }
     }
   }
   ```

6. **安全性增强**
   ```typescript
   // 限制可访问的文件路径范围
   const ALLOWED_PATHS = [
     process.env.HOME,
     process.env.USERPROFILE,
     '/tmp',
     '/var/tmp',
   ]
   
   function isPathAllowed(path: string): boolean {
     return ALLOWED_PATHS.some(allowed => path.startsWith(allowed))
   }
   ```

7. **单元测试覆盖**
   - 各种图像格式的读取和转换
   - 路径清理逻辑（注入攻击防护）
   - 平台特定命令的错误处理
   - 尺寸和大小限制边界条件
