# channelAllowlist.ts 深度研究文档

## 场景与职责

`channelAllowlist.ts` 是 Claude Code MCP 服务中的**通道插件白名单管理模块**，负责控制哪些插件可以通过 `--channels` 参数注册为通道服务器。该模块是 MCP Channel 功能的安全门禁系统。

### 核心职责

1. **白名单校验**：验证插件是否被允许作为通道服务器注册
2. **功能开关控制**：通过 GrowthBook 特性标志控制 Channels 功能的整体启用/禁用
3. **插件粒度控制**：以插件（plugin）为单位进行授权，而非单个服务器

### 业务背景

Channels 是 Claude Code 的一项功能，允许插件提供可以处理特定类型消息的"通道服务器"。出于安全考虑，不是所有插件都能随意注册通道服务器——需要经过 Anthropic 审核并加入白名单。

**设计决策**（来自代码注释）：
- 使用**插件级粒度**：一个插件被授权后，其所有通道服务器都被允许
- 不使用服务器级粒度：避免无害重构导致授权失效，且恶意插件添加第二个服务器时已经意味着插件本身被攻破

## 功能点目的

### 1. 白名单获取 (`getChannelAllowlist`)

**目的**：从 GrowthBook 获取当前批准的通道插件列表。

**实现**：
- 读取 GrowthBook 特性 `tengu_harbor_ledger`
- 返回结构：`{ marketplace: string, plugin: string }[]`
- 使用 Zod 进行运行时类型验证
- 解析失败返回空数组（安全降级）

**数据结构**：
```typescript
type ChannelAllowlistEntry = {
  marketplace: string  // 插件市场名称，如 "anthropic"
  plugin: string       // 插件名称
}
```

### 2. Channels 功能开关 (`isChannelsEnabled`)

**目的**：控制 Channels 功能的整体可用性。

**实现**：
- 读取 GrowthBook 特性 `tengu_harbor`
- 默认 `false`（功能默认关闭）
- GrowthBook 5 分钟刷新周期

**使用场景**：
- 功能发布控制 (Feature Rollout)
- 紧急关闭 (Kill Switch)
- 分阶段发布

### 3. 白名单校验 (`isChannelAllowlisted`)

**目的**：判断特定插件是否在白名单中。

**参数**：`pluginSource` - 插件标识符（如 `slack@anthropic`）

**校验逻辑**：
1. 空值检查：`undefined` → `false`
2. 解析插件标识符：提取 `name` 和 `marketplace`
3. 无市场信息：无 `@` 或解析失败 → `false`
4. 白名单匹配：检查 `{marketplace, plugin}` 组合是否在列表中

**使用场景**：
- UI 预过滤：在 IDE 中只显示"启用通道？"选项给白名单内的服务器
- 通道注册门控：`channel_enable` 运行时检查

**注意**：这不是安全边界，真正的安全检查在 `channel_enable` 中执行。此函数用于 UI 预过滤，避免向用户展示无法使用的选项。

## 具体技术实现

### 类型定义

```typescript
// 白名单条目
export type ChannelAllowlistEntry = {
  marketplace: string
  plugin: string
}
```

### Zod Schema

```typescript
const ChannelAllowlistSchema = lazySchema(() =>
  z.array(
    z.object({
      marketplace: z.string(),
      plugin: z.string(),
    }),
  ),
)
```

使用 `lazySchema` 延迟初始化，避免模块加载时的循环依赖问题。

### 核心函数实现

#### getChannelAllowlist

```typescript
export function getChannelAllowlist(): ChannelAllowlistEntry[] {
  const raw = getFeatureValue_CACHED_MAY_BE_STALE<unknown>(
    'tengu_harbor_ledger',
    [],
  )
  const parsed = ChannelAllowlistSchema().safeParse(raw)
  return parsed.success ? parsed.data : []
}
```

**特点**：
- 使用 `getFeatureValue_CACHED_MAY_BE_STALE` 获取 GrowthBook 特性值
- 失败安全 (fail-safe)：解析失败返回空数组，意味着没有插件被授权
- 缓存值可能过期（函数名提示）

#### isChannelsEnabled

```typescript
export function isChannelsEnabled(): boolean {
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_harbor', false)
}
```

**特点**：
- 默认 `false`：新安装或缓存缺失时功能关闭
- 使用布尔类型的特性标志

#### isChannelAllowlisted

```typescript
export function isChannelAllowlisted(pluginSource: string | undefined): boolean {
  if (!pluginSource) return false
  const { name, marketplace } = parsePluginIdentifier(pluginSource)
  if (!marketplace) return false
  return getChannelAllowlist().some(
    e => e.plugin === name && e.marketplace === marketplace
  )
}
```

**特点**：
- 防御性编程：空值和无效格式都返回 `false`
- 依赖 `parsePluginIdentifier` 解析插件标识符
- 严格匹配：插件名和市场都必须匹配

### 插件标识符解析

依赖 `../../utils/plugins/pluginIdentifier.js` 的 `parsePluginIdentifier`：

```typescript
// 输入: "slack@anthropic"
// 输出: { name: "slack", marketplace: "anthropic" }

// 输入: "builtin-tool" (无 @)
// 输出: { name: "builtin-tool", marketplace: undefined }
```

**解析规则**：
- 只使用第一个 `@` 作为分隔符
- 多个 `@` 符号时，第二个及之后被忽略
- 市场名不应包含 `@`

## 关键代码路径与文件引用

### 文件依赖关系

```
channelAllowlist.ts
├── zod/v4  (Schema 验证)
├── ../../utils/lazySchema.js  (延迟 Schema 初始化)
├── ../../utils/plugins/pluginIdentifier.js  (插件标识符解析)
└── ../analytics/growthbook.js  (GrowthBook 特性获取)
```

### 调用关系

**被调用方**：
```
channelPermissions.ts (推测)
  → isChannelAllowlisted()  // 通道注册门控

UI 组件 (推测)
  → isChannelAllowlisted()  // 预过滤显示
  → isChannelsEnabled()     // 功能开关检查
```

**依赖的 GrowthBook 特性**：

| 特性键 | 类型 | 默认值 | 用途 |
|--------|------|--------|------|
| `tengu_harbor_ledger` | `ChannelAllowlistEntry[]` | `[]` | 白名单数据 |
| `tengu_harbor` | `boolean` | `false` | 功能总开关 |

### 命名约定

- `tengu_harbor`：Channels 功能的代号（Harbor 港口，比喻通道入口）
- `tengu_harbor_ledger`：白名单账本

## 依赖与外部交互

### 外部依赖

| 依赖 | 用途 |
|------|------|
| `zod/v4` | 运行时类型验证 |
| `lodash-es` | 可能通过 GrowthBook 间接使用 |

### 内部依赖

| 模块 | 用途 |
|------|------|
| `../../utils/lazySchema.js` | 延迟初始化 Zod Schema，避免循环依赖 |
| `../../utils/plugins/pluginIdentifier.js` | 解析 `plugin@marketplace` 格式 |
| `../analytics/growthbook.js` | 获取远程配置特性值 |

### GrowthBook 集成

```typescript
// growthbook.ts 中的相关函数
export function getFeatureValue_CACHED_MAY_BE_STALE<T>(
  key: string,
  defaultValue: T
): T
```

**特点**：
- 值可能被缓存，不一定是最新
- 适用于热路径（如渲染循环）
- GrowthBook 后台每 5 分钟刷新

## 风险、边界与改进建议

### 已知风险

1. **缓存延迟**
   - 白名单更新后，客户端最多 5 分钟后才能感知
   - 新授权的插件可能暂时无法使用
   - 已撤销的插件可能暂时仍能注册通道

2. **空列表安全降级**
   - GrowthBook 不可用或返回无效数据时返回空数组
   - 所有通道功能将被禁用
   - 这是安全的设计，但可能影响用户体验

3. **插件标识符解析歧义**
   - `plugin@marketplace` 格式中，插件名本身不能包含 `@`
   - 当前实现只分割第一个 `@`，但文档未明确约束

### 边界条件

1. **插件来源格式**
   - `undefined` → `false`（非插件服务器）
   - `builtin-tool`（无 `@`）→ `false`（内置工具）
   - `slack@anthropic` → 正常检查
   - `slack@anthropic@extra` → 忽略 `@extra`，按 `slack@anthropic` 处理

2. **白名单条目匹配**
   - 大小写敏感：`Slack` ≠ `slack`
   - 市场名和插件名都必须完全匹配

3. **与 `--dangerously-load-development-channels` 的交互**
   - 该标志可以绕过白名单检查
   - 用于开发和测试场景
   - 代码注释提到但未在此模块中实现

### 改进建议

1. **缓存刷新机制**
   - 当前：被动等待 GrowthBook 5 分钟刷新
   - 建议：添加手动刷新 API，供紧急场景使用
   - 例如：`refreshChannelAllowlist()`

2. **更灵活的匹配规则**
   - 当前：精确匹配
   - 建议：支持通配符或前缀匹配
   - 例如：`*.official@anthropic` 匹配所有官方插件

3. **白名单变更通知**
   - 当前：无通知机制
   - 建议：添加事件系统，白名单变更时通知订阅者
   - UI 可以实时更新可用通道列表

4. **审计日志**
   - 当前：无日志记录
   - 建议：记录白名单检查事件（调试/审计用途）
   - 例如：`logMCPDebug('channel', 'Allowlist check for ${pluginSource}: ${result}')`

5. **缓存一致性优化**
   - 当前：`CACHED_MAY_BE_STALE` 提示开发者注意
   - 建议：添加缓存时间戳，超过阈值强制刷新
   - 或者提供 `getChannelAllowlist_FRESH` 变体

6. **错误监控**
   - 当前：解析失败静默返回空数组
   - 建议：添加错误上报，了解配置问题的频率
   - 例如：`logEvent('tengu_channel_allowlist_parse_error', { raw })`

### 测试建议

1. **单元测试场景**：
   - `getChannelAllowlist` 返回有效/无效/空数据
   - `isChannelAllowlisted` 各种输入格式
   - 大小写敏感性测试
   - 特殊字符处理

2. **集成测试场景**：
   - GrowthBook 特性变更后的行为
   - 缓存过期后的刷新
   - 网络中断时的降级

3. **安全测试**：
   - 尝试绕过白名单的各种方式
   - 畸形插件标识符处理
   - GrowthBook 响应注入攻击
