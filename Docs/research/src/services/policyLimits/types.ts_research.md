# 研究文档：src/services/policyLimits/types.ts

## 场景与职责

`src/services/policyLimits/types.ts` 是 `policyLimits` 服务的**类型契约层**，负责定义：
1. 后端 API 响应体的运行时校验模式（Zod Schema）。
2. TypeScript 类型推导结果，供 `index.ts` 及外部调用方使用。
3. 策略获取操作的返回结果类型（`PolicyLimitsFetchResult`），统一表达成功、失败、缓存命中（304）等多种状态。

该文件虽小，但处于整个策略限制数据流的上游：任何对后端响应格式的误解或类型漂移，都会直接传递到缓存持久化、同步查询、功能开关等核心路径。因此其设计强调**运行时校验优先**（Zod `safeParse`）和**延迟初始化**（`lazySchema`），以控制模块加载成本。

---

## 功能点目的

| 功能点 | 目的 |
|--------|------|
| **`PolicyLimitsResponseSchema`** | 用 Zod v4 定义后端 `/api/claude_code/policy_limits` 的响应结构，确保运行时收到的 JSON 符合预期。 |
| **`PolicyLimitsResponse` 类型** | 从 Schema 自动推断出 TypeScript 类型，消除手写类型与运行时校验之间的不同步风险。 |
| **`PolicyLimitsFetchResult` 类型** | 为 `fetchPolicyLimits` / `fetchWithRetry` 提供统一的返回类型，明确区分成功、失败、304 缓存有效、认证错误不可重试等语义。 |
| **`lazySchema` 包装** | 延迟 Zod Schema 的实例化，避免在仅导入类型声明时触发不必要的对象创建，优化冷启动性能。 |

---

## 具体技术实现

### 3.1 Zod Schema 定义

```ts
export const PolicyLimitsResponseSchema = lazySchema(() =>
  z.object({
    restrictions: z.record(z.string(), z.object({ allowed: z.boolean() })),
  }),
)
```

**结构解析**：
- `restrictions` 是一个**字符串到对象的映射**（`z.record(z.string(), ...) `）。
- 每个策略键对应的值必须包含一个 `allowed` 字段，且类型严格为 `boolean`。
- 该 Schema 对未知策略键**不拒绝**：只要键是字符串、值包含 `allowed: boolean`，即视为合法。这与 `index.ts` 中 "未知策略 = 允许" 的 fail-open 语义一致。
- 该 Schema **不要求**特定的策略键必须存在（如 `allow_remote_control`、`allow_remote_sessions` 等），因为后端只返回被显式配置的策略；缺失的键在业务层被解释为允许。

**运行时校验点**：
- `index.ts:352`：`PolicyLimitsResponseSchema().safeParse(response.data)` 对 HTTP 200 响应进行校验。
- `index.ts:396`：`PolicyLimitsResponseSchema().safeParse(data)` 对本地缓存文件内容进行校验。

### 3.2 类型推导

```ts
export type PolicyLimitsResponse = z.infer<
  ReturnType<typeof PolicyLimitsResponseSchema>
>
```

**推导结果等价于**：
```ts
type PolicyLimitsResponse = {
  restrictions: Record<string, { allowed: boolean }>
}
```

使用 `ReturnType<typeof PolicyLimitsResponseSchema>` 而非直接引用 Schema 对象，是因为 `PolicyLimitsResponseSchema` 本身是一个**工厂函数**（由 `lazySchema` 返回），调用后才得到真正的 Zod Schema 实例。`z.infer<>` 作用于该实例的类型，因此需要 `ReturnType` 解包。

### 3.3 Fetch 结果类型

```ts
export type PolicyLimitsFetchResult = {
  success: boolean
  restrictions?: PolicyLimitsResponse['restrictions'] | null
  etag?: string
  error?: string
  skipRetry?: boolean
}
```

**字段语义**：

| 字段 | 类型 | 语义 |
|------|------|------|
| `success` | `boolean` | 操作是否成功。`true` 不代表一定有新的策略内容（可能是 304 或 404）。 |
| `restrictions` | `PolicyLimitsResponse['restrictions'] \| null \| undefined` | 成功时携带的策略内容。`null` 具有特殊含义：**304 Not Modified**，表示本地缓存仍然有效。`undefined` 通常不出现在成功路径。`{}` 表示 404，即无策略限制。 |
| `etag` | `string \| undefined` | 当前有效的校验和。在 304 路径中用于回传已有的 `cachedChecksum`。 |
| `error` | `string \| undefined` | 失败时的可读错误信息，用于日志和调试。 |
| `skipRetry` | `boolean \| undefined` | 当为 `true` 时，上层 `fetchWithRetry` 不会继续重试。典型场景：认证失败（401/403）、无可用认证凭证。 |

**状态组合矩阵**（基于 `index.ts` 实现）：

| 场景 | `success` | `restrictions` | `etag` | `skipRetry` |
|------|-----------|----------------|--------|-------------|
| 200 新策略 | `true` | `{ ... }` | `undefined` | `undefined` |
| 304 缓存有效 | `true` | `null` | `cachedChecksum` | `undefined` |
| 404 无策略 | `true` | `{}` | `undefined` | `undefined` |
| 认证失败 | `false` | `undefined` | `undefined` | `true` |
| 超时/网络/HTTP 错误 | `false` | `undefined` | `undefined` | `undefined` |
| 格式非法 | `false` | `undefined` | `undefined` | `undefined` |

### 3.4 lazySchema 的使用动机

```ts
import { lazySchema } from '../../utils/lazySchema.js'
```

`lazySchema` 是一个简单的工厂包装器（通常实现为 `() => schema`），其目的是：
- **延迟 Zod 对象创建**：如果某个模块仅导入 `PolicyLimitsResponse` 类型（TypeScript 类型擦除后无运行时影响），`lazySchema` 确保不会在模块加载时立即构造 Zod Schema 对象。
- **避免循环依赖或重量级初始化**：在大型代码库中，Zod Schema 可能引用其他模块的类型或辅助函数，`lazySchema` 将这些引用推迟到实际调用 `safeParse` 时。
- **与 `z.lazy` 的区别**：这里的 `lazySchema` 是应用层工具，用于延迟整个 Schema 的创建；而 Zod 内置的 `z.lazy` 用于处理递归类型定义。两者解决的问题域不同。

在 `index.ts` 中，Schema 的调用方式为 `PolicyLimitsResponseSchema().safeParse(...)`，即先调用工厂函数获取实例，再调用实例方法。

---

## 关键代码路径与文件引用

### 4.1 本模块内部

| 符号 | 行号 | 说明 |
|------|------|------|
| `PolicyLimitsResponseSchema` | 8-12 | Zod Schema 工厂函数 |
| `PolicyLimitsResponse` | 14-16 | 从 Schema 推断的 TypeScript 类型 |
| `PolicyLimitsFetchResult` | 21-27 | Fetch 操作结果类型 |

### 4.2 消费方（谁在使用这些类型/Schema）

| 消费方文件 | 使用的符号 | 用途 |
|------------|-----------|------|
| `src/services/policyLimits/index.ts` | `PolicyLimitsResponseSchema`, `PolicyLimitsResponse`, `PolicyLimitsFetchResult` | 运行时校验、类型标注、fetch 结果构造 |

`types.ts` 的导出符号目前**仅被 `index.ts` 直接消费**。外部调用方（如 `main.tsx`、`bridge.tsx` 等）只使用 `index.ts` 封装后的 `isPolicyAllowed`、`waitForPolicyLimitsToLoad` 等函数，不直接触碰 Schema 和原始响应类型。这种封装保持了类型层的内聚性，降低了外部模块的耦合度。

### 4.3 依赖方（types.ts 依赖谁）

| 依赖文件 | 使用的符号 | 说明 |
|----------|-----------|------|
| `zod/v4` | `z` | Zod v4 运行时库 |
| `src/utils/lazySchema.ts` | `lazySchema` | 延迟 Schema 实例化的应用层工具 |

---

## 依赖与外部交互

### 5.1 与后端 API 的契约

`types.ts` 是前端对后端 `/api/claude_code/policy_limits` 响应格式的**唯一正式契约**。当前契约要求：

- 根对象必须包含 `restrictions` 字段。
- `restrictions` 的值是一个对象，其所有键为字符串，所有值为 `{ allowed: boolean }`。

**契约示例**：
```json
{
  "restrictions": {
    "allow_remote_control": { "allowed": false },
    "allow_remote_sessions": { "allowed": true },
    "allow_product_feedback": { "allowed": false }
  }
}
```

**非契约内容**：
- 不限制 `restrictions` 的键名集合（前端通过 `isPolicyAllowed` 按需查询）。
- 不限制额外顶层字段的存在（当前 Schema 使用 `z.object()` 默认会**剥离**未知键，但不会导致解析失败）。

### 5.2 与 index.ts 的协作数据流

```
后端 API 响应
    │
    ▼
index.ts: fetchPolicyLimits()
    │
    ├── 200 ──► PolicyLimitsResponseSchema().safeParse(response.data)
    │              │
    │              ▼
    │         校验通过 ──► 构造 PolicyLimitsFetchResult
    │         校验失败 ──► 构造失败 PolicyLimitsFetchResult
    │
    ├── 304 ──► 构造 success=true, restrictions=null 的 PolicyLimitsFetchResult
    │
    └── 404 ──► 构造 success=true, restrictions={} 的 PolicyLimitsFetchResult
```

`types.ts` 在这个数据流中扮演了"守门员"角色：任何不符合 `PolicyLimitsResponseSchema` 的数据都会在 `index.ts` 中被拦截，避免污染 `sessionCache` 和磁盘缓存。

---

## 风险、边界与改进建议

### 6.1 已知风险

1. **Schema 过于宽松导致静默失败**
   - `z.record(z.string(), z.object({ allowed: z.boolean() }))` 对键名没有任何约束。如果后端因 bug 将 `allow_remote_sessions` 拼写为 `allow_remote_session`，前端会静默接受该键，但 `isPolicyAllowed('allow_remote_sessions')` 会因查找不到而返回 `true`（fail open）。这意味着拼写错误可能导致策略**实际上失效**。

2. **`restrictions` 为 `null` 的边界未在 Schema 中体现**
   - `PolicyLimitsResponseSchema` 要求 `restrictions` 必须是一个对象。但在 `PolicyLimitsFetchResult` 中，`restrictions` 可以为 `null`（表示 304）。这个 `null` 语义是业务层（`index.ts`）引入的，不在 Schema 中。如果未来有开发者误以为 `null` 也是合法的 API 响应值，可能会产生混淆。

3. **Zod v4 升级风险**
   - 项目使用 `zod/v4`（`import { z } from 'zod/v4'`）。Zod v4 与 v3 在 API 和行为上存在差异（如 `.pipe()` 行为、错误消息格式等）。若未来升级或降级 Zod 版本，`lazySchema` 内部的 Schema 定义可能需要调整。

4. **类型导出范围有限**
   - `PolicyLimitsFetchResult` 目前只在 `index.ts` 内部使用。如果未来其他服务（如 `remoteManagedSettings`）希望复用类似的 fetch result 模式，当前没有提供泛型抽象。

### 6.2 边界情况

- **空 `restrictions` 对象 `{}`**：Schema 校验通过，`index.ts` 将其视为"无策略限制"，会删除本地缓存文件。这是预期行为。
- **超大 `restrictions` 对象**：虽然理论上 `z.record` 可以处理大量键，但 `index.ts` 中的 `computeChecksum` 需要对整个对象做稳定 JSON 序列化和 SHA-256 计算。如果后端返回了异常大的策略对象（如数千个键），可能会影响性能。
- **非布尔 `allowed` 值**：如 `"true"`（字符串）或 `1`（数字），Zod 会严格拒绝，导致 `safeParse` 失败，进而触发 `index.ts` 的格式错误路径。

### 6.3 改进建议

1. **增加已知策略键的警告机制**
   - 在 `index.ts` 的 `fetchPolicyLimits` 中，于 `safeParse` 成功后增加一层白名单检查：
     ```ts
     const KNOWN_POLICIES = new Set(['allow_remote_control', 'allow_remote_sessions', 'allow_product_feedback'])
     for (const key of Object.keys(parsed.data.restrictions)) {
       if (!KNOWN_POLICIES.has(key)) {
         logForDebugging(`Policy limits: unknown restriction key "${key}"`)
       }
     }
     ```
     这能在后端出现键名漂移时快速暴露问题，同时保持 Schema 的向后兼容性。

2. **将 304 语义从 `null` 改为显式 discriminated union**
   - 当前 `restrictions: ... | null` 的语义有些隐晦。可以考虑将 `PolicyLimitsFetchResult` 重构为 discriminated union：
     ```ts
     type PolicyLimitsFetchResult =
       | { success: true; kind: 'fresh'; restrictions: PolicyLimitsResponse['restrictions'] }
       | { success: true; kind: 'not_modified'; etag: string }
       | { success: true; kind: 'empty' }
       | { success: false; error: string; skipRetry?: boolean }
     ```
     这样能彻底消除 `null` 的歧义，使调用方通过 `kind` 字段做穷尽检查（exhaustiveness check）。

3. **提取泛型 FetchResult 类型**
   - `remoteManagedSettings` 等服务也有类似的 `success/restrictions/error/skipRetry` 模式。可以将 `PolicyLimitsFetchResult` 泛化为：
     ```ts
     type FetchResult<T> =
       | { success: true; data: T; etag?: string }
       | { success: false; error: string; skipRetry?: boolean }
     ```
     然后在 `policyLimits` 和 `remoteManagedSettings` 中复用，减少重复类型定义。

4. **Schema 增加 `strict()` 或 `passthrough()` 的显式声明**
   - 当前 `z.object({ restrictions: ... })` 默认会剥离未知顶层字段。如果希望后端新增字段时前端能感知（用于未来扩展），可以改为：
     ```ts
     z.object({ ... }).passthrough()
     ```
     这样未知字段会被保留在 `parsed.data` 中，便于后续平滑升级。

5. **文档化策略键名列表**
   - 在 `types.ts` 或 `index.ts` 的 JSDoc 中维护一个当前已知的策略键名列表，作为前后端开发者的契约参考。例如：
     ```ts
     /**
      * Known policy keys (as of 2026-04):
      * - 'allow_remote_control': Enables /remote-control and bridge features.
      * - 'allow_remote_sessions': Enables teleport, scheduled remote agents, and remote environments.
      * - 'allow_product_feedback': Enables in-app feedback surveys and transcript sharing.
      */
     ```
