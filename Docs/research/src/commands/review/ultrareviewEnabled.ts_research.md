# ultrareviewEnabled.ts 深度研究文档

> 文件路径：`src/commands/review/ultrareviewEnabled.ts`  
> 文件大小：526 bytes（14 行）  
> 研究日期：2026-04-01  
> 执行器：kimi (k2p5)

---

## 一、场景与职责

### 1.1 模块定位

`ultrareviewEnabled.ts` 是 Claude Code CLI 中 `/ultrareview` 功能的**运行时开关门控**。它通过读取 GrowthBook 功能标志（feature flag）决定当前用户是否可以看到并使用 `/ultrareview` 命令。该模块是整个 ultrareview 功能的"总闸"，在命令注册阶段即被调用。

### 1.2 核心职责

| 职责 | 说明 |
|------|------|
| **功能开关判断** | 读取 GrowthBook 的 `tengu_review_bughunter_config` 配置，检查 `enabled` 字段 |
| **命令可见性控制** | 返回 `false` 时，`/ultrareview` 从命令列表中完全隐藏，用户无法看到或触发 |
| **非阻塞快速判断** | 使用 `getFeatureValue_CACHED_MAY_BE_STALE`，避免阻塞启动或渲染 |

### 1.3 调用场景

该模块在以下两处被调用：

1. **命令注册阶段**：`src/commands/review.ts` 中 `ultrareview.isEnabled`
2. **输入提示阶段**：`src/components/PromptInput/PromptInput.tsx` 中 `ultrareviewTriggers` 的计算

---

## 二、功能点目的

### 2.1 为什么需要独立模块

- **单一职责**：将功能开关逻辑从命令定义和 UI 组件中抽离，便于统一修改
- **懒加载兼容**：`review.ts` 中 `ultrareview` 命令是懒加载的，但 `isEnabled` 需要在命令注册时立即判断，不能等待 `ultrareviewCommand.tsx` 加载
- **多处复用**：PromptInput 的彩虹高亮和通知逻辑也需要知道功能是否开启

### 2.2 功能矩阵

| 导出 | 签名 | 目的 |
|------|------|------|
| `isUltrareviewEnabled` | `() => boolean` | 判断当前用户是否可以使用 `/ultrareview` |

---

## 三、具体技术实现

### 3.1 源码实现

```typescript
// src/commands/review/ultrareviewEnabled.ts
import { getFeatureValue_CACHED_MAY_BE_STALE } from '../../services/analytics/growthbook.js'

/**
 * Runtime gate for /ultrareview. GB config's `enabled` field controls
 * visibility — isEnabled() on the command filters it from getCommands()
 * when false, so ungated users don't see the command at all.
 */
export function isUltrareviewEnabled(): boolean {
  const cfg = getFeatureValue_CACHED_MAY_BE_STALE<Record<
    string,
    unknown
  > | null>('tengu_review_bughunter_config', null)
  return cfg?.enabled === true
}
```

### 3.2 关键技术点

#### 3.2.1 `getFeatureValue_CACHED_MAY_BE_STALE`

- 来源：`src/services/analytics/growthbook.ts`（line 734-775）
- 特点：
  - **非阻塞**：直接从内存缓存或磁盘缓存读取，不等待网络请求
  - **快速**：适用于渲染热路径（如 `isEnabled()` 在每次 `getCommands()` 时调用）
  - **可能陈旧**：返回的值可能是上一次成功初始化时的缓存，新用户可能需要等待 GrowthBook 初始化完成后才能看到功能

#### 3.2.2 严格的真值判断

```typescript
return cfg?.enabled === true
```

- 使用 `=== true` 而非简单的 `!!cfg?.enabled`
- 原因：
  - 防止 `enabled: 'true'`（字符串）被误判为开启
  - 防止 `enabled: 1`（数字）被误判为开启
  - 确保只有显式的布尔值 `true` 才能开启功能

#### 3.2.3 配置结构预期

GrowthBook 中 `tengu_review_bughunter_config` 的预期结构：

```typescript
{
  enabled: boolean,
  fleet_size?: number,
  max_duration_minutes?: number,
  agent_timeout_seconds?: number,
  total_wallclock_minutes?: number,
  // ... 其他 bughunter 参数
}
```

`ultrareviewEnabled.ts` 只关心 `enabled` 字段，其他参数由 `reviewRemote.ts` 读取和使用。

---

## 四、关键代码路径与文件引用

### 4.1 调用链路

```
GrowthBook 远程配置 / 磁盘缓存
    │
    ▼
src/services/analytics/growthbook.ts
    │ getFeatureValue_CACHED_MAY_BE_STALE('tengu_review_bughunter_config', null)
    ▼
src/commands/review/ultrareviewEnabled.ts
    │ isUltrareviewEnabled()
    ├──→ src/commands/review.ts
    │    └── ultrareview: Command { isEnabled: () => isUltrareviewEnabled() }
    │        └── 控制 /ultrareview 是否在命令列表中可见
    │
    └──→ src/components/PromptInput/PromptInput.tsx
         └── ultrareviewTriggers = isUltrareviewEnabled() ? findUltrareviewTriggerPositions(displayedValue) : []
             └── 控制输入框中 "ultrareview" 关键词是否触发彩虹高亮和通知
```

### 4.2 核心文件清单

| 文件路径 | 行数 | 职责 |
|----------|------|------|
| `src/commands/review/ultrareviewEnabled.ts` | 14 | **本文件**：功能开关判断 |
| `src/commands/review.ts` | 57 | 命令注册，使用 `isUltrareviewEnabled()` |
| `src/components/PromptInput/PromptInput.tsx` | 1000+ | 输入框组件，使用 `isUltrareviewEnabled()` 控制关键词高亮 |
| `src/services/analytics/growthbook.ts` | 1000+ | GrowthBook 客户端，提供 `getFeatureValue_CACHED_MAY_BE_STALE` |
| `src/utils/ultraplan/keyword.ts` | 127 | 提供 `findUltrareviewTriggerPositions()` |

### 4.3 代码位置速查

| 元素 | 行号 |
|------|------|
| `getFeatureValue_CACHED_MAY_BE_STALE` 导入 | 1 |
| `isUltrareviewEnabled` 函数导出 | 8-14 |
| `tengu_review_bughunter_config` flag 读取 | 9-12 |
| `enabled` 判断 | 13 |

---

## 五、依赖与外部交互

### 5.1 内部依赖

```
ultrareviewEnabled.ts
└── ../../services/analytics/growthbook.js
    └── getFeatureValue_CACHED_MAY_BE_STALE
```

### 5.2 外部依赖

| 依赖 | 用途 | 说明 |
|------|------|------|
| GrowthBook | 功能标志管理 | `tengu_review_bughunter_config` 由 GrowthBook 远程配置或磁盘缓存提供 |

### 5.3 无直接网络交互

`getFeatureValue_CACHED_MAY_BE_STALE` 是纯本地读取：
1. 先检查环境变量覆盖（`CLAUDE_INTERNAL_FC_OVERRIDES`）
2. 再检查内存缓存（`remoteEvalFeatureValues` Map）
3. 最后回退到磁盘缓存（`~/.claude.json` 中的 `cachedGrowthBookFeatures`）

因此 `isUltrareviewEnabled()` 的调用是**同步、非阻塞、零网络**的。

---

## 六、风险、边界与改进建议

### 6.1 已知风险

#### 6.1.1 缓存陈旧风险

- **风险**：新用户首次启动 CLI 时，GrowthBook 可能尚未完成初始化，`cachedGrowthBookFeatures` 中可能没有 `tengu_review_bughunter_config`
- **行为**：此时 `getFeatureValue_CACHED_MAY_BE_STALE` 返回默认值 `null`，`isUltrareviewEnabled()` 返回 `false`
- **影响**：用户需要等待 GrowthBook 初始化完成（通常几秒到几十秒）或重启 CLI 后才能看到 `/ultrareview` 命令
- **缓解**：这是 `CACHED_MAY_BE_STALE` 的设计权衡，优先保证启动速度和渲染流畅性

#### 6.1.2 配置结构变更风险

- **风险**：如果 GrowthBook 后端将 `enabled` 字段重命名或改为嵌套结构（如 `features.ultrareview.enabled`），本模块会立即返回 `false`
- **影响**：所有用户都无法看到 `/ultrareview` 命令
- **缓解**：`=== true` 的严格判断虽然安全，但也意味着任何配置格式变更都会导致功能关闭，属于"fail-safe"设计

#### 6.1.3 与 `reviewRemote.ts` 的重复读取

- `reviewRemote.ts` 也读取同一 GrowthBook flag：
  ```typescript
  const raw = getFeatureValue_CACHED_MAY_BE_STALE<Record<string, unknown> | null>(
    'tengu_review_bughunter_config',
    null
  )
  ```
- 虽然 GrowthBook 缓存读取很快，但两处读取增加了维护成本
- **建议**：将配置读取和解析集中到一个模块，导出 `getBughunterConfig()` 供两者使用

### 6.2 边界情况

| 场景 | 行为 |
|------|------|
| GrowthBook 未初始化 | `cfg === null` → 返回 `false` |
| `cfg.enabled` 为 `false` | 返回 `false` |
| `cfg.enabled` 为 `undefined` | `cfg?.enabled === true` 为 `false` |
| `cfg.enabled` 为字符串 `"true"` | `=== true` 为 `false` |
| `cfg.enabled` 为数字 `1` | `=== true` 为 `false` |
| 环境变量 `CLAUDE_INTERNAL_FC_OVERRIDES` 设置了覆盖值 | 优先返回覆盖值（ant 构建专用） |

### 6.3 改进建议

#### 6.3.1 增加日志记录

当前模块完全静默，建议增加调试日志：

```typescript
export function isUltrareviewEnabled(): boolean {
  const cfg = getFeatureValue_CACHED_MAY_BE_STALE<Record<string, unknown> | null>(
    'tengu_review_bughunter_config',
    null
  )
  const enabled = cfg?.enabled === true
  if (process.env.USER_TYPE === 'ant') {
    logForDebugging(`isUltrareviewEnabled: ${enabled} (cfg=${JSON.stringify(cfg)})`)
  }
  return enabled
}
```

#### 6.3.2 统一配置读取

建议新建 `src/commands/review/bughunterConfig.ts`：

```typescript
export function getBughunterConfig(): Record<string, unknown> | null {
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_review_bughunter_config', null)
}

export function isUltrareviewEnabled(): boolean {
  return getBughunterConfig()?.enabled === true
}
```

然后 `reviewRemote.ts` 改为：

```typescript
import { getBughunterConfig } from './bughunterConfig.js'
const raw = getBughunterConfig()
```

#### 6.3.3 考虑增加渐进式暴露

对于缓存未命中的新用户，可以考虑：
- 在 GrowthBook 初始化完成后触发一次命令列表刷新
- 或在 `onGrowthBookRefresh` 中订阅配置变化，动态更新命令可见性

当前 `src/commands/review.ts` 中的 `isEnabled` 是一个普通函数，没有订阅机制：

```typescript
isEnabled: () => isUltrareviewEnabled(),
```

如果 GrowthBook 在启动后异步刷新，`getCommands()` 不会自动重新评估 `isEnabled`。用户可能需要发送一条消息或触发其他重渲染才能看到新出现的 `/ultrareview` 命令。

#### 6.3.4 类型安全增强

当前使用 `Record<string, unknown>` 读取配置，建议定义更严格的类型：

```typescript
export type BughunterConfig = {
  enabled: boolean
  fleet_size?: number
  max_duration_minutes?: number
  agent_timeout_seconds?: number
  total_wallclock_minutes?: number
}

export function getBughunterConfig(): BughunterConfig | null {
  const cfg = getFeatureValue_CACHED_MAY_BE_STALE<unknown>(
    'tengu_review_bughunter_config',
    null
  )
  // 运行时验证...
  return cfg as BughunterConfig | null
}
```

---

## 附录：相关配置速查

### GrowthBook Flag

| 属性 | 说明 |
|------|------|
| Flag Key | `tengu_review_bughunter_config` |
| 类型 | JSON Object |
| 默认值 | `{ enabled: false }` |
| 读取方式 | `getFeatureValue_CACHED_MAY_BE_STALE` |
| 用途 | 控制 `/ultrareview` 功能开关及 bughunter 运行参数 |

### 环境变量覆盖（仅 ant 构建）

```bash
CLAUDE_INTERNAL_FC_OVERRIDES='{"tengu_review_bughunter_config": {"enabled": true}}'
```
