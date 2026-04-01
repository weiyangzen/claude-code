# api.ts 研究文档

## 场景与职责

`api.ts` 是 Claude Code CLI 与 Anthropic API 交互的核心工具模块，负责：

1. **工具模式转换**：将内部 Tool 对象转换为 Anthropic API 的 BetaTool 格式
2. **系统提示词管理**：分割和管理系统提示词块，支持缓存控制
3. **工具输入规范化**：为特定工具（Bash、FileEdit、FileWrite 等）预处理输入
4. **上下文指标收集**：收集和上报会话上下文大小、工具数量等分析数据

## 功能点目的

### 1. 工具模式转换 (`toolToAPISchema`)
- 将内部 Tool 定义转换为 Anthropic API 格式
- 支持工具模式缓存（通过 `toolSchemaCache.ts`）
- 支持特性开关控制（strict 模式、细粒度工具流）
- 支持延迟加载标记（`defer_loading`）
- 支持缓存控制（`cache_control`）

### 2. Swarm 字段过滤 (`filterSwarmFieldsFromSchema`)
- 当 Swarm 功能未启用时，从工具模式中移除相关字段
- 防止外部用户看到未发布的功能

### 3. 系统提示词分割 (`splitSysPromptPrefix`)
- 根据特性开关将系统提示词分割为多个块
- 支持全局缓存（global scope）和组织级缓存（org scope）
- 处理动态边界标记（`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`）

### 4. 工具输入规范化 (`normalizeToolInput`)
- BashTool：规范化命令（去除 cwd 前缀、处理 Windows 路径）
- FileEditTool：规范化文件编辑输入
- FileWriteTool：去除尾随空白（Markdown 除外）
- ExitPlanModeV2：注入计划内容和文件路径
- TaskOutputTool：处理遗留参数名

### 5. API 工具输入清理 (`normalizeToolInputForAPI`)
- 移除 `normalizeToolInput` 添加的额外字段
- 确保发送到 API 的数据符合模式要求

### 6. 上下文指标收集 (`logContextMetrics`)
- 收集 git 状态、claude.md 大小
- 统计 MCP 工具和非 MCP 工具数量
- 估算工具 token 使用量

## 具体技术实现

### 关键数据结构

```typescript
// 带额外字段的 BetaTool 类型
type BetaToolWithExtras = BetaTool & {
  strict?: boolean
  defer_loading?: boolean
  cache_control?: {
    type: 'ephemeral'
    scope?: 'global' | 'org'
    ttl?: '5m' | '1h'
  }
  eager_input_streaming?: boolean
}

// 缓存范围
type CacheScope = 'global' | 'org'

// 系统提示词块
type SystemPromptBlock = {
  text: string
  cacheScope: CacheScope | null
}
```

### 工具模式缓存策略

```typescript
// 缓存键生成（考虑 inputJSONSchema）
const cacheKey =
  'inputJSONSchema' in tool && tool.inputJSONSchema
    ? `${tool.name}:${jsonStringify(tool.inputJSONSchema)}`
    : tool.name

// 会话级缓存防止 mid-session GrowthBook 翻转
const cache = getToolSchemaCache()
let base = cache.get(cacheKey)
if (!base) {
  // 计算基础模式（一次性）
  // ...
  cache.set(cacheKey, base)
}

// 每次请求叠加可变字段
cache.defer_loading = options.deferLoading
cache.cache_control = options.cacheControl
```

### 系统提示词分割策略

1. **MCP 工具存在时**（`skipGlobalCacheForSystemPrompt=true`）：
   - 归因头（无缓存）
   - 系统提示词前缀（org 缓存）
   - 其余内容（org 缓存）

2. **全局缓存模式**（找到边界标记）：
   - 归因头（无缓存）
   - 系统提示词前缀（无缓存）
   - 静态内容（global 缓存）
   - 动态内容（无缓存）

3. **默认模式**：
   - 归因头（无缓存）
   - 系统提示词前缀（org 缓存）
   - 其余内容（org 缓存）

### 工具输入规范化示例

```typescript
case BashTool.name: {
  const parsed = BashTool.inputSchema.parse(input)
  const { command, timeout, description } = parsed
  const cwd = getCwd()
  
  // 移除 cwd 前缀（如果存在）
  let normalizedCommand = command.replace(`cd ${cwd} && `, '')
  
  // Windows 路径处理
  if (getPlatform() === 'windows') {
    normalizedCommand = normalizedCommand.replace(
      `cd ${windowsPathToPosixPath(cwd)} && `,
      '',
    )
  }
  
  // 处理转义分号
  normalizedCommand = normalizedCommand.replace(/\\\\;/g, '\\;')
  
  return { command: normalizedCommand, ... }
}
```

## 关键代码路径与文件引用

### 本文件导出
- `toolToAPISchema(tool, options)` - 工具模式转换
- `splitSysPromptPrefix(systemPrompt, options?)` - 系统提示词分割
- `appendSystemContext(systemPrompt, context)` - 追加系统上下文
- `prependUserContext(messages, context)` - 前置用户上下文
- `logContextMetrics(mcpConfigs, toolPermissionContext)` - 上下文指标
- `normalizeToolInput(tool, input, agentId?)` - 工具输入规范化
- `normalizeToolInputForAPI(tool, input)` - API 工具输入清理
- `logAPIPrefix(systemPrompt)` - 记录系统提示词前缀

### 依赖文件
| 文件 | 用途 |
|------|------|
| `src/utils/toolSchemaCache.ts` | 工具模式缓存 |
| `src/utils/betas.ts` | 特性开关检查 |
| `src/utils/agentSwarmsEnabled.ts` | Swarm 功能检测 |
| `src/utils/model/providers.ts` | API 提供商检测 |
| `src/utils/envUtils.ts` | 环境变量工具 |
| `src/utils/zodToJsonSchema.ts` | Zod 模式转换 |
| `src/utils/plans.ts` | 计划文件操作 |
| `src/utils/permissions/filesystem.ts` | 文件权限模式 |
| `src/utils/ripgrep.ts` | 文件计数 |
| `src/services/analytics/growthbook.ts` | 特性开关 |
| `src/services/analytics/index.ts` | 事件上报 |
| `src/services/mcp/client.ts` | MCP 资源预取 |
| `src/tools/BashTool/BashTool.ts` | Bash 工具定义 |
| `src/tools/FileEditTool/FileEditTool.ts` | 文件编辑工具 |
| `src/tools/FileWriteTool/FileWriteTool.ts` | 文件写入工具 |
| `src/constants/system.ts` | 系统提示词前缀常量 |
| `src/constants/prompts.ts` | 动态边界常量 |

### 调用方
- `src/utils/betas.ts` - 导入 `modelSupportsStructuredOutputs`
- `src/utils/toolSchemaCache.ts` - 相关缓存逻辑
- `src/commands/remote-setup/remote-setup.tsx` - 远程设置

## 依赖与外部交互

### 外部 npm 依赖
- `@anthropic-ai/sdk` - Anthropic API SDK
- `zod/v4` - 模式验证

### 内部依赖
- 工具系统（`src/tools/*`）
- 分析系统（`src/services/analytics/*`）
- MCP 系统（`src/services/mcp/*`）
- 配置系统（`src/utils/config.ts`）

## 风险、边界与改进建议

### 已知限制
1. **缓存键复杂度**：包含 `inputJSONSchema` 的缓存键可能很长
2. **类型安全**：`normalizeToolInput` 使用类型断言绕过 TypeScript 限制
3. **同步/异步混合**：部分操作同步，部分异步，调用方需注意

### 安全风险
1. **命令注入**：Bash 工具规范化需确保不会引入新的注入向量
2. **路径遍历**：文件路径处理需验证安全性

### 性能考虑
1. **缓存命中率**：工具模式缓存减少重复计算
2. **ripgrep 调用**：`logContextMetrics` 中的文件计数可能较慢（1秒超时）

### 改进建议
1. 将 `normalizeToolInput` 拆分为每个工具的独立函数
2. 添加更完善的错误处理和重试机制
3. 考虑将上下文指标收集改为采样模式
4. 添加工具模式缓存命中率监控

### 测试建议
1. 测试各种 ANSI 颜色组合的解析
2. 测试系统提示词分割的边界条件
3. 测试工具输入规范化的各种输入格式
4. 测试缓存行为（命中、未命中、过期）
