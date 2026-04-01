# prompt.ts 研究文档

## 场景与职责

prompt.ts 是 SendMessageTool 的提示模板定义文件，负责：

1. **工具描述**：定义工具的简短描述（`DESCRIPTION`）
2. **动态提示生成**：根据功能开关（UDS_INBOX）生成不同的工具使用说明

该文件的内容会被提供给 AI 模型，指导模型如何正确使用 SendMessageTool。

## 功能点目的

### 1. 工具描述
- 简短的工具功能描述：`Send a message to another agent`
- 用于工具列表展示和模型理解

### 2. 动态提示生成
根据 `UDS_INBOX` 功能开关状态，生成不同的使用说明：

**UDS_INBOX 关闭时**：
- 基本的 teammate 名称和广播（`*`）用法
- 协议响应说明（legacy）

**UDS_INBOX 开启时**：
- 额外的跨会话通信支持
- UDS 地址格式：`uds:/path/to.sock`
- Bridge 地址格式：`bridge:session_...`
- 使用 `ListPeers` 发现目标
- 跨会话消息回复说明

## 具体技术实现

### 代码结构

```typescript
import { feature } from 'bun:bundle'

export const DESCRIPTION = 'Send a message to another agent'

export function getPrompt(): string {
  // 根据 UDS_INBOX feature 开关动态生成内容
  const udsRow = feature('UDS_INBOX') ? '...' : ''
  const udsSection = feature('UDS_INBOX') ? '...' : ''
  
  return `
# SendMessage

Send a message to another agent.

...  // 基础用法和表格

${udsSection}  // 跨会话说明（条件）

## Protocol responses (legacy)

...  // 协议响应说明
`.trim()
}
```

### 提示模板内容详解

#### 基础部分（始终包含）

```markdown
# SendMessage

Send a message to another agent.

{"to": "researcher", "summary": "assign task 1", "message": "start on task #1"}

| `to` | |
|---|---|
| `"researcher"` | Teammate by name |
| `"*"` | Broadcast to all teammates — expensive (linear in team size), use only when everyone genuinely needs it |
```

**关键指导**：
- 强调 plain text 输出对其他 agent 不可见
- 必须使用此工具进行通信
- 通过名称引用 teammates，而非 UUID
- 转发时不要引用原文（已渲染给用户）

#### UDS_INBOX 启用时额外内容

**表格扩展**：
```markdown
| `"uds:/path/to.sock"` | Local Claude session's socket (same machine; use `ListPeers`) |
| `"bridge:session_..."` | Remote Control peer session (cross-machine; use `ListPeers`) |
```

**跨会话说明**：
```markdown
## Cross-session

Use `ListPeers` to discover targets, then:

{"to": "uds:/tmp/cc-socks/1234.sock", "message": "check if tests pass over there"}
{"to": "bridge:session_01AbCd...", "message": "what branch are you on?"}

A listed peer is alive and will process your message — no "busy" state; 
messages enqueue and drain at the receiver's next tool round. 
Your message arrives wrapped as `<cross-session-message from="...">`. 
**To reply to an incoming message, copy its `from` attribute as your `to`.**
```

**关键指导**：
- 使用 `ListPeers` 发现目标
- 消息在接收者的下一个 tool round 处理
- 消息包装格式：`<cross-session-message from="...">`
- 回复时复制 `from` 属性作为 `to`

#### 协议响应（Legacy）

```markdown
## Protocol responses (legacy)

If you receive a JSON message with `type: "shutdown_request"` or 
`type: "plan_approval_request"`, respond with the matching `_response` type — 
echo the `request_id`, set `approve` true/false:

{"to": "team-lead", "message": {"type": "shutdown_response", "request_id": "...", "approve": true}}
{"to": "researcher", "message": {"type": "plan_approval_response", "request_id": "...", "approve": false, "feedback": "add error handling"}}

Approving shutdown terminates your process. Rejecting plan sends the teammate back to revise. 
Don't originate `shutdown_request` unless asked. Don't send structured JSON status messages — use TaskUpdate.
```

**关键指导**：
- 收到请求时回显 `request_id`
- `approve` 设置为 true/false
- 批准 shutdown 会终止进程
- 拒绝 plan 会让 teammate 重新修改
- 不要主动发起 `shutdown_request`
- 结构化状态消息使用 TaskUpdate 而非 SendMessage

## 关键代码路径与文件引用

### 核心文件
- `src/tools/SendMessageTool/prompt.ts` - 提示模板定义（49 行）

### 依赖
- `bun:bundle` 的 `feature` 函数 - 功能开关检查

### 使用该提示的文件
- `src/tools/SendMessageTool/SendMessageTool.ts`:
  ```typescript
  import { DESCRIPTION, getPrompt } from './prompt.js'
  
  async description() { return DESCRIPTION }
  async prompt() { return getPrompt() }
  ```

## 依赖与外部交互

### 运行时依赖
```typescript
import { feature } from 'bun:bundle'
```

`feature` 函数用于检查功能开关状态：
- `feature('UDS_INBOX')` - 是否启用跨会话通信

## 风险、边界与改进建议

### 已知限制

1. **功能开关依赖**
   - 提示内容在运行时根据 `UDS_INBOX` 开关变化
   - 如果开关状态在会话中改变，提示不会自动更新

2. **硬编码格式**
   - 消息格式示例是硬编码的 Markdown
   - 如果协议改变，需要同步更新此处

3. **国际化缺失**
   - 提示仅支持英文
   - 无多语言支持机制

### 边界情况

1. **空团队上下文**
   - 提示假设用户已在团队中
   - 如果不在团队中，工具会返回错误

2. **跨会话消息限制**
   - 提示说明结构化消息不能跨会话
   - 但此限制在 `validateInput` 中强制执行

### 改进建议

1. **动态示例生成**
   - 根据当前团队中的实际 teammates 生成示例
   - 提高模型理解的准确性

2. **错误示例**
   - 添加常见错误用法示例
   - 例如：错误地使用 `@` 符号、忘记 summary 等

3. **格式化输出**
   - 说明消息在接收方 UI 中的显示方式
   - 帮助用户理解消息的可视化效果

4. **版本控制**
   - 如果协议有重大变更，考虑添加版本标识
   - 便于调试和兼容性处理

5. **测试覆盖**
   - 验证生成的提示符合预期格式
   - 确保 feature 开关正确影响输出

### 代码质量

1. **可读性**：模板字符串结构清晰，易于理解
2. **可维护性**：功能开关逻辑集中，便于修改
3. **一致性**：与项目中其他工具的提示风格一致
