# modelAllowlist.ts 深度研究文档

## 场景与职责

本模块负责**模型访问控制的白名单（allowlist）机制**。管理员可以通过设置 `availableModels` 配置来限制用户可以选择的模型范围，实现细粒度的模型访问控制。支持家族别名、版本前缀和完整模型 ID 三级匹配策略。

**核心职责：**
- 检查模型是否在允许列表中
- 支持家族别名通配（如 `opus` 允许所有 Opus 模型）
- 支持版本前缀匹配（如 `opus-4-5` 匹配该版本的所有构建）
- 处理家族别名与具体版本的优先级关系

## 功能点目的

### 1. 企业级访问控制

组织管理员可能希望：
- 限制用户只能使用特定模型（如仅 Sonnet，禁用 Opus）
- 控制成本（禁用高成本的 Opus 模型）
- 确保合规性（仅使用经过审核的模型版本）

### 2. 三级匹配策略

| 级别 | 示例 | 匹配范围 |
|-----|------|---------|
| 家族别名 | `opus`, `sonnet`, `haiku` | 该家族的所有模型版本 |
| 版本前缀 | `opus-4-5`, `claude-opus-4-5` | 该版本的所有构建 |
| 完整 ID | `claude-opus-4-5-20251101` | 仅该精确模型 |

### 3. 优先级处理

当 allowlist 同时包含家族别名和具体版本时，具体版本优先：
- `["opus", "opus-4-5"]` → 仅允许 Opus 4.5（家族别名被具体版本"收窄"）
- `["opus"]` → 允许所有 Opus 模型

## 具体技术实现

### 核心函数

```typescript
// 行 100-170
export function isModelAllowed(model: string): boolean {
  const settings = getSettings_DEPRECATED() || {}
  const { availableModels } = settings
  
  if (!availableModels) return true  // 无限制
  if (availableModels.length === 0) return false  // 空列表阻止所有模型

  // 规范化处理
  const resolvedModel = resolveOverriddenModel(model)
  const normalizedModel = resolvedModel.trim().toLowerCase()
  const normalizedAllowlist = availableModels.map(m => m.trim().toLowerCase())

  // 1. 直接匹配
  if (normalizedAllowlist.includes(normalizedModel)) {
    if (
      !isModelFamilyAlias(normalizedModel) ||
      !familyHasSpecificEntries(normalizedModel, normalizedAllowlist)
    ) {
      return true
    }
  }

  // 2. 家族别名匹配（无具体版本时）
  for (const entry of normalizedAllowlist) {
    if (
      isModelFamilyAlias(entry) &&
      !familyHasSpecificEntries(entry, normalizedAllowlist) &&
      modelBelongsToFamily(normalizedModel, entry)
    ) {
      return true
    }
  }

  // 3. 别名解析匹配
  if (isModelAlias(normalizedModel)) {
    const resolved = parseUserSpecifiedModel(normalizedModel).toLowerCase()
    if (normalizedAllowlist.includes(resolved)) return true
  }

  // 4. 版本前缀匹配
  for (const entry of normalizedAllowlist) {
    if (!isModelFamilyAlias(entry) && !isModelAlias(entry)) {
      if (modelMatchesVersionPrefix(normalizedModel, entry)) return true
    }
  }

  return false
}
```

### 家族成员检测

```typescript
// 行 10-20
function modelBelongsToFamily(model: string, family: string): boolean {
  if (model.includes(family)) return true
  // 解析别名后再次检查
  if (isModelAlias(model)) {
    const resolved = parseUserSpecifiedModel(model).toLowerCase()
    return resolved.includes(family)
  }
  return false
}
```

### 版本前缀匹配

```typescript
// 行 27-32
function prefixMatchesModel(modelName: string, prefix: string): boolean {
  if (!modelName.startsWith(prefix)) return false
  // 前缀必须匹配到段边界（- 或结尾）
  return modelName.length === prefix.length || modelName[prefix.length] === '-'
}

// 行 39-57
function modelMatchesVersionPrefix(model: string, entry: string): boolean {
  // 解析别名
  const resolvedModel = isModelAlias(model)
    ? parseUserSpecifiedModel(model).toLowerCase()
    : model

  // 尝试直接匹配
  if (prefixMatchesModel(resolvedModel, entry)) return true
  
  // 尝试添加 "claude-" 前缀
  if (!entry.startsWith('claude-')) {
    if (prefixMatchesModel(resolvedModel, `claude-${entry}`)) return true
  }
  return false
}
```

### 家族收窄检测

```typescript
// 行 65-87
function familyHasSpecificEntries(
  family: string,
  allowlist: string[],
): boolean {
  for (const entry of allowlist) {
    if (isModelFamilyAlias(entry)) continue  // 跳过其他家族别名
    
    // 检查 entry 是否是该家族的版本限定变体
    const idx = entry.indexOf(family)
    if (idx === -1) continue
    
    const afterFamily = idx + family.length
    // 必须在段边界匹配（避免 "opusplan" 匹配 "opus"）
    if (afterFamily === entry.length || entry[afterFamily] === '-') {
      return true
    }
  }
  return false
}
```

## 关键代码路径与文件引用

### 依赖文件

| 文件路径 | 依赖内容 |
|---------|---------|
| `src/utils/settings/settings.ts` | `getSettings_DEPRECATED()` - 获取用户设置 |
| `src/utils/model/aliases.ts` | `isModelAlias()`, `isModelFamilyAlias()` - 别名检测 |
| `src/utils/model/model.ts` | `parseUserSpecifiedModel()` - 别名解析 |
| `src/utils/model/modelStrings.ts` | `resolveOverriddenModel()` - 模型 ID 解析 |

### 被调用方

| 文件路径 | 使用场景 |
|---------|---------|
| `src/utils/model/model.ts` | `getUserSpecifiedModelSetting()` - 过滤用户指定模型 |
| `src/utils/model/modelOptions.ts` | `filterModelOptionsByAllowlist()` - 过滤模型选项 |
| `src/utils/model/validateModel.ts` | 验证模型前的 allowlist 检查 |

## 依赖与外部交互

### 设置结构

allowlist 通过 `settings.json` 中的 `availableModels` 数组配置：

```json
{
  "availableModels": ["sonnet", "haiku"]
}
```

### 匹配示例

| Allowlist | 用户输入 | 结果 | 原因 |
|-----------|---------|------|------|
| `["opus"]` | `opus` | ✅ 允许 | 家族别名匹配 |
| `["opus"]` | `claude-opus-4-6` | ✅ 允许 | 属于 opus 家族 |
| `["opus", "opus-4-5"]` | `opus` | ❌ 拒绝 | 家族被具体版本收窄 |
| `["opus-4-5"]` | `claude-opus-4-5-20251101` | ✅ 允许 | 版本前缀匹配 |
| `["opus-4-5"]` | `claude-opus-4-6` | ❌ 拒绝 | 版本不匹配 |
| `["sonnet"]` | `opus` | ❌ 拒绝 | 不属于 sonnet 家族 |

## 风险、边界与改进建议

### 风险点

1. **大小写敏感**: 所有匹配都转为小写处理，但自定义模型 ID（如 Azure Foundry）可能对大小写敏感

2. **别名解析递归**: `modelBelongsToFamily` 和 `modelMatchesVersionPrefix` 都调用 `parseUserSpecifiedModel`，可能产生循环依赖风险

3. **段边界匹配歧义**: `opus-4` 应该匹配 `opus-4-5` 还是 `opus-4-6`？当前实现会匹配两者

4. **性能问题**: 每次模型检查需要遍历 allowlist 多次（直接匹配、家族匹配、前缀匹配）

### 边界情况

| 场景 | 行为 |
|-----|------|
| `availableModels` 未设置 | 允许所有模型 |
| `availableModels` 为空数组 | 阻止所有用户指定模型 |
| 模型是别名但 allowlist 包含解析后的 ID | 允许（双向解析） |
| allowlist 包含别名但用户输入完整 ID | 允许（如果解析后匹配） |
| `opusplan` 匹配 `opus` 家族 | 不会（段边界检查阻止） |

### 改进建议

1. **缓存机制**: 缓存 allowlist 的解析结果，避免重复计算

2. **明确优先级文档**: 提供清晰的文档说明匹配优先级和收窄行为

3. **通配符支持**: 考虑支持正则或通配符模式（如 `opus-4.*`）

4. **否定模式**: 支持排除特定模型（如 `["opus", "!opus-4-0"]`）

5. **验证工具**: 提供 CLI 工具验证 allowlist 配置是否按预期工作

```typescript
// 建议：缓存优化
const allowlistCache = new Map<string, Set<string>>()

export function isModelAllowed(model: string): boolean {
  const settings = getSettings_DEPRECATED()
  const cacheKey = JSON.stringify(settings?.availableModels)
  
  let allowedSet = allowlistCache.get(cacheKey)
  if (!allowedSet) {
    allowedSet = precomputeAllowedModels(settings?.availableModels)
    allowlistCache.set(cacheKey, allowedSet)
  }
  
  return allowedSet.has(model.toLowerCase())
}
```
