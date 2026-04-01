# pdfUtils.ts 研究文档

## 场景与职责

本模块提供 PDF 相关的工具函数，主要用于页面范围解析和 PDF 支持检测。核心职责包括：

1. **页面范围解析**：将用户输入的页面范围字符串解析为 firstPage/lastPage 数字
2. **PDF 支持检测**：检测当前模型是否支持 PDF document blocks
3. **PDF 扩展名检测**：判断文件扩展名是否为 PDF

该模块是 FileReadTool 的辅助模块，用于处理用户指定的 PDF 页面范围和功能可用性检测。

## 功能点目的

### 1. `parsePDFPageRange()` - 页面范围解析
- **目的**：解析用户输入的页面范围字符串
- **支持格式**：
  - `"5"` → `{ firstPage: 5, lastPage: 5 }`（单页）
  - `"1-10"` → `{ firstPage: 1, lastPage: 10 }`（范围）
  - `"3-"` → `{ firstPage: 3, lastPage: Infinity }`（从第 3 页到末尾）
- **验证规则**：
  - 页码必须为正整数
  - 结束页必须大于等于起始页
  - 无效输入返回 `null`

### 2. `isPDFSupported()` - PDF 支持检测
- **目的**：检测当前模型是否支持 PDF document blocks
- **实现**：检查模型名称是否包含 `claude-3-haiku`
- **背景**：Haiku 3 是唯一不支持 PDF 的模型，其他模型（1P、Vertex、Bedrock、Foundry）都支持
- **用途**：决定使用 PDF document blocks 还是页面提取（poppler-utils）

### 3. `isPDFExtension()` - PDF 扩展名检测
- **目的**：判断文件扩展名是否为 PDF
- **输入**：带或不带前导点的扩展名
- **实现**：标准化后检查是否在 `DOCUMENT_EXTENSIONS` 集合中

## 具体技术实现

### 关键流程

#### 页面范围解析流程

```
输入 pages 字符串
    ↓
trim()
    ↓
以 '-' 结尾?（如 "3-"）
    是 → 解析起始页 → 返回 { firstPage, lastPage: Infinity }
    ↓
包含 '-'?
    否 → 解析为单页 → 返回 { firstPage: page, lastPage: page }
    ↓
是 → 分割为起始和结束
    ↓
两者都有效正整数? 且 结束 >= 起始?
    是 → 返回 { firstPage, lastPage }
    否 → 返回 null
```

### 数据结构

```typescript
// 页面范围结果
{
  firstPage: number    // 起始页（1-indexed）
  lastPage: number     // 结束页（1-indexed，可为 Infinity）
}

// 文档扩展名集合
const DOCUMENT_EXTENSIONS = new Set(['pdf'])
```

### 解析逻辑详解

```typescript
// 开放范围 "N-"
if (trimmed.endsWith('-')) {
  const first = parseInt(trimmed.slice(0, -1), 10)
  if (isNaN(first) || first < 1) return null
  return { firstPage: first, lastPage: Infinity }
}

// 单页 "N"
const dashIndex = trimmed.indexOf('-')
if (dashIndex === -1) {
  const page = parseInt(trimmed, 10)
  if (isNaN(page) || page < 1) return null
  return { firstPage: page, lastPage: page }
}

// 范围 "M-N"
const first = parseInt(trimmed.slice(0, dashIndex), 10)
const last = parseInt(trimmed.slice(dashIndex + 1), 10)
if (isNaN(first) || isNaN(last) || first < 1 || last < 1 || last < first) {
  return null
}
return { firstPage: first, lastPage: last }
```

### 模型检测逻辑

```typescript
export function isPDFSupported(): boolean {
  // Haiku 3 是唯一不支持 PDF 的模型
  // 子串匹配覆盖所有提供商 ID 格式（Bedrock 前缀、Vertex @-dates）
  return !getMainLoopModel().toLowerCase().includes('claude-3-haiku')
}
```

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `./model/model.js` | `getMainLoopModel()` 获取当前模型 |

### 调用方

| 调用方 | 用途 |
|--------|------|
| `src/tools/FileReadTool/FileReadTool.ts` | 页面范围解析、PDF 支持检测 |
| `src/tools/FileReadTool/prompt.ts` | PDF 扩展名检测 |
| `src/utils/attachments.ts` | PDF 扩展名检测 |

### 相关常量

- `DOCUMENT_EXTENSIONS`：可扩展的文档扩展名集合（当前仅 PDF）

## 风险、边界与改进建议

### 已知风险

1. **页面范围歧义**
   - 风险：用户可能输入 "1-10-20" 等无效格式
   - 处理：返回 `null`，调用方需处理
   - 建议：提供更详细的错误信息

2. **模型检测准确性**
   - 风险：未来模型命名可能变化
   - 现状：基于子串匹配 `"claude-3-haiku"`
   - 潜在问题：新模型可能误分类
   - 建议：使用模型能力 API 而非名称匹配

3. **扩展名大小写**
   - 当前：`toLowerCase()` 标准化
   - 风险：某些文件系统大小写敏感
   - 处理：已正确处理

### 边界情况

| 场景 | 行为 |
|------|------|
| 空字符串 | 返回 `null` |
| 空白字符 | trim 后处理，全空白返回 `null` |
| "0" 或 "0-5" | 返回 `null`（页码必须 >= 1） |
| "5-3"（逆序） | 返回 `null` |
| "abc" | 返回 `null` |
| "1.5" | 返回 `null`（非整数） |
| 超大数字 | `parseInt` 处理，可能精度丢失 |
| 扩展名 ".PDF" | 正确识别（标准化后比较） |
| 扩展名 "pdf" | 正确识别 |
| 无扩展名 | 返回 `false` |

### 改进建议

1. **错误信息增强**
   - 当前：返回 `null` 表示无效
   - 建议：返回 `{ error: string }` 说明具体原因
   - 用途：向用户显示友好错误消息

2. **更多范围格式**
   - 建议支持：
     - 逗号分隔：`"1,3,5"`（非连续页面）
     - 负数索引：`"-5"`（最后 5 页）
     - 相对范围：`"+5"`（从当前页后 5 页）

3. **模型能力 API**
   - 当前：基于名称子串匹配
   - 建议：
     - 添加模型能力查询接口
     - 显式声明 PDF 支持
     - 支持动态能力检测

4. **多文档类型扩展**
   - 当前：仅 PDF
   - 建议：
     - 扩展 `DOCUMENT_EXTENSIONS`
     - 支持 EPUB、DOCX 等
     - 统一文档处理接口

5. **页面验证**
   - 当前：仅语法验证
   - 建议：
     - 结合 `getPDFPageCount()` 验证页码存在
     - 自动裁剪超出范围的部分

6. **国际化**
   - 建议：
     - 支持全角数字
     - 支持本地化分隔符（如中文破折号）

7. **性能优化**
   - 当前：简单字符串操作
   - 建议：如需频繁调用，考虑缓存解析结果

8. **单元测试**
   - 建议：
     - 边界值测试（1、Infinity、超大数）
     - 格式变体测试
     - 无效输入测试
