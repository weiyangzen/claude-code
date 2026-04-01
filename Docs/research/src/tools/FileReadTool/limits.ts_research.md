# limits.ts 研究文档

## 场景与职责

limits.ts 是 FileReadTool 的限制配置模块，负责管理文件读取操作的各项限制参数。它提供了一个灵活的分层配置系统，支持从多个来源获取限制值，并确保合理的默认值。

核心职责：
1. **定义文件读取限制** - 包括 token 数和字节数上限
2. **分层配置优先级** - 环境变量 > GrowthBook 特性标志 > 硬编码默认值
3. **配置验证** - 确保配置值有效（正数、有限值）
4. **性能优化** - 使用 memoization 缓存配置结果

该模块在以下场景发挥关键作用：
- 用户读取大文件时决定是否允许
- 根据实验配置动态调整限制
- 防止超出模型上下文长度
- 控制内存使用

## 功能点目的

### 1. 双限制系统

| 限制 | 默认值 | 检查时机 | 超出处理 |
|------|--------|----------|----------|
| `maxSizeBytes` | 256 KB | 读取前 | 抛出错误，建议使用 offset/limit |
| `maxTokens` | 25000 | 读取后 | 抛出错误，内容不会发送到模型 |

**设计考量**：
- `maxSizeBytes` 基于文件总大小（非读取范围），快速拒绝超大文件
- `maxTokens` 基于实际内容 token 数，精确控制模型上下文

### 2. 分层配置优先级

```
maxTokens 优先级（从高到低）：
1. CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS 环境变量
2. GrowthBook 特性标志 tengu_amber_wren
3. 硬编码默认值 DEFAULT_MAX_OUTPUT_TOKENS (25000)

maxSizeBytes 优先级：
1. GrowthBook 特性标志 tengu_amber_wren
2. MAX_OUTPUT_SIZE (0.25 MB)
```

### 3. 提示词控制

通过 GrowthBook 配置控制提示词内容：
- `includeMaxSizeInPrompt`: 是否在提示词中包含大小限制说明
- `targetedRangeNudge`: 是否提示"仅读取需要的部分"

## 具体技术实现

### 核心数据结构

```typescript
interface FileReadingLimits {
  maxTokens: number        // 最大输出 token 数
  maxSizeBytes: number     // 最大文件大小（字节）
  includeMaxSizeInPrompt?: boolean  // 提示词中包含大小限制
  targetedRangeNudge?: boolean      // 提示词中建议针对性读取
}
```

### 关键流程

#### 1. 获取环境变量配置

```typescript
function getEnvMaxTokens(): number | undefined {
  const override = process.env.CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS
  if (override) {
    const parsed = parseInt(override, 10)
    if (!isNaN(parsed) && parsed > 0) {
      return parsed
    }
  }
  return undefined
}
```

#### 2. 获取默认限制（Memoized）

```typescript
export const getDefaultFileReadingLimits = memoize((): FileReadingLimits => {
  // 1. 获取 GrowthBook 配置
  const override = getFeatureValue_CACHED_MAY_BE_STALE<Partial<FileReadingLimits> | null>(
    'tengu_amber_wren',
    {},
  )

  // 2. 计算 maxSizeBytes（GrowthBook > 默认值）
  const maxSizeBytes = validateNumber(override?.maxSizeBytes) 
    ?? MAX_OUTPUT_SIZE

  // 3. 计算 maxTokens（环境变量 > GrowthBook > 默认值）
  const envMaxTokens = getEnvMaxTokens()
  const maxTokens = envMaxTokens 
    ?? validateNumber(override?.maxTokens) 
    ?? DEFAULT_MAX_OUTPUT_TOKENS

  // 4. 计算提示词控制标志
  const includeMaxSizeInPrompt = override?.includeMaxSizeInPrompt
  const targetedRangeNudge = override?.targetedRangeNudge

  return { maxSizeBytes, maxTokens, includeMaxSizeInPrompt, targetedRangeNudge }
})
```

### 关键代码路径

| 功能 | 代码路径 | 行号 |
|------|----------|------|
| 默认 token 数 | `DEFAULT_MAX_OUTPUT_TOKENS` | 18 |
| 环境变量读取 | `getEnvMaxTokens()` | 24-33 |
| 限制类型定义 | `FileReadingLimits` | 35-40 |
| 获取默认限制 | `getDefaultFileReadingLimits()` | 53-92 |

### 依赖与外部交互

#### 直接依赖模块

| 模块 | 用途 |
|------|------|
| `lodash-es/memoize.js` | 缓存配置结果，避免重复计算 |
| `src/services/analytics/growthbook.js` | `getFeatureValue_CACHED_MAY_BE_STALE` 获取特性标志 |
| `src/utils/file.js` | `MAX_OUTPUT_SIZE` 常量（0.25 MB）|

#### 配置来源

| 来源 | 配置项 | 说明 |
|------|--------|------|
| 环境变量 | `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` | 用户级覆盖 |
| GrowthBook | `tengu_amber_wren` | 实验配置，支持 maxTokens/maxSizeBytes/includeMaxSizeInPrompt/targetedRangeNudge |
| 代码常量 | `DEFAULT_MAX_OUTPUT_TOKENS`, `MAX_OUTPUT_SIZE` | 最终回退 |

### 配置验证逻辑

```typescript
// 验证数值是否有效
function validateNumber(value: unknown): number | undefined {
  if (
    typeof value === 'number' &&
    Number.isFinite(value) &&
    value > 0
  ) {
    return value
  }
  return undefined
}
```

验证规则：
- 类型必须是 number
- 必须是有限值（非 Infinity, -Infinity, NaN）
- 必须大于 0

## 风险、边界与改进建议

### 已知风险

1. **maxSizeBytes 检查粒度问题**
   - 当前：基于文件总大小，而非实际读取范围
   - 影响：即使只读取 10 行，100MB 文件也会被拒绝
   - 历史：曾尝试改为截断而非报错，但导致平均 token 消耗上升，已回滚

2. **GrowthBook 缓存延迟**
   - 使用 `CACHED_MAY_BE_STALE` 后缀，表示可能返回缓存值
   - 影响：特性标志变更后，配置不会立即生效
   - 缓解：memoization 在首次调用后固定，避免会话中配置变化

3. **环境变量解析严格**
   - 当前：`parseInt` 解析，非数字字符串返回 undefined
   - 风险：用户设置 `25000tokens` 会被静默忽略
   - 建议：增加日志或警告提示无效配置

4. **Token 预估不准确**
   - `maxTokens` 在读取后通过 API 精确计算
   - 风险：大文件读取后才发现超出限制，浪费 I/O
   - 缓解：FileReadTool.ts 中有预检查（roughTokenCountEstimationForFileType）

### 边界情况

| 场景 | 处理方式 |
|------|----------|
| 环境变量为负数 | 视为无效，使用下一优先级 |
| 环境变量为 0 | 视为无效，使用下一优先级 |
| GrowthBook 返回 null | 使用默认值 |
| GrowthBook 返回非数字 | 使用默认值 |
| 所有来源都无效 | 使用硬编码默认值 |

### 改进建议

1. **配置热更新**
   - 当前：memoization 在进程生命周期内固定
   - 建议：增加 `resetFileReadingLimits()` 函数，支持配置刷新
   - 用途：长会话中动态调整限制

2. **更灵活的大小检查**
   - 建议：对于显式 offset/limit 的请求，基于读取范围而非文件总大小
   - 风险：需要准确预估 token 数，避免读取后超出限制

3. **配置验证日志**
   - 建议：在调试模式下记录配置来源和最终值
   - 示例：`logDebug('File reading limits: tokens=25000 (from env), size=262144 (from default)')`

4. **用户反馈**
   - 建议：当配置被忽略时，向用户显示警告
   - 示例："Warning: CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS='invalid' is not a valid number"

5. **动态限制调整**
   - 建议：根据模型类型自动调整限制
   - 示例：Claude 3 Opus 支持更大上下文，可提高 maxTokens

6. **限制说明优化**
   - 当前：includeMaxSizeInPrompt 是布尔值
   - 建议：支持自定义提示词模板，更灵活

### 与 FileReadTool 的协作

limits.ts 被 FileReadTool.ts 在以下位置使用：

1. **提示词生成** (`prompt()` 方法)
   ```typescript
   const limits = getDefaultFileReadingLimits()
   const maxSizeInstruction = limits.includeMaxSizeInPrompt
     ? `. Files larger than ${formatFileSize(limits.maxSizeBytes)} will return an error...`
     : ''
   ```

2. **调用时限制获取** (`call()` 方法)
   ```typescript
   const defaults = getDefaultFileReadingLimits()
   const maxSizeBytes = fileReadingLimits?.maxSizeBytes ?? defaults.maxSizeBytes
   const maxTokens = fileReadingLimits?.maxTokens ?? defaults.maxTokens
   ```

3. **图片读取** (`readImageWithTokenBudget()` 函数)
   ```typescript
   maxTokens: number = getDefaultFileReadingLimits().maxTokens
   ```

### 测试注意事项

- 需要 mock GrowthBook 服务以测试特性标志
- 需要清理/重置 memoization 缓存以测试不同配置
- 环境变量测试需要在隔离进程中运行
- 边界值测试：0, -1, NaN, Infinity, 极大值

### 相关文件

| 文件 | 关系 |
|------|------|
| `src/tools/FileReadTool/FileReadTool.ts` | 主要调用方 |
| `src/services/analytics/growthbook.js` | 特性标志服务 |
| `src/utils/file.js` | MAX_OUTPUT_SIZE 常量 |
