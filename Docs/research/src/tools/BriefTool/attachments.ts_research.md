# attachments.ts 深度研究文档

## 场景与职责

attachments.ts 是 BriefTool 的**附件处理核心模块**，负责附件的验证和解析。它作为 `SendUserMessage` 和 `SendUserFile` 工具的共享基础设施，实现了以下关键职责：

1. **附件路径验证**：验证用户提供的文件路径是否有效、可访问
2. **附件元数据解析**：获取文件大小、类型（图片/普通文件）等信息
3. **条件性上传**：在 Bridge 模式下将附件上传到私有 API，供 Web 查看器预览

该模块的设计充分考虑了**树摇优化（Tree Shaking）**：上传相关代码（axios、crypto、auth utils）仅在 `BRIDGE_MODE` 构建时包含，非 Bridge 构建完全排除。

## 功能点目的

### 1. validateAttachmentPaths()

**职责**：验证附件路径的有效性和可访问性

**验证流程**：
1. 遍历每个路径，使用 `expandPath()` 展开（支持 `~` 和相对路径）
2. 调用 `fs.stat()` 检查文件状态
3. 验证是否为普通文件（非目录）
4. 处理常见错误：
   - `ENOENT`：文件不存在，返回带当前工作目录的友好错误
   - `EACCES`/`EPERM`：权限不足
   - 其他错误：向上抛出

**返回值**：
```typescript
// 成功
{ result: true }

// 失败
{
  result: false,
  message: string,      // 用户友好的错误信息
  errorCode: 1          // 错误代码
}
```

### 2. resolveAttachments()

**职责**：解析附件元数据并可选上传

**处理流程**：
1. **串行 stat**：按顺序获取每个文件的元数据（保持顺序确定性）
2. **构建元数据**：创建 `ResolvedAttachment` 对象数组
3. **条件上传**（仅 BRIDGE_MODE）：
   - 检查 `replBridgeEnabled` 或 `CLAUDE_CODE_BRIEF_UPLOAD` 环境变量
   - 并行上传所有附件
   - 上传失败返回 `undefined`，不影响本地渲染

**返回数据结构**：
```typescript
ResolvedAttachment = {
  path: string;        // 绝对路径
  size: number;        // 文件大小（字节）
  isImage: boolean;    // 是否为图片（基于扩展名）
  file_uuid?: string;  // 上传成功后返回的 UUID
}
```

## 具体技术实现

### 关键流程

#### 验证流程
```
validateAttachmentPaths(rawPaths: string[])
  ├── 获取当前工作目录 cwd
  ├── 遍历 rawPaths
  │   ├── expandPath(rawPath) → fullPath
  │   ├── stat(fullPath)
  │   │   ├── 成功且是文件 → 继续下一个
  │   │   ├── 成功但是目录 → 返回错误 "is not a regular file"
  │   │   └── 失败
  │   │       ├── ENOENT → 返回错误 "does not exist"（带 cwd）
  │   │       ├── EACCES/EPERM → 返回错误 "permission denied"
  │   │       └── 其他 → 抛出异常
  │   └── ...
  └── 全部通过 → 返回 { result: true }
```

#### 解析流程
```
resolveAttachments(rawPaths, uploadCtx)
  ├── stated: ResolvedAttachment[] = []
  ├── 串行遍历 rawPaths
  │   ├── expandPath(rawPath) → fullPath
  │   ├── stat(fullPath)  // TOCTOU 注释说明
  │   └── stated.push({
  │       path: fullPath,
  │       size: stats.size,
  │       isImage: IMAGE_EXTENSION_REGEX.test(fullPath)
  │     })
  │
  ├── 检查 feature('BRIDGE_MODE')
  │   └── 否 → 直接返回 stated
  │
  ├── 是 BRIDGE_MODE
  │   ├── 计算 shouldUpload = replBridgeEnabled || CLAUDE_CODE_BRIEF_UPLOAD
  │   ├── 动态导入 './upload.js'  // 树摇优化关键
  │   ├── 并行上传：Promise.all(stated.map(...))
  │   └── 合并结果：将 file_uuid 添加到对应 attachment
  │
  └── 返回附件数组
```

### 数据结构

#### ResolvedAttachment
```typescript
export type ResolvedAttachment = {
  path: string;        // 展开后的绝对路径
  size: number;        // 文件大小（字节）
  isImage: boolean;    // 基于 IMAGE_EXTENSION_REGEX 判断
  file_uuid?: string;  // 可选：上传后的远程标识
}
```

#### ValidationResult（来自 Tool.ts）
```typescript
type ValidationResult =
  | { result: true }
  | { result: false; message: string; errorCode: number }
```

### 图片检测

使用正则表达式检测图片扩展名：
```typescript
// 来自 src/utils/imagePaste.ts
IMAGE_EXTENSION_REGEX = /\.(png|jpe?g|gif|webp)$/i
```

**注意**：与 `upload.ts` 中的 `MIME_BY_EXT` 保持同步，支持的格式：
- `.png` → `image/png`
- `.jpg`/`.jpeg` → `image/jpeg`
- `.gif` → `image/gif`
- `.webp` → `image/webp`

### 树摇优化（Tree Shaking）

**关键设计**：
```typescript
// 动态导入在 feature() 守卫内部
if (feature('BRIDGE_MODE')) {
  const { uploadBriefAttachment } = await import('./upload.js')
  // ...
}
```

**效果**：
- 非 `BRIDGE_MODE` 构建：`upload.ts` 及其依赖（axios、crypto、zod、auth utils）完全从产物中排除
- `BRIDGE_MODE` 构建：包含完整上传逻辑

**引用 CLAUDE.md**：
> "helpers defined outside remain in the build even if never called"

因此必须将 `import` 语句放在 `feature()` 守卫内部，而非模块顶部。

## 关键代码路径与文件引用

### 核心文件
| 文件 | 职责 |
|------|------|
| `attachments.ts` | 附件验证和解析实现 |
| `upload.ts` | 附件上传实现（可选依赖） |

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/Tool.ts` | `ValidationResult` 类型 |
| `src/utils/cwd.ts` | `getCwd()` 获取当前工作目录 |
| `src/utils/envUtils.ts` | `isEnvTruthy()` 环境变量检查 |
| `src/utils/errors.ts` | `getErrnoCode()` 错误码提取 |
| `src/utils/imagePaste.ts` | `IMAGE_EXTENSION_REGEX` 图片扩展名正则 |
| `src/utils/path.ts` | `expandPath()` 路径展开 |

### 调用方
- `BriefTool.ts`：`validateInput` 调用 `validateAttachmentPaths`，`call` 调用 `resolveAttachments`
- `SendUserFile` 工具（如存在）：共享附件处理逻辑

## 依赖与外部交互

### 文件系统操作
- `fs/promises.stat`：异步获取文件元数据
- 路径操作：`expandPath()` 处理 `~` 展开和相对路径解析

### 环境变量
| 变量 | 用途 |
|------|------|
| `CLAUDE_CODE_BRIEF_UPLOAD` | 允许非 TTY 宿主（如 cowork desktop bridge）上传附件 |

### 构建时特性标志
- `BRIDGE_MODE`：控制是否包含上传功能代码

### 外部 API（间接通过 upload.ts）
- **私有 API**：`/api/oauth/file_upload`（multipart/form-data 上传）

## 风险、边界与改进建议

### 风险点

1. **TOCTOU（Time-of-Check-Time-of-Use）竞争条件**
   ```typescript
   // 注释说明：
   // validateInput ran before us, but the file could have moved since
   // (TOCTOU); if it did, let the error propagate so the model sees it.
   ```
   - 验证和调用之间文件可能被修改或删除
   - 当前策略是让错误传播给模型
   - 建议：考虑添加重试逻辑或更优雅的错误恢复

2. **大文件 stat 阻塞**
   - 串行 stat 大量大文件可能耗时
   - 虽然本地文件通常很快，但网络文件系统可能延迟
   - 建议：添加超时机制或并行 stat

3. **图片检测局限性**
   - 仅基于扩展名判断，可能被欺骗
   - 不支持 SVG（被 `upload.ts` 明确排除）
   - 建议：考虑添加 MIME 类型检测作为二次验证

4. **上传失败静默处理**
   - `uploadBriefAttachment` 失败返回 `undefined`
   - 附件仍保留本地元数据，但 Web 预览不可用
   - 用户无感知，可能导致困惑
   - 建议：添加上传失败的通知机制（可选）

### 边界情况

1. **符号链接**
   - `stat()` 默认跟随符号链接
   - 循环链接会导致 `ELOOP` 错误
   - 建议：添加符号链接检测和限制

2. **特殊文件**
   - 设备文件、FIFO 等通过 `stats.isFile()` 过滤
   - 但某些系统上可能有例外
   - 建议：添加文件类型白名单验证

3. **路径长度限制**
   - 依赖 Node.js 的路径处理
   - 极长路径在某些文件系统上可能失败
   - 建议：添加路径长度预检查

4. **并发附件数量**
   - 无明确数量限制
   - 大量附件（数百个）可能导致内存或性能问题
   - 建议：添加附件数量上限

### 改进建议

1. **性能优化**
   - 将串行 stat 改为并行（保持顺序可通过索引映射）
   - 添加文件大小预检查，避免 stat 超大文件
   - 考虑使用 `fs.promises.opendir` 批量处理

2. **安全增强**
   - 添加路径遍历防护（虽然 `expandPath` 有基础防护）
   - 限制可访问的文件范围（如项目目录外需要确认）
   - 添加文件内容类型验证（魔术字节检测）

3. **可观测性**
   - 记录附件处理指标（数量、大小、上传成功率）
   - 添加上传延迟监控
   - 记录 TOCTOU 错误频率

4. **用户体验**
   - 大附件上传进度指示
   - 上传失败时提供重试选项
   - 附件大小预估和警告

5. **代码结构**
   - 将验证和解析逻辑进一步分离
   - 添加更详细的 JSDoc 注释
   - 考虑将图片类型检测配置化
