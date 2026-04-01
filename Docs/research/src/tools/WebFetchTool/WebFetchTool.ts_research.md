# WebFetchTool/WebFetchTool.ts 研究文档

## 场景与职责

WebFetchTool 是 Claude Code CLI 中用于**从指定 URL 获取网页内容并进行 AI 处理**的核心工具。它实现了完整的工具生命周期，包括：

1. **权限控制** - 基于域名的细粒度权限管理（预批准域名、用户规则）
2. **内容获取** - 通过 HTTP 请求获取网页内容，支持缓存和重定向处理
3. **内容处理** - 将 HTML 转换为 Markdown，并使用 Haiku 模型根据用户 prompt 处理内容
4. **结果格式化** - 返回结构化的结果数据（大小、状态码、处理后的内容）

该工具是只读工具，不会修改任何数据，主要用于帮助 Claude 获取外部文档、参考资料等信息。

## 功能点目的

### 1. 工具定义与配置

**目的**：定义工具的元数据、输入/输出 schema 和基本行为。

**关键配置**：
- `maxResultSizeChars: 100_000` - 工具结果持久化阈值
- `shouldDefer: true` - 支持延迟加载（需通过 ToolSearch 查找）
- `isConcurrencySafe: true` - 支持并发执行
- `isReadOnly: true` - 只读工具

**代码位置**：行 66-307

### 2. 权限检查 (checkPermissions)

**目的**：确定是否允许访问特定 URL，实现多层权限控制。

**权限检查层级**（按优先级）：
1. **预批准域名检查** - 检查域名是否在 `PREAPPROVED_HOSTS` 列表中（如 docs.python.org, react.dev 等）
2. **deny 规则检查** - 检查用户是否明确拒绝了该域名
3. **ask 规则检查** - 检查用户是否设置了"始终询问"规则
4. **allow 规则检查** - 检查用户是否明确允许了该域名
5. **默认询问** - 如果没有匹配规则，向用户请求权限

**代码位置**：行 104-180

### 3. 输入验证 (validateInput)

**目的**：验证用户提供的 URL 是否有效。

**验证内容**：
- URL 格式是否有效（通过 `new URL(url)` 解析）
- 返回详细的错误信息（包括错误码和原因）

**代码位置**：行 191-204

### 4. 工具调用 (call)

**目的**：执行实际的内容获取和处理流程。

**执行流程**：
1. 调用 `getURLMarkdownContent` 获取 URL 内容
2. 处理跨主机重定向（返回重定向信息给用户）
3. 判断是否为预批准域名 + Markdown 内容 + 内容长度小于阈值
4. 对于预批准域名的纯 Markdown 内容直接返回，否则使用 Haiku 模型处理
5. 处理二进制内容（PDF 等）的持久化存储
6. 返回结构化的输出结果

**代码位置**：行 208-299

### 5. Prompt 生成

**目的**：为模型提供工具描述和使用说明。

**特殊处理**：
- 始终包含认证警告（说明 WebFetch 对认证/私有 URL 会失败）
- 提示用户优先使用 MCP 提供的 web fetch 工具
- 警告检查 URL 是否指向认证服务（Google Docs, Confluence, Jira, GitHub 等）

**代码位置**：行 181-190

## 具体技术实现

### 输入/输出 Schema

```typescript
// 输入 Schema（使用 Zod 验证）
{
  url: z.string().url(),     // 要获取的 URL
  prompt: z.string(),         // 用于处理内容的 prompt
}

// 输出 Schema
{
  bytes: z.number(),          // 内容大小（字节）
  code: z.number(),           // HTTP 状态码
  codeText: z.string(),       // HTTP 状态文本
  result: z.string(),         // 处理后的结果
  durationMs: z.number(),     // 执行耗时
  url: z.string(),            // 实际获取的 URL
}
```

### 权限规则格式

```typescript
// 权限规则内容格式：domain:hostname
// 示例：domain:docs.python.org
function webFetchToolInputToPermissionRuleContent(input) {
  const { url } = parsedInput.data;
  const hostname = new URL(url).hostname;
  return `domain:${hostname}`;
}
```

### 关键流程详解

#### 重定向处理流程

```
getURLMarkdownContent 返回 RedirectInfo
    ↓
判断 response.type === 'redirect'
    ↓
构造重定向提示消息，包含：
- 原始 URL
- 重定向目标 URL
- HTTP 状态码和状态文本
- 建议用户使用重定向 URL 重新请求
```

#### 内容处理流程

```
获取到内容后
    ↓
判断：isPreapproved && contentType.includes('text/markdown') && content.length < MAX_MARKDOWN_LENGTH
    ↓
是 → 直接返回原始内容
否 → 调用 applyPromptToMarkdown 使用 Haiku 模型处理
    ↓
如果存在 persistedPath（二进制内容已保存到磁盘）
    ↓
在结果后附加文件保存信息
```

## 关键代码路径与文件引用

### 核心函数

| 函数 | 行号 | 描述 |
|------|------|------|
| `webFetchToolInputToPermissionRuleContent` | 50-64 | 将输入转换为权限规则格式 |
| `buildSuggestions` | 309-318 | 构建权限更新建议（用于"始终允许"选项） |

### 工具定义对象

```typescript
export const WebFetchTool = buildTool({
  name: WEB_FETCH_TOOL_NAME,           // 行 67
  searchHint: 'fetch and extract...',  // 行 68
  maxResultSizeChars: 100_000,         // 行 70
  shouldDefer: true,                   // 行 71
  description,                         // 行 72-80
  userFacingName,                      // 行 81-83
  getToolUseSummary,                   // 行 84
  getActivityDescription,              // 行 85-88
  inputSchema,                         // 行 89-91
  outputSchema,                        // 行 92-94
  isConcurrencySafe,                   // 行 95-97
  isReadOnly,                          // 行 98-100
  toAutoClassifierInput,               // 行 101-103
  checkPermissions,                    // 行 104-180
  prompt,                              // 行 181-190
  validateInput,                       // 行 191-204
  renderToolUseMessage,                // 行 205
  renderToolUseProgressMessage,        // 行 206
  renderToolResultMessage,             // 行 207
  call,                                // 行 208-299
  mapToolResultToToolResultBlockParam, // 行 300-306
});
```

### 导入依赖

```typescript
// 核心框架
import { z } from 'zod/v4'
import { buildTool, type ToolDef } from '../../Tool.js'

// 权限相关
import type { PermissionUpdate } from '../../types/permissions.js'
import type { PermissionDecision } from '../../utils/permissions/PermissionResult.js'
import { getRuleByContentsForTool } from '../../utils/permissions/permissions.js'

// 工具内部模块
import { isPreapprovedHost } from './preapproved.js'
import { DESCRIPTION, WEB_FETCH_TOOL_NAME } from './prompt.js'
import {
  getToolUseSummary,
  renderToolResultMessage,
  renderToolUseMessage,
  renderToolUseProgressMessage,
} from './UI.js'
import {
  applyPromptToMarkdown,
  type FetchedContent,
  getURLMarkdownContent,
  isPreapprovedUrl,
  MAX_MARKDOWN_LENGTH,
} from './utils.js'

// 工具通用
import { formatFileSize } from '../../utils/format.js'
import { lazySchema } from '../../utils/lazySchema.js'
```

## 依赖与外部交互

### 被调用方

| 调用方 | 文件路径 | 用途 |
|--------|----------|------|
| tools.ts | src/tools.ts | 注册到全局工具列表 |
| WebFetchPermissionRequest | src/components/permissions/WebFetchPermissionRequest/WebFetchPermissionRequest.tsx | 权限请求 UI |
| claudeCodeGuideAgent | src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts | Agent 工具引用 |
| verificationAgent | src/tools/AgentTool/built-in/verificationAgent.ts | Agent 工具引用 |
| contextSuggestions | src/utils/contextSuggestions.ts | 上下文建议 |
| microCompact | src/services/compact/microCompact.ts | 紧凑模式处理 |
| apiMicrocompact | src/services/compact/apiMicrocompact.ts | API 紧凑处理 |
| REPL | src/screens/REPL.tsx | REPL 模式 |
| PermissionRequest | src/components/permissions/PermissionRequest.tsx | 权限请求处理 |
| ToolSelector | src/components/agents/ToolSelector.tsx | 工具选择器 |
| caches | src/commands/clear/caches.ts | 缓存清理 |
| PermissionRuleInput | src/components/permissions/rules/PermissionRuleInput.tsx | 权限规则输入 |
| sandbox-adapter | src/utils/sandbox/sandbox-adapter.ts | 沙箱适配 |

### 外部服务交互

| 服务 | 用途 | 位置 |
|------|------|------|
| `getURLMarkdownContent` | HTTP 获取网页内容 | utils.ts |
| `applyPromptToMarkdown` | 使用 Haiku 模型处理内容 | utils.ts |

## 风险、边界与改进建议

### 安全风险

1. **数据外泄风险**
   - 已缓解：URL 长度限制（MAX_URL_LENGTH = 2000），防止 JWT 签名 URL 被滥用
   - 已缓解：需要用户权限确认（非预批准域名）
   - 已缓解：域名黑名单检查（通过 api.anthropic.com 预检）

2. **重定向攻击**
   - 已缓解：不自动跟随跨主机重定向，提示用户手动确认
   - 已缓解：限制重定向次数（MAX_REDIRECTS = 10）

3. **私有内容访问**
   - 已缓解：Prompt 中明确警告认证/私有 URL 会失败
   - 已缓解：检查 URL 中的用户名/密码（parsed.username/password）

### 边界情况

| 场景 | 处理 |
|------|------|
| URL 解析失败 | validateInput 返回错误结果 |
| 域名被黑名单 | checkPermissions 返回 deny 决策 |
| 域名检查失败 | 抛出 DomainCheckFailedError |
| 跨主机重定向 | 返回 RedirectInfo，提示用户重新请求 |
| 二进制内容 | 保存到磁盘，返回文件路径 |
| 内容过长 | 截断至 MAX_MARKDOWN_LENGTH（100K 字符） |
| 请求超时 | FETCH_TIMEOUT_MS（60秒）限制 |

### 改进建议

1. **缓存策略优化**
   - 当前 URL_CACHE 使用 15 分钟 TTL，可考虑根据内容类型设置不同 TTL
   - 预批准域名的文档类内容可延长缓存时间

2. **错误处理增强**
   - 可为常见错误（DNS 失败、连接超时、SSL 错误）提供更友好的错误消息
   - 添加重试机制处理临时网络故障

3. **性能优化**
   - 对于大页面，可考虑流式处理而非全量加载后再处理
   - Turndown 服务可进一步优化（当前使用懒加载）

4. **可观测性**
   - 可添加更多指标：缓存命中率、平均响应大小、域名分布等
   - 当前仅对 USER_TYPE='ant' 记录 tengu_web_fetch_host 事件

5. **测试覆盖**
   - 未发现针对 WebFetchTool 的单元测试
   - 建议添加：权限检查逻辑、重定向处理、内容处理流程的测试

### 已知限制

1. **不支持认证内容**：无法访问需要登录的页面（Google Docs、私有 GitHub 仓库等）
2. **JavaScript 渲染页面**：无法获取 SPA（单页应用）动态加载的内容
3. **Cookie/Session**：不支持维护会话状态
4. **POST/表单提交**：仅支持 GET 请求
