# growthbook.ts 研究文档

## 场景与职责

`growthbook.ts` 是 Claude Code 的功能开关（Feature Flag）和 A/B 测试平台集成模块，使用 GrowthBook SDK 实现动态配置和实验管理。它替代了早期的 Statsig 实现，提供更灵活的本地评估和远程评估混合模式。

核心职责：
- GrowthBook 客户端初始化和生命周期管理
- 功能开关值获取（阻塞和非阻塞 API）
- 动态配置管理
- 实验曝光日志
- 配置热更新和定期刷新
- 本地覆盖（环境变量和配置文件）

## 功能点目的

### 1. 功能开关 API

**目的**：提供多种获取功能开关值的 API，适应不同性能要求。

**API 层级**：

| API | 阻塞性 | 使用场景 |
|-----|--------|----------|
| `getFeatureValue_DEPRECATED` | 阻塞 | 旧代码，需要等待初始化 |
| `getFeatureValue_CACHED_MAY_BE_STALE` | 非阻塞 | 启动关键路径，优先使用 |
| `getFeatureValue_CACHED_WITH_REFRESH` | 非阻塞（已废弃） | 原 TTL 刷新，现直接代理到 CACHED |
| `checkStatsigFeatureGate_CACHED_MAY_BE_STALE` | 非阻塞 | Statsig 迁移期兼容 |
| `checkSecurityRestrictionGate` | 条件阻塞 | 安全检查，等待重新初始化 |
| `checkGate_CACHED_OR_BLOCKING` | 混合 | 用户触发功能，staleness 敏感 |

### 2. 远程评估 (`remoteEval`)

**目的**：在服务器端评估功能开关，减少客户端逻辑和敏感数据传输。

**工作流程**：
```
1. 客户端发送用户属性到 GrowthBook API
2. 服务器评估所有功能开关
3. 返回预计算的值
4. 客户端缓存结果
```

**优势**：
- 保护实验分配算法
- 减少客户端 SDK 复杂度
- 支持更复杂的定位规则

### 3. 配置覆盖系统

**目的**：支持开发测试和紧急配置变更。

**覆盖层级**（优先级从高到低）：
1. **环境变量覆盖** (`CLAUDE_INTERNAL_FC_OVERRIDES`): ant 专用，JSON 格式
2. **配置文件覆盖** (`growthBookOverrides`): ant 专用，通过 `/config` 命令设置
3. **远程评估值**: 正常流程
4. **磁盘缓存**: 上次成功的远程评估结果
5. **默认值**: 代码中硬编码

**环境变量覆盖示例**：
```bash
CLAUDE_INTERNAL_FC_OVERRIDES='{"my_feature": true, "my_config": {"key": "val"}}'
```

### 4. 实验曝光日志

**目的**：记录用户被分配到实验组，用于实验分析。

**去重机制**：
```typescript
const loggedExposures = new Set<string>()
```
- 每个会话每个功能只记录一次
- 防止热路径（如渲染循环）产生重复日志

**延迟记录**：
- 初始化前访问的功能加入 `pendingExposures`
- 初始化完成后统一记录

### 5. 定期刷新

**目的**：长期运行的会话保持配置新鲜。

**刷新间隔**：
- 外部用户：6 小时
- ant 用户：20 分钟

**刷新机制**：
- Light refresh：不重建客户端，仅获取新值
- Auth change：重建客户端（auth headers 不能更新）

### 6. 动态配置

**目的**：提供类似 Statsig Dynamic Config 的 API。

**映射关系**：
```typescript
// GrowthBook 中 dynamic config 就是 object 类型的 feature
getDynamicConfig_BLOCKS_ON_INIT(feature) === getFeatureValue_DEPRECATED(feature)
getDynamicConfig_CACHED_MAY_BE_STALE(feature) === getFeatureValue_CACHED_MAY_BE_STALE(feature)
```

## 具体技术实现

### 数据结构

**用户属性** (`GrowthBookUserAttributes`)：
```typescript
{
  id: string              // 设备 ID
  sessionId: string
  deviceID: string        // 同 id
  platform: 'win32' | 'darwin' | 'linux'
  apiBaseUrlHost?: string // 企业代理部署标识
  organizationUUID?: string
  accountUUID?: string
  userType?: string
  subscriptionType?: string
  rateLimitTier?: string
  firstTokenTime?: number
  email?: string
  appVersion?: string
  github?: GitHubActionsMetadata
}
```

**实验数据存储**：
```typescript
type StoredExperimentData = {
  experimentId: string
  variationId: number
  inExperiment?: boolean
  hashAttribute?: string
  hashValue?: string
}
// Map<featureName, StoredExperimentData>
```

### 核心流程

#### 初始化流程 (`initializeGrowthBook`)

```
1. 检查 GrowthBook 是否启用
2. 获取用户属性
3. 检查认证状态
4. 创建 GrowthBook 客户端
   - apiHost: https://api.anthropic.com
   - clientKey: 从 keys.ts 获取
   - remoteEval: true
   - cacheKeyAttributes: ['id', 'organizationUUID']
5. 调用 client.init()
6. processRemoteEvalPayload() 处理响应
7. 记录待处理的曝光
8. 同步到磁盘
9. 通知订阅者
10. 设置定期刷新
```

#### 远程评估处理 (`processRemoteEvalPayload`)

```
1. 获取 payload
2. 检查 features 非空
3. 转换格式（value → defaultValue）
4. 提取实验数据
5. 缓存评估值到 remoteEvalFeatureValues
6. 重新设置 payload 到客户端
```

#### 功能值获取 (`getFeatureValueInternal`)

```
1. 检查环境变量覆盖
2. 检查配置文件覆盖
3. 检查是否启用
4. 等待初始化完成（阻塞 API）
5. 检查内存缓存
6. 调用 SDK getFeatureValue
7. 记录曝光（如果请求）
```

### 刷新订阅机制

```typescript
type GrowthBookRefreshListener = () => void | Promise<void>
const refreshed = createSignal()

export function onGrowthBookRefresh(listener: GrowthBookRefreshListener): () => void
```

**使用场景**：
- `firstPartyEventLogger.ts`: 配置变化时重建批处理器
- `useMainLoopModel.ts`: 模型覆盖变化时更新
- `useSkillsChange.ts`: 技能配置变化时更新

**Catch-up 机制**：
- 如果注册时初始化已完成，下次微任务触发回调
- 处理初始化与 React mount 的竞态

## 关键代码路径与文件引用

### 被调用方

| 文件 | 调用 | 说明 |
|------|------|------|
| `src/services/analytics/sink.ts` | `checkStatsigFeatureGate_CACHED_MAY_BE_STALE` | Datadog gate 检查 |
| `src/services/analytics/firstPartyEventLogger.ts` | `getDynamicConfig_CACHED_MAY_BE_STALE` | 批处理配置 |
| `src/services/analytics/sinkKillswitch.ts` | `getDynamicConfig_CACHED_MAY_BE_STALE` | killswitch 检查 |
| `src/main.tsx` | `initializeGrowthBook`, `onGrowthBookRefresh` | 启动初始化 |
| `src/utils/auth.ts` | `refreshGrowthBookAfterAuthChange` | 登录/登出刷新 |
| `src/hooks/useDynamicConfig.ts` | `getDynamicConfig_CACHED_MAY_BE_STALE` | React hook |
| `src/hooks/useMainLoopModel.ts` | `onGrowthBookRefresh` | 模型更新 |
| `src/hooks/useSkillsChange.ts` | `onGrowthBookRefresh` | 技能更新 |

### 依赖文件

| 文件 | 提供功能 |
|------|----------|
| `src/bootstrap/state.ts` | 信任状态检查 |
| `src/constants/keys.ts` | `getGrowthBookClientKey()` |
| `src/utils/config.ts` | 全局配置读写 |
| `src/utils/debug.ts` | 调试日志 |
| `src/utils/http.ts` | 认证头获取 |
| `src/utils/log.ts` | 错误日志 |
| `src/utils/signal.ts` | 信号创建 |
| `src/utils/slowOperations.ts` | JSON 序列化 |
| `src/utils/user.ts` | 用户数据获取 |
| `src/services/analytics/firstPartyEventLogger.ts` | 实验曝光日志 |

### 配置文件

| 路径 | 用途 |
|------|------|
| `~/.claude.json` | `cachedGrowthBookFeatures`, `growthBookOverrides` |

## 依赖与外部交互

### 外部服务

**GrowthBook API**:
- 端点：`https://api.anthropic.com`（生产）
- 路径：`/api/features/eval`（远程评估）
- 认证：可选（OAuth Bearer token）
- 请求头：`x-service-name: claude-code`

**SDK 配置**：
```typescript
new GrowthBook({
  apiHost: 'https://api.anthropic.com',
  clientKey: 'sdk_...',
  remoteEval: true,
  cacheKeyAttributes: ['id', 'organizationUUID'],
})
```

### 第三方库

- `@growthbook/growthbook`: GrowthBook SDK
- `lodash-es`: `isEqual`, `memoize`

### 环境变量

| 变量 | 用途 |
|------|------|
| `USER_TYPE` | ant 用户启用调试日志和短刷新间隔 |
| `CLAUDE_INTERNAL_FC_OVERRIDES` | ant 功能覆盖（JSON） |
| `CLAUDE_CODE_GB_BASE_URL` | ant 自定义 API 主机 |
| `ANTHROPIC_BASE_URL` | 确定 staging/prod |

### 磁盘缓存格式

```json
{
  "cachedGrowthBookFeatures": {
    "feature_name": "value",
    "tengu_1p_event_batch_config": {
      "scheduledDelayMillis": 10000,
      "maxExportBatchSize": 200
    }
  },
  "growthBookOverrides": {
    "ant_only_feature": true
  }
}
```

## 风险、边界与改进建议

### 风险点

1. **API 格式不匹配**
   - 问题：API 返回 `value`，SDK 期望 `defaultValue`
   - 现状：`processRemoteEvalPayload` 中进行转换
   - 风险：API 变更时转换逻辑失效
   - 建议：与后端协调统一格式

2. **缓存污染**
   - 场景：空或损坏的 payload 被写入磁盘
   - 缓解：`processRemoteEvalPayload` 检查 `Object.keys(features).length === 0`
   - 风险：部分损坏仍可能通过

3. **刷新竞态**
   - 场景：`refreshGrowthBookFeatures` 与 `refreshGrowthBookAfterAuthChange` 并发
   - 缓解：客户端引用检查（`growthBookClient !== client`）
   - 风险：复杂场景下仍可能处理过期数据

4. **订阅者泄漏**
   - 场景：React 组件卸载时未取消订阅
   - 缓解：`onGrowthBookRefresh` 返回取消函数
   - 建议：提供 React hook 包装

5. **初始化阻塞**
   - 场景：`_BLOCKS_ON_INIT` API 在网络慢时阻塞启动
   - 缓解：推荐使用 `_CACHED_MAY_BE_STALE`
   - 风险：旧代码或误用仍可能影响启动

### 边界情况

1. **无网络初始化**
   - 使用磁盘缓存值
   - 初始化 Promise 仍然 resolve
   - 定期刷新继续尝试

2. **认证变化**
   - 需要完全重建客户端（auth headers 不可更新）
   - `refreshGrowthBookAfterAuthChange` 处理
   - 期间使用旧值或默认值

3. **组织切换**
   - `cacheKeyAttributes` 包含 `organizationUUID`
   - 自动触发重新获取

4. **多会话并发**
   - 每个进程独立客户端
   - 磁盘缓存共享（可能冲突）
   - 写入时整体替换，非合并

5. **功能删除**
   - 服务器删除功能后，磁盘缓存仍有旧值
   - `processRemoteEvalPayload` 先 clear 再 rebuild
   - 空 payload 不会清除缓存（有非空检查）

### 改进建议

1. **API 格式标准化**
   - 与后端统一使用 `defaultValue`
   - 移除转换逻辑

2. **增量缓存更新**
   ```typescript
   // 当前：整体替换
   // 建议：合并策略，保留未变更的功能
   function mergeFeatures(old: Features, new: Features): Features
   ```

3. **刷新冲突解决**
   ```typescript
   // 添加刷新令牌避免竞态
   let currentRefreshId = 0
   async function refresh(): Promise<void> {
     const refreshId = ++currentRefreshId
     const result = await fetch()
     if (refreshId !== currentRefreshId) return // 过期
   }
   ```

4. **React Hook**
   ```typescript
   function useGrowthBookFeature<T>(feature: string, defaultValue: T): T {
     const [value, setValue] = useState(() => 
       getFeatureValue_CACHED_MAY_BE_STALE(feature, defaultValue)
     )
     useEffect(() => {
       return onGrowthBookRefresh(() => {
         setValue(getFeatureValue_CACHED_MAY_BE_STALE(feature, defaultValue))
       })
     }, [feature])
     return value
   }
   ```

5. **功能依赖图**
   ```typescript
   // 支持功能间的依赖关系
   interface FeatureDependency {
     feature: string
     dependsOn: string[]
     fallback: unknown
   }
   ```

6. **A/B 测试分析**
   ```typescript
   // 导出实验分配统计
   export function getExperimentStats(): {
     totalExposures: number
     experiments: Record<string, { variations: number[] }>
   }
   ```

7. **配置验证**
   ```typescript
   // 运行时验证配置类型
   function validateFeatureValue<T>(
     feature: string, 
     value: unknown, 
     schema: z.ZodType<T>
   ): T
   ```
