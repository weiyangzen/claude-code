# mcpValidation.ts 研究文档

## 场景与职责

本模块负责 MCP (Model Context Protocol) 工具输出的验证、截断和 token 计数管理。核心职责包括：

1. **Token 限制管理**：根据配置计算 MCP 输出的最大 token 数
2. **内容大小估算**：快速估算 MCP 输出内容的 token 数量
3. **智能截断**：当内容超过限制时，进行智能截断并添加提示信息
4. **图片处理**：在截断时处理图片块的压缩和尺寸调整

该模块是 MCP 输出质量控制的关键组件，确保输出不会超出 API 的 token 限制，同时尽可能保留有价值的内容。

## 功能点目的

### 1. `getMaxMcpOutputTokens()` - 最大输出 Token 数获取
- **目的**：解析并返回 MCP 输出的最大允许 token 数
- **优先级**：
  1. `MAX_MCP_OUTPUT_TOKENS` 环境变量（用户显式覆盖）
  2. GrowthBook flag `tengu_satin_quoll` 的 `mcp_tool` 键
  3. 硬编码默认值 `25000`

### 2. `getContentSizeEstimate()` - 内容大小估算
- **目的**：快速估算 MCP 输出的 token 数量，避免频繁的 API 调用
- **策略**：
  - 字符串内容：使用 `roughTokenCountEstimation` 估算
  - 文本块：同上
  - 图片块：固定估算 `IMAGE_TOKEN_ESTIMATE = 1600`

### 3. `mcpContentNeedsTruncation()` - 截断必要性检查
- **目的**：判断内容是否需要截断
- **优化策略**：
  - 先进行快速大小检查（阈值因子 0.5）
  - 超过阈值才调用 API 进行精确 token 计数
  - 错误时保守返回 `false`（不截断）

### 4. `truncateMcpContent()` / `truncateMcpContentIfNeeded()` - 内容截断
- **目的**：将超长的 MCP 输出截断到允许范围内
- **截断策略**：
  - 字符串：直接切片并添加截断提示
  - 内容块数组：逐个处理，优先保留文本，图片尝试压缩
  - 图片压缩：使用 `compressImageBlock` 尝试适配剩余空间

## 具体技术实现

### 关键流程

```
MCP 输出内容
    ↓
getContentSizeEstimate() 快速估算
    ↓
估算值 <= 阈值因子 × 最大限制?
    ↓
    是 → 不需要截断
    否 → 调用 API 精确计数
              ↓
         精确计数 > 最大限制?
              ↓
              是 → truncateMcpContent()
              否 → 不需要截断
```

### 数据结构

```typescript
// MCP 工具结果类型
export type MCPToolResult = string | ContentBlockParam[] | undefined

// 关键常量
const MCP_TOKEN_COUNT_THRESHOLD_FACTOR = 0.5  // 快速检查阈值因子
const IMAGE_TOKEN_ESTIMATE = 1600             // 图片 token 估算值
const DEFAULT_MAX_MCP_OUTPUT_TOKENS = 25000   // 默认最大 token 数
```

### 截断算法细节

1. **字符串截断**：
   ```typescript
   maxChars = getMaxMcpOutputTokens() * 4  // 假设平均 4 字符/token
   truncateString(content, maxChars) + truncationMsg
   ```

2. **内容块截断**：
   - 遍历每个块，累计字符数
   - 文本块：如果剩余空间足够则保留，否则切片
   - 图片块：尝试压缩以适应剩余空间
   - 最后添加截断提示块

3. **图片压缩策略**：
   ```typescript
   remainingChars = maxChars - currentChars
   remainingBytes = Math.floor(remainingChars * 0.75)  // base64 比例
   compressedBlock = await compressImageBlock(block, remainingBytes)
   ```

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `@anthropic-ai/sdk/resources/index.mjs` | SDK 类型定义 |
| `../services/analytics/growthbook.js` | 功能开关获取 |
| `../services/tokenEstimation.js` | Token 计数 API |
| `./imageResizer.js` | 图片压缩 |
| `./log.js` | 错误日志 |

### 调用方

| 调用方 | 用途 |
|--------|------|
| `src/services/mcp/client.ts` | MCP 客户端输出验证和截断 |
| `src/tools/MCPTool/UI.tsx` | MCP 工具 UI 渲染 |
| `src/utils/claudeInChrome/toolRendering.tsx` | Chrome 环境工具渲染 |
| `src/utils/computerUse/toolRendering.tsx` | Computer Use 工具渲染 |

### Token 计数服务

- `countMessagesTokensWithAPI()`：调用 Anthropic API 进行精确 token 计数
- `roughTokenCountEstimation()`：快速估算（约 4 字符/token）

## 风险、边界与改进建议

### 已知风险

1. **Token 估算误差**
   - 风险：`roughTokenCountEstimation` 使用简单启发式（4 字符/token）
   - 实际 token 数可能因语言、内容类型而异
   - 缓解：超过阈值后使用 API 精确计数

2. **图片压缩失败**
   - 风险：`compressImageBlock` 可能抛出异常
   - 当前处理：捕获异常，跳过该图片
   - 潜在问题：重要图片信息丢失

3. **截断提示位置**
   - 当前：截断提示追加在内容末尾
   - 风险：模型可能忽略末尾提示，不知道内容被截断

### 边界情况

| 场景 | 行为 |
|------|------|
| 内容为空/undefined | 返回原内容，不处理 |
| API 计数失败 | 保守返回 `false`，不截断 |
| 图片压缩失败 | 跳过该图片，继续处理其他块 |
| 单个图片超过整个预算 | 尝试压缩，失败则跳过 |
| 内容恰好等于限制 | 不截断，避免不必要的处理 |

### 改进建议

1. **动态字符/token 比例**
   - 当前固定 4:1 比例
   - 建议：根据实际内容类型（代码、中文、标记语言）动态调整

2. **智能截断策略**
   - 当前：简单切片
   - 建议：
     - 优先保留结构化内容（JSON/XML）的完整性
     - 在段落/句子边界处截断
     - 保留开头和结尾，截断中间

3. **图片处理优化**
   - 当前：固定 1600 token 估算
   - 建议：根据图片尺寸计算更准确的估算
   - 考虑图片内容重要性（OCR 检测文本密度）

4. **渐进式截断**
   - 当前：一次性截断到限制
   - 建议：先尝试轻度压缩/调整，逐步增加强度

5. **截断反馈机制**
   - 当前：静态提示信息
   - 建议：
     - 记录截断比例到遥测
     - 向用户显示截断警告（如果交互式会话）
     - 提供获取完整内容的方式

6. **配置热更新**
   - 当前：`getMaxMcpOutputTokens()` 每次调用都读取环境变量和 GrowthBook
   - 建议：添加缓存机制，定期刷新而非每次调用
