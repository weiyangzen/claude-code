# sinkKillswitch.ts 深度研究文档

## 场景与职责

`sinkKillswitch.ts` 是 Claude Code 分析系统的紧急切断开关模块，提供按 sink（数据目的地）细粒度控制分析数据发送的能力。当某个分析后端出现问题（如 API 故障、数据异常、成本过高等）时，可以通过远程配置快速禁用特定 sink，而无需发布新版本或重启应用。

### 核心职责

1. **按 Sink 紧急切断**：支持单独禁用 `datadog` 或 `firstParty` sink
2. **远程配置驱动**：通过 GrowthBook 动态配置控制开关状态
3. **故障开放设计**：配置缺失或格式错误时，默认保持 sink 开启
4. **防止循环依赖**：特别注意避免与 `growthbook.ts` 中的 `is1PEventLoggingEnabled()` 产生循环调用

---

## 功能点目的

### 1. Sink 类型定义

```typescript
export type SinkName = 'datadog' | 'firstParty'
```

**目的**：明确定义支持的 sink 类型，提供类型安全。

### 2. Sink Killswitch 检查

```typescript
export function isSinkKilled(sink: SinkName): boolean
```

**目的**：检查指定 sink 是否被禁用。实现细节：
- 从 GrowthBook 获取配置 `tengu_frond_boric`
- 配置格式：`{ datadog?: boolean, firstParty?: boolean }`
- `true` 表示禁用该 sink
- 配置缺失或格式错误时返回 `false`（故障开放）

**重要约束**：
> 绝对不能从 `is1PEventLoggingEnabled()` 内部调用此函数，因为 `growthbook.ts:isGrowthBookEnabled()` 会调用 `is1PEventLoggingEnabled()`，产生循环依赖。

---

## 具体技术实现

### 关键数据结构

```typescript
// Mangled name: per-sink analytics killswitch
const SINK_KILLSWITCH_CONFIG_NAME = 'tengu_frond_boric'

export type SinkName = 'datadog' | 'firstParty'
```

### 关键流程

#### 1. Killswitch 检查流程

```
isSinkKilled(sink: SinkName)
    │
    ├──> getDynamicConfig_CACHED_MAY_BE_STALE<Partial<Record<SinkName, boolean>>>(
    │       SINK_KILLSWITCH_CONFIG_NAME, 
    │       {}  // 默认空对象
    │    )
    │
    └──> return config?.[sink] === true
            ├──> true  → sink 被禁用
            └──> false → sink 保持开启（包括 undefined、null、false 等情况）
```

### 配置格式示例

```json
{
  "datadog": false,      // Datadog sink 保持开启
  "firstParty": true     // 1P 事件日志 sink 被禁用
}
```

```json
{
  "datadog": true        // 仅禁用 Datadog，firstParty 保持开启（未指定 = 默认开启）
}
```

```json
{}                       // 空对象，所有 sink 保持开启
```

---

## 关键代码路径与文件引用

### 内部依赖

| 导入路径 | 用途 |
|---------|------|
| `./growthbook.js` | `getDynamicConfig_CACHED_MAY_BE_STALE` - 获取动态配置 |

### 调用方

| 函数 | 调用方 |
|------|--------|
| `isSinkKilled` | `sink.ts` (shouldTrackDatadog), `firstPartyEventLogger.ts` (logEventTo1P, logGrowthBookExperimentTo1P), `firstPartyEventLoggingExporter.ts` (构造函数 isKilled 回调) |

### 依赖关系图

```
sinkKillswitch.ts
    └──> growthbook.ts (getDynamicConfig_CACHED_MAY_BE_STALE)
            └──> (注意：isGrowthBookEnabled 调用 is1PEventLoggingEnabled，
                 所以 sinkKillswitch 不能反过来被 is1PEventLoggingEnabled 调用)
```

### 调用点详细分析

#### 1. sink.ts

```typescript
function shouldTrackDatadog(): boolean {
  if (isSinkKilled('datadog')) {
    return false
  }
  // ...
}
```

在每个事件分发点检查 Datadog sink 是否被禁用。

#### 2. firstPartyEventLogger.ts

```typescript
export function logEventTo1P(eventName: string, metadata: Record<string, number | boolean | undefined> = {}): void {
  if (!is1PEventLoggingEnabled()) {
    return
  }
  if (!firstPartyEventLogger || isSinkKilled('firstParty')) {
    return
  }
  // ...
}

export function logGrowthBookExperimentTo1P(data: GrowthBookExperimentData): void {
  if (!is1PEventLoggingEnabled()) {
    return
  }
  if (!firstPartyEventLogger || isSinkKilled('firstParty')) {
    return
  }
  // ...
}
```

在 1P 事件日志函数入口检查 firstParty sink 是否被禁用。

#### 3. firstPartyEventLoggingExporter.ts

```typescript
constructor(options: { ... isKilled?: () => boolean }) {
  this.isKilled = options.isKilled ?? (() => false)
}

private async sendBatchWithRetry(payload: FirstPartyEventLoggingPayload): Promise<void> {
  if (this.isKilled()) {
    throw new Error('firstParty sink killswitch active')
  }
  // ...
}
```

通过构造函数注入 `isKilled` 回调，在每次网络请求前检查，确保 killswitch 激活时立即停止网络流量。

---

## 依赖与外部交互

### 外部系统交互

1. **GrowthBook**：
   - 配置键名：`tengu_frond_boric`
   - 使用 `getDynamicConfig_CACHED_MAY_BE_STALE` 获取配置
   - 配置更新通过 GrowthBook 的定期刷新机制（6小时/20分钟）

2. **调用方模块**：
   - `sink.ts`：控制 Datadog 事件发送
   - `firstPartyEventLogger.ts`：控制 1P 事件日志发送
   - `firstPartyEventLoggingExporter.ts`：控制 1P 导出器重试行为

---

## 风险、边界与改进建议

### 风险点

1. **循环依赖风险**（已规避）
   - `growthbook.ts:isGrowthBookEnabled()` → `is1PEventLoggingEnabled()`
   - 如果 `is1PEventLoggingEnabled()` → `isSinkKilled()` → `getDynamicConfig_CACHED_MAY_BE_STALE()` → `isGrowthBookEnabled()`，形成循环
   - 当前通过在 `isSinkKilled` 文档中明确警告来规避

2. **配置传播延迟**
   - 使用 `_CACHED_MAY_BE_STALE` 后缀的函数，意味着配置可能有延迟
   - 在紧急情况下，killswitch 可能需要最多 6 小时（外部用户）或 20 分钟（ant 用户）才能完全生效

3. **配置格式错误**
   - 如果配置不是有效的 JSON 对象，可能导致意外行为
   - 当前使用 `config?.[sink] === true` 严格检查，格式错误时故障开放

4. **命名混淆**
   - 配置名 `tengu_frond_boric` 是混淆后的名称，不易理解
   - 需要文档说明其含义

### 边界情况

1. **配置为 `null`**：
   ```typescript
   // getFeatureValue_CACHED_MAY_BE_STALE guards on `!== undefined`, 
   // so a cached JSON null leaks through instead of falling back to {}.
   // 因此需要检查 config?.[sink] === true
   ```

2. **配置为数组或其他非对象类型**：
   - `config?.[sink]` 会返回 `undefined`，视为未禁用

3. **Sink 名称拼写错误**：
   - TypeScript 类型系统会捕获编译期错误
   - 但如果通过字符串索引访问，可能返回 `undefined`，视为未禁用

4. **GrowthBook 完全不可用**：
   - 回退到默认空对象 `{}`，所有 sink 保持开启

### 改进建议

1. **添加运行时类型验证**
   ```typescript
   export function isSinkKilled(sink: SinkName): boolean {
     const config = getDynamicConfig_CACHED_MAY_BE_STALE<unknown>(SINK_KILLSWITCH_CONFIG_NAME, {})
     
     // 运行时验证配置格式
     if (!config || typeof config !== 'object' || Array.isArray(config)) {
       return false
     }
     
     const value = (config as Record<string, unknown>)?.[sink]
     return value === true  // 严格等于 true，排除其他 truthy 值
   }
   ```

2. **添加监控和告警**
   ```typescript
   export function isSinkKilled(sink: SinkName): boolean {
     const config = getDynamicConfig_CACHED_MAY_BE_STALE<Partial<Record<SinkName, boolean>>>(
       SINK_KILLSWITCH_CONFIG_NAME, 
       {}
     )
     const killed = config?.[sink] === true
     
     // 添加指标或日志，便于监控 killswitch 状态
     if (process.env.USER_TYPE === 'ant' && killed) {
       logForDebugging(`Sink killswitch: ${sink} is disabled`)
     }
     
     return killed
   }
   ```

3. **支持更多细粒度控制**
   ```typescript
   // 考虑支持按事件类型或用户组控制
   export type SinkKillswitchConfig = {
     datadog?: boolean | { events?: string[] }
     firstParty?: boolean | { events?: string[] }
   }
   ```

4. **添加测试覆盖**
   - 当前没有专门的测试文件
   - 建议添加单元测试验证：
     - 配置解析逻辑
     - 故障开放行为
     - 类型安全

5. **文档改进**
   - 在代码注释中添加配置变更流程说明
   - 添加示例配置和预期行为对照表

6. **配置键名可读性**
   - 考虑添加一个常量映射，将混淆名称与可读名称关联
   ```typescript
   // 内部文档
   const CONFIG_KEY_DOCS = {
     'tengu_frond_boric': 'Per-sink analytics killswitch. Format: { datadog?: boolean, firstParty?: boolean }'
   }
   ```
