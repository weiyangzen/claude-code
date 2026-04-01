# Telemetry Instrumentation 研究文档

## 场景与职责

`instrumentation.ts` 是 Claude Code 的 OpenTelemetry 遥测基础设施核心模块，负责初始化和管理完整的 OTel 链路（Metrics、Logs、Traces）。该模块是应用启动的关键路径，服务于以下场景：

1. **第三方遥测**：支持用户配置的 OTel 后端（Datadog、Honeycomb、Jaeger 等）
2. **内部遥测**：Anthropic 内部使用的 BigQuery 指标导出
3. **Beta 追踪**：独立的详细追踪管道用于调试
4. **Perfetto 追踪**：Ant 专用的性能分析追踪

### 架构特点

- **懒加载**：gRPC 导出器动态导入，避免加载 ~700KB 的 `@grpc/grpc-js`
- **协议支持**：支持 grpc、http/json、http/protobuf 三种 OTLP 协议
- **多导出器**：支持同时配置多个导出器（console + otlp + prometheus）
- **优雅关闭**：注册进程退出处理，确保数据刷新

## 功能点目的

### 1. 环境变量引导
- **目的**：Ant 用户使用 `ANT_*` 前缀变量，避免与外部用户配置冲突
- **实现**：`bootstrapTelemetry()` 函数将 `ANT_OTEL_*` 复制到标准 `OTEL_*` 变量

### 2. 协议动态选择
- **目的**：根据配置动态加载对应导出器，减少内存占用
- **实现**：switch 语句 + 动态 import，只加载实际使用的协议实现

### 3. 信任对话框感知
- **目的**：在交互模式下，用户接受信任对话框前不发送遥测数据
- **实现**：`BigQueryMetricsExporter` 中检查 `checkHasTrustDialogAccepted()`

### 4. 格式化输出兼容
- **目的**：避免 console 导出器破坏 JSON 流输出（SDK 模式）
- **实现**：`getHasFormattedOutput()` 为 true 时自动移除 console 导出器

### 5. 代理和 mTLS 支持
- **目的**：支持企业代理环境和双向 TLS 认证
- **实现**：`getOTLPExporterConfig()` 集成代理和 mTLS 配置

## 具体技术实现

### 关键数据结构

```typescript
// 默认导出间隔
const DEFAULT_METRICS_EXPORT_INTERVAL_MS = 60000   // 60s
const DEFAULT_LOGS_EXPORT_INTERVAL_MS = 5000       // 5s
const DEFAULT_TRACES_EXPORT_INTERVAL_MS = 5000     // 5s

// 超时错误类
class TelemetryTimeoutError extends Error {}

// 导出器类型解析
export function parseExporterTypes(value: string | undefined): string[] {
  return (value || '')
    .trim()
    .split(',')
    .filter(Boolean)
    .map(t => t.trim())
    .filter(t => t !== 'none')
}
```

### 关键流程

#### 1. 遥测初始化流程 (`initializeTelemetry`)

```
initializeTelemetry():
1. 性能标记：profileCheckpoint('telemetry_init_start')
2. 引导环境变量：bootstrapTelemetry()
3. 格式化输出处理：
   - 如果 getHasFormattedOutput() 为 true
   - 移除 OTEL_*_EXPORTER 中的 'console'
   
4. 设置诊断日志器：
   diag.setLogger(new ClaudeCodeDiagLogger(), DiagLogLevel.ERROR)
   
5. 初始化 Perfetto 追踪：
   initializePerfettoTracing()
   
6. 构建 MetricReader 列表：
   - 如果 telemetryEnabled：添加 OTLP readers
   - 如果 isBigQueryMetricsEnabled()：添加 BigQuery reader
   
7. 创建基础 Resource：
   - service.name = 'claude-code'
   - service.version = MACRO.VERSION
   - 添加 WSL 版本（如适用）
   
8. 合并资源检测器：
   - osDetector
   - hostDetector（仅 host.arch）
   - envDetector
   
9. Beta 追踪检查：
   - 如果 isBetaTracingEnabled()：
     * 初始化独立的 trace/log 导出器
     * 创建 MeterProvider
     * 注册关闭处理
     * 返回 meter
     
10. 标准遥测初始化：
    - 创建 MeterProvider
    - 如果 telemetryEnabled：
      * 初始化 Logs（LoggerProvider + BatchLogRecordProcessor）
      * 注册刷新处理（beforeExit/exit）
    - 如果 isEnhancedTelemetryEnabled()：
      * 初始化 Traces（BasicTracerProvider + BatchSpanProcessor）
    - 注册关闭处理
    - 返回 meter
```

#### 2. 环境变量引导 (`bootstrapTelemetry`)

```
bootstrapTelemetry():
1. 仅当 USER_TYPE === 'ant' 时执行
2. 复制 ANT_ 前缀变量到标准变量：
   - ANT_OTEL_METRICS_EXPORTER → OTEL_METRICS_EXPORTER
   - ANT_OTEL_LOGS_EXPORTER → OTEL_LOGS_EXPORTER
   - ANT_OTEL_TRACES_EXPORTER → OTEL_TRACES_EXPORTER
   - ANT_OTEL_EXPORTER_OTLP_PROTOCOL → OTEL_EXPORTER_OTLP_PROTOCOL
   - ANT_OTEL_EXPORTER_OTLP_ENDPOINT → OTEL_EXPORTER_OTLP_ENDPOINT
   - ANT_OTEL_EXPORTER_OTLP_HEADERS → OTEL_EXPORTER_OTLP_HEADERS
3. 设置默认 temporality：
   OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE = 'delta'
```

#### 3. OTLP Metric Reader 创建 (`getOtlpReaders`)

```
getOtlpReaders():
1. 解析 OTEL_METRICS_EXPORTER（逗号分隔）
2. 获取导出间隔（OTEL_METRIC_EXPORT_INTERVAL，默认 60s）
3. 遍历导出器类型：
   - 'console'：创建 ConsoleMetricExporter（包装以显示资源属性）
   - 'otlp'：
     * 确定协议（OTEL_EXPORTER_OTLP_METRICS_PROTOCOL 或 OTEL_EXPORTER_OTLP_PROTOCOL）
     * switch 动态导入对应导出器：
       - 'grpc'：@opentelemetry/exporter-metrics-otlp-grpc
       - 'http/json'：@opentelemetry/exporter-metrics-otlp-http
       - 'http/protobuf'：@opentelemetry/exporter-metrics-otlp-proto
   - 'prometheus'：@opentelemetry/exporter-prometheus
4. 包装为 PeriodicExportingMetricReader 返回
```

#### 4. OTLP Log Exporter 创建 (`getOtlpLogExporters`)

```
getOtlpLogExporters():
1. 解析 OTEL_LOGS_EXPORTER
2. 确定协议（OTEL_EXPORTER_OTLP_LOGS_PROTOCOL 或 OTEL_EXPORTER_OTLP_PROTOCOL）
3. 遍历导出器类型：
   - 'console'：ConsoleLogRecordExporter
   - 'otlp'：
     * switch 动态导入：
       - 'grpc'：@opentelemetry/exporter-logs-otlp-grpc
       - 'http/json'：@opentelemetry/exporter-logs-otlp-http
       - 'http/protobuf'：@opentelemetry/exporter-logs-otlp-proto
```

#### 5. OTLP Trace Exporter 创建 (`getOtlpTraceExporters`)

```
getOtlpTraceExporters():
1. 解析 OTEL_TRACES_EXPORTER
2. 确定协议（OTEL_EXPORTER_OTLP_TRACES_PROTOCOL 或 OTEL_EXPORTER_OTLP_PROTOCOL）
3. 遍历导出器类型：
   - 'console'：ConsoleSpanExporter
   - 'otlp'：
     * switch 动态导入对应导出器（同 logs）
```

#### 6. Beta 追踪初始化 (`initializeBetaTracing`)

```
initializeBetaTracing(resource):
1. 检查 BETA_TRACING_ENDPOINT 是否存在
2. 动态导入：
   - @opentelemetry/exporter-trace-otlp-http
   - @opentelemetry/exporter-logs-otlp-http
3. 创建 Trace Exporter：
   - url: ${endpoint}/v1/traces
   - BatchSpanProcessor（5s 调度延迟）
   - BasicTracerProvider
4. 创建 Log Exporter：
   - url: ${endpoint}/v1/logs
   - BatchLogRecordProcessor（5s 调度延迟）
   - LoggerProvider
5. 设置全局 Provider
6. 创建 EventLogger
7. 注册刷新处理（beforeExit/exit）
```

#### 7. OTLP 导出器配置 (`getOTLPExporterConfig`)

```
getOTLPExporterConfig():
1. 获取代理 URL 和 mTLS 配置
2. 解析静态 Headers（OTEL_EXPORTER_OTLP_HEADERS）
3. 配置动态 Headers（如设置了 otelHeadersHelper）
4. 代理处理：
   - 如果无代理或端点绕过代理：
     * 配置 httpAgentOptions（mTLS + CA certs）
   - 如果需要代理：
     * 创建 HttpsProxyAgent
     * 集成 mTLS 和 CA certs
5. 返回配置对象
```

#### 8. 遥测刷新 (`flushTelemetry`)

```
flushTelemetry():
1. 获取所有 Provider（meterProvider, loggerProvider, tracerProvider）
2. 创建刷新 Promise 列表
3. 使用 Promise.race 与超时竞争（默认 5s）
4. 超时后记录警告但不抛出（避免阻塞登出）
```

## 关键代码路径与文件引用

### 导出函数

| 函数名 | 用途 | 调用位置 |
|-------|------|---------|
| `bootstrapTelemetry()` | 引导 Ant 环境变量 | initializeTelemetry 内部 |
| `parseExporterTypes()` | 解析导出器类型字符串 | getOtlpReaders, getOtlpLogExporters, getOtlpTraceExporters |
| `isTelemetryEnabled()` | 检查遥测是否启用 | 多处 |
| `initializeTelemetry()` | 主初始化函数 | init.ts, main.tsx |
| `flushTelemetry()` | 强制刷新遥测 | 登出、组织切换时 |

### 依赖文件

| 文件 | 用途 |
|-----|------|
| `src/bootstrap/state.js` | Provider 状态管理（get/set Meter/Logger/Tracer/EventLogger） |
| `src/utils/auth.js` | `getOtelHeadersFromHelper()`, `getSubscriptionType()`, `isClaudeAISubscriber()`, `is1PApiCustomer()` |
| `src/utils/platform.js` | `getPlatform()`, `getWslVersion()` |
| `src/utils/caCerts.js` | `getCACertificates()` |
| `src/utils/cleanupRegistry.js` | `registerCleanup()` |
| `src/utils/debug.js` | `getHasFormattedOutput()`, `logForDebugging()` |
| `src/utils/envUtils.js` | `isEnvTruthy()` |
| `src/utils/errors.js` | `errorMessage()` |
| `src/utils/mtls.js` | `getMTLSConfig()` |
| `src/utils/proxy.js` | `getProxyUrl()`, `shouldBypassProxy()` |
| `src/utils/settings/settings.js` | `getSettings_DEPRECATED()` |
| `src/utils/slowOperations.js` | `jsonStringify()` |
| `src/utils/startupProfiler.js` | `profileCheckpoint()` |
| `src/utils/telemetry/betaSessionTracing.js` | `isBetaTracingEnabled()` |
| `src/utils/telemetry/bigqueryExporter.js` | `BigQueryMetricsExporter` |
| `src/utils/telemetry/logger.js` | `ClaudeCodeDiagLogger` |
| `src/utils/telemetry/perfettoTracing.js` | `initializePerfettoTracing()` |
| `src/utils/telemetry/sessionTracing.js` | `endInteractionSpan()`, `isEnhancedTelemetryEnabled()` |

### OpenTelemetry 依赖

- `@opentelemetry/api` - 核心 API（trace, logs, diag）
- `@opentelemetry/api-logs` - Logs API
- `@opentelemetry/resources` - 资源检测
- `@opentelemetry/sdk-logs` - Logs SDK
- `@opentelemetry/sdk-metrics` - Metrics SDK
- `@opentelemetry/sdk-trace-base` - Traces SDK
- `@opentelemetry/semantic-conventions` - 标准属性名
- `@opentelemetry/exporter-*` - 各类导出器（动态导入）

### 外部依赖

- `https-proxy-agent` - HTTP 代理支持

## 依赖与外部交互

### 环境变量

| 变量名 | 用途 |
|-------|------|
| `CLAUDE_CODE_ENABLE_TELEMETRY` | 启用第三方遥测 |
| `ANT_OTEL_*` | Ant 用户专用配置 |
| `OTEL_METRICS_EXPORTER` | 指标导出器类型（console,otlp,prometheus） |
| `OTEL_LOGS_EXPORTER` | 日志导出器类型（console,otlp） |
| `OTEL_TRACES_EXPORTER` | 追踪导出器类型（console,otlp） |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | OTLP 协议（grpc,http/json,http/protobuf） |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP 端点 URL |
| `OTEL_EXPORTER_OTLP_HEADERS` | OTLP 请求头 |
| `OTEL_EXPORTER_OTLP_*_PROTOCOL` | 信号特定协议覆盖 |
| `OTEL_METRIC_EXPORT_INTERVAL` | 指标导出间隔 |
| `OTEL_LOGS_EXPORT_INTERVAL` | 日志导出间隔 |
| `OTEL_TRACES_EXPORT_INTERVAL` | 追踪导出间隔 |
| `CLAUDE_CODE_OTEL_SHUTDOWN_TIMEOUT_MS` | 关闭超时（默认 2s） |
| `CLAUDE_CODE_OTEL_FLUSH_TIMEOUT_MS` | 刷新超时（默认 5s） |
| `BETA_TRACING_ENDPOINT` | Beta 追踪端点 |
| `USER_TYPE` | 用户类型（'ant'） |

### 外部服务

- OTLP 端点（用户配置的遥测后端）
- BigQuery 指标端点（`api.anthropic.com/api/claude_code/metrics`）
- Beta 追踪端点（`BETA_TRACING_ENDPOINT`）

## 风险、边界与改进建议

### 风险

1. **启动性能影响**
   - 动态导入虽然延迟加载，但仍可能阻塞启动
   - 资源检测器可能执行同步文件系统操作

2. **内存泄漏**
   - `pendingExports` 数组在 BigQueryExporter 中可能增长
   - Span 处理器持有对导出器的引用

3. **网络阻塞**
   - 关闭时的 `forceFlush` 可能阻塞进程退出
   - 超时机制虽存在，但默认 2s 可能不够

4. **配置复杂性**
   - 大量环境变量容易配置错误
   - 协议/导出器组合验证不足

5. **错误静默**
   - 许多错误仅记录到调试日志
   - 用户可能不知道遥测未正常工作

### 边界情况

1. **重复初始化**
   - `telemetryInitialized` 标志防止重复初始化
   - 但 Provider 全局设置可能冲突

2. **无导出器配置**
   - 空导出器列表时，Provider 仍创建但无数据流出

3. **协议不匹配**
   - 未知协议抛出错误，可能崩溃应用

4. **代理配置冲突**
   - mTLS + 代理组合复杂，可能配置错误

5. **关闭时刷新失败**
   - 网络不可用时不重试，数据丢失

### 改进建议

1. **启动优化**
   - 将遥测初始化移到后台线程（Worker）
   - 延迟初始化直到第一个指标/日志产生
   - 添加启动性能指标

2. **可靠性增强**
   - 添加本地持久化队列
   - 实现指数退避重试
   - 添加导出失败告警

3. **配置简化**
   - 提供配置验证函数
   - 添加配置示例和文档
   - 支持配置文件（非仅环境变量）

4. **可观测性**
   - 遥测系统自身的健康指标
   - 导出延迟、成功率监控
   - 配置状态诊断端点

5. **错误处理**
   - 更详细的错误分类
   - 用户友好的错误提示
   - 自动恢复机制

6. **安全增强**
   - 敏感配置（headers）加密存储
   - 支持配置热更新
   - 审计日志记录配置变更
