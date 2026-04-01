# modelCapabilities.ts 深度研究文档

## 场景与职责

本模块负责**从 Anthropic API 获取和缓存模型能力信息**。模型能力（如最大输入 token 数、最大输出 token 数）可能随时间变化，模块通过定期从 API 获取最新信息并本地缓存，确保客户端始终使用准确的模型能力数据。

**核心职责：**
- 从 Anthropic API 获取模型能力列表
- 本地缓存模型能力数据
- 提供同步查询接口供上下文窗口计算使用
- 支持 Ant-only 内部模型的能力获取

## 功能点目的

### 1. 动态能力获取

模型能力（特别是最大上下文窗口）可能：
- 随新模型发布而变化
- 因实验性功能（如 1M 上下文）而扩展
- 在不同账户类型间存在差异

### 2. 本地缓存策略

- 缓存位置: `~/.claude/cache/model-capabilities.json`
- 文件权限: `0o600`（仅用户可读）
- 缓存格式: Zod 验证的 JSON 结构

### 3. 匹配策略

使用最长 ID 优先的子串匹配：
- 优先匹配完整模型 ID
- 回退到子串匹配（如 `claude-opus-4-6` 匹配包含该字符串的模型）

## 具体技术实现

### 数据结构

```typescript
// 行 19-27
const ModelCapabilitySchema = z.object({
  id: z.string(),
  max_input_tokens: z.number().optional(),
  max_tokens: z.number().optional(),
}).strip()  // .strip() 移除内部字段

// 行 29-34
const CacheFileSchema = z.object({
  models: z.array(ModelCapabilitySchema()),
  timestamp: z.number(),
})

export type ModelCapability = z.infer<ReturnType<typeof ModelCapabilitySchema>>
```

### 能力查询

```typescript
// 行 75-83
export function getModelCapability(model: string): ModelCapability | undefined {
  if (!isModelCapabilitiesEligible()) return undefined
  
  const cached = loadCache(getCachePath())
  if (!cached || cached.length === 0) return undefined
  
  const m = model.toLowerCase()
  
  // 1. 精确匹配
  const exact = cached.find(c => c.id.toLowerCase() === m)
  if (exact) return exact
  
  // 2. 子串匹配（最长 ID 优先）
  return cached.find(c => m.includes(c.id.toLowerCase()))
}
```

### 缓存加载

```typescript
// 行 61-73
const loadCache = memoize(
  (path: string): ModelCapability[] | null => {
    try {
      const raw = readFileSync(path, 'utf-8')
      const parsed = CacheFileSchema().safeParse(safeParseJSON(raw, false))
      return parsed.success ? parsed.data.models : null
    } catch {
      return null
    }
  },
  path => path,
)
```

### 缓存刷新

```typescript
// 行 85-118
export async function refreshModelCapabilities(): Promise<void> {
  if (!isModelCapabilitiesEligible()) return
  if (isEssentialTrafficOnly()) return  // 隐私模式跳过

  try {
    const anthropic = await getAnthropicClient({ maxRetries: 1 })
    const betas = isClaudeAISubscriber() ? [OAUTH_BETA_HEADER] : undefined
    
    const parsed: ModelCapability[] = []
    for await (const entry of anthropic.models.list({ betas })) {
      const result = ModelCapabilitySchema().safeParse(entry)
      if (result.success) parsed.push(result.data)
    }
    
    if (parsed.length === 0) return

    const path = getCachePath()
    const models = sortForMatching(parsed)  // 最长 ID 优先排序
    
    // 如果缓存未变化，跳过写入
    if (isEqual(loadCache(path), models)) {
      logForDebugging('[modelCapabilities] cache unchanged, skipping write')
      return
    }

    await mkdir(getCacheDir(), { recursive: true })
    await writeFile(path, jsonStringify({ models, timestamp: Date.now() }), {
      encoding: 'utf-8',
      mode: 0o600,
    })
    loadCache.cache.delete(path)  // 清除 memoize 缓存
    logForDebugging(`[modelCapabilities] cached ${models.length} models`)
  } catch (error) {
    logForDebugging(`[modelCapabilities] fetch failed: ${error}`)
  }
}
```

### 资格检查

```typescript
// 行 46-51
function isModelCapabilitiesEligible(): boolean {
  if (process.env.USER_TYPE !== 'ant') return false      // 仅 Ant 用户
  if (getAPIProvider() !== 'firstParty') return false    // 仅 firstParty
  if (!isFirstPartyAnthropicBaseUrl()) return false      // 仅官方 API
  return true
}
```

## 关键代码路径与文件引用

### 依赖文件

| 文件路径 | 依赖内容 |
|---------|---------|
| `src/services/api/client.ts` | `getAnthropicClient()` - Anthropic API 客户端 |
| `src/constants/oauth.ts` | `OAUTH_BETA_HEADER` - OAuth Beta 头 |
| `src/utils/auth.ts` | `isClaudeAISubscriber()` - 订阅者检测 |
| `src/utils/envUtils.ts` | `getClaudeConfigHomeDir()` - 配置目录 |
| `src/utils/privacyLevel.ts` | `isEssentialTrafficOnly()` - 隐私模式检测 |
| `src/utils/lazySchema.ts` | `lazySchema()` - 延迟 Zod schema 创建 |

### 被调用方

| 文件路径 | 使用场景 |
|---------|---------|
| `src/utils/context.ts` | `getContextWindowForModel()` - 获取模型上下文窗口大小 |
| `src/utils/model/model.ts` | `getModelMaxOutputTokens()` - 获取最大输出 token 数 |

### 调用链示例

```
getContextWindowForModel('claude-opus-4-6')
    ↓
getModelCapability('claude-opus-4-6')
    ↓
isModelCapabilitiesEligible() → true (Ant + firstParty)
    ↓
loadCache('~/.claude/cache/model-capabilities.json')
    ↓
精确匹配? → 返回 { id, max_input_tokens, max_tokens }
    ↓
未匹配 → 返回 undefined → 使用硬编码默认值
```

## 依赖与外部交互

### API 端点

使用 Anthropic SDK 的 `anthropic.models.list()` 端点：
- 需要 OAuth Beta 头（订阅者）
- 返回模型列表及其能力信息
- 分页获取（使用 `for await...of`）

### 缓存文件格式

```json
{
  "models": [
    {
      "id": "claude-opus-4-6",
      "max_input_tokens": 200000,
      "max_tokens": 128000
    }
  ],
  "timestamp": 1704067200000
}
```

### 排序策略

```typescript
// 行 54-58
function sortForMatching(models: ModelCapability[]): ModelCapability[] {
  return [...models].sort(
    (a, b) => b.id.length - a.id.length || a.id.localeCompare(b.id),
  )
}
```

最长 ID 优先确保子串匹配时优先选择最具体的模型（如 `claude-opus-4-6` 优先于 `claude-opus-4`）。

## 风险、边界与改进建议

### 风险点

1. **Ant-only 限制**: 功能仅对内部用户启用，外部用户无法受益于动态能力获取

2. **缓存过期**: 没有 TTL 机制，缓存可能长期不更新

3. **API 失败静默**: 获取失败仅记录调试日志，用户无感知

4. **子串匹配歧义**: 多个模型 ID 可能匹配同一子串（如 `claude-opus-4` 匹配 4.0、4.1、4.5、4.6）

5. **隐私模式**: `isEssentialTrafficOnly()` 为 true 时完全跳过刷新，可能导致缓存长期不更新

### 边界情况

| 场景 | 行为 |
|-----|------|
| 非 Ant 用户 | 始终返回 `undefined`，使用硬编码默认值 |
| 3P 提供商 | 返回 `undefined`，使用硬编码默认值 |
| 缓存文件损坏 | 返回 `null`，使用硬编码默认值 |
| API 返回空列表 | 跳过缓存更新，保留旧缓存 |
| 缓存与 API 相同 | 跳过写入，减少磁盘 I/O |
| 模型不在缓存中 | 返回 `undefined`，使用硬编码默认值 |

### 改进建议

1. **扩大可用性**: 考虑向所有用户开放，不仅限于 Ant 用户

2. **TTL 机制**: 添加缓存过期时间，强制定期刷新

```typescript
const CACHE_TTL_MS = 24 * 60 * 60 * 1000  // 24 小时

function isCacheStale(timestamp: number): boolean {
  return Date.now() - timestamp > CACHE_TTL_MS
}
```

3. **后台刷新**: 在应用启动时后台刷新缓存，避免阻塞用户操作

4. **匹配算法改进**: 考虑使用更精确的模式匹配（如正则表达式）替代子串匹配

5. **错误处理增强**: API 失败时向用户显示警告，提示能力信息可能过期

6. **缓存预热**: 预置常见模型的能力信息，减少对 API 的依赖
