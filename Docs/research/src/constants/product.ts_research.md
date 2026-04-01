# product.ts 深度研究文档

## 场景与职责

`product.ts` 是 Claude Code CLI 中管理产品 URL 和远程会话环境检测的核心模块。它定义了 Claude.ai 各环境的基础 URL，并提供会话环境（生产、预发、本地开发）的检测和 URL 生成功能。

### 核心使用场景
1. **产品链接生成**：生成指向 Claude.ai 各页面的链接
2. **远程会话 URL 构造**：生成远程会话的 Web 访问链接
3. **环境检测**：根据会话 ID 和入口 URL 判断当前环境
4. **多环境支持**：支持生产、预发、本地开发三种环境

---

## 功能点目的

### 1. 产品 URL

| 常量 | 值 | 用途 |
|------|-----|------|
| `PRODUCT_URL` | `https://claude.com/claude-code` | Claude Code 产品主页 |

### 2. Claude.ai 基础 URL

| 常量 | 值 | 环境 |
|------|-----|------|
| `CLAUDE_AI_BASE_URL` | `https://claude.ai` | 生产环境 |
| `CLAUDE_AI_STAGING_BASE_URL` | `https://claude-ai.staging.ant.dev` | 预发环境 |
| `CLAUDE_AI_LOCAL_BASE_URL` | `http://localhost:4000` | 本地开发 |

### 3. 环境检测函数

#### `isRemoteSessionStaging(sessionId?, ingressUrl?)`

**检测逻辑**：
```typescript
return (
  sessionId?.includes('_staging_') === true ||
  ingressUrl?.includes('staging') === true
)
```

**检测依据**：
- 会话 ID 包含 `_staging_` 子串
- 入口 URL 包含 `staging` 子串

#### `isRemoteSessionLocal(sessionId?, ingressUrl?)`

**检测逻辑**：
```typescript
return (
  sessionId?.includes('_local_') === true ||
  ingressUrl?.includes('localhost') === true
)
```

**检测依据**：
- 会话 ID 包含 `_local_` 子串（如 `session_local_abc123`）
- 入口 URL 包含 `localhost`

### 4. URL 生成函数

#### `getClaudeAiBaseUrl(sessionId?, ingressUrl?)`

**逻辑**：
```
[isLocal] → CLAUDE_AI_LOCAL_BASE_URL
[isStaging] → CLAUDE_AI_STAGING_BASE_URL
[默认] → CLAUDE_AI_BASE_URL
```

#### `getRemoteSessionUrl(sessionId, ingressUrl?)`

**功能**：生成远程会话的完整 Web URL

**会话 ID 兼容性处理**：
- 背景：Worker 端点使用 `cse_*` 前缀，但前端路由使用 `session_*` 前缀
- 临时 shim：通过 `toCompatSessionId()` 转换
- 转换逻辑：`cse_*` → `session_*`，相同 UUID 主体，不同标签前缀

**实现**：
```typescript
export function getRemoteSessionUrl(sessionId: string, ingressUrl?: string): string {
  // 懒加载避免模块加载时循环依赖
  const { toCompatSessionId } = require('../bridge/sessionIdCompat.js')
  const compatId = toCompatSessionId(sessionId)
  const baseUrl = getClaudeAiBaseUrl(compatId, ingressUrl)
  return `${baseUrl}/code/${compatId}`
}
```

---

## 具体技术实现

### 数据结构

```typescript
// URL 常量
export const PRODUCT_URL = 'https://claude.com/claude-code'
export const CLAUDE_AI_BASE_URL = 'https://claude.ai'
export const CLAUDE_AI_STAGING_BASE_URL = 'https://claude-ai.staging.ant.dev'
export const CLAUDE_AI_LOCAL_BASE_URL = 'http://localhost:4000'

// 环境检测函数
export function isRemoteSessionStaging(sessionId?: string, ingressUrl?: string): boolean
export function isRemoteSessionLocal(sessionId?: string, ingressUrl?: string): boolean

// URL 生成函数
export function getClaudeAiBaseUrl(sessionId?: string, ingressUrl?: string): string
export function getRemoteSessionUrl(sessionId: string, ingressUrl?: string): string
```

### 关键代码路径

#### 1. 远程会话链接生成路径

```
创建或恢复远程会话
    ↓
src/tasks/RemoteAgentTask/RemoteAgentTask.tsx
    ↓
调用 getRemoteSessionUrl(sessionId, ingressUrl)
    ↓
生成可分享的 Web 链接
    ↓
显示给用户
```

**关键文件引用**：
- `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx`: 远程代理任务

#### 2. 桥接主流程路径

```
桥接模式初始化
    ↓
src/bridge/bridgeMain.ts
    ↓
检测环境并生成适当的 URL
    ↓
连接到远程会话
```

**关键文件引用**：
- `src/bridge/bridgeMain.ts`: 桥接主流程

#### 3. 桥接状态工具路径

```
获取桥接状态
    ↓
src/bridge/bridgeStatusUtil.ts
    ↓
使用环境检测函数
    ↓
返回状态信息
```

**关键文件引用**：
- `src/bridge/bridgeStatusUtil.ts`: 桥接状态工具

#### 4. CLI 打印路径

```
打印会话信息
    ↓
src/cli/print.ts
    ↓
生成远程会话 URL
    ↓
显示给用户
```

**关键文件引用**：
- `src/cli/print.ts`: CLI 打印

#### 5. 主入口路径

```
应用启动
    ↓
src/main.tsx
    ↓
检测远程会话环境
    ↓
初始化相应配置
```

**关键文件引用**：
- `src/main.tsx`: 应用主入口

#### 6. MCP 客户端路径

```
MCP 客户端初始化
    ↓
src/services/mcp/client.ts
    ↓
使用产品 URL 进行配置
    ↓
连接 MCP 服务
```

**关键文件引用**：
- `src/services/mcp/client.ts`: MCP 客户端

#### 7. REPL 桥接 Hook 路径

```
REPL 桥接模式
    ↓
src/hooks/useReplBridge.tsx
    ↓
使用环境检测
    ↓
管理桥接状态
```

**关键文件引用**：
- `src/hooks/useReplBridge.tsx`: REPL 桥接 Hook

#### 8. UltraPlan 命令路径

```
执行 UltraPlan 命令
    ↓
src/commands/ultraplan.tsx
    ↓
使用产品 URL
    ↓
生成计划链接
```

**关键文件引用**：
- `src/commands/ultraplan.tsx`: UltraPlan 命令

#### 9. 归因追踪路径

```
追踪功能使用
    ↓
src/utils/attribution.ts
    ↓
使用产品 URL
    ↓
记录归因数据
```

**关键文件引用**：
- `src/utils/attribution.ts`: 归因追踪

---

## 依赖与外部交互

### 内部依赖

**零依赖**：此文件不导入任何其他模块（除了 `require` 懒加载）。

### 被依赖方

| 文件 | 使用的常量/函数 | 用途 |
|------|----------------|------|
| `src/bridge/bridgeMain.ts` | `getClaudeAiBaseUrl`, `isRemoteSessionStaging`, `isRemoteSessionLocal` | 桥接主流程 |
| `src/bridge/bridgeStatusUtil.ts` | `isRemoteSessionStaging`, `isRemoteSessionLocal` | 桥接状态 |
| `src/cli/print.ts` | `getRemoteSessionUrl` | CLI 输出 |
| `src/main.tsx` | `getClaudeAiBaseUrl`, `isRemoteSessionStaging`, `isRemoteSessionLocal` | 应用入口 |
| `src/hooks/useReplBridge.tsx` | `isRemoteSessionLocal` | REPL 桥接 |
| `src/services/mcp/client.ts` | `PRODUCT_URL` | MCP 客户端 |
| `src/commands/ultraplan.tsx` | `PRODUCT_URL` | UltraPlan 命令 |
| `src/utils/attribution.ts` | `PRODUCT_URL` | 归因追踪 |
| `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | `getRemoteSessionUrl` | 远程代理任务 |

### 懒加载依赖

| 模块 | 用途 | 原因 |
|------|------|------|
| `../bridge/sessionIdCompat.js` | `toCompatSessionId` | 避免模块加载时循环依赖 |

### 外部服务交互

| 服务 | 交互方式 | 说明 |
|------|---------|------|
| **Claude.ai** | URL 生成 | 生成指向 claude.ai/code 的链接 |
| **Staging 环境** | URL 生成 | 内部测试环境 |
| **本地开发** | URL 生成 | 本地开发服务器 |

---

## 风险、边界与改进建议

### 当前风险

1. **硬编码 URL**
   - URL 硬编码在源代码中
   - 域名变更需要代码更新

2. **会话 ID 格式依赖**
   - 环境检测依赖会话 ID 包含特定子串
   - 格式变更可能导致检测失效

3. **懒加载复杂性**
   - `getRemoteSessionUrl` 使用 `require` 懒加载
   - 可能掩盖模块依赖问题

4. **CSE Shim 临时性**
   - 会话 ID 转换是临时 shim
   - 服务器端标签机制变更后需要更新

### 边界情况

| 场景 | 行为 |
|------|------|
| 会话 ID 为 undefined | 视为生产环境 |
| 同时匹配 local 和 staging | local 优先（代码顺序） |
| 会话 ID 已是 session_* | `toCompatSessionId` 无操作 |
| 入口 URL 包含多个关键词 | 第一个匹配决定环境 |
| 本地开发使用 IP 而非 localhost | 可能无法正确识别为本地环境 |

### 改进建议

1. **配置外部化**
   ```typescript
   // 建议从配置文件读取 URL
   export function getClaudeAiBaseUrl(sessionId?: string, ingressUrl?: string): string {
     const config = loadProductConfig()
     if (isRemoteSessionLocal(sessionId, ingressUrl)) {
       return config.localBaseUrl || CLAUDE_AI_LOCAL_BASE_URL
     }
     // ...
   }
   ```

2. **环境检测增强**
   ```typescript
   // 建议添加更多检测方式
   export function detectEnvironment(
     sessionId?: string,
     ingressUrl?: string,
     apiEndpoint?: string
   ): 'production' | 'staging' | 'local' | 'unknown' {
     // 综合多种信号
   }
   ```

3. **URL 构建器**
   ```typescript
   // 建议添加 URL 构建器
   export class ClaudeAiUrlBuilder {
     private baseUrl: string
     
     constructor(env: Environment) {
       this.baseUrl = getBaseUrlForEnv(env)
     }
     
     session(sessionId: string): string {
       return `${this.baseUrl}/code/${sessionId}`
     }
     
     settings(): string {
       return `${this.baseUrl}/settings`
     }
     
     connectors(): string {
       return `${this.baseUrl}/settings/connectors`
     }
   }
   ```

4. **会话 ID 类型安全**
   ```typescript
   // 建议使用 branded type
   type SessionId = string & { __brand: 'SessionId' }
   type CseSessionId = string & { __brand: 'CseSessionId' }
   
   export function toCompatSessionId(id: CseSessionId | SessionId): SessionId
   export function getRemoteSessionUrl(sessionId: SessionId, ingressUrl?: string): string
   ```

5. **文档化 shim**
   ```typescript
   /**
    * 生成远程会话 URL
    * 
    * @deprecated CSE shim 临时方案，将在服务器端标签机制变更后移除
    * @see https://github.com/anthropics/claude-code/issues/XXXX
    */
   export function getRemoteSessionUrl(sessionId: string, ingressUrl?: string): string
   ```

6. **自动化测试**
   - 各种会话 ID 格式的环境检测
   - URL 生成正确性
   - 环境切换边界条件

### 与桥接系统的关系

```
product.ts (URL 和环境管理)
    ↓ 被使用
bridge/*.ts (桥接系统)
    ↓ 连接
Claude.ai Web (远程会话)
```

产品 URL 和环境检测是桥接功能的基础，确保 CLI 能正确连接到适当的远程会话环境。
