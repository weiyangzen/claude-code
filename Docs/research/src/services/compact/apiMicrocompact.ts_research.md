# apiMicrocompact.ts 深度研究文档

## 场景与职责

`apiMicrocompact.ts` 实现了基于 API 原生上下文管理策略的 microcompact 功能。它是 Claude Code 上下文压缩体系中的 API 层组件，负责生成与 Anthropic API 兼容的上下文管理配置，用于在服务端自动清理工具使用记录和 thinking 块。

该模块的主要定位是作为客户端 microcompact 的 API 端补充，当对话上下文接近模型限制时，通过 API 原生机制而非客户端消息重写来实现上下文压缩。

## 功能点目的

### 1. API 上下文管理策略配置生成
- 根据当前会话状态（thinking 模式、redact-thinking 激活状态等）生成对应的 API 上下文管理策略
- 支持两种核心策略类型：
  - `clear_tool_uses_20250919`: 清理工具使用记录
  - `clear_thinking_20251015`: 清理 thinking 块

### 2. 工具结果清理策略
- **工具结果清理** (`clear_tool_results`): 清理指定工具的输出结果，但保留工具调用记录
- **工具使用清理** (`clear_tool_uses`): 清理工具调用及其结果，但保留特定排除的工具

### 3. Thinking 块管理
- 在非 redact-thinking 模式下保留所有历史 thinking 块
- 在长时间空闲后（cache miss 场景）仅保留最后一个 thinking turn

### 4. 环境驱动的功能开关
- 通过 `USE_API_CLEAR_TOOL_RESULTS` 和 `USE_API_CLEAR_TOOL_USES` 环境变量控制功能启用
- 通过 `API_MAX_INPUT_TOKENS` 和 `API_TARGET_INPUT_TOKENS` 配置触发阈值

## 具体技术实现

### 关键数据结构

```typescript
// 上下文管理策略联合类型
export type ContextEditStrategy =
  | {
      type: 'clear_tool_uses_20250919'
      trigger?: { type: 'input_tokens'; value: number }
      keep?: { type: 'tool_uses'; value: number }
      clear_tool_inputs?: boolean | string[]
      exclude_tools?: string[]
      clear_at_least?: { type: 'input_tokens'; value: number }
    }
  | {
      type: 'clear_thinking_20251015'
      keep: { type: 'thinking_turns'; value: number } | 'all'
    }

// 配置包装器
export type ContextManagementConfig = {
  edits: ContextEditStrategy[]
}
```

### 核心流程

#### 1. 策略生成流程 (`getAPIContextManagement`)

```
输入: options (hasThinking, isRedactThinkingActive, clearAllThinking)
  │
  ├─> 初始化 strategies 数组
  │
  ├─> Thinking 策略判断
  │   ├─ hasThinking && !isRedactThinkingActive
  │   │   └─> 添加 clear_thinking_20251015 策略
  │   │       ├─ clearAllThinking ? keep 1 turn : keep all
  │   │
  │   └─ isRedactThinkingActive
  │       └─> 跳过 thinking 策略（redacted 块无模型可见内容）
  │
  ├─> 用户类型检查 (USER_TYPE !== 'ant')
  │   └─> 仅返回 thinking 策略（工具清理仅限内部用户）
  │
  ├─> 环境变量检查
  │   ├─ USE_API_CLEAR_TOOL_RESULTS
  │   │   └─> 添加 clear_tool_uses 策略（清理工具结果）
  │   │       - trigger: input_tokens >= threshold
  │   │       - clear_at_least: threshold - target
  │   │       - clear_tool_inputs: TOOLS_CLEARABLE_RESULTS
  │   │
  │   └─ USE_API_CLEAR_TOOL_USES
  │       └─> 添加 clear_tool_uses 策略（清理工具使用）
  │           - trigger: input_tokens >= threshold
  │           - clear_at_least: threshold - target
  │           - exclude_tools: TOOLS_CLEARABLE_USES
  │
  └─> 返回 ContextManagementConfig 或 undefined
```

#### 2. 工具清单定义

```typescript
// 可清理结果的工具（保留调用，清理输出）
const TOOLS_CLEARABLE_RESULTS = [
  ...SHELL_TOOL_NAMES,  // Bash, Shell 等
  GLOB_TOOL_NAME,
  GREP_TOOL_NAME,
  FILE_READ_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
]

// 可清理使用的工具（完全排除在清理外）
const TOOLS_CLEARABLE_USES = [
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
  NOTEBOOK_EDIT_TOOL_NAME,
]
```

### 默认阈值配置

```typescript
const DEFAULT_MAX_INPUT_TOKENS = 180_000    // 警告阈值
const DEFAULT_TARGET_INPUT_TOKENS = 40_000  // 保留目标（与客户端 microcompact 一致）
```

## 关键代码路径与文件引用

### 内部依赖

| 导入路径 | 用途 |
|---------|------|
| `src/tools/*/constants.js` / `prompt.js` | 工具名称常量 |
| `src/utils/shell/shellToolUtils.js` | Shell 工具名称集合 |
| `src/utils/envUtils.js` | 环境变量解析 (`isEnvTruthy`) |

### 外部调用方

| 调用方 | 路径 | 用途 |
|-------|------|------|
| `claude.ts` | `src/services/api/claude.ts:1633` | 在 API 请求中注入上下文管理配置 |

### 调用代码片段

```typescript
// src/services/api/claude.ts
const contextManagement = getAPIContextManagement({
  hasThinking,
  isRedactThinkingActive: betasParams.includes(REDACT_THINKING_BETA_HEADER),
  clearAllThinking: thinkingClearLatched,
})
// contextManagement 随后被合并到 API 请求参数中
```

## 依赖与外部交互

### 环境变量依赖

| 环境变量 | 类型 | 默认值 | 说明 |
|---------|------|--------|------|
| `USER_TYPE` | string | - | 用户类型，'ant' 才启用工具清理 |
| `USE_API_CLEAR_TOOL_RESULTS` | boolean | false | 启用工具结果清理 |
| `USE_API_CLEAR_TOOL_USES` | boolean | false | 启用工具使用清理 |
| `API_MAX_INPUT_TOKENS` | number | 180000 | 触发清理的输入 token 阈值 |
| `API_TARGET_INPUT_TOKENS` | number | 40000 | 清理后目标保留 token 数 |

### API 协议集成

该模块生成的配置直接映射到 Anthropic API 的 `context_management` 参数，遵循官方文档规范：
- 参考文档: https://docs.google.com/document/d/1oCT4evvWTh3P6z-kcfNQwWTCxAhkoFndSaNS9Gm40uw/edit

## 风险、边界与改进建议

### 已知风险

1. **用户类型限制**: 工具清理功能仅限 `USER_TYPE='ant'` 用户，外部用户无法使用，可能导致功能预期不一致

2. **阈值硬编码**: 默认阈值（180K/40K）是静态值，未针对不同模型上下文窗口动态调整

3. **策略冲突**: 当 `clear_tool_results` 和 `clear_tool_uses` 同时启用时，可能产生重叠或冲突的清理行为

4. **Thinking 清理边界**: 在 `clearAllThinking` 场景下强制保留 1 个 turn 是 API schema 要求，但可能不符合所有业务场景

### 边界条件

| 场景 | 行为 |
|------|------|
| `hasThinking=false` | 不生成 thinking 清理策略 |
| `isRedactThinkingActive=true` | 跳过 thinking 策略（redacted 块无可见内容） |
| `USER_TYPE !== 'ant'` | 仅返回 thinking 策略，无工具清理 |
| 两个环境变量都未设置 | 返回 undefined 或仅 thinking 策略 |
| `strategies` 为空数组 | 返回 undefined |

### 改进建议

1. **动态阈值调整**: 根据模型上下文窗口大小动态计算阈值，而非固定值
   ```typescript
   // 建议: 基于模型上下文窗口的百分比
   const threshold = contextWindow * 0.9
   const target = contextWindow * 0.2
   ```

2. **配置外部化**: 将阈值和工具清单迁移到远程配置（GrowthBook），支持动态调整

3. **策略优先级**: 明确多策略共存时的优先级规则，避免潜在的冲突

4. **外部用户策略**: 考虑为外部用户提供受限的工具清理能力，提升功能一致性

5. **监控与告警**: 添加策略生效的埋点，监控实际清理效果与预期的偏差
