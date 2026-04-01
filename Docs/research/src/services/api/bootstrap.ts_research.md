# 研究文档: src/services/api/bootstrap.ts

## 场景与职责

`bootstrap.ts` 是 Claude Code CLI 的**启动引导数据获取模块**，负责在应用启动时从 Anthropic API 获取配置数据和功能开关，以支持动态功能发布和 A/B 测试。

### 核心使用场景

1. **启动时获取客户端配置**: 获取服务器端实验配置、功能开关等
2. **获取额外模型选项**: 动态获取可用的模型列表（如实验性模型、新发布的模型）
3. **支持 GrowthBook 实验**: 获取服务器端实验配置用于功能发布

### 业务背景

- 现代 SaaS 应用需要在不发布新版本的情况下动态调整功能
- 通过启动时的 API 调用，可以获取最新的模型选项和功能配置
- 支持渐进式发布（gradual rollout）和 A/B 测试

---

## 功能点目的

### 1. 获取引导数据 (`fetchBootstrapAPI`)

**目的**: 内部函数，负责实际的 API 调用和数据获取。

**关键行为**:
- 检查隐私设置（`isEssentialTrafficOnly()`），在限制模式下跳过请求
- 检查 API 提供商（仅支持 First Party，不支持 Bedrock/Vertex/Foundry）
- 支持 OAuth 和 API Key 两种认证方式
- 使用 `withOAuth401Retry` 处理 OAuth token 过期和刷新
- 5 秒超时，避免阻塞启动

### 2. 获取并持久化引导数据 (`fetchBootstrapData`)

**目的**: 公共函数，获取引导数据并缓存到本地配置文件。

**关键行为**:
- 调用 `fetchBootstrapAPI` 获取数据
- 使用 lodash `isEqual` 比较新旧数据，避免不必要的磁盘写入
- 将数据保存到全局配置（`clientDataCache` 和 `additionalModelOptionsCache`）
- 错误处理：捕获并记录错误，不影响应用启动

---

## 具体技术实现

### 数据 Schema

```typescript
// 使用 Zod 进行运行时验证
const bootstrapResponseSchema = lazySchema(() =>
  z.object({
    client_data: z.record(z.unknown()).nullish(),  // 服务器端实验配置
    additional_model_options: z
      .array(
        z
          .object({
            model: z.string(),
            name: z.string(),
            description: z.string(),
          })
          .transform(({ model, name, description }) => ({
            value: model,
            label: name,
            description,
          })),
      )
      .nullish(),
  }),
)

type BootstrapResponse = z.infer<ReturnType<typeof bootstrapResponseSchema>>
```

### API 端点

```
GET ${BASE_API_URL}/api/claude_cli/bootstrap
```

**请求头**:
```typescript
{
  'Content-Type': 'application/json',
  'User-Agent': getClaudeCodeUserAgent(),  // claude-code/{version}
  // OAuth 认证:
  'Authorization': `Bearer ${accessToken}`,
  'anthropic-beta': OAUTH_BETA_HEADER,  // 'oauth-2025-04-20'
  // 或 API Key 认证:
  'x-api-key': apiKey,
}
```

**超时**: 5000ms (5秒)

### 认证优先级

```
1. OAuth (优先)
   - 需要 user:profile scope
   - 支持自动刷新（通过 withOAuth401Retry）
   
2. API Key (回退)
   - 用于 Console 用户（无 OAuth）
   - 不支持刷新，401 直接失败
```

### 缓存策略

```typescript
// 数据比较逻辑
const config = getGlobalConfig()
if (
  isEqual(config.clientDataCache, clientData) &&
  isEqual(config.additionalModelOptionsCache, additionalModelOptions)
) {
  logForDebugging('[Bootstrap] Cache unchanged, skipping write')
  return
}

// 仅在有变化时写入
saveGlobalConfig(current => ({
  ...current,
  clientDataCache: clientData,
  additionalModelOptionsCache: additionalModelOptions,
}))
```

### 隐私和流量控制

```typescript
// 检查是否仅允许必要流量
if (isEssentialTrafficOnly()) {
  logForDebugging('[Bootstrap] Skipped: Nonessential traffic disabled')
  return null
}

// 检查 API 提供商
if (getAPIProvider() !== 'firstParty') {
  logForDebugging('[Bootstrap] Skipped: 3P provider')
  return null
}
```

---

## 关键代码路径与文件引用

### 当前文件
- `src/services/api/bootstrap.ts` - 本模块，提供引导数据获取功能

### 依赖文件

| 文件路径 | 用途 |
|----------|------|
| `src/utils/auth.ts` | 提供 `getAnthropicApiKey()`, `getClaudeAIOAuthTokens()`, `hasProfileScope()` |
| `src/constants/oauth.ts` | 提供 `getOauthConfig()`, `OAUTH_BETA_HEADER` |
| `src/utils/config.ts` | 提供 `getGlobalConfig()`, `saveGlobalConfig()` |
| `src/utils/debug.ts` | 提供 `logForDebugging()` |
| `src/utils/http.ts` | 提供 `withOAuth401Retry()` |
| `src/utils/lazySchema.ts` | 提供 `lazySchema()` 延迟初始化 Zod schema |
| `src/utils/log.ts` | 提供 `logError()` |
| `src/utils/model/providers.ts` | 提供 `getAPIProvider()` |
| `src/utils/privacyLevel.ts` | 提供 `isEssentialTrafficOnly()` |
| `src/utils/userAgent.ts` | 提供 `getClaudeCodeUserAgent()` |

### 调用方文件

| 文件路径 | 调用函数 | 用途 |
|----------|----------|------|
| `src/main.tsx` | `fetchBootstrapData()` | 应用启动时调用 |

### 调用方代码示例

```typescript
// src/main.tsx (简化)
import { fetchBootstrapData } from './services/api/bootstrap.js'

// 在应用初始化时调用
await fetchBootstrapData()
```

---

## 依赖与外部交互

### 运行时依赖

```typescript
import axios from 'axios'
import isEqual from 'lodash-es/isEqual.js'
import { z } from 'zod'
import {
  getAnthropicApiKey,
  getClaudeAIOAuthTokens,
  hasProfileScope,
} from 'src/utils/auth.js'
import { getOauthConfig, OAUTH_BETA_HEADER } from '../../constants/oauth.js'
import { getGlobalConfig, saveGlobalConfig } from '../../utils/config.js'
import { logForDebugging } from '../../utils/debug.js'
import { withOAuth401Retry } from '../../utils/http.js'
import { lazySchema } from '../../utils/lazySchema.js'
import { logError } from '../../utils/log.js'
import { getAPIProvider } from '../../utils/model/providers.js'
import { isEssentialTrafficOnly } from '../../utils/privacyLevel.js'
import { getClaudeCodeUserAgent } from '../../utils/userAgent.js'
```

### 外部 API 依赖

- **Anthropic API**: `https://api.anthropic.com/api/claude_cli/bootstrap`
- **认证方式**: OAuth Bearer token 或 API Key
- **Beta Header**: `oauth-2025-04-20`（用于 OAuth 认证）

### 配置存储

获取的数据存储在 `GlobalConfig` 中:

```typescript
// src/utils/config.ts
export type GlobalConfig = {
  // ... 其他配置
  
  // Client data for server-side experiments (fetched during bootstrap)
  clientDataCache?: Record<string, unknown> | null
  
  // Additional model options for the model picker (fetched during bootstrap)
  additionalModelOptionsCache?: ModelOption[]
  
  // ...
}
```

### 数据使用方

获取的引导数据被以下模块使用:

1. **`clientDataCache`**: 
   - GrowthBook 功能开关
   - 服务器端实验配置
   - 由 `src/services/analytics/growthbook.ts` 消费

2. **`additionalModelOptionsCache`**:
   - 模型选择器中的额外模型选项
   - 由 `src/utils/model/modelOptions.ts` 消费

---

## 风险、边界与改进建议

### 潜在风险

1. **启动阻塞风险**
   - 虽然设置了 5 秒超时，但网络问题仍可能影响启动速度
   - 建议: 考虑改为非阻塞的后台获取

2. **认证复杂性**
   - 需要处理 OAuth 和 API Key 两种认证方式
   - OAuth 需要 profile scope，否则 403
   - 建议: 更清晰的认证失败提示

3. **数据一致性**
   - 缓存数据可能在会话期间过期
   - 建议: 添加缓存过期时间，定期刷新

4. **隐私合规**
   - 在 `essential-traffic` 模式下完全跳过
   - 需要确保这符合所有地区的隐私法规

### 边界条件

1. **网络不可用**
   - 捕获错误并静默处理，不影响启动
   - 使用现有缓存数据

2. **认证失败**
   - OAuth 401: 尝试刷新 token 并重试
   - API Key 401: 直接失败，返回 null

3. **第三方提供商**
   - Bedrock/Vertex/Foundry 用户跳过此 API
   - 这些提供商有自己的配置机制

4. **数据验证失败**
   - Zod schema 验证失败时记录错误并返回 null
   - 防止无效数据破坏应用状态

### 改进建议

1. **添加缓存过期机制**
   ```typescript
   // 建议添加时间戳和 TTL
   saveGlobalConfig(current => ({
     ...current,
     clientDataCache: clientData,
     clientDataCacheTimestamp: Date.now(),
   }))
   
   // 使用时检查过期
   const CACHE_TTL = 24 * 60 * 60 * 1000 // 24小时
   if (Date.now() - timestamp > CACHE_TTL) {
     // 触发后台刷新
   }
   ```

2. **后台刷新机制**
   - 启动时使用缓存数据立即渲染
   - 后台异步获取最新数据
   - 数据变化时通过事件通知 UI 更新

3. **更细粒度的错误分类**
   ```typescript
   // 区分不同类型的错误
   type BootstrapError =
     | { type: 'network'; retryable: true }
     | { type: 'auth'; retryable: false }
     | { type: 'validation'; retryable: false }
   ```

4. **添加请求取消支持**
   ```typescript
   const controller = new AbortController()
   const response = await axios.get(url, {
     signal: controller.signal,
     timeout: 5000,
   })
   ```

5. **增强可观测性**
   - 添加更多调试日志
   - 记录缓存命中率
   - 监控 API 响应时间

### 测试建议

- **单元测试**:
  - 模拟各种 API 响应（成功、失败、超时）
  - 测试缓存比较逻辑
  - 测试认证方式切换

- **集成测试**:
  - 使用 mock server 测试完整流程
  - 验证缓存持久化

- **E2E 测试**:
  - 测试不同网络条件下的启动行为
  - 验证隐私模式下的跳过逻辑

### 相关配置项

```typescript
// 环境变量影响
process.env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC  // 跳过 bootstrap
process.env.CLAUDE_CODE_USE_BEDROCK                   // 跳过 bootstrap
process.env.CLAUDE_CODE_USE_VERTEX                    // 跳过 bootstrap
process.env.CLAUDE_CODE_USE_FOUNDRY                   // 跳过 bootstrap
```
