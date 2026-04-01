# keys.ts 深度研究文档

## 场景与职责

`keys.ts` 是 Claude Code CLI 中管理 GrowthBook（功能开关服务）客户端密钥的核心模块。它根据用户类型（内部 ant 用户 vs 外部用户）和环境配置，提供相应的 GrowthBook SDK 密钥。

### 核心使用场景
1. **功能开关初始化**：为 GrowthBook SDK 提供客户端密钥
2. **环境区分**：区分生产环境和开发环境的功能开关
3. **用户类型区分**：内部员工与外部用户使用不同的密钥和配置
4. **延迟读取**：支持在模块加载后动态设置的环境变量

---

## 功能点目的

### 1. GrowthBook 客户端密钥 (`getGrowthBookClientKey`)

**功能**：根据用户类型和环境返回适当的 GrowthBook SDK 密钥。

**密钥分配策略**：

| 用户类型 | 环境 | 密钥 | 用途 |
|---------|------|------|------|
| 外部用户 (`USER_TYPE !== 'ant'`) | 任何 | `sdk-zAZezfDKGoZuXXKe` | 外部用户功能开关 |
| 内部 ant 用户 | 生产 | `sdk-xRVcrliHIlrg4og4` | 内部生产环境 |
| 内部 ant 用户 | 开发 | `sdk-yZQvlplybuXjYh6L` | 内部开发测试 |

**开发环境检测**：
- 通过 `ENABLE_GROWTHBOOK_DEV` 环境变量控制
- 使用 `isEnvTruthy()` 解析环境变量值

### 2. 延迟读取设计

**问题场景**：
```
模块加载顺序：
1. keys.ts 加载
2. globalSettings.env 应用（设置 ENABLE_GROWTHBOOK_DEV）
3. 如果 keys.ts 在加载时就读取 ENABLE_GROWTHBOOK_DEV，会错过后续设置
```

**解决方案**：
```typescript
// 使用函数包装，延迟到调用时才读取
export function getGrowthBookClientKey(): string {
  // 每次调用时读取最新环境变量值
  return process.env.USER_TYPE === 'ant'
    ? isEnvTruthy(process.env.ENABLE_GROWTHBOOK_DEV)
      ? 'sdk-yZQvlplybuXjYh6L'
      : 'sdk-xRVcrliHIlrg4og4'
    : 'sdk-zAZezfDKGoZuXXKe'
}
```

---

## 具体技术实现

### 数据结构

```typescript
// 延迟读取的函数导出
export function getGrowthBookClientKey(): string
```

### 实现细节

```typescript
import { isEnvTruthy } from '../utils/envUtils.js'

export function getGrowthBookClientKey(): string {
  return process.env.USER_TYPE === 'ant'
    ? isEnvTruthy(process.env.ENABLE_GROWTHBOOK_DEV)
      ? 'sdk-yZQvlplybuXjYh6L'      // 内部开发环境
      : 'sdk-xRVcrliHIlrg4og4'      // 内部生产环境
    : 'sdk-zAZezfDKGoZuXXKe'        // 外部用户
}
```

**关键点**：
- 使用函数而非常量，实现延迟求值
- `USER_TYPE` 是构建时 define，安全使用
- `ENABLE_GROWTHBOOK_DEV` 运行时读取，通过函数延迟

### 关键代码路径

#### 1. GrowthBook 初始化路径

```
应用启动
    ↓
src/services/analytics/growthbook.ts
    ↓
调用 getGrowthBookClientKey()
    ↓
根据当前环境返回适当密钥
    ↓
初始化 GrowthBook SDK
```

**关键文件引用**：
- `src/services/analytics/growthbook.ts`: GrowthBook 服务初始化

**初始化代码示例**（来自 `growthbook.ts`）：
```typescript
import { getGrowthBookClientKey } from '../../constants/keys.js'

const growthbook = new GrowthBook({
  clientKey: getGrowthBookClientKey(),
  // ... 其他配置
})
```

---

## 依赖与外部交互

### 内部依赖

| 导入 | 用途 |
|------|------|
| `../utils/envUtils.js` 的 `isEnvTruthy` | 解析环境变量布尔值 |

### 被依赖方

| 文件 | 使用的函数 | 用途 |
|------|-----------|------|
| `src/services/analytics/growthbook.ts` | `getGrowthBookClientKey` | GrowthBook SDK 初始化 |

### 外部服务交互

| 服务 | 交互方式 | 说明 |
|------|---------|------|
| **GrowthBook** | SDK 密钥 | 用于连接 GrowthBook 服务获取功能开关配置 |

### 环境变量

| 变量 | 类型 | 用途 |
|------|------|------|
| `USER_TYPE` | 构建时 define | 区分内部（ant）和外部用户 |
| `ENABLE_GROWTHBOOK_DEV` | 运行时环境变量 | 内部用户切换到开发环境 |

---

## 风险、边界与改进建议

### 当前风险

1. **密钥硬编码**
   - GrowthBook 密钥硬编码在源代码中
   - 虽然客户端密钥相对安全，但仍可能被提取

2. **密钥泄露风险**
   - 如果代码仓库被公开，密钥可能泄露
   - 需要定期轮换密钥

3. **环境变量依赖**
   - `ENABLE_GROWTHBOOK_DEV` 依赖运行时环境变量
   - 如果在错误时机读取，可能获取错误密钥

4. **单点故障**
   - 只有一个函数提供密钥
   - 如果函数有 bug，所有功能开关都受影响

### 边界情况

| 场景 | 行为 |
|------|------|
| `USER_TYPE` 未定义 | 视为外部用户，使用外部密钥 |
| `ENABLE_GROWTHBOOK_DEV` 为各种真值 | `isEnvTruthy` 处理：`true`, `1`, `yes` 等 |
| `ENABLE_GROWTHBOOK_DEV` 为各种假值 | `isEnvTruthy` 处理：`false`, `0`, `no`, 空字符串等 |
| 函数被频繁调用 | 每次调用重新读取环境变量，无缓存 |

### 改进建议

1. **密钥外部化**
   ```typescript
   // 建议从配置文件或环境变量读取密钥
   export function getGrowthBookClientKey(): string {
     // 优先从环境变量读取
     if (process.env.GROWTHBOOK_CLIENT_KEY) {
       return process.env.GROWTHBOOK_CLIENT_KEY
     }
     // 回退到硬编码
     return process.env.USER_TYPE === 'ant'
       ? isEnvTruthy(process.env.ENABLE_GROWTHBOOK_DEV)
         ? 'sdk-yZQvlplybuXjYh6L'
         : 'sdk-xRVcrliHIlrg4og4'
       : 'sdk-zAZezfDKGoZuXXKe'
   }
   ```

2. **密钥缓存**
   ```typescript
   // 建议添加可选缓存
   let cachedKey: string | undefined
   
   export function getGrowthBookClientKey(useCache = true): string {
     if (useCache && cachedKey) {
       return cachedKey
     }
     const key = /* 计算密钥 */
     cachedKey = key
     return key
   }
   
   export function clearGrowthBookClientKeyCache(): void {
     cachedKey = undefined
   }
   ```

3. **错误处理增强**
   ```typescript
   export function getGrowthBookClientKey(): string {
     const userType = process.env.USER_TYPE
     if (userType !== 'ant' && userType !== undefined) {
       console.warn(`Unexpected USER_TYPE: ${userType}`)
     }
     // ...
   }
   ```

4. **配置验证**
   ```typescript
   // 建议添加配置验证函数
   export function validateGrowthBookConfig(): { valid: boolean; errors: string[] } {
     const errors: string[] = []
     
     const key = getGrowthBookClientKey()
     if (!key.startsWith('sdk-')) {
       errors.push('Invalid GrowthBook client key format')
     }
     
     return { valid: errors.length === 0, errors }
   }
   ```

5. **遥测分离**
   ```typescript
   // 建议区分遥测密钥和功能开关密钥
   export function getGrowthBookClientKey(): string  // 功能开关
   export function getGrowthBookTelemetryKey(): string  // 遥测数据
   ```

6. **文档化密钥用途**
   ```typescript
   /**
    * GrowthBook SDK 客户端密钥
    * 
    * 密钥用途：
    * - sdk-zAZezfDKGoZuXXKe: 外部用户功能开关（生产）
    * - sdk-xRVcrliHIlrg4og4: 内部员工功能开关（生产）
    * - sdk-yZQvlplybuXjYh6L: 内部员工功能开关（开发测试）
    * 
    * 安全说明：
    * - 这些是客户端密钥，仅用于读取功能开关配置
    * - 不包含敏感的服务端密钥
    * - 密钥泄露风险较低，但仍应定期轮换
    */
   export function getGrowthBookClientKey(): string
   ```

### 与 GrowthBook 服务的关系

```
keys.ts (提供密钥)
    ↓ 被调用
services/analytics/growthbook.ts (初始化 SDK)
    ↓ 连接
GrowthBook 服务
    ↓ 返回
功能开关配置
    ↓ 控制
应用功能启用/禁用
```

GrowthBook 客户端密钥是连接应用与功能开关服务的凭证，虽然权限有限（只读），但仍应妥善管理。
