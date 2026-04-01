# src/utils/imageValidation.ts 研究文档

## 场景与职责

`imageValidation.ts` 是 API 边界的最后一道图像大小防线。它的职责是在消息发往 Anthropic API 之前，快速扫描消息数组中的所有 base64 图像块，确保没有任何图像的 base64 编码长度超过 `API_IMAGE_MAX_BASE64_SIZE`（5MB）。

该模块定位明确：**不是图像处理模块，而是验证模块**。它不对图像进行任何修改、压缩或格式转换，仅在发现超限时抛出清晰的错误，让用户知道哪些图像需要手动处理。

调用方主要是 `src/utils/messages.ts`，在消息序列化或发送前进行最终校验。

## 功能点目的

### 1. `validateImagesForAPI`
核心验证函数，接受一个 `unknown[]` 消息数组，遍历其中所有用户消息（`type === 'user'`）的内容块，检查每个 `base64` 类型图像块的 `source.data.length` 是否超过 5MB。

设计上的关键考量：
- **检查的是 base64 字符串长度，不是解码后的 buffer 长度**。因为 API 限制的是 base64 payload 大小，而非原始字节数。
- **兼容两种消息格式**：
  - 包装格式 `{ type: 'user', message: { role, content } }`（内部 `UserMessage` 类型）
  - 原始格式 `{ role, content }`（Anthropic SDK `MessageParam` 类型）
- **多图像错误聚合**：若有多张图像超限，会一次性收集所有违规图像的索引和大小，抛出包含全部信息的 `ImageSizeError`。

### 2. `ImageSizeError`
自定义错误类，根据违规图像数量生成不同的错误消息：
- 单张图像："Image base64 size (X) exceeds API limit (Y). Please resize the image before sending."
- 多张图像："N images exceed the API limit (Y): Image 1: X1, Image 2: X2, ... Please resize these images before sending."

### 3. `isBase64ImageBlock`
内部类型守卫函数，用于安全地识别 `{ type: 'image', source: { type: 'base64', data: string } }` 结构。

## 具体技术实现

### 遍历逻辑
```typescript
for (const msg of messages) {
  if (typeof msg !== 'object' || msg === null) continue
  const m = msg as Record<string, unknown>
  if (m.type !== 'user') continue  // 只检查用户消息

  const innerMessage = m.message as Record<string, unknown> | undefined
  if (!innerMessage) continue

  const content = innerMessage.content
  if (typeof content === 'string' || !Array.isArray(content)) continue

  for (const block of content) {
    if (isBase64ImageBlock(block)) {
      imageIndex++
      const base64Size = block.source.data.length
      if (base64Size > API_IMAGE_MAX_BASE64_SIZE) {
        logEvent('tengu_image_api_validation_failed', { base64_size_bytes: base64Size, max_bytes: API_IMAGE_MAX_BASE64_SIZE })
        oversizedImages.push({ index: imageIndex, size: base64Size })
      }
    }
  }
}
```

### 图像索引计数
`imageIndex` 是一个跨消息累加的计数器，用于在错误消息中告诉用户"第几张图"出了问题。注意它从 0 开始累加，但显示时以 1-based 索引呈现（由 `ImageSizeError` 的调用方或消息模板决定，此处代码中 `index` 实际上是 0-based 的累加值，但错误消息中直接用 `img.index` 展示）。

> 经仔细审阅代码：`imageIndex` 初始为 0，每遇到一个 base64 图像块就 `imageIndex++`，所以 `index` 实际上是 1-based 的（因为第一次遇到时变为 1）。

### 错误上报
每次发现超限图像时，都会调用：
```typescript
logEvent('tengu_image_api_validation_failed', {
  base64_size_bytes: base64Size,
  max_bytes: API_IMAGE_MAX_BASE64_SIZE,
})
```

## 关键代码路径与文件引用

| 路径 | 作用 |
|------|------|
| `src/utils/imageValidation.ts:65-104` | `validateImagesForAPI` 核心验证 |
| `src/utils/imageValidation.ts:16-35` | `ImageSizeError` 错误类 |
| `src/utils/imageValidation.ts:40-49` | `isBase64ImageBlock` 类型守卫 |
| `src/constants/apiLimits.ts` | `API_IMAGE_MAX_BASE64_SIZE` 常量 |
| `src/utils/format.ts` | `formatFileSize` 格式化文件大小 |
| `src/services/analytics/index.ts` | `logEvent` 遥测 |

## 依赖与外部交互

### 内部依赖
- `../constants/apiLimits.js`：`API_IMAGE_MAX_BASESS64_SIZE`
- `../services/analytics/index.js`：`logEvent`
- `./format.js`：`formatFileSize`

### 调用方
- `src/utils/messages.ts`：在消息组装或发送前调用验证

## 风险、边界与改进建议

### 风险与边界
1. **验证时机偏晚**：该模块在 API 请求边界处验证，若上游处理模块（如 `imageResizer.ts`）未能成功压缩图像，此时抛出错误会导致用户已完成的输入被中断。虽然这是预期的安全网行为，但理想情况下上游不应让超限图像到达此处。
2. **仅验证 base64 图像**：对于 `source.type !== 'base64'` 的图像（如未来可能支持的 URL 引用图像），该验证完全跳过。若 API 未来对 URL 图像也有大小限制，需要扩展验证逻辑。
3. **消息格式硬编码**：`m.type !== 'user'` 的过滤逻辑假设只有用户消息包含图像。若未来助手消息或系统消息也包含图像，验证会遗漏。
4. **无总请求大小校验**：API 还有 32MB 的总请求大小限制（见 `apiLimits.ts` 中 PDF 相关注释），但该模块只校验单张图像，不校验整个请求体大小。
5. **性能问题**：对于包含大量图像的消息，线性扫描所有内容块的时间复杂度为 O(n)，虽然通常 n 很小，但极端情况下（如 100 张图）可能 noticeable。

### 改进建议
1. **增加总请求大小校验**：在 `validateImagesForAPI` 或相邻模块中，计算所有消息内容的总大小，确保不超过 API 总请求限制。
2. **扩展图像来源支持**：若未来支持非 base64 图像源，更新 `isBase64ImageBlock` 为更通用的图像块检测器。
3. **早期拒绝**：考虑在图像被用户粘贴时就进行大小估算（base64 长度 ≈ 原始大小 × 4/3），在输入阶段就提示用户，而不是等到发送时才报错。
4. **索引准确性**：当前 `imageIndex` 是全局计数，但用户可能更关心"在当前消息中是第几张图"。可以考虑同时提供全局索引和局部索引。
5. **与 imageResizer 联动**：当验证失败时，错误消息可以更智能地提示用户"请尝试使用更小的图像或调整图像尺寸"，甚至提供自动压缩的入口（如果错误是在可恢复的场景中抛出）。
