# Ultrareview Quota 服务研究文档

## 文件信息
- **路径**: `src/services/api/ultrareviewQuota.ts`
- **大小**: 1,219 bytes
- **最后更新**: 2026-04-01

---

## 场景与职责

Ultrareview Quota 服务负责管理 Claude Code 的 **Ultrareview 功能配额查询**，用于在代码审查前展示用户的剩余审查额度。该服务处理以下核心场景：

1. **配额查询**: 获取用户的 Ultrareview 使用情况和剩余额度
2. **资格筛选**: 仅对 Claude.ai 订阅者启用
3. **非阻塞设计**: 错误时返回 null，不阻塞审查流程

---

## 功能点目的

### 1. 配额获取 (`fetchUltrareviewQuota`)
从后端 API 获取 Ultrareview 配额信息：
- 端点: `/v1/ultrareview/quota`
- 仅对 `isClaudeAISubscriber()` 返回 true 的用户启用
- 5 秒超时
- 错误时返回 null（静默失败）

### 2. 配额展示
返回的配额信息用于：
- 显示已使用/剩余审查次数
- 判断是否处于超额使用状态
- 在审查前展示额度警告或提示

---

## 具体技术实现

### 关键数据结构

```typescript
// API 响应类型
export type UltrareviewQuotaResponse = {
  reviews_used: number        // 已使用的审查次数
  reviews_limit: number       // 审查次数上限
  reviews_remaining: number   // 剩余审查次数
  is_overage: boolean         // 是否处于超额状态
}
```

### 关键流程

```
fetchUltrareviewQuota()
├── 检查 isClaudeAISubscriber()
│   └── false → 返回 null
├── 准备 API 请求
│   ├── 获取 accessToken
│   └── 获取 orgUUID
├── 发送 GET 请求
│   ├── 端点: /v1/ultrareview/quota
│   ├── 认证: OAuth Bearer + x-organization-uuid
│   └── 超时: 5s
├── 成功 → 返回 UltrareviewQuotaResponse
└── 错误 → logForDebugging，返回 null
```

---

## 关键代码路径与文件引用

### 核心实现
- `src/services/api/ultrareviewQuota.ts` - 本文件

### 调用方
| 文件 | 调用函数 | 用途 |
|------|---------|------|
| `src/commands/review/reviewRemote.ts` | `fetchUltrareviewQuota` | 远程审查前检查配额 |

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/utils/auth.ts` | `isClaudeAISubscriber` |
| `src/utils/debug.ts` | `logForDebugging` |
| `src/utils/teleport/api.ts` | `getOAuthHeaders`, `prepareApiRequest` |
| `src/constants/oauth.ts` | `getOauthConfig` |

---

## 依赖与外部交互

### API 端点

| 端点 | 方法 | 认证 | 超时 | 用途 |
|------|------|------|------|------|
| `/v1/ultrareview/quota` | GET | OAuth Bearer + x-organization-uuid | 5s | 获取配额信息 |

### 请求头
```typescript
{
  'Authorization': 'Bearer {accessToken}',
  'Content-Type': 'application/json',
  'anthropic-version': '2023-06-01',
  'x-organization-uuid': orgUUID
}
```

---

## 风险、边界与改进建议

### 已知风险

1. **无缓存机制**
   - 每次审查前都调用 API
   - 可能导致不必要的网络请求

2. **硬编码超时**
   - 5 秒超时可能不适合所有网络环境
   - 无重试机制

3. **订阅者检查前置**
   - 在函数开头就检查 `isClaudeAISubscriber()`
   - 如果后端扩展资格标准（如 Pro 用户也可使用），需要前端更新

### 边界情况

| 场景 | 行为 |
|------|------|
| 非订阅者 | 立即返回 null |
| API 401 | 返回 null（prepareApiRequest 会抛出）|
| API 超时（5s） | 返回 null |
| 网络错误 | 返回 null |
| `prepareApiRequest` 失败 | 抛出错误（无 accessToken 或 orgUUID）|

### 改进建议

1. **添加缓存机制**
   ```typescript
   // 参考其他配额服务的缓存模式
   const CACHE_TTL_MS = 5 * 60 * 1000  // 5 分钟
   
   export async function fetchUltrareviewQuota(): Promise<UltrareviewQuotaResponse | null> {
     const cached = getUltrareviewQuotaCache()
     if (cached && Date.now() - cached.timestamp < CACHE_TTL_MS) {
       return cached.data
     }
     // ... 获取并缓存
   }
   ```

2. **可配置超时**
   ```typescript
   const timeout = getConfig('ultrareview.timeoutMs', 5000)
   ```

3. **重试机制**
   ```typescript
   // 添加指数退避重试
   for (let attempt = 0; attempt < MAX_RETRIES; attempt++) {
     try {
       return await axios.get(...)
     } catch (error) {
       if (attempt < MAX_RETRIES - 1) {
         await sleep(BASE_DELAY_MS * Math.pow(2, attempt))
       }
     }
   }
   ```

4. **Analytics 事件**
   ```typescript
   logEvent('tengu_ultrareview_quota_checked', {
     reviews_used,
     reviews_remaining,
     is_overage
   })
   ```

5. **错误分类**
   - 当前所有错误都返回 null
   - 建议区分可恢复错误（网络）和不可恢复错误（401）

6. **配额预警**
   ```typescript
   // 在配额即将耗尽时发送事件
   if (response.reviews_remaining <= 2) {
     logEvent('tengu_ultrareview_quota_low', {
       reviews_remaining: response.reviews_remaining
     })
   }
   ```

### 相关模式对比

| 服务 | 缓存 | 超时 | 重试 | 错误处理 |
|------|------|------|------|---------|
| `ultrareviewQuota.ts` | ❌ | 5s | ❌ | 返回 null |
| `usage.ts` | ❌ | 5s | ❌ | 抛出/返回空对象 |
| `overageCreditGrant.ts` | ✅ 1h | 默认 | ❌ | 返回 null |
| `referral.ts` | ✅ 24h | 5s | ❌ | 返回 null |

`ultrareviewQuota.ts` 是这些服务中最简单的实现，缺乏缓存和重试机制，适合低频调用场景。
