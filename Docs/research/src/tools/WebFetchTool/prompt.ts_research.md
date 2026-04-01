# WebFetchTool/prompt.ts 研究文档

## 场景与职责

prompt.ts 是 WebFetchTool 的**Prompt 生成模块**，负责：

1. **定义工具常量** - 工具名称和描述文本
2. **生成工具描述** - 提供给 Claude 模型的工具使用说明
3. **构建二次模型 Prompt** - 用于 Haiku 模型处理网页内容的 prompt 模板

该模块是工具与模型交互的文本层，确保模型理解工具的能力和限制。

## 功能点目的

### 1. WEB_FETCH_TOOL_NAME - 工具名称常量

**目的**：定义工具的标准名称，用于工具注册、权限规则匹配和日志记录。

```typescript
export const WEB_FETCH_TOOL_NAME = 'WebFetch'
```

**代码位置**：行 1

### 2. DESCRIPTION - 工具描述

**目的**：向 Claude 模型说明 WebFetch 工具的功能、使用方法和限制。

**描述内容要点**：
- 功能概述：获取 URL 内容并使用 AI 模型处理
- 输入参数：URL 和 prompt
- 处理流程：获取 → HTML 转 Markdown → 使用小模型处理 → 返回结果
- 使用场景：需要检索和分析网页内容时

**重要使用说明**：
- 优先使用 MCP 提供的 web fetch 工具（限制更少）
- URL 必须是完整格式
- HTTP 自动升级为 HTTPS
- 工具只读，不修改文件
- 大内容可能被摘要
- 15 分钟自清理缓存
- 跨主机重定向需用户手动重新请求
- GitHub URL 建议使用 gh CLI

**代码位置**：行 3-21

### 3. makeSecondaryModelPrompt - 二次模型 Prompt 生成

**目的**：为 Haiku 模型构建处理网页内容的 prompt，根据域名类型应用不同的内容处理规则。

**参数**：
- `markdownContent` - 网页内容（已转换为 Markdown）
- `prompt` - 用户的处理指令
- `isPreapprovedDomain` - 是否为预批准域名

**预批准域名规则**（更宽松）：
- 提供基于内容的简洁回复
- 可包含相关细节、代码示例、文档摘录

**非预批准域名规则**（更严格，出于版权考虑）：
- 严格限制引用长度（最多 125 字符）
- 引用必须使用引号标注
- 非引用部分不得与原文逐字相同
- 禁止评论提示和回复的合法性
- 禁止生成或复制精确歌词

**代码位置**：行 23-46

## 具体技术实现

### 常量定义

```typescript
// 工具名称
export const WEB_FETCH_TOOL_NAME = 'WebFetch'

// 工具描述（多行字符串模板）
export const DESCRIPTION = `
- Fetches content from a specified URL and processes it using an AI model
- Takes a URL and a prompt as input
- Fetches the URL content, converts HTML to markdown
- Processes the content with the prompt using a small, fast model
- Returns the model's response about the content
- Use this tool when you need to retrieve and analyze web content

Usage notes:
  - IMPORTANT: If an MCP-provided web fetch tool is available, prefer using that tool instead of this one, as it may have fewer restrictions.
  - The URL must be a fully-formed valid URL
  - HTTP URLs will be automatically upgraded to HTTPS
  - The prompt should describe what information you want to extract from the page
  - This tool is read-only and does not modify any files
  - Results may be summarized if the content is very large
  - Includes a self-cleaning 15-minute cache for faster responses when repeatedly accessing the same URL
  - When a URL redirects to a different host, the tool will inform you and provide the redirect URL in a special format. You should then make a new WebFetch request with the redirect URL to fetch the content.
  - For GitHub URLs, prefer using the gh CLI via Bash instead (e.g., gh pr view, gh issue view, gh api).
`
```

### Prompt 模板结构

```typescript
export function makeSecondaryModelPrompt(
  markdownContent: string,
  prompt: string,
  isPreapprovedDomain: boolean,
): string {
  // 根据域名类型选择不同的指导原则
  const guidelines = isPreapprovedDomain
    ? `Provide a concise response based on the content above. Include relevant details, code examples, and documentation excerpts as needed.`
    : `Provide a concise response based only on the content above. In your response:
 - Enforce a strict 125-character maximum for quotes from any source document. Open Source Software is ok as long as we respect the license.
 - Use quotation marks for exact language from articles; any language outside of the quotation should never be word-for-word the same.
 - You are not a lawyer and never comment on the legality of your own prompts and responses.
 - Never produce or reproduce exact song lyrics.`

  // 组合成完整 prompt
  return `
Web page content:
---
${markdownContent}
---

${prompt}

${guidelines}
`
}
```

## 关键代码路径与文件引用

### 导出内容

```typescript
// 行 1
export const WEB_FETCH_TOOL_NAME: string

// 行 3-21
export const DESCRIPTION: string

// 行 23-46
export function makeSecondaryModelPrompt(
  markdownContent: string,
  prompt: string,
  isPreapprovedDomain: boolean,
): string
```

### 被调用方

| 调用方 | 文件路径 | 用途 |
|--------|----------|------|
| WebFetchTool.ts | ./WebFetchTool.ts | 导入 WEB_FETCH_TOOL_NAME 和 DESCRIPTION |
| utils.ts | ./utils.ts | 调用 makeSecondaryModelPrompt 构建 Haiku 请求 |

## 依赖与外部交互

### 导入依赖

```typescript
// 无外部导入 - 纯文本模块
```

### 外部交互

- **无网络请求**：纯静态文本模块
- **无文件系统操作**：所有文本内联在代码中

## 风险、边界与改进建议

### 版权风险

1. **内容引用限制**
   - 非预批准域名有严格的 125 字符引用限制
   - 预批准域名（主要是开源技术文档）允许更自由的引用
   - 禁止复制歌词等受版权保护的内容

2. **免责声明**
   - 明确告知模型不要评论提示和回复的合法性
   - 这不是法律建议的替代

### 边界情况

| 场景 | 处理 |
|------|------|
| markdownContent 为空 | 仍会生成 prompt，Haiku 会基于空内容回复 |
| markdownContent 超长 | 调用方（utils.ts）会在传入前截断至 MAX_MARKDOWN_LENGTH |
| prompt 为空 | 生成的 prompt 会有空行，但结构仍有效 |

### 改进建议

1. **国际化支持**
   - 当前 DESCRIPTION 为英文，可考虑多语言支持
   - 根据用户语言环境返回对应语言描述

2. **动态描述**
   - 当前描述是静态的，可根据配置动态调整
   - 例如：缓存时间、大小限制等可从实际配置读取

3. **Prompt 优化**
   - 可考虑 A/B 测试不同的 prompt 模板效果
   - 针对不同类型的内容（文档、博客、API 参考）提供专门的处理指导

4. **版权指导细化**
   - 当前版权规则较为笼统
   - 可考虑根据内容类型（代码、文章、音乐等）提供更精确的指导

5. **预批准域名列表同步**
   - makeSecondaryModelPrompt 中的规则与 preapproved.ts 的列表逻辑关联
   - 建议确保两者逻辑一致（如预批准域名的宽松规则）

### 测试建议

- 未发现针对 prompt.ts 的专门测试
- 建议添加：
  - makeSecondaryModelPrompt 输出格式验证
  - 预批准和非预批准域名的规则差异验证
  - DESCRIPTION 内容完整性检查
