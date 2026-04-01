# cronJitterConfig.ts 深度研究文档

## 场景与职责

`cronJitterConfig.ts` 是 Claude Code 定时任务系统的 **抖动配置管理模块**，负责从 GrowthBook（特性标志服务）获取 cron 抖动配置，用于在分布式环境中分散定时任务的执行时间，避免"惊群效应"（thundering herd）。

### 核心职责

1. **远程配置获取**：从 GrowthBook 读取 `tengu_kairos_cron_config` 配置
2. **配置验证**：使用 Zod 模式验证配置值，防止无效配置导致系统异常
3. **默认值回退**：在配置缺失或无效时提供安全的默认值
4. **缓存刷新**：定期刷新配置以支持运营调整

### 使用场景

- **REPL 模式**：`useScheduledTasks.ts` 钩子传递 `getCronJitterConfig` 给调度器
- **负载分散**：当大量用户设置相同时间（如每小时 :00）的定时任务时，分散执行时间
- **运营调控**：在负载高峰时通过 GrowthBook 调整抖动参数，无需重启客户端

---

## 功能点目的

### 1. 配置获取与验证 (`getCronJitterConfig`)

**目的**：安全地获取远程抖动配置，确保配置值在合理范围内。

**配置项说明**：

| 配置项 | 类型 | 默认值 | 范围 | 用途 |
|--------|------|--------|------|------|
| `recurringFrac` | number | 0.1 | 0-1 | 周期性任务延迟比例（相对间隔） |
| `recurringCapMs` | number | 15min | 0-30min | 周期性任务最大延迟 |
| `oneShotMaxMs` | number | 90s | 0-30min | 一次性任务最大提前时间 |
| `oneShotFloorMs` | number | 0 | 0-30min | 一次性任务最小提前时间 |
| `oneShotMinuteMod` | number | 30 | 1-60 | 一次性任务抖动触发的分钟模数 |
| `recurringMaxAgeMs` | number | 7days | 0-30days | 周期性任务自动过期时间 |

**验证规则**：
- `oneShotFloorMs <= oneShotMaxMs`（防止范围反转）
- 所有数值在预定义的上下界内
- 无效配置整体回退到默认值（而非部分使用）

### 2. 缓存刷新机制

**目的**：在保持性能的同时支持配置的动态更新。

**实现细节**：
- 使用 `getFeatureValue_CACHED_WITH_REFRESH` 进行缓存
- 刷新间隔：60 秒（`JITTER_CONFIG_REFRESH_MS`）
- 底层是同步缓存读取，刷新在后台进行

---

## 具体技术实现

### 核心数据结构

```typescript
// 从 cronTasks.ts 导入的类型
type CronJitterConfig = {
  recurringFrac: number        // 周期性任务延迟比例
  recurringCapMs: number       // 周期性任务延迟上限
  oneShotMaxMs: number         // 一次性任务最大提前
  oneShotFloorMs: number       // 一次性任务最小提前
  oneShotMinuteMod: number     // 分钟模数（默认 30 → :00/:30）
  recurringMaxAgeMs: number    // 周期性任务最大存活时间
}

// 默认值
const DEFAULT_CRON_JITTER_CONFIG: CronJitterConfig = {
  recurringFrac: 0.1,                    // 10% 的间隔时间
  recurringCapMs: 15 * 60 * 1000,        // 最多 15 分钟
  oneShotMaxMs: 90 * 1000,               // 最多 90 秒提前
  oneShotFloorMs: 0,                     // 最少 0 秒（可能正好在目标时间）
  oneShotMinuteMod: 30,                  // 只在 :00 和 :30 触发抖动
  recurringMaxAgeMs: 7 * 24 * 60 * 60 * 1000,  // 7 天后过期
}
```

### Zod 验证模式

```typescript
const cronJitterConfigSchema = z.object({
  recurringFrac: z.number().min(0).max(1),
  recurringCapMs: z.number().int().min(0).max(HALF_HOUR_MS),
  oneShotMaxMs: z.number().int().min(0).max(HALF_HOUR_MS),
  oneShotFloorMs: z.number().int().min(0).max(HALF_HOUR_MS),
  oneShotMinuteMod: z.number().int().min(1).max(60),
  recurringMaxAgeMs: z.number().int().min(0).max(THIRTY_DAYS_MS).default(DEFAULT),
}).refine(c => c.oneShotFloorMs <= c.oneShotMaxMs)
```

### 配置获取流程

```
getCronJitterConfig()
    ↓
getFeatureValue_CACHED_WITH_REFRESH('tengu_kairos_cron_config', DEFAULT, 60s)
    ↓
[缓存命中] → 返回缓存值
[缓存过期] → 后台刷新 → 返回旧值
    ↓
cronJitterConfigSchema().safeParse(raw)
    ↓
[验证成功] → 返回解析后的配置
[验证失败] → 返回 DEFAULT_CRON_JITTER_CONFIG
```

---

## 关键代码路径与文件引用

### 核心导出函数

| 函数 | 行号 | 用途 |
|------|------|------|
| `getCronJitterConfig` | 67-75 | 获取并验证抖动配置 |

### 常量定义

| 常量 | 行号 | 值 | 说明 |
|------|------|-----|------|
| `JITTER_CONFIG_REFRESH_MS` | 24 | 60000 | 配置刷新间隔（60秒） |
| `HALF_HOUR_MS` | 35 | 1800000 | 30分钟毫秒值 |
| `THIRTY_DAYS_MS` | 36 | 2592000000 | 30天毫秒值 |

### 依赖文件

```
cronJitterConfig.ts
├── 被调用方（上游）
│   └── src/hooks/useScheduledTasks.ts   # REPL 调度器钩子
├── 被依赖模块（下游）
│   ├── src/utils/cronTasks.ts           # CronJitterConfig 类型定义
│   ├── src/services/analytics/growthbook.ts  # GrowthBook 接口
│   └── src/utils/lazySchema.ts          # Zod 模式延迟加载
└── 无其他导出
```

---

## 依赖与外部交互

### 运行时依赖

| 模块 | 用途 |
|------|------|
| `zod/v4` | 配置验证模式定义 |

### 内部模块依赖

| 模块 | 导入内容 | 用途 |
|------|----------|------|
| `src/services/analytics/growthbook.js` | `getFeatureValue_CACHED_WITH_REFRESH` | 远程配置获取 |
| `src/utils/cronTasks.js` | `CronJitterConfig`, `DEFAULT_CRON_JITTER_CONFIG` | 类型和默认值 |
| `src/utils/lazySchema.js` | `lazySchema` | 延迟加载 Zod 模式 |

### 架构设计

**分离原因**：
- `cronScheduler.ts` 需要被打包到 Agent SDK 公共构建中
- `growthbook.ts` 有庞大的传递依赖（settings/hooks/config 循环）
- 分离后 SDK/Daemon 调用者可以不依赖 GrowthBook 直接使用默认配置

**使用模式**：
```typescript
// REPL 调用者（有 GrowthBook）
createCronScheduler({
  getJitterConfig: getCronJitterConfig,  // 动态配置
})

// Daemon 调用者（无 GrowthBook）
createCronScheduler({
  // 省略 getJitterConfig → 使用 DEFAULT_CRON_JITTER_CONFIG
})
```

---

## 风险、边界与改进建议

### 已知风险

1. **配置传播延迟**
   - 60 秒刷新间隔意味着配置变更最多需要 60 秒才能生效
   - 在紧急负载调控场景下可能不够及时

2. **验证严格性**
   - 任何字段无效都会导致整个配置被拒绝
   - 可能因单个字段的 typo 导致配置回退

3. **GrowthBook 依赖**
   - 模块功能完全依赖 GrowthBook 服务可用性
   - 服务不可用时依赖缓存值，但新启动的客户端可能无法获取配置

### 边界情况

| 场景 | 处理 |
|------|------|
| GrowthBook 返回 null | 使用默认值 |
| 配置字段缺失 | `recurringMaxAgeMs` 有默认值，其他字段导致验证失败 |
| 数值超界 | Zod 验证失败，使用默认值 |
| floor > max | `refine` 验证失败，使用默认值 |
| 缓存过期但刷新失败 | 继续使用旧缓存值 |

### 改进建议

1. **配置粒度**
   - 考虑支持按任务类型或用户分组的差异化配置
   - 添加白名单/黑名单机制控制哪些任务受抖动影响

2. **验证策略**
   - 考虑部分验证：无效字段使用默认值，而非整体回退
   - 添加配置变更日志，便于调试配置问题

3. **刷新策略**
   - 添加指数退避机制处理 GrowthBook 服务不可用
   - 支持强制刷新接口供运营紧急使用

4. **监控与可观测性**
   - 添加配置获取成功率指标
   - 记录配置变更事件用于审计

5. **文档完善**
   - 添加运营手册，说明如何调整配置应对负载高峰
   - 记录各配置项对用户体验的具体影响

### 相关 Issue/PR 参考

- 本模块是 `#19931`（定时任务功能）的运营调控基础设施
- 设计遵循 `pollConfig.ts` 的防御性配置模式
