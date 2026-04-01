# utils.ts 研究文档

## 场景与职责

`utils.ts` 是 `ManagedSettingsSecurityDialog` 组件的核心工具模块，负责**危险设置的提取、检测、比较和格式化**。它是远程托管设置安全审批流程的数据处理层。

### 核心职责

1. **危险设置提取**：从 `SettingsJson` 对象中识别并提取潜在危险的配置项
2. **变更检测**：比较新旧设置，判断危险设置是否发生变化
3. **存在性检测**：快速判断设置中是否包含任何危险项
4. **格式化输出**：将危险设置转换为 UI 可展示的格式

### 使用场景

- 安全对话框渲染前提取危险设置清单
- 判断是否需要显示安全审批对话框
- 检测远程设置更新时是否有新增危险配置
- 生成用户友好的设置列表展示

---

## 功能点目的

### 1. extractDangerousSettings - 危险设置提取器

从完整的设置对象中筛选出三类危险配置：

| 类别 | 来源 | 检测逻辑 |
|------|------|----------|
| `shellSettings` | `settings.*` | 匹配 `DANGEROUS_SHELL_SETTINGS` 列表中的键 |
| `envVars` | `settings.env` | 键名不在 `SAFE_ENV_VARS` 白名单中 |
| `hasHooks` | `settings.hooks` | 对象存在、非空且包含键 |

### 2. hasDangerousSettings - 危险设置存在性检测

快速判断提取结果中是否包含任何危险项：
- 用于提前退出，避免不必要的对话框渲染
- 用于判断是否需要执行详细比较

### 3. hasDangerousSettingsChanged - 变更检测器

比较新旧设置的危险部分，判断是否需要重新提示用户：
- 新增危险设置 → 需要提示
- 修改危险设置 → 需要提示
- 仅移除危险设置 → 不需要提示
- 无变化 → 不需要提示

### 4. formatDangerousSettingsList - 格式化器

将危险设置对象转换为字符串数组：
- 仅包含设置名称，不包含值（安全考虑）
- 用于 UI 列表展示

---

## 具体技术实现

### 关键数据结构

```typescript
// 危险 Shell 设置类型（从常量派生）
type DangerousShellSetting = (typeof DANGEROUS_SHELL_SETTINGS)[number]
// 实际值: 'apiKeyHelper' | 'awsAuthRefresh' | 'awsCredentialExport' | 
//         'gcpAuthRefresh' | 'otelHeadersHelper' | 'statusLine'

// 危险设置对象
export type DangerousSettings = {
  shellSettings: Partial<Record<DangerousShellSetting, string>>  // 危险 shell 设置
  envVars: Record<string, string>                                 // 危险环境变量
  hasHooks: boolean                                               // 是否有 hooks
  hooks?: unknown                                                 // hooks 原始值（可选）
}
```

### 核心算法实现

#### 1. 危险 Shell 设置提取

```typescript
const shellSettings: Partial<Record<DangerousShellSetting, string>> = {}
for (const key of DANGEROUS_SHELL_SETTINGS) {
  const value = settings[key]
  // 仅提取非空字符串值
  if (typeof value === 'string' && value.length > 0) {
    shellSettings[key] = value
  }
}
```

**依赖常量**（来自 `managedEnvConstants.ts`）：
```typescript
export const DANGEROUS_SHELL_SETTINGS = [
  'apiKeyHelper',      // API 密钥辅助脚本
  'awsAuthRefresh',    // AWS 认证刷新
  'awsCredentialExport', // AWS 凭证导出
  'gcpAuthRefresh',    // GCP 认证刷新
  'otelHeadersHelper', // OpenTelemetry 头辅助
  'statusLine',        // 状态栏命令
] as const
```

#### 2. 危险环境变量提取

```typescript
const envVars: Record<string, string> = {}
if (settings.env && typeof settings.env === 'object') {
  for (const [key, value] of Object.entries(settings.env)) {
    if (typeof value === 'string' && value.length > 0) {
      // 关键：不在安全白名单中的变量才被视为危险
      if (!SAFE_ENV_VARS.has(key.toUpperCase())) {
        envVars[key] = value
      }
    }
  }
}
```

**安全白名单策略**：
- `SAFE_ENV_VARS` 包含约 80+ 个被认为安全的环境变量
- 采用**黑名单取反**策略：任何不在白名单中的变量都被视为危险
- 变量名统一转为大写比较（避免大小写绕过）

#### 3. Hooks 检测

```typescript
const hasHooks =
  settings.hooks !== undefined &&
  settings.hooks !== null &&
  typeof settings.hooks === 'object' &&
  Object.keys(settings.hooks).length > 0
```

**注意**：仅检测存在性，不展开 hooks 内容（hooks 结构复杂，在 `hooks.ts` 中定义）。

#### 4. 变更检测算法

```typescript
export function hasDangerousSettingsChanged(
  oldSettings: SettingsJson | null | undefined,
  newSettings: SettingsJson | null | undefined,
): boolean {
  const oldDangerous = extractDangerousSettings(oldSettings)
  const newDangerous = extractDangerousSettings(newSettings)

  // 新设置无危险项 → 无需提示
  if (!hasDangerousSettings(newDangerous)) {
    return false
  }

  // 旧设置无危险项，新设置有 → 需要提示
  if (!hasDangerousSettings(oldDangerous)) {
    return true
  }

  // 比较危险部分的 JSON 序列化结果
  const oldJson = jsonStringify({
    shellSettings: oldDangerous.shellSettings,
    envVars: oldDangerous.envVars,
    hooks: oldDangerous.hooks,
  })
  const newJson = jsonStringify({
    shellSettings: newDangerous.shellSettings,
    envVars: newDangerous.envVars,
    hooks: newDangerous.hooks,
  })

  return oldJson !== newJson
}
```

**关键设计**：
- 使用 `jsonStringify`（来自 `slowOperations.ts`）而非原生 `JSON.stringify`
- `jsonStringify` 包装了性能监控，可检测慢操作
- 比较时包含 `hooks` 原始值，确保 hooks 内容变化也能被检测

---

## 关键代码路径与文件引用

### 当前文件

| 路径 | 说明 |
|------|------|
| `src/components/ManagedSettingsSecurityDialog/utils.ts` | 工具函数实现 |

### 直接依赖

| 路径 | 说明 |
|------|------|
| `src/utils/managedEnvConstants.ts` | `DANGEROUS_SHELL_SETTINGS`, `SAFE_ENV_VARS` 常量 |
| `src/utils/settings/types.ts` | `SettingsJson` 类型定义 |
| `src/utils/slowOperations.ts` | `jsonStringify` 性能监控版序列化 |

### 调用方

| 路径 | 说明 |
|------|------|
| `src/components/ManagedSettingsSecurityDialog/ManagedSettingsSecurityDialog.tsx` | 对话框组件，使用提取和格式化功能 |
| `src/services/remoteManagedSettings/securityCheck.tsx` | 安全检测服务，使用变更检测功能 |

---

## 依赖与外部交互

### 1. 与 managedEnvConstants.ts 的交互

```typescript
import {
  DANGEROUS_SHELL_SETTINGS,  // 危险 shell 设置键名列表
  SAFE_ENV_VARS,             // 安全环境变量白名单
} from '../../utils/managedEnvConstants.js'
```

**DANGEROUS_SHELL_SETTINGS 详细说明**：

| 设置键 | 风险描述 |
|--------|----------|
| `apiKeyHelper` | 执行任意脚本获取 API 密钥，可能导致密钥泄露 |
| `awsAuthRefresh` | 执行 AWS 认证刷新命令，可能泄露 AWS 凭证 |
| `awsCredentialExport` | 导出 AWS 凭证，可能导致凭证泄露 |
| `gcpAuthRefresh` | 执行 GCP 认证刷新，可能泄露 GCP 凭证 |
| `otelHeadersHelper` | 执行脚本获取 OTel 头，可能泄露遥测端点信息 |
| `statusLine` | 执行状态栏命令，可能泄露系统信息 |

**SAFE_ENV_VARS 白名单分类**：

| 类别 | 示例变量 |
|------|----------|
| 模型配置 | `ANTHROPIC_MODEL`, `ANTHROPIC_DEFAULT_*_MODEL` |
| 提供商选择 | `CLAUDE_CODE_USE_BEDROCK`, `CLAUDE_CODE_USE_VERTEX` |
| 功能开关 | `CLAUDE_CODE_ENABLE_*`, `DISABLE_*` |
| 超时配置 | `BASH_*_TIMEOUT_MS`, `MCP_TIMEOUT` |
| 遥测配置 | `OTEL_*`（除端点外的配置） |
| AWS/GCP 区域 | `AWS_REGION`, `VERTEX_REGION_*` |

### 2. 与 slowOperations.ts 的交互

```typescript
import { jsonStringify } from '../../utils/slowOperations.js'
```

`jsonStringify` 特点：
- 包装了 `JSON.stringify`
- 添加了性能监控（使用 `slowLogging`）
- 当操作超过阈值时记录慢操作日志
- 用于变更检测时的对象序列化比较

### 3. 与 SettingsJson 类型的交互

```typescript
import type { SettingsJson } from '../../utils/settings/types.js'
```

`SettingsJson` 包含的字段（部分）：
- `apiKeyHelper?: string`
- `awsAuthRefresh?: string`
- `env?: Record<string, string>`
- `hooks?: HooksSettings`
- ...（还有更多配置项）

---

## 风险、边界与改进建议

### 潜在风险

1. **白名单维护风险**
   - `SAFE_ENV_VARS` 需要持续维护
   - 新增功能可能引入新的环境变量，若未加入白名单会被误判为危险
   - 建议：建立新增环境变量的审查流程

2. **大小写敏感问题**
   - 环境变量比较时统一转为大写：
     ```typescript
     if (!SAFE_ENV_VARS.has(key.toUpperCase()))
     ```
   - 这可能导致某些合法的大小写敏感变量被误判
   - 实际上环境变量在大多数系统中是大小写敏感的

3. **序列化比较局限**
   - 使用 `JSON.stringify` 比较对象
   - 对象键顺序不同会导致误判为变更
   - 例如：`{a:1, b:2}` vs `{b:2, a:1}` 会被认为不同

4. **空值处理**
   - 空字符串值被过滤（`value.length > 0`）
   - 这可能隐藏某些"故意清空"的配置意图

### 边界情况

1. **null/undefined 输入**
   ```typescript
   if (!settings) {
     return {
       shellSettings: {},
       envVars: {},
       hasHooks: false,
     }
   }
   ```
   - 安全处理，返回空对象

2. **空 env 对象**
   - `settings.env` 为空对象 `{}` → 不提取任何变量

3. **hooks 为 null**
   - `settings.hooks = null` → `hasHooks = false`

4. **大写环境变量名**
   - `key.toUpperCase()` 确保与 `SAFE_ENV_VARS`（全大写）匹配

### 改进建议

1. **深度比较替代 JSON 序列化**
   ```typescript
   // 建议：使用深比较而非 JSON 序列化
   import { isEqual } from 'lodash-es'
   
   return !isEqual(
     { shellSettings: oldDangerous.shellSettings, envVars: oldDangerous.envVars, hooks: oldDangerous.hooks },
     { shellSettings: newDangerous.shellSettings, envVars: newDangerous.envVars, hooks: newDangerous.hooks }
   )
   ```

2. **保留大小写敏感性**
   ```typescript
   // 建议：环境变量保持原始大小写比较
   if (!SAFE_ENV_VARS.has(key) && !SAFE_ENV_VARS.has(key.toUpperCase())) {
     envVars[key] = value
   }
   ```

3. **添加详细日志**
   ```typescript
   export function extractDangerousSettings(
     settings: SettingsJson | null | undefined,
   ): DangerousSettings {
     // ... 添加调试日志
     if (process.env.DEBUG_MANAGED_SETTINGS) {
       console.log('[DangerousSettings] Extracted:', { shellSettings, envVars, hasHooks })
     }
     // ...
   }
   ```

4. **支持部分变更检测**
   ```typescript
   export type DangerousSettingsChangeType = 
     | 'none'
     | 'added'      // 新增危险设置
     | 'removed'    // 移除危险设置
     | 'modified'   // 修改危险设置
   
   export function getDangerousSettingsChangeDetail(
     oldSettings: SettingsJson | null | undefined,
     newSettings: SettingsJson | null | undefined,
   ): { type: DangerousSettingsChangeType; details: string[] }
   ```

5. **环境变量值哈希**
   - 出于安全考虑，当前不返回环境变量值
   - 建议：返回值的哈希，用于变更检测而不暴露实际值

### 测试建议

当前未发现针对此模块的单元测试，建议添加：

1. **正常情况测试**
   - 包含各类危险设置的设置对象
   - 验证正确提取和格式化

2. **边界情况测试**
   - `null/undefined` 输入
   - 空对象输入
   - 空字符串值

3. **变更检测测试**
   - 相同设置比较
   - 键顺序不同但内容相同
   - 新增/删除/修改场景

4. **大小写测试**
   - 大小写不同的环境变量名
   - 验证白名单匹配逻辑
