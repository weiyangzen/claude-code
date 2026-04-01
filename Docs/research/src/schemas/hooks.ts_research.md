# src/schemas/hooks.ts 研究文档

## 1. 场景与职责

### 1.1 文件定位

`src/schemas/hooks.ts` 是 Claude Code 项目中 **Hook 系统的核心 Schema 定义文件**，专门用于解决循环依赖问题而提取的独立模块。该文件定义了用户可配置的 Hook（钩子）的数据结构和验证规则。

### 1.2 核心职责

| 职责 | 说明 |
|------|------|
| **Schema 定义** | 使用 Zod 定义所有 Hook 相关的数据验证 schema |
| **类型导出** | 导出 TypeScript 类型供整个项目使用 |
| **循环依赖破解** | 原本位于 `src/utils/settings/types.ts`，提取后解决了与 `src/utils/plugins/schemas.ts` 的循环依赖 |
| **配置验证** | 验证用户 settings.json 中的 hooks 配置合法性 |

### 1.3 使用场景

1. **用户配置验证**：验证 `.claude/settings.json` 中的 `hooks` 字段
2. **插件 Hook 验证**：验证插件提供的 `hooks/hooks.json` 配置
3. **Agent Frontmatter 验证**：验证 Agent 定义文件中的 hooks 配置
4. **Skill Hook 验证**：验证 Skill 目录中的 hooks 配置

---

## 2. 功能点目的

### 2.1 Hook 类型支持

文件定义了 **4 种可序列化的 Hook 类型**：

| Hook 类型 | 用途 | 执行方式 |
|-----------|------|----------|
| `command` | 执行 shell 命令 | Bash/PowerShell 子进程 |
| `prompt` | LLM 提示词评估 | 调用轻量级模型（Haiku） |
| `agent` | 多轮 Agent 验证 | 完整 query() 循环 |
| `http` | HTTP POST 请求 | Axios 发送 JSON 到指定 URL |

> **注意**：`callback` 和 `function` 类型的 Hook 是内存中的运行时类型，**不能持久化到 settings.json**，因此不包含在此 schema 中。

### 2.2 Hook 事件类型

Hook 可以绑定到 **24 种生命周期事件**（定义在 `src/entrypoints/sdk/coreTypes.ts`）：

```typescript
HOOK_EVENTS = [
  'PreToolUse',        // 工具使用前
  'PostToolUse',       // 工具使用后
  'PostToolUseFailure', // 工具使用失败
  'PermissionRequest', // 权限请求时
  'PermissionDenied',  // 权限被拒绝
  'UserPromptSubmit',  // 用户提交提示词
  'SessionStart',      // 会话开始
  'SessionEnd',        // 会话结束
  'Stop',              // 停止时
  'StopFailure',       // 停止失败
  'SubagentStart',     // 子代理开始
  'SubagentStop',      // 子代理停止
  'PreCompact',        // 压缩前
  'PostCompact',       // 压缩后
  'Notification',      // 通知事件
  'Setup',             // 设置时
  'TeammateIdle',      // 队友空闲
  'TaskCreated',       // 任务创建
  'TaskCompleted',     // 任务完成
  'Elicitation',       // MCP 引导请求
  'ElicitationResult', // MCP 引导结果
  'ConfigChange',      // 配置变更
  'CwdChanged',        // 工作目录变更
  'FileChanged',       // 文件变更
  'WorktreeCreate',    // 工作树创建
  'WorktreeRemove',    // 工作树删除
  'InstructionsLoaded', // 指令加载
]
```

### 2.3 条件过滤（If Condition）

支持 `if` 条件字段，使用权限规则语法（如 `"Bash(git *)"`、`"Read(*.ts)"`）在 spawn 前过滤 Hook，避免不必要的进程创建。

---

## 3. 具体技术实现

### 3.1 核心数据结构

```typescript
// 文件导出的主要 Schema 和类型
export const HookCommandSchema     // 单个 Hook 命令的联合类型 Schema
export const HookMatcherSchema     // Hook 匹配器 Schema（matcher + hooks 数组）
export const HooksSchema           // 完整 Hooks 配置 Schema（按事件分组的记录）

// 导出的 TypeScript 类型
export type HookCommand            // 联合类型：BashCommandHook | PromptHook | AgentHook | HttpHook
export type BashCommandHook        // { type: 'command', command: string, shell?: 'bash'|'powershell', ... }
export type PromptHook             // { type: 'prompt', prompt: string, model?: string, ... }
export type AgentHook              // { type: 'agent', prompt: string, model?: string, ... }
export type HttpHook               // { type: 'http', url: string, headers?: Record, ... }
export type HookMatcher            // { matcher?: string, hooks: HookCommand[] }
export type HooksSettings          // Partial<Record<HookEvent, HookMatcher[]>>
```

### 3.2 内部工厂函数

```typescript
// 内部工厂函数，构建 4 种 Hook Schema
function buildHookSchemas() {
  const BashCommandHookSchema = z.object({...})
  const PromptHookSchema = z.object({...})
  const HttpHookSchema = z.object({...})
  const AgentHookSchema = z.object({...})
  return { BashCommandHookSchema, PromptHookSchema, HttpHookSchema, AgentHookSchema }
}
```

### 3.3 延迟加载模式（Lazy Schema）

所有 Schema 使用 `lazySchema()` 包装，延迟到首次访问时才构建，避免模块初始化时的性能开销：

```typescript
export const HookCommandSchema = lazySchema(() => {
  const { ... } = buildHookSchemas()
  return z.discriminatedUnion('type', [...])
})
```

### 3.4 各 Hook 类型的详细字段

#### BashCommandHook
```typescript
{
  type: 'command',
  command: string,           // 要执行的 shell 命令
  if?: string,               // 条件过滤（权限规则语法）
  shell?: 'bash' | 'powershell', // 默认 'bash'
  timeout?: number,          // 超时秒数
  statusMessage?: string,    // 旋转提示文本
  once?: boolean,            // 执行一次后移除
  async?: boolean,           // 后台异步执行
  asyncRewake?: boolean,     // 后台执行，exit code 2 时唤醒模型
}
```

#### PromptHook
```typescript
{
  type: 'prompt',
  prompt: string,            // LLM 提示词，支持 $ARGUMENTS 占位符
  if?: string,               // 条件过滤
  timeout?: number,          // 超时秒数
  model?: string,            // 模型 ID（如 "claude-sonnet-4-6"）
  statusMessage?: string,    // 旋转提示文本
  once?: boolean,            // 执行一次后移除
}
```

#### AgentHook
```typescript
{
  type: 'agent',
  prompt: string,            // Agent 提示词，支持 $ARGUMENTS 占位符
  if?: string,               // 条件过滤
  timeout?: number,          // 超时秒数（默认 60s）
  model?: string,            // 模型 ID（默认 Haiku）
  statusMessage?: string,    // 旋转提示文本
  once?: boolean,            // 执行一次后移除
}
```

#### HttpHook
```typescript
{
  type: 'http',
  url: string,               // POST 目标 URL
  if?: string,               // 条件过滤
  timeout?: number,          // 超时秒数
  headers?: Record<string, string>, // 请求头（支持 $VAR 环境变量）
  allowedEnvVars?: string[], // 允许插值的环境变量白名单
  statusMessage?: string,    // 旋转提示文本
  once?: boolean,            // 执行一次后移除
}
```

---

## 4. 关键代码路径与文件引用

### 4.1 导入本文件的模块

| 文件路径 | 用途 |
|----------|------|
| `src/utils/settings/types.ts` | 重新导出所有类型和 Schema，向后兼容 |
| `src/utils/plugins/schemas.ts` | PluginHooksSchema 使用 HooksSchema 验证插件 hooks |
| `src/tools/AgentTool/loadAgentsDir.ts` | 验证 Agent frontmatter 中的 hooks |
| `src/skills/loadSkillsDir.ts` | 验证 Skill frontmatter 中的 hooks |

### 4.2 执行 Hook 的核心模块

| 文件路径 | 职责 |
|----------|------|
| `src/utils/hooks.ts` | **主执行引擎**，包含 `executeHooks()` 和所有事件特定的执行函数 |
| `src/utils/hooks/execCommandHook.ts` | 执行 `command` 类型 Hook（内部函数，在 hooks.ts 中） |
| `src/utils/hooks/execPromptHook.ts` | 执行 `prompt` 类型 Hook |
| `src/utils/hooks/execAgentHook.ts` | 执行 `agent` 类型 Hook |
| `src/utils/hooks/execHttpHook.ts` | 执行 `http` 类型 Hook |
| `src/utils/hooks/sessionHooks.ts` | 管理会话级别的内存 Hook（callback/function 类型） |
| `src/utils/hooks/hookHelpers.ts` | 共享工具函数（参数替换、结构化输出工具） |

### 4.3 关键执行流程

```
用户操作/生命周期事件
    ↓
executeXXXHooks()  // 如 executePreToolHooks(), executeStopHooks()
    ↓
getMatchingHooks() // 从 settings/plugins/skills/session 获取匹配的 hooks
    ↓
executeHooks()     // 主执行引擎
    ↓
按 hook.type 分发：
    - 'command' → execCommandHook() → spawn bash/powershell
    - 'prompt'  → execPromptHook()  → queryModelWithoutStreaming()
    - 'agent'   → execAgentHook()   → query() 多轮循环
    - 'http'    → execHttpHook()    → axios.post()
    - 'callback'→ 直接调用 callback 函数
    - 'function'→ 直接调用 callback 函数
    ↓
processHookJSONOutput() // 解析 Hook 返回的 JSON 结果
    ↓
应用结果（阻止操作、修改输入、添加上下文等）
```

### 4.4 Hook JSON 输出协议

Hook 可以通过 stdout/HTTP body 返回 JSON 控制执行流程：

```typescript
// 同步响应
{
  continue?: boolean,        // false = 阻止继续执行
  stopReason?: string,       // 阻止原因
  decision?: 'approve'|'block', // 权限决定
  reason?: string,           // 决定原因
  systemMessage?: string,    // 系统消息
  suppressOutput?: boolean,  // 隐藏输出
  hookSpecificOutput?: {     // 事件特定输出
    hookEventName: 'PreToolUse'|'UserPromptSubmit'|...
    // ... 事件特定字段
  }
}

// 异步响应（后台执行）
{
  async: true,
  asyncTimeout?: number
}
```

---

## 5. 依赖与外部交互

### 5.1 直接依赖

```typescript
import { HOOK_EVENTS, type HookEvent } from 'src/entrypoints/agentSdkTypes.js'
import { z } from 'zod/v4'
import { lazySchema } from '../utils/lazySchema.js'
import { SHELL_TYPES } from '../utils/shell/shellProvider.js'
```

| 依赖 | 用途 |
|------|------|
| `agentSdkTypes.js` | 获取 HOOK_EVENTS 常量（24 个生命周期事件） |
| `zod/v4` | Schema 定义和验证 |
| `lazySchema.js` | 延迟加载包装器 |
| `shellProvider.js` | 获取支持的 shell 类型（bash/powershell） |

### 5.2 反向依赖（使用本文件的模块）

```
src/utils/settings/types.ts       ← 重新导出
src/utils/plugins/schemas.ts      ← PluginHooksSchema 使用 HooksSchema
src/tools/AgentTool/loadAgentsDir.ts ← Agent frontmatter hooks 验证
src/skills/loadSkillsDir.ts       ← Skill frontmatter hooks 验证
```

### 5.3 配置集成

Hook 配置可以来自多个来源，按优先级合并：

1. **Managed Settings**（`managed-settings.json`）- 企业策略
2. **User Settings**（`~/.claude/settings.json`）
3. **Project Settings**（`.claude/settings.json`）
4. **Local Settings**（`.claude/local.json`）
5. **Plugins**（`hooks/hooks.json`）
6. **Skills**（frontmatter 中的 hooks）
7. **Session Hooks**（运行时内存中的 callback/function hooks）

---

## 6. 风险、边界与改进建议

### 6.1 已知风险

| 风险 | 说明 | 缓解措施 |
|------|------|----------|
| **RCE 风险** | Hook 执行任意 shell 命令 | 强制要求 workspace trust；`shouldSkipHookDueToTrust()` 检查 |
| **SSRF 风险** | HTTP Hook 可能访问内部网络 | `ssrfGuardedLookup` 拦截；URL 白名单 `allowedHttpHookUrls` |
| **Header 注入** | HTTP Hook headers 可能包含 CRLF | `sanitizeHeaderValue()` 清理 `\r\n\x00` |
| **Env Var 泄露** | HTTP headers 可能泄露敏感环境变量 | `allowedEnvVars` 白名单限制插值 |
| **无限递归** | Prompt Hook 可能触发 UserPromptSubmit Hook | `execPromptHook()` 使用 `createUserMessage()` 而非 `processUserInput()` |
| **Agent Hook 循环** | Agent Hook 可能创建子 Agent | `ALL_AGENT_DISALLOWED_TOOLS` 禁止子代理工具 |

### 6.2 边界情况

1. **SessionStart/Setup 不支持 HTTP Hooks**：headless 模式下 sandbox ask callback 会死锁
2. **Windows 路径转换**：bash hooks 需要 POSIX 路径（`/c/Users/...`），PowerShell hooks 使用原生路径
3. **Exit Code 2 语义**：命令 hooks 返回 exit code 2 表示"阻止操作"（blocking error）
4. **Async Hook 生命周期**：asyncRewake hooks 在 exit code 2 时通过 `enqueuePendingNotification` 唤醒模型
5. **Hook 去重**：相同命令/prompt/URL 在同一来源内会去重，但不同来源（settings vs plugin）会分别执行

### 6.3 代码注释中的重要警告

```typescript
// AgentHookSchema.prompt 字段的注释警告：
// DO NOT add .transform() here. This schema is used by parseSettingsFile,
// and updateSettingsForSource round-trips the parsed result through
// JSON.stringify — a transformed function value is silently dropped,
// deleting the user's prompt from settings.json (gh-24920, CC-79).
```

### 6.4 改进建议

| 建议 | 优先级 | 说明 |
|------|--------|------|
| **Schema 版本控制** | 中 | 当前无版本字段，未来 breaking changes 难以迁移 |
| **Hook 执行超时默认** | 低 | 不同 hook 类型使用不同默认超时，考虑统一配置 |
| **HTTP Hook 重试机制** | 低 | 当前无自动重试，网络抖动可能导致失败 |
| **Hook 性能监控** | 中 | 已有 `hook_duration_ms` 指标，可考虑添加 P95/P99 统计 |
| **Hook 调试工具** | 低 | 当前依赖 `--debug` 日志，可考虑专用 `/hooks/debug` 命令 |
| **TypeScript 严格模式** | 低 | 部分 Hook 类型定义可进一步收紧（如 model 字段的枚举） |

### 6.5 测试覆盖建议

当前未发现专门针对 `src/schemas/hooks.ts` 的单元测试文件。建议测试：

1. **Schema 验证测试**：验证各种合法/非法配置的正确识别
2. **类型兼容性测试**：确保导出的 TypeScript 类型与 Zod schema 一致
3. **循环依赖测试**：确保提取到独立文件后无循环依赖
4. **Hook 执行集成测试**：覆盖 4 种 hook 类型的完整执行流程

---

## 7. 附录：相关文件索引

### 7.1 核心文件

- `src/schemas/hooks.ts` - **本文件**，Schema 定义
- `src/utils/settings/types.ts` - 设置类型定义，重新导出本文件
- `src/utils/hooks.ts` - Hook 执行主引擎（~4000 行）
- `src/types/hooks.ts` - Hook 运行时类型定义（HookJSONOutput 等）

### 7.2 执行实现

- `src/utils/hooks/execPromptHook.ts` - Prompt Hook 执行
- `src/utils/hooks/execAgentHook.ts` - Agent Hook 执行
- `src/utils/hooks/execHttpHook.ts` - HTTP Hook 执行
- `src/utils/hooks/sessionHooks.ts` - Session Hook 管理
- `src/utils/hooks/hookHelpers.ts` - 共享工具函数
- `src/utils/hooks/hookEvents.ts` - Hook 事件发射
- `src/utils/hooks/hooksConfigSnapshot.ts` - Hook 配置快照
- `src/utils/hooks/hooksSettings.ts` - Hook 设置工具

### 7.3 使用方

- `src/utils/plugins/schemas.ts` - 插件 Schema，使用 HooksSchema
- `src/tools/AgentTool/loadAgentsDir.ts` - Agent 加载，验证 frontmatter hooks
- `src/skills/loadSkillsDir.ts` - Skill 加载，验证 frontmatter hooks
- `src/entrypoints/sdk/coreTypes.ts` - HOOK_EVENTS 常量定义

---

*文档生成时间：2026-04-01*
*研究范围：代码、配置、测试、脚本*
*排除范围：README、docs、Docs、markdown 等文档*
