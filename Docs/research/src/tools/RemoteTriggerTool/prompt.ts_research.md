# prompt.ts 深度研究文档

## 1. 场景与职责

### 1.1 功能定位
`prompt.ts` 是 `RemoteTriggerTool` 的**配置和文档中心**，负责定义：
- 工具的常量标识（名称、描述）
- 给 LLM 的系统提示词（Prompt）
- 工具的用户可见描述

### 1.2 设计原则
该文件遵循 Claude Code CLI 的工具提示词设计模式：
- **单一职责**: 纯常量定义，无业务逻辑
- **自文档化**: Prompt 本身包含完整的使用说明
- **安全优先**: 强调 OAuth Token 的进程内处理特性

### 1.3 使用场景
- **工具注册**: `RemoteTriggerTool.ts` 导入 `REMOTE_TRIGGER_TOOL_NAME` 和 `PROMPT`
- **LLM 上下文**: Prompt 被注入到系统提示词中，指导模型如何使用该工具
- **用户帮助**: `DESCRIPTION` 可用于工具列表和文档生成

---

## 2. 功能点目的

### 2.1 导出常量

| 常量 | 类型 | 用途 |
|------|------|------|
| `REMOTE_TRIGGER_TOOL_NAME` | `string` | 工具唯一标识符 |
| `DESCRIPTION` | `string` | 用户可见的简短描述 |
| `PROMPT` | `string` | 给 LLM 的详细使用指南 |

### 2.2 常量详解

#### REMOTE_TRIGGER_TOOL_NAME
```typescript
export const REMOTE_TRIGGER_TOOL_NAME = 'RemoteTrigger'
```
- 工具在系统中的唯一标识
- 用于工具注册、日志记录、权限检查

#### DESCRIPTION
```typescript
export const DESCRIPTION =
  'Manage scheduled remote Claude Code agents (triggers) via the claude.ai CCR API. Auth is handled in-process — the token never reaches the shell.'
```

**关键信息**：
1. 功能: 管理定时远程 Claude Code 代理
2. 技术: 通过 claude.ai CCR API
3. 安全: 进程内认证，Token 不暴露给 shell

#### PROMPT
```typescript
export const PROMPT = `Call the claude.ai remote-trigger API. Use this instead of curl — the OAuth token is added automatically in-process and never exposed.

Actions:
- list: GET /v1/code/triggers
- get: GET /v1/code/triggers/{trigger_id}
- create: POST /v1/code/triggers (requires body)
- update: POST /v1/code/triggers/{trigger_id} (requires body, partial update)
- run: POST /v1/code/triggers/{trigger_id}/run

The response is the raw JSON from the API.`
```

**结构分析**：
| 部分 | 内容 | 目的 |
|------|------|------|
| 概述 | 调用 claude.ai remote-trigger API | 明确工具用途 |
| 安全提示 | 使用此工具替代 curl，OAuth Token 自动添加 | 强调安全优势 |
| Actions | 5 个操作的详细说明 | 指导 LLM 正确使用 |
| 响应说明 | 返回 API 原始 JSON | 设定期望 |

---

## 3. 具体技术实现

### 3.1 文件结构

```typescript
// 纯常量导出，无依赖
export const REMOTE_TRIGGER_TOOL_NAME = 'RemoteTrigger'
export const DESCRIPTION = '...'
export const PROMPT = `...`
```

### 3.2 代码特点

1. **零依赖**: 不导入任何外部模块
2. **编译时常量**: 所有值在编译时确定
3. **字符串模板**: PROMPT 使用模板字符串支持多行
4. **类型推断**: TypeScript 自动推断为 `string` 类型

### 3.3 使用方式

```typescript
// RemoteTriggerTool.ts 中的导入和使用
import { DESCRIPTION, PROMPT, REMOTE_TRIGGER_TOOL_NAME } from './prompt.js'

export const RemoteTriggerTool = buildTool({
  name: REMOTE_TRIGGER_TOOL_NAME,
  // ...
  async description() {
    return DESCRIPTION
  },
  async prompt() {
    return PROMPT
  },
  // ...
})
```

---

## 4. 关键代码路径与文件引用

### 4.1 文件位置
```
src/tools/RemoteTriggerTool/
├── prompt.ts       # 本文件 (15 行)
├── RemoteTriggerTool.ts  # 主要使用者
└── UI.tsx          # 可能引用常量
```

### 4.2 引用关系

| 导入方 | 导入内容 | 用途 |
|--------|----------|------|
| `RemoteTriggerTool.ts` | `REMOTE_TRIGGER_TOOL_NAME` | 工具名称注册 |
| `RemoteTriggerTool.ts` | `DESCRIPTION` | 工具描述方法 |
| `RemoteTriggerTool.ts` | `PROMPT` | 工具提示词方法 |

### 4.3 代码路径

```
prompt.ts
├── 行 1: REMOTE_TRIGGER_TOOL_NAME 定义
├── 行 3-4: DESCRIPTION 定义
└── 行 6-15: PROMPT 定义
```

---

## 5. 依赖与外部交互

### 5.1 依赖分析

| 类型 | 数量 | 说明 |
|------|------|------|
| 导入模块 | 0 | 纯常量文件，无依赖 |
| 导出常量 | 3 | NAME, DESCRIPTION, PROMPT |

### 5.2 被依赖分析

```
RemoteTriggerTool.ts
  └── import { DESCRIPTION, PROMPT, REMOTE_TRIGGER_TOOL_NAME } from './prompt.js'
```

### 5.3 依赖关系图

```
prompt.ts (纯常量)
    │
    ├──► RemoteTriggerTool.ts (工具定义)
    │      ├──► Tool.ts (buildTool)
    │      └──► 系统工具注册表
    │
    └──► [可能的测试文件]
```

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

#### 风险 1: API 版本漂移
- **场景**: CCR API 端点或参数变更
- **当前状态**: Prompt 中硬编码了 API 路径
- **影响**: API 变更后 Prompt 可能过时
- **缓解**: 需要同步更新 `prompt.ts` 和 `RemoteTriggerTool.ts`

#### 风险 2: 提示词注入
- **场景**: 虽然不太可能，但用户输入可能通过某种方式影响 Prompt
- **当前状态**: Prompt 是纯静态字符串
- **评估**: ✅ 低风险，无动态内容

#### 风险 3: 功能描述不完整
- **场景**: Prompt 只描述了基本操作，未说明：
  - `trigger_id` 的格式要求（`^[\w-]+$`）
  - `body` 的具体结构
  - 错误处理方式
- **影响**: LLM 可能需要多次尝试才能正确使用

### 6.2 边界情况

| 场景 | 当前行为 | 评估 |
|------|----------|------|
| Prompt 超长 | 正常包含在系统提示词中 | ⚠️ 可能占用 Token |
| 多语言支持 | 仅英文 | ⚠️ 非英语用户可能受限 |
| 动态配置 | 编译时常量 | ✅ 确定性行为 |

### 6.3 改进建议

#### 建议 1: 添加 trigger_id 格式说明
```typescript
export const PROMPT = `Call the claude.ai remote-trigger API...

Actions:
- list: GET /v1/code/triggers
- get: GET /v1/code/triggers/{trigger_id}
- create: POST /v1/code/triggers (requires body)
- update: POST /v1/code/triggers/{trigger_id} (requires body, partial update)
- run: POST /v1/code/triggers/{trigger_id}/run

Parameters:
- trigger_id: Required for get, update, and run. Format: alphanumeric, underscores, and hyphens only.
- body: JSON object for create and update operations.

The response is the raw JSON from the API.`
```

#### 建议 2: 添加示例
```typescript
export const PROMPT = `Call the claude.ai remote-trigger API...

Actions:
...

Examples:
- List all triggers: { "action": "list" }
- Get a trigger: { "action": "get", "trigger_id": "my-trigger-123" }
- Create a trigger: { "action": "create", "body": { "name": "Daily Report", "schedule": "0 9 * * *" } }
- Run a trigger: { "action": "run", "trigger_id": "my-trigger-123" }

The response is the raw JSON from the API.`
```

#### 建议 3: 添加错误处理指南
```typescript
export const PROMPT = `...

Error Handling:
- 401 Unauthorized: User needs to run /login to authenticate
- 403 Forbidden: Organization policy may disallow remote sessions
- 404 Not Found: Trigger ID does not exist
- 422 Validation Error: Check body format and required fields

The response is the raw JSON from the API.`
```

#### 建议 4: 提取为配置对象
如果未来需要更复杂的配置，可以重构为：

```typescript
export const RemoteTriggerConfig = {
  name: 'RemoteTrigger',
  description: 'Manage scheduled remote Claude Code agents...',
  actions: {
    list: { method: 'GET', path: '/v1/code/triggers' },
    get: { method: 'GET', path: '/v1/code/triggers/{trigger_id}', requires: ['trigger_id'] },
    create: { method: 'POST', path: '/v1/code/triggers', requires: ['body'] },
    update: { method: 'POST', path: '/v1/code/triggers/{trigger_id}', requires: ['trigger_id', 'body'] },
    run: { method: 'POST', path: '/v1/code/triggers/{trigger_id}/run', requires: ['trigger_id'] },
  },
} as const

// 动态生成 PROMPT
export const PROMPT = generatePrompt(RemoteTriggerConfig)
```

#### 建议 5: 版本控制
添加版本号便于追踪变更：
```typescript
export const REMOTE_TRIGGER_TOOL_VERSION = '1.0.0'
export const REMOTE_TRIGGER_TOOL_NAME = 'RemoteTrigger'
```

### 6.4 国际化考虑

虽然 CLI 目前主要面向英语用户，但可以考虑：

```typescript
// 未来可能的国际化结构
const PROMPTS = {
  en: `...`,
  zh: `调用 claude.ai 远程触发器 API...`,
  ja: `...`,
}

export const PROMPT = PROMPTS[getLocale()] ?? PROMPTS.en
```

---

## 7. 附录

### 7.1 与其他工具 Prompt 的对比

| 工具 | Prompt 长度 | 复杂度 | 特点 |
|------|-------------|--------|------|
| RemoteTrigger | ~300 字符 | 低 | 清晰的 Action 列表 |
| BashTool | ~2000 字符 | 高 | 详细的权限和安全说明 |
| FileEditTool | ~1500 字符 | 中 | 包含 diff 格式说明 |
| WebFetchTool | ~500 字符 | 低 | 简单的 URL 获取 |

### 7.2 Prompt 工程分析

当前 Prompt 遵循了良好的 Prompt 工程实践：

1. **清晰的角色定义**: "Call the claude.ai remote-trigger API"
2. **明确的行动指南**: 5 个 Action 的详细说明
3. **安全强调**: 突出 OAuth Token 的进程内处理
4. **输出期望**: "The response is the raw JSON"

### 7.3 测试建议

1. **Prompt 完整性测试**: 确保所有 Action 都有文档
2. **一致性测试**: 验证 Prompt 与代码实现一致
3. **Token 计数测试**: 监控 Prompt 长度，避免占用过多上下文
4. **LLM 效果测试**: 评估 Prompt 对工具使用准确性的影响

### 7.4 相关文档

- [Claude Code 工具开发指南](../../../../../docs/tools.md)
- [系统提示词设计规范](../../../../../docs/prompt-design.md)
- [CCR API 文档](https://docs.anthropic.com/claude-code/api/triggers)
