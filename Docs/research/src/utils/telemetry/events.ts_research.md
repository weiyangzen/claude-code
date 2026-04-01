# Telemetry Events 研究文档

## 场景与职责

`events.ts` 是 Claude Code 的 OpenTelemetry 事件日志模块，提供轻量级的事件记录功能。该模块用于发送结构化事件数据到 OTel 后端（如 Honeycomb），服务于以下场景：

1. **系统提示词记录**：在 Beta 追踪中记录系统提示词内容（去重后）
2. **工具模式记录**：记录工具定义和模式信息
3. **会话生命周期事件**：记录会话启动、技能加载等事件
4. **调试和诊断**：发送详细的调试信息用于故障排查

### 核心特点

- **轻量级**：基于 OTel Events API，不依赖复杂追踪
- **序列化保证**：使用单调递增的序列号保证事件顺序
- **隐私控制**：支持用户提示词脱敏（`<REDACTED>`）
- **测试环境安全**：测试环境自动跳过事件发送

## 功能点目的

### 1. 事件序列号
- **目的**：保证同一会话内事件的顺序性，便于时序分析
- **实现**：模块级 `eventSequence` 计数器，每次发送事件自增

### 2. 用户提示词脱敏
- **目的**：保护用户隐私，避免在遥测中记录敏感输入
- **实现**：`OTEL_LOG_USER_PROMPTS` 环境变量控制，默认为 `<REDACTED>`

### 3. 事件日志器懒加载
- **目的**：避免在事件日志器初始化前发送事件导致错误
- **实现**：检查 `getEventLogger()` 返回值，未初始化时记录一次警告后静默丢弃

### 4. 工作区路径记录
- **目的**：关联事件与特定工作区（仅用于分析，不用于指标）
- **实现**：从 `CLAUDE_CODE_WORKSPACE_HOST_PATHS` 环境变量读取

## 具体技术实现

### 关键数据结构

```typescript
// 模块级状态
let eventSequence = 0                    // 单调递增序列号
let hasWarnedNoEventLogger = false       // 防止重复警告

// 事件属性（自动添加）
interface EventAttributes {
  'event.name': string                    // 事件名称
  'event.timestamp': string               // ISO 时间戳
  'event.sequence': number                // 序列号
  'prompt.id': string | undefined         // 当前提示词 ID
  'workspace.host_paths': string[]        // 工作区路径（如有）
}
```

### 关键流程

#### 1. 用户提示词脱敏检查

```typescript
function isUserPromptLoggingEnabled() {
  return isEnvTruthy(process.env.OTEL_LOG_USER_PROMPTS)
}

export function redactIfDisabled(content: string): string {
  return isUserPromptLoggingEnabled() ? content : '<REDACTED>'
}
```

#### 2. 事件发送流程 (`logOTelEvent`)

```
logOTelEvent(eventName, metadata):
1. 获取事件日志器：
   eventLogger = getEventLogger()
   
2. 检查日志器状态：
   - 如果为 null：
     * 如果未警告过，记录警告日志
     * 返回（静默丢弃事件）
     
3. 测试环境检查：
   - 如果 NODE_ENV === 'test'：
     * 返回（测试环境不发送事件）
     
4. 构建属性对象：
   - 合并 getTelemetryAttributes() 返回的基础属性
   - 添加 event.name, event.timestamp, event.sequence
   - 添加 prompt.id（如有）
   - 添加 workspace.host_paths（如有）
   - 合并调用方提供的 metadata
   
5. 发送事件：
   eventLogger.emit({
     body: `claude_code.${eventName}`,
     attributes
   })
```

### 属性构建详解

```typescript
const attributes: Attributes = {
  // 基础遥测属性（用户 ID、会话 ID 等）
  ...getTelemetryAttributes(),
  
  // 事件元数据
  'event.name': eventName,
  'event.timestamp': new Date().toISOString(),
  'event.sequence': eventSequence++,
}

// 关联当前提示词（用于追踪请求链路）
const promptId = getPromptId()
if (promptId) {
  attributes['prompt.id'] = promptId
}

// 工作区路径（高基数，仅用于事件，不用于指标）
const workspaceDir = process.env.CLAUDE_CODE_WORKSPACE_HOST_PATHS
if (workspaceDir) {
  attributes['workspace.host_paths'] = workspaceDir.split('|')
}
```

## 关键代码路径与文件引用

### 导出函数

| 函数名 | 用途 | 调用位置 |
|-------|------|---------|
| `logOTelEvent(eventName, metadata)` | 发送 OTel 事件 | betaSessionTracing.ts, toolExecution.ts, hooks.ts 等 |
| `redactIfDisabled(content)` | 脱敏敏感内容 | sessionTracing.ts |

### 依赖文件

| 文件 | 用途 |
|-----|------|
| `src/bootstrap/state.js` | `getEventLogger()`, `getPromptId()` 获取事件日志器和提示词 ID |
| `src/utils/debug.js` | `logForDebugging()` 调试日志 |
| `src/utils/envUtils.js` | `isEnvTruthy()` 环境变量检查 |
| `src/utils/telemetryAttributes.ts` | `getTelemetryAttributes()` 获取基础遥测属性 |

### 被调用方

- `src/utils/telemetry/betaSessionTracing.ts` - 发送系统提示词和工具事件
- `src/utils/telemetry/sessionTracing.ts` - 使用 `redactIfDisabled()` 脱敏用户提示词
- `src/services/tools/toolExecution.ts` - 发送工具执行事件
- `src/utils/hooks.ts` - 发送 Hook 执行事件
- `src/services/api/logging.ts` - 发送 API 相关事件
- `src/components/FeedbackSurvey/*.tsx` - 发送反馈调查事件

### OpenTelemetry 依赖

- `@opentelemetry/api` - `Attributes` 类型

## 依赖与外部交互

### 环境变量

| 变量名 | 用途 |
|-------|------|
| `OTEL_LOG_USER_PROMPTS` | 启用用户提示词记录（truthy 值） |
| `CLAUDE_CODE_WORKSPACE_HOST_PATHS` | 工作区路径（来自桌面应用） |
| `NODE_ENV` | 测试环境检测（'test' 时跳过） |

### 状态依赖

- **EventLogger**：通过 `getEventLogger()` 获取，在 `instrumentation.ts` 中初始化
- **PromptId**：通过 `getPromptId()` 获取，在会话中更新

### 外部服务

- OTel Logs API 后端（通过 `eventLogger.emit()` 发送）
- 通常是 OTLP 端点（Honeycomb 或内部服务）

## 风险、边界与改进建议

### 风险

1. **事件丢失**
   - EventLogger 未初始化时事件被静默丢弃
   - 仅记录一次警告，后续事件无痕迹丢失
   - 测试环境完全跳过事件发送

2. **序列号限制**
   - `eventSequence` 是模块级变量，进程重启后重置
   - 长会话中可能溢出（虽然 2^53 很大）

3. **属性污染**
   - metadata 直接合并到属性中，无键名冲突检查
   - 可能意外覆盖系统属性（event.name 等）

4. **性能影响**
   - 每次事件调用 `new Date().toISOString()`
   - `getTelemetryAttributes()` 可能涉及同步计算

### 边界情况

1. **EventLogger 未初始化**
   - 启动初期或初始化失败时，事件被丢弃
   - 仅记录一次警告，避免日志 spam

2. **空 metadata**
   - 支持不传 metadata 或传空对象
   - 只发送基础事件属性

3. **Workspace 路径格式**
   - 使用 `|` 分隔多个路径
   - 调用方需确保格式正确

4. **Prompt ID 变化**
   - 事件发送时的 promptId 可能与实际处理时不一致
   - 异步操作可能导致关联错误

### 改进建议

1. **可靠性增强**
   - 添加事件队列，EventLogger 初始化后补发
   - 记录丢弃事件的数量统计
   - 添加事件发送失败重试机制

2. **可观测性**
   - 添加事件发送延迟指标
   - 记录事件队列深度
   - 导出事件丢失率统计

3. **性能优化**
   - 缓存 `getTelemetryAttributes()` 结果（如适用）
   - 使用高分辨率时间戳替代 Date
   - 批量发送事件减少网络开销

4. **安全增强**
   - 添加 metadata 键名白名单/黑名单
   - 自动脱敏敏感模式（如 email、phone）
   - 添加事件内容大小限制

5. **代码改进**
   - 使用 WeakRef 避免内存泄漏
   - 添加单元测试覆盖边界情况
   - 考虑使用结构化克隆替代对象展开

6. **配置灵活性**
   - 支持按事件名称配置采样率
   - 支持动态调整日志级别
   - 支持自定义事件属性前缀
