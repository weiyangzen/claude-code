# memoize.ts 研究文档

## 场景与职责

本模块提供了三种高级记忆化（memoization）工具函数，用于缓存函数结果以提升性能。核心职责包括：

1. **TTL 记忆化**：带生存时间（Time-To-Live）的缓存，支持后台异步刷新
2. **异步 TTL 记忆化**：专为异步函数设计，支持并发去重和后台刷新
3. **LRU 记忆化**：基于最近最少使用（Least Recently Used）策略的有限容量缓存

该模块是性能优化的基础设施，广泛应用于配置加载、认证凭证获取、平台检测等场景。

## 功能点目的

### 1. `memoizeWithTTL()` - TTL 记忆化（同步）
- **目的**：缓存同步函数结果，在 TTL 过期后后台刷新
- **核心特性**：
  - **写穿透缓存**：缓存未命中时立即计算并存储
  - **后台刷新**：TTL 过期后返回旧值，异步计算新值
  - **防重复刷新**：使用 `refreshing` 标志防止并发刷新
  - **身份保护**：刷新完成后验证缓存条目未被清除

### 2. `memoizeWithTTLAsync()` - TTL 记忆化（异步）
- **目的**：缓存异步函数结果，专为 AWS 凭证刷新等场景设计
- **核心特性**：
  - **并发去重**：使用 `inFlight` Map 防止并发冷启动调用
  - **Promise 缓存**：首次调用存储 Promise，后续调用共享结果
  - **错误处理**：刷新失败时清除缓存，下次调用重新尝试
  - **clear 一致性**：清除缓存同时清除 `inFlight` Map

### 3. `memoizeWithLRU()` - LRU 记忆化
- **目的**：限制缓存大小，防止无界内存增长
- **核心特性**：
  - **容量限制**：通过 `maxCacheSize` 参数限制条目数
  - **LRU 淘汰**：使用 `lru-cache` 库实现最近最少使用淘汰
  - **缓存管理**：提供 `clear/size/delete/get/has` 等管理方法
  - **自定义键函数**：允许调用者指定缓存键生成逻辑

## 具体技术实现

### 关键流程

#### memoizeWithTTL 执行流程

```
调用 memoized(...args)
    ↓
生成缓存键：jsonStringify(args)
    ↓
缓存是否存在？
    ↓
    否 → 计算值 → 存储 → 返回值
    ↓
    是 → TTL 是否过期？
              ↓
              否 → 返回缓存值
              ↓
              是 → 是否正在刷新？
                        ↓
                        是 → 返回旧值
                        否 → 标记刷新中 → 调度后台刷新 → 返回旧值
```

#### memoizeWithTTLAsync 执行流程

```
调用 memoized(...args)
    ↓
生成缓存键
    ↓
缓存是否存在？
    ↓
    否 → inFlight 中是否存在？
              ↓
              是 → 返回已有 Promise
              否 → 创建 Promise → 存入 inFlight → await → 存入缓存 → 清除 inFlight
    ↓
    是 → TTL 是否过期？
              ↓
              否 → 返回缓存值
              ↓
              是 → 后台刷新（类似同步版本）→ 返回旧值
```

### 数据结构

```typescript
// 缓存条目结构
type CacheEntry<T> = {
  value: T
  timestamp: number      // 缓存创建时间
  refreshing: boolean    // 是否正在后台刷新
}

// 同步记忆化函数类型
type MemoizedFunction<Args extends unknown[], Result> = {
  (...args: Args): Result
  cache: { clear: () => void }
}

// LRU 记忆化函数类型（扩展管理方法）
type LRUMemoizedFunction<Args extends unknown[], Result> = {
  (...args: Args): Result
  cache: {
    clear: () => void
    size: () => number
    delete: (key: string) => boolean
    get: (key: string) => Result | undefined
    has: (key: string) => boolean
  }
}
```

### 身份保护机制

身份保护是防止竞态条件的关键设计：

```typescript
// 后台刷新完成后的验证
.then(newValue => {
  if (cache.get(key) === staleEntry) {  // 验证条目未被修改
    cache.set(key, { value: newValue, timestamp: Date.now(), refreshing: false })
  }
})
.catch(e => {
  if (cache.get(key) === staleEntry) {  // 仅清除自己刷新的条目
    cache.delete(key)
  }
})
```

### 并发去重实现

```typescript
const inFlight = new Map<string, Promise<Result>>()

// 冷启动时检查是否有进行中的请求
const pending = inFlight.get(key)
if (pending) return pending

// 创建新的 Promise 并记录
const promise = f(...args)
inFlight.set(key, promise)

try {
  const result = await promise
  // 身份保护验证后存入缓存
  if (inFlight.get(key) === promise) {
    cache.set(key, { value: result, timestamp: now, refreshing: false })
  }
  return result
} finally {
  // 仅清除自己创建的 Promise
  if (inFlight.get(key) === promise) {
    inFlight.delete(key)
  }
}
```

## 依赖与外部交互

### 直接依赖

| 模块 | 用途 |
|------|------|
| `lru-cache` | LRU 缓存实现 |
| `./log.js` | 错误日志记录 |
| `./slowOperations.js` | JSON 序列化（用于缓存键） |

### 调用方

| 调用方 | 用途 |
|--------|------|
| `src/utils/auth.ts` | `memoizeWithTTLAsync` 用于 AWS 凭证刷新 |
| `src/utils/platform.ts` | `memoize`（lodash）用于平台检测 |
| `src/utils/mtls.ts` | `memoize`（lodash）用于 mTLS 配置 |
| `src/utils/caCerts.ts` | `memoize`（lodash）用于 CA 证书加载 |
| `src/utils/windowsPaths.ts` | `memoizeWithLRU` 用于路径转换 |
| `src/utils/messageQueueManager.ts` | `objectGroupBy` 用于消息分组 |

### 相关工具

- `lodash-es/memoize`：项目中广泛使用的简单记忆化
- `lru-cache`：本模块 LRU 功能的底层实现

## 风险、边界与改进建议

### 已知风险

1. **缓存键冲突**
   - 风险：`jsonStringify(args)` 可能产生相同键的不同参数
   - 示例：`memoizeWithTTL(fn)({a:1,b:2})` 和 `fn({b:2,a:1})` 键相同
   - 缓解：对象键顺序通常稳定，但非绝对保证

2. **内存泄漏**
   - `memoizeWithTTL`：无界 Map 可能无限增长
   - `memoizeWithTTLAsync`：同上
   - 缓解：使用 `memoizeWithLRU` 限制容量

3. **后台刷新失败**
   - 风险：异步刷新失败可能导致缓存被清除
   - 影响：下次调用需要重新计算，性能下降
   - 现状：错误被记录，缓存被清除

4. **TTL 精度**
   - 风险：`Date.now()` 可能受系统时间调整影响
   - 建议：使用 `performance.now()` 或单调时钟

### 边界情况

| 场景 | 行为 |
|------|------|
| TTL = 0 | 每次调用都视为过期，但仍有后台刷新逻辑 |
| TTL = Infinity | 永不过期，相当于普通记忆化 |
| 函数抛出异常 | 异常传播，不缓存错误结果 |
| 异步函数拒绝 | 清除 inFlight，下次调用重试 |
| 并发清除 + 刷新 | 身份保护机制确保正确性 |
| LRU 容量为 0 | 每次调用都缓存未命中 |
| LRU peek 操作 | 不更新访问时间，仅观察 |

### 改进建议

1. **缓存统计**
   - 当前：无命中率统计
   - 建议：添加 `hits/misses` 计数器
   - 用途：性能调优和监控

2. **过期回调**
   - 当前：TTL 过期仅触发后台刷新
   - 建议：添加可选的过期回调函数
   - 用途：资源清理、日志记录

3. **多级缓存**
   - 当前：单级内存缓存
   - 建议：支持 L1（内存）+ L2（磁盘/Redis）
   - 用途：跨进程缓存共享

4. **缓存序列化**
   - 当前：仅内存存储
   - 建议：支持缓存持久化到磁盘
   - 用途：进程重启后快速恢复

5. **LRU 增强**
   - 当前：仅支持基本操作
   - 建议：
     - 添加 `keys()/values()` 遍历
     - 支持 TTL + LRU 组合策略
     - 添加权重支持（不同条目占用不同容量）

6. **类型安全增强**
   - 当前：缓存键使用 `string`
   - 建议：使用 branded type 区分不同函数的缓存
   - 用途：防止意外的缓存键冲突

7. **性能优化**
   - 当前：`jsonStringify` 生成缓存键
   - 建议：
     - 支持自定义键函数（如 `memoizeWithLRU`）
     - 对于简单参数使用 `String(args)`
     - 考虑使用 WeakMap 用于对象参数

8. **错误重试策略**
   - 当前：刷新失败立即清除缓存
   - 建议：添加指数退避重试
   - 配置：最大重试次数、退避基数
