# agentId.ts 深度研究文档

## 场景与职责

`agentId.ts` 是 Claude Code CLI 中用于**确定性代理 ID 生成与解析**的实用工具模块。它为 Swarm/Teammate 系统提供了一套标准化的 ID 格式，确保代理身份在崩溃恢复、重启后仍能保持可预测性和可重连性。

### 核心场景

1. **Swarm 队友身份标识**：在多代理团队中唯一标识每个队友代理
2. **请求追踪**：生成可解析的请求 ID，用于追踪关闭请求、计划审批等操作
3. **消息路由**：团队领导可以通过计算而非查找来确定队友的 ID，简化消息路由
4. **崩溃恢复**：确定性 ID 使得代理在崩溃/重启后能够被重新连接

### 设计哲学

模块注释明确阐述了确定性 ID 的优势：
- **可重现性**：相同名称的代理在相同团队中总是获得相同 ID
- **人类可读**：ID 具有语义（如 `tester@my-project`），便于调试
- **可预测性**：团队领导可以直接计算队友 ID，无需查找

---

## 功能点目的

### 1. Agent ID 格式

格式：`agentName@teamName`

示例：
- `team-lead@my-project`
- `researcher@my-project`
- `tester@my-project`

### 2. Request ID 格式

格式：`{requestType}-{timestamp}@{agentId}`

示例：
- `shutdown-1702500000000@researcher@my-project`
- `plan-approval-1702500000000@team-lead@my-project`

### 3. 解析功能

- `parseAgentId()`: 将 Agent ID 解析为组件
- `parseRequestId()`: 将 Request ID 解析为组件（requestType, timestamp, agentId）

---

## 具体技术实现

### ID 格式规范

```
Agent ID:    agentName@teamName
             └─────┬─────┘└──┬──┘
                name     team

Request ID:  requestType-timestamp@agentId
             └─────┬──────┘└───┬───┘└──┬──┘
                type      timestamp  agent
```

### 核心函数实现

#### 1. Agent ID 格式化
```typescript
export function formatAgentId(agentName: string, teamName: string): string {
  return `${agentName}@${teamName}`
}
```
- 简单字符串拼接
- 约束：`agentName` 不能包含 `@` 符号（作为分隔符）

#### 2. Agent ID 解析
```typescript
export function parseAgentId(
  agentId: string,
): { agentName: string; teamName: string } | null {
  const atIndex = agentId.indexOf('@')
  if (atIndex === -1) {
    return null
  }
  return {
    agentName: agentId.slice(0, atIndex),
    teamName: agentId.slice(atIndex + 1),
  }
}
```
- 查找第一个 `@` 分隔符
- 返回 `null` 表示格式无效

#### 3. Request ID 生成
```typescript
export function generateRequestId(requestType: string, agentId: string): string {
  const timestamp = Date.now()
  return `${requestType}-${timestamp}@${agentId}`
}
```
- 使用 `Date.now()` 获取毫秒级时间戳
- 格式：`{type}-{timestamp}@{agentId}`

#### 4. Request ID 解析
```typescript
export function parseRequestId(
  requestId: string,
): { requestType: string; timestamp: number; agentId: string } | null {
  const atIndex = requestId.indexOf('@')
  if (atIndex === -1) {
    return null
  }

  const prefix = requestId.slice(0, atIndex)
  const agentId = requestId.slice(atIndex + 1)

  const lastDashIndex = prefix.lastIndexOf('-')
  if (lastDashIndex === -1) {
    return null
  }

  const requestType = prefix.slice(0, lastDashIndex)
  const timestampStr = prefix.slice(lastDashIndex + 1)
  const timestamp = parseInt(timestampStr, 10)

  if (isNaN(timestamp)) {
    return null
  }

  return { requestType, timestamp, agentId }
}
```

**解析逻辑**：
1. 找到 `@` 分隔符，分割出前缀和 `agentId`
2. 在前缀中找到最后一个 `-`（因为 `requestType` 本身可能包含 `-`）
3. 分割出 `requestType` 和时间戳字符串
4. 解析时间戳，验证是否为有效数字

---

## 关键代码路径与文件引用

### 调用方（Consumers）

| 文件 | 用途 |
|------|------|
| `src/state/AppStateStore.ts` | 应用状态存储中的代理 ID 管理 |
| `src/utils/teammateMailbox.ts` | 队友邮箱系统 |
| `src/tools/TeamCreateTool/TeamCreateTool.ts` | 创建团队时生成代理 ID |
| `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts` | 退出计划模式工具 |
| `src/tools/shared/spawnMultiAgent.ts` | 多代理生成 |
| `src/tools/SendMessageTool/SendMessageTool.ts` | 发送消息时解析目标代理 |
| `src/utils/swarm/spawnInProcess.ts` | 进程内代理生成 |
| `src/utils/swarm/backends/InProcessBackend.ts` | 进程内后端 |
| `src/utils/swarm/backends/PaneBackendExecutor.ts` | Pane 后端执行器 |

### 使用模式

```typescript
// 生成 Agent ID
const agentId = formatAgentId('researcher', 'my-team')
// → "researcher@my-team"

// 解析 Agent ID
const parsed = parseAgentId('researcher@my-team')
// → { agentName: "researcher", teamName: "my-team" }

// 生成 Request ID
const requestId = generateRequestId('shutdown', 'researcher@my-team')
// → "shutdown-1702500000000@researcher@my-team"

// 解析 Request ID
const parsedReq = parseRequestId('shutdown-1702500000000@researcher@my-team')
// → { requestType: "shutdown", timestamp: 1702500000000, agentId: "researcher@my-team" }
```

---

## 依赖与外部交互

### 直接依赖

该模块**无外部依赖**，是一个纯工具函数模块：
```typescript
// 无任何 import 语句
```

### 仅使用 JavaScript 内置功能

- `String.prototype.indexOf()` - 查找分隔符位置
- `String.prototype.slice()` - 提取子字符串
- `String.prototype.lastIndexOf()` - 查找最后一个分隔符
- `parseInt()` - 解析时间戳
- `Date.now()` - 获取当前时间戳

---

## 风险、边界与改进建议

### 已知风险

1. **名称冲突风险**
   - `agentName` 不能包含 `@` 符号，否则会破坏解析
   - 注释提到使用 `sanitizeAgentName()` 从 `TeammateTool.ts` 来清理名称
   - 风险：如果未正确清理，可能导致解析失败或错误解析

2. **时间戳精度**
   - 使用 `Date.now()` 毫秒级时间戳
   - 在极高并发场景下，同一毫秒内生成的 Request ID 可能冲突
   - 当前设计假设这种情况很少发生，或调用方会处理冲突

3. **Request Type 解析歧义**
   - `requestType` 本身可以包含 `-`（如 `plan-approval`）
   - 解析逻辑使用 `lastIndexOf('-')`，这要求时间戳必须是数字
   - 如果 `requestType` 以数字结尾，可能导致解析错误

### 边界情况

1. **无效格式处理**
   ```typescript
   parseAgentId('invalid-id')        // → null
   parseRequestId('invalid')         // → null
   parseRequestId('type-nan@agent')  // → null (timestamp 不是数字)
   ```

2. **多 @ 符号情况**
   ```typescript
   parseAgentId('name@team@extra')   // → { agentName: "name", teamName: "team@extra" }
   // 只分割第一个 @，剩余部分作为 teamName
   ```

3. **空字符串处理**
   ```typescript
   parseAgentId('')                  // → null
   parseAgentId('@team')             // → { agentName: "", teamName: "team" }
   parseAgentId('name@')             // → { agentName: "name", teamName: "" }
   ```

### 改进建议

1. **添加名称验证**
   ```typescript
   export function formatAgentId(agentName: string, teamName: string): string {
     if (agentName.includes('@') || teamName.includes('@')) {
       throw new Error('Agent/team name cannot contain @ symbol')
     }
     return `${agentName}@${teamName}`
   }
   ```

2. **添加 Request ID 去重机制**
   ```typescript
   // 建议：添加序列号或随机后缀处理高并发场景
   export function generateRequestId(requestType: string, agentId: string): string {
     const timestamp = Date.now()
     const randomSuffix = Math.random().toString(36).slice(2, 6)
     return `${requestType}-${timestamp}-${randomSuffix}@${agentId}`
   }
   ```

3. **类型安全增强**
   ```typescript
   // 建议：使用 branded types 区分不同类型的 ID
   type AgentId = string & { __brand: 'AgentId' }
   type RequestId = string & { __brand: 'RequestId' }
   
   export function formatAgentId(agentName: string, teamName: string): AgentId {
     return `${agentName}@${teamName}` as AgentId
   }
   ```

4. **添加更多解析辅助函数**
   ```typescript
   // 建议：添加验证函数
   export function isValidAgentId(id: string): boolean {
     return parseAgentId(id) !== null
   }
   
   export function isValidRequestId(id: string): boolean {
     return parseRequestId(id) !== null
   }
   ```

5. **文档完善**
   - 添加 JSDoc 示例
   - 明确说明 ID 格式的约束和限制
   - 提供常见使用模式的代码示例

6. **考虑添加版本控制**
   - 未来如果需要改变 ID 格式，可以添加版本前缀
   - 例如：`v1:agentName@teamName`
