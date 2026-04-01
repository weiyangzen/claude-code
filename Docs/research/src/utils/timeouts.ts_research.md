# timeouts.ts 研究文档

## 场景与职责

`timeouts.ts` 是 Claude Code CLI 的超时配置管理模块，专门用于管理 Bash 工具（以及 PowerShell 工具）的默认和最大超时时间。该模块提供集中式的超时配置读取，支持通过环境变量进行自定义。

主要使用场景：
1. **Bash 工具超时**：为 `BashTool` 和 `PowerShellTool` 提供默认和最大超时时间
2. **用户自定义**：允许用户通过环境变量调整超时策略
3. **安全边界**：确保超时值在合理范围内，防止无限等待

## 功能点目的

### 1. 可配置的超时时间
- **默认值**：2 分钟（`120_000 ms`）
- **最大值**：10 分钟（`600_000 ms`）
- **自定义方式**：通过环境变量 `BASH_DEFAULT_TIMEOUT_MS` 和 `BASH_MAX_TIMEOUT_MS`

### 2. 安全边界检查
- 确保最大超时 >= 默认超时
- 解析失败时回退到安全默认值
- 负数或零值被视为无效

## 具体技术实现

### 核心常量

```typescript
const DEFAULT_TIMEOUT_MS = 120_000  // 2 minutes
const MAX_TIMEOUT_MS = 600_000      // 10 minutes
```

### 类型定义

```typescript
type EnvLike = Record<string, string | undefined>
```

**设计意图**：允许传入自定义环境变量对象，便于测试和不同环境（如浏览器 vs Node.js）。

### 函数实现

#### `getDefaultBashTimeoutMs`

```typescript
export function getDefaultBashTimeoutMs(env: EnvLike = process.env): number {
  const envValue = env.BASH_DEFAULT_TIMEOUT_MS
  if (envValue) {
    const parsed = parseInt(envValue, 10)
    if (!isNaN(parsed) && parsed > 0) {
      return parsed
    }
  }
  return DEFAULT_TIMEOUT_MS
}
```

**逻辑**：
1. 读取 `BASH_DEFAULT_TIMEOUT_MS` 环境变量
2. 尝试解析为整数
3. 验证：不是 NaN 且 > 0
4. 无效时回退到 `DEFAULT_TIMEOUT_MS`

#### `getMaxBashTimeoutMs`

```typescript
export function getMaxBashTimeoutMs(env: EnvLike = process.env): number {
  const envValue = env.BASH_MAX_TIMEOUT_MS
  if (envValue) {
    const parsed = parseInt(envValue, 10)
    if (!isNaN(parsed) && parsed > 0) {
      return Math.max(parsed, getDefaultBashTimeoutMs(env))
    }
  }
  return Math.max(MAX_TIMEOUT_MS, getDefaultBashTimeoutMs(env))
}
```

**关键逻辑**：
- 使用 `Math.max` 确保最大值 >= 默认值
- 即使环境变量设置了较小的值，也会被提升到默认值

## 关键代码路径与文件引用

### 调用方（被谁使用）

| 文件路径 | 使用场景 |
|---------|---------|
| `src/tools/BashTool/prompt.ts` | Bash 工具提示中的超时配置 |
| `src/tools/PowerShellTool/prompt.ts` | PowerShell 工具提示中的超时配置 |

### 依赖模块

本模块无外部依赖，仅使用 JavaScript 内置功能：
- `parseInt`：字符串转数字
- `isNaN`：数字有效性检查
- `Math.max`：取最大值

## 依赖与外部交互

### 与工具系统的集成

超时值通过工具的 prompt 配置传递给工具执行逻辑：

```typescript
// 示例：BashTool/prompt.ts
import { getDefaultBashTimeoutMs, getMaxBashTimeoutMs } from '../../utils/timeouts.js'

const defaultTimeout = getDefaultBashTimeoutMs()
const maxTimeout = getMaxBashTimeoutMs()
```

### 环境变量接口

| 变量名 | 用途 | 默认值 |
|-------|------|-------|
| `BASH_DEFAULT_TIMEOUT_MS` | Bash 命令默认超时 | 120000 (2分钟) |
| `BASH_MAX_TIMEOUT_MS` | Bash 命令最大允许超时 | 600000 (10分钟) |

**使用示例**：
```bash
# 设置默认超时为 5 分钟
export BASH_DEFAULT_TIMEOUT_MS=300000

# 设置最大超时为 30 分钟
export BASH_MAX_TIMEOUT_MS=1800000
```

## 风险、边界与改进建议

### 潜在风险

1. **命名不一致**
   - 模块名为 `timeouts.ts`，但只处理 Bash/PowerShell 超时
   - 其他超时（如 API 超时、MCP 超时）在其他模块管理

2. **单位不明确**
   - 环境变量名包含 `_MS` 后缀，但用户可能误以为是秒
   - 无运行时单位验证

3. **整数溢出**
   - `parseInt` 对极大值（如 `Number.MAX_SAFE_INTEGER`）处理可能产生意外结果
   - 未设置上限（如 24 小时）

### 边界条件

| 场景 | 行为 |
|-----|------|
| `BASH_DEFAULT_TIMEOUT_MS=` | 空字符串，视为未设置，使用默认值 |
| `BASH_DEFAULT_TIMEOUT_MS=abc` | 解析为 NaN，使用默认值 |
| `BASH_DEFAULT_TIMEOUT_MS=0` | 零值，视为无效，使用默认值 |
| `BASH_DEFAULT_TIMEOUT_MS=-1000` | 负值，视为无效，使用默认值 |
| `BASH_MAX_TIMEOUT_MS < BASH_DEFAULT_TIMEOUT_MS` | 被提升到默认值 |
| 两个环境变量都未设置 | 使用硬编码默认值 |

### 改进建议

1. **统一超时管理**
   ```typescript
   // 建议：扩展为通用超时管理模块
   export const Timeouts = {
     bash: { default: 120_000, max: 600_000 },
     api: { default: 60_000, max: 300_000 },
     mcp: { default: 30_000, max: 120_000 },
   } as const
   ```

2. **更友好的环境变量格式**
   ```typescript
   // 支持人类可读的格式
   // BASH_DEFAULT_TIMEOUT=5m → 300000
   // BASH_DEFAULT_TIMEOUT=1h → 3600000
   function parseDuration(input: string): number | null {
     const match = input.match(/^(\d+)(ms|s|m|h)$/)
     // ...
   }
   ```

3. **配置验证和警告**
   ```typescript
   function validateTimeoutConfig(): void {
     const default_ = getDefaultBashTimeoutMs()
     const max = getMaxBashTimeoutMs()
     
     if (default_ > max) {
       console.warn(`Bash timeout config: default (${default_}ms) > max (${max}ms)`)
     }
     
     if (max > 3_600_000) {  // 1 hour
       console.warn(`Bash max timeout (${max}ms) exceeds 1 hour, this may cause issues`)
     }
   }
   ```

4. **运行时调整支持**
   ```typescript
   // 支持会话期间动态调整
   let runtimeDefaultTimeout: number | undefined
   
   export function setDefaultBashTimeout(ms: number): void {
     runtimeDefaultTimeout = ms
   }
   ```

5. **文档和类型安全**
   ```typescript
   /**
    * Get the default timeout for bash operations in milliseconds.
    * @param env Environment variables (defaults to process.env)
    * @returns Timeout in milliseconds (positive integer)
    * @default 120000 (2 minutes)
    */
   export function getDefaultBashTimeoutMs(env?: EnvLike): number
   ```

### 测试建议

应覆盖以下场景：
- 各种有效/无效的环境变量值
- 默认值 > 最大值的边界情况
- 极大值（如 `999999999999`）的处理
- 自定义 `EnvLike` 对象的传入
- 环境变量动态修改后的行为（如果实现运行时调整）
