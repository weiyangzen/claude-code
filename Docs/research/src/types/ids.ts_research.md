# ids.ts 研究文档

## 场景与职责

`src/types/ids.ts` 是 Claude Code CLI 的标识符类型定义文件，使用 TypeScript 的品牌类型（Branded Types）模式实现编译时的 ID 类型安全。核心职责：

1. **区分 Session ID 与 Agent ID**: 防止在编译期意外混用两种标识符
2. **提供类型安全的转换函数**: 支持字符串到品牌类型的安全转换
3. **Agent ID 格式验证**: 运行时验证 Agent ID 格式是否符合规范

该文件虽然代码量小（44 行），但在整个系统的类型安全架构中扮演关键角色，被 30+ 文件依赖。

## 功能点目的

### 1. 品牌类型（Branded Types）模式
使用交叉类型和 `readonly __brand` 属性创建名义类型（Nominal Typing）：
```typescript
type SessionId = string & { readonly __brand: 'SessionId' }
type AgentId = string & { readonly __brand: 'AgentId' }
```

优势：
- 编译时阻止 SessionId 与 AgentId 的误用
- 零运行时开销（品牌属性仅存在于类型层面）
- 与字符串兼容，可无缝用于需要 string 的 API

### 2. 类型转换函数
- `asSessionId(id: string)`: 将字符串断言为 SessionId（宽松转换）
- `asAgentId(id: string)`: 将字符串断言为 AgentId（宽松转换）
- `toAgentId(s: string)`: 验证格式后转换为 AgentId（严格转换，可能返回 null）

### 3. Agent ID 格式规范
正则表达式：`/^a(?:.+-)?[0-9a-f]{16}$/`

格式说明：
- 以 `a` 开头
- 可选的标签前缀（`<label>-`）
- 16 位十六进制字符
- 示例：`a1234567890abcdef`, `aworker-1234567890abcdef`

## 具体技术实现

### 关键数据结构

```typescript
/**
 * SessionId 品牌类型
 * 唯一标识一个 Claude Code 会话
 * 由 getSessionId() 返回
 */
export type SessionId = string & { readonly __brand: 'SessionId' }

/**
 * AgentId 品牌类型
 * 唯一标识会话内的子代理
 * 由 createAgentId() 返回
 * 存在时表示上下文是子代理（非主会话）
 */
export type AgentId = string & { readonly __brand: 'AgentId' }
```

### 转换函数实现

```typescript
// 宽松转换：直接类型断言
export function asSessionId(id: string): SessionId {
  return id as SessionId
}

export function asAgentId(id: string): AgentId {
  return id as AgentId
}

// 严格转换：格式验证
const AGENT_ID_PATTERN = /^a(?:.+-)?[0-9a-f]{16}$/

export function toAgentId(s: string): AgentId | null {
  return AGENT_ID_PATTERN.test(s) ? (s as AgentId) : null
}
```

### 使用模式

```typescript
// 正确：编译时类型检查
function processSession(sessionId: SessionId) { ... }
function processAgent(agentId: AgentId) { ... }

const session = asSessionId("abc123")
const agent = asAgentId("aworker-def4567890abcdef")

processSession(session)  // OK
processAgent(agent)      // OK
processSession(agent)    // 编译错误：Type 'AgentId' is not assignable to type 'SessionId'
```

## 关键代码路径与文件引用

### 类型定义
- `src/types/ids.ts` - 本文件，品牌类型定义

### ID 生成与管理
- `src/utils/uuid.ts` - UUID 和 AgentId 生成（`createAgentId()`）
- `src/bootstrap/state.ts` - Session ID 管理（`getSessionId()`）

### 主要使用者
- `src/Task.ts` - 任务管理
- `src/screens/REPL.tsx` - REPL 主界面
- `src/screens/ResumeConversation.tsx` - 会话恢复
- `src/services/compact/compact.ts` - 会话压缩
- `src/services/compact/sessionMemoryCompact.ts` - 内存压缩
- `src/services/api/claude.ts` - API 调用
- `src/services/api/promptCacheBreakDetection.ts` - 缓存检测
- `src/tools/AgentTool/AgentTool.tsx` - Agent 工具
- `src/tools/AgentTool/runAgent.ts` - Agent 执行
- `src/tools/AgentTool/resumeAgent.ts` - Agent 恢复
- `src/tools/SendMessageTool/SendMessageTool.ts` - 消息发送
- `src/tools/BashTool/BashTool.tsx` - Bash 工具
- `src/tools/PowerShellTool/PowerShellTool.tsx` - PowerShell 工具

### 会话存储
- `src/utils/sessionStorage.ts` - 会话存储（Agent 元数据、远程 Agent 元数据）
- `src/utils/sessionRestore.ts` - 会话恢复

### 子代理相关
- `src/tasks/LocalAgentTask/LocalAgentTask.tsx` - 本地 Agent 任务
- `src/tasks/LocalShellTask/LocalShellTask.tsx` - 本地 Shell 任务
- `src/tasks/LocalShellTask/killShellTasks.ts` - 任务终止
- `src/tasks/LocalShellTask/guards.ts` - 任务守卫
- `src/utils/forkedAgent.ts` - Fork Agent

## 依赖与外部交互

### 导入依赖
无外部导入，纯类型定义文件。

### 被依赖方（30+ 文件）
主要分布：
- 任务系统（`src/tasks/*`）
- 工具系统（`src/tools/*`）
- 服务层（`src/services/*`）
- UI 层（`src/screens/*`）
- CLI（`src/cli/*`）

## 风险、边界与改进建议

### 潜在风险

1. **宽松转换的安全性**
   - `asSessionId` 和 `asAgentId` 是纯类型断言，无运行时检查
   - 误用可能导致无效 ID 传播到系统各处

2. **Agent ID 格式变更**
   - `AGENT_ID_PATTERN` 硬编码在类型文件中
   - 如果 `createAgentId()` 的实现变更，正则可能不匹配

3. **序列化/反序列化**
   - 品牌类型在 JSON 序列化后会丢失品牌属性
   - 反序列化后需要重新应用类型断言

### 边界情况

1. **toAgentId 返回 null 的处理**
   ```typescript
   // 调用方必须处理 null 情况
   const agentId = toAgentId(someString)
   if (agentId) {
     // 是有效的 AgentId
   } else {
     // 可能是 teammate 名称、团队寻址等其他格式
   }
   ```

2. **SessionId 无格式验证**
   - 与 AgentId 不同，SessionId 没有 `toSessionId` 验证函数
   - 依赖外部系统（如 `getSessionId()`）保证格式正确

3. **品牌属性的运行时可见性**
   ```typescript
   // 编译时存在，运行时消失
   const id = asSessionId("abc")
   console.log(id.__brand)  // undefined，仅类型检查使用
   ```

### 改进建议

1. **增加 SessionId 格式验证**
   ```typescript
   const SESSION_ID_PATTERN = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i
   export function toSessionId(s: string): SessionId | null {
     return SESSION_ID_PATTERN.test(s) ? (s as SessionId) : null
   }
   ```

2. **统一 ID 生成与验证**
   - 将 `AGENT_ID_PATTERN` 与 `createAgentId()` 放在同一文件
   - 避免格式定义与实现分离导致的不一致

3. **增加 ID 类型工具**
   ```typescript
   // 建议：增加类型守卫
   export function isSessionId(id: string | SessionId | AgentId): id is SessionId {
     return typeof id === 'string' && SESSION_ID_PATTERN.test(id)
   }
   
   export function isAgentId(id: string | SessionId | AgentId): id is AgentId {
     return typeof id === 'string' && AGENT_ID_PATTERN.test(id)
   }
   ```

4. **文档完善**
   - 当前注释仅说明函数用途
   - 建议增加 Agent ID 格式规范的详细说明和使用示例

5. **考虑使用 Symbol 品牌**
   ```typescript
   // 更强的类型隔离（但会破坏字符串兼容性）
   declare const SessionIdBrand: unique symbol
   export type SessionId = string & { [SessionIdBrand]: never }
   ```
   权衡：Symbol 品牌提供更强隔离，但无法直接赋值给需要 string 的 API。
