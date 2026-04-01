# modelStrings.ts 研究文档

## 场景与职责

`modelStrings.ts` 负责管理 Claude Code 中各平台特定的模型 ID 字符串。这是模型系统的关键适配层：

1. **平台特定模型 ID**：为每个模型版本提供 First-Party、Bedrock、Vertex、Foundry 的模型 ID
2. **Bedrock 推理配置文件发现**：动态获取用户 AWS 账户中可用的推理配置文件
3. **用户模型覆盖**：支持用户通过设置覆盖默认模型 ID
4. **延迟初始化**：Bedrock 环境下异步获取推理配置文件，避免阻塞启动

## 功能点目的

### 1. 模型字符串获取
- `getModelStrings()`: 获取当前平台的模型字符串映射
  - 同步返回，已处理覆盖
  - Bedrock 环境下可能返回临时默认值

### 2. Bedrock 推理配置文件集成
- `getBedrockModelStrings()`: 从 AWS Bedrock 获取可用的推理配置文件
  - 调用 `getBedrockInferenceProfiles()` 获取配置文件列表
  - 匹配 `ALL_MODEL_CONFIGS` 中的模型
  - 失败时回退到硬编码配置

### 3. 用户覆盖应用
- `applyModelOverrides()`: 应用用户设置的模型覆盖
  - 从 `settings.json` 的 `modelOverrides` 读取
  - 支持将 Bedrock ARN 映射到标准模型

### 4. 覆盖反向解析
- `resolveOverriddenModel()`: 将覆盖后的模型 ID 解析回规范名称
  - 用于 allowlist 验证
  - 安全地在模块初始化时调用

### 5. 初始化控制
- `initModelStrings()`: 初始化模型字符串
- `ensureModelStringsInitialized()`: 确保初始化完成（异步）
- `updateBedrockModelStrings()`: 更新 Bedrock 模型字符串（带顺序控制）

## 具体技术实现

### 模型字符串类型

```typescript
export type ModelStrings = Record<ModelKey, string>
// ModelKey 来自 configs.ts: 'haiku35' | 'haiku45' | 'sonnet35' | ...
```

### 内置模型字符串获取

```typescript
function getBuiltinModelStrings(provider: APIProvider): ModelStrings {
  const out = {} as ModelStrings
  for (const key of MODEL_KEYS) {
    out[key] = ALL_MODEL_CONFIGS[key][provider]
  }
  return out
}
```

### Bedrock 推理配置文件匹配

```typescript
async function getBedrockModelStrings(): Promise<ModelStrings> {
  const fallback = getBuiltinModelStrings('bedrock')
  let profiles: string[] | undefined
  
  try {
    profiles = await getBedrockInferenceProfiles()
  } catch (error) {
    logError(error as Error)
    return fallback
  }
  
  if (!profiles?.length) {
    return fallback
  }

  // 使用 firstParty ID 作为搜索关键词匹配推理配置文件
  const out = {} as ModelStrings
  for (const key of MODEL_KEYS) {
    const needle = ALL_MODEL_CONFIGS[key].firstParty  // e.g., "claude-opus-4-6"
    out[key] = findFirstMatch(profiles, needle) || fallback[key]
  }
  return out
}
```

### 用户覆盖应用

```typescript
function applyModelOverrides(ms: ModelStrings): ModelStrings {
  const overrides = getInitialSettings().modelOverrides
  if (!overrides) return ms

  const out = { ...ms }
  for (const [canonicalId, override] of Object.entries(overrides)) {
    const key = CANONICAL_ID_TO_KEY[canonicalId as CanonicalModelId]
    if (key && override) {
      out[key] = override
    }
  }
  return out
}
```

覆盖配置示例（`settings.json`）：
```json
{
  "modelOverrides": {
    "claude-opus-4-6": "arn:aws:bedrock:eu-west-1:123456789:inference-profile/eu.anthropic.claude-opus-4-6-v1"
  }
}
```

### 覆盖反向解析

```typescript
export function resolveOverriddenModel(modelId: string): string {
  let overrides: Record<string, string> | undefined
  try {
    overrides = getInitialSettings().modelOverrides
  } catch {
    return modelId
  }
  if (!overrides) return modelId

  for (const [canonicalId, override] of Object.entries(overrides)) {
    if (override === modelId) {
      return canonicalId
    }
  }
  return modelId
}
```

### 顺序控制初始化

```typescript
const updateBedrockModelStrings = sequential(async () => {
  if (getModelStringsState() !== null) {
    // 已初始化，跳过
    return
  }
  try {
    const ms = await getBedrockModelStrings()
    setModelStringsState(ms)
  } catch (error) {
    logError(error as Error)
  }
})

function initModelStrings(): void {
  const ms = getModelStringsState()
  if (ms !== null) return

  // 非 Bedrock 环境：同步初始化
  if (getAPIProvider() !== 'bedrock') {
    setModelStringsState(getBuiltinModelStrings(getAPIProvider()))
    return
  }

  // Bedrock 环境：后台异步更新，不阻塞
  void updateBedrockModelStrings()
}
```

## 关键代码路径与文件引用

### 导出函数
| 函数 | 行号 | 用途 |
|------|------|------|
| `getModelStrings` | 136-145 | 获取模型字符串（主入口）|
| `resolveOverriddenModel` | 84-100 | 反向解析覆盖的模型 |
| `ensureModelStringsInitialized` | 152-166 | 确保初始化完成 |

### 内部函数
| 函数 | 行号 | 用途 |
|------|------|------|
| `getBuiltinModelStrings` | 25-31 | 获取内置模型字符串 |
| `getBedrockModelStrings` | 33-55 | 从 Bedrock 获取 |
| `applyModelOverrides` | 63-76 | 应用用户覆盖 |
| `initModelStrings` | 118-134 | 初始化 |
| `updateBedrockModelStrings` | 102-116 | 更新 Bedrock 字符串 |

### 依赖导入
| 导入 | 来源 | 用途 |
|------|------|------|
| `getModelStringsState`, `setModelStringsState` | `src/bootstrap/state.js` | 状态管理 |
| `logError` | `../log.js` | 错误日志 |
| `sequential` | `../sequential.js` | 顺序控制 |
| `getInitialSettings` | `../settings/settings.js` | 初始设置 |
| `findFirstMatch`, `getBedrockInferenceProfiles` | `./bedrock.js` | Bedrock 集成 |
| `ALL_MODEL_CONFIGS`, `CANONICAL_ID_TO_KEY` | `./configs.js` | 模型配置 |
| `getAPIProvider` | `./providers.js` | API 提供商 |

### 调用方
- `src/utils/model/model.ts` - 获取模型字符串用于解析和显示
- `src/utils/model/modelOptions.ts` - 生成模型选项
- `src/utils/auth.ts` - 获取模型信息
- `src/services/api/client.ts` - 创建 API 客户端

## 依赖与外部交互

### 与 bootstrap/state.ts 的交互
- 使用 `getModelStringsState()` / `setModelStringsState()` 管理状态
- 状态在 `bootstrap/state.ts` 中定义

### 与 bedrock.ts 的交互
- 使用 `getBedrockInferenceProfiles()` 获取推理配置文件
- 使用 `findFirstMatch()` 匹配模型

### 与 configs.ts 的交互
- 使用 `ALL_MODEL_CONFIGS` 获取模型配置
- 使用 `CANONICAL_ID_TO_KEY` 映射规范 ID 到内部键

### 与 settings/settings.ts 的交互
- 使用 `getInitialSettings()` 获取 `modelOverrides`
- 用户可以通过设置覆盖默认模型 ID

## 风险、边界与改进建议

### 风险点

1. **Bedrock 初始化延迟**
   - Bedrock 环境下异步获取推理配置文件
   - 在获取完成前使用硬编码默认值
   - 可能导致短暂的模型 ID 不匹配

2. **推理配置文件匹配失败**
   - `findFirstMatch` 使用子串匹配
   - 可能匹配到错误的配置文件

3. **覆盖配置错误**
   - 用户配置的覆盖可能无效
   - 没有验证覆盖的模型 ID 是否可用

4. **状态同步问题**
   - `sequential` 确保顺序执行，但测试环境可能需要重置
   - `resetModelStringsForTestingOnly()` 仅用于测试

### 边界情况

1. **Bedrock 无可用配置文件**
   - 回退到硬编码配置
   - 可能使用错误的区域前缀

2. **覆盖配置中的无效键**
   - `CANONICAL_ID_TO_KEY[canonicalId]` 可能为 `undefined`
   - 无效键被忽略

3. **设置未加载**
   - `getInitialSettings()` 可能抛出异常
   - `resolveOverriddenModel` 捕获异常并返回原值

4. **并发调用**
   - `sequential` 包装确保多次调用不会导致并发问题
   - 但测试环境需要手动重置状态

### 改进建议

1. **添加验证**
   ```typescript
   function validateModelOverride(canonicalId: string, override: string): boolean {
     // 验证 canonicalId 是否有效
     // 验证 override 格式是否正确（如 ARN 格式）
   }
   ```

2. **改进匹配算法**
   ```typescript
   // 使用更精确的匹配，而非子串匹配
   function findBestMatch(profiles: string[], needle: string): string | null {
     // 优先匹配包含版本号的完整名称
     // 其次匹配基础名称
   }
   ```

3. **添加加载状态**
   ```typescript
   export function getModelStringsLoadingState(): 'loading' | 'complete' | 'error'
   // 让 UI 可以显示加载状态
   ```

4. **缓存推理配置文件**
   ```typescript
   // 将 Bedrock 推理配置文件列表缓存到本地
   // 减少每次启动的 API 调用
   ```

5. **支持通配覆盖**
   ```typescript
   // 支持批量覆盖
   "modelOverrides": {
     "claude-opus-*": "arn:aws:bedrock:.../eu.anthropic.claude-opus-4-*"
   }
   ```

6. **改进错误处理**
   ```typescript
   // 区分网络错误、权限错误、配置错误
   // 向用户显示有用的错误消息
   ```
