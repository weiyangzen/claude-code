# fetchTelemetry.ts 深度研究文档

## 场景与职责

`fetchTelemetry.ts` 为 Claude Code 插件/市场获取操作提供网络层遥测能力。该模块源于 **inc-5046**（GitHub 抱怨 claude-plugins-official 负载过高），旨在解决此前仅依赖调试日志无法量化实际网络流量的问题。

核心职责：
1. **网络流量可观测**：区分 GitHub/GCS/用户自托管的流量来源
2. **迁移效果验证**：监控 GCS 迁移（Dickson 主导）的实际效果
3. **回归预警**：在 GitHub 再次投诉前捕获热路径回归
4. **错误分类**：将原始错误映射为有限集合的稳定分类

## 功能点目的

### 1. 插件获取遥测 (`logPluginFetch`)
- **触发时机**：
  - 启动时（install-counts 24h-TTL 刷新）
  - 显式用户操作（安装/更新）
  - **非每交互触发**（控制流量）
- **数据信封**：与 `tengu_binary_download_*` 事件类似
- **字段设计**：
  - `source`: 获取来源类型（install_counts, marketplace_clone, marketplace_pull, marketplace_url, plugin_clone, mcpb）
  - `host`: 主机名（白名单内公开主机名，其他归为 'other'，无法解析为 'unknown'）
  - `is_official`: 是否指向 anthropics/claude-plugins-official
  - `outcome`: success / failure / cache_hit
  - `duration_ms`: 四舍五入的毫秒耗时
  - `error_kind`: 错误分类（仅 failure 时）

### 2. 主机名提取与归一化 (`extractHost`)
- **支持格式**：
  - HTTPS URL: `https://host/...`
  - SCP 格式: `git@host:path`
  - SSH URL: `ssh://host/...`
- **白名单机制**：仅报告已知公共主机，内部主机名归为 'other'
- **已知公共主机**：
  ```
  github.com, raw.githubusercontent.com, objects.githubusercontent.com,
  gist.githubusercontent.com, gitlab.com, bitbucket.org, codeberg.org,
  dev.azure.com, ssh.dev.azure.com, storage.googleapis.com
  ```

### 3. 官方仓库检测 (`isOfficialRepo`)
- **用途**：让仪表板区分"我们的问题"流量与用户配置的市场
- **匹配模式**：URL/spec 包含 `anthropics/${OFFICIAL_MARKETPLACE_NAME}`

### 4. 错误分类 (`classifyFetchError`)
- **设计目标**：保持基数有限，避免原始错误消息爆炸
- **处理类型**：
  - Axios Error 对象（Node.js 错误码如 ENOTFOUND）
  - Git stderr 字符串（人类可读短语如 "Could not resolve host"）
- **分类优先级**：DNS > Timeout（Git 错误增强可能将 DNS 失败重写为包含 "timeout"）
- **分类映射**：
  | 模式 | 分类 |
  |------|------|
  | ENOTFOUND, ECONNREFUSED, EAI_AGAIN, "Could not resolve host", "Connection refused" | `dns_or_refused` |
  | ETIMEDOUT, "timed out", "timeout" | `timeout` |
  | ECONNRESET, "socket hang up", "Connection reset by peer", "remote end hung up" | `conn_reset` |
  | 403, 401, "authentication", "permission denied" | `auth` |
  | 404, "not found", "repository not found" | `not_found` |
  | "certificate", "SSL", "TLS", "unable to get local issuer" | `tls` |
  | "Invalid response format", "Invalid marketplace schema" | `invalid_schema` |
  | 其他 | `other` |

## 具体技术实现

### 主机名提取算法
```typescript
function extractHost(urlOrSpec: string): string {
  // 1. 尝试 SCP 格式匹配 (git@host:path)
  const scpMatch = /^[^@/]+@([^:/]+):/.exec(urlOrSpec)
  if (scpMatch) {
    host = scpMatch[1]!
  } else {
    // 2. 尝试 URL 解析
    try {
      host = new URL(urlOrSpec).hostname
    } catch {
      return 'unknown'
    }
  }
  // 3. 归一化并检查白名单
  const normalized = host.toLowerCase()
  return KNOWN_PUBLIC_HOSTS.has(normalized) ? normalized : 'other'
}
```

### 错误分类实现
```typescript
export function classifyFetchError(error: unknown): string {
  const msg = String((error as { message?: unknown })?.message ?? error)
  
  // DNS 检查优先于 Timeout（Git 错误增强可能混淆）
  if (/ENOTFOUND|ECONNREFUSED|EAI_AGAIN|Could not resolve host|Connection refused/i.test(msg)) {
    return 'dns_or_refused'
  }
  if (/ETIMEDOUT|timed out|timeout/i.test(msg)) return 'timeout'
  if (/ECONNRESET|socket hang up|Connection reset by peer|remote end hung up/i.test(msg)) {
    return 'conn_reset'
  }
  // ... 其他分类
}
```

### 隐私保护设计
- **字符串值**：有限枚举/仅主机名，无代码、无路径、无原始错误消息
- **与 `tengu_web_fetch_host`** 使用相同的隐私信封
- **PII 标记**：插件名/市场名使用 `_PROTO_*` 标记的 BQ 列

## 关键代码路径与文件引用

### 内部调用图
```
fetchTelemetry.ts
  ├─ logPluginFetch(source, urlOrSpec, outcome, durationMs, errorKind?)
  │   ├─ extractHost(urlOrSpec) → host
  │   ├─ isOfficialRepo(urlOrSpec) → boolean
  │   └─ services/analytics/index.ts: logEvent('tengu_plugin_remote_fetch', {...})
  └─ classifyFetchError(error) → errorKind
```

### 外部调用方
| 调用方 | 用途 |
|--------|------|
| `installCounts.ts` | 获取安装统计时记录遥测 |
| `marketplaceManager.ts` | 市场克隆/拉取/URL获取时记录 |
| `pluginInstallationHelpers.ts` | 插件克隆时记录 |
| `mcpbHandler.ts` | MCPB 获取时记录 |

## 依赖与外部交互

### 依赖模块
| 模块 | 用途 |
|------|------|
| `services/analytics/index.ts` | `logEvent` 上报 |
| `officialMarketplace.ts` | `OFFICIAL_MARKETPLACE_NAME` 常量 |

### 类型导出
```typescript
export type PluginFetchSource = 
  | 'install_counts' 
  | 'marketplace_clone' 
  | 'marketplace_pull' 
  | 'marketplace_url' 
  | 'plugin_clone' 
  | 'mcpb'

export type PluginFetchOutcome = 'success' | 'failure' | 'cache_hit'
```

## 风险、边界与改进建议

### 已知风险

1. **主机名白名单维护滞后**
   - 风险：新公共托管服务（如 AWS CodeCommit）未被加入白名单
   - 现状：归为 'other'，不丢失数据但减少可见性
   - 缓解：定期审查新增公共 Git 托管服务

2. **Git 错误增强导致的误分类**
   - 风险：`marketplaceManager.ts:~950` 将 DNS 失败重写为包含 "timeout"
   - 现状：分类器优先检查 DNS 模式
   - 潜在问题：若重写模式变化，可能误分类

3. **SCP 格式解析局限性**
   - 风险：非标准 SCP 格式（如包含端口号）可能解析错误
   - 现状：正则 `^[^@/]+@([^:/]+):` 不捕获端口
   - 结果：返回 'unknown'，保守处理

### 边界条件

| 场景 | 行为 |
|------|------|
| urlOrSpec 为 undefined | host = 'unknown', is_official = false |
| URL 解析失败 | host = 'unknown' |
| 主机名在白名单 | 返回小写主机名 |
| 主机名不在白名单 | 返回 'other'（保护内部主机名） |
| 错误消息为 undefined | 转为空字符串分类为 'other' |
| 多错误模式匹配 | 按代码顺序优先（DNS > Timeout > ...） |

### 改进建议

1. **自动主机名发现**
   - 当前：硬编码白名单
   - 建议：基于历史流量自动建议新增白名单条目

2. **更细粒度的错误分类**
   - 当前：HTTP 状态码仅区分 403/401 vs 404
   - 建议：增加 `rate_limited` (429), `server_error` (5xx) 分类

3. **重试遥测**
   - 当前：仅记录最终 outcome
   - 建议：增加 `retry_count` 字段，监控重试模式

4. **缓存命中率细分**
   - 当前：cache_hit 为二元值
   - 建议：区分 memory_cache / disk_cache / cdn_cache
