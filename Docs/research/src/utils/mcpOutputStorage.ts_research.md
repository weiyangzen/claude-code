# mcpOutputStorage.ts 研究文档

## 场景与职责

本模块负责 MCP (Model Context Protocol) 工具输出的持久化存储与格式处理。核心职责包括：

1. **大输出处理**：当 MCP 工具输出超过 token 限制时，将内容保存到磁盘而非直接返回给模型
2. **二进制内容持久化**：处理图片、PDF、音频等二进制内容的保存
3. **格式描述生成**：为不同类型的 MCP 结果生成人类可读的格式说明
4. **指令生成**：为 Claude 生成读取大文件的指导指令

该模块是 MCP 客户端与文件系统之间的桥梁，确保大输出不会导致 API token 超限，同时让模型能够按需读取内容。

## 功能点目的

### 1. `getFormatDescription()` - 格式描述生成
- **目的**：根据 MCP 结果类型（toolResult/structuredContent/contentArray）生成格式说明
- **场景**：用于告知 Claude 输出内容的格式，帮助其正确解析

### 2. `getLargeOutputInstructions()` - 大输出读取指令
- **目的**：当输出被保存到文件时，生成详细的读取指导
- **关键要求**：
  - 强制要求顺序读取文件内容
  - 处理截断警告（truncation warnings）
  - 要求明确声明已读取的内容比例

### 3. `extensionForMimeType()` - MIME 类型映射
- **目的**：将 MIME 类型映射为文件扩展名
- **支持类型**：PDF、JSON、CSV、图片（PNG/JPEG/GIF/WebP/SVG）、音视频、Office 文档等
- **安全设计**：未知类型默认返回 'bin'，避免路径注入

### 4. `isBinaryContentType()` - 二进制内容检测
- **目的**：判断内容类型是否为二进制（应保存到磁盘而非放入模型上下文）
- **启发式规则**：
  - `text/*` 类型视为文本
  - `application/json`、`application/xml`、JavaScript 等视为文本
  - 其他 `application/*` 类型视为二进制

### 5. `persistBinaryContent()` - 二进制内容持久化
- **目的**：将原始字节写入工具结果目录
- **特点**：
  - 使用 MIME 类型推导扩展名
  - 返回文件路径、大小、扩展名信息
  - 记录遥测事件 `tengu_binary_content_persisted`

### 6. `getBinaryBlobSavedMessage()` - 保存确认消息
- **目的**：生成简洁的二进制内容保存通知
- **输出格式**：包含文件路径、MIME 类型、文件大小

## 具体技术实现

### 关键流程

```
MCP 工具输出
    ↓
判断输出大小/类型
    ↓
┌─────────────────┬─────────────────┐
↓                 ↓                 ↓
文本大输出      二进制内容       小输出
    ↓              ↓               ↓
生成读取指令   persistBinary   直接返回
    ↓              ↓
保存到磁盘    返回文件信息
    ↓
返回文件引用
```

### 数据结构

```typescript
// 二进制持久化结果
export type PersistBinaryResult =
  | { filepath: string; size: number; ext: string }
  | { error: string }
```

### 关键代码路径

1. **文件保存路径**：`tool-results/${persistId}.${ext}`
   - 通过 `ensureToolResultsDir()` 确保目录存在
   - 使用 `getToolResultsDir()` 获取基础路径

2. **MIME 类型处理**：
   - 去除 charset/boundary 参数：`mimeType.split(';')[0]`
   - 大小写不敏感比较：`.trim().toLowerCase()`

3. **错误处理**：
   - 使用 `toError()` 统一错误转换
   - 使用 `logError()` 记录错误详情

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `fs/promises` | 文件写入操作 |
| `path` | 路径拼接 |
| `../services/analytics/index.js` | 遥测事件记录 |
| `../services/mcp/client.js` | MCP 结果类型定义 |
| `./errors.js` | 错误处理工具 |
| `./format.js` | 文件大小格式化 |
| `./log.js` | 日志记录 |
| `./toolResultStorage.js` | 工具结果目录管理 |

### 调用方

| 调用方 | 用途 |
|--------|------|
| `src/services/mcp/client.ts` | MCP 客户端保存大输出和二进制内容 |
| `src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts` | 读取 MCP 资源时的输出处理 |
| `src/tools/WebFetchTool/utils.ts` | Web 获取内容的持久化 |

### 遥测事件

- `tengu_binary_content_persisted`：二进制内容保存时触发
  - 元数据：mimeType, sizeBytes, ext

## 风险、边界与改进建议

### 已知风险

1. **路径遍历风险**
   - 缓解：`extensionForMimeType` 使用固定词汇表，不直接使用用户输入
   - `persistId` 由调用方生成，需确保其安全性

2. **磁盘空间耗尽**
   - 风险：大输出持续保存可能占满磁盘
   - 现状：依赖外部清理机制（如 `cleanup.ts`）

3. **MIME 类型欺骗**
   - 风险：攻击者可能伪装 MIME 类型
   - 缓解：实际文件内容验证由调用方负责

### 边界情况

| 场景 | 行为 |
|------|------|
| MIME 类型为空 | 返回 'bin' 扩展名 |
| 文件写入失败 | 返回 `{ error: string }` 结构 |
| 未知 MIME 类型 | 保守返回 'bin' |
| 内容类型含参数 | 仅使用主类型（分号前部分） |

### 改进建议

1. **添加文件大小限制**
   - 当前 `persistBinaryContent` 无大小检查
   - 建议添加 `MAX_BINARY_SIZE` 常量防止超大文件写入

2. **异步流式写入**
   - 当前使用 `writeFile` 一次性写入
   - 对于超大内容，可考虑流式写入减少内存占用

3. **清理策略集成**
   - 考虑在模块内集成过期文件自动清理
   - 或提供 `cleanupBinaryContent()` 接口

4. **MIME 类型验证**
   - 考虑使用 `file-type` 等库验证实际内容 vs 声明的 MIME 类型
   - 防止 MIME 欺骗攻击

5. **压缩支持**
   - 对于文本类大内容，可考虑 gzip 压缩后存储
   - 减少磁盘占用和后续读取的 I/O
