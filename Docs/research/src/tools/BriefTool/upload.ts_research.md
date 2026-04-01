# upload.ts 深度研究文档

## 场景与职责

upload.ts 是 BriefTool 的**附件上传模块**，负责将本地附件上传到 Anthropic 私有 API，使 Web 查看器能够预览这些文件。这是 Bridge 模式（`BRIDGE_MODE`）特有的功能，实现了本地终端与 Web 界面之间的文件共享。

核心职责：
1. **文件上传**：将附件上传到 `/api/oauth/file_upload` 端点
2. **认证处理**：管理 OAuth token 和 API 基础 URL
3. **MIME 类型检测**：根据文件扩展名推断内容类型
4. **优雅降级**：任何失败都不影响本地功能，仅影响 Web 预览

## 功能点目的

### 1. uploadBriefAttachment()

**职责**：上传单个附件文件

**前置检查**（按顺序）：
1. **功能标志**：`feature('BRIDGE_MODE')` 必须为真
2. **Bridge 启用**：`ctx.replBridgeEnabled` 必须为真
3. **文件大小**：不超过 `MAX_UPLOAD_BYTES`（30MB）
4. **认证令牌**：必须存在有效的 OAuth token
5. **文件可读**：能够读取文件内容

**上传流程**：
1. 读取文件内容为 Buffer
2. 构建 multipart/form-data 请求体（手动组装）
3. 发送 POST 请求到 `/api/oauth/file_upload`
4. 解析响应，提取 `file_uuid`

**返回**：
- 成功：返回 `file_uuid` 字符串
- 失败：返回 `undefined`（优雅降级）

### 2. MIME 类型检测

支持的图片格式：
| 扩展名 | MIME 类型 | 后端处理 |
|--------|-----------|----------|
| `.png` | `image/png` | `upload_image_wrapped`（生成预览/缩略图） |
| `.jpg`/`.jpeg` | `image/jpeg` | `upload_image_wrapped` |
| `.gif` | `image/gif` | `upload_image_wrapped` |
| `.webp` | `image/webp` | `upload_image_wrapped` |
| 其他 | `application/octet-stream` | `upload_generic_file`（仅原始文件） |

**注意**：SVG、BMP、ICO、PDF 被明确排除，因为后端转码器可能返回 400 错误。

### 3. 基础 URL 解析

优先级（从高到低）：
1. `getBridgeBaseUrlOverride()`：Ant 开发环境覆盖（`CLAUDE_BRIDGE_BASE_URL`）
2. `process.env.ANTHROPIC_BASE_URL`：子进程宿主传递（如 cowork desktop bridge）
3. `getOauthConfig().BASE_API_URL`：默认 OAuth 配置

**关键设计**：支持子进程宿主（如 cowork）传递自定义基础 URL，避免 staging token 命中生产 API 导致 401。

## 具体技术实现

### 关键流程

#### 上传决策流程
```
uploadBriefAttachment(fullPath, size, ctx)
  ├── feature('BRIDGE_MODE')?
  │   └── 否 → 返回 undefined
  ├── ctx.replBridgeEnabled?
  │   └── 否 → 返回 undefined
  ├── size > MAX_UPLOAD_BYTES (30MB)?
  │   └── 是 → 记录 debug 日志，返回 undefined
  ├── getBridgeAccessToken()?
  │   └── 否 → 记录 debug 日志，返回 undefined
  ├── readFile(fullPath)
  │   └── 失败 → 记录 debug 日志，返回 undefined
  ├── 构建 multipart 请求体
  │   ├── boundary = "----FormBoundary" + randomUUID()
  │   ├── headers: Content-Disposition, Content-Type
  │   └── body: headers + content + closing boundary
  ├── axios.post(url, body, { headers, timeout, signal })
  │   ├── status !== 201 → 记录 debug，返回 undefined
  │   ├── 解析响应
  │   │   ├── 无效格式 → 记录 debug，返回 undefined
  │   │   └── 成功 → 返回 file_uuid
  │   └── 异常 → 记录 debug，返回 undefined
  └── 返回 undefined（所有失败路径）
```

### 数据结构

#### 请求体格式（multipart/form-data）
```
------FormBoundary<uuid>
Content-Disposition: form-data; name="file"; filename="<filename>"
Content-Type: <mimeType>

<raw file content>
------FormBoundary<uuid>--
```

#### 响应 Schema
```typescript
const uploadResponseSchema = z.object({
  file_uuid: z.string()
})
```

#### 上传上下文
```typescript
export type BriefUploadContext = {
  replBridgeEnabled: boolean  // Bridge 是否启用
  signal?: AbortSignal        // 用于取消上传
}
```

### 常量定义

```typescript
const MAX_UPLOAD_BYTES = 30 * 1024 * 1024  // 30MB，与后端限制一致
const UPLOAD_TIMEOUT_MS = 30_000           // 30秒超时
```

### 手动 multipart 组装

```typescript
const body = Buffer.concat([
  Buffer.from(
    `--${boundary}\r\n` +
    `Content-Disposition: form-data; name="file"; filename="${filename}"\r\n` +
    `Content-Type: ${mimeType}\r\n\r\n`,
  ),
  content,
  Buffer.from(`\r\n--${boundary}--\r\n`),
])
```

**设计原因**：
- 与 `filesApi.ts` 使用相同模式
- OAuth 端点只需要单个 "file" 部分（不像公共 Files API 需要 "purpose" 字段）
- 避免引入额外的 multipart 库依赖

## 关键代码路径与文件引用

### 核心文件
| 文件 | 职责 |
|------|------|
| `upload.ts` | 附件上传实现 |

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/bridge/bridgeConfig.ts` | `getBridgeAccessToken()`, `getBridgeBaseUrlOverride()` |
| `src/constants/oauth.ts` | `getOauthConfig()` |
| `src/utils/debug.ts` | `logForDebugging()` |
| `src/utils/lazySchema.ts` | `lazySchema()` 延迟 schema 加载 |
| `src/utils/slowOperations.ts` | `jsonStringify()` |

### 调用方
- `attachments.ts`：动态导入并调用 `uploadBriefAttachment`

### 相关模块
| 模块 | 关系 |
|------|------|
| `attachments.ts` | 调用方，控制上传触发条件 |
| `filesApi.ts` | 参考实现（相同的 multipart 模式） |

## 依赖与外部交互

### 外部库
- **axios**：HTTP 客户端，用于上传请求
- **zod**：响应数据验证
- **crypto**：`randomUUID()` 生成 multipart boundary

### 环境变量
| 变量 | 用途 |
|------|------|
| `CLAUDE_BRIDGE_OAUTH_TOKEN` | Ant 开发环境 OAuth token 覆盖 |
| `CLAUDE_BRIDGE_BASE_URL` | Ant 开发环境 API URL 覆盖 |
| `ANTHROPIC_BASE_URL` | 子进程宿主传递的 API URL |

### 构建时特性标志
- `BRIDGE_MODE`：控制是否包含上传功能

### 外部 API
- **端点**：`POST /api/oauth/file_upload`
- **认证**：`Authorization: Bearer <token>`
- **内容类型**：`multipart/form-data; boundary=<boundary>`
- **响应**：`{ file_uuid: string }`（201 Created）

### 后端处理逻辑
- **图片**（`image/*`）：`upload_image_wrapped`
  - 写入 PREVIEW/THUMBNAIL
  - 不保留 ORIGINAL
- **其他文件**：`upload_generic_file`
  - 仅保留 ORIGINAL
  - 无预览
- **PDF**：`upload_pdf_file_wrapped`
  - 特殊处理
  - 跳过 ORIGINAL

## 风险、边界与改进建议

### 风险点

1. **完全静默失败**
   - 所有错误路径都返回 `undefined`，仅记录 debug 日志
   - 用户无法知道上传失败，Web 预览不可用
   - 调试困难（需要开启 debug 模式才能看到日志）
   - 建议：
     - 区分"可恢复失败"和"需要通知的失败"
     - 添加可选的错误报告机制
     - 在开发模式下显示上传状态

2. **文件大小检查前置**
   - 在读取文件前检查大小，避免大文件内存问题
   - 但 `stat` 和 `readFile` 之间文件可能增长
   - 建议：添加读取后二次检查

3. **MIME 类型欺骗**
   - 仅基于扩展名判断 MIME 类型
   - 恶意文件可能伪装成支持的图片格式
   - 建议：添加魔术字节验证

4. **超时设置**
   - 固定 30 秒超时，对大文件或慢网络可能不足
   - 无重试机制
   - 建议：
     - 基于文件大小动态计算超时
     - 添加指数退避重试

5. **内存使用**
   - 整个文件读取为 Buffer，再组装 multipart body
   - 大文件（接近 30MB）可能导致内存压力
   - 建议：考虑流式上传

### 边界情况

1. **特殊字符文件名**
   - 文件名直接拼接到 Content-Disposition 头
   - 特殊字符（如 `"`, `\r`, `\n`）可能导致 header 解析错误
   - 建议：添加文件名转义或验证

2. **空文件**
   - 空文件可以上传（0 字节）
   - 后端可能拒绝或特殊处理
   - 建议：添加空文件预检查

3. **并发上传**
   - 多个附件并行上传（由 `attachments.ts` 控制）
   - 大量并发可能导致网络或内存压力
   - 建议：添加上传并发限制

4. **取消信号**
   - 支持 `AbortSignal`，但测试覆盖可能不足
   - 取消后资源清理需要验证
   - 建议：添加取消场景测试

### 改进建议

1. **可观测性**
   - 添加上传成功率/延迟指标
   - 记录文件类型分布
   - 添加错误分类统计

2. **性能优化**
   - 实现流式上传，减少内存占用
   - 添加压缩（gzip）支持
   - 使用连接池复用 HTTP 连接

3. **健壮性**
   - 添加重试机制（指数退避）
   - 实现断点续传（大文件）
   - 添加校验和验证

4. **用户体验**
   - 大文件上传进度指示
   - 上传失败时提供重试选项
   - 预览生成状态反馈

5. **安全性**
   - 添加文件内容类型验证（魔术字节）
   - 实现文件名安全转义
   - 添加上传速率限制

6. **代码结构**
   - 将 multipart 组装抽取为可复用工具
   - 添加更详细的错误类型（而非统一返回 undefined）
   - 实现上传队列管理
