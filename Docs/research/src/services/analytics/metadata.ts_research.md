# metadata.ts 深度研究文档

## 场景与职责

`metadata.ts` 是 Claude Code 分析系统的核心元数据模块，负责为所有分析事件（Datadog 和 1P 事件日志）提供统一、丰富的元数据收集和格式化功能。它是整个分析系统的"单一真相源"（single source of truth），确保所有分析事件都包含一致的环境、运行时和上下文信息。

### 核心职责

1. **事件元数据丰富化**：为所有分析事件添加环境、运行时、用户、模型等核心元数据
2. **PII 数据脱敏**：提供类型安全机制防止敏感数据（代码片段、文件路径）意外进入遥测
3. **工具名称清理**：处理 MCP 工具名称，避免泄露用户特定的服务器配置
4. **文件扩展名提取**：从文件路径和 Bash 命令中提取扩展名用于分析
5. **进程指标收集**：收集内存使用、CPU 使用率等进程级指标
6. **多代理识别**：支持 teammate（集群代理）、subagent（Agent 工具子代理）和 standalone（独立代理）的身份识别

---

## 功能点目的

### 1. PII 安全类型系统

```typescript
// 标记类型：强制开发者显式验证字符串不包含敏感数据
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never
```

**目的**：通过 TypeScript 类型系统在编译期防止敏感数据泄露。任何要记录到遥测的字符串都必须显式类型断言，迫使开发者思考数据敏感性。

### 2. 工具名称脱敏

```typescript
export function sanitizeToolNameForAnalytics(toolName: string): AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
```

**目的**：MCP 工具名称遵循 `mcp__<server>__<tool>` 格式，可能包含用户特定的服务器配置（PII-medium）。此函数将 MCP 工具名脱敏为 `'mcp_tool'`，同时保留内置工具名称（Bash、Read、Write 等）。

### 3. 工具详情日志门控

```typescript
export function isToolDetailsLoggingEnabled(): boolean
export function isAnalyticsToolDetailsLoggingEnabled(mcpServerType?: string, mcpServerBaseUrl?: string): boolean
```

**目的**：根据环境决定是否记录详细的 MCP 服务器/工具名称。允许记录的场景：
- Cowork 模式（entrypoint=local-agent）
- claude.ai 代理连接器（claudeai-proxy）
- 官方 MCP 注册表 URL

### 4. MCP 工具详情提取

```typescript
export function mcpToolDetailsForAnalytics(toolName: string, mcpServerType?: string, mcpServerBaseUrl?: string): { mcpServerName?, mcpToolName? }
export function extractMcpToolDetails(toolName: string): { serverName, mcpToolName } | undefined
```

**目的**：从 `mcp__<server>__<tool>` 格式中提取服务器和工具名，并在门控通过后返回。

### 5. Skill 名称提取

```typescript
export function extractSkillName(toolName: string, input: unknown): AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS | undefined
```

**目的**：从 Skill 工具调用的输入中提取 Skill 名称用于分析。

### 6. 工具输入截断

```typescript
export function extractToolInputForTelemetry(input: unknown): string | undefined
```

**目的**：为 OTel tool_result 事件序列化工具输入参数。截断长字符串和深层嵌套以保持输出有界，同时保留文件路径、URL 等取证有用字段。

**截断规则**：
- 字符串超过 512 字符截断至 128 字符
- 最大 JSON 字符数：4KB
- 集合最大项数：20
- 最大深度：2

### 7. 文件扩展名提取

```typescript
export function getFileExtensionForAnalytics(filePath: string): AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS | undefined
export function getFileExtensionsFromBashCommand(command: string, simulatedSedEditFilePath?: string): AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS | undefined
```

**目的**：提取文件扩展名用于分析，同时避免记录潜在敏感的长扩展名（如哈希文件名）。支持从 Bash 命令中提取多个扩展名。

**支持的命令**：rm, mv, cp, touch, mkdir, chmod, chown, cat, head, tail, sort, stat, diff, wc, grep, rg, sed

### 8. 环境上下文构建

```typescript
export type EnvContext = { platform, arch, nodeVersion, terminal, packageManagers, runtimes, isCi, isClaubbit, ... }
const buildEnvContext = memoize(async (): Promise<EnvContext> => { ... })
```

**目的**：收集运行环境信息，包括平台、架构、运行时、CI 环境、远程环境等。使用 `memoize` 缓存避免重复计算。

### 9. 进程指标收集

```typescript
export type ProcessMetrics = { uptime, rss, heapTotal, heapUsed, external, arrayBuffers, constrainedMemory, cpuUsage, cpuPercent }
function buildProcessMetrics(): ProcessMetrics | undefined
```

**目的**：收集进程级指标，包括内存使用和 CPU 使用率（通过前后两次调用的差值计算）。

### 10. 事件元数据生成

```typescript
export async function getEventMetadata(options?: EnrichMetadataOptions): Promise<EventMetadata>
```

**目的**：生成所有分析系统共享的核心事件元数据，包括模型、会话 ID、用户类型、环境上下文、进程指标、代理识别等。

### 11. 1P 事件格式转换

```typescript
export function to1PEventFormat(metadata: EventMetadata, userMetadata: CoreUserData, additionalMetadata?: Record<string, unknown>): FirstPartyEventLoggingMetadata
```

**目的**：将元数据转换为 1P 事件日志格式（snake_case 字段），符合 `/api/event_logging/batch` 端点期望的格式。

---

## 具体技术实现

### 关键数据结构

```typescript
// 事件元数据核心类型
export type EventMetadata = {
  model: string                    // 使用的模型
  sessionId: string               // 会话 ID
  userType: string                // 用户类型
  betas?: string                  // 启用的 beta 功能
  envContext: EnvContext          // 环境上下文
  entrypoint?: string             // 入口点
  agentSdkVersion?: string        // Agent SDK 版本
  isInteractive: string           // 是否交互式会话
  clientType: string              // 客户端类型
  processMetrics?: ProcessMetrics // 进程指标
  sweBenchRunId: string           // SWE-bench 运行 ID
  sweBenchInstanceId: string      // SWE-bench 实例 ID
  sweBenchTaskId: string          // SWE-bench 任务 ID
  agentId?: string                // 代理 ID（格式：agentName@teamName 或 UUID）
  parentSessionId?: string        // 父会话 ID（团队领导的会话）
  agentType?: 'teammate' | 'subagent' | 'standalone'
  teamName?: string               // 团队名称
  subscriptionType?: string       // 订阅级别
  rh?: string                     // 仓库远程 URL 哈希（前 16 字符）
  kairosActive?: true             // KAIROS 助手模式激活
  skillMode?: 'discovery' | 'coach' | 'discovery_and_coach'
  observerMode?: 'backseat' | 'skillcoach' | 'both'
}

// 环境上下文
export type EnvContext = {
  platform: string
  platformRaw: string
  arch: string
  nodeVersion: string
  terminal: string | null
  packageManagers: string
  runtimes: string
  isRunningWithBun: boolean
  isCi: boolean
  isClaubbit: boolean
  isClaudeCodeRemote: boolean
  isLocalAgentMode: boolean
  isConductor: boolean
  remoteEnvironmentType?: string
  coworkerType?: string
  claudeCodeContainerId?: string
  claudeCodeRemoteSessionId?: string
  tags?: string
  isGithubAction: boolean
  isClaudeCodeAction: boolean
  isClaudeAiAuth: boolean
  version: string
  versionBase?: string
  buildTime: string
  deploymentEnvironment: string
  githubEventName?: string
  githubActionsRunnerEnvironment?: string
  githubActionsRunnerOs?: string
  githubActionRef?: string
  wslVersion?: string
  linuxDistroId?: string
  linuxDistroVersion?: string
  linuxKernel?: string
  vcs?: string
}
```

### 关键流程

#### 1. 代理识别流程

```typescript
function getAgentIdentification(): { agentId?, parentSessionId?, agentType?, teamName? } {
  // 1. 首先检查 AsyncLocalStorage 上下文（用于子代理）
  const agentContext = getAgentContext()
  if (agentContext) {
    return { agentId, parentSessionId, agentType, teamName }
  }
  
  // 2. 回退到环境变量（用于集群 teammate）
  const agentId = getAgentId()
  const parentSessionId = getTeammateParentSessionId()
  const teamName = getTeamName()
  
  // 3. 判断 agentType
  const agentType = isSwarmAgent ? 'teammate' : agentId ? 'standalone' : undefined
  
  // 4. 最后检查 bootstrap state
  const stateParentSessionId = getParentSessionIdFromState()
}
```

#### 2. 环境上下文构建流程

```typescript
const buildEnvContext = memoize(async (): Promise<EnvContext> => {
  const [packageManagers, runtimes, linuxDistroInfo, vcs] = await Promise.all([
    env.getPackageManagers(),
    env.getRuntimes(),
    getLinuxDistroInfo(),
    detectVcs(),
  ])
  
  return {
    platform: getHostPlatformForAnalytics(),
    platformRaw: process.env.CLAUDE_CODE_HOST_PLATFORM || process.platform,
    // ... 其他字段
  }
})
```

#### 3. 1P 格式转换流程

```typescript
export function to1PEventFormat(metadata, userMetadata, additionalMetadata): FirstPartyEventLoggingMetadata {
  // 1. 解构元数据
  const { envContext, processMetrics, rh, kairosActive, skillMode, observerMode, ...coreFields } = metadata
  
  // 2. 转换 envContext 为 snake_case（符合 proto 定义）
  const env: EnvironmentMetadata = { platform, platform_raw, arch, ... }
  
  // 3. 转换 core 字段为 snake_case
  const core: FirstPartyEventLoggingCoreMetadata = { session_id, model, user_type, ... }
  
  // 4. 构建 auth 对象
  const auth: PublicApiAuth = { account_uuid, organization_uuid }
  
  // 5. 返回完整结构
  return { env, process, auth, core, additional }
}
```

---

## 关键代码路径与文件引用

### 内部依赖

| 导入路径 | 用途 |
|---------|------|
| `../../utils/env.js` | `env`, `getHostPlatformForAnalytics` |
| `../../utils/envDynamic.js` | `envDynamic` |
| `../../utils/betas.js` | `getModelBetas` |
| `../../utils/model/model.js` | `getMainLoopModel` |
| `../../bootstrap/state.js` | `getSessionId`, `getIsInteractive`, `getKairosActive`, `getClientType`, `getParentSessionId` |
| `../../utils/envUtils.js` | `isEnvTruthy` |
| `../mcp/officialRegistry.js` | `isOfficialMcpUrl` |
| `../../utils/auth.js` | `isClaudeAISubscriber`, `getSubscriptionType` |
| `../../utils/git.js` | `getRepoRemoteHash` |
| `../../utils/platform.js` | `getWslVersion`, `getLinuxDistroInfo`, `detectVcs` |
| `../../utils/agentContext.js` | `getAgentContext` |
| `../../utils/teammate.js` | `getAgentId`, `getParentSessionId`, `getTeamName`, `isTeammate` |
| `../../utils/slowOperations.js` | `jsonStringify` |
| `bun:bundle` | `feature` (特性门控) |

### 被调用方（导出函数的使用者）

| 导出函数 | 使用者 |
|---------|--------|
| `sanitizeToolNameForAnalytics` | `useCanUseTool.tsx`, `permissionLogging.ts`, `PermissionContext.ts`, `FileReadTool.ts`, `toolExecution.ts`, `messages.ts`, `permissions.ts`, `sideQuery.ts`, `betaSessionTracing.ts`, `execAgentHook.ts`, `extractMemories.ts`, `yoloClassifier.ts`, `permissionExplainer.ts`, `toolResultStorage.ts`, `handlePromptSubmit.ts`, `FallbackPermissionRequest.tsx`, `useShellPermissionFeedback.ts`, `hooks.ts`, `usePermissionHandler.ts`, `BashPermissionRequest.tsx`, `useFilePermissionDialog.ts`, `SkillPermissionRequest.tsx`, `PowerShellPermissionRequest.tsx`, `logging.ts`, `auth.ts` |
| `getEventMetadata` | `datadog.ts`, `firstPartyEventLogger.ts` |
| `to1PEventFormat` | `firstPartyEventLoggingExporter.ts` |
| `extractMcpToolDetails` | 内部使用 |
| `extractSkillName` | 内部使用 |
| `extractToolInputForTelemetry` | 内部使用 |
| `getFileExtensionForAnalytics` | 内部使用 |
| `getFileExtensionsFromBashCommand` | 内部使用 |
| `isToolDetailsLoggingEnabled` | 内部使用 |
| `isAnalyticsToolDetailsLoggingEnabled` | 内部使用 |
| `mcpToolDetailsForAnalytics` | 内部使用 |

---

## 依赖与外部交互

### 外部系统交互

1. **环境变量**：读取大量环境变量以检测运行环境
   - `CI`, `CLAUBBIT`, `CLAUDE_CODE_REMOTE`, `CLAUDE_CODE_ENTRYPOINT`
   - `GITHUB_ACTIONS`, `GITHUB_EVENT_NAME`, `RUNNER_ENVIRONMENT`, `RUNNER_OS`
   - `CLAUDE_CODE_HOST_PLATFORM`, `CLAUDE_CODE_CONTAINER_ID`
   - `USER_TYPE`, `KAIROS` 等

2. **文件系统**：通过 `env.getPackageManagers()`, `env.getRuntimes()` 等间接交互

3. **Git**：通过 `getRepoRemoteHash()` 获取仓库远程哈希

4. **平台检测**：通过 `getWslVersion()`, `getLinuxDistroInfo()`, `detectVcs()` 获取平台信息

5. **AsyncLocalStorage**：通过 `getAgentContext()` 获取子代理上下文

6. **特性门控**：通过 `feature('CHICAGO_MCP')`, `feature('KAIROS')`, `feature('COWORKER_TYPE_TELEMETRY')` 检查功能开关

---

## 风险、边界与改进建议

### 风险点

1. **PII 泄露风险**
   - 尽管有类型系统保护，但仍需人工审查每个 `as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 断言
   - 工具输入截断可能仍包含敏感信息（如 API 密钥）

2. **性能风险**
   - `buildEnvContext` 使用 `memoize`，但 `getEventMetadata` 每次调用都会重新获取进程指标
   - CPU 百分比计算依赖全局状态（`prevCpuUsage`, `prevWallTimeMs`），在并发场景下可能不准确

3. **内存泄漏风险**
   - `memoize` 缓存的 `buildEnvContext` 和 `getVersionBase` 不会过期

4. **依赖循环风险**
   - 文件顶部注释强调 "NO dependencies to avoid import cycles"，但实际依赖较多
   - 需要小心维护依赖关系避免循环

### 边界情况

1. **AsyncLocalStorage 未设置**：`getAgentContext()` 返回 `undefined`，回退到环境变量
2. **环境变量缺失**：大量字段使用 `|| ''` 或条件展开，确保不会产生 `undefined`
3. **Git 仓库不存在**：`getRepoRemoteHash()` 可能返回 `undefined`
4. **平台检测失败**：`getLinuxDistroInfo()` 等可能返回 `undefined`
5. **进程指标获取失败**：`buildProcessMetrics()` 捕获所有异常并返回 `undefined`

### 改进建议

1. **类型安全增强**
   ```typescript
   // 考虑添加运行时验证
   export function assertSafeForAnalytics(value: unknown): asserts value is AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS {
     // 运行时检查：不包含路径分隔符、代码片段模式等
   }
   ```

2. **性能优化**
   - 考虑将进程指标收集改为采样模式，而非每次事件都收集
   - 使用更轻量的方式获取 CPU 使用率

3. **测试覆盖**
   - 当前没有专门的测试文件
   - 建议添加单元测试验证 PII 脱敏逻辑

4. **文档完善**
   - `EventMetadata` 中的 `rh` 字段含义不够直观，建议添加更详细的注释
   - `skillMode` 和 `observerMode` 的枚举值含义需要说明

5. **错误处理**
   - `getEventMetadata` 中的 `Promise.all` 如果失败会导致整个元数据获取失败
   - 建议为每个异步操作添加独立的错误处理
