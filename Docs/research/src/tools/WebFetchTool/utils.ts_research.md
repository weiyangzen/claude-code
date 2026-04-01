# WebFetchTool/utils.ts 研究文档

## 场景与职责

utils.ts 是 WebFetchTool 的**核心工具函数模块**，实现了网页内容获取、处理和缓存的完整流程。它是 WebFetchTool 最复杂的子模块，职责包括：

1. **URL 验证** - 验证 URL 格式、长度和安全性
2. **域名安全检查** - 通过 Anthropic API 检查域名是否在黑名单中
3. **HTTP 内容获取** - 使用 axios 获取网页内容，处理重定向和二进制内容
4. **内容转换** - 使用 Turndown 将 HTML 转换为 Markdown
5. **内容缓存** - 实现 LRU 缓存机制，避免重复请求
6. **AI 内容处理** - 使用 Haiku 模型根据用户 prompt 处理内容
7. **二进制内容持久化** - 将 PDF 等二进制内容保存到磁盘

## 功能点目的

### 1. URL 验证 (validateURL)

**目的**：确保 URL 符合安全和使用要求。

**验证规则**：
- 长度限制：MAX_URL_LENGTH = 2000 字符（原为 250，后因 JWT 签名 URL 需求放宽）
- 格式验证：必须是有效的 URL 格式（通过 `new URL()` 解析）
- 禁止认证信息：URL 中不能包含用户名/密码
- 域名有效性：hostname 必须至少包含两个部分（防止内部域名）

**代码位置**：行 106, 139-169

### 2. 域名黑名单检查 (checkDomainBlocklist)

**目的**：通过 Anthropic API 检查域名是否安全。

**实现细节**：
- 请求地址：`https://api.anthropic.com/api/web/domain_info?domain={domain}`
- 超时：DOMAIN_CHECK_TIMEOUT_MS = 10 秒
- 缓存：DOMAIN_CHECK_CACHE（5 分钟 TTL，最大 128 条目）
- 仅缓存 'allowed' 结果，blocked/failed 下次重新检查

**返回结果**：
- `allowed` - 域名允许访问
- `blocked` - 域名被阻止
- `check_failed` - 检查失败（网络问题或企业安全策略）

**代码位置**：行 75-78, 171-203

### 3. 安全重定向处理 (isPermittedRedirect, getWithPermittedRedirects)

**目的**：安全地处理 HTTP 重定向，防止开放重定向攻击。

**允许的重定向规则** (isPermittedRedirect)：
- 协议必须相同
- 端口必须相同
- 目标 URL 不能包含认证信息
- 仅允许以下 hostname 变化：
  - 添加 www.（example.com → www.example.com）
  - 移除 www.（www.example.com → example.com）
  - 相同 host（路径/查询参数可变更）

**递归获取实现** (getWithPermittedRedirects)：
- 最大重定向次数：MAX_REDIRECTS = 10
- 请求超时：FETCH_TIMEOUT_MS = 60 秒
- 最大内容长度：MAX_HTTP_CONTENT_LENGTH = 10MB
- 不自动跟随重定向（maxRedirects: 0），手动检查每个重定向
- 检测到出口代理阻止时抛出 EgressBlockedError

**代码位置**：行 121-126, 205-329

### 4. 内容获取与转换 (getURLMarkdownContent)

**目的**：获取 URL 内容并转换为 Markdown 格式。

**执行流程**：
1. URL 验证
2. 检查缓存（URL_CACHE，15 分钟 TTL，50MB 大小限制）
3. HTTP 升级为 HTTPS
4. 域名黑名单检查（可配置跳过）
5. 发送 HTTP 请求获取内容
6. 处理重定向（跨主机重定向返回 RedirectInfo）
7. 二进制内容保存到磁盘
8. HTML 转换为 Markdown（使用 Turndown）
9. 缓存结果并返回

**Turndown 服务**：
- 懒加载：首次 HTML 获取时才导入（~1.4MB 堆内存）
- 单例模式：复用实例（构造时创建 15 个规则对象）

**代码位置**：行 85-97, 331-482

### 5. 内容 AI 处理 (applyPromptToMarkdown)

**目的**：使用 Haiku 模型根据用户 prompt 处理网页内容。

**处理流程**：
1. 截断内容至 MAX_MARKDOWN_LENGTH（100K 字符）
2. 构建模型 prompt（调用 makeSecondaryModelPrompt）
3. 调用 queryHaiku 发送请求
4. 检查是否被取消（AbortSignal）
5. 提取并返回模型回复文本

**代码位置**：行 484-530

### 6. 缓存管理

**URL 缓存 (URL_CACHE)**：
- 类型：LRUCache<string, CacheEntry>
- TTL：15 分钟
- 大小限制：50MB
- 缓存键：原始 URL（非升级或重定向后的 URL）

**域名检查缓存 (DOMAIN_CHECK_CACHE)**：
- 类型：LRUCache<string, true>
- TTL：5 分钟
- 最大条目：128
- 仅缓存 'allowed' 结果

**缓存清理函数** (clearWebFetchCache)：
- 被 caches.ts 在会话清理时调用

**代码位置**：行 51-83

## 具体技术实现

### 错误类定义

```typescript
// 域名被阻止
class DomainBlockedError extends Error {
  constructor(domain: string) {
    super(`Claude Code is unable to fetch from ${domain}`)
    this.name = 'DomainBlockedError'
  }
}

// 域名检查失败
class DomainCheckFailedError extends Error {
  constructor(domain: string) {
    super(`Unable to verify if domain ${domain} is safe to fetch...`)
    this.name = 'DomainCheckFailedError'
  }
}

// 出口代理阻止
class EgressBlockedError extends Error {
  constructor(public readonly domain: string) {
    super(JSON.stringify({
      error_type: 'EGRESS_BLOCKED',
      domain,
      message: `Access to ${domain} is blocked by the network egress proxy.`,
    }))
    this.name = 'EgressBlockedError'
  }
}
```

### 缓存条目结构

```typescript
type CacheEntry = {
  bytes: number           // 原始内容字节数
  code: number            // HTTP 状态码
  codeText: string        // HTTP 状态文本
  content: string         // Markdown 内容
  contentType: string     // Content-Type 头
  persistedPath?: string  // 二进制文件保存路径
  persistedSize?: number  // 二进制文件大小
}
```

### 关键常量

```typescript
const CACHE_TTL_MS = 15 * 60 * 1000           // 15 分钟
const MAX_CACHE_SIZE_BYTES = 50 * 1024 * 1024 // 50MB
const MAX_URL_LENGTH = 2000                   // URL 最大长度
const MAX_HTTP_CONTENT_LENGTH = 10 * 1024 * 1024 // 10MB
const FETCH_TIMEOUT_MS = 60_000               // 60 秒
const DOMAIN_CHECK_TIMEOUT_MS = 10_000        // 10 秒
const MAX_REDIRECTS = 10                      // 最大重定向次数
export const MAX_MARKDOWN_LENGTH = 100_000    // Markdown 最大长度
```

### 二进制内容处理

```typescript
// 检测是否为二进制内容
if (isBinaryContentType(contentType)) {
  const persistId = `webfetch-${Date.now()}-${Math.random().toString(36).slice(2, 8)}`
  const result = await persistBinaryContent(rawBuffer, contentType, persistId)
  if (!('error' in result)) {
    persistedPath = result.filepath
    persistedSize = result.size
  }
}

// 在结果中附加文件信息
if (persistedPath) {
  result += `\n\n[Binary content (${contentType}, ${formatFileSize(persistedSize ?? bytes)}) also saved to ${persistedPath}]`
}
```

## 关键代码路径与文件引用

### 导出函数

| 函数 | 行号 | 描述 |
|------|------|------|
| `clearWebFetchCache` | 80-83 | 清理所有缓存 |
| `isPreapprovedUrl` | 130-137 | 检查 URL 是否为预批准域名 |
| `validateURL` | 139-169 | 验证 URL 有效性 |
| `checkDomainBlocklist` | 176-203 | 检查域名黑名单 |
| `isPermittedRedirect` | 212-243 | 检查重定向是否允许 |
| `getWithPermittedRedirects` | 262-329 | 递归获取内容（处理重定向）|
| `getURLMarkdownContent` | 347-482 | 获取 URL 内容并转换为 Markdown |
| `applyPromptToMarkdown` | 484-530 | 使用 Haiku 处理内容 |

### 导出类型

| 类型 | 行号 | 描述 |
|------|------|------|
| `FetchedContent` | 337-345 | 获取到的内容结构 |
| `CacheEntry` | 51-59 | 缓存条目结构（内部）|

### 导入依赖

```typescript
// HTTP 和内容处理
import axios, { type AxiosResponse } from 'axios'
import { LRUCache } from 'lru-cache'

// AI 服务
import { queryHaiku } from '../../services/api/claude.js'

// 工具内部
import { isPreapprovedHost } from './preapproved.js'
import { makeSecondaryModelPrompt } from './prompt.js'

// 工具通用
import { getWebFetchUserAgent } from '../../utils/http.js'
import { isBinaryContentType, persistBinaryContent } from '../../utils/mcpOutputStorage.js'
import { getSettings_DEPRECATED } from '../../utils/settings/settings.js'
import { asSystemPrompt } from '../../utils/systemPromptType.js'
import { AbortError } from '../../utils/errors.js'
import { logError } from '../../utils/log.js'

// 分析
import { type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS, logEvent } from '../../services/analytics/index.js'
```

## 依赖与外部交互

### 外部服务调用

| 服务 | 用途 | 位置 |
|------|------|------|
| `api.anthropic.com/api/web/domain_info` | 域名黑名单检查 | checkDomainBlocklist |
| `queryHaiku` | AI 内容处理 | applyPromptToMarkdown |
| `任意 HTTP 站点` | 内容获取 | getWithPermittedRedirects |

### 被调用方

| 调用方 | 文件路径 | 用途 |
|--------|----------|------|
| WebFetchTool.ts | ./WebFetchTool.ts | 调用 getURLMarkdownContent 和 applyPromptToMarkdown |
| caches.ts | src/commands/clear/caches.ts | 调用 clearWebFetchCache 清理缓存 |

## 风险、边界与改进建议

### 安全风险

1. **数据外泄防护**
   - URL 长度限制（2000 字符）防止通过 URL 参数外泄数据
   - 域名黑名单检查阻止访问恶意域名
   - 用户权限确认提供主要安全边界

2. **重定向攻击防护**
   - 不自动跟随跨主机重定向
   - 严格检查重定向目标（协议、端口、认证信息）
   - 限制重定向次数（10 次）

3. **资源消耗控制**
   - 最大内容长度 10MB
   - 请求超时 60 秒
   - 缓存大小限制 50MB
   - Markdown 截断 100K 字符

4. **出口代理检测**
   - 检测 X-Proxy-Error: blocked-by-allowlist 头
   - 提供清晰的错误信息

### 边界情况

| 场景 | 处理 |
|------|------|
| URL 超过 2000 字符 | validateURL 返回 false |
| URL 包含用户名/密码 | validateURL 返回 false |
| 域名检查超时 | 抛出 DomainCheckFailedError |
| 重定向次数超过 10 次 | 抛出 "Too many redirects" 错误 |
| 内容超过 10MB | axios 自动截断或报错 |
| 二进制内容 | 保存到磁盘，同时返回文本提取内容 |
| 请求被取消 | applyPromptToMarkdown 抛出 AbortError |
| 缓存过期 | LRUCache 自动清理 |

### 性能考虑

1. **Turndown 懒加载**
   - 首次 HTML 获取时才导入（~1.4MB）
   - 单例复用避免重复构造

2. **内存管理**
   - 获取 ArrayBuffer 后立即转换为 Buffer
   - 释放 axios 持有的 ArrayBuffer（`response.data = null`）
   - 避免 Turndown 构建 DOM 树时内存翻倍

3. **缓存策略**
   - 域名检查缓存避免重复 API 调用
   - URL 缓存避免重复 HTTP 请求
   - 缓存大小和 TTL 限制防止内存泄漏

### 改进建议

1. **缓存优化**
   - 可考虑根据 HTTP Cache-Control 头调整缓存策略
   - 添加缓存预热或预取机制

2. **重定向处理**
   - 可考虑支持 308 Permanent Redirect 的缓存
   - 添加重定向链记录用于调试

3. **错误处理**
   - 为常见网络错误（DNS 失败、SSL 错误）提供更友好的消息
   - 添加重试机制处理临时故障

4. **内容处理**
   - 支持更多内容类型的转换（如 DOCX、PDF 文本提取）
   - 考虑使用流式处理大页面

5. **可观测性**
   - 添加更多指标：缓存命中率、平均响应时间、域名分布
   - 记录重定向链信息用于分析

6. **测试覆盖**
   - 未发现针对 utils.ts 的专门测试
   - 建议添加：URL 验证、重定向处理、缓存行为、错误场景的测试

### 已知限制

1. **JavaScript 渲染页面**：无法获取 SPA 动态加载的内容
2. **Cookie/Session**：不支持维护会话状态
3. **POST/表单**：仅支持 GET 请求
4. **Turndown 转换**：HTML 到 Markdown 的转换可能不完美
