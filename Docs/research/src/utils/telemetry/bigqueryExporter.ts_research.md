# BigQuery Metrics Exporter 研究文档

## 场景与职责

`bigqueryExporter.ts` 实现了 Claude Code 的内部指标导出器，专门用于将 OpenTelemetry 指标数据导出到 Anthropic 内部的 BigQuery 分析管道。该模块是产品分析和运维监控的关键组件，服务于以下场景：

1. **产品分析**：收集 Claude Code 的使用指标，用于产品改进和用户体验优化
2. **成本追踪**：监控 API 调用成本、Token 使用量
3. **性能监控**：追踪工具执行时间、API 延迟
4. **业务指标**：企业用户（C4E/Teams）的使用情况分析

### 目标用户群体

- **1P API 客户**：直接使用 Anthropic API 的客户（排除 Claude.ai 订阅者和 Bedrock/Vertex 用户）
- **Claude for Enterprise (C4E)**：企业版用户
- **Claude for Teams**：团队版用户

## 功能点目的

### 1. 指标数据转换
将 OpenTelemetry 的 `ResourceMetrics` 格式转换为内部 BigQuery 管道期望的 `InternalMetricsPayload` 格式：

```typescript
type InternalMetricsPayload = {
  resource_attributes: Record<string, string>  // 资源级属性
  metrics: Metric[]                            // 指标数组
}

type Metric = {
  name: string
  description?: string
  unit?: string
  data_points: DataPoint[]
}

type DataPoint = {
  attributes: Record<string, string>
  value: number
  timestamp: string  // ISO 格式
}
```

### 2. 客户类型识别
自动识别用户类型并添加到资源属性：
- `user.customer_type`: 'claude_ai' | 'api'
- `user.subscription_type`: 'enterprise' | 'team' | undefined

### 3. 信任与权限检查
- **信任对话框**：在交互模式下，确保用户已接受信任对话框后才发送指标
- **组织级退出**：检查组织级别的指标退出设置（`checkMetricsEnabled()`）

### 4. 认证与授权
使用 `getAuthHeaders()` 获取认证头，确保指标数据与特定用户/组织关联

## 具体技术实现

### 类定义：BigQueryMetricsExporter

```typescript
export class BigQueryMetricsExporter implements PushMetricExporter {
  private readonly endpoint: string     // BQ 端点 URL
  private readonly timeout: number      // 请求超时（默认 5s）
  private pendingExports: Promise<void>[]  // 待处理的导出请求
  private isShutdown = false            // 关闭状态标志
}
```

### 关键流程

#### 1. 构造函数

```
端点选择逻辑：
1. Ant 用户 + ANT_CLAUDE_CODE_METRICS_ENDPOINT 存在：
   endpoint = ANT_CLAUDE_CODE_METRICS_ENDPOINT + '/api/claude_code/metrics'
2. 其他用户：
   endpoint = 'https://api.anthropic.com/api/claude_code/metrics'
```

#### 2. 指标导出流程 (`export`)

```
export(metrics, resultCallback):
1. 检查 isShutdown，已关闭则返回 FAILED
2. 创建 doExport Promise 并添加到 pendingExports
3. Promise.finally 中从 pendingExports 移除
4. 不等待完成，立即返回（异步导出）
```

#### 3. 实际导出逻辑 (`doExport`)

```
doExport(metrics, resultCallback):
1. 信任检查：
   - 交互模式：需已接受信任对话框
   - 非交互模式：跳过
   
2. 组织级退出检查：
   - 调用 checkMetricsEnabled()
   - 如禁用，返回 SUCCESS（静默跳过）
   
3. 数据转换：
   - transformMetricsForInternal(metrics)
   
4. 认证：
   - getAuthHeaders() 获取认证头
   - 如失败，返回 FAILED
   
5. HTTP 请求：
   - POST 到 endpoint
   - 超时：this.timeout (默认 5s)
   - Headers: Content-Type, User-Agent, Auth headers
   
6. 结果处理：
   - 成功：记录调试日志，返回 SUCCESS
   - 失败：记录错误，返回 FAILED
```

#### 4. 指标转换 (`transformMetricsForInternal`)

```
transformMetricsForInternal(metrics):
1. 提取资源属性：
   - service.name, service.version
   - os.type, os.version
   - host.arch
   - wsl.version（如存在）
   - aggregation.temporality: 'delta' | 'cumulative'
   
2. 添加客户类型属性：
   - 如果是 Claude.ai 订阅者：
     * user.customer_type = 'claude_ai'
     * user.subscription_type = subscriptionType
   - 否则：
     * user.customer_type = 'api'
     
3. 转换指标数据：
   - flatMap scopeMetrics → metrics
   - 每个指标包含：name, description, unit, data_points
```

#### 5. 数据点提取 (`extractDataPoints`)

```
extractDataPoints(metric):
1. 过滤：只保留数值类型的数据点
2. 映射转换：
   - attributes: convertAttributes(point.attributes)
   - value: point.value
   - timestamp: hrTimeToISOString(point.endTime || point.startTime)
```

### 辅助方法

| 方法 | 用途 |
|-----|------|
| `convertAttributes()` | 将 OTel Attributes 转换为字符串键值对 |
| `hrTimeToISOString()` | 将高分辨率时间戳转换为 ISO 字符串 |
| `selectAggregationTemporality()` | 返回 DELTA（强制，不可更改） |
| `shutdown()` | 关闭导出器，等待所有待处理导出完成 |
| `forceFlush()` | 强制刷新所有待处理导出 |

## 关键代码路径与文件引用

### 导出类

- `BigQueryMetricsExporter` - 主导出类，实现 OpenTelemetry 的 `PushMetricExporter` 接口

### 依赖文件

| 文件 | 用途 |
|-----|------|
| `src/services/api/metricsOptOut.js` | `checkMetricsEnabled()` 检查组织级指标退出 |
| `src/bootstrap/state.js` | `getIsNonInteractiveSession()` 检查非交互模式 |
| `src/utils/auth.js` | `getSubscriptionType()`, `isClaudeAISubscriber()`, `is1PApiCustomer()` 客户类型识别 |
| `src/utils/config.js` | `checkHasTrustDialogAccepted()` 信任对话框状态 |
| `src/utils/debug.js` | `logForDebugging()` 调试日志 |
| `src/utils/errors.js` | `errorMessage()`, `toError()` 错误处理 |
| `src/utils/http.js` | `getAuthHeaders()` 获取认证头 |
| `src/utils/log.js` | `logError()` 错误日志 |
| `src/utils/slowOperations.js` | `jsonStringify()` JSON 序列化 |
| `src/utils/userAgent.js` | `getClaudeCodeUserAgent()` 用户代理字符串 |

### 被调用方

- `src/utils/telemetry/instrumentation.ts` - `getBigQueryExportingReader()` 创建 MetricReader

### OpenTelemetry 依赖

- `@opentelemetry/api` - `Attributes`, `HrTime`
- `@opentelemetry/core` - `ExportResult`, `ExportResultCode`
- `@opentelemetry/sdk-metrics` - `PushMetricExporter`, `MetricData`, `ResourceMetrics`, `AggregationTemporality`

## 依赖与外部交互

### 环境变量

| 变量名 | 用途 |
|-------|------|
| `ANT_CLAUDE_CODE_METRICS_ENDPOINT` | Ant 用户专用端点（可选） |
| `USER_TYPE` | 用户类型识别（'ant' 为内部员工） |

### 外部 API

- **端点**：`https://api.anthropic.com/api/claude_code/metrics`
- **方法**：POST
- **认证**：通过 `getAuthHeaders()` 获取（通常是 API Key 或 OAuth Token）
- **超时**：默认 5 秒（可通过构造函数选项配置）

### 内部服务

- **metricsOptOut**：检查组织是否禁用指标收集
- **Auth 系统**：获取用户认证信息

## 风险、边界与改进建议

### 风险

1. **数据丢失风险**
   - 导出是"即发即忘"模式，不保证送达
   - 进程退出时可能丢失未发送的指标
   - 网络故障时指标丢失（无本地持久化）

2. **隐私合规风险**
   - 需要确保不收集 PII（个人身份信息）
   - 组织级退出检查必须可靠
   - 信任对话框检查防止过早发送数据

3. **性能影响**
   - 每 5 分钟导出一次（由 instrumentation.ts 配置）
   - HTTP 请求可能阻塞（虽然使用 async/await）
   - 大量指标时内存占用增长（`pendingExports` 数组）

4. **认证失败**
   - 如果 `getAuthHeaders()` 失败，指标被丢弃
   - 未登录状态下指标完全丢失

### 边界情况

1. **信任对话框未接受**
   - 交互模式下，用户未接受信任对话框前跳过导出
   - 返回 SUCCESS（静默跳过），不触发重试

2. **组织级退出**
   - 组织禁用指标时，所有导出静默跳过
   - 需要重新登录才能获取最新设置

3. **关闭状态**
   - 调用 `shutdown()` 后，新导出请求返回 FAILED
   - 待处理导出继续完成

4. **时间戳转换**
   - 使用 `HrTime`（[seconds, nanoseconds]）转 ISO 字符串
   - 纳秒级精度在转换为毫秒时丢失

### 改进建议

1. **可靠性增强**
   - 添加本地持久化队列，网络故障时缓存指标
   - 实现指数退避重试机制
   - 添加指标丢失监控和告警

2. **性能优化**
   - 批量压缩指标数据（gzip）减少传输大小
   - 使用 HTTP/2 连接复用
   - 添加指标采样率配置（高流量场景）

3. **可观测性**
   - 导出器自身指标：导出延迟、成功率、队列深度
   - 添加导出失败详细原因分类
   - 记录导出数据量统计

4. **配置灵活性**
   - 支持通过环境变量配置导出间隔
   - 支持配置端点超时时间
   - 支持自定义指标过滤器

5. **安全增强**
   - 指标数据本地加密（敏感环境）
   - 支持 mTLS 连接
   - 添加请求签名验证

6. **代码改进**
   - `pendingExports` 数组可能无限增长（虽然通常很小）
   - 考虑设置最大并发导出限制
   - 添加单元测试覆盖边界情况
