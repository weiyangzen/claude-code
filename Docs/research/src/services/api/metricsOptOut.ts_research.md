# Metrics Opt-Out 服务研究文档

## 文件信息
- **路径**: `src/services/api/metricsOptOut.ts`
- **大小**: 5,355 bytes
- **最后更新**: 2026-04-01

---

## 场景与职责

Metrics Opt-Out 服务负责管理 **组织级别的指标收集启用状态**，控制 BigQuery 导出器是否应该发送遥测数据。该服务处理以下核心场景：

1. **组织指标设置检查**: 查询用户所属组织是否启用了指标日志记录
2. **双层缓存策略**: 内存缓存（1小时）+ 磁盘缓存（24小时），减少 API 调用
3. **非阻塞冷启动**: 首次运行时后台获取，避免阻塞启动流程
4. **权限感知**: 区分完整 OAuth 会话和服务密钥会话（后者无 profile scope）

---

## 功能点目的

### 1. 指标启用状态检查 (`checkMetricsEnabled`)
主入口函数，实现双层缓存策略：
- **磁盘缓存（24h）**: 持久化在 `~/.claude/config.json`，跨进程共享
- **内存缓存（1h）**: 使用 `memoizeWithTTLAsync`，同进程内去重
- **后台刷新**: 缓存过期时返回旧值并触发后台更新

### 2. 实时状态刷新 (`refreshMetricsStatus`)
强制刷新指标状态并持久化到磁盘：
- 调用 `_checkMetricsEnabledAPI` 获取最新状态
- 仅在数据变化或缓存过期时写入磁盘（避免写入放大）
- 错误不持久化（避免瞬态失败污染缓存）

### 3. 内部 API 检查 (`_checkMetricsEnabledAPI`)
实际调用后端 API：
- 端点: `https://api.anthropic.com/api/claude_code/organizations/metrics_enabled`
- 支持 OAuth 401 重试（带 403 撤销检测）
- 受 `isEssentialTrafficOnly()` 保护

---

## 具体技术实现

### 关键数据结构

```typescript
// API 响应类型
type MetricsEnabledResponse = {
  metrics_logging_enabled: boolean
}

// 内部状态类型
type MetricsStatus = {
  enabled: boolean   // 是否启用指标
  hasError: boolean  // 是否发生错误
}

// 缓存配置
const CACHE_TTL_MS = 60 * 60 * 1000        // 1 小时（内存）
const DISK_CACHE_TTL_MS = 24 * 60 * 60 * 1000  // 24 小时（磁盘）
```

### 关键流程

#### 主检查流程
```
checkMetricsEnabled()
├── 权限预检查
│   ├── isClaudeAISubscriber() && !hasProfileScope() 
│   │   └── 返回 { enabled: false, hasError: false }
│   └── 服务密钥会话直接返回 false
├── 检查磁盘缓存
│   ├── 缓存存在且过期 → 后台刷新 + 返回缓存值
│   ├── 缓存存在且有效 → 直接返回
│   └── 无缓存 → 阻塞式刷新（首次运行）
└── 返回 MetricsStatus
```

#### 刷新流程
```
refreshMetricsStatus()
├── 调用 memoizedCheckMetrics()（内存缓存层）
├── 检查错误 → 直接返回
├── 检查数据是否变化
│   └── 未变化且时间戳新鲜 → 跳过写入
└── 调用 saveGlobalConfig() 持久化
```

### 缓存写入优化

```typescript
// 避免不必要的磁盘写入
const unchanged = cached !== undefined && cached.enabled === result.enabled
if (unchanged && Date.now() - cached.timestamp < DISK_CACHE_TTL_MS) {
  return result  // 跳过写入
}
```

---

## 关键代码路径与文件引用

### 核心实现
- `src/services/api/metricsOptOut.ts` - 本文件

### 调用方
| 文件 | 调用函数 | 用途 |
|------|---------|------|
| `src/utils/telemetry/bigqueryExporter.ts` | `checkMetricsEnabled` | 决定是否导出遥测数据 |

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/utils/auth.ts` | `isClaudeAISubscriber`, `hasProfileScope` |
| `src/utils/config.ts` | `getGlobalConfig`, `saveGlobalConfig` |
| `src/utils/http.ts` | `getAuthHeaders`, `withOAuth401Retry` |
| `src/utils/memoize.ts` | `memoizeWithTTLAsync` |
| `src/utils/privacyLevel.ts` | `isEssentialTrafficOnly` |
| `src/utils/userAgent.ts` | `getClaudeCodeUserAgent` |

---

## 依赖与外部交互

### API 端点

| 端点 | 方法 | 认证 | 超时 |
|------|------|------|------|
| `/api/claude_code/organizations/metrics_enabled` | GET | OAuth Bearer | 5s |

### 请求头
```typescript
{
  'Content-Type': 'application/json',
  'User-Agent': getClaudeCodeUserAgent(),
  'Authorization': 'Bearer {accessToken}'
}
```

### 配置存储

缓存存储在 `~/.claude/config.json`：
```json
{
  "metricsStatusCache": {
    "enabled": true,
    "timestamp": 1712345678901
  }
}
```

**注意**: 缓存是全局的（不按组织分区），因为 `checkMetricsEnabled` 在调用前已检查 `isClaudeAISubscriber()`。

---

## 风险、边界与改进建议

### 已知风险

1. **权限预检查顺序**
   ```typescript
   if (isClaudeAISubscriber() && !hasProfileScope()) {
     return { enabled: false, hasError: false }
   }
   ```
   - 服务密钥会话返回 false，但不会污染磁盘缓存（检查在磁盘读取前）
   - 这是有意设计，确保后续完整 OAuth 会话能正确获取状态

2. **全局缓存风险**
   - 多组织用户切换组织时，缓存可能返回错误组织的设置
   - 当前实现假设 `isClaudeAISubscriber()` 已筛选，但多组织场景可能需要按组织分区

3. **后台刷新错误处理**
   ```typescript
   void refreshMetricsStatus().catch(logError)
   ```
   - 使用 `void` 忽略 Promise，错误仅记录到日志
   - 连续失败可能导致缓存长期过期

### 边界情况

| 场景 | 行为 |
|------|------|
| `isEssentialTrafficOnly()` | 返回 `{ enabled: false, hasError: false }`，跳过网络 |
| 服务密钥会话（无 profile scope） | 返回 false，不调用 API，不写入缓存 |
| API 401/403 | `withOAuth401Retry` 自动刷新 token 后重试 |
| API 超时（5s） | 返回 `{ enabled: false, hasError: true }` |
| 网络错误 | 返回错误状态，不覆盖现有缓存 |
| `saveGlobalConfig` 失败 | `refreshMetricsStatus` 捕获并返回错误状态 |

### 改进建议

1. **按组织分区缓存**
   ```typescript
   // 当前
   metricsStatusCache?: { enabled: boolean; timestamp: number }
   
   // 建议
   metricsStatusCache?: Record<string, { enabled: boolean; timestamp: number }>
   // key 为 organizationUuid
   ```

2. **缓存失效事件**
   - 添加组织切换时的缓存失效逻辑
   - 监听 `orgUUID` 变化事件

3. **错误重试策略**
   - 当前仅依赖 `withOAuth401Retry` 的一次重试
   - 考虑添加指数退避重试

4. **测试覆盖**
   - 添加服务密钥会话的单元测试
   - 测试缓存过期和后台刷新逻辑
   - 测试 `saveGlobalConfig` 失败时的降级行为

5. **监控增强**
   ```typescript
   // 建议添加 analytics 事件
   logEvent('tengu_metrics_opt_out_check', {
     enabled,
     hasError,
     cacheHit: !!cached,
     cacheAge: cached ? Date.now() - cached.timestamp : undefined
   })
   ```

### 相关模式对比

| 服务 | 缓存策略 | TTL |
|------|---------|-----|
| `metricsOptOut.ts` | 内存 + 磁盘 | 1h / 24h |
| `grove.ts` | 内存 + 磁盘 | 会话 / 24h |
| `overageCreditGrant.ts` | 仅磁盘 | 1h |
| `referral.ts` | 仅磁盘 | 24h |

`metricsOptOut` 的双层缓存模式是这些服务中最复杂的，适合高频调用场景（如每次 BigQuery 导出前检查）。
