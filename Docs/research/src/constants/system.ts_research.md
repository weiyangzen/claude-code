# system.ts 深度研究文档

## 场景与职责

`src/constants/system.ts` 是 Claude Code CLI 的核心系统常量模块，负责管理与系统提示词前缀（System Prompt Prefix）和 API 请求归因头（Attribution Header）相关的关键功能。该模块的设计目标是打破循环依赖（circular dependencies），将原本分散在各处的关键系统级常量集中管理。

**主要使用场景：**
1. **API 请求构建**：为每个发送到 Anthropic API 的请求生成归因头，用于计费追踪、版本识别和客户端验证
2. **系统提示词构建**：根据运行环境（交互式/非交互式、Agent SDK/Vertex 等）选择合适的前缀文本
3. **OAuth 验证支持**：通过指纹计算和客户端认证令牌支持安全的 API 访问

## 功能点目的

### 1. CLI 系统提示词前缀管理

定义了三种不同的系统提示词前缀，用于标识 Claude Code 的运行环境和身份：

- `DEFAULT_PREFIX`: 标准 Claude Code CLI 前缀（"You are Claude Code, Anthropic's official CLI for Claude."）
- `AGENT_SDK_CLAUDE_CODE_PRESET_PREFIX`: Agent SDK 中使用 Claude Code 预设时的前缀
- `AGENT_SDK_PREFIX`: 通用 Agent SDK 前缀（"You are a Claude agent, built on Anthropic's Claude Agent SDK."）

**设计目的**：
- 区分不同的运行模式（直接 CLI 使用 vs Agent SDK 集成）
- 为 API 后端提供客户端类型标识
- 支持 Vertex AI 等第三方平台的特殊处理

### 2. 归因头（Attribution Header）生成

生成 `x-anthropic-billing-header` HTTP 头，包含以下信息：
- `cc_version`: 客户端版本（含指纹）
- `cc_entrypoint`: 入口点标识（如 'cli', 'sdk', 'unknown'）
- `cch`: 客户端认证哈希（可选，用于原生客户端认证）
- `cc_workload`: 工作负载类型（用于 QoS 路由）

**安全机制**：
- 支持通过环境变量 `CLAUDE_CODE_ATTRIBUTION_HEADER` 禁用
- 支持通过 GrowthBook killswitch 远程控制
- 原生客户端认证使用占位符（`cch=00000`），由 Bun 的 HTTP 栈在发送前替换为实际哈希值

## 具体技术实现

### 关键数据结构

```typescript
// 系统提示词前缀类型
export type CLISyspromptPrefix = (typeof CLI_SYSPROMPT_PREFIX_VALUES)[number]

// 前缀集合（用于内容匹配）
export const CLI_SYSPROMPT_PREFIXES: ReadonlySet<string>
```

### 核心函数实现

#### `getCLISyspromptPrefix(options?)`

```typescript
export function getCLISyspromptPrefix(options?: {
  isNonInteractive: boolean
  hasAppendSystemPrompt: boolean
}): CLISyspromptPrefix
```

**逻辑流程**：
1. 检查 API Provider：如果是 Vertex，返回默认前缀
2. 检查是否非交互式模式：
   - 如果有追加系统提示词 → 返回 Agent SDK Claude Code 预设前缀
   - 否则 → 返回通用 Agent SDK 前缀
3. 默认返回标准 CLI 前缀

#### `getAttributionHeader(fingerprint: string)`

```typescript
export function getAttributionHeader(fingerprint: string): string
```

**构建流程**：
1. 检查归因头是否被禁用（环境变量或 GrowthBook）
2. 构造版本字符串：`${MACRO.VERSION}.${fingerprint}`
3. 获取入口点：`process.env.CLAUDE_CODE_ENTRYPOINT ?? 'unknown'`
4. 条件添加 `cch` 占位符（当 `NATIVE_CLIENT_ATTESTATION` 特性启用时）
5. 添加工作负载标识（通过 `getWorkload()` 获取）
6. 返回完整头字符串

**原生客户端认证机制**：
```typescript
const cch = feature('NATIVE_CLIENT_ATTESTATION') ? ' cch=00000;' : ''
```
- 使用固定长度占位符避免 Content-Length 变化
- Bun 的 HTTP 栈在序列化请求体时查找并替换此占位符
- 服务器验证此令牌以确认请求来自真实的 Claude Code 客户端

### 依赖模块

| 模块 | 用途 |
|------|------|
| `bun:bundle` | 特性标志检查（`feature()`） |
| `../services/analytics/growthbook.js` | GrowthBook 功能开关 |
| `../utils/debug.js` | 调试日志 |
| `../utils/envUtils.js` | 环境变量检查 |
| `../utils/model/providers.js` | API Provider 检测 |
| `../utils/workloadContext.js` | 工作负载上下文 |

## 关键代码路径与文件引用

### 调用方分析

| 调用文件 | 调用内容 | 用途 |
|----------|----------|------|
| `src/utils/api.ts` | `CLI_SYSPROMPT_PREFIXES` | 系统提示词前缀分割和识别 |
| `src/services/api/claude.ts` | `getAttributionHeader`, `getCLISyspromptPrefix` | API 请求构建 |
| `src/utils/sideQuery.ts` | `getAttributionHeader`, `getCLISyspromptPrefix` | 侧边查询 API 调用 |
| `src/utils/sessionRestore.ts` | `getCLISyspromptPrefix` | 会话恢复时的前缀匹配 |
| `src/tools/EnterWorktreeTool/EnterWorktreeTool.ts` | `getCLISyspromptPrefix` | 工作树进入时的提示词处理 |
| `src/tools/ExitWorktreeTool/ExitWorktreeTool.ts` | `getCLISyspromptPrefix` | 工作树退出时的提示词处理 |
| `src/services/compact/postCompactCleanup.ts` | 相关常量 | 压缩后的清理操作 |

### 代码示例：API 请求中的使用

```typescript
// src/services/api/claude.ts
import {
  getAttributionHeader,
  getCLISyspromptPrefix,
} from '../../constants/system.js'

// 在构建 API 请求时
const attributionHeader = getAttributionHeader(fingerprint)
const prefix = getCLISyspromptPrefix({
  isNonInteractive: !isInteractive,
  hasAppendSystemPrompt: hasAppendSystemPrompt,
})
```

## 依赖与外部交互

### 外部依赖

1. **Bun 运行时特性**
   - `bun:bundle` 模块提供 `feature()` 函数用于特性开关检查
   - `MACRO.VERSION` 由构建系统注入

2. **GrowthBook 分析服务**
   - `getFeatureValue_CACHED_MAY_BE_STALE()` 用于获取远程配置
   - 支持 `tengu_attribution_header` killswitch

3. **环境变量**
   - `CLAUDE_CODE_ATTRIBUTION_HEADER`: 禁用归因头
   - `CLAUDE_CODE_ENTRYPOINT`: 标识入口点
   - `USER_TYPE`: 影响 Agent 工具权限

### 输出影响

该模块的输出直接影响：
1. **API 请求头**：每个 API 调用都包含归因头
2. **系统提示词结构**：影响模型接收的初始上下文
3. **计费追踪**：版本和指纹用于成本归因
4. **QoS 路由**：工作负载标识影响请求优先级

## 风险、边界与改进建议

### 潜在风险

1. **循环依赖风险**
   - 该模块被设计为"叶子节点"模块，不应依赖其他业务逻辑模块
   - 当前依赖 `growthbook.js`、`envUtils.js` 等工具模块，需保持这些模块的轻量级

2. **指纹计算依赖**
   - `getAttributionHeader` 依赖外部传入的 fingerprint 参数
   - 调用方需确保指纹计算的正确性和一致性

3. **GrowthBook 缓存陈旧**
   - 使用 `getFeatureValue_CACHED_MAY_BE_STALE` 意味着配置变更可能有延迟
   - 归因头开关变更不会立即生效

4. **Vertex 平台特殊处理**
   - Vertex provider 强制使用默认前缀，可能与其他逻辑冲突
   - 需要确保 Vertex 路径的测试覆盖

### 边界情况

1. **空指纹处理**：函数接受任意字符串作为 fingerprint，不验证格式
2. **环境变量优先级**：环境变量可以覆盖 GrowthBook 配置
3. **工作负载标识**：`getWorkload()` 可能返回 undefined，需正确处理

### 改进建议

1. **类型安全增强**
   ```typescript
   // 建议：为 fingerprint 添加品牌类型
   type Fingerprint = string & { __brand: 'Fingerprint' }
   ```

2. **配置验证**
   - 添加对 `CLAUDE_CODE_ENTRYPOINT` 有效值的验证
   - 在开发模式下警告未知的入口点值

3. **测试覆盖**
   - 添加 Vertex provider 路径的单元测试
   - 测试归因头禁用逻辑
   - 测试不同环境变量组合的行为

4. **文档完善**
   - 添加 `MACRO.VERSION` 的注入机制说明
   - 说明 `cch` 占位符的替换机制

5. **性能优化**
   - 考虑缓存 `isAttributionHeaderEnabled()` 的结果
   - 归因头构建涉及多次字符串拼接，可考虑使用模板
